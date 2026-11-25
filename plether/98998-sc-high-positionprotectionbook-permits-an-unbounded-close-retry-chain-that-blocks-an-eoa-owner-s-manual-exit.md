# \#98998 [SC-High] PositionProtectionBook permits an unbounded close retry chain that blocks an EOA owner's manual exit

**Submitted on Nov 11th 2026 at 11:50:37 UTC by @cgboal for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98998
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

# PositionProtectionBook permits repeated close retries that block an EOA owner's manual exit

## Brief/Intro

Any caller can expire a protection close child and retry the close in the same transaction. The retry creates another child under the owner's account and returns the protection to `Triggered`. An EOA owner cannot atomically expire the child and amend the protection's close limits because amendment requires the owner to be `msg.sender`. A caller who keeps winning this race can prevent the owner from cancelling the protection, changing its limits, or submitting a fresh manual close. The position can still close through a protection child at the stored limits.

## Vulnerability Details

At the scope commit, `retryPositionProtectionClose` accepts any caller when a protection is `Latched` and the owner's pending-order count is zero. It has no retry limit or cooldown. `_recordCloseAttempt` then links a new close child and returns the protection to `Triggered` (`PositionProtectionBook.sol`, lines 382-406 and 564-578).

An expired child can be cleared by ID without being the FIFO head. A caller can therefore put `expireOrder(childId)` and `retryPositionProtectionClose(protectionId)` in one helper-contract transaction. The expiry clears the pending order under the owner's account; the retry immediately creates its replacement under that account. The new child copies the stored close limits, and the default execution window is 60 seconds (`OrderRecoverySidecar.sol`, lines 49-64; `OrderLiquidationBatchLogic.sol`, lines 655-681; `OrderExecutionOrchestrator.sol`, lines 11-14).

The EOA owner has no equivalent atomic amendment. `amendLatchedPositionProtectionCloseLimits` requires `protection.account == msg.sender`, `Latched` status, and no pending orders for that account. A helper contract is not the protection owner, so its amendment reverts. Alice must expire the child and amend in separate transactions, leaving a transaction-ordering opportunity for a caller to retry first. A contract wallet that owns the protection can batch expiry and amendment (`PositionProtectionBook.sol`, lines 315-328).

The other owner actions do not end the cycle. `cancelPositionProtection` requires `PendingOpen` or `Armed`, and `cancelPositionProtectionAndCommitClose` requires `Armed`; the cycle visits `Triggered` and `Latched`. A fresh public `commitOrder` for a manual close reverts while `activePositionProtectionId(account) != 0`. The attached tests exercise these calls in both states (`PositionProtectionBook.sol`, lines 341-367 and 408-426; `OrderLiquidationBatchLogic.sol`, lines 217-239 and 313-315).

## Impact Details

The selected in-scope impact is Medium: “Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol).” A caller can repeatedly restore `Triggered` before an EOA owner amends the close limits. During the chain, the owner cannot cancel the protection or commit a fresh manual close. The caller pays for each retry and has no demonstrated direct profit.

The protection child remains executable at the stored limits. If it executes, the chain ends; if it does not, the owner remains in the position with the old limits while the caller continues to retry. The caller can continue beyond 24 hours if they keep paying for transactions and winning the relevant ordering races, but the attached PoC demonstrates four cycles and a fifth successful retry, not a continuous 24-hour lock. An EOA owner can amend if the caller misses a `Latched` gap, and a contract-wallet owner can batch the counter-move. This report claims Medium griefing rather than the program's High temporary-freezing impact.

## References

Plether competition scope and impacts: https://immunefi.com/audit-competition/plether/scope/

Source at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`: https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src

Secret Gist containing the runnable Foundry test and verification output: https://gist.github.com/Cgboal/eb9d268f8470df913b8d65fa03c4f15a



## Link to Proof of Concept

https://gist.github.com/Cgboal/eb9d268f8470df913b8d65fa03c4f15a

## Proof of Concept

## Proof of Concept

The test runs against Plether commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` in the repository's Foundry suite. Save `ProtectionRetryChain.t.sol` from the secret Gist at `packages/perps/test/perps/spec/trader/ProtectionRetryChain.t.sol`, then run from `packages/perps`:

```sh
forge test --match-test 'test_RetryChain|test_EOAOwner|test_AttackerHelper|test_OwnerCancel|test_OwnerClose' -vv
```

The tests reproduce the following sequence:

1. Alice opens a position, creates a take-profit protection, and a keeper triggers it. The protection enters `Triggered` with a linked close child.
2. After the child's execution deadline, Bob expires it. `test_RetryChain_LocksOwnerExits` repeats expiry and permissionless retry four times, then creates a fifth child. Each retry succeeds and returns the protection to `Triggered`; Alice's amendment reverts while the child is live.
3. `test_AttackerHelper_AtomicExpireAndRetry` calls `expireOrder(childId)` and `retryPositionProtectionClose(protectionId)` from Bob's helper contract in one transaction. It asserts that the new child has a greater ID, is linked to the protection, and leaves it `Triggered`.
4. `test_EOAOwner_HelperContractCannotBatchExpireAmend` tries the corresponding owner escape through Alice's helper contract. The amendment fails because the helper, rather than Alice, is `msg.sender`. The transaction rolls back, leaving the original child pending.
5. `test_OwnerCancel_BlockedInTriggeredAndLatched` shows that Alice's cancellation reverts in both states. `test_OwnerClose_BlockedWhileProtectionActive` submits a fresh manual close through `commitOrder` in each state; both calls revert with `OrderRouter__ProtectionActive`.

The recorded run passed all five tests with no failures; `verification-2026-11-11.txt` in the Gist contains the output. The test demonstrates repeated retries and the blocked owner actions. It does not run a 24-hour chain, though it could be extended. 
