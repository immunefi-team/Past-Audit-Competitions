# \#99498 [SC-Low] A temporary adverse mark at the first LP settlement after a recapitalization resets the Senior high-water mark and pays Senior principal to Junior

**Submitted on Nov 13th 2026 at 22:14:30 UTC by @Cryptor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99498
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

When the Senior tranche is impaired, the protocol's recovery is `CfdEngine.recapitalizePool`, an immediate owner action that moves USDG into HousePool and queues it as a pending `Recapitalization` inflow for "normal pool reconciliation" (`CfdEngine.sol:627-630`, `SECURITY.md:155-162`). The repository asserts what that reconciliation must do for an impaired Senior tranche that still holds principal: "Recapitalization should still restore senior principal to the HWM" (`test/perps/accounting/waterfall/HousePoolWaterfall.t.sol:389-418`).

`HousePool._reconcile` first reconciles the stored tranche principals against the current mark and writes the result to storage (`HousePool.sol:1584-1585`). Only afterwards does it apply the queued recapitalization, from a state re-read from storage (`HousePool.sol:1598`, `:1768`). `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent` picks its branch from that post-mark state. If the mark has just zeroed both principals, it takes the bootstrap branch and overwrites `seniorHighWaterMark` with the recapitalization amount (`HousePoolPendingPreviewLib.sol:118-122`). The previous high-water mark, which `absorbLoss` deliberately keeps "so future revenue continues to restore senior impairment" (`HousePoolWaterfallAccountingLib.sol:161-163`), is erased.

So the first committed LP settlement after a recapitalization decides the outcome. If it runs at a mark where open positions show an unrealized trader gain at least as large as the Senior principal left before the recapitalization, Senior's restoration target drops from its high-water mark `H` to the recapitalization amount `H - S`. When the mark returns, the waterfall restores Senior only to `H - S` and credits the rest of the recovery to Junior principal (`HousePoolWaterfallAccountingLib.sol:147-158`). A Junior holder whose shares were worth zero redeems it in USDG. Every step after the owner's recapitalization is a public call. The settlement is the permissionless `OrderRouter.settleLpEpoch`, and a Junior holder's own zero-value redemption request supplies the epoch progress it needs.

The proof of concept runs at release risk parameters, with the pool never in degraded mode. After a recapitalization of exactly the recorded Senior gap, a 5.08% temporary mark at the first settlement makes the Senior LP exit with 25,871.625416 USDG less, and the Junior holder redeems 25,452.816597 USDG, against zero in the same-state control. With a 22.23% mark the figures are 116,147.796536 and 106,701.447771 USDG.

## Vulnerability Details

#### The recapitalization is routed after the mark-driven plan is stored

```solidity
// packages/perps/src/HousePool.sol:1555
function _reconcile(...) internal {
    ...
    HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets = _getPendingClaimantBuckets();
    HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets =
        _settleablePendingClaimantBuckets(reconcileSnapshot, claimantBuckets);
    bool allowRevenueContinuation = seniorPrincipal + juniorPrincipal != 0;              // :1567
    HousePoolReconcilePlanLib.ReconcilePlan memory plan = HousePoolReconcilePlanLib.planReconcile(
        _currentPendingAccountingState(),
        reconcileSnapshot,
        _pendingClaimantBucketAssets(settleableClaimantBuckets),                        // pending reserved first
        ...
    );
    ...
    _setWaterfallState(plan.state.waterfall);                                            // :1584, mark-driven result stored
    unassignedAssets = plan.state.unassignedAssets;
    ...
    _applyPendingClaimantBucketsLive(settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation); // :1598
}
```

`planReconcile` subtracts the settleable pending buckets from distributable equity and reconciles the stored principals against what is left at the current mark. When an open position's unrealized gain is at least the stored Senior principal, nothing is left. `absorbLoss` then zeroes both principals and leaves the high-water mark unchanged (`HousePoolWaterfallAccountingLib.sol:161-185`), and the zeros are written to storage.

`_applyPendingClaimantBucketsLive` then reads the stored state back (`HousePool.sol:1768`) and applies the recapitalization to it:

```solidity
// packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:111
function applyClaimantRecapitalizationIntent(PendingAccountingState memory state, uint256 amount, uint256 recapitalizationTargetAssets)
    internal pure
{
    uint256 remaining = amount;
    if (state.seniorSupply > 0) {
        if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {   // :118, post-mark zeros
            state.waterfall.seniorPrincipal += remaining;
            state.waterfall.seniorHighWaterMark =                                          // :120, old mark overwritten
                recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
            remaining = 0;
        } else {
            uint256 gap = state.waterfall.seniorHighWaterMark > state.waterfall.seniorPrincipal  // :124, restore branch
                ? state.waterfall.seniorHighWaterMark - state.waterfall.seniorPrincipal
                : 0;
            ...
        }
    }
    if (remaining > 0) {
        state.unassignedAssets += remaining;
    }
}
```

The recapitalization was recorded while Senior held principal, and the owner sized it as a restoration of `H - S`. Because the branch is chosen after the mark, the same funding is applied as a fresh bootstrap. `recapitalizationTargetAssets` is that `H - S`, so the high-water mark becomes `H - S` and the record of `H` is gone. Nothing revisits the decision when the mark moves back.

#### The rebound is credited to Junior

On the next reconciliation the unrealized gain reverses into LP surplus. `distributeRevenue` restores Senior only to the new mark and credits the rest to Junior principal:

```solidity
// packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:147
if (remaining > 0 && nextState.seniorPrincipal < nextState.seniorHighWaterMark) {
    uint256 deficit = nextState.seniorHighWaterMark - nextState.seniorPrincipal;
    uint256 restore = remaining < deficit ? remaining : deficit;
    nextState.seniorPrincipal += restore;
    remaining -= restore;
}
...
nextState.juniorPrincipal += remaining;                                               // :158
```

The part of the recapitalization that was not settleable on the first pass meets a closed gap and becomes `unassignedAssets` (`HousePoolPendingPreviewLib.sol:134-136`).

#### The invariant that breaks

The waterfall library states the high-water mark rule: "losses do not reduce it, while senior withdrawals scale it and senior gains above it increase it" (`HousePoolWaterfallAccountingLib.sol:9-10`). `ACCOUNTING_SPEC.md:337` repeats it as reconciliation step 3: "Charge a shortfall to Junior first, then Senior. Losses do not reduce the Senior high-water mark." `ACCOUNTING_SPEC.md:634` defines a `Recapitalization` inflow as "recapitalization intended to restore waterfall claimants".

In this path an unrealized loss at step 3 lowers the high-water mark through step 5, and a recapitalization intended to restore Senior permanently reduces Senior's entitlement. The repository's own test of a recapitalization recorded while Senior holds impaired principal expects the opposite: `test_FreshPendingSeniorMutation_RestoresHwmWithoutDebtQueue` asserts "Recapitalization should still restore senior principal to the HWM". The proof of concept's control branch reproduces that expected outcome. Its adverse branch differs only by the mark at the first settlement.

#### The band is reachable in a solvent pool that is not degraded

The pre-recapitalization LP equity has to be at or below zero at the settlement mark while the recapitalized pool stays solvent. The queued recapitalization is pool cash for every risk gate:

- `HousePool.totalAssets()` is `min(raw balance, accountedAssets)` (`HousePool.sol:426-429`). A `CashArrived` recapitalization raises `accountedAssets` immediately (`HousePool.sol:640-642`).
- Open admission prices depth from `totalAssets()` (`OrderRouterExecutionSidecar.sol:933`, `:953`; `CfdEngine.sol:1138`).
- Fresh trader payouts, the degraded-mode latch and `clearDegradedMode` all use `_availableCashForFreshPoolPayouts`, which is `pool.totalAssets()` minus trader claims (`CfdEngine.sol:865-871`, `:849`, `:1351-1369`, `:655-667`).

Positions opened after the recapitalization are therefore admitted and backed by it. An ordinary adverse mark on them can take pre-recapitalization equity below zero without degrading the pool. The proof of concept asserts `engine.degradedMode() == false` after the open.

#### Root cause

1. `HousePool._reconcile` stores the mark-driven waterfall plan (`src/HousePool.sol:1584-1585`) before it applies the pending recapitalization (`src/HousePool.sol:1598`).
2. `_applyPendingClaimantBucketsLive` rebuilds its state from storage (`src/HousePool.sol:1768`), so the recapitalization sees the principals as the current mark left them, not as they were when the recapitalization was recorded or when the reconciliation began.
3. `applyClaimantRecapitalizationIntent` takes the bootstrap branch when both of those principals are zero and overwrites `seniorHighWaterMark` with the recapitalization target (`src/libraries/HousePoolPendingPreviewLib.sol:118-122`).
4. The pre-reconcile principal and high-water mark are not consulted. `allowRevenueContinuation` is captured from them at `:1567`, but only for revenue routing.
5. On the rebound, `distributeRevenue` restores Senior to the overwritten mark and credits the remainder to Junior principal (`src/libraries/HousePoolWaterfallAccountingLib.sol:147-158`).

#### The commit step

The overwrite becomes storage only when a reconciliation commits. The permissionless route with open positions is `OrderRouter.settleLpEpoch` (`src/OrderRouter.sol:479-484`), which validates a signed Pyth update, refreshes the Engine mark and settles in one call. `HousePool._settleLpEpoch` reconciles first (`src/HousePool.sol:835`) and reverts with `HousePool__NoLpEpochProgress` when no epoch advances (`src/HousePool.sol:1183-1184`), which also rolls back the reconciliation.

A matured Junior redemption with zero value supplies that progress. The funding phase refunds the zero-value head and counts it (`src/HousePool.sol:960-986`), and the refund returns the shares to the holder for a later redemption. LP epochs are hourly (`HousePool.sol:178`) and requests close five minutes before a boundary (`TrancheVault.sol:39`). When no other LP request is pending, a Junior holder's own request therefore sets which hourly boundary is the first committed settlement after the recapitalization. Junior shares are transferable ERC20 subject to the deposit cooldown (`TrancheVault.sol:314-334`), so the holder can be anyone who holds or acquires wiped Junior shares.

### Attack scenario

Starting state: Junior principal is zero with live Junior shares, Senior principal `S` is below its high-water mark `H` after realized trading losses, and the pool is not degraded.

0. The owner calls `CfdEngine.recapitalizePool(H - S)`, the documented recovery for the recorded gap. The amount is queued.
1. A trader opens a position through the public Router path. Admission counts the queued recapitalization as depth. In the proof of concept this is a fully margined 579,100-notional Short at 1.00, 39% of pool cash and inside the 40% skew cap.
2. A Junior holder requests redemption of its zero-value shares before the epoch cutoff, and the request matures at the next hourly boundary.
3. The basket moves until the position's unrealized gain is at least `S`. Any address calls `OrderRouter.settleLpEpoch` with a valid signed update. The plan zeroes both principals, the recapitalization bootstraps Senior with `seniorHighWaterMark = H - S`, and the Junior request is refunded as progress.
4. The basket returns and the trader closes at its entry price, with no realized gain.
5. At the next settlement, Senior is restored only to `H - S`, the remainder of the recovered equity is credited to Junior principal, and the unapplied part of the recapitalization goes to `unassignedAssets`.
6. The Junior holder claims its refunded shares, waits the ordinary cooldown, and redeems Junior principal in USDG. A Senior LP who exits receives `H - S` worth of value instead of `H`.

