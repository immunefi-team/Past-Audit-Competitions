# \#98365 [SC-High] Permissionless protection-close retry blocks owner limit amendment, freezing a trader's exit

**Submitted on Nov 7th 2026 at 11:50:15 UTC by @DiFors for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98365
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Temporary freezing of funds for at least 1 hour

## Description

Summary
-------
PositionProtectionBook.retryPositionProtectionClose has no caller restriction. Any address can call it whenever a trader's position protection is in the Latched state, which moves the protection back to Triggered. The owner's only repair path, amendLatchedPositionProtectionCloseLimits, works exclusively in Latched with an empty order queue. By re-triggering at each Latched transition, a griefer (or an honest keeper following the documented retry procedure) can prevent the owner from amending unexecutable close limits, while the active protection simultaneously blocks every other exit route. The protected position cannot be closed and stays exposed to adverse price and accruing carry until it is liquidated.

Affected code (frozen commit a82d4fef...ee24fea8)
-------------------------------------------------
- packages/perps/src/PositionProtectionBook.sol
    - retryPositionProtectionClose (permissionless; requires only Latched + empty queue)
    - amendLatchedPositionProtectionCloseLimits (owner-only; requires Latched + empty queue)
- packages/perps/src/router/OrderLiquidationBatchLogic.sol
    - commitProtectionCloseAttempt (queues the retry close reusing the stored limits)
    - enforceProtectionLock path that reverts ordinary commits with OrderRouter__ProtectionActive

Root cause
----------
The protection state machine gives the owner exactly one state in which limits can be repaired (Latched, empty queue) but exposes a permissionless transition out of that state:

1. amendLatchedPositionProtectionCloseLimits requires status == Latched and pendingOrderCounts(owner) == 0.
2. retryPositionProtectionClose requires only status == Latched and pendingOrderCounts == 0, has no owner check, and queues a fresh close attempt reusing the UNCHANGED stored limits, moving the protection to Triggered.
3. If those unchanged limits still reject execution (e.g. ConstraintViolation because maxExecutionNotionalAssets / maxExplicitFeesAssets are below the position's close cost at any price), failure cleanup relatches to Latched, reopening the same race.
4. The owner has no alternative exit while the protection is active:
   - cancelPositionProtection / replacePositionProtection revert OrderRouter__ProtectionNotArmed (only PendingOpen / Armed are cancellable).
   - router.commitOrder(..., isClose=true) reverts OrderRouter__ProtectionActive.

Whoever wins transaction ordering at each Latched transition therefore controls whether the owner can amend. The permissionless retry defeats the owner-only amendment.

Preconditions
-------------
A protection whose stored closeLimits reject the generated close deterministically. This is exactly the situation the amendment feature exists to recover from: close cost (fees + notional) rises relative to the limits the trader originally set, the triggered close fails ConstraintViolation, and the protection latches for a trader-authorized retry with repaired limits. No privileged role, no victim signature, and no capital from the griefer beyond gas.

Impact
------
- The owner cannot repair the close limits unless they batch the repair atomically (see Limitations); a separately submitted amend is exposed to the retry at every cycle.
- The owner cannot exit by any other route (cancel/replace rejected; ordinary close rejected by the protection lock; only addMargin is permitted, which does not close the position).
- The position stays open, exposed to adverse mark movement and accruing carry, until it is liquidated or until market conditions move enough that the stale limits happen to pass. This is a temporary-to-indefinite freeze of the trader's ability to exit a protected position — griefing with no gain to the attacker.
- Each cycle also queues another binding zero-bounty close attempt that keepers must process.

Novelty / not a known issue
---------------------------
The owner amendment path (amendLatchedPositionProtectionCloseLimits) is post-audit remediation code added after the Cecuro audit commit (6d6622f2), addressing MED-9. The MED-9 note states the owner may amend tight limits while Latched; it does not acknowledge that the amendment can be raced out of existence by the permissionless retry. The prior AI scan only noted that Triggered/Latched protections cannot be cancelled; it did not identify the amendment starvation.

Limitations (stated for fair triage)
------------------------------------
Sustained blocking requires winning transaction ordering at each return to Latched. Arbitrum has no public mempool, so this is not classical front-running; a griefer competes by terminalizing the failing attempt and retrying in one transaction as soon as the attempt becomes processable. An owner who batches "terminalize the failed attempt + amend" into a single transaction (smart account or EIP-7702 delegation) avoids the intervening retry. An owner who submits amend as a separate transaction is exposed to the race at every cycle. Independently of any attacker, the protocol's own keeper guidance (CANONICAL_ENTRYPOINTS.md, SECURITY.md) instructs keepers to submit a retry immediately upon observing Latched, which removes the owner's amendment window in normal operation unless the owner batches.

Suggested fix
-------------
Restrict retryPositionProtectionClose to the protection owner, or give the owner a preemption-safe repair: allow amend to apply from Triggered for the owner, or add an owner-only transition back to Armed/cancellable while a protection is stuck.

## Proof of Concept

setup mirrors packages/perps/test/perps/spec/trader/PositionProtection.t.sol (BasePerpTest):
trader ALICE funded, mark = 1e8. Every step below is already demonstrated by a passing test
in that file at the frozen commit; run them with:
    forge test --root packages/perps --match-path test/perps/spec/trader/PositionProtection.t.sol

Step 1 — Create a protection with limits that reject the close.
  ALICE opens a LONG position and calls createPositionProtection with
  closeLimits.maxExecutionNotionalAssets below the position's close notional (any value that
  guarantees a policy failure). Protection becomes Armed.
  (Construction identical to the _createProtectionFor / _params helpers.)

