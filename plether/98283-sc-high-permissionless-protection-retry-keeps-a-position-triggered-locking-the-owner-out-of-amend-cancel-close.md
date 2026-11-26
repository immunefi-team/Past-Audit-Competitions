# \#98283 [SC-High] # Permissionless protection retry keeps a position Triggered, locking the owner out of amend/cancel/close

**Submitted on Nov 6th 2026 at 17:50:44 UTC by @xeQted for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98283
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary

Closing a position protection is a two-party state machine: a keeper drives the trigger/retry side, and
the owner drives the amend/cancel side. `retryPositionProtectionClose` has no caller restriction, so
anyone can run the keeper side. When a triggered close fails against the trader's own `closeLimits`, the
protection relatches to `Latched`, and the owner is meant to fix the limits with
`amendLatchedPositionProtectionCloseLimits`. Amend only works in `Latched` with no pending order. A third
party can expire the failed attempt and re-queue a retry in one transaction, so between the attacker's
transactions the protection is always back in `Triggered`. The owner never sees the `Latched` window, and
every exit they have reverts. The position and its margin stay open as long as the attacker keeps paying
gas. The PoC holds a position open for over an hour; a trader without the loop running closes it in one
call.

This is reachable because the MED-9 fix added `closeLimits` plus owner amendment, and the amendment is the
only way out of a too-tight limit. The amendment is deniable.

## Affected assets

| File | Lines | Role |
|---|---|---|
| `packages/perps/src/PositionProtectionBook.sol` | 382-406 | `retryPositionProtectionClose`, permissionless |
| `packages/perps/src/PositionProtectionBook.sol` | 315-328 | `amendLatchedPositionProtectionCloseLimits`, owner-only, `Latched`-gated |
| `packages/perps/src/router/OrderLiquidationBatchLogic.sol` | 313-315 | `OrderRouter__ProtectionActive` gate blocking the owner's own close |

## Root cause

`retryPositionProtectionClose` resolves the protection with `_activeProtection` (by id, no owner check)
and re-commits a close attempt, moving the status back to `Triggered`:

```solidity
function retryPositionProtectionClose(uint64 protectionId) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId);        // no msg.sender check
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    _requireMatchingPosition(protection);
    IPositionProtectionRouterHost router = IPositionProtectionRouterHost(ROUTER);
    if (router.pendingOrderCounts(protection.account) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }
    ...
    linkedOrderId = IPositionProtectionCloseAttemptHost(ROUTER)
        .commitProtectionCloseAttempt(protectionId, protection.account, protection.side, protection.size);
    _recordCloseAttempt(protection, linkedOrderId);                               // -> status Triggered
}
```

The owner's recovery, `amendLatchedPositionProtectionCloseLimits`, resolves with
`_ownedActiveProtection` and requires `Latched` plus no pending order:

```solidity
StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
    revert OrderRouter__ProtectionNotLatched();
}
if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
    revert OrderRouter__PendingOrdersExist();
}
```

A failed or expired attempt relatches to `Latched` through `handleFailedProtectionAttempt`. The attacker
expires the live attempt and calls retry in the same transaction, so the `Latched`-with-no-pending state
exists only inside the attacker's own call and is never observable to the owner between blocks. While the
status is `Triggered` (or `Latched` with the retry's child pending), the owner's three exits all revert:
amend reverts `OrderRouter__ProtectionNotLatched`; `cancelPositionProtection` reverts
`OrderRouter__ProtectionNotArmed` (`_cancel` accepts only `PendingOpen`/`Armed`, lines 408-419); a trader
close through `commitOrder` reverts `OrderRouter__ProtectionActive` because `activePositionProtectionId`
is nonzero (`router/OrderLiquidationBatchLogic.sol:313-315`). The `cancelPositionProtectionAndCommitClose`
manual exit added in the same release requires `Armed` (line 346) and is unreachable here too.

## Link to Proof of Concept

https://gist.github.com/EngineerTunerNorway/1b1c6f0de01fb949e53dda2305129a74

## Proof of Concept

## Proof of concept

`packages/perps/test/perps/exploit/LatchedRetryGrief.t.sol`. Runs against the repository's own perps test
fixtures at the competition commit, no parameter changes.

```
forge test --root packages/perps --match-contract LatchedRetryGriefTest -vv
```

```
[PASS] test_Control_OwnerCanAmendWithoutGriefer() (gas: 3945471)
[PASS] test_Exploit_PermissionlessRetryDeniesOwnerAmendmentAndLocksPosition() (gas: 60311750)
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 88.29ms
```

The exploit test runs 70 cycles. Each cycle warps to the live attempt's `executionDeadline`, then a
`GriefBot` contract calls `executeOrder` (the attempt expires `Failed`, the Book relatches to `Latched`)
and `retryPositionProtectionClose` in one transaction (a new child is queued, status back to `Triggered`).
Every cycle asserts the owner's `amendLatchedPositionProtectionCloseLimits` reverts
`OrderRouter__ProtectionNotLatched`, `cancelPositionProtection` reverts `OrderRouter__ProtectionNotArmed`,
and a close `commitOrder` reverts `OrderRouter__ProtectionActive`. Accumulated locked time is asserted
`>= 1 hours` and the final position size equals the opening size.

The control, `test_Control_OwnerCanAmendWithoutGriefer`, runs the same setup without the bot: the owner
amends after relatch and closes, final size 0. The only changed variable between pass-locked and
pass-exited is the presence of the attacker.

## Impact

Program impact list, Medium: "Temporary freezing of funds for ≥1 hour." The trader's margin is locked
inside a position they cannot close, amend, or cancel for the duration of the loop; the PoC asserts over
one hour. Also Medium: "Griefing (no profit motive, but damage to users/protocol)" — the attacker gains
nothing and pays gas each cycle.

A trader using a smart-contract or EIP-7702 account that executes `executeOrder` and `amend` atomically
can win a given cycle, which makes the lock a race for that trader rather than an unconditional freeze. An
externally-owned account cannot, because only the attacker acts atomically; the PoC models the EOA case.

## Attack scenario

1. A trader holds a position with a take-profit whose `closeLimits` ceiling (for example
   `maxExecutionNotionalAssets`, or a `maxActionChargeAssets` later overtaken by accrued carry) is below
   what the triggered close will cost. This is the condition amend exists to fix.
2. The protection triggers and the close attempt fails the limit, relatching to `Latched`.
3. The attacker, any address, calls `executeOrder` on the expiring attempt and `retryPositionProtectionClose`
   in one transaction. Cost: one transaction plus the Pyth update fee per cycle.
4. The attacker repeats once per execution window. The trader's amend, cancel, and close all revert for as
   long as step 3 continues.

## Recommended fix

Restrict the keeper-driven retry so the owner always regains a `Latched` window. Either gate the caller:

```solidity
function retryPositionProtectionClose(uint64 protectionId) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
    ...
```

or give the owner an exit valid from `Triggered` that force-terminalizes the active child, or require a
minimum interval between relatch and a third-party retry. The owner-gated resolve pattern
(`_ownedActiveProtection`) is already used by amend and cancel in the same contract.

## References

- Reviewed revision: `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (Plether Audit Competition scope).
- Prior finding this derives from, and is distinct from: Cecuro AI audit MED-9 ("Keepers can retry
  protection closes without the trader's financial limits"), Acknowledged. MED-9 is the inverse problem
  (retries ignored the caps); its fix introduced the owner-amendment this report denies.
- Test environment: Foundry (forge), Solidity 0.8.35.