Steps 1 to 6 are public calls. The Junior holder needs no role and takes no price risk. It only holds shares that are worth nothing in the control branch and files an ordinary request. The trader in step 1 can be any unrelated account. A participant who opens that position itself can hedge it off-chain, because the transfer depends on the mark seen at the settlement, not on any realized trading gain.

The program's privileged-role exclusions cover attacks that require a malicious, compromised or out-of-procedure trusted role. Neither applies here. The program page lists the recapitalization among the owner's immediate trusted actions and places "a legitimate, documented action by a trusted role that triggers a bug harming users" in scope. The owner funds exactly the gap read from chain and gains nothing, and the defect is in how the reconciliation applies that funding.

### Relation to documented limitations and known issues

The program treats `packages/perps/SECURITY.md` Known Limitations and the listed audit reports as known issues, and accepts a report on a documented root cause only where the fix at `a82d4fef` is incomplete or introduces a new bug.

**The bootstrap branch and its tests.** The NatSpec of `applyClaimantRecapitalizationIntent` documents the bootstrap: "With senior owners and zero claimed principal, all `amount` bootstraps senior principal and sets the high-water mark to `max(recapitalizationTargetAssets, amount)`" (`HousePoolPendingPreviewLib.sol:103-107`). The repository pins zero-principal resets in five tests. Each one commits a realized wipe first, and none lets a mark produce the zero inside the reconciliation that applies a recapitalization recorded while Senior held principal:

| Test | How principal reaches zero | Reset source |
|---|---|---|
| `TrancheRecoveryBehavior.t.sol:24-55` `test_PostWipeoutRequiresExplicitRecapForHwmReset` | pool tokens burned, wipe reconciled | recapitalization recorded after the wipe |
| `HousePoolSeededSetup.t.sol:58-114` `test_MaxDepositAndMaxMint_ReopenForPendingSeniorRecapAfterWipeout` | pool tokens burned, wipe reconciled ("Senior principal should be wiped out before recap") | recapitalization recorded after the wipe |
| `HousePoolBootstrap.t.sol:301-324` `test_RecordRecapitalizationInflow_SeedsSeniorWhenNoPrincipalButSeedSharesExist` | pool tokens burned, wipe reconciled | recapitalization recorded after the wipe |
| `HousePoolSnapshotParity.t.sol:155-169` `test_PartialMixedInflowsPreserveFullIntentAndSettleResidualLater` | `_wipePrincipal()` burns pool tokens and reconciles | recapitalization recorded after the wipe |
| `HousePoolWaterfall.t.sol:420-445` `test_AssignUnassignedAssets_ResetsSeniorHwmWhenSeniorIsEmptyButJuniorStillExists` | storage write | owner assignment of unassigned assets |

The one repository test that records a recapitalization while Senior holds impaired principal, `HousePoolWaterfall.t.sol:389-418`, asserts restoration to the high-water mark. The documented bootstrap reset is a policy for a tranche whose wipe has been committed. The reported path applies it to a tranche that was not wiped when the owner funded its restoration.

**The reconciliation order.** `ACCOUNTING_SPEC.md:335-340` lists pending-bucket routing as step 5, after the loss and surplus steps, and explains the order: "Reserving them in step 2 prevents counting the same inflow twice." That reason concerns double counting. The same list states, at step 3, that losses do not reduce the Senior high-water mark, and that is the rule this path breaks.

**Known Limitations.** The LP accounting limitations note that "exact terminal NAV is only as current as its authenticated mark" (`SECURITY.md:1382-1383`). That entry concerns a NAV that follows the mark in both directions. The Senior high-water mark is defined as the value that does not follow a loss. No entry in the LP accounting, LP claim and allocation, or rounding limitations (`SECURITY.md:1380-1433`) covers recapitalization branch selection or a loss of high-water-mark entitlement beyond the one-atom rounding bound at `:1417-1419`.

**Published audit findings:**

| Published issue | Its root cause and impact | This report | Why its fix does not stop this path |
|---|---|---|---|
| SC Audit Studio SC-L1, permissionless zero-value senior withdraw can persist overstated tranche principal | Pending claimant buckets were applied after mark-to-market without a cap, overstating stored principal | Pending buckets are capped. The defect is which recapitalization branch runs after the mark, and the high-water mark it writes | The cap by settleable equity (`_settleablePendingClaimantBuckets`) only sets how much of the recapitalization is applied in the first pass. The branch is chosen from the zeroed principals either way |
| Cecuro MED-12, mixed recovery inflows can strand LP revenue in ownerless pool assets | A stale `allowRevenueContinuation` decision parked realized LP revenue as ownerless. The response's worked case (40,000 recapitalization and 15,000 revenue giving Senior 40,000 and Junior 15,000) starts from principal that a wipe had already zeroed, and it states "Recapitalization remains senior-first" | Senior held principal when the recapitalization was recorded. Its high-water mark is overwritten and Senior value moves to Junior | The fix recomputes revenue continuation (`HousePool.sol:1577-1578`, `HousePoolPendingPreviewLib.sol:50-52`) and does not touch the recapitalization branch or the high-water mark |
| Cecuro MED-24, LP can delay other redeemers with zero-payout refunds | Refund epochs consume bounded settlement work and delay later redeemers | One zero-value refund is the epoch progress that commits the reconciliation. No redeemer is delayed | The accepted work-cap limitation does not concern waterfall state |

