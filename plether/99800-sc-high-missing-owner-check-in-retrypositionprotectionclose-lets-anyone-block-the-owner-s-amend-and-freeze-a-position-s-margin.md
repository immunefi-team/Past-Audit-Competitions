# \#99800 [SC-High] Missing owner check in retryPositionProtectionClose lets anyone block the owner's amend and freeze a position's margin

**Submitted on Nov 15th 2026 at 10:35:50 UTC by @itsactuallyomar for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99800
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

`PositionProtectionBook.retryPositionProtectionClose` has no owner check, and it needs the same state as the owner's remedy for a failed protected close, `amendLatchedPositionProtectionCloseLimits`: both require the protection to be `Latched` with no pending order, and only the amend is owner-only. When a stop-loss close fails because the market gapped past the trader's own price ceiling, anyone whose transaction lands first can re-queue the same failing close before the owner's amend. The protection is then `Triggered` again, the amend reverts, and cancel, cancel-and-close and an ordinary close are unavailable by design while a protection is triggered. In the PoC, the owner of a 10,000,000 USDG LONG who uses the documented single-call amend cannot amend for 24 hours (57 hours across a weekend), so the position stays open and its margin (974,045.71 USDG at the end of the 24-hour lock) cannot be withdrawn, while the attacker pays USD 2,481.74 of gas per day at 0.02 gwei.

## Vulnerability Details

### The owner's remedy and the retry need the same state

The owner's remedy, `PositionProtectionBook.sol:315-328`:

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

The retry, `PositionProtectionBook.sol:382-406`, callable by anyone:

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
    // ... :396-399 (linked-order check, expected order id)
    linkedOrderId = IPositionProtectionCloseAttemptHost(ROUTER)
        .commitProtectionCloseAttempt(protectionId, protection.account, protection.side, protection.size);
    // ... :402-404 (host-response check)
    _recordCloseAttempt(protection, linkedOrderId);
}
```

`_ownedActiveProtection` reverts `OrderRouter__Unauthorized` for anyone but the protection's account (`:983-991`). `_activeProtection`, which the retry uses, only checks that the protection is the account's active one (`:993-999`).

### The cycle

1. A keeper triggers the protection. The Book queues a close of the whole position (an attempt) whose request copies the trader's stored ceilings, with a zero bounty and the full execution window (`router/OrderLiquidationBatchLogic.sol:670-677`). `_recordCloseAttempt` sets `Triggered` (`PositionProtectionBook.sol:573`).
2. The market is past the ceiling, so executing the attempt fails with `ConstraintViolation`. Inside the executor's call, `settleOrderFromSidecar` (`OrderRouter.sol:292-312`) passes the failed attempt to `handleFailedProtectionAttempt` (`:324-329`), which sets `Latched` because side and size still match (`PositionProtectionBook.sol:662-667`), and `_deleteOrder` decrements `pendingOrderCounts` (`router/OrderRouterBase.sol:122-141`).
3. The protection is now `Latched` with no pending order. That is the precondition of both functions, so whoever transacts first decides. A retry queues a new attempt with the same stored ceilings and sets `Triggered` with one pending order, and the owner's amend that follows reverts `OrderRouter__ProtectionNotLatched` (`:320-322`).
4. The new attempt fails the same way while the market stays past the ceiling, and the cycle repeats. Neither the retry nor `_recordCloseAttempt` (`:564-577`) counts or limits retries.

The attacker can run the cycle in two ways, both in the PoC: **(A)** a bare `retryPositionProtectionClose` sent right after each failed keeper execution, or **(B)** a contract that executes the attempt itself and retries in the same transaction, so that the protection is `Triggered` with one pending order at every transaction boundary.

### The other exits are closed by design once a protection has triggered

- Cancellation accepts only `PendingOpen` or `Armed` (`PositionProtectionBook.sol:414-419`).
- `cancelPositionProtectionAndCommitClose` accepts only `Armed` (`:346-348`).
- An ordinary close through `commitOrder` reverts `OrderRouter__ProtectionActive` while a protection is active (`router/OrderLiquidationBatchLogic.sol:313-315`).

This is documented (`CANONICAL_ENTRYPOINTS.md:142-144`): "Once a trigger is recorded, neither the owner nor a keeper may replace or cancel it." The attacker does not cause these three reverts. They are why the amend is the owner's remedy.

While the position is open, its margin stays in `MarginClearinghouse`: a withdrawal reverts `MarginClearinghouse__InsufficientFreeEquity` unless the remaining equity covers all locked margin (`MarginClearinghouse.sol:467-470`).

### The intended order is amend, then retry

- `IPositionProtectionActions.sol:32`: "Lets the owner change close limits after a failed attempt relatches, before a fresh retry."
- `packages/perps-aa-client/README.md:204-205`: "`buildAmendLatchedProtectionCloseLimitsAction` lets the owner change the close limits of a `Latched` protection before a permissionless retry."
- `CANONICAL_ENTRYPOINTS.md:145-147`: "The owner may amend the close ceilings of a `Latched` protection with `PositionProtectionBook.amendLatchedPositionProtectionCloseLimits(uint64,CloseFinancialLimits)` when the account has no pending Router order. This changes the limits used for a future retry".
- The team's reply to MED-9 ("Keepers can retry protection closes without the trader's financial limits") in `audits/plether-ai-audit-report-2026-10-26.pdf`, the finding whose fix made the trader's ceilings persist across retries: "Only the owner may amend the retained ceilings while protection is Latched; keepers cannot relax them." and "Tight limits may prevent execution until conditions improve or the owner amends them."
- `DEPLOYMENT.md:419-421`, guidance for keeper operators: "A conservative worker retries expiry failures only after confirming the account has no pending orders, queue capacity is available, and valid execution data is obtainable. Investigate planner, constraint, or other persistent failures before retrying; do not continuously queue the same failing attempt." This is practice for honest keepers. The contract does not enforce it, and anyone can call the retry.
- The team test `test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry` (`test/perps/spec/trader/PositionProtection.t.sol:986-996`) runs that order: the owner amends, a third party retries, and the owner's second amend then reverts.

## Impact Details

**Selected impact: Temporary freezing of funds for at least 24 hours.**

**Setup.** The PoC deploys the production contracts of `packages/perps/src` the way the release script does, with release parameters. Junior LPs deposit 11,500,000 USDG and Senior LPs 39,990,000 USDG, and six background traders hold positions. The victim, an EOA, opens a LONG of 10,000,000 USDG notional at 10x (1,000,000 USDG margin, plus 100,000 USDG free collateral). It protects the LONG with a stop-loss at basket 1.0100 and a 5 bp stop-limit: `maxExecutionNotionalAssets` is the notional at 1.0105, set from the trader's close policy as `CANONICAL_ENTRYPOINTS.md:121-125` instructs. In the PoC's authored price path, at a US data release the basket gaps 30 bp, from 1.0095 to 1.0125, in one tick, then trends to 1.0170 without returning under 1.0105. Every attempt fails on the ceiling.

**Results** (unmodified code, Forge 1.5.1, 9 of 9 tests pass). Costs are at 0.02 gwei and USD 1,837.76 per ETH (measured 2026-11-14). The figure in brackets is at the 0.01 gwei floor. The PoC asserts frozen margin / cost >= 100 in A, B and C.

| | A: keeper present, attacker back-runs | B: attacker executes and retries atomically | C: weekend, keeper present |
|---|---|---|---|
| Lock, first failure to the owner's first successful amend | 86,400 s (24 h) | 86,400 s (24 h) | 205,200 s (57 h, Fri 13:30:01 to Sun 22:30:01 UTC) |
| Attacker transactions | 86,400 retries | 86,400 execute + retry | 205,200 retries |
| Owner amends sent after them / reverted `ProtectionNotLatched` | 86,400 / 86,400 | 86,400 / 86,400 | 205,200 / 205,200 |
| Transaction boundaries checked | 259,199 | 172,800, each `Triggered` with one pending order | 615,599 |
| Attacker cost | USD 2,481.74 (1,240.87) | USD 7,475.71 (3,737.85) | USD 5,894.14 (2,947.07) |
| Frozen position margin at the end of the lock | USD 974,045.71 | USD 974,045.71 | USD 972,782.72 |
| Frozen / cost | 392 (784) | 130 (260) | 165 (330) |

- **Withdrawals during the lock** (executed in A, B, C and a 10-minute test): the clearinghouse balance is 1,084,628.540764 USDG and the locked buckets total 984,628.940764 USDG. `withdraw` of the position margin (974,629.042503 USDG) and `withdraw(free + 1)` both revert `MarginClearinghouse__InsufficientFreeEquity`. Only the free balance, 99,999.6 USDG, can be withdrawn.
- **The attacker holds the lock.** When it stops, the owner's very next amend succeeds, the close fills and the owner withdraws its whole balance.
- **Without the attacker (control)**, the owner amends in its next transaction and the close fills 1 s after the first failure, at basket 1.01250016. The owner's custody goes from 1,084,628.540764 USDG at the first failure to 1,026,969.584688 USDG, a loss of 57,658.96 USDG. After the 24 h lock the exit is at 1.01700022 and custody is 981,919.525152 USDG, a loss of 102,709.02 USDG. The extra 45,050.06 USDG is specific to this price path.
- **The weekend.** C holds through the FAD window before the close (1,800 executions), the 48 h FX closure (172,800 keeper executions with no Pyth payload) and the Sunday reopen. Against a keeper that executes in every block while the market is closed, C costs USD 20,784.61 (ratio 46).
- **Busy queue.** Other traders' orders ahead of the attempt do not help the owner. `executeOrderBatch` lets one attacker transaction execute them, then the attempt, then retry: 10 batches, 27 other orders executed, 10 of 10 amends reverted.
- **Keeper cost.** In A the keeper executes every attempt the attacker queues: 86,400 executions failed and relatched, each using up to 918,665 metered gas.
- **B's assumptions.** B's cost includes two assumptions stated in the PoC header: a real six-feed Pyth payload of at most 4,400 bytes, and at most 500,000 extra gas for real Pyth verification. The Pyth fee is priced at MockPyth's 1 wei. Without the verification allowance B uses 1,854,075 gas per transaction. A and C carry no Pyth payload.

**Limits.** The lock lasts only while the market stays beyond the victim's ceiling and the attacker keeps paying. In `test_lockEndsWhenBasketReturnsInsideCeiling`, once the basket returns to 1.0104, inside the 1.0105 ceiling, the keeper's next execution fills the attempt the attacker queued, 600 s after the first failure, and the attacker's next retry reverts `OrderRouter__ProtectionNotFound`.

The attack needs its transactions ordered first: in A and C the attacker's retry must land between the keeper's failed execution and the owner's next transaction, and in B the attacker's execution must come before any other execution of the attempt. Arbitrum One orders transactions by priority fee within 125 ms rounds, so against an owner who pays a priority fee the attacker must out-bid it on every transaction. The PoC sets the order by construction and prices every transaction at the base fee, with no priority fee.

An owner that executes the attempt and amends in one transaction escapes (`test_batchingOwnerWinsOneRace_escapes`: closed 4 s after the first failure). An owner using the documented single-call amend does not, even after winning the execution race, when the attacker's retry lands before its amend (`test_eoaOwnerWinsExecutionRace_attackerBackruns_amendStillReverts`: 10 of 10 amends reverted).

Protections created in the Plether web app sign unlimited ceilings. The victims are traders who set a finite ceiling, as `CANONICAL_ENTRYPOINTS.md:121-125` instructs: "Set the limits from the trader's intended close policy; integrations must not assume an implicit unlimited default."

**Known issues.**
- The known issue "AI scan findings (October 15, 2026)" includes "Protection trigger bounty is paid before a zero-bounty close is executed" (`audits/scan-plether-fi-plether-core-d4541b49.md:542`). Its cause is a trigger caller who collects the trigger bounty and abandons the close, and its end state (a triggered protection cannot be cancelled) is the design rule this report leaves unchanged. Here, the attacker re-queues every failed close to keep the owner's amend out of reach, an entry point added after the scan by the MED-9 fix.
- `SECURITY.md` "Router and keeper limitations" (known issue #6) records that users cannot cancel queued intents and that protection liveness depends on keepers appending retries, which covers missing retries. This report is about hostile retries. The fix below leaves the permissionless retry, the irreversible trigger and the cancellation rules unchanged.

## Recommendation

Let the owner amend while an attempt is live. The amended ceilings then apply from the next attempt:

```diff
--- a/packages/perps/src/PositionProtectionBook.sol
+++ b/packages/perps/src/PositionProtectionBook.sol
@@ -317,12 +317,12 @@
         PositionProtectionTypes.CloseFinancialLimits calldata limits
     ) external nonReentrant {
         StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
-        if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
+        if (
+            protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched
+                && protection.status != PositionProtectionTypes.PositionProtectionStatus.Triggered
+        ) {
             revert OrderRouter__ProtectionNotLatched();
         }
-        if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
-            revert OrderRouter__PendingOrdersExist();
-        }
         _setCloseLimits(protection, limits);
         _emitCloseLimitsUpdated(protection);
     }
