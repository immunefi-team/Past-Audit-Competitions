# \#98857 [SC-Medium] Incorrect Timestamp Check in `PletherOracle.updateBatchOrderExecutionPrice()` Causes `executeOrderBatch()` to Revert on Multi-Feed Baskets, Breaking Batch Execution and Griefing Queued Orders

**Submitted on Nov 10th 2026 at 15:49:46 UTC by @n0fr33w1f14u for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98857
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description


### Description

## Brief/Intro

`PletherOracle.updateBatchOrderExecutionPrice()` is designed to allow `OrderRouter.executeOrderBatch()` to reuse a validated historical Pyth basket across consecutive orders in the global FIFO queue without paying redundant Pyth verification fees. However, when verifying stored feed coverage on a reused basket, `updateBatchOrderExecutionPrice()` checks that every component feed in Pyth storage has a `publishTime` at least equal to `cache.publishTime` (the newest component timestamp across all 6 FX feeds, `maxPublishTime`) instead of each feed's own timestamp or `cache.oldestPublishTime` (`minPublishTime`). Because the 6 FX feeds publish asynchronously (permitted to diverge by up to `maxComponentPublishTimeDivergence = 5` seconds), any feed with a timestamp older than `maxPublishTime` triggers a `PletherOracle__StoredFeedBehind` revert on the second order in the batch. Because `OrderRouterExecutionSidecar._prepareBatchOracle()` does not catch `PletherOracle__StoredFeedBehind`, the entire `executeOrderBatch()` transaction reverts outside the per-item rollback boundary—undoing all preceding fills in the batch, wasting keeper gas, and blocking batch queue execution.

## Vulnerability Details

Plether's reference mark is a 6-currency FX basket (`EUR`, `JPY`, `GBP`, `CAD`, `SEK`, `CHF`). Because independent Pyth FX feeds do not all publish in the exact same second, `PletherOracle._computeBasketPriceFromFeeds()` allows component timestamps to diverge by up to `maxComponentPublishTimeDivergence` (`5` seconds by default) and records both the newest timestamp (`basket.publishTime`) and the oldest timestamp (`basket.oldestPublishTime`):

```solidity
// packages/perps/src/PletherOracle.sol
if (maxPublishTime > minPublishTime + maxPublishTimeDivergence) {
    revert PletherOracle__PublishTimeDivergence(mode, minPublishTime, maxPublishTime, maxPublishTimeDivergence);
}
...
basket.publishTime = uint64(maxPublishTime);
basket.oldestPublishTime = uint64(minPublishTime);
```

When `OrderRouter.executeOrderBatch()` processes the first order using a historical Pyth update, `PletherOracle.updateBatchOrderExecutionPrice()` calls `_resolveOrderExecutionBasket()`, which in turn calls `_resolveHistoricalOrderBasket()`. That function updates Pyth storage via `pyth.updatePriceFeeds(pythUpdateData)` and verifies stored coverage against each individual feed's own parsed timestamp (`parsedFeeds[i].price.publishTime`):

```solidity
// packages/perps/src/PletherOracle.sol
pyth.updatePriceFeeds{value: pythFee}(pythUpdateData);
for (uint256 i; i < parsedFeeds.length; ++i) {
    _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);
}
```

`updateBatchOrderExecutionPrice()` then returns `nextCache` to the router sidecar with `publishTime = basket.publishTime` (`maxPublishTime`) and `oldestPublishTime = basket.oldestPublishTime` (`minPublishTime`):

```solidity
// packages/perps/src/PletherOracle.sol
if (!policy.oracleFrozen && !reusedBasket) {
    nextCache = BatchOrderPriceCache({
        hasHistoricalBasket: true,
        minReusableCommitTime: request.commitTime,
        price: basket.price,
        confidence: basket.confidence,
        publishTime: basket.publishTime,
        oldestPublishTime: basket.oldestPublishTime
    });
}
```

When `executeOrderBatch()` advances to the second order in the batch that qualifies for historical basket reuse (`commitTime >= cache.minReusableCommitTime && commitTime < cache.oldestPublishTime`), `OrderRouterExecutionSidecar._prepareBatchOracle()` sets `fee = 0` and calls `oracle.updateBatchOrderExecutionPrice()` with the cached basket. Inside `PletherOracle.updateBatchOrderExecutionPrice()`, the cache-reuse branch validates stored feed coverage using `cache.publishTime` (`maxPublishTime`):

```solidity
// packages/perps/src/PletherOracle.sol
if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
    // Cache contents are caller-supplied; independently prove storage coverage, not price authenticity.
    for (uint256 i; i < pythFeedIds.length; ++i) {
        _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
    }
```

Where `_requireStoredFeedCoverage()` enforces:

```solidity
// packages/perps/src/PletherOracle.sol
function _requireStoredFeedCoverage(
    bytes32 feedId,
    uint256 requiredPublishTime
) internal view {
    uint256 storedPublishTime = pyth.getPriceUnsafe(feedId).publishTime;
    if (storedPublishTime < requiredPublishTime) {
        revert PletherOracle__StoredFeedBehind(feedId, storedPublishTime, requiredPublishTime);
    }
}
```

Unless Pyth on-chain storage was already updated to at least `maxPublishTime` for every single feed prior to the batch call, any component feed whose historical update timestamp is between `minPublishTime` and `maxPublishTime - 1` will have `storedPublishTime < cache.publishTime`. Consequently, `_requireStoredFeedCoverage()` reverts with `PletherOracle__StoredFeedBehind`.

In `OrderRouterExecutionSidecar._prepareBatchOracle()`, the `try/catch` block only catches `PletherOracle__HistoricalParserFailure` (selector + 64 bytes = 68 bytes) and re-reverts on all other errors:

```solidity
// packages/perps/src/OrderRouterExecutionSidecar.sol
} catch (bytes memory reason) {
    if (
        reason.length != 68
            || _selector(reason) != IPletherOracle.PletherOracle__HistoricalParserFailure.selector
    ) {
        assembly ("memory-safe") {
            revert(add(reason, 32), mload(reason))
        }
    }
    return (OrderV3Types.PendingReason.OracleFailure, result, state.oracleCache);
}
```

Because `_prepareBatchOracle()` executes in the outer batch loop—outside the per-order `try host.executeOrderItemFromSidecar` rollback frame—this revert aborts the entire `executeOrderBatch()` transaction and rolls back all earlier orders executed in the batch.

### Step-by-Step Walkthrough

1. **Order Commitment:**
   - At timestamp `1000`, Trader A commits `Order 1` (`commitTime = 1000`).
   - In the same second (`1000`) or at timestamp `1001`, Trader B commits `Order 2` (`commitTime = 1000` or `1001`).
2. **Historical Pyth Basket:**
   - A keeper fetches the unique post-commit Pyth update for the basket.
   - Five of the FX feeds have `publishTime = 1002`, and one feed (for example, `EUR/USD`) has `publishTime = 1003`.
   - The component publish-time divergence is `1003 - 1002 = 1` second, which is well within the allowed `maxComponentPublishTimeDivergence` of `5` seconds.
3. **First Order Execution in Batch (`Order 1`):**
   - The keeper calls `OrderRouter.executeOrderBatch(2, pythUpdateData)` at timestamp `1005`.
   - For `Order 1`, `_prepareBatchOracle()` calls `PletherOracle.updateBatchOrderExecutionPrice()` with an empty cache.
   - `_resolveHistoricalOrderBasket()` parses the feeds, updates Pyth storage (setting 5 feeds to `publishTime = 1002` and 1 feed to `1003`), and verifies each feed against its own `parsedFeeds[i].price.publishTime`. This succeeds.
   - `updateBatchOrderExecutionPrice()` returns `nextCache` with `oldestPublishTime = 1002` and `publishTime = 1003`.
   - `Order 1` is executed and finalized in its item frame.
4. **Second Order Cache Reuse & Revert (`Order 2`):**
   - The loop advances to `Order 2`.
   - `_canReuseHistoricalBatchBasket()` evaluates to `true` because `Order 2`'s `commitTime` (`1000` or `1001`) is at or after `minReusableCommitTime` (`1000`), strictly before `oldestPublishTime` (`1002`), and within `orderSettlementWindow` (`15` seconds).
   - `_prepareBatchOracle()` calls `oracle.updateBatchOrderExecutionPrice{value: 0}(..., state.oracleCache)`.
   - `updateBatchOrderExecutionPrice()` loops over all 6 `pythFeedIds` and calls `_requireStoredFeedCoverage(pythFeedIds[i], 1003)`.
   - On the very first feed whose stored `publishTime` is `1002`, the check `1002 < 1003` evaluates to true, and `PletherOracle` reverts with `PletherOracle__StoredFeedBehind(feedId, 1002, 1003)`.
5. **Transaction Rollback:**
   - `_prepareBatchOracle()` catches the revert bytes (`length == 100`), sees it does not match `PletherOracle__HistoricalParserFailure`, and re-reverts.
   - The entire `executeOrderBatch()` transaction reverts, rolling back `Order 1`'s execution and consuming the keeper's gas.

## Impact Details