SC-L1 is the closest published entry. It also arose because `_reconcile` applies the pending claimant buckets after the mark-to-market step, and the auditor recommended making that settlement "part of the same liability-adjusted reconcile kernel instead of applying it afterward". The response fixed the amount instead: "Pending recap/revenue is now capped to what's actually settleable after MTM/deferred liabilities" (`audits/plether-pre-audit-report.pdf`, SC-L1). If that entry is read as covering the ordering itself, the fix at `a82d4fef` is incomplete for this consequence. The capped recapitalization is still routed by a waterfall state that the current mark has just rewritten, and that produces the outcome measured here.

A separate report by the same researcher (#98615) covers a temporary mark clamping an existing `unassignedAssets` balance (`HousePoolReconcilePlanLib.sol:73-74`), with the rebound credited to Junior. This report concerns a different write (`seniorHighWaterMark` in the recapitalization bootstrap branch, `HousePoolPendingPreviewLib.sol:118-122`), a different victim (Senior LPs lose their own principal; no pre-existing ownerless balance is involved) and a different fix. Restoring a clamped reserve leaves the bootstrap overwrite unchanged, and keeping the high-water mark leaves the reserve clamp unchanged.

**Companion report.** A separate report submitted alongside this one covers the favorable direction at the same settlement. There, a temporary trader loss restores Senior before the recapitalization is applied, the recapitalization's restore branch finds no gap, and the recapitalization is stranded in `unassignedAssets` (`HousePoolPendingPreviewLib.sol:123-136`). The two defects sit in different branches of `applyClaimantRecapitalizationIntent`, have different victims and outcomes, and have disjoint narrow fixes (see the fix check in the proof of concept).

## Impact Details

Measured on the release configuration with the pool never degraded. Each row is one same-state comparison: the first settlement after the recapitalization runs at 1.00 in the control and at the temporary mark in the other branch. Everything else is identical, including the trader's close at 1.00 and the later exits.

| Measure (USDG) | 5.08% temporary mark | 22.23% temporary mark |
|---|---:|---:|
| Senior principal before the recapitalization (`S`) | 26,768.271103 | 117,044.503056 |
| Senior high-water mark before the recapitalization (`H`) | 1,485,028.127781 | 1,485,028.127760 |
| Owner recapitalization (`H - S`) | 1,458,259.856678 | 1,367,983.624704 |
| High-water mark after the first settlement, control | 1,485,028.127781 | 1,485,028.127760 |
| High-water mark after the first settlement, temporary mark | **1,458,259.856678** | **1,367,983.624704** |
| Senior LP full exit, control | 1,484,130.499438 | 1,484,130.499417 |
| Senior LP full exit, temporary mark | 1,458,258.874022 | 1,367,982.702881 |
| **Senior LP loss** | **25,871.625416** | **116,147.796536** |
| Junior holder redemption, control | 0 | 0 |
| **Junior holder redemption, temporary mark** | **25,452.816597** | **106,701.447771** |
| Left in `unassignedAssets`, control | 1,129.126121 | 1,129.126121 |
| Left in `unassignedAssets`, temporary mark | 1,547.706688 | 10,575.322677 |

In both runs the Senior LP loses about the Senior principal held before the recapitalization, the Junior holder receives most of it, and the remainder sits in `unassignedAssets`. In the control, 1,129.126121 USDG ends in `unassignedAssets` because the first settlement credits the open's VPI charge as LP revenue before it applies the recapitalization. That routing does not depend on the mark. The loss grows with `S`, and so does the move needed to trigger it.

### Severity

The mapped impact is **Critical: direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield**. The value taken is the Senior LPs' principal at rest. In the same-state control the recapitalization restores Senior to its high-water mark. In the adverse branch Senior is restored only to the recapitalization amount, so the Senior LP loses about the principal it held before the recapitalization. A Junior holder, whose redemption was worth zero in both branches before the settlement, redeems that value as USDG. The beneficiary is identified and the loss is realized in settlement token.

The High tier "theft of unclaimed yield" is not claimed, because the transferred amount is Senior principal recorded before the recapitalization, not accrued yield. The Medium griefing tier is not claimed either, because a party other than the victim receives the value and withdraws it.

The program assesses feasibility separately from impact. The conditions that bound this path are stated here so they can be weighed:

- Junior principal is zero and Senior is impaired, the state that `recapitalizePool` exists to cure.
- The owner recapitalizes, a documented trusted action that the program lists as in scope when it triggers a bug ("Immediately, can ... recapitalize the pool"). In the proof of concept the amount is exactly the recorded gap, read from chain.
- At the first committed settlement after the recapitalization, open positions carry an unrealized trader gain of at least `S`, the Senior principal before the recapitalization, and less than `S` plus the recapitalization. Net exposure is capped by `maxSkewRatio` (40% of pool depth, which includes the recapitalization). The required move is therefore about `S / (0.4 x depth)`, and the loss is about `S`.

No attacker is required for the loss. The same transfer happens when an honest keeper commits the first post-recapitalization settlement at such a mark.

### Mitigation

Choose the recapitalization branch from the claimed principal as it stood when the recapitalization was recorded, or when the reconciliation began, rather than from the state the current mark has just written. Equivalently, do not lower `seniorHighWaterMark` in a reconciliation that began with nonzero claimed principal. Keep the documented bootstrap reset for a tranche whose wipe was committed before the recapitalization, as pinned by `TrancheRecoveryBehavior.t.sol:24-55`.

Regression coverage should record a recapitalization while Senior holds impaired principal, commit the first settlement at an adverse mark that zeroes the remaining principal, return the mark, and assert that Senior is restored to its original high-water mark and that the Junior payout matches the control.

## References

- `packages/perps/src/HousePool.sol:1555-1599` (`_reconcile`: plan stored at `:1584-1585`, pending buckets applied at `:1598`), `:1759-1776` (`_applyPendingClaimantBucketsLive`, state re-read at `:1768`), `:426-429` (`totalAssets`), `:636-648` (`recordClaimantInflow`, `accountedAssets` raised at `:641`), `:794-835` (`settleLpEpoch`, reconcile at `:835`), `:960-986` (zero-value refund counted as progress), `:1183-1184` (`HousePool__NoLpEpochProgress`), `:178` (`LP_EPOCH_DURATION`)
- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:103-137` (`applyClaimantRecapitalizationIntent`; bootstrap branch `:118-122`; restore branch `:123-132`; remainder to unassigned `:134-136`), `:40-57` (`applyPendingClaimantBucketsPreview`)
- `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:9-10` (high-water mark rule), `:140-159` (`distributeRevenue`), `:161-185` (`absorbLoss`, high-water mark "deliberately unchanged")
- `packages/perps/src/libraries/HousePoolReconcilePlanLib.sol:47-106` (`planReconcile`)
- `packages/perps/src/CfdEngine.sol:627-654` (`recapitalizePool`), `:655-667` (`clearDegradedMode`), `:840-871` (fresh payouts and `_availableCashForFreshPoolPayouts`), `:1138` (net pool depth), `:1351-1369` (`_enterDegradedModeIfInsolvent`)
- `packages/perps/src/OrderRouter.sol:479-484` (`settleLpEpoch`), `packages/perps/src/OrderRouterExecutionSidecar.sol:933`, `:953` (open admission depth)
- `packages/perps/src/TrancheVault.sol:39` (request cutoff), `:314-334` (share transfers)
- `packages/perps/src/libraries/HousePoolTrancheGateLib.sol:28-30` (deposits closed while unassigned assets exist)
- `packages/perps/ACCOUNTING_SPEC.md:329-340` (waterfall and reconciliation order), `:634`, `:639-643` (recapitalization intent and routing)
- `packages/perps/SECURITY.md:155-162` (owner-funded recapitalization), `:1380-1420` (LP limitations)
- `packages/perps/test/perps/accounting/waterfall/HousePoolWaterfall.t.sol:389-418` (recapitalization restores impaired Senior to the high-water mark), the five bootstrap tests listed above

## Proof of Concept

## Proof of Concept

The test runs on an unmodified checkout of `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, using the repository's own `BasePerpTest` fixture with the v2.0.0 release risk parameters. One file, added to the frozen commit with no other change. It compiles and passes against the pristine repository. No production source is touched, `test/perps/BasePerpTest.sol` is used unmodified, and no `vm.mockCall` appears anywhere.

Notes on the fixture:

- `_riskParams` sets the release risk parameters from `DEPLOYMENT.md` (`vpiFactor` 0.01e18, `maxSkewRatio` 0.4e18, `maintMarginBps` 10, `initMarginBps` 20, `fadMarginBps` 300, `baseCarryBps` 500), and `setUp` applies the release Senior limits (40,000,000 USDG, 8000 bps).
- `MockSettlementToken` stands in for USDG and `MockPyth` for Pyth Network. The production `PletherOracle` reads `MockPyth` through its real staleness, confidence and divergence checks.
- Every value-moving call after setup goes through a public entry point: `router.commitOrder`, `router.executeOrder`, `router.updateMarkPrice`, `router.settleLpEpoch`, `pool.settleLpEpoch`, `seniorVault.requestRedeem`, `juniorVault.requestRedeem`, `juniorVault.claimRedeemRefund`. The single owner call, `engine.recapitalizePool`, is the documented recovery, and its amount is read from `pool.seniorHighWaterMark() - pool.seniorPrincipal()`.
- The direct `pool.settleLpEpoch(0, 0)` in `_impairSenior` runs at zero open interest, which `HousePool.sol:794-796` documents as permissionless from the cached mark. Every settlement with an open position goes through `router.settleLpEpoch`.
- `_impairSenior` only builds the starting state that `recapitalizePool` exists to cure. Each round closes two opposite positions at 0.01 and 1.99 to realize a trader gain larger than Junior principal in a few rounds. These prices are not meant as basket moves. The reported mechanism starts after that state is committed, and from the recapitalization on the test uses only 1.00 and the computed temporary mark.
- The basket price convention is the protocol's own: `CfdMath.calculateExactPnl` makes a `Side.SHORT` position profitable as the basket price rises (`src/CfdMath.sol:61-66`). The trader opens a Short at 1.00, is marked in profit at the temporary mark, and closes at 1.00.
- The temporary mark is computed, not tuned: it is the lowest price at which the Short's unrealized gain is 110% of `S`.

Each test carries its own same-state control through `vm.snapshotState`. After the recapitalization, the open and the Junior request, the test runs the first settlement once at 1.00 and once at the temporary mark, with identical steps afterwards.

### Step-by-step

1. Setup: the release risk parameters and Senior limits are applied, both tranches are seeded with 1 USDG, and a Junior LP and a Senior LP deposit 500,000 and 1,500,000 USDG.
2. `_impairSenior` builds the starting state. Realized two-sided shocks wipe Junior principal and impair Senior, and an ordinary epoch with a small Senior redemption persists the impairment. The test asserts Junior principal 0, Senior principal above 0 and below its high-water mark, and no degraded mode.
3. The owner calls `engine.recapitalizePool` for exactly `seniorHighWaterMark() - seniorPrincipal()`.
4. A trader opens a fully margined Short sized at 39% of pool cash, inside the 40% skew cap. The test asserts the pool is not degraded.
5. The wiped Junior holder requests redemption of all its shares, and the request matures.
6. The temporary mark is computed as the lowest price at which the Short's unrealized gain is 110% of the Senior principal held before the recapitalization.
7. The state is snapshotted. The control branch runs the first `router.settleLpEpoch` at 1.00, and the other branch runs it at the temporary mark. Both branches then do the same things: the trader closes at 1.00, the Senior LP's full redemption is funded by a settlement at 1.00, and the Junior holder claims its refunded shares, waits the cooldown and has its redemption funded at 1.00.
8. Assertions: the control keeps the high-water mark, the temporary-mark branch replaces it with the recapitalization amount, the Senior LP exits with less USDG, and the Junior holder redeems more.

### Setup

Save the file below as `packages/perps/test/bounty/r126/RecapBootstrapHwmResetPoC.t.sol` and run from `packages/perps`:

```sh
forge test --match-path 'test/bounty/r126/RecapBootstrapHwmResetPoC.t.sol' -vv
```

### Test file

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console2} from "forge-std/console2.sol";

