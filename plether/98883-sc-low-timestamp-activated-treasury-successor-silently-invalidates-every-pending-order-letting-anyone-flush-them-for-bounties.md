# \#98883 [SC-Low] Timestamp activated treasury successor silently invalidates every pending order, letting anyone flush them for bounties

**Submitted on Nov 10th 2026 at 18:18:47 UTC by @justadev for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98883
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

Plether pins every external order to an execution-config digest at commit time. `OrderLifecycleBook.currentExecutionConfigHash()` commits to `engine.protocolTreasury()` among its inputs. The `CfdTreasury` module added at commit `1209ca8a` (PR #130, 2026-10-27) lets the owner schedule a treasury successor that becomes effective at the current lock's expiry by `block.timestamp` alone, with no transaction and no event.

At that publicly announced second the digest changes. Every pending external order becomes `ConfigMismatch`-invalid while still inside its execution window. The permissionless `OrderRouter.clearInvalidOrders` path then lets anyone kill the entire pending queue in one transaction, with no ETH and no Pyth payload, and be credited each open order's reserved execution bounty.

All of this is in `packages/perps/src`.

## Vulnerability Details

Three things combine here, none of which is a bug on its own:

- The execution digest commits to the protocol treasury address.
- A scheduled treasury successor is resolved by timestamp, without a transaction or an activation event.
- Config-mismatch cleanup is permissionless, terminal inside the execution window, and pays the caller.

The relevant behavior at reviewed commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` is:

- `OrderLifecycleBook.currentExecutionConfigHash()` (`OrderLifecycleBook.sol:172-203`) encodes `engine.protocolTreasury()` at line 199 alongside the wiring, admin identities, active config versions, and mark staleness limit.
- `CfdTreasury._activeTreasury()` (`CfdTreasury.sol:101-106`), inherited by `CfdEngine`, returns the pending successor once `block.timestamp >= _currentTreasury.lockedUntil`. Storage only catches up on the next owner call. The natspec states that scheduled activation "is resolved by timestamp and emits no automatic event".
- Every other digest input changes only through an explicit owner transaction: one-time wiring, 72-hour timelocked `activeConfigVersion` finalizations, or the immediate treasury setters.
- `OrderLifecycleBook.sol:263` requires a fresh external request to pin the current nonzero digest; only Router-created protection close attempts use the zero unpinned marker.
- `OrderRecoverySidecar.clearInvalidOrders` (`OrderRecoverySidecar.sol:67-108`), reachable through `OrderRouter.clearInvalidOrders`, is permissionless and non-payable. It treats `pending.bounds.expectedConfigHash != currentExecutionConfigHash()` as terminal regardless of `executionDeadline`, and builds the cleanup request with `executor = msg.sender` (`OrderRecoverySidecar.sol:127`).
- `OrderRouterExecutionSidecar.sol:913-917` applies the same terminal `ConfigMismatch` classification on the ordinary execution path.
- `OrderRouter._settleTerminalBounty` (`OrderRouter.sol:314-347`) credits a nonzero reserved open-order bounty to the executor through `engine.creditBounty`.

The resulting sequence is:

- The owner locks treasury A and later calls `proposeProtocolTreasury(B, nextLockDuration)`. `ProtocolTreasuryProposed` publishes `activationTime`, up to 90 days ahead; `pendingProtocolTreasury()` returns it on demand.
- Trading goes on as normal. Each order pins the current digest and reserves its execution bounty.
- At `activationTime`, with no engine transaction, `currentExecutionConfigHash()` differs from every pending order's pinned value.
- An unprivileged caller submits `clearInvalidOrders(pendingIds)` with zero `msg.value`. Each pinned order is finalized `Failed / ConfigMismatch`, its reserved open bounty is credited to the caller, and its client id is consumed.
- Traders must re-commit under the rotated digest with fresh client ids and a second bounty reservation. A trader whose close got flushed is still exposed and may be liquidated before they can resubmit.

The caller does not hold any role, does not update the oracle, does not spend ETH, and does not interact with the owner. The owner's action is the documented, legitimate `proposeProtocolTreasury`. Once a proposal matures it cannot be cancelled, and the activation cannot be sequenced against an empty queue because its timestamp was fixed when the lock was set.

### Deterministic queue-flush demonstration

The PoC runs on the repository's own `BasePerpTest` fixture at the frozen commit, with the fixture router configuration (`openOrderExecutionBountyBps = 1`, minimum `0.01 USDG`, cap `0.20 USDG`, 60-second execution window, `maxPendingOrders = 5`):

- Treasury A locked for 86 days; successor B proposed.
- Three traders commit pinned opens of `10,000 USDG` notional five seconds before activation. Each reserves the `0.20 USDG` cap.
- At the activation second, no transaction has touched the engine and the digest differs from the pinned value.
- `block.timestamp` is at or below every order's `executionDeadline`.
- One `clearInvalidOrders` call from an unfunded outsider returns `cleared == 3`.
- Each `terminalOutcome` is `Failed` with reason `ConfigMismatch`.
- Outsider credited: `0.600000 USDG`
- Traders' aggregate settlement loss: `0.600000 USDG`
- Positions opened: `0`
- `engine.protocolTreasury()` already reports B.

A second test shows the ordinary keeper path, `executeOrder` on the head inside its window, returning `Failed / ConfigMismatch` and charging the trader identically. A third test shows that replaying the consumed client id reverts and that a fresh commit reserves another `0.20 USDG`.

### Deterministic liquidation differential

A fourth test measures the cost of a flushed risk-reducing close from one state snapshot:

- Position: LONG, `100,000 USDG` notional, `2,500 USDG` margin (40x), opened at `1.00000000`.
- Treasury A locked for 1 hour; successor B proposed.
- Mark moves to `1.01200000`; the position is healthy under the 1 percent maintenance regime.
- The trader commits a full close five seconds before activation.
- Branch A: a keeper executes the close inside its window. Trader settlement balance: `18,719.265734 USDG`.
- Branch B: the close is flushed at the activation second as `ConfigMismatch`; the mark reaches `1.02000000`; a keeper liquidates. Trader settlement balance: `17,859.745704 USDG`.
- Difference: `859.520030 USDG`, of which `100.000000 USDG` is the liquidation charge the voluntary close would have avoided and `759.520030 USDG` is the adverse move during the forced gap.

### Control

A fifth test repeats the queue-flush timing with no scheduled successor. `clearInvalidOrders` returns `cleared == 0`, every order remains `Pending`, and the head executes normally. This isolates the scheduled succession as the cause.

## Impact Details

The impact is **Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)**.

At a public, pre-announced second every in-flight external order is terminated with no execution route. Closes committed to reduce risk die without payout; opens pay their reserved bounty to the flushing caller. Every affected trader has to resubmit the same order under the new digest and pay a second bounty. No event is emitted at activation, so neither the owner nor monitoring learns that the digest rotated until orders fail.

The loss is bounded by:

- The router open-bounty cap per order (`0.20 USDG` in the fixture and the v2.0.0 deployment table).
- The number of pinned pending orders at the activation second (`maxPendingOrders` per account, queue-wide).
- For flushed closes, the liquidation charge (10 bps of notional at fixture parameters) plus the market move during the forced re-commit gap.

I am not claiming theft or frozen funds here. The bounty transfer is small and incidental; the harm is the simultaneous, unannounced termination of every pending order and the liquidation exposure it creates for traders who were exiting.

### Recommended remediation

Remove `engine.protocolTreasury()` from the execution digest; the treasury determines where fees are credited and does not alter execution semantics. Alternatively, require an explicit activation transaction for a scheduled successor that emits an event and can be sequenced against an empty queue, or make `clearInvalidOrders` treat a digest mismatch as non-terminal while the order is inside its execution window. Whatever the fix, an order that is still inside its window should never die because of a change nobody sent a transaction for.


## Link to Proof of Concept

https://gist.github.com/victorl2/5612c48b8dba598685dfc8c4f687680c

## Proof of Concept

## Proof of Concept

The linked secret Gist is a small Foundry project: one test file with five deterministic tests and a `foundry.toml` that points at a sibling clone of the competition repository. The in-scope contracts are compiled from that clone as they are; nothing in it is modified. The tests use the repository's own `BasePerpTest` fixture and its `MockPyth` for the out-of-scope Pyth dependency. No fork, RPC, or environment variable is needed.

- `test_Succession_OutsiderFlushesWholeQueueInsideWindowsAndCollectsEveryBounty`: three pinned opens are still inside their windows when an unfunded outsider kills all of them with one `clearInvalidOrders` call at the activation second. The outsider is credited `0.60 USDG`, the traders lose `0.60 USDG`, and no position exists.
- `test_Succession_ExecuteOrderInsideWindowFailsHeadAsConfigMismatch`: the normal keeper path ends the same way and charges the trader the same.
- `test_Succession_FlushedRiskReducingCloseLeavesPositionToBeLiquidated`: same state, two branches. If the close executes, the trader keeps `18,719.265734 USDG`. If it is flushed and the move continues, a keeper liquidates and the trader keeps `17,859.745704 USDG`. The gap is `859.520030 USDG`, `100 USDG` of it the liquidation charge.
- `test_Succession_RecoveryRequiresFreshIdentityAndSecondBounty`: the old client id can't be reused; the trader has to commit a fresh one and pay another bounty.
- `test_Control_NoSuccession_ClearInvalidOrdersClearsNothingAndOrdersExecute`: same timing with no successor scheduled. Nothing gets cleared and the head executes.

### Requirements

- Foundry 1.8.5 (the repository pin) with solc 0.8.35
- Git with submodule support
- The competition repository, default branch (`master`), cloned with submodules

### Setup / Test Execution

Clone the competition repository with submodules, clone the Gist next to it, and run the tests from the Gist directory:

```sh
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
git clone https://gist.github.com/5612c48b8dba598685dfc8c4f687680c.git plether-plt01-poc
cd plether-plt01-poc
forge test -vvv
```

The Gist's `foundry.toml` expects the competition clone next to it as `audit-comp-plether` and pulls the `BasePerpTest` fixture, libraries, and sources from there through remappings.

Note: the first time running `forge test` it compiles the in-scope sources and prints upstream compiler warnings
before the test output; those warnings belong to the frozen repository, not the PoC. 

Expected result:
```
Ran 5 tests for TreasurySuccessionQueueFlush.t.sol:TreasurySuccessionQueueFlushTest
[PASS] test_Control_NoSuccession_ClearInvalidOrdersClearsNothingAndOrdersExecute() (gas: 7504398)
Logs:
  [*] Control: treasury A locked for 86 days, NO successor proposed
  [*] Timestamp 1716965996 (activation - 5 s): mark refreshed at 1.00000000, digest 0xc7fbbd50...
  [*] Alice commits pinned LONG open #1 (10,000 USDG notional), deadline 1716966056, bounty reserved 0.200000 USDG
  [*] Bob commits pinned LONG open #2 (10,000 USDG notional), deadline 1716966056, bounty reserved 0.200000 USDG
  [*] Carol commits pinned LONG open #3 (10,000 USDG notional), deadline 1716966056, bounty reserved 0.200000 USDG
  [*] Timestamp 1716966002 (lock expired): digest 0xc7fbbd50... unchanged
  [*] Outsider calls clearInvalidOrders on all ids: cleared = 0
  [*] All three orders remain Pending inside their windows
  [*] Keeper executes head #1: status Executed, Alice position size 10000 units
  [*] Control result: without a scheduled successor nothing is cleared and execution works

[PASS] test_Succession_ExecuteOrderInsideWindowFailsHeadAsConfigMismatch() (gas: 5906195)
Logs:
  [*] Scenario: ordinary keeper execution after the silent rotation
  [*] Owner locks treasury A for 86 days; lock expires at timestamp 1716966001
  [*] Owner proposes successor B; pendingProtocolTreasury() publishes activation at 1716966001 for 0x00000000000000000000000000000000000FeE91
  [*] Timestamp 1716965996 (activation - 5 s): mark refreshed at 1.00000000, digest 0xc7fbbd50...
  [*] Alice commits pinned LONG open #1 (10,000 USDG notional), deadline 1716966056, bounty reserved 0.200000 USDG
  [*] Bob commits pinned LONG open #2 (10,000 USDG notional), deadline 1716966056, bounty reserved 0.200000 USDG
  [*] Carol commits pinned LONG open #3 (10,000 USDG notional), deadline 1716966056, bounty reserved 0.200000 USDG
  [*] Timestamp 1716966001 (activation): keeper executes head with a fresh Pyth update
  [*] executeOrder(#1): status Failed / ConfigMismatch
  [*] Alice balance 20000.000000 USDG -> 19999.800000 USDG, position size 0
  [*] Result: the keeper path cannot execute the pinned order either; the trader is charged for nothing

[PASS] test_Succession_FlushedRiskReducingCloseLeavesPositionToBeLiquidated() (gas: 8757899)
Logs:
  [*] Scenario: a trader's risk-reducing close is flushed at the activation second
  [*] Alice opens LONG 100000 units (100,000 USDG notional) with 2500.000000 USDG margin at 1.00000000
  [*] Owner locks treasury A for 1 hour and proposes successor B; activation published at 1709539202
  [*] Timestamp 1709539197 (activation - 5 s): mark moves against the LONG to 1.01200000; previewLiquidation -> healthy
  [*] Alice commits full market close #2 pinned to digest 0xc7fbbd50..., deadline 1709539257
  [*] --- Branch A: a keeper executes the close before activation ---
  [*] Timestamp 1709539201: close #2 Executed at 1.01200000, position size 0, Alice balance 18719.265734 USDG
  [*] --- Branch B: nobody executes before activation; the close is flushed ---
  [*] Timestamp 1709539202 (activation): digest 0xc7fbbd50... -> 0x34b5a5b8..., close still inside its window
  [*] Outsider flushes close #2 with msg.value 0: Failed / ConfigMismatch; Alice's position (100000 units) is still open
  [*] Timestamp 1709539203: mark reaches 1.02000000; previewLiquidation -> liquidatable, charge 100.000000 USDG
  [*] Keeper liquidates Alice: position size 0, Alice balance 17859.745704 USDG
  [*] Branch A (close executed) Alice balance:            18719.265734 USDG
  [*] Branch B (close flushed, liquidated) Alice balance: 17859.745704 USDG
  [*] Liquidation charge the voluntary close would have avoided: 100.000000 USDG
  [*] Total extra loss caused by the flush: 859.520030 USDG
  [*] Result: the flushed close turned a voluntary exit into a keeper liquidation

[PASS] test_Succession_OutsiderFlushesWholeQueueInsideWindowsAndCollectsEveryBounty() (gas: 6943890)
Logs:
  [*] Scenario: scheduled treasury succession flushes every pinned pending order
  [*] Owner locks treasury A for 86 days; lock expires at timestamp 1716966001
  [*] Owner proposes successor B; pendingProtocolTreasury() publishes activation at 1716966001 for 0x00000000000000000000000000000000000FeE91
  [*] Timestamp 1716965996 (activation - 5 s): mark refreshed at 1.00000000, digest 0xc7fbbd50...
  [*] Alice commits pinned LONG open #1 (10,000 USDG notional), deadline 1716966056, bounty reserved 0.200000 USDG
  [*] Bob commits pinned LONG open #2 (10,000 USDG notional), deadline 1716966056, bounty reserved 0.200000 USDG
  [*] Carol commits pinned LONG open #3 (10,000 USDG notional), deadline 1716966056, bounty reserved 0.200000 USDG
  [*] Outsider 0x0000000000000000000000000000000000000A77 holds 0.000000 USDG in the clearinghouse and sends no ETH
  [*] Timestamp 1716966001 (activation): no engine transaction, protocolTreasury() now 0x00000000000000000000000000000000000FeE91
  [*] Digest before 0xc7fbbd50... -> after 0x34b5a5b8... (rotated by clock)
  [*] Every pending order is still inside its 60-second execution window
  [*] Outsider calls clearInvalidOrders([1,2,3]) with msg.value 0 and no Pyth data: cleared = 3
  [*] Order #1 (Alice): Failed / ConfigMismatch, balance 20000.000000 USDG -> 19999.800000 USDG, no position
  [*] Order #2 (Bob): Failed / ConfigMismatch, balance 20000.000000 USDG -> 19999.800000 USDG, no position
  [*] Order #3 (Carol): Failed / ConfigMismatch, balance 20000.000000 USDG -> 19999.800000 USDG, no position
  [*] Outsider credited: 0.600000 USDG
  [*] Traders' aggregate loss: 0.600000 USDG
  [*] Router open-bounty config: 1 bps, min 0.010000 USDG, cap 0.200000 USDG
  [*] Result: whole queue terminally failed inside its windows by an unprivileged, unfunded caller

[PASS] test_Succession_RecoveryRequiresFreshIdentityAndSecondBounty() (gas: 8141714)
Logs:
  [*] Scenario: what a flushed trader has to do to recover
  [*] Owner locks treasury A for 86 days; lock expires at timestamp 1716966001
  [*] Owner proposes successor B; pendingProtocolTreasury() publishes activation at 1716966001 for 0x00000000000000000000000000000000000FeE91
  [*] Timestamp 1716965996 (activation - 5 s): mark refreshed at 1.00000000, digest 0xc7fbbd50...
  [*] Alice commits pinned LONG open #1 (10,000 USDG notional), deadline 1716966056, bounty reserved 0.200000 USDG
  [*] Bob commits pinned LONG open #2 (10,000 USDG notional), deadline 1716966056, bounty reserved 0.200000 USDG
  [*] Carol commits pinned LONG open #3 (10,000 USDG notional), deadline 1716966056, bounty reserved 0.200000 USDG
  [*] Timestamp 1716966001: outsider flushes the queue, cleared = 3
  [*] Alice re-commits with her original client id under the new digest: reverts (id consumed)
  [*] Alice commits a fresh client id: order #4 appended at the tail, second bounty reserved 0.200000 USDG
  [*] Result: recovery costs a new identity and a second bounty for the same intent

Suite result: ok. 5 passed; 0 failed; 0 skipped; finished in 17.46ms (36.84ms CPU time)

Ran 1 test suite in 17.84ms (17.46ms CPU time): 5 tests passed, 0 failed, 0 skipped (5 total tests)
```

Balances are in 6-decimal USDG units. Gas and timing figures are omitted from the block above and may differ by a few units between runs.

### What to look for

- The digest changes across the activation timestamp with no transaction touching the engine.
- `block.timestamp <= executionDeadline` for every order at the moment of the flush.
- `clearInvalidOrders` returns `cleared == 3` in the flush test and `cleared == 0` in the control.
- Every flushed order's `terminalOutcome` is `Failed` with reason `ConfigMismatch`.
- The outsider's clearinghouse balance increases by exactly the sum of the reserved bounties, and the traders' balances decrease by the same total.
- In the liquidation differential, `engineLens.previewLiquidation` reports the position healthy at `1.012` and liquidatable at `1.02`; the flushed branch ends with the position wiped and the trader `859.520030 USDG` below the executed-close branch.

### Cheatcodes and mocks

- `vm.warp` advances to the publicly announced activation timestamp; `vm.roll` advances one block.
- `vm.prank(engine.owner())` makes the documented `proposeProtocolTreasury` call; the fixture test contract is the engine owner and calls `setProtocolTreasury(TREASURY_A, 86 days)` (`1 hours` in the liquidation test).
- `vm.prank(<trader>)` commits ordinary orders; `vm.prank(OUTSIDER)` runs the flush from an address with no funds and no role.
- `vm.snapshotState` / `vm.revertToState` run the executed-close and flushed-close branches from the same pre-activation state with the same mock price path.
- `MockPyth` is the repository's mock for the out-of-scope Pyth dependency; the flush itself consumes no oracle data.
- Amounts come from the fixture router config. Production bounty settings may differ, which changes the per-order amount, not the mechanism.