```

The owner check stays. A live attempt keeps the ceilings it was committed with, because they are copied into its request at commit (`router/OrderLiquidationBatchLogic.sol:674-677`). "A live child cannot be changed" (`IPositionProtectionActions.sol:33`) stays true.

With the diff applied, in every test that prints it, the owner's first amend lands 0 s after the first failure and the position closes 2 s after it. All seven attack tests fail: six at their lock assertion (for example `A: no owner amend succeeded while the attacker was active: 2 != 0`), and the busy-queue test with `OrderRouter__ProtectionNotFound()`, because the amended attempt fills inside the attacker's batch. The control and the batching-owner tests pass.

In the team's `test/perps/spec/trader/PositionProtection.t.sol`, 104 of 104 tests pass on unmodified code and 103 of 104 under the fix. The one failure is `test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry`, at its expectation that the amend reverts while a retried attempt is live (`:994-996`), which is the behaviour the fix changes. That expectation would be updated, together with `CANONICAL_ENTRYPOINTS.md:145-147` ("when the account has no pending Router order"), `IPositionProtectionActions.sol:32` and `packages/perps-aa-client/README.md:204-205`.

## Proof of Concept: steps, tests, expected output and notes

### Steps

1. Junior LPs deposit 11,500,000 USDG and Senior LPs 39,990,000 USDG, and six background traders open positions, all through the public entry points.
2. Mon 2026-11-09 13:00 UTC: the victim, an EOA, opens a LONG of 10,000,000 USDG notional at 10x (1,000,000 USDG margin, plus 100,000 USDG free collateral) at basket 1.0060. It creates a stop-loss at 1.0100 with `maxExecutionNotionalAssets` set to the notional at 1.0105 (a 5 bp stop-limit).
3. Tue 13:30:00 UTC: the basket gaps 30 bp, from 1.0095 to 1.0125, and a keeper triggers the stop. The basket then trends to 1.0170 without returning under 1.0105, so every close attempt fails with `ConstraintViolation` and relatches.
4. The attacker runs the cycle. In A and C a keeper executes each attempt at the earliest moment the code allows, and the attacker sends `retryPositionProtectionClose` in the next block. In B the attacker's contract executes each attempt itself and retries in the same transaction.
5. After every attacker transaction, the owner sends `amendLatchedPositionProtectionCloseLimits`, and it reverts `OrderRouter__ProtectionNotLatched`. The protection's status and the account's pending-order count are asserted at every transaction boundary.
6. At the end of the lock, the test executes the owner's withdrawals and asserts frozen margin / attacker cost >= 100. It then stops the attacker: the owner's next amend succeeds, the close fills and the owner withdraws its whole balance.

Timing follows the code. Outside a frozen market, an order executes only in a block after its commit block, on a price published after its commit time (`OrderRouterExecutionSidecar.sol:447-452`, `PletherOracle.sol:610-614`). With four 250 ms blocks per second, one cycle takes one second.

### What each test proves

- `test_A_keeperPresent_attackerBackruns_24h`: with an honest keeper executing every attempt, the attacker's bare retry after each failure keeps the owner's amend reverting for 24 h (86,400 of 86,400). The margin stays locked, and the owner gets out only after the attacker stops.
- `test_B_atomic_noWindow_24h`: when the attacker executes and retries in one transaction, the amend's precondition holds at none of the 172,800 transaction boundaries over 24 h.
- `test_C_weekend_keeperPresent_attackerBackruns_57h`: as A, from Friday 13:30:01 UTC through the FX closure and the Sunday reopen, 57 h.
- `test_fundsFrozen_withdrawalsDuringLock`: during a lock, withdrawing the position margin, or one unit more than the free balance, reverts `MarginClearinghouse__InsufficientFreeEquity`. Only the free balance pays out.
- `test_control_noAttacker_ownerAmendsAndExits`: without the attacker, the owner amends in its next transaction and the close fills 1 s after the first failure.
- `test_eoaOwnerWinsExecutionRace_attackerBackruns_amendStillReverts`: an EOA owner that executes its own attempt first still cannot amend when the attacker's retry lands before its next transaction, an order the test sets by construction (10 of 10).
- `test_busyQueue_attackerBatchesThroughOtherOrders`: other traders' orders ahead of the attempt do not stop the attacker. `executeOrderBatch` executes them, then the attempt, then retries.
- `test_lockEndsWhenBasketReturnsInsideCeiling`: the bound. When the basket returns inside the ceiling, the keeper fills the attempt the attacker queued, and the attacker's next retry reverts.
- `test_batchingOwnerWinsOneRace_escapes`: the limitation. An owner that executes, amends and retries in one transaction escapes.

### Output (unmodified code, Forge 1.5.1; selected lines, verbatim)

```text
Ran 3 tests for test/perps/poc/ProtectionRetryLock.t.sol:ProtectionRetryLockAtomicTest
[PASS] test_B_atomic_noWindow_24h() (gas: 3780042)
Logs:
  B: owner amends that succeeded while the attacker was active 0
  B: clearinghouse balance (6 dec) 1084628540764
  B: position margin (6 dec) 974629042503
  B: total locked buckets (6 dec) 984628940764
  B: free balance above the locked buckets (6 dec) 99999600000
  B: withdraw(position margin) reverted MarginClearinghouse__InsufficientFreeEquity
  B: withdraw(free + 1) reverted; withdraw(free) paid out (6 dec) 99999600000
  B: attacker execute + retry transactions 86400
  B: owner amends sent after them / reverted ProtectionNotLatched 86400 86400
  B: transaction boundaries checked, each Triggered with one pending order 172800
  B: attacker cost, US cents at 0.02 gwei 747571
  B: frozen position margin, US cents 97404571
  B: frozen / cost at 0.02 gwei (integer) 130
  B: owner's first successful amend, seconds after the first failure 86400

[PASS] test_busyQueue_attackerBatchesThroughOtherOrders() (gas: 56922236)
Logs:
  BUSY QUEUE: owner amends that succeeded while the attacker was active 0
  BUSY QUEUE: attacker batch transactions / other traders' orders they executed 10 27
  BUSY QUEUE: owner amends that reverted ProtectionNotLatched 10

[PASS] test_eoaOwnerWinsExecutionRace_attackerBackruns_amendStillReverts() (gas: 11194797)
Logs:
  EOA RACE: owner amends that succeeded while the attacker was active 0
  EOA RACE: execution races won by the owner / amends that reverted 10 10

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 138.39s (138.35s CPU time)

Ran 1 test for test/perps/poc/ProtectionRetryLock.t.sol:ProtectionRetryLockBasketReturnTest
[PASS] test_lockEndsWhenBasketReturnsInsideCeiling() (gas: 717593921)
Logs:
  RETURN: owner amends that succeeded while the attacker was active 0
  RETURN: attacker back-runs / owner amends that reverted ProtectionNotLatched 600 600
  RETURN: the attacker's next retry reverted OrderRouter__ProtectionNotFound
  RETURN: basket from Tue 13:40:01 UTC (8 dec) 101040000
  RETURN: seconds from the first failure to the close 600
  RETURN: fill price of the close (8 dec) 101040415

Ran 1 test for test/perps/poc/ProtectionRetryLock.t.sol:ProtectionRetryLockBatchingOwnerTest
[PASS] test_batchingOwnerWinsOneRace_escapes() (gas: 5698643)
Logs:
  BATCHING OWNER: attacker cycles before the owner's batch 3
  BATCHING OWNER: closed, seconds after the first failure 4

Ran 3 tests for test/perps/poc/ProtectionRetryLock.t.sol:ProtectionRetryLockKeeperPresentTest
[PASS] test_A_keeperPresent_attackerBackruns_24h() (gas: 3860704)
Logs:
  A: owner amends that succeeded while the attacker was active 0
  BY DESIGN for any triggered protection: cancel, cancel-and-close, close order each revert (3 of 3)
  A: clearinghouse balance (6 dec) 1084628540764
  A: position margin (6 dec) 974629042503
  A: total locked buckets (6 dec) 984628940764
  A: free balance above the locked buckets (6 dec) 99999600000
  A: withdraw(position margin) reverted MarginClearinghouse__InsufficientFreeEquity
  A: withdraw(free + 1) reverted; withdraw(free) paid out (6 dec) 99999600000
  A: attacker back-run retries 86400
  A: owner amends sent after them / reverted ProtectionNotLatched 86400 86400
  A: keeper executions that failed and relatched 86400
  A: transaction boundaries checked 259199
  A: metered execution gas, keeper execution, max (the keeper's cost) 918665
  A: attacker cost, US cents at 0.02 gwei 248174
  A: attacker cost, US cents at 0.01 gwei 124087
  A: frozen position margin, US cents 97404571
  A: frozen / cost at 0.02 gwei (integer) 392
  A: owner's first successful amend, seconds after the first failure 86400
  A: exit basket after the lock (8 dec) 101700022
  A: owner's USDG custody when the lock started / after the exit (6 dec) 1084628540764 981919525152

[PASS] test_control_noAttacker_ownerAmendsAndExits() (gas: 1838361)
Logs:
  CONTROL: seconds from the first failure to the fill 1
  CONTROL: exit basket (8 dec) 101250016
  CONTROL: after the exit the owner withdrew its whole clearinghouse balance (6 dec) 1026969584688
  CONTROL: owner's USDG custody when the lock started / after the exit (6 dec) 1084628540764 1026969584688

[PASS] test_fundsFrozen_withdrawalsDuringLock() (gas: 719982359)
Logs:
  FUNDS: owner amends that succeeded while the attacker was active 0
  FUNDS: clearinghouse balance (6 dec) 1084628540764
  FUNDS: position margin (6 dec) 974629042503
  FUNDS: total locked buckets (6 dec) 984628940764
  FUNDS: free balance above the locked buckets (6 dec) 99999600000
  FUNDS: withdraw(position margin) reverted MarginClearinghouse__InsufficientFreeEquity
  FUNDS: withdraw(free + 1) reverted; withdraw(free) paid out (6 dec) 99999600000
  FUNDS: owner's first successful amend, seconds after the first failure 600

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 144.89s (144.86s CPU time)

Ran 1 test for test/perps/poc/ProtectionRetryLock.t.sol:ProtectionRetryLockWeekendTest
[PASS] test_C_weekend_keeperPresent_attackerBackruns_57h() (gas: 3811165)
Logs:
  C: owner amends that succeeded while the attacker was active 0
  C: clearinghouse balance (6 dec) 1084628540764
  C: position margin (6 dec) 974629042503
  C: total locked buckets (6 dec) 984628940764
  C: free balance above the locked buckets (6 dec) 99999600000
  C: withdraw(position margin) reverted MarginClearinghouse__InsufficientFreeEquity
  C: withdraw(free + 1) reverted; withdraw(free) paid out (6 dec) 99999600000
  C: attacker back-run retries 205200
  C: owner amends sent after them / reverted ProtectionNotLatched 205200 205200
  C: keeper executions that failed and relatched 205200
  C: of which in the frozen market (no Pyth payload, Engine mark replay) 172800
  C: lock executions by regime: live Friday / FAD 28798 1800
  C: lock executions by regime: closed / after the reopen 172800 1801
  C: transaction boundaries checked 615599
  C: attacker cost, US cents at 0.02 gwei 589414
  C: attacker cost, US cents at 0.01 gwei 294707
  C: frozen position margin, US cents 97278272
  C: frozen / cost at 0.02 gwei (integer) 165
  C, keeper in every block while frozen: attacker cost, US cents at 0.02 gwei 2078461
  C, keeper in every block while frozen: frozen / cost at 0.02 gwei (integer) 46
  C: owner's first successful amend, seconds after the first failure 205200
  C: seconds from the first failure (Fri 13:30:01 UTC) to the owner's first successful amend 205200

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 298.06s (298.00s CPU time)

Ran 5 test suites in 582.40s (582.25s CPU time): 9 tests passed, 0 failed, 0 skipped (9 total tests)
```

### Result with the recommended fix applied

```bash
git apply fix.diff   # fix.diff = the diff in the Recommendation section, saved at the repository root
forge test --root packages/perps --match-path test/perps/poc/ProtectionRetryLock.t.sol -vv --threads 1
```

The owner's first amend lands 0 s after the first failure and the position closes 2 s after it. All seven attack tests fail, and the control and the batching-owner tests pass (selected lines, verbatim):

```text
[FAIL: B: no owner amend succeeded while the attacker was active: 2 != 0] test_B_atomic_noWindow_24h() (gas: 20283)
  B: owner amends that succeeded while the attacker was active 2
  B: first successful owner amend, seconds after the first failure 0
  B: position closed, seconds after the first failure 2
  B: attacker transactions until then 3
[FAIL: OrderRouter__ProtectionNotFound()] test_busyQueue_attackerBatchesThroughOtherOrders() (gas: 20525893)
[FAIL: race: no owner amend succeeded: 2 != 0] test_eoaOwnerWinsExecutionRace_attackerBackruns_amendStillReverts() (gas: 4519805)
[FAIL: RETURN: no owner amend succeeded while the attacker was active: 2 != 0] test_lockEndsWhenBasketReturnsInsideCeiling() (gas: 3961967)
[PASS] test_batchingOwnerWinsOneRace_escapes() (gas: 5697458)
[FAIL: A: no owner amend succeeded while the attacker was active: 2 != 0] test_A_keeperPresent_attackerBackruns_24h() (gas: 20112)
  A: owner amends that succeeded while the attacker was active 2
  A: first successful owner amend, seconds after the first failure 0
  A: position closed, seconds after the first failure 2
  A: attacker transactions until then 2
[PASS] test_control_noAttacker_ownerAmendsAndExits() (gas: 1837176)
[FAIL: FUNDS: no owner amend succeeded while the attacker was active: 2 != 0] test_fundsFrozen_withdrawalsDuringLock() (gas: 3965998)
[FAIL: C: no owner amend succeeded while the attacker was active: 2 != 0] test_C_weekend_keeperPresent_attackerBackruns_57h() (gas: 19826)

Ran 5 test suites in 261.82ms (260.41ms CPU time): 2 tests passed, 7 failed, 0 skipped (9 total tests)
```

### Notes on the PoC file: fixture, scenario, cost model, cheat codes and assumptions

```text
=====================================================================================================================
Anyone can re-queue a failed protection close before the owner's amend can land: the owner's only remedy for a
gapped stop-limit is kept unreachable, and the position margin stays frozen, for as long as the attacker repeats it.
=====================================================================================================================

RUN (from the repository root, Forge 1.5.1; peak memory: A or B alone ~1.2 GB, C alone ~2.2 GB, the file ~3.1 GB):
  forge test --root packages/perps --match-path test/perps/poc/ProtectionRetryLock.t.sol -vv --threads 1
  one test: add --match-test <name>.  Wall time on 8 cores: A and B about 3 minutes each, C about 6, the rest < 1 s.
  Forge 1.5.1 is the version the repository's CI pins (.github/workflows/ci.yml:64). Every test contract sets
  isolate = false by inline config (forge-config), the Forge 1.5.1 default, because Forge 1.8.3 defaults to
  isolation: each call from the test contract then runs as its own transaction (21,000 gas and cold access each),
  so the 600-cycle tests exceed Forge's 2^30 test gas limit and the gas samples inside a chunk change.

THE BUG
  PositionProtectionBook.retryPositionProtectionClose (src/PositionProtectionBook.sol:382-406) has no owner check
  (it loads the protection with _activeProtection, :385, :993-999) and needs exactly the state the owner's only
  remedy needs: amendLatchedPositionProtectionCloseLimits (:315-328) is owner-only and requires Latched (:320-322)
  and zero pending orders (:323-325). A failed close attempt relatches (:662-667) and its order is deleted, with
  pendingOrderCounts decremented, inside the executor's call (src/OrderRouter.sol:292-329,
  src/router/OrderRouterBase.sol:122-141). A retry sent right after every failure therefore leaves the protection
  Triggered with one pending order at every boundary the owner can reach, and the amend reverts
  OrderRouter__ProtectionNotLatched. The new attempt copies the same stored ceilings
  (src/router/OrderLiquidationBatchLogic.sol:657-678), so it fails again while the market stays past them.