Step 2 — Trigger the protection.
  A keeper triggers at the take-profit price. Status -> Triggered; one zero-bounty child
  close is queued.
  Backed by: _triggerAt + _assertTriggered.

Step 3 — The triggered close fails and relatches.
  The child close is executed; policy rejects it with ConstraintViolation, the position is
  left open, and cleanup relatches the protection to Latched.
  PROVEN BY EXISTING TEST:
    test_ProtectionCloseLimits_ExcessExecutionNotionalFailsWithoutSettlingPosition
    (sets maxExecutionNotionalAssets = expectedNotional - 1; asserts result Failed with
     TerminalReason.ConstraintViolation, position size unchanged, status == Latched).

Step 4 — Owner repair is only possible in Latched + empty queue.
  To fix the limits ALICE must call amendLatchedPositionProtectionCloseLimits, which succeeds
  only while Latched with no pending orders.

Step 5 — A non-owner re-triggers permissionlessly.
  Any address (a griefer, or per the protocol docs an honest keeper) calls
  retryPositionProtectionClose(protectionId). It is permissionless and moves the protection
  Latched -> Triggered, reusing the UNCHANGED limits.
  PROVEN BY EXISTING TEST:
    test_RetryPositionProtectionClose_IsPermissionlessFreshAndAppendsFifoTail
    (CAROL, who is NOT the owner, calls retry; status becomes Triggered).

Step 6 — The owner's amend now reverts.
  ALICE's amendLatchedPositionProtectionCloseLimits reverts OrderRouter__ProtectionNotLatched.
  PROVEN BY EXISTING TEST:
    test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry
    (amend before Latch reverts ProtectionNotLatched; a non-owner amend reverts Unauthorized;
     an amend attempted after a retry reverts ProtectionNotLatched).

Step 7 — The loop.
  The re-queued child fails again on the same limits -> relatch Latched -> step 5 repeats. At every Latched window the permissionless retry races a separately submitted owner amend.

Step 8 — No fallback exit while protection is active.
  - cancelPositionProtection / replacePositionProtection revert OrderRouter__ProtectionNotArmed.
    PROVEN BY: test_TriggeredProtection_RejectsReplacementAndCancellation.
  - router.commitOrder(side, size, 0, 0, true) (ordinary close) reverts OrderRouter__ProtectionActive.
    PROVEN BY: test_ActiveProtection_BlocksDiscretionaryOrdersButAllowsAddMargin.

Result
------
ALICE cannot amend the close limits and cannot close the position by any route. The position
remains open, accruing carry and exposed to adverse price, until liquidation — a freeze of
the trader's exit produced purely by a permissionless state transition. Each composed step is
an existing, passing test; the vulnerability is their composition.