1. **Griefing & Batch Rollback of Queued Orders (Medium — `Griefing`):**
   - `OrderRouter.executeOrderBatch()` is explicitly designed so that each order executes in an isolated rollback frame (`executeOrderItemFromSidecar`), ensuring a failure on a later order does not undo earlier finalized orders in the batch.
   - Because `PletherOracle__StoredFeedBehind` is thrown during `_prepareBatchOracle()` outside the item frame whenever cache reuse is triggered on a basket with non-zero component timestamp divergence, any second order committed within the same reusable window causes the entire batch transaction to revert.
   - An attacker (or ordinary concurrent user) can commit a minimum-size order alongside legitimate users' orders in the same timestamp window (`commitTime < oldestPublishTime`), causing any keeper calling `executeOrderBatch(maxOrderId, pythUpdateData)` across those orders to revert repeatedly, wasting keeper gas and preventing batch execution of the queue. If orders are not executed individually before `executionWindowSeconds` (`60` seconds default) elapses, they expire and users forfeit their reserved execution bounties.


## Proof of Concept

`test/poc/BatchCacheStoredFeedBehind.t.sol`
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {OracleSynchronizationTestBase} from
    "../../packages/perps/test/perps/shared/OracleSynchronizationTestBase.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice PoC: `updateBatchOrderExecutionPrice` checks stored coverage on cache reuse against `cache.publishTime`
///         (the NEWEST component time) instead of each component's own time / `cache.oldestPublishTime`.
///         With any (allowed, <= 5s) component divergence, the first order's own `updatePriceFeeds` leaves the
///         lagging feeds at their own (older) time, so the second order's reuse reverts `StoredFeedBehind`.
///         `_prepareBatchOracle` re-throws everything except `HistoricalParserFailure`, so the whole
///         `executeOrderBatch` reverts and the already-finalized first order is rolled back.
contract BatchCacheStoredFeedBehindPoC is OracleSynchronizationTestBase {

    /// Control: identical scenario, all components share a publish time -> both orders fill with one parse.
    function test_Control_NoDivergence_BatchReusesCache() public {
        uint64 commit = uint64(block.timestamp);
        _commit(ALICE, 0);
        _commit(BOB, 0);
        _advance(commit + 5);

        bytes[] memory historical = _payload(commit + 2, commit);

        OrderV3Types.BatchResult memory result = router.executeOrderBatch{value: 4 * FEE}(2, historical);
        assertEq(result.terminalCount, 2, "both orders fill");
        assertEq(baseMockPyth.parseUniqueCallCount(), 1, "second order reused the cached basket");
    }

    /// Bug: one component 1s newer than the other five (well within maxComponentPublishTimeDivergence = 5s).
    function test_PoC_ComponentDivergence_RevertsWholeBatch() public {
        uint64 commit = uint64(block.timestamp);
        uint64 aliceId = _commit(ALICE, 0);
        _commit(BOB, 0);
        _advance(commit + 5);

        // Five feeds at commit+2, EUR-like feed[0] at commit+3. Divergence 1s, valid basket.
        bytes[] memory historical = _payload(commit + 2, commit);
        MockPyth.FeedUpdate[] memory updates = abi.decode(historical[0], (MockPyth.FeedUpdate[]));
        updates[0].price.publishTime = commit + 3;
        historical[0] = abi.encode(updates);

        // Sanity: the same payload is perfectly valid for single-order execution.
        // (Checked via oracle helper on a snapshot so the batch below starts from clean state.)
        uint256 snap = vm.snapshotState();
        IPletherOracle.BatchOrderPriceCache memory empty;
        (bool ok,, IPletherOracle.BatchOrderPriceCache memory cache) = pletherOracle.updateBatchOrderExecutionPrice{
            value: 2 * FEE
        }(address(this), historical, _request(commit, false), empty);
        assertTrue(ok, "first order resolves from the historical basket");
        assertEq(cache.publishTime, commit + 3, "cache.publishTime = newest component");
        assertEq(cache.oldestPublishTime, commit + 2, "cache.oldestPublishTime = oldest component");
        assertEq(baseMockPyth.getPriceUnsafe(updates[1].id).publishTime, commit + 2, "lagging feed stored at own time");
        // Reuse for a second order with the same commit time reverts on the lagging feed.
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector, updates[1].id, commit + 2, commit + 3
            )
        );
        pletherOracle.updateBatchOrderExecutionPrice(address(this), new bytes[](0), _request(commit, false), cache);
        vm.revertToState(snap);

        // Router batch: order 1 fills inside its item frame, then order 2's reuse throws in _prepareBatchOracle
        // (outside the item frame) and the entire batch reverts.
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector, updates[1].id, commit + 2, commit + 3
            )
        );
        router.executeOrderBatch{value: 4 * FEE}(2, historical);

        // Order 1 was rolled back: queue head unchanged, Alice has no position.
        assertEq(router.nextExecuteId(), aliceId, "batch prefix rolled back");
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0, "Alice's fill undone");

        // Single-order execution of the same order with the same payload works -> failure is purely the reuse check.
        router.executeOrder{value: 2 * FEE}(aliceId, historical);
        (size,,,,,,) = engine.positions(ALICE);
        assertGt(size, 0, "same payload fills order 1 alone");
    }

}
```