WHAT THE FIXTURE DEPLOYS (ProtectionRetryLockFixture)
  The production contracts of packages/perps/src, created from their compiled artifacts with deployCode, in the
  order and with the wiring, roles and release parameters of script/DeployPerpsArbitrumSepolia.s.sol:206-427, every
  call from `deployer` (the release broadcaster, owner of every Ownable): MarginClearinghouse; CfdEngine (CAP 2.00,
  the release risk parameters :434-447, frozen-close spread 50 bps); TerminalNavBookV2; CfdEnginePlanner,
  CfdEngineSettlementSidecar, CfdEngineAdmin; HousePoolRedemptionMathSidecar; the release HousePool (Senior cap
  40,000,000 USDG and 80 % share); Senior and Junior TrancheVaults; CfdEngineAccountLens; CfdEngineLens; the
  release PletherOracle (adverse-confidence multiplier 2,500 bps; the six production FX feed ids, weights, base
  prices and inversions); CfdOrderPolicyEvaluator; CfdClosePreview; OrderRouterExecutionSidecar; OrderLifecycleBook
  and OrderRouterLiquidationBatchSidecar bound to the predicted Router address; the release OrderRouter (minimum
  open notional 1,000 USDG; its constructor creates the PositionProtectionBook and the RouterAdmin); then the
  script's set-once wiring; PerpsPublicLens; SettlementMonitorLens; EmergencyPauseCoordinator as pauser of the
  HousePool and the RouterAdmin. The three release wrappers and the mock USDG are verbatim copies of the script's
  (:68-131, :731-748), because the script lives outside packages/perps. Then the release steps of
  script/BootstrapPerpsArbitrumSepolia.s.sol: guardian (:449), a 1 USDG seed per tranche, Junior first
  (:472-485), the first mark through OrderRouter.updateMarkPrice (:534), activateTrading (:541). Every other
  release parameter is a code default: those packages/perps/DEPLOYMENT.md:128-156 lists (execution fee 0.04 %,
  trigger bounty 0.20 USDG, close bounty zero), and two that the table does not list, the 60 s execution window
  (src/router/OrderExecutionOrchestrator.sol:11) and the 15 s oracle settlement window (src/PletherOracle.sol:87),
  which _bootMarket asserts at runtime (router.maxExecutionWindowSeconds(), oracle.orderSettlementWindow()).
  Mocked, both outside the scope, nothing else: Pyth (the team's MockPyth, @plether/test-utils, fee 1 wei per
  update; every payload is an explicit signed-update stand-in for the six feeds with real exponents and a 0.0025 %
  confidence) and USDG (the script's mintable six-decimal MockUSDG).

MARKET AND SCENARIO
  Release-scale pool: Junior LPs deposit 11,500,000 USDG (one LP epoch), then Senior LPs 39,990,000 (the next):
  Senior just under its 40,000,000 cap and 77.7 % of the pool (limit 80 %). Six background traders (LONG 2.0M,
  1.0M, 0.25M; SHORT 1.5M, 0.8M, 0.1M notional). Every deposit, open and epoch goes through the public entry points
  and is executed by a keeper. Mon 2026-11-09 13:00 UTC the victim, an EOA, opens LONG 10,000,000 USDG notional at
  10x (1,000,000 margin, plus 100,000 free collateral) at basket 1.0060 (a LONG loses as the basket rises,
  CfdTypes.sol:14) and protects it: stop-loss at 1.0100, close ceiling maxExecutionNotionalAssets = the notional at
  1.0105, a 5 bp stop-limit set from the trader's close policy as CANONICAL_ENTRYPOINTS.md:121-125 asks. The
  position is 19 % of pool depth; the skew cap is 40 %.
  Weekday path: the basket drifts to 1.0095 by Tue 13:00 UTC (+0.35 %, hourly steps <= 5 bp). Tue 13:30:00 UTC, a
  US data release, it gaps 30 bp to 1.0125 in one tick; a keeper triggers the stop; then a trending day to 1.0170
  (+0.45 %, hourly steps <= 8 bp, low 1.0122), never back under 1.0105. Every attempt fails ConstraintViolation on
  the ceiling and relatches.
  Weekend path: the same drift over four days and the same gap on Friday 13:30 UTC (1.0125-1.0136 through the
  afternoon); FAD from 21:30 UTC; the FX closure Fri 22:00 - Sun 22:00 UTC (17:00 New York, MarketCalendarLib.sol
  :38-49), during which the oracle replays the Engine mark and no Pyth payload exists or is needed
  (PletherOracle.sol:496-506, :582-593, :729-742); the Sunday 22:00 UTC reopen gaps +8 bp to 1.0140.

TIMING MODEL (from the code)
  A delayed order executes only in a later block than its commit (OrderRouterExecutionSidecar.sol:447-449) and on a
  Pyth update published in (commitTime, min(commitTime + 15 s, block.timestamp)] (PletherOracle.sol:610-614; the
  MEV boundary at OrderRouterExecutionSidecar.sol:450-452). With 4 L2 blocks per second (Arbitrum One ~250 ms), the
  earliest execution of an attempt committed in second c is the first block of second c + 1. Per cycle: the
  keeper (or, in B, the attacker) executes in block 0 of the second, the attacker's transaction is in block 1 and
  the owner's in block 2. While the oracle is frozen both checks are skipped (:447, :450); there the model keeps
  the reference keeper's 1 s poll (plether-app KEEPER_POLL_SECONDS = 1 while orders are pending), and test C also
  prints the cost against a keeper that executes in every block.

TESTS (all on unmodified src; the numbers each one prints are labelled)
  ProtectionRetryLockKeeperPresentTest - setUp ends right after the keeper's first, failed, execution
    test_A_keeperPresent_attackerBackruns_24h    HEADLINE A, 24 h = 86,400 cycles. The keeper executes every attempt
      at the earliest moment the code allows; each execution fails and relatches; the attacker back-runs every
      relatch with a bare retry in the next block; after every attacker transaction the owner sends the documented
      remedy, the amend (CANONICAL_ENTRYPOINTS.md:145-148), and it reverts ProtectionNotLatched: 86,400 of 86,400.
      The protection state is asserted at every transaction boundary. At the end of the lock the owner's
      withdrawals are executed, frozen margin / attacker cost >= 100 is asserted, then the attacker stops: the
      owner's very next amend succeeds, the close fills and the whole balance is withdrawn.
    test_control_noAttacker_ownerAmendsAndExits  CONTROL: without the attacker the same failure is amended in the
      owner's next transaction, a third party retries and the close fills one second later.
    test_fundsFrozen_withdrawalsDuringLock       10 minutes into a lock: withdraw(position margin) reverts
      MarginClearinghouse__InsufficientFreeEquity (MarginClearinghouse.sol:466-471), one unit above the free
      balance reverts the same way, exactly the free balance pays out; after the attacker stops and the position
      closes, the whole balance is withdrawable.
  ProtectionRetryLockAtomicTest - setUp ends right after the trigger (attempt 1 live)
    test_B_atomic_noWindow_24h                   HEADLINE B, 24 h = 86,400 cycles. The attacker executes every
      attempt itself at the earliest moment and retries in the same transaction. After EVERY transaction the
      protection is Triggered with exactly one pending order (asserted 172,800 times): the amend's precondition
      holds at no transaction boundary, and the owner's amend after every attacker transaction reverts.
    test_eoaOwnerWinsExecutionRace_attackerBackruns_amendStillReverts  An EOA owner that wins the execution race
      (it executes its own attempt first; it fails and relatches) sends the amend as its next transaction; the
      attacker's back-run lands in between, so the amend reverts. 10 races.
    test_busyQueue_attackerBatchesThroughOtherOrders  Other traders' orders ahead of the attempt: a single
      executeOrder of the attempt reverts OrderRouter__OrderNotQueueHead (for the owner too), but executeOrderBatch
      lets one attacker transaction execute the orders ahead, the attempt, and then retry. 10 cycles.
  ProtectionRetryLockBatchingOwnerTest - the victim is a smart account
    test_batchingOwnerWinsOneRace_escapes        LIMITATION, stated honestly: an owner that wins ONE execution race
      and batches execute + amend + retry in one transaction escapes.
  ProtectionRetryLockWeekendTest - setUp ends right after the keeper's first, failed, execution (Fri 13:30:01 UTC)
    test_C_weekend_keeperPresent_attackerBackruns_57h  WEEKEND, Fri 13:30:01 -> Sun 22:30:01 UTC = 57 h: as A,
      through FAD, the 48 h closure (keeper executions with no Pyth payload) and the reopen.
  ProtectionRetryLockBasketReturnTest - setUp as ProtectionRetryLockKeeperPresentTest
    test_lockEndsWhenBasketReturnsInsideCeiling  HONEST BOUND: the lock of the funds test for 10 minutes (600
      back-runs, 600 of 600 amends revert), then from Tue 13:40:01 UTC the basket is at 1.0104, 1 bp inside the
      1.0105 ceiling (the nearest 1 bp level; the fill adds the adverse confidence shift, PletherOracle.sol:905-923,
      well under 1 bp here), while the attacker keeps back-running. The keeper's next execution fills the attempt
      the attacker queued, within the ceiling; the protection ends Executed, the attacker's next retry reverts
      OrderRouter__ProtectionNotFound, and the owner withdraws its whole balance.

HOW THE LONG RUNS ARE EXECUTED
  A, B and C run their lock before the test function, as committed one-hour transactions: beforeTestSetup returns
  24 (A, B) or 57 (C) calls of runChunk(), and Forge executes and commits each one in order after setUp. This
  bounds memory and makes the first call of every chunk, an attacker transaction, a fresh EVM transaction against
  committed state (cold accounts and slots, committed original values), as on chain. The test function then checks
  the end state. Inside a chunk, step() is one transaction of one actor.

COST MODEL (inputs measured 2026-11-14 18:12 UTC, read-only)
  Per attacker transaction: the maximum metered execution gas observed (every chunk's first attacker transaction,
  plus the first 50 and 50 in the middle of the lock) + 21,000 intrinsic + calldata gas (16 per non-zero byte, 4 per
  zero byte) + the L1 data component from NodeInterface.gasEstimateL1Component on Arbitrum One (1,359 gas for a
  36-byte call, 17,939 for 2,200 bytes, 34,866 for 4,400 bytes), priced at 0.02 gwei (the measured base fee; the
  0.01 gwei floor is printed too) and USD 1,837.76 per ETH (CoinGecko). Attacker transactions that carry a Pyth
  payload (B) also take two ASSUMPTIONS: the real six-feed signed payload is at most 4,400 bytes (priced as non-zero
  bytes) and real Pyth verification costs at most 500,000 gas more than MockPyth; B also prints its per-transaction
  gas without it. Frozen amount = the victim's position margin at the end of the lock, after carry is realised
  (1 USDG = 1 USD). The keeper's own gas is printed but is not the attacker's cost.

CHEAT CODES AND WHY
  vm.prank / vm.startPrank: each actor's own calls (deployer, LPs, traders, keeper, attacker, owner).
  vm.deal: the keeper's, attacker's and owners' own ETH for Pyth fees. MockUSDG.mint: the actors' own capital.
  vm.warp / vm.roll: the calendar and 4 blocks per second. vm.pauseGasMetering / vm.resumeGasMetering: only inside
  runChunk, around the cycles that are not gas samples (a metered one-hour chunk would exceed the per-transaction
  gas limit). deployCode, vm.computeCreateAddress, vm.getNonce: the deployment, as the script does.
  makeAddr / vm.label: names. vm.getBlockTimestamp / vm.getBlockNumber: reads (test code never reads block.timestamp
  under via-IR). vm.recordLogs / vm.getRecordedLogs: only in test_lockEndsWhenBasketReturnsInsideCeiling, to read
  the fill price the Router emits in OrderExecuted (src/OrderRouter.sol:307); read-only. Not used: vm.store,
  vm.mockCall, vm.etch, deployCodeTo, patched copies, vm.chainId. The chain id is Forge's default 31337, on which
  ExecutionBlockNumberLib reads block.number; here block.number plays the role that ArbSys.arbBlockNumber (the L2
  block number) plays on Arbitrum (ExecutionBlockNumberLib.sol:18-23).

ASSUMPTIONS
  Prices: the paths above are authored (the programme allows mocked Pyth updates). The lock lasts while the basket
  stays beyond the victim's ceiling; test_lockEndsWhenBasketReturnsInsideCeiling shows it ending when it returns.
  Pyth: publishes every second while FX is open (each update's previous update is one second older) and nothing
  while it is closed; the first update after the Sunday reopen follows the last Friday update. MockPyth charges a
  1 wei update fee; the real per-update fee and access to Hermes are not in the cost model.
  Ordering: A and C assume the attacker's retry lands after the keeper's failed execution and before the owner's
  next transaction; B assumes the attacker's execute + retry lands before any other execution of the attempt (if a
  keeper wins it, A's back-run applies). Blocks are 250 ms slots. Arbitrum One orders transactions by priority fee
  within 125 ms rounds, so this holds while the attacker's priority fee is above the owner's. An owner that
  executes the attempt and amends in one transaction (test_batchingOwnerWinsOneRace_escapes), or sends the two
  back to back with a higher priority fee than the attacker's, escapes. An owner that uses the single-call amend
  with a lower priority fee does not.
  Keeper: executes each attempt in the first block of the earliest second the code allows; once per second while
  the oracle is frozen. Gas and prices: as in COST MODEL.
=====================================================================================================================
The 12 imports forge-lint reports as unused are deployCode artifacts: importing them makes --match-path compile them.
```

## References

- `retryPositionProtectionClose`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L406
- `amendLatchedPositionProtectionCloseLimits`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328
- `_ownedActiveProtection` / `_activeProtection`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L983-L999
- `_recordCloseAttempt` (sets `Triggered`): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L564-L577
- Relatch after a failed attempt: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L637-L667
- Cancellation and cancel-and-close status checks: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L414-L419 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L346-L348
- Failed-attempt settlement in the Router: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L292-L329
- `_deleteOrder`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderRouterBase.sol#L122-L141
- `ProtectionActive` check and the attempt's copied ceilings: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L670-L677
- Withdrawal check: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L464-L471
- NatSpec of the amend: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IPositionProtectionActions.sol#L32-L33
- `CANONICAL_ENTRYPOINTS.md`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md?plain=1#L121-L148
- `DEPLOYMENT.md` keeper guidance: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md?plain=1#L417-L421
- SDK README: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps-aa-client/README.md?plain=1#L204-L205
- `SECURITY.md` "Router and keeper limitations": https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1343-L1378
- Team test: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/trader/PositionProtection.t.sol#L986-L996
- Scan item: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md?plain=1#L542-L585
- Audit report with MED-9: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf
- Arbitrum One transaction ordering (Priority Gas Auctions): https://docs.arbitrum.io/how-arbitrum-works/timeboost


## Proof of Concept

## Proof of Concept

Steps, the test-by-test results and the expected output are at the end of the Description. In short: after a gap past the victim's own stop-limit ceiling, its protected close fails; the attacker re-queues it after every failed execution (A, C) or executes and re-queues it in one transaction (B), so every owner amend reverts for 24 h (57 h over a weekend) and withdrawals of the position margin revert.

Run in a clone of https://github.com/immunefi-team/audit-comp-plether at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (with `git submodule update --init --recursive`), with the file below saved as `packages/perps/test/perps/poc/ProtectionRetryLock.t.sol`:

```bash
forge test --root packages/perps --match-path test/perps/poc/ProtectionRetryLock.t.sol -vv --threads 1
```

All 9 tests pass: about 11 minutes and 3 GB of memory on Forge 1.5.1 (the version CI pins), about 17.5 minutes on Forge 1.8.3.

```solidity
// SPDX-License-Identifier: AGPL-3.0
// PoC: retryPositionProtectionClose has no owner check, so anyone can keep a failed protected close re-queued and the
// owner's amend reverting. Audited commit a82d4fefb02768e5b3bddcc97dee371eee24fea8. Save as
// packages/perps/test/perps/poc/ProtectionRetryLock.t.sol and run (Forge 1.5.1, the version CI pins, or 1.8.3):
//   forge test --root packages/perps --match-path test/perps/poc/ProtectionRetryLock.t.sol -vv --threads 1
// Production contracts are deployed from their artifacts as script/DeployPerpsArbitrumSepolia.s.sol does, with the
// release parameters. Mocks: MockPyth and MockUSDG only; no vm.store, vm.mockCall, vm.etch or deployCodeTo. The
// fixture, cheat codes, cost model and assumptions are listed in the report's description.
pragma solidity 0.8.35;
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {CfdClosePreview} from "@plether/perps/CfdClosePreview.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdEngineAccountLens} from "@plether/perps/CfdEngineAccountLens.sol";
import {CfdEngineAdmin} from "@plether/perps/CfdEngineAdmin.sol";
import {CfdEngineLens} from "@plether/perps/CfdEngineLens.sol";
import {CfdEnginePlanner} from "@plether/perps/CfdEnginePlanner.sol";
import {CfdEngineSettlementSidecar} from "@plether/perps/CfdEngineSettlementSidecar.sol";
import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {EmergencyPauseCoordinator} from "@plether/perps/EmergencyPauseCoordinator.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {HousePoolRedemptionMathSidecar} from "@plether/perps/HousePoolRedemptionMathSidecar.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderRouterAdmin} from "@plether/perps/OrderRouterAdmin.sol";
import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PerpsPublicLens} from "@plether/perps/PerpsPublicLens.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {SettlementMonitorLens} from "@plether/perps/SettlementMonitorLens.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MarketCalendarLib} from "@plether/perps/libraries/MarketCalendarLib.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {Test, Vm, console2} from "forge-std/Test.sol";
contract ArbitrumSepoliaReleaseHousePool is HousePool {
	constructor(address settlementToken, address engine, address redemptionMathSidecar) HousePool(settlementToken, engine, redemptionMathSidecar) {
		poolConfig.maxSeniorExposureAssets = 40_000_000e6;
		poolConfig.maxSeniorShareBps = 8000;
		_validatePoolConfig(poolConfig);
	}
}
contract ArbitrumSepoliaReleaseOracle is PletherOracle {
	constructor(address engine, address housePool, address pyth, bytes32[] memory feedIds, uint256[] memory quantities, uint256[] memory basePrices, bool[] memory inversions
	) PletherOracle(engine, housePool, pyth, feedIds, quantities, basePrices, inversions) {
		adverseConfidenceMultiplierBps = 2500;
	}
}
contract ArbitrumSepoliaReleaseRouter is OrderRouter {
	constructor(address engine, address engineLens, address housePool, address pletherOracle_, address keeperSidecar, address policyEvaluator, address executionSidecar, address lifecycleBook)
		OrderRouter(engine, engineLens, housePool, pletherOracle_, keeperSidecar, policyEvaluator, executionSidecar, lifecycleBook)
	{
		minOpenNotionalAssets = 1000e6;
	}
}
contract MockUSDG is ERC20 {
	constructor() ERC20("Mock USDG", "USDG") {}
	function decimals() public pure override returns (uint8) {
		return 6;
	}
	function mint(address to, uint256 amount) external {
		_mint(to, amount);
	}
}
contract RetryAttacker {
	address internal immutable OPERATOR;
	constructor() {
		OPERATOR = msg.sender;
	}
	function executeAndRetry(OrderRouter router, address book, uint64 attemptId, bytes[] calldata pythUpdateData, uint64 protectionId
	) external payable returns (uint8 status, uint8 reason, uint64 next) {
		require(msg.sender == OPERATOR, "operator");
		try router.executeOrder{value: msg.value}(attemptId, pythUpdateData) returns (OrderV3Types.ExecutionResult memory r) {
			status = uint8(r.status);
			reason = uint8(r.terminalReason);
		} catch {}
		try IPositionProtectionActions(book).retryPositionProtectionClose(protectionId) returns (uint64 id) {
			next = id;
		} catch {}
	}
	function batchAndRetry(OrderRouter router, address book, uint64 maxOrderId, bytes[] calldata pythUpdateData, uint64 protectionId) external payable returns (uint32 terminalCount, uint64 next) {
		require(msg.sender == OPERATOR, "operator");
		terminalCount = router.executeOrderBatch{value: msg.value}(maxOrderId, pythUpdateData).terminalCount;
		next = IPositionProtectionActions(book).retryPositionProtectionClose(protectionId);
	}
	receive() external payable {}
}
contract BatchingOwnerAccount {
	address internal immutable OWNER;
	constructor() {
		OWNER = msg.sender;
	}
	function exec(address target, bytes calldata data) external returns (bytes memory ret) {
		require(msg.sender == OWNER, "owner");
		bool ok;
		(ok, ret) = target.call(data);
		if (!ok) {
			assembly ("memory-safe") {
				revert(add(ret, 32), mload(ret))
			}
		}
	}
	function executeAmendRetry(OrderRouter router, address book, uint64 attemptId, bytes[] calldata pythUpdateData, uint64 protectionId, PositionProtectionTypes.CloseFinancialLimits calldata limits
	) external payable returns (uint64 next) {
		require(msg.sender == OWNER, "owner");
		router.executeOrder{value: msg.value}(attemptId, pythUpdateData);
		IPositionProtectionActions(book).amendLatchedPositionProtectionCloseLimits(protectionId, limits);
		next = IPositionProtectionActions(book).retryPositionProtectionClose(protectionId);
	}
	receive() external payable {}
}
abstract contract ProtectionRetryLockFixture is Test {
	uint32 internal constant CAP_PRICE = 2e8;
	uint256 internal constant FROZEN_CLOSE_SPREAD_BPS = 50;
	uint256 internal constant JUNIOR_MAINTENANCE_FEE_APR_BPS = 0;
	uint256 internal constant RELEASE_SEED_ASSETS = 1e6;
	uint256 internal constant PYTH_FEE_WEI = 1;
	uint256 internal constant ONE = 1e8;
	uint256 internal constant T0 = 1_794_218_400;
	uint256 internal constant BLOCK0 = 100_000_000;
	uint256 internal constant BLOCKS_PER_SECOND = 4;
	uint256 internal constant GAS_PRICE_WEI = 20_000_000;
	uint256 internal constant FLOOR_GAS_PRICE_WEI = 10_000_000;
	uint256 internal constant USD_CENTS_PER_ETH = 183_776;
	uint256 internal constant TX_INTRINSIC_GAS = 21_000;
	uint256 internal constant L1_GAS_36_BYTES = 1359;
	uint256 internal constant L1_GAS_2200_BYTES = 17_939;
	uint256 internal constant L1_GAS_4400_BYTES = 34_866;
	uint256 internal constant REAL_PYTH_CALLDATA_BYTES = 4400;
	uint256 internal constant REAL_PYTH_EXTRA_GAS = 500_000;
	uint256 internal constant SEEDED_BASKET = 1.006e8;
	uint256 internal constant VICTIM_MARGIN = 1_000_000e6;
	uint256 internal constant VICTIM_NOTIONAL = 10_000_000e6;
	uint256 internal constant VICTIM_FREE = 100_000e6;
	uint256 internal constant STOP = 1.01e8;
	uint256 internal constant CEIL = 1.0105e8;
	uint256 internal constant T_JUMP = T0 + 1 days + 3.5 hours;
	MockUSDG internal usdg;
	MockPyth internal pyth;
	MarginClearinghouse internal clearinghouse;
	CfdEngine internal engine;
	HousePool internal pool;
	TrancheVault internal seniorVault;
	TrancheVault internal juniorVault;
	CfdEngineLens internal engineLens;
	PletherOracle internal oracle;
	OrderLifecycleBook internal lifecycleBook;
	OrderRouter internal router;
	OrderRouterAdmin internal routerAdmin;
	EmergencyPauseCoordinator internal emergency;
	address internal book;
	address internal deployer = makeAddr("deployer");
	address internal guardian = makeAddr("guardian");
	address internal seedReceiver = makeAddr("seedReceiver");
	address internal keeper = makeAddr("keeper");
	address internal attacker = makeAddr("attacker");
	address internal victim = makeAddr("victim");
	uint256 internal basket;
	uint256 internal lastPublish;
	uint256 internal nonce;
	uint64 internal pid;
	uint256 internal size;
	uint64 internal live;
	mapping(uint64 => uint256) internal commitTimeOf;
	mapping(uint64 => uint256) internal commitBlockOf;
	function _deploy(string memory artifact, bytes memory args) internal returns (address addr) {
		addr = deployCode(artifact, args);
		vm.label(addr, artifact);
	}
	function _deployProtocol() internal {
		vm.startPrank(deployer);
		usdg = MockUSDG(_deploy("ProtectionRetryLock.t.sol:MockUSDG", ""));
		pyth = MockPyth(_deploy("MockPyth.sol:MockPyth", ""));
		pyth.setFee(PYTH_FEE_WEI);
		clearinghouse = MarginClearinghouse(_deploy("MarginClearinghouse.sol:MarginClearinghouse", abi.encode(address(usdg))));
		engine = CfdEngine(_deploy("CfdEngine.sol:CfdEngine", abi.encode(address(usdg), address(clearinghouse), CAP_PRICE, _riskParams(), FROZEN_CLOSE_SPREAD_BPS)));
		address navBook = _deploy("TerminalNavBookV2.sol:TerminalNavBookV2", abi.encode(address(engine), CAP_PRICE));
		engine.setTerminalNavBook(navBook);
		address planner = _deploy("CfdEnginePlanner.sol:CfdEnginePlanner", "");
		address settlementSidecar = _deploy("CfdEngineSettlementSidecar.sol:CfdEngineSettlementSidecar", abi.encode(address(engine)));
		address engineAdmin = _deploy("CfdEngineAdmin.sol:CfdEngineAdmin", abi.encode(address(engine), deployer));
		engine.setDependencies(planner, settlementSidecar, engineAdmin);
		address redemptionMath = _deploy("HousePoolRedemptionMathSidecar.sol:HousePoolRedemptionMathSidecar", "");
		pool = HousePool(_deploy("ProtectionRetryLock.t.sol:ArbitrumSepoliaReleaseHousePool", abi.encode(address(usdg), address(engine), redemptionMath)));
		seniorVault = TrancheVault(_deploy("TrancheVault.sol:TrancheVault", abi.encode(address(usdg), address(pool), true, "Plether Senior LP", "psLP", uint256(0), address(0))));
		pool.setSeniorVault(address(seniorVault));
		address juniorFeeRecipient = engine.protocolTreasury();
		juniorVault = TrancheVault(_deploy("TrancheVault.sol:TrancheVault", abi.encode(address(usdg), address(pool), false, "Plether Junior LP", "pjLP", JUNIOR_MAINTENANCE_FEE_APR_BPS,
					juniorFeeRecipient)));
		pool.setJuniorVault(address(juniorVault));
		engine.setPool(address(pool));
		address accountLens = _deploy("CfdEngineAccountLens.sol:CfdEngineAccountLens", abi.encode(address(engine)));
		engineLens = CfdEngineLens(_deploy("CfdEngineLens.sol:CfdEngineLens", abi.encode(address(engine))));
		oracle = PletherOracle(_deploy("ProtectionRetryLock.t.sol:ArbitrumSepoliaReleaseOracle", abi.encode(address(engine), address(pool), address(pyth), _feedIds(), _quantities(), _basePrices(),
					_inversions())));
		address policyEvaluator = _deploy("CfdOrderPolicyEvaluator.sol:CfdOrderPolicyEvaluator", "");
		_deploy("CfdClosePreview.sol:CfdClosePreview", "");
		address executionSidecar = _deploy("OrderRouterExecutionSidecar.sol:OrderRouterExecutionSidecar", "");
		address expectedRouter = vm.computeCreateAddress(deployer, uint256(vm.getNonce(deployer)) + 2);
		lifecycleBook = OrderLifecycleBook(_deploy("OrderLifecycleBook.sol:OrderLifecycleBook", abi.encode(expectedRouter, address(engine), address(clearinghouse), address(pool))));
		address liquidationBatchSidecar = _deploy("OrderRouterLiquidationBatchSidecar.sol:OrderRouterLiquidationBatchSidecar", abi.encode(expectedRouter));
		router = OrderRouter(payable(_deploy("ProtectionRetryLock.t.sol:ArbitrumSepoliaReleaseRouter", abi.encode(address(engine), address(engineLens), address(pool), address(oracle),
						liquidationBatchSidecar, policyEvaluator, executionSidecar, address(lifecycleBook)))));
		require(address(router) == expectedRouter, "OrderRouter CREATE address mismatch");
		book = address(router.positionProtectionBook());
		routerAdmin = OrderRouterAdmin(router.admin());
		engine.setOrderRouter(address(router));
		clearinghouse.setEngine(address(engine));
		_deploy("PerpsPublicLens.sol:PerpsPublicLens", abi.encode(accountLens, address(engine), address(router), address(pool)));
		_deploy("SettlementMonitorLens.sol:SettlementMonitorLens", abi.encode(address(router)));
		emergency = EmergencyPauseCoordinator(_deploy("EmergencyPauseCoordinator.sol:EmergencyPauseCoordinator", abi.encode(address(routerAdmin), address(pool), deployer)));
		pool.setPauser(address(emergency));
		routerAdmin.setPauser(address(emergency));
		vm.stopPrank();
		vm.label(book, "PositionProtectionBook");
	}
	function _bootstrapRelease() internal {
		vm.startPrank(deployer);
		emergency.setGuardian(guardian);
		usdg.mint(deployer, 2 * RELEASE_SEED_ASSETS);
		usdg.approve(address(pool), 2 * RELEASE_SEED_ASSETS);
		pool.initializeSeedPosition(false, RELEASE_SEED_ASSETS, seedReceiver);
		pool.initializeSeedPosition(true, RELEASE_SEED_ASSETS, seedReceiver);
		vm.stopPrank();
		_setPrice(ONE);
		vm.prank(deployer);
		pool.activateTrading();
	}
	function _riskParams() internal pure returns (CfdTypes.RiskParams memory) {
		return CfdTypes.RiskParams({
			vpiFactor: 0.01e18, maxSkewRatio: 0.4e18, maintMarginBps: 10, initMarginBps: 20, fadMarginBps: 300, baseCarryBps: 500, minBountyAssets: 1e6, bountyBps: 10, keeperShareBps: 5000,
			protocolShareBps: 0
		});
	}
	function _feedIds() internal pure returns (bytes32[] memory ids) {
		ids = new bytes32[](6);
		ids[0] = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
		ids[1] = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
		ids[2] = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
		ids[3] = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
		ids[4] = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
		ids[5] = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;
	}
	function _quantities() internal pure returns (uint256[] memory q) {
		q = new uint256[](6);
		q[0] = 576e15;
		q[1] = 136e15;
		q[2] = 119e15;
		q[3] = 91e15;
		q[4] = 42e15;
		q[5] = 36e15;
	}
	function _basePrices() internal pure returns (uint256[] memory b) {
		b = new uint256[](6);
		b[0] = 117_500_000;
		b[1] = 638_000;
		b[2] = 134_480_000;
		b[3] = 72_880_000;
		b[4] = 10_860_000;
		b[5] = 126_100_000;
	}
	function _inversions() internal pure returns (bool[] memory inv) {
		inv = new bool[](6);
		inv[1] = true;
		inv[3] = true;
		inv[4] = true;
		inv[5] = true;
	}
	function _seedMarket() internal {
		address[3] memory jLps = [makeAddr("lpJunior1"), makeAddr("lpJunior2"), makeAddr("lpJunior3")];
		uint256[3] memory jAmt = [uint256(3_000_000e6), 1_000_000e6, 7_500_000e6];
		address[3] memory sLps = [makeAddr("lpSenior1"), makeAddr("lpSenior2"), makeAddr("lpSenior3")];
		uint256[3] memory sAmt = [uint256(5_000_000e6), 3_000_000e6, 31_990_000e6];
		uint256[3] memory ids;
		for (uint256 i; i < 3; ++i) {
			ids[i] = _lpRequestDeposit(juniorVault, jLps[i], jAmt[i]);
		}
		_settleEpoch();
		for (uint256 i; i < 3; ++i) {
			_lpClaimDeposit(juniorVault, jLps[i], ids[i]);
		}
		for (uint256 i; i < 3; ++i) {
			ids[i] = _lpRequestDeposit(seniorVault, sLps[i], sAmt[i]);
		}
		_settleEpoch();
		for (uint256 i; i < 3; ++i) {
			_lpClaimDeposit(seniorVault, sLps[i], ids[i]);
		}
		_openAtNotional(makeAddr("alice"), CfdTypes.Side.LONG, 2_000_000e6, 200_000e6);
		_openAtNotional(makeAddr("bob"), CfdTypes.Side.LONG, 1_000_000e6, 50_000e6);
		_openAtNotional(makeAddr("carol"), CfdTypes.Side.LONG, 250_000e6, 5000e6);
		_openAtNotional(makeAddr("dave"), CfdTypes.Side.SHORT, 1_500_000e6, 300_000e6);
		_openAtNotional(makeAddr("erin"), CfdTypes.Side.SHORT, 800_000e6, 32_000e6);
		_openAtNotional(makeAddr("frank"), CfdTypes.Side.SHORT, 100_000e6, 10_000e6);
		_setPrice(SEEDED_BASKET);
		_warp(1 hours);
		engine.checkpointCarryIndexes();
		_setPrice(SEEDED_BASKET);
	}
	function _lpRequestDeposit(TrancheVault vault, address lp, uint256 assets) internal returns (uint256 requestId) {
		usdg.mint(lp, assets);
		vm.startPrank(lp);
		usdg.approve(address(vault), assets);
		requestId = vault.requestDeposit(assets, lp, lp);
		vm.stopPrank();
	}
	function _lpClaimDeposit(TrancheVault vault, address lp, uint256 requestId) internal {
		uint256 claimable = vault.claimableDepositRequest(requestId, lp);
		require(claimable > 0, "deposit not claimable");
		vm.prank(lp);
		vault.claimDeposit(requestId, claimable, lp, lp);
	}
	function _settleEpoch() internal {
		uint256 next = pool.lpEpochStart(pool.currentLpEpoch() + 1);
		_warp(next + 1 - vm.getBlockTimestamp());
		lastPublish = vm.getBlockTimestamp();
		bytes[] memory data = _pythData(basket, vm.getBlockTimestamp(), vm.getBlockTimestamp() - 1);
		uint256 fee = oracle.getUpdateFee(data);
		vm.prank(keeper);
		router.settleLpEpoch{value: fee}(data);
	}
	function _pythData(uint256 basket8, uint256 publishTime, uint256 prevPublishTime) internal pure returns (bytes[] memory data) {
		uint256[] memory bases = _basePrices();
		bytes32[] memory ids = _feedIds();
		MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](6);
		for (uint256 i; i < 6; ++i) {
			uint256 usdPerUnit = (bases[i] * basket8 + ONE / 2) / ONE;
			bool inverted = (i == 1 || i == 3 || i == 4 || i == 5);
			int32 expo = i == 1 ? int32(-3) : int32(-5);
			uint256 raw;
			if (inverted) {
				uint256 scale = 10 ** uint256(uint32(-expo));
				raw = (scale * ONE + usdPerUnit / 2) / usdPerUnit;
			} else {
				raw = (usdPerUnit + 500) / 1000;
			}
			uint64 conf = uint64(raw / 40_000);
			if (conf == 0) {
				conf = 1;
			}
			u[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(raw)), conf, expo, publishTime, prevPublishTime));
		}
		data = new bytes[](1);
		data[0] = abi.encode(u);
	}
	function _setPrice(uint256 basket8) internal {
		vm.roll(vm.getBlockNumber() + 1);
		vm.warp(vm.getBlockTimestamp() + 1);
		basket = basket8;
		lastPublish = vm.getBlockTimestamp();
		bytes[] memory data = _pythData(basket8, vm.getBlockTimestamp(), vm.getBlockTimestamp() - 1);
		uint256 fee = oracle.getUpdateFee(data);
		vm.prank(keeper);
		router.updateMarkPrice{value: fee}(data);
	}
	function _warp(uint256 dt) internal {
		vm.warp(vm.getBlockTimestamp() + dt);
		vm.roll(vm.getBlockNumber() + (dt == 0 ? 1 : dt * BLOCKS_PER_SECOND));
	}
	function _at(uint256 t, uint256 slot) internal {
		vm.warp(t);
		vm.roll(BLOCK0 + (t - T0) * BLOCKS_PER_SECOND + slot);
	}
	function _fundTrader(address trader, uint256 amount) internal {
		usdg.mint(trader, amount);
		vm.startPrank(trader);
		usdg.approve(address(clearinghouse), amount);
		clearinghouse.deposit(trader, amount);
		vm.stopPrank();
	}
	function _req(CfdTypes.Side side, uint256 sizeDelta, uint256 marginDelta, bool isClose) internal returns (OrderV3Types.OrderRequest memory r) {
		r.clientOrderId = keccak256(abi.encode("order", ++nonce));
		r.side = side;
		r.sizeDelta = sizeDelta;
		r.marginDelta = marginDelta;
		r.isClose = isClose;
		r.targetPrice = (side == CfdTypes.Side.LONG) != isClose ? 1 : type(uint256).max;
		uint256 window = router.maxExecutionWindowSeconds();
		r.bounds = OrderV3Types.ExecutionBounds({
			submitBy: uint64(vm.getBlockTimestamp() + window), executionWindowSeconds: uint32(window), allowedExecutionModes: 7, expectedConfigHash: lifecycleBook.currentExecutionConfigHash(),
			maxExecutionBountyAssets: type(uint256).max, maxExecutionNotionalAssets: type(uint256).max, maxGrossAccountDebitAssets: type(uint256).max, maxActionChargeAssets: type(uint256).max,
			maxExplicitFeesAssets: type(uint256).max, maxPostPositionSize: type(uint256).max, minPostSettlementBalanceAssets: 0, minPostPositionEquityAssets: 0, maxPostLeverageBps: type(uint32).max
		});
	}
	function _record(uint64 id) internal {
		commitTimeOf[id] = vm.getBlockTimestamp();
		commitBlockOf[id] = vm.getBlockNumber();
	}
	function _keeperExecute(uint64 id, uint256 basket8) internal returns (OrderV3Types.ExecutionResult memory res) {
		uint256 ct = commitTimeOf[id];
		if (vm.getBlockNumber() <= commitBlockOf[id]) {
			vm.roll(commitBlockOf[id] + 1);
		}
		if (vm.getBlockTimestamp() < ct + 2) {
			vm.warp(ct + 2);
		}
		basket = basket8;
		lastPublish = ct + 1;
		bytes[] memory data = _pythData(basket8, ct + 1, ct);
		uint256 fee = oracle.getOrderExecutionFee(data);
		vm.prank(keeper);
		res = router.executeOrder{value: fee}(id, data);
	}
	function _open(address trader, CfdTypes.Side side, uint256 sizeDelta, uint256 marginDelta) internal {
		OrderV3Types.OrderRequest memory r = _req(side, sizeDelta, marginDelta, false);
		vm.prank(trader);
		uint64 id = router.commitOrder(r);
		_record(id);
		OrderV3Types.ExecutionResult memory res = _keeperExecute(id, basket);
		require(res.status == OrderV3Types.LifecycleStatus.Executed, "open not executed");
	}
	function _openAtNotional(address trader, CfdTypes.Side side, uint256 notional6, uint256 margin6) internal {
		_fundTrader(trader, margin6 + notional6 / 100);
		_open(trader, side, _sizeFor(notional6, basket), margin6);
	}
	function _sizeFor(uint256 notional6, uint256 basket8) internal pure returns (uint256 s) {
		s = (notional6 * 1e20) / basket8;
		s -= s % CfdTypes.SIZE_QUANTUM;
	}
	function _positionSize(address trader) internal view returns (uint256 s) {
		(s,,,,,,) = engine.positions(trader);
	}
	function _positionMargin(address trader) internal view returns (uint256 m) {
		(, m,,,,,) = engine.positions(trader);
	}
	function _usdgOf(address who) internal view returns (uint256) {
		return usdg.balanceOf(who) + clearinghouse.balanceAssets(who);
	}
	function _status() internal view returns (PositionProtectionTypes.PositionProtectionStatus) {
		return IPositionProtectionViews(book).getPositionProtection(pid).status;
	}
	function _maxLimits() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory l) {
		l = PositionProtectionTypes.CloseFinancialLimits(type(uint256).max, type(uint256).max, type(uint256).max, type(uint256).max);
	}
	function _tryAs(address who, address target, bytes memory callData) internal returns (bool ok, bytes4 sel) {
		bytes memory ret;
		vm.prank(who);
		(ok, ret) = target.call(callData);
		if (!ok && ret.length >= 4) {
			sel = bytes4(ret);
		}
	}
	function _preAnchor(uint256 i) internal pure returns (uint256) {
		uint16[25] memory a = [uint16(10_060), 10_063, 10_061, 10_066, 10_070, 10_068, 10_072, 10_075, 10_071, 10_074, 10_078, 10_080,
			10_077, 10_081, 10_084, 10_082, 10_086, 10_088, 10_085, 10_089, 10_091, 10_088, 10_092, 10_094, 10_095];
		return uint256(a[i]) * 1e4;
	}
	function _postAnchor(uint256 i) internal pure returns (uint256) {
		uint16[26] memory a = [uint16(10_125), 10_128, 10_122, 10_130, 10_135, 10_131, 10_138, 10_142, 10_136, 10_140, 10_145, 10_150,
			10_147, 10_141, 10_138, 10_144, 10_149, 10_155, 10_151, 10_158, 10_162, 10_157, 10_163, 10_168, 10_170, 10_174];
		return uint256(a[i > 25 ? 25 : i]) * 1e4;
	}
	function _weekdayPrice(uint256 t) internal pure returns (uint256) {
		uint256 dt = t - T_JUMP;
		uint256 i = dt / 1 hours;
		if (i >= 25) {
			return _postAnchor(25);
		}
		uint256 a = _postAnchor(i);
		uint256 b = _postAnchor(i + 1);
		uint256 f = dt % 1 hours;
		return b >= a ? a + ((b - a) * f) / 1 hours : a - ((a - b) * f) / 1 hours;
	}
	function _bootMarket() internal {
		vm.warp(T0);
		vm.roll(BLOCK0);
		vm.deal(keeper, 1000 ether);
		vm.deal(attacker, 10 ether);
		_deployProtocol();
		_bootstrapRelease();
		_seedMarket();
		assertEq(router.maxExecutionWindowSeconds(), 60, "release execution window");
		assertEq(oracle.orderSettlementWindow(), 15, "release settlement window");
	}
	function _openAndProtectVictim() internal {
		_fundTrader(victim, VICTIM_MARGIN + VICTIM_FREE);
		size = _sizeFor(VICTIM_NOTIONAL, basket);
		_open(victim, CfdTypes.Side.LONG, size, VICTIM_MARGIN);
		_setPrice(SEEDED_BASKET);
		PositionProtectionTypes.PositionProtectionParams memory pp;
		pp.stopLossTriggerPrice = STOP;
		pp.closeLimits = _maxLimits();
		pp.closeLimits.maxExecutionNotionalAssets = (size * CEIL) / 1e20;
		vm.prank(victim);
		pid = IPositionProtectionActions(book).createPositionProtection(pp);
	}
	function _weekdayScenario() internal {
		_bootMarket();
		_openAndProtectVictim();
		for (uint256 i = 1; i < 25; ++i) {
			_warp(1 hours - 1);
			_setPrice(_preAnchor(i));
		}
		assertEq(uint8(_status()), uint8(PositionProtectionTypes.PositionProtectionStatus.Armed), "armed before the jump");
		_at(T_JUMP, 0);
		basket = _postAnchor(0);
		lastPublish = T_JUMP;
		bytes[] memory data = _pythData(basket, T_JUMP, T_JUMP - 1);
		uint256 fee = oracle.getUpdateFee(data);
		vm.prank(keeper);
		live = IPositionProtectionActions(book).triggerPositionProtection{value: fee}(pid, data);
		_record(live);
		assertEq(uint8(_status()), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered), "triggered at the jump");
	}
}
abstract contract ProtectionRetryLockCycles is ProtectionRetryLockFixture {
	bytes4 internal constant NOT_LATCHED = IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector;
	uint256 internal constant CHUNK = 1 hours;
	RetryAttacker internal attackerContract;
	bool internal atomicMode;
	uint256 internal lockEnd;
	uint256 internal curT;
	uint256 internal nextActor;
	bool internal stopped;
	uint256 internal midSample;
	uint256 internal firstFailAt;
	uint256 internal attackerTxs;
	uint256 internal amendReverts;
	uint256 internal keeperExecs;
	uint256 internal frozenExecs;
	uint256 internal boundaryChecks;
	uint256 internal attackerFeesWei;
	uint256 internal chunksRun;
	uint256 internal coldSamples;
	uint256 internal maxColdGas;
	uint256 internal minColdGas;
	uint256 internal warmSamples;
	uint256 internal maxWarmGas;
	uint256 internal maxKeeperGas;
	bool internal coldNext;
	uint256 internal v0;
	bool internal lockBroken;
	uint256 internal amendSuccesses;
	uint256 internal firstAmendOkAt;
	uint256 internal closedAt;
	function _price(uint256 t) internal view virtual returns (uint256);
	function _prevPublish(uint256 t) internal view virtual returns (uint256) {
		return t - 1;
	}
	function _payloadAt(uint256 t) internal view returns (bytes[] memory data, uint256 fee) {
		(, bool frozen) = MarketCalendarLib.marketStatus(t, false, false, 1 hours);
		if (frozen) {
			return (new bytes[](0), 0);
		}
		return (_pythData(_price(t), t, _prevPublish(t)), 2 * PYTH_FEE_WEI);
	}
	function _requireLive() internal view {
		require(_status() == PositionProtectionTypes.PositionProtectionStatus.Triggered && router.pendingOrderCounts(victim) == 1, "boundary: not Triggered with exactly one pending order");
	}
	function _checkBoundary() internal {
		if (stopped || lockBroken) {
			return;
		}
		require(_positionSize(victim) == size, "the position must stay open and unchanged");
		if (!atomicMode && nextActor == 1) {
			require(_status() == PositionProtectionTypes.PositionProtectionStatus.Latched && router.pendingOrderCounts(victim) == 0, "after a keeper execution: Latched, zero pending");
		} else {
			_requireLive();
		} ++boundaryChecks;
	}
	function _afterExecution() internal virtual {}
	function step() external {
		require(msg.sender == address(this), "self");
		if (stopped) {
			return;
		}
		if (nextActor == 0) {
			uint256 t = commitTimeOf[live] + 1;
			if (t >= lockEnd) {
				stopped = true;
				return;
			}
			curT = t;
			bool closed = atomicMode ? _attackerExecutesAndRetries(t) : _keeperExecutes(t);
			if (closed) {
				stopped = true;
				return;
			}
			_afterExecution();
			nextActor = 1;
		} else if (nextActor == 1 && !atomicMode) {
			_attackerRetries(curT);
			nextActor = 2;
		} else {
			_at(curT, atomicMode ? 1 : 2);
			_ownerAmends();
			nextActor = 0;
		}
		_checkBoundary();
	}
	function _nextAttacker() internal view returns (uint256 t, bool attackerNext) {
		if (atomicMode) {
			return (commitTimeOf[live] + 1, nextActor == 0);
		}
		return (curT, nextActor == 1);
	}
	function runChunk() external {
		if (stopped) {
			return;
		}
		(uint256 s, bool attackerFirst) = _nextAttacker();
		require(attackerFirst, "a chunk starts with an attacker transaction");
		uint256 chunkEnd = s + CHUNK;
		coldNext = true;
		bool paused;
		while (!stopped) {
			(uint256 t, bool attackerNext) = _nextAttacker();
			if (attackerNext && t >= chunkEnd && t < lockEnd) {
				break;
			}
			bool window = attackerTxs < 50 || (attackerTxs >= midSample && attackerTxs < midSample + 50);
			bool meter = (attackerNext && coldNext) || window;
			if (meter == paused) {
				if (meter) {
					vm.resumeGasMetering();
				} else {
					vm.pauseGasMetering();
				}
				paused = !meter;
			}
			this.step();
		}
		if (paused) {
			vm.resumeGasMetering();
		} ++chunksRun;
	}
	function _chunks(uint256 n) internal view returns (bytes[] memory calls) {
		calls = new bytes[](n);
		for (uint256 i; i < n; ++i) {
			calls[i] = abi.encodeCall(this.runChunk, ());
		}
	}
	function _recordAttackerGas(uint256 g) internal {
		if (g == 0) {
			return;
		}
		if (coldNext) {
			coldNext = false; ++coldSamples;
			if (g > maxColdGas) {
				maxColdGas = g;
			}
			if (minColdGas == 0 || g < minColdGas) {
				minColdGas = g;
			}
		} else { ++warmSamples;
			if (g > maxWarmGas) {
				maxWarmGas = g;
			}
		}
	}
	function _maxAttackerGas() internal view returns (uint256) {
		return maxColdGas > maxWarmGas ? maxColdGas : maxWarmGas;
	}
	function _keeperExecutes(uint256 t) internal returns (bool closed) {
		_at(t, 0);
		(bytes[] memory data, uint256 fee) = _payloadAt(t);
		if (data.length == 0) { ++frozenExecs;
		}
		vm.prank(keeper);
		uint256 g = gasleft();
		OrderV3Types.ExecutionResult memory r = router.executeOrder{value: fee}(live, data);
		g -= gasleft();
		if (g > maxKeeperGas) {
			maxKeeperGas = g;
		}
		if (r.status == OrderV3Types.LifecycleStatus.Executed) {
			require(lockBroken, "keeper: the close filled although no owner amend landed");
			closedAt = t;
			return true;
		}
		require(r.status == OrderV3Types.LifecycleStatus.Failed && r.terminalReason == OrderV3Types.TerminalReason.ConstraintViolation, "keeper: the attempt did not fail on the ceiling");
		require(_status() == PositionProtectionTypes.PositionProtectionStatus.Latched && router.pendingOrderCounts(victim) == 0, "keeper: the failure did not relatch");
		if (firstFailAt == 0) {
			firstFailAt = t;
		} ++keeperExecs;
	}
	function _attackerRetries(uint256 t) internal {
		_at(t, 1);
		vm.prank(attacker);
		uint256 g = gasleft();
		uint64 next = IPositionProtectionActions(book).retryPositionProtectionClose(pid);
		g -= gasleft();
		_recordAttackerGas(g);
		_requireLive();
		_record(next);
		live = next; ++attackerTxs;
	}
	function _attackerExecutesAndRetries(uint256 t) internal returns (bool closed) {
		_at(t, 0);
		(bytes[] memory data, uint256 fee) = _payloadAt(t);
		vm.prank(attacker);
		uint256 g = gasleft();
		(uint8 st, uint8 reason, uint64 next) = attackerContract.executeAndRetry{value: fee}(router, book, live, data, pid);
		g -= gasleft();
		_recordAttackerGas(g); ++attackerTxs;
		attackerFeesWei += fee;
		if (st == uint8(OrderV3Types.LifecycleStatus.Executed)) {
			require(lockBroken, "attacker: the close filled although no owner amend landed");
			closedAt = t;
			return true;
		}
		require(st == uint8(OrderV3Types.LifecycleStatus.Failed) && reason == uint8(OrderV3Types.TerminalReason.ConstraintViolation) && next != 0, "attacker: execute + retry failed");
		_requireLive();
		if (firstFailAt == 0) {
			firstFailAt = t;
		}
		_record(next);
		live = next;
	}
	function _ownerAmends() internal {
		(bool ok, bytes4 sel) = _tryAs(victim, book, abi.encodeCall(IPositionProtectionActions.amendLatchedPositionProtectionCloseLimits, (pid, _maxLimits())));
		if (ok) { ++amendSuccesses;
			if (!lockBroken) {
				lockBroken = true;
				firstAmendOkAt = vm.getBlockTimestamp();
			}
			return;
		}
		require(sel == NOT_LATCHED, "owner amend: unexpected revert"); ++amendReverts;
	}
	function _logOutcome(string memory tag) internal view {
		console2.log(string.concat(tag, ": owner amends that succeeded while the attacker was active"), amendSuccesses);
		if (amendSuccesses > 0) {
			console2.log(string.concat(tag, ": first successful owner amend, seconds after the first failure"), firstAmendOkAt - firstFailAt);
			if (closedAt != 0) {
				console2.log(string.concat(tag, ": position closed, seconds after the first failure"), closedAt - firstFailAt);
			}
			console2.log(string.concat(tag, ": attacker transactions until then"), attackerTxs);
		}
	}
	function _calldataGas(bytes memory data) internal pure returns (uint256 g) {
		for (uint256 i; i < data.length; ++i) {
			g += data[i] == 0 ? 4 : 16;
		}
	}
	function _l1Gas(uint256 len) internal pure returns (uint256) {
		if (len <= 36) {
			return L1_GAS_36_BYTES;
		}
		if (len <= 2200) {
			return L1_GAS_2200_BYTES;
		}
		require(len <= 4400, "calldata larger than the measured L1 points");
		return L1_GAS_4400_BYTES;
	}
	function _retryTxGas() internal view returns (uint256) {
		bytes memory cd = abi.encodeCall(IPositionProtectionActions.retryPositionProtectionClose, (pid));
		return _maxAttackerGas() + TX_INTRINSIC_GAS + _calldataGas(cd) + _l1Gas(cd.length);
	}
	function _atomicTxGas() internal view returns (uint256) {
		return _maxAttackerGas() + REAL_PYTH_EXTRA_GAS + TX_INTRINSIC_GAS + 16 * REAL_PYTH_CALLDATA_BYTES + _l1Gas(REAL_PYTH_CALLDATA_BYTES);
	}
	function _reportCost(string memory tag, uint256 txs, uint256 perTxGas, uint256 valueWei, uint256 frozenMargin6) internal pure returns (uint256 costWei) {
		costWei = txs * perTxGas * GAS_PRICE_WEI + valueWei;
		uint256 costWeiFloor = txs * perTxGas * FLOOR_GAS_PRICE_WEI + valueWei;
		uint256 costCents = (costWei * USD_CENTS_PER_ETH) / 1e18;
		uint256 costCentsFloor = (costWeiFloor * USD_CENTS_PER_ETH) / 1e18;
		uint256 minMargin6 = (100 * 1e4 * USD_CENTS_PER_ETH * costWei + 1e18 - 1) / 1e18;
		console2.log(string.concat(tag, ": attacker transactions"), txs);
		console2.log(string.concat(tag, ": L2 gas per attacker transaction (cost model)"), perTxGas);
		console2.log(string.concat(tag, ": attacker cost, wei at 0.02 gwei (incl. Pyth fees)"), costWei);
		console2.log(string.concat(tag, ": attacker cost, wei at 0.01 gwei (incl. Pyth fees)"), costWeiFloor);
		console2.log(string.concat(tag, ": attacker cost, US cents at 0.02 gwei"), costCents);
		console2.log(string.concat(tag, ": attacker cost, US cents at 0.01 gwei"), costCentsFloor);
		console2.log(string.concat(tag, ": frozen position margin, US cents"), frozenMargin6 / 1e4);
		console2.log(string.concat(tag, ": frozen / cost at 0.02 gwei (integer)"), frozenMargin6 / 1e4 / costCents);
		console2.log(string.concat(tag, ": frozen / cost at 0.01 gwei (integer)"), frozenMargin6 / 1e4 / costCentsFloor);
		console2.log(string.concat(tag, ": smallest margin for 100:1 at 0.02 gwei (USDG, 6 dec)"), minMargin6);
		require(frozenMargin6 * 1e18 >= 100 * 1e4 * USD_CENTS_PER_ETH * costWei, "frozen / cost below 100:1");
	}
	function _logGas(string memory tag) internal view {
		console2.log(string.concat(tag, ": committed chunk transactions"), chunksRun);
		console2.log(string.concat(tag, ": attacker gas samples, fresh transaction (chunk start)"), coldSamples);
		console2.log(string.concat(tag, ": metered execution gas, fresh-transaction samples, min"), minColdGas);
		console2.log(string.concat(tag, ": metered execution gas, fresh-transaction samples, max"), maxColdGas);
		console2.log(string.concat(tag, ": attacker gas samples inside a chunk"), warmSamples);
		console2.log(string.concat(tag, ": metered execution gas, in-chunk samples, max"), maxWarmGas);
		if (maxKeeperGas != 0) {
			console2.log(string.concat(tag, ": metered execution gas, keeper execution, max (the keeper's cost)"), maxKeeperGas);
		}
	}
	function _withdrawDuringLock(string memory tag, bool takeFree) internal returns (uint256 freeAssets) {
		uint256 marginAssets = _positionMargin(victim);
		freeAssets = clearinghouse.getFreeBuyingPowerAssets(victim);
		console2.log(string.concat(tag, ": clearinghouse balance (6 dec)"), clearinghouse.balanceAssets(victim));
		console2.log(string.concat(tag, ": position margin (6 dec)"), marginAssets);
		console2.log(string.concat(tag, ": total locked buckets (6 dec)"), clearinghouse.lockedMarginAssets(victim));
		console2.log(string.concat(tag, ": free balance above the locked buckets (6 dec)"), freeAssets);
		(bool ok, bytes4 sel) = _tryAs(victim, address(clearinghouse), abi.encodeCall(MarginClearinghouse.withdraw, (victim, marginAssets)));
		assertFalse(ok, "the position margin cannot be withdrawn during the lock");
		assertEq(sel, MarginClearinghouse.MarginClearinghouse__InsufficientFreeEquity.selector, "withdraw(position margin) reverts InsufficientFreeEquity");
		console2.log(string.concat(tag, ": withdraw(position margin) reverted MarginClearinghouse__InsufficientFreeEquity"));
		if (takeFree) {
			(ok, sel) = _tryAs(victim, address(clearinghouse), abi.encodeCall(MarginClearinghouse.withdraw, (victim, freeAssets + 1)));
			assertFalse(ok, "one unit above the free balance cannot be withdrawn");
			assertEq(sel, MarginClearinghouse.MarginClearinghouse__InsufficientFreeEquity.selector, "withdraw(free + 1) reverts InsufficientFreeEquity");
			uint256 w0 = usdg.balanceOf(victim);
			vm.prank(victim);
			clearinghouse.withdraw(victim, freeAssets);
			assertEq(usdg.balanceOf(victim) - w0, freeAssets, "the free balance is withdrawable");
			console2.log(string.concat(tag, ": withdraw(free + 1) reverted; withdraw(free) paid out (6 dec)"), freeAssets);
		}
	}
	function _byDesignExitsRevert() internal {
		bool ok;
		bytes4 sel;
		(ok, sel) = _tryAs(victim, book, abi.encodeCall(IPositionProtectionActions.cancelPositionProtection, (pid)));
		assertTrue(!ok && sel == IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector, "by design: cancel");
		OrderV3Types.OrderRequest memory r = _req(CfdTypes.Side.LONG, size, 0, true);
		(ok, sel) = _tryAs(victim, book, abi.encodeCall(IPositionProtectionActions.cancelPositionProtectionAndCommitClose, (pid, r)));
		assertTrue(!ok && sel == IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector, "by design: cancel-and-close");
		r = _req(CfdTypes.Side.LONG, size, 0, true);
		(ok, sel) = _tryAs(victim, address(router), abi.encodeWithSelector(OrderRouter.commitOrder.selector, r));
		assertTrue(!ok && sel == IOrderRouterErrors.OrderRouter__ProtectionActive.selector, "by design: close order");
		console2.log("BY DESIGN for any triggered protection: cancel, cancel-and-close, close order each revert (3 of 3)");
	}
	function _attackerStopsOwnerExits(string memory tag) internal returns (uint256 amendAt) {
		uint256 t = commitTimeOf[live] + 1;
		require(!_keeperExecutes(t), "release: the last attempt still fails on the ceiling");
		_at(t, 1);
		vm.prank(victim);
		IPositionProtectionActions(book).amendLatchedPositionProtectionCloseLimits(pid, _maxLimits());
		amendAt = t;
		_at(t, 2);
		vm.prank(victim);
		uint64 next = IPositionProtectionActions(book).retryPositionProtectionClose(pid);
		_record(next);
		_at(t + 1, 0);
		(bytes[] memory data, uint256 fee) = _payloadAt(t + 1);
		vm.prank(keeper);
		OrderV3Types.ExecutionResult memory r = router.executeOrder{value: fee}(next, data);
		assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "the amended attempt fills");
		assertEq(_positionSize(victim), 0, "position closed after the attacker stops");
		console2.log(string.concat(tag, ": owner's first successful amend, seconds after the first failure"), amendAt - firstFailAt);
		console2.log(string.concat(tag, ": exit basket after the lock (8 dec)"), _price(t + 1));
	}
	function _withdrawAllAfterExit(string memory tag) internal {
		uint256 rest = clearinghouse.balanceAssets(victim);
		vm.prank(victim);
		clearinghouse.withdraw(victim, rest);
		assertEq(clearinghouse.balanceAssets(victim), 0, "everything withdrawn after the exit");
		console2.log(string.concat(tag, ": after the exit the owner withdrew its whole clearinghouse balance (6 dec)"), rest);
		console2.log(string.concat(tag, ": owner's USDG custody when the lock started / after the exit (6 dec)"), v0, _usdgOf(victim));
	}
}
contract ProtectionRetryLockKeeperPresentTest is ProtectionRetryLockCycles {
	uint256 internal constant LOCK = 24 hours;
	function _price(uint256 t) internal pure override returns (uint256) {
		return _weekdayPrice(t);
	}
	function setUp() public {
		_weekdayScenario();
		_keeperExecutes(T_JUMP + 1);
		v0 = _usdgOf(victim);
		curT = firstFailAt;
		nextActor = 1;
		lockEnd = firstFailAt + LOCK;
		midSample = LOCK / 2;
	}
	function beforeTestSetup(bytes4 testSelector) public view returns (bytes[] memory calls) {
		if (testSelector == this.test_A_keeperPresent_attackerBackruns_24h.selector) {
			calls = _chunks(LOCK / CHUNK);
		}
	}
	function test_A_keeperPresent_attackerBackruns_24h() public {
		_logOutcome("A");
		assertEq(amendSuccesses, 0, "A: no owner amend succeeded while the attacker was active");
		assertTrue(stopped && nextActor == 0, "A: the lock ran to its end");
		_requireLive();
		assertEq(_positionSize(victim), size, "A: the position is still open at the end of the lock");
		_byDesignExitsRevert();
		uint256 freeAssets = _withdrawDuringLock("A", true);
		uint256 frozen = _positionMargin(victim);
		console2.log("A: attacker back-run retries", attackerTxs);
		console2.log("A: owner amends sent after them / reverted ProtectionNotLatched", amendReverts, amendReverts);
		console2.log("A: keeper executions that failed and relatched", keeperExecs);
		console2.log("A: transaction boundaries checked", boundaryChecks);
		_logGas("A");
		uint256 costWei = _reportCost("A", attackerTxs, _retryTxGas(), 0, frozen);
		uint256 amendAt = _attackerStopsOwnerExits("A");
		uint256 lockedFor = amendAt - firstFailAt;
		console2.log("A: seconds from the first failure to the owner's first successful amend", lockedFor);
		_withdrawAllAfterExit("A");
		assertGe(lockedFor, 24 hours, "A: the owner could not amend for >= 24 h");
		assertEq(chunksRun, 24, "A: 24 committed one-hour transactions");
		assertEq(attackerTxs, 86_400, "A: one back-run per keeper execution, one per second for 24 h");
		assertEq(amendReverts, attackerTxs, "A: the owner's amend reverted after every attacker transaction");
		assertEq(keeperExecs, attackerTxs + 1, "A: every keeper execution failed and relatched");
		assertEq(boundaryChecks, 3 * attackerTxs - 1, "A: every transaction boundary of the lock was checked");
		assertEq(coldSamples, 24, "A: one fresh-transaction gas sample per chunk");
		assertGe(frozen, 900_000e6, "A: the position margin stayed locked");
		assertTrue(freeAssets > 0 && costWei > 0, "A: withdrawals and cost recorded");
	}
	function test_control_noAttacker_ownerAmendsAndExits() public {
		uint256 t = firstFailAt;
		_at(t, 1);
		vm.prank(victim);
		IPositionProtectionActions(book).amendLatchedPositionProtectionCloseLimits(pid, _maxLimits());
		_at(t, 2);
		vm.prank(keeper);
		uint64 next = IPositionProtectionActions(book).retryPositionProtectionClose(pid);
		_record(next);
		_at(t + 1, 0);
		(bytes[] memory data, uint256 fee) = _payloadAt(t + 1);
		vm.prank(keeper);
		OrderV3Types.ExecutionResult memory r = router.executeOrder{value: fee}(next, data);
		assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "control: the amended retry fills");
		assertEq(_positionSize(victim), 0, "control: closed");
		console2.log("CONTROL: seconds from the first failure to the fill", (t + 1) - firstFailAt);
		console2.log("CONTROL: exit basket (8 dec)", _price(t + 1));
		_withdrawAllAfterExit("CONTROL");
	}
	function test_fundsFrozen_withdrawalsDuringLock() public {
		lockEnd = firstFailAt + 10 minutes;
		while (!stopped) {
			this.step();
		}
		_logOutcome("FUNDS");
		assertEq(amendSuccesses, 0, "FUNDS: no owner amend succeeded while the attacker was active");
		_requireLive();
		_withdrawDuringLock("FUNDS", true);
		uint256 amendAt = _attackerStopsOwnerExits("FUNDS");
		_withdrawAllAfterExit("FUNDS");
		assertEq(attackerTxs, 600, "FUNDS: 600 back-runs in 10 minutes");
		assertEq(amendReverts, attackerTxs, "FUNDS: every amend reverted");
		assertEq(amendAt - firstFailAt, 10 minutes, "FUNDS: the amend is reachable only once the attacker stops");
	}
}
contract ProtectionRetryLockAtomicTest is ProtectionRetryLockCycles {
	uint256 internal constant LOCK = 24 hours;
	function _price(uint256 t) internal pure override returns (uint256) {
		return _weekdayPrice(t);
	}
	function setUp() public {
		_weekdayScenario();
		vm.prank(attacker);
		attackerContract = new RetryAttacker();
		v0 = _usdgOf(victim);
		atomicMode = true;
		nextActor = 0;
		lockEnd = T_JUMP + 1 + LOCK;
		midSample = LOCK / 2;
	}
	function beforeTestSetup(bytes4 testSelector) public view returns (bytes[] memory calls) {
		if (testSelector == this.test_B_atomic_noWindow_24h.selector) {
			calls = _chunks(LOCK / CHUNK);
		}
	}
	function test_B_atomic_noWindow_24h() public {
		_logOutcome("B");
		assertEq(amendSuccesses, 0, "B: no owner amend succeeded while the attacker was active");
		assertTrue(stopped && nextActor == 0, "B: the lock ran to its end");
		_requireLive();
		assertEq(_positionSize(victim), size, "B: the position is still open at the end of the lock");
		_withdrawDuringLock("B", true);
		uint256 frozen = _positionMargin(victim);
		console2.log("B: attacker execute + retry transactions", attackerTxs);
		console2.log("B: owner amends sent after them / reverted ProtectionNotLatched", amendReverts, amendReverts);
		console2.log("B: transaction boundaries checked, each Triggered with one pending order", boundaryChecks);
		console2.log("B: Pyth fees paid by the attacker (wei)", attackerFeesWei);
		_logGas("B");
		console2.log("B: L2 gas per attacker transaction without the real-Pyth allowance", _atomicTxGas() - REAL_PYTH_EXTRA_GAS);
		uint256 costWei = _reportCost("B", attackerTxs, _atomicTxGas(), attackerFeesWei, frozen);
		uint256 amendAt = _attackerStopsOwnerExits("B");
		uint256 lockedFor = amendAt - firstFailAt;
		console2.log("B: seconds from the first failure to the owner's first successful amend", lockedFor);
		_withdrawAllAfterExit("B");
		assertGe(lockedFor, 24 hours, "B: the owner could not amend for >= 24 h");
		assertEq(chunksRun, 24, "B: 24 committed one-hour transactions");
		assertEq(attackerTxs, 86_400, "B: one attacker transaction per second for 24 h");
		assertEq(amendReverts, attackerTxs, "B: the owner's amend reverted after every attacker transaction");
		assertEq(boundaryChecks, 2 * attackerTxs, "B: both boundaries of every cycle were checked");
		assertEq(coldSamples, 24, "B: one fresh-transaction gas sample per chunk");
		assertGe(frozen, 900_000e6, "B: the position margin stayed locked");
		assertGt(costWei, 0, "B: cost recorded");
	}
	function test_busyQueue_attackerBatchesThroughOtherOrders() public {
		address[3] memory others = [makeAddr("traderX"), makeAddr("traderY"), makeAddr("traderZ")];
		for (uint256 i; i < 3; ++i) {
			_fundTrader(others[i], 100_000e6);
		}
		vm.deal(victim, 1 ether);
		uint64[3] memory ahead;
		uint256 othersExecuted;
		uint256 batchGas;
		bool headRuleShown;
		for (uint256 k; k < 10; ++k) {
			uint256 t = commitTimeOf[live] + 1;
			_at(t, 0);
			(bytes[] memory data, uint256 fee) = _payloadAt(t);
			if (k > 0 && !headRuleShown) {
				vm.prank(victim);
				(bool ok, bytes memory ret) = address(router).call{value: fee}(abi.encodeCall(OrderRouter.executeOrder, (live, data)));
				assertTrue(!ok && bytes4(ret) == IOrderRouterErrors.OrderRouter__OrderNotQueueHead.selector, "busy: a single execution of a non-head attempt reverts OrderNotQueueHead");
				headRuleShown = true;
			}
			uint64[3] memory fresh;
			for (uint256 i; i < 3; ++i) {
				OrderV3Types.OrderRequest memory r = _req(i == 1 ? CfdTypes.Side.SHORT : CfdTypes.Side.LONG, _sizeFor(1100e6, _price(t)), 200e6, false);
				vm.prank(others[i]);
				fresh[i] = router.commitOrder(r);
			}
			uint64 attempt = live;
			vm.prank(attacker);
			uint256 g = gasleft();
			(uint32 n, uint64 next) = attackerContract.batchAndRetry{value: 4 * fee}(router, book, attempt, data, pid);
			g -= gasleft();
			OrderV3Types.TerminalOutcome memory o = lifecycleBook.terminalOutcome(attempt);
			assertEq(uint8(o.reason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "busy: attempt fails");
			assertEq(n, k == 0 ? 1 : 4, "busy: the batch ended the three orders ahead and the attempt");
			if (k > 0) {
				for (uint256 i; i < 3; ++i) {
					assertEq(uint8(lifecycleBook.terminalOutcome(ahead[i]).status), uint8(OrderV3Types.LifecycleStatus.Executed), "busy: the other traders' orders executed"); ++othersExecuted;
				}
				if (g > batchGas) {
					batchGas = g;
				}
			}
			if (firstFailAt == 0) {
				firstFailAt = t;
			}
			_requireLive();
			_record(next);
			live = next; ++attackerTxs;
			ahead = fresh;
			_at(t, 1);
			_ownerAmends();
		}
		_logOutcome("BUSY QUEUE");
		assertEq(amendSuccesses, 0, "busy: no owner amend succeeded");
		assertEq(amendReverts, 10, "busy: the owner's amend reverted after every attacker transaction");
		assertEq(othersExecuted, 27, "busy: 27 other orders were ahead of an attempt and executed by the batch");
		assertEq(_positionSize(victim), size, "busy: the position is still open");
		console2.log("BUSY QUEUE: attacker batch transactions / other traders' orders they executed", attackerTxs, othersExecuted);
		console2.log("BUSY QUEUE: owner amends that reverted ProtectionNotLatched", amendReverts);
		console2.log("BUSY QUEUE: metered gas of one batch (three other orders + attempt + retry), max", batchGas);
	}
	function test_eoaOwnerWinsExecutionRace_attackerBackruns_amendStillReverts() public {
		vm.deal(victim, 1 ether);
		uint256 races = 10;
		for (uint256 i; i < races; ++i) {
			uint256 t = commitTimeOf[live] + 1;
			_at(t, 0);
			(bytes[] memory data, uint256 fee) = _payloadAt(t);
			vm.prank(victim);
			OrderV3Types.ExecutionResult memory r = router.executeOrder{value: fee}(live, data);
			if (r.status == OrderV3Types.LifecycleStatus.Executed) {
				closedAt = t;
				break;
			}
			assertEq(uint8(r.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "race: the owner's own execution fails on the ceiling");
			assertEq(uint8(_status()), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched), "race: relatched");
			if (firstFailAt == 0) {
				firstFailAt = t;
			}
			_attackerRetries(t);
			_at(t, 2);
			_ownerAmends();
		}
		_logOutcome("EOA RACE");
		assertEq(amendSuccesses, 0, "race: no owner amend succeeded");
		assertEq(amendReverts, races, "race: every amend after a won execution race reverted");
		assertEq(_positionSize(victim), size, "race: the position is still open");
		console2.log("EOA RACE: execution races won by the owner / amends that reverted", races, amendReverts);
	}
}
contract ProtectionRetryLockBatchingOwnerTest is ProtectionRetryLockCycles {
	BatchingOwnerAccount internal account;
	address internal accountOwner = makeAddr("accountOwner");
	function _price(uint256 t) internal pure override returns (uint256) {
		return _weekdayPrice(t);
	}
	function setUp() public {
		vm.prank(accountOwner);
		account = new BatchingOwnerAccount();
		victim = address(account);
		_weekdayScenario();
		vm.prank(attacker);
		attackerContract = new RetryAttacker();
		vm.deal(accountOwner, 1 ether);
	}
	function test_batchingOwnerWinsOneRace_escapes() public {
		for (uint256 i; i < 3; ++i) {
			require(!_attackerExecutesAndRetries(commitTimeOf[live] + 1), "batching: attacker cycle");
		}
		uint256 t = commitTimeOf[live] + 1;
		_at(t, 0);
		(bytes[] memory data, uint256 fee) = _payloadAt(t);
		uint64 old = live;
		vm.prank(accountOwner);
		uint64 n = account.executeAmendRetry{value: fee}(router, book, old, data, pid, _maxLimits());
		_record(n);
		_at(t, 1);
		vm.prank(attacker);
		(uint8 st,, uint64 gn) = attackerContract.executeAndRetry{value: fee}(router, book, old, data, pid);
		assertEq(st, 0, "batching: the attacker's execute of the old attempt does nothing");
		assertEq(gn, 0, "batching: the attacker cannot re-queue while the owner's amended attempt is live");
		_at(t + 1, 0);
		(data, fee) = _payloadAt(t + 1);
		vm.prank(keeper);
		OrderV3Types.ExecutionResult memory r = router.executeOrder{value: fee}(n, data);
		assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "batching: the owner's attempt fills");
		assertEq(_positionSize(victim), 0, "batching: closed");
		console2.log("BATCHING OWNER: attacker cycles before the owner's batch", attackerTxs);
		console2.log("BATCHING OWNER: closed, seconds after the first failure", (t + 1) - firstFailAt);
	}
}
contract ProtectionRetryLockWeekendTest is ProtectionRetryLockCycles {
	uint256 internal constant W_JUMP = T0 + 4 days + 3.5 hours;
	uint256 internal constant W_FAD = W_JUMP + 8 hours;
	uint256 internal constant W_FROZEN = W_JUMP + 8.5 hours;
	uint256 internal constant W_REOPEN = W_FROZEN + 48 hours;
	uint256 internal constant LOCK = 57 hours;
	uint256 internal liveExecs;
	uint256 internal fadExecs;
	uint256 internal closedExecs;
	uint256 internal reopenExecs;
	function _weekPre(uint256 h) internal pure returns (uint256) {
		int8[8] memory wob = [int8(0), 2, -1, 3, 1, -2, 2, 0];
		int256 p = int256(10_060 * 1e4) + int256((35 * 1e4 * h) / 96) + int256(wob[h % 8]) * 1e4;
		return uint256(p);
	}
	function _price(uint256 t) internal pure override returns (uint256) {
		if (t < W_FROZEN) {
			uint16[10] memory a = [uint16(10_125), 10_129, 10_126, 10_131, 10_134, 10_130, 10_133, 10_136, 10_132, 10_132];
			uint256 dt = t - W_JUMP;
			uint256 i = dt / 1 hours;
			uint256 x = uint256(a[i]) * 1e4;
			uint256 y = uint256(a[i + 1]) * 1e4;
			uint256 f = dt % 1 hours;
			return y >= x ? x + ((y - x) * f) / 1 hours : x - ((x - y) * f) / 1 hours;
		}
		require(t >= W_REOPEN, "no Pyth FX price exists while the market is closed");
		uint16[4] memory s = [uint16(10_140), 10_143, 10_138, 10_141];
		uint256 d2 = t - W_REOPEN;
		uint256 j = d2 / 1 hours;
		if (j > 2) {
			return uint256(s[3]) * 1e4;
		}
		uint256 u = uint256(s[j]) * 1e4;
		uint256 v = uint256(s[j + 1]) * 1e4;
		uint256 g = d2 % 1 hours;
		return v >= u ? u + ((v - u) * g) / 1 hours : u - ((u - v) * g) / 1 hours;
	}
	function _prevPublish(uint256 t) internal pure override returns (uint256) {
		return t == W_REOPEN ? W_FROZEN - 1 : t - 1;
	}
	function _afterExecution() internal override {
		bool f = engine.isFadWindow();
		bool z = engine.isOracleFrozen();
		uint256 t = vm.getBlockTimestamp();
		if (z) { ++closedExecs;
		} else if (t >= W_REOPEN) { ++reopenExecs;
		} else if (f) { ++fadExecs;
		} else { ++liveExecs;
		}
	}
	function setUp() public {
		_bootMarket();
		_openAndProtectVictim();
		for (uint256 h = 1; h <= 96; ++h) {
			_warp(1 hours - 1);
			_setPrice(_weekPre(h));
		}
		assertFalse(engine.isFadWindow(), "Friday 13:00 UTC is outside FAD");
		assertEq(uint8(_status()), uint8(PositionProtectionTypes.PositionProtectionStatus.Armed), "armed");
		_at(W_JUMP, 0);
		basket = _price(W_JUMP);
		lastPublish = W_JUMP;
		bytes[] memory data = _pythData(basket, W_JUMP, W_JUMP - 1);
		uint256 fee = oracle.getUpdateFee(data);
		vm.prank(keeper);
		live = IPositionProtectionActions(book).triggerPositionProtection{value: fee}(pid, data);
		_record(live);
		_keeperExecutes(W_JUMP + 1);
		v0 = _usdgOf(victim);
		curT = firstFailAt;
		nextActor = 1;
		lockEnd = firstFailAt + LOCK;
		midSample = 32.5 hours;
	}
	function beforeTestSetup(bytes4 testSelector) public view returns (bytes[] memory calls) {
		if (testSelector == this.test_C_weekend_keeperPresent_attackerBackruns_57h.selector) {
			calls = _chunks(LOCK / CHUNK);
		}
	}
	function test_C_weekend_keeperPresent_attackerBackruns_57h() public {
		_logOutcome("C");
		assertEq(amendSuccesses, 0, "C: no owner amend succeeded while the attacker was active");
		assertTrue(stopped && nextActor == 0, "C: the lock ran to its end");
		_requireLive();
		assertEq(_positionSize(victim), size, "C: the position is still open at the end of the lock");
		_withdrawDuringLock("C", true);
		uint256 frozen = _positionMargin(victim);
		console2.log("C: attacker back-run retries", attackerTxs);
		console2.log("C: owner amends sent after them / reverted ProtectionNotLatched", amendReverts, amendReverts);
		console2.log("C: keeper executions that failed and relatched", keeperExecs);
		console2.log("C: of which in the frozen market (no Pyth payload, Engine mark replay)", frozenExecs);
		console2.log("C: lock executions by regime: live Friday / FAD", liveExecs, fadExecs);
		console2.log("C: lock executions by regime: closed / after the reopen", closedExecs, reopenExecs);
		console2.log("C: transaction boundaries checked", boundaryChecks);
		_logGas("C");
		uint256 costWei = _reportCost("C", attackerTxs, _retryTxGas(), 0, frozen);
		uint256 fastTxs = attackerTxs + 3 * frozenExecs;
		console2.log("C, keeper in every block while frozen: attacker transactions", fastTxs);
		uint256 fastWei = fastTxs * _retryTxGas() * GAS_PRICE_WEI;
		console2.log("C, keeper in every block while frozen: attacker cost, US cents at 0.02 gwei", (fastWei * USD_CENTS_PER_ETH) / 1e18);
		console2.log("C, keeper in every block while frozen: frozen / cost at 0.02 gwei (integer)", (frozen * 1e18) / (fastWei * USD_CENTS_PER_ETH * 1e4));
		console2.log("C, keeper in every block while frozen: smallest margin for 100:1 (USDG, 6 dec)", (100 * 1e4 * USD_CENTS_PER_ETH * fastWei + 1e18 - 1) / 1e18);
		uint256 amendAt = _attackerStopsOwnerExits("C");
		uint256 lockedFor = amendAt - firstFailAt;
		console2.log("C: seconds from the first failure (Fri 13:30:01 UTC) to the owner's first successful amend", lockedFor);
		_withdrawAllAfterExit("C");
		assertGe(lockedFor, 49 hours, "C: the owner could not amend for >= 49 h");
		assertEq(lockedFor, LOCK, "C: Fri 13:30:01 -> Sun 22:30:01 UTC");
		assertEq(chunksRun, 57, "C: 57 committed one-hour transactions");
		assertEq(attackerTxs, LOCK, "C: one back-run per second");
		assertEq(amendReverts, attackerTxs, "C: the owner's amend reverted after every attacker transaction");
		assertEq(keeperExecs, attackerTxs + 1, "C: every keeper execution failed and relatched");
		assertEq(frozenExecs, 48 hours, "C: one keeper execution per second through the closure, no Pyth payload");
		assertEq(closedExecs, 48 hours, "C: the lock covered the whole closure");
		assertEq(fadExecs, 30 minutes, "C: the lock covered Friday's FAD half hour");
		assertGt(liveExecs, 0, "C: the lock started in the live market");
		assertGt(reopenExecs, 0, "C: the lock continued after the Sunday reopen");
		assertGt(costWei, 0, "C: cost recorded");
	}
}
contract ProtectionRetryLockBasketReturnTest is ProtectionRetryLockCycles {
	uint256 internal constant HOLD = 10 minutes;
	uint256 internal constant T_RETURN = T_JUMP + 1 + HOLD;
	uint256 internal constant RETURN_BASKET = 1.0104e8;
	function _price(uint256 t) internal pure override returns (uint256) {
		return t >= T_RETURN ? RETURN_BASKET : _weekdayPrice(t);
	}
	function setUp() public {
		_weekdayScenario();
		_keeperExecutes(T_JUMP + 1);
		v0 = _usdgOf(victim);
		curT = firstFailAt;
		nextActor = 1;
		lockEnd = T_RETURN;
	}
	function _fillPrice(uint64 id) internal view returns (uint256 price) {
		Vm.Log[] memory logs = vm.getRecordedLogs();
		bytes32 sig = keccak256("OrderExecuted(uint64,uint256)");
		for (uint256 i; i < logs.length; ++i) {
			if (logs[i].emitter == address(router) && logs[i].topics.length == 2 && logs[i].topics[0] == sig && logs[i].topics[1] == bytes32(uint256(id))) {
				price = abi.decode(logs[i].data, (uint256));
			}
		}
		require(price != 0, "no OrderExecuted event for the attempt");
	}
	function test_lockEndsWhenBasketReturnsInsideCeiling() public {
		while (!stopped) {
			this.step();
		}
		_logOutcome("RETURN");
		assertEq(amendSuccesses, 0, "RETURN: no owner amend succeeded while the attacker was active");
		assertEq(attackerTxs, 600, "RETURN: 600 back-runs in 10 minutes");
		assertEq(amendReverts, attackerTxs, "RETURN: every amend reverted");
		assertEq(keeperExecs, attackerTxs, "RETURN: every keeper execution before the return failed and relatched");
		_requireLive();
		assertEq(_positionSize(victim), size, "RETURN: the position is still open when the basket returns");
		console2.log("RETURN: attacker back-runs / owner amends that reverted ProtectionNotLatched", attackerTxs, amendReverts);
		uint256 t = commitTimeOf[live] + 1;
		assertEq(t, T_RETURN, "RETURN: the keeper's next execution is at the first second at the return level");
		_at(t, 0);
		(bytes[] memory data, uint256 fee) = _payloadAt(t);
		vm.recordLogs();
		vm.prank(keeper);
		OrderV3Types.ExecutionResult memory r = router.executeOrder{value: fee}(live, data);
		assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "RETURN: the keeper's execution fills");
		assertEq(uint8(_status()), uint8(PositionProtectionTypes.PositionProtectionStatus.Executed), "RETURN: the protection ends Executed");
		assertEq(_positionSize(victim), 0, "RETURN: the position is closed");
		uint256 fill = _fillPrice(live);
		assertGe(fill, RETURN_BASKET, "RETURN: the fill is the return level plus the adverse shift");
		assertLe(fill, CEIL, "RETURN: the fill is within the victim's 1.0105 ceiling");
		_at(t, 1);
		(bool ok, bytes4 sel) = _tryAs(attacker, book, abi.encodeCall(IPositionProtectionActions.retryPositionProtectionClose, (pid)));
		assertTrue(!ok && sel == IOrderRouterErrors.OrderRouter__ProtectionNotFound.selector, "RETURN: the attacker's next retry reverts ProtectionNotFound");
		console2.log("RETURN: the attacker's next retry reverted OrderRouter__ProtectionNotFound");
		console2.log("RETURN: basket from Tue 13:40:01 UTC (8 dec)", RETURN_BASKET);
		console2.log("RETURN: seconds from the first failure to the close", t - firstFailAt);
		console2.log("RETURN: fill price of the close (8 dec)", fill);
		_withdrawAllAfterExit("RETURN");
	}
}
```

