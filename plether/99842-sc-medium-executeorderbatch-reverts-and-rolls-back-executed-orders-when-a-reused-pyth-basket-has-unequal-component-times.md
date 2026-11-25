# \#99842 [SC-Medium] executeOrderBatch reverts and rolls back executed orders when a reused Pyth basket has unequal component times

**Submitted on Nov 15th 2026 at 12:43:43 UTC by @Audittens for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99842
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
`executeOrderBatch` caches the first historical Pyth basket it parses and reuses it for later orders in the same call. Before reusing it, the oracle requires every feed stored in Pyth to be at least as new as the basket's newest component. But the parse that created the basket only brings each feed to its own tick. So when the components of a valid basket have different publish times and Pyth storage holds nothing newer for the lagging feeds, the reuse check fails for them with `PletherOracle__StoredFeedBehind`, and the router re-raises the error. The whole batch reverts, including the orders that were already executed in it. Any two orders committed in the same second are enough, so anyone can put a victim's order into such a pair with one commit, and every batch covering it keeps reverting until the orders are executed one by one, Pyth storage holds newer prices for the lagging feeds, or the orders expire.

## Vulnerability Details
The fresh path in `PletherOracle._resolveHistoricalOrderBasket` parses the first tick of every feed after the commit, writes the payload to Pyth storage and checks each feed against its own parsed time:

```solidity
pyth.updatePriceFeeds{value: pythFee}(pythUpdateData);
for (uint256 i; i < parsedFeeds.length; ++i) {
    _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);
}
```

The batch cache keeps only two times of that basket: `publishTime` (the newest component) and `oldestPublishTime`. The reuse path in `updateBatchOrderExecutionPrice` then checks every feed against the newest one:

```solidity
if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
    for (uint256 i; i < pythFeedIds.length; ++i) {
        _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
    }
```

The oracle accepts component times up to `maxComponentPublishTimeDivergence` apart (5 seconds by default). If they are not all equal, every feed that ticked earlier than the newest one is stored below `cache.publishTime`, and `_requireStoredFeedCoverage` reverts. The only exception is a feed for which Pyth storage already held a price at least as new as the newest component before the batch.

The router cannot recover from this. Its own `_canReuseHistoricalBatchBasket` checks only the timing conditions, so it forwards zero fee and is committed to the reuse branch. `_prepareBatchOracle` tolerates only `PletherOracle__HistoricalParserFailure` and re-raises everything else, so the transaction reverts and the earlier items of the batch are rolled back.

The rejected data is valid. The reused basket is exactly what a fresh parse would return for the second order, and the same payload executes both orders through `executeOrder` (see the PoC). ORACLE_SYNCHRONIZATION.md says that when reuse conditions are not met "the Router requests a new historical resolution", and that the fill is valid when "each stored feed timestamp covers its corresponding parsed component". The project's own reuse test with unequal component times (`test_BatchReusesBasketWhenEveryComponentFollowsBothCommits`) passes only because it first writes a newer payload into Pyth storage.

### Scenario
1. Alice commits an order at time `T`.
2. Mallory (or any other trader) commits an order in a later block with the same timestamp `T`.
3. The first Pyth ticks after `T` are at `T+1` for five feeds and at `T+3` for one feed. This is a valid basket.
4. A keeper calls `executeOrderBatch` for both orders with this payload. Order 1 is executed with a fresh parse. For order 2 the cache is reused, the first feed is stored at `T+1 < T+3`, and the call reverts with `StoredFeedBehind`. Order 1's execution is rolled back.
5. Every retry with the same range fails the same way. The orders can only be executed one by one, each with its own parse and Pyth fee, or with a payload that also contains newer updates for the lagging feeds.

The same happens when the second order is committed later than the first but before the earliest component tick.

## Impact Details
* A batch that contains such a pair reverts completely. Orders executed earlier in the same batch are rolled back and the keeper's gas is wasted on every retry.
* If the pair is at the head of the FIFO queue, every batch starting from the head fails until someone executes the two orders separately or they expire (60 second window). Orders that are not executed in time expire: the trader's open or close does not happen and an opening bounty goes to whoever cleans up.
* Anyone can create the pair for a chosen order with one commit in the same second. The attacker does not control the Pyth component times, so the attack works only when they differ, which the project expects to happen (SECURITY.md mentions "feeds that publish at different times").
* The batch passes when Pyth storage already holds, for every feed, a price at least as new as the basket's newest component, for example after another update of the same feeds, or when the keeper adds such updates to the payload (`test3` in the PoC). So the failure needs storage that holds only the basket's own ticks for the lagging feeds.
* No price is wrong and no funds are moved by the revert itself. Single `executeOrder` is not affected.

## References
* Reuse check against the newest component time: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L262-L266
* Fresh path checks each feed against its own time: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L613-L626
* Cache stores only the newest and the oldest time: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L288-L297
* Router re-raises the error: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1009-L1030
* Router reuse predicate without a coverage condition: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1074-L1093
* Docs: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md?plain=1#L46-L63

## Proof of Concept

The following PoC uses the project's six-feed oracle test fixture (`OracleSynchronizationTestBase`, Pyth is mocked). Two orders are committed in the same second and the payload contains the first tick after the commit for every feed. It contains three tests:

* `test1_unequalComponentTimes_batchReverts` -- five feeds tick at `T+1` and one at `T+3`. `executeOrderBatch` reverts with `StoredFeedBehind` and nothing is executed. Then the same payload executes both orders through `executeOrder`.
* `test2_equalComponentTimes_batchExecutesBoth` -- all feeds tick at `T+1`. The batch executes both orders with one parse.
* `test3_unequalComponentTimes_newerStoredPrices_batchExecutesBoth` -- the payload of `test1`, but Pyth storage already holds a newer price for every feed. The batch executes both orders.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {OracleSynchronizationTestBase} from "../shared/OracleSynchronizationTestBase.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console} from "forge-std/console.sol";

/// Uses the project's six-feed oracle fixture (OracleSynchronizationTestBase). Pyth itself is mocked.
contract PoC_BatchReuseStoredFeedBehind is OracleSynchronizationTestBase {

    function test1_unequalComponentTimes_batchReverts() public {
        (uint64 commit, bytes[] memory payload) = _twoOrdersAndPayload(3);

        // The batch executes order 1, then reverts on order 2 and rolls everything back.
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector,
                _synchronizationFeedIds()[0],
                commit + 1,
                commit + 3
            )
        );
        router.executeOrderBatch{value: 4 * FEE}(2, payload);
        console.log(
            "executeOrderBatch reverted with StoredFeedBehind, next order to execute: %s", router.nextExecuteId()
        );

        // The same payload is valid for each order when they are executed one by one.
        OrderV3Types.ExecutionResult memory result = router.executeOrder{value: 2 * FEE}(1, payload);
        console.log("executeOrder(1) status: %s (2 = Executed)", uint8(result.status));
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        result = router.executeOrder{value: 2 * FEE}(2, payload);
        console.log("executeOrder(2) status: %s (2 = Executed)", uint8(result.status));
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
    }

    function test2_equalComponentTimes_batchExecutesBoth() public {
        (, bytes[] memory payload) = _twoOrdersAndPayload(1);

        OrderV3Types.BatchResult memory result = router.executeOrderBatch{value: 4 * FEE}(2, payload);
        console.log("executeOrderBatch executed orders: %s", result.terminalCount);
        console.log("Historical parses used: %s", baseMockPyth.parseUniqueCallCount());
        assertEq(result.terminalCount, 2);
    }

    /// The same unequal payload passes when Pyth storage already holds a newer price for every feed, for example
    /// after any other update of these feeds.
    function test3_unequalComponentTimes_newerStoredPrices_batchExecutesBoth() public {
        (uint64 commit, bytes[] memory payload) = _twoOrdersAndPayload(3);
        baseMockPyth.updatePriceFeeds{value: FEE}(_payload(commit + 4, commit + 3));

        OrderV3Types.BatchResult memory result = router.executeOrderBatch{value: 4 * FEE}(2, payload);
        console.log("executeOrderBatch executed orders: %s", result.terminalCount);
        assertEq(result.terminalCount, 2);
    }

    /// Alice and Bob commit in the same second (different blocks). The payload holds the first tick after the commit
    /// for every feed: five feeds at `commit + 1` and the last feed at `commit + lastFeedDelay`.
    function _twoOrdersAndPayload(
        uint64 lastFeedDelay
    ) internal returns (uint64 commit, bytes[] memory payload) {
        commit = uint64(block.timestamp);
        _commit(ALICE, 0);
        vm.roll(vm.getBlockNumber() + 1);
        _commit(BOB, 0);
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(commit + 5);

        payload = _payload(commit + 1, commit);
        MockPyth.FeedUpdate[] memory updates = abi.decode(payload[0], (MockPyth.FeedUpdate[]));
        updates[5].price.publishTime = commit + lastFeedDelay;
        payload[0] = abi.encode(updates);
    }

}
```

Save this code to the file `packages/perps/test/perps/poc/PoC_BatchReuseStoredFeedBehind.t.sol` and run `forge test --root packages/perps --mc PoC_BatchReuseStoredFeedBehind -vv`. It will print the following logs:

```
Ran 3 tests for test/perps/poc/PoC_BatchReuseStoredFeedBehind.t.sol:PoC_BatchReuseStoredFeedBehind
[PASS] test1_unequalComponentTimes_batchReverts() (gas: 5973853)
Logs:
  executeOrderBatch reverted with StoredFeedBehind, next order to execute: 1
  executeOrder(1) status: 2 (2 = Executed)
  executeOrder(2) status: 2 (2 = Executed)

[PASS] test2_equalComponentTimes_batchExecutesBoth() (gas: 4231872)
Logs:
  executeOrderBatch executed orders: 2
  Historical parses used: 1

[PASS] test3_unequalComponentTimes_newerStoredPrices_batchExecutesBoth() (gas: 4252808)
Logs:
  executeOrderBatch executed orders: 2

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 60.62ms (65.82ms CPU time)
```

In `test1` the batch reverts with `StoredFeedBehind(feed 1, T+1, T+3)` and order 1 is still the next order to execute, although the same payload is accepted for both orders one by one. In `test2` the only difference is that all component times are equal, and the batch executes both orders. In `test3` the payload is the one from `test1`, and the batch executes both orders because Pyth storage was already newer.
