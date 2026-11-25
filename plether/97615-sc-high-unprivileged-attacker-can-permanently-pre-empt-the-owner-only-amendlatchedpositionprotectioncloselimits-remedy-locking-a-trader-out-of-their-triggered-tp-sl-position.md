# \#97615 [SC-High] Unprivileged attacker can permanently pre-empt the owner-only amendLatchedPositionProtectionCloseLimits remedy, locking a trader out of their triggered TP/SL position

**Submitted on Nov 2nd 2026 at 14:28:50 UTC by @nullvault for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97615
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Temporary freezing of funds for at least 1 hour

## Description

## Summary
When a triggered protection close attempt fails or expires, the protection goes Triggered -> Latched. The trader's only documented remedy for unfillable close ceilings is the owner-only amendLatchedPositionProtectionCloseLimits. It is callable only while the protection is Latched AND the account has zero pending orders. retryPositionProtectionClose is permissionless and immediately moves the protection back to Triggered with a new pending child. Terminating the current attempt is also permissionless: oracle-free after its deadline (executeOrder(id, []), or clearInvalidOrders([id]) which works on non-head ids), or inside the window with public Pyth data so the attempt fails on the trader's own ceiling. An attacker can do both steps in one transaction, every window. At no transaction boundary is the protection Latched with an empty queue, so the owner's amend can never succeed. A triggered protection cannot be cancelled or replaced, and the active protection blocks any ordinary close. The trader cannot exit, reduce, or re-parameterise the position for as long as the attacker keeps paying ~1.13M gas per window, while the position stays exposed to price moves and liquidation.

## Target
Plether perps, commit a82d4fefb02768e5b3bddcc97dee371eee24fea8, packages/perps/src/** — PositionProtectionBook.sol, OrderRouterExecutionSidecar.sol, OrderRecoverySidecar.sol, router/OrderLiquidationBatchLogic.sol

## Relationship to Cecuro MED-9 (known issue) - why this is new
Cecuro MED-9 (keepers can retry protection closes without the trader's financial limits, Acknowledged) was about retries IGNORING the trader's limits. The post-audit remediation (commit 8a2853f, PR #124 eadc5c58) retains the trader's ceilings on every attempt and adds amendLatchedPositionProtectionCloseLimits. This report does NOT re-raise MED-9 — here the limits are enforced correctly. The bug is in the NEW remedy added by the fix: the escape hatch it promises can be made unreachable by any third party. It is post-audit code, absent from every prior report.

## Root cause
- PositionProtectionBook.sol:315-328 amendLatchedPositionProtectionCloseLimits: owner-only; requires status==Latched (:320) AND pendingOrderCounts(owner)==0 (:323).
- PositionProtectionBook.sol:382-405 retryPositionProtectionClose: PERMISSIONLESS; Latched->Triggered with a fresh pending child. No owner-priority window, no cooldown.
- PositionProtectionBook.sol:637-675 handleFailedProtectionAttempt: any failed/expired attempt on an unchanged position relatches (:665-667).
- OrderRouterExecutionSidecar.sol:371-392: anyone can terminate an attempt after executionDeadline as Expired without oracle data.
- OrderRecoverySidecar.sol:67-115 clearInvalidOrders: same oracle-free expiry, for non-head ids too.
- router/OrderLiquidationBatchLogic.sol:657-680 _protectionCloseRequest: every retry copies the same stored ceilings; window = maxExecutionWindowSeconds.
- PositionProtectionBook.sol:408-418 _cancel, :341-346 cancelPositionProtectionAndCommitClose: only from PendingOpen/Armed. Triggered/Latched cannot be cancelled.
- router/OrderLiquidationBatchLogic.sol:313-314 OrderRouter__ProtectionActive: an ordinary close is rejected while protection is active.
The design gives the owner exactly one remedy and gates it on a state a permissionless actor controls; the actor can skip that state atomically.

## Preconditions
1. A trader has a TP/SL protection whose retained close ceilings are unfillable at execution time (the case the remediation explicitly anticipates). Realistic example (PoC Variant C): trader caps maxExecutionNotionalAssets at exactly size x TP price (9,000 USDG in the fixture); TP triggers; price moves back 5% before the post-commit execution tick; a recovered price does not cancel the latched mandate (SECURITY.md:547-552), so every attempt now exceeds the cap with ConstraintViolation.
2. Attacker: no role, no capital, no position. Pyth update data is public (Hermes). Variants A and D need no oracle data at all.

## Attack (one transaction per window)
- Variant A (oracle-free): at executionDeadline+1, executeOrder(attemptId, []) -> Expired, Latched. Then retryPositionProtectionClose(protectionId) -> Triggered.
- Variant B/C (in-window): executeOrder(attemptId, pythData) -> ConstraintViolation -> Latched. Then retry in same tx.
- Variant D (non-head): as A but via clearInvalidOrders([attemptId]).

## Impact
The owner cannot amend (OrderRouter__ProtectionNotLatched), cancel (OrderRouter__ProtectionNotArmed), or close/reduce (OrderRouter__ProtectionActive). The position and its backing margin are stuck for as long as the attacker continues (PoC: 1,417 consecutive windows, >24h), remaining exposed to adverse moves; the attacker can continue until the trader is liquidated. Cost to attacker: measured 1,134,664 gas per expiry+retry cycle, ~one per 61s => ~1.6B gas/day (~0.016-0.032 ETH/day at typical Arbitrum fees).

## Recommended fix (any one)
1. Owner-priority window: store relatchedAt; allow non-owners to retry only after relatchedAt + ownerAmendWindow.
2. Allow the owner to amend while Triggered.
3. Extend cancelPositionProtectionAndCommitClose to Latched/Triggered so the owner can replace the mandate with an ordinary close.
4. Owner-only retryWithLimits(protectionId, limits) combined with (1).


## Proof of Concept

File: packages/perps/test/poc/LatchedAmendGrief.t.sol, extends the repo's PositionProtectionTest fixture. 6/6 PASS (local Foundry, no fork): test_PoC_ExpiryRetry_LocksOwnerOutFor24h (1417 cycles, 86437s), test_PoC_PythFailRetry_LocksOwnerOut (241 cycles), test_PoC_RealisticCeiling_PriceBounce_LocksOwnerOut (241 cycles), test_PoC_ClearInvalidOrdersRetry_LocksOwnerOut (119 cycles), test_PoC_SingleCycleGas (1,134,664 gas/cycle), and control test_Control_RealisticCeiling_NoGriefer_OwnerAmendsAndExits (owner amends and exits when no attacker, proving the lock-out is solely attacker-caused). Run from packages/perps with FOUNDRY_GAS_LIMIT=18446744073709551615 (needed only because the 24h loop exceeds forge's default 2^30 per-test gas budget).