/// Routine owner recapitalization of an impaired Senior tranche, followed by the first committed LP settlement at a
/// temporarily adverse mark. HousePool._reconcile stores the mark-driven waterfall (both principals zero) before it
/// applies the queued recapitalization, so the recapitalization takes the bootstrap branch and replaces the Senior
/// high-water mark with the recapitalization amount. After the mark returns, the Senior LP exits with less USDG and the
/// Junior holder redeems the difference.
contract RecapBootstrapHwmResetPoC is BasePerpTest {

    address internal constant LONG_SHOCK = address(0xA11CE);
    address internal constant SHORT_SHOCK = address(0xB0B);
    address internal constant JUNIOR_LP = address(0xBEEF);
    address internal constant SENIOR_LP = address(0x5E1);
    address internal constant KEEPER = address(0x5EEDED);
    address internal constant TRADER = address(0xA77ACC);

    uint256 internal juniorRequestId;

    struct Outcome {
        uint256 hwmAfterFirstSettle;
        uint256 seniorPaid;
        uint256 juniorPaid;
        uint256 unassignedFinal;
    }

    // Release risk parameters (DEPLOYMENT.md): VPI 0.01, max skew 40%, maintenance 10 bps, initial 20 bps.
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory p) {
        p = super._riskParams();
        p.vpiFactor = 0.01e18;
        p.maxSkewRatio = 0.4e18;
        p.maintMarginBps = 10;
        p.initMarginBps = 20;
        p.fadMarginBps = 300;
        p.baseCarryBps = 500;
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

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
    }

    // Fully margined: 1.2 USDG per token at a 1.0 mark.
    function _marginFor(
        uint256 size
    ) internal pure returns (uint256) {
        return size * 12 / 1e18 * 1e5;
    }

    function _execute(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        vm.prank(account);
        uint64 id = router.commitOrder(side, size, margin, 0, isClose);
        bytes[] memory data = _mockPythUpdateData(price);
        uint256 fee = pletherOracle.getOrderExecutionFee(data);
        vm.deal(KEEPER, fee);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder{value: fee}(id, data);
        require(uint256(r.status) == uint256(OrderV3Types.LifecycleStatus.Executed), "order not executed");
    }

    function _settleAt(
        uint256 price
    ) internal {
        bytes[] memory data = _mockPythUpdateData(price);
        uint256 fee = pletherOracle.getUpdateFee(data);
        vm.deal(KEEPER, fee);
        vm.prank(KEEPER);
        router.settleLpEpoch{value: fee}(data);
    }

    function _lots(
        uint256 cashAssets,
        uint256 shareBps
    ) internal pure returns (uint256 size) {
        size = cashAssets * 1e12 * shareBps / 10_000;
        size -= size % 1e20;
    }

    /// Realized two-sided shocks inside the 40% skew cap wipe Junior and leave Senior impaired.
    function _impairSenior(
        uint256 rounds
    ) internal {
        _fundJunior(JUNIOR_LP, 500_000e6);
        _fundSenior(SENIOR_LP, 1_500_000e6);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        _fundTrader(LONG_SHOCK, 4_000_000e6);
        _fundTrader(SHORT_SHOCK, 4_000_000e6);
        for (uint256 i; i < rounds; ++i) {
            uint256 size = i == 0 ? 750_000e18 : _lots(pool.totalAssets(), 3900);
            _execute(LONG_SHOCK, CfdTypes.Side.LONG, size, _marginFor(size), false, 1e8);
            _execute(SHORT_SHOCK, CfdTypes.Side.SHORT, size, _marginFor(size), false, 1e8);
            _execute(LONG_SHOCK, CfdTypes.Side.LONG, size, 0, true, 0.01e8);
            _execute(SHORT_SHOCK, CfdTypes.Side.SHORT, size, 0, true, 1.99e8);
            bytes[] memory reset = _mockPythUpdateData(1e8);
            uint256 fee = pletherOracle.getUpdateFee(reset);
            vm.deal(KEEPER, fee);
            vm.prank(KEEPER);
            router.updateMarkPrice{value: fee}(reset);
        }
        // Persist the realized impairment through an ordinary epoch (no open interest, direct settlement).
        uint256 shares = seniorVault.maxRequestRedeem(SENIOR_LP) / 100;
        vm.prank(SENIOR_LP);
        uint256 id = seniorVault.requestRedeem(shares, SENIOR_LP, SENIOR_LP);
        vm.warp(seniorVault.depositEpochStart(id) + 1);
        pool.settleLpEpoch(0, 0);
    }

    function _run(
        uint256 firstSettlePrice,
        uint256 traderSize
    ) internal returns (Outcome memory o) {
        // First committed settlement after the recapitalization (any caller; a zero-value Junior refund is progress).
        _settleAt(firstSettlePrice);
        o.hwmAfterFirstSettle = pool.seniorHighWaterMark();

        // The mark returns and the trader closes at the entry price.
        _execute(TRADER, CfdTypes.Side.SHORT, traderSize, 0, true, 1e8);

        // The Senior LP redeems its whole position at the next epoch.
        uint256 seniorShares = seniorVault.maxRequestRedeem(SENIOR_LP);
        vm.prank(SENIOR_LP);
        uint256 seniorId = seniorVault.requestRedeem(seniorShares, SENIOR_LP, SENIOR_LP);
        vm.warp(seniorVault.depositEpochStart(seniorId) + 1);
        _settleAt(1e8);
        o.seniorPaid = seniorVault.claimableRedeemAssets(seniorId, SENIOR_LP);
        o.unassignedFinal = pool.unassignedAssets();

        // The Junior holder takes back its refunded shares and redeems the Junior balance.
        if (juniorVault.refundableRedeemRequest(juniorRequestId, JUNIOR_LP) != 0) {
            vm.prank(JUNIOR_LP);
            juniorVault.claimRedeemRefund(juniorRequestId, JUNIOR_LP, JUNIOR_LP);
        }
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        uint256 juniorShares = juniorVault.maxRequestRedeem(JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        uint256 exitId = juniorVault.requestRedeem(juniorShares, JUNIOR_LP, JUNIOR_LP);
        vm.warp(juniorVault.depositEpochStart(exitId) + 1);
        _settleAt(1e8);
        o.juniorPaid = juniorVault.claimableRedeemAssets(exitId, JUNIOR_LP);
    }

    function _scenario(
        uint256 rounds
    ) internal {
        _impairSenior(rounds);
        uint256 seniorBefore = pool.seniorPrincipal();
        uint256 hwmBefore = pool.seniorHighWaterMark();
        assertEq(pool.juniorPrincipal(), 0, "Junior wiped by realized losses");
        assertGt(seniorBefore, 0, "Senior still holds principal");
        assertGt(hwmBefore, seniorBefore, "Senior impaired");
        assertFalse(engine.degradedMode(), "pool is not degraded");

        // Routine owner recovery (SECURITY.md:155-163): fund exactly the recorded Senior impairment.
        uint256 recap = hwmBefore - seniorBefore;
        settlementToken.mint(address(this), recap);
        settlementToken.approve(address(engine), recap);
        engine.recapitalizePool(recap);

        // An ordinary trader opens a fully margined short inside the 40% skew cap.
        uint256 traderSize = _lots(pool.totalAssets(), 3900);
        _fundTrader(TRADER, 2_000_000e6);
        _execute(TRADER, CfdTypes.Side.SHORT, traderSize, _marginFor(traderSize), false, 1e8);
        assertFalse(engine.degradedMode(), "pool is not degraded");

        // The wiped Junior holder files a redemption; it matures before the next settlement.
        uint256 juniorShares = juniorVault.maxRequestRedeem(JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        juniorRequestId = juniorVault.requestRedeem(juniorShares, JUNIOR_LP, JUNIOR_LP);
        vm.warp(juniorVault.depositEpochStart(juniorRequestId) + 1);

        // Smallest mark at which the short's unrealized gain exceeds the stored Senior principal by 10%.
        uint256 target = seniorBefore + seniorBefore / 10;
        uint256 dip = 1e8 + (target * 1e8 * 1e12 + traderSize - 1) / traderSize;

        uint256 snap = vm.snapshotState();
        Outcome memory control = _run(1e8, traderSize);
        assertTrue(vm.revertToState(snap));
        Outcome memory attack = _run(dip, traderSize);

        console2.log("Senior principal before recap", seniorBefore);
        console2.log("Senior high-water mark before recap", hwmBefore);
        console2.log("recapitalization", recap);
        console2.log("trader short (tokens)", traderSize / 1e18);
        console2.log("temporary mark (1e8 = 1.0)", dip);
        console2.log("HWM after first settlement, neutral mark", control.hwmAfterFirstSettle);
        console2.log("HWM after first settlement, temporary mark", attack.hwmAfterFirstSettle);
        console2.log("Senior LP exit, neutral mark", control.seniorPaid);
        console2.log("Senior LP exit, temporary mark", attack.seniorPaid);
        console2.log("Junior holder paid, neutral mark", control.juniorPaid);
        console2.log("Junior holder paid, temporary mark", attack.juniorPaid);
        console2.log("unassigned after rebound, neutral mark", control.unassignedFinal);
        console2.log("unassigned after rebound, temporary mark", attack.unassignedFinal);

        assertEq(control.hwmAfterFirstSettle, hwmBefore, "neutral mark: recapitalization restores toward the HWM");
        assertEq(attack.hwmAfterFirstSettle, recap, "temporary mark: HWM replaced by the recapitalization amount");
        assertLt(attack.seniorPaid, control.seniorPaid, "Senior LP exits with less USDG");
        assertGt(attack.juniorPaid, control.juniorPaid, "Junior holder redeems the difference");
        console2.log("Senior LP loss", control.seniorPaid - attack.seniorPaid);
    }

    /// About a 5% temporary move: Senior holds about 26.8k before the recapitalization.
    function test_temporaryMarkResetsSeniorHwm_smallMove() public {
        _scenario(3);
    }

    /// About a 22% temporary move: Senior holds about 117k before the recapitalization.
    function test_temporaryMarkResetsSeniorHwm_largeMove() public {
        _scenario(2);
    }

}
```

### Output

```text
Ran 2 tests for test/bounty/r126/RecapBootstrapHwmResetPoC.t.sol:RecapBootstrapHwmResetPoC
[PASS] test_temporaryMarkResetsSeniorHwm_largeMove() (gas: 23452340)
Logs:
  Senior principal before recap 117044503056
  Senior high-water mark before recap 1485028127760
  recapitalization 1367983624704
  trader short (tokens) 579100
  temporary mark (1e8 = 1.0) 122232595
  HWM after first settlement, neutral mark 1485028127760
  HWM after first settlement, temporary mark 1367983624704
  Senior LP exit, neutral mark 1484130499417
  Senior LP exit, temporary mark 1367982702881
  Junior holder paid, neutral mark 0
  Junior holder paid, temporary mark 106701447771
  unassigned after rebound, neutral mark 1129126121
  unassigned after rebound, temporary mark 10575322677
  Senior LP loss 116147796536
[PASS] test_temporaryMarkResetsSeniorHwm_smallMove() (gas: 29935051)
Logs:
  Senior principal before recap 26768271103
  Senior high-water mark before recap 1485028127781
  recapitalization 1458259856678
  trader short (tokens) 579100
  temporary mark (1e8 = 1.0) 105084632
  HWM after first settlement, neutral mark 1485028127781
  HWM after first settlement, temporary mark 1458259856678
  Senior LP exit, neutral mark 1484130499438
  Senior LP exit, temporary mark 1458258874022
  Junior holder paid, neutral mark 0
  Junior holder paid, temporary mark 25452816597
  unassigned after rebound, neutral mark 1129126121
  unassigned after rebound, temporary mark 1547706688
  Senior LP loss 25871625416
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

Amounts are six-decimal USDG atoms and prices are 8-decimal (`1e8` = 1.00).

Reading `test_temporaryMarkResetsSeniorHwm_smallMove`: three realized shock rounds wipe Junior and leave Senior with 26,768.271103 USDG against a high-water mark of 1,485,028.127781. The owner recapitalizes 1,458,259.856678, the exact gap. A trader opens a 579,100-notional Short at 1.00 and the pool stays out of degraded mode. The wiped Junior holder files a redemption and lets it mature.

In the control branch, the first settlement runs at 1.00, the high-water mark stays at 1,485,028.127781, the Senior LP later exits with 1,484,130.499438 USDG and the Junior holder's redemption is worth 0. In the other branch, the first settlement runs at 1.05084632, and the high-water mark becomes 1,458,259.856678, the recapitalization amount. The trader then closes at 1.00 as in the control, the Senior LP exits with 1,458,258.874022, and the Junior holder redeems 25,452.816597. The assertions check the high-water mark in both branches, the lower Senior exit and the higher Junior redemption.

### Fix check

**Root-cause attribution.** The claimed root cause was patched locally and the tests were re-run to confirm they measure that defect. The patch adds five lines to `HousePool._reconcile`. It records `seniorHighWaterMark` and whether claimed principal was nonzero when the reconciliation began, and after `_applyPendingClaimantBucketsLive` it restores the recorded high-water mark if the reconciliation began with claimed principal and the mark was lowered:

```diff
@@ HousePool._reconcile
         bool allowRevenueContinuation = seniorPrincipal + juniorPrincipal != 0;
+        uint256 r126HwmBefore = seniorHighWaterMark;
+        bool r126ClaimedBefore = allowRevenueContinuation;
         HousePoolReconcilePlanLib.ReconcilePlan memory plan = HousePoolReconcilePlanLib.planReconcile(
@@
         _applyPendingClaimantBucketsLive(settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation);
+        if (r126ClaimedBefore && seniorHighWaterMark < r126HwmBefore) {
+            seniorHighWaterMark = r126HwmBefore;
+        }
     }
```

Under that patch:

- both tests fail at `assertEq(attack.hwmAfterFirstSettle, recap, "temporary mark: HWM replaced by the recapitalization amount")`, with `1485028127781 != 1458259856678` and `1485028127760 != 1367983624704`. The high-water mark after the adverse first settlement equals the control value,
- the Junior holder's redemption in the adverse branch falls from 25,452.816597 and 106,701.447771 USDG to 0, the control value,
- the Senior LP's full exit in the adverse branch rises from 1,458,258.874022 and 1,367,982.702881 USDG to 1,485,027.127087 and 1,485,027.127066 USDG. This is slightly above the control exit, because in the control the open's VPI charge is credited as revenue before the recapitalization and routes 1,129.126121 of it to `unassignedAssets`,
- the five repository files that pin zero-principal resets and recapitalization restoration pass unchanged under the patch, 70 tests in total: `TrancheRecoveryBehavior.t.sol` 12, `HousePoolBootstrap.t.sol` 25, `HousePoolSnapshotParity.t.sol` 11, `HousePoolSeededSetup.t.sol` 3, `HousePoolWaterfall.t.sol` 19. The documented reset after a committed wipe is unaffected.

**Separation from the companion report.** Under this patch the companion report's proof of concept, in which a favorable mark leaves the recapitalization stranded in `unassignedAssets`, still passes with identical figures. Under the companion's own patch, which keeps that part of the recapitalization pending, this proof of concept still passes with identical figures.

The patch was reverted before the output above was produced. Every figure in this report comes from the unmodified frozen commit.

**Size.** The two tests use the same procedure with different starting impairments. The required move scales with `S` divided by net marked exposure, and the loss scales with `S`.
