# \#99976 [SC-Medium] `PletherOracle.updateBatchOrderExecutionPrice` Checks `cache.publishTime` (`maxPublishTime`) Instead of Per-Feed / `oldestPublishTime` Coverage, Causing `executeOrderBatch` to Revert on Any Mixed-Timestamp Historical Basket

**Submitted on Nov 15th 2026 at 23:42:17 UTC by @hunraj for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99976
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description



## Executive Summary
To remediate **Cecuro MED-23** (*"Mixed-time basket can block trader protections during valid mark updates"*, commits `2fca7941` and `7de2f742`), `PletherOracle._computeBasketPriceFromFeeds` was updated so that `basket.publishTime` records the **newest** component timestamp (`maxPublishTime`), while `basket.oldestPublishTime` records the **oldest** component timestamp (`minPublishTime`).


However, when `PletherOracle.updateBatchOrderExecutionPrice` reuses a cached historical basket for a subsequent order in `OrderRouter.executeOrderBatch`, its storage-coverage loop (`_requireStoredFeedCoverage`) still validates every component feed in Pyth storage against `cache.publishTime` (`maxPublishTime`). Because Pyth FX feeds naturally publish at slightly different timestamps (explicitly permitted up to `maxComponentPublishTimeDivergence = 5` seconds), any component feed in the basket published earlier than `maxPublishTime` has `storedPublishTime < cache.publishTime`. As a result, the second order in the batch deterministically reverts with `PletherOracle__StoredFeedBehind`, which goes uncaught in `OrderRouterExecutionSidecar._prepareBatchOracle` and rolls back the entire `executeOrderBatch` transaction—including already-executed orders in the batch.


---


## Vulnerability Description
During live or FAD delayed-order execution, `OrderRouter.executeOrderBatch` processes consecutive FIFO orders. For the first order requiring a historical price, `OrderRouterExecutionSidecar._prepareBatchOracle` calls `PletherOracle.updateBatchOrderExecutionPrice`, which resolves the post-commit historical basket in `_resolveHistoricalOrderBasket`:


1. `pyth.parsePriceFeedUpdatesUnique` parses the first qualifying tick for each configured feed within `(commitTime, min(commitTime + orderSettlementWindow, block.timestamp)]`.
2. `_computeBasketPriceFromFeeds` allows individual component publish times to differ by up to `maxComponentPublishTimeDivergence` (5 seconds by default) and sets:
  * `basket.publishTime = uint64(maxPublishTime)`
  * `basket.oldestPublishTime = uint64(minPublishTime)`
3. `pyth.updatePriceFeeds(pythUpdateData)` updates Pyth on-chain storage with the supplied payload.
4. For this first order, `_resolveHistoricalOrderBasket` verifies storage coverage against each feed's **individual** parsed timestamp:
  ```solidity
  for (uint256 i; i < parsedFeeds.length; ++i) {
     _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);
  }
  ```
5. `updateBatchOrderExecutionPrice` then populates `nextCache` with `publishTime = basket.publishTime` (`maxPublishTime`) and `oldestPublishTime = basket.oldestPublishTime` (`minPublishTime`).


When the next order in `executeOrderBatch` is processed, `_canReuseHistoricalBatchBasket` checks whether the cached basket covers that order's `commitTime` (requiring `commitTime < cache.oldestPublishTime`). When it returns `true`:
* `OrderRouterExecutionSidecar._prepareBatchOracle` sets `fee = 0` and calls `oracle.updateBatchOrderExecutionPrice` with `state.oracleCache`.
* `PletherOracle.updateBatchOrderExecutionPrice` enters the cache-reuse branch (`PletherOracle.sol:217-221`):
 ```solidity
 if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
    // Cache contents are caller-supplied; independently prove storage coverage, not price authenticity.
    for (uint256 i; i < pythFeedIds.length; ++i) {
        _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
    }
 ```
* `_requireStoredFeedCoverage` reads `pyth.getPriceUnsafe(pythFeedIds[i]).publishTime` and reverts with `PletherOracle__StoredFeedBehind` if `storedPublishTime < cache.publishTime` (`maxPublishTime`).
* Whenever the historical basket had unequal component timestamps (`minPublishTime < maxPublishTime`), at least one feed in Pyth storage has `storedPublishTime == minPublishTime < cache.publishTime`.
* `_requireStoredFeedCoverage` reverts with `PletherOracle__StoredFeedBehind`. Because `OrderRouterExecutionSidecar._prepareBatchOracle` (`OrderRouterExecutionSidecar.sol:707-716`) only catches `PletherOracle__HistoricalParserFailure`, the revert bubbles up and aborts the entire `executeOrderBatch` transaction.


---


## Root Cause Analysis
In `packages/perps/src/PletherOracle.sol`:


```solidity
//
if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
   // Cache contents are caller-supplied; independently prove storage coverage, not price authenticity.
   for (uint256 i; i < pythFeedIds.length; ++i) {
       _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
   }
   basket = BasketPrice({
       price: cache.price,
       confidence: cache.confidence,
       publishTime: cache.publishTime,
       oldestPublishTime: cache.oldestPublishTime,
       pythFee: 0
   });
   reusedBasket = true;
}
```


And in `packages/perps/src/PletherOracle.sol`:
```solidity
//
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


Before commits `2fca7941` and `7de2f742`, `basket.publishTime` was `minPublishTime`, so every component feed was guaranteed to have `parsedFeeds[i].price.publishTime >= basket.publishTime`. When `basket.publishTime` was changed to `maxPublishTime` (and `oldestPublishTime` was introduced for `minPublishTime`), `_resolveHistoricalOrderBasket` (line 452) was updated to check `parsedFeeds[i].price.publishTime`, but `updateBatchOrderExecutionPrice` (line 220) was left checking `cache.publishTime` (`maxPublishTime`).


---


## Technical Walkthrough
1. Two traders (or one trader with two orders) commit `Order #1` and `Order #2` at `commitTime = 1709535600`.
2. In the post-commit window `(1709535600, 1709535615]`, the 6 FX feeds publish updates within the permitted 5-second `maxComponentPublishTimeDivergence`:
  * `FEED_A` publishes at `t = 1709535602`.
  * `FEED_B` publishes at `t = 1709535603`.
3. At `block.timestamp = 1709535605`, a keeper calls `router.executeOrderBatch(2, pythUpdateData)`.
4. **Order #1 Execution:**
  * `_canReuseHistoricalBatchBasket` is `false` (initial empty cache).
  * `_resolveHistoricalOrderBasket` parses the historical basket (`oldestPublishTime = 1709535602`, `publishTime = 1709535603`).
  * `pyth.updatePriceFeeds(pythUpdateData)` stores `1709535602` for `FEED_A` and `1709535603` for `FEED_B`.
  * `_requireStoredFeedCoverage` checks `FEED_A` against `1709535602` (passes) and `FEED_B` against `1709535603` (passes).
  * `Order #1` executes and finalizes in its item frame, and `state.oracleCache` is updated with `publishTime = 1709535603` and `oldestPublishTime = 1709535602`.
5. **Order #2 Execution:**
  * `_canReuseHistoricalBatchBasket` evaluates `commitTime (1709535600) < cache.oldestPublishTime (1709535602)` and returns `true`.
  * `updateBatchOrderExecutionPrice` enters the cache-reuse branch and calls `_requireStoredFeedCoverage(FEED_A, 1709535603)`.
  * `pyth.getPriceUnsafe(FEED_A).publishTime` is `1709535602 < 1709535603`.
  * `updateBatchOrderExecutionPrice` reverts with `PletherOracle__StoredFeedBehind(FEED_A, 1709535602, 1709535603)`.
  * `OrderRouterExecutionSidecar._prepareBatchOracle` re-throws the revert outside the item frame, rolling back the entire batch (including `Order #1`).


---


## Impact Assessment
* **Broken Batch Execution / Rollback of Earlier Batch Progress:** `OrderRouter.executeOrderBatch` deterministically reverts on the second order and rolls back all earlier orders processed in the same batch whenever a reused historical basket has unequal component publish times.
* **Forced Fallback to Uncached Single-Order Execution ($2\times$ Pyth Fees & $O(N)$ Transactions):** Because keepers cannot disable cache reuse inside `executeOrderBatch` when `_canReuseHistoricalBatchBasket` is `true`, keepers cannot batch-execute orders that share a mixed-time historical tick unless they either execute 1 order per transaction (`executeOrder` or `executeOrderBatch` with a 1-order bound, paying `2 * pythFee` and full base transaction gas on every single order instead of `0` fee on cached orders) or bundle extra newer Pyth VAAs to artificially advance Pyth storage.


---


## Likelihood Assessment
**High.** Real-world Pyth FX feeds (EUR/USD, USD/JPY, GBP/USD, USD/CAD, USD/SEK, USD/CHF) have independent publisher schedules and routinely publish 1–5 seconds apart within the protocol's `maxComponentPublishTimeDivergence = 5` second window. As noted in `ORACLE_SYNCHRONIZATION.md`, the existing unit test suite only missed this because its pinned mock fixture uses identical timestamps across all feeds.


---


## Supporting Evidence
* `packages/perps/src/PletherOracle.sol:` (`updateBatchOrderExecutionPrice` cache reuse loop checking `cache.publishTime`)
* `packages/perps/src/PletherOracle.sol:` (`_resolveHistoricalOrderBasket` checking `parsedFeeds[i].price.publishTime`)
* `packages/perps/src/PletherOracle.sol:` (`_requireStoredFeedCoverage`)
* `packages/perps/src/PletherOracle.sol:` (`_computeBasketPriceFromFeeds` setting `publishTime = maxPublishTime` and `oldestPublishTime = minPublishTime`)
* `packages/perps/src/OrderRouterExecutionSidecar.sol:695-720` (`_prepareBatchOracle` re-throwing `PletherOracle__StoredFeedBehind`)


---


## Step-by-Step Reproduction
1. Save the Proof of Concept below as `packages/perps/test/perps/spec/lp/BatchHistoricalCacheSkewRevert.t.sol`.
2. Run the test from the repository root:
  ```bash
  forge test --match-contract BatchHistoricalCacheSkewRevertTest -vvv
  ```
3. Observe that `test_BatchExecutionRevertsOnMixedTimestampHistoricalCacheReuse` passes: `Order #1` and `Order #2` are committed at the same timestamp, a valid historical basket with a 1-second component timestamp difference (`t0 + 2` and `t0 + 3`, within the 5-second `maxComponentPublishTimeDivergence`) is supplied, and `router.executeOrderBatch(2, pythUpdateData)` reverts with `PletherOracle__StoredFeedBehind(FEED_A, t0 + 2, t0 + 3)`.

---


## Security Recommendations
In `PletherOracle.updateBatchOrderExecutionPrice` (`packages/perps/src/PletherOracle.sol:217-221`), validate stored feed coverage against `cache.oldestPublishTime` (the minimum component timestamp guaranteed to be met by all feeds in the cached basket) or track per-component timestamps in `BatchOrderPriceCache`:


```diff
if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
    // Cache contents are caller-supplied; independently prove storage coverage, not price authenticity.
    for (uint256 i; i < pythFeedIds.length; ++i) {
-        _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
+        _requireStoredFeedCoverage(pythFeedIds[i], cache.oldestPublishTime);
    }
```


## Proof of Concept



## Proof of Concept


```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;


import {OrderRouterTestBase} from "../../shared/OrderRouterTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IPyth, PythStructs} from "@plether/shared/interfaces/IPyth.sol";


contract BatchHistoricalCacheSkewRevertTest is OrderRouterTestBase {
   /// @notice Proves that `executeOrderBatch` deterministically reverts with `PletherOracle__StoredFeedBehind`
   /// when reusing a valid historical basket whose component feeds have unequal publish times (`t0 + 2` and `t0 + 3`).
   function test_BatchExecutionRevertsOnMixedTimestampHistoricalCacheReuse() public {
       _startRecordingLogs();
       address trader1 = alice;
       address trader2 = bob;
       _fundTrader(trader1, 50_000e6);
       _fundTrader(trader2, 50_000e6);


       uint64 commitTime = uint64(vm.getBlockTimestamp());


       // 1. Two orders are committed in the same block/timestamp
       vm.prank(trader1);
       uint64 order1 = router.commitOrder(CfdTypes.Side.LONG, 10_000e18, 2_000e6, 1e8, false);
       vm.prank(trader2);
       uint64 order2 = router.commitOrder(CfdTypes.Side.LONG, 10_000e18, 2_000e6, 1e8, false);
       assertEq(order1, 1);
       assertEq(order2, 2);


       // 2. Advance into the post-commit settlement window (commitTime + 5s, within the 15s window)
       uint64 execTime = commitTime + 5;
       vm.warp(execTime);
       vm.roll(vm.getBlockNumber() + 1);


       // 3. Construct a valid historical basket with a 1-second component timestamp difference
       // (well within `maxComponentPublishTimeDivergence = 5` seconds):
       // - Feed 0 published at `commitTime + 2`
       // - Feed 1 published at `commitTime + 3`
       bytes32 feed0 = pletherOracle.pythFeedIds(0);
       bytes32 feed1 = pletherOracle.pythFeedIds(1);
       uint64 tFeed0 = commitTime + 2;
       uint64 tFeed1 = commitTime + 3;


       bytes32[] memory feedIds = new bytes32[](2);
       feedIds[0] = feed0;
       feedIds[1] = feed1;


       PythStructs.Price memory p0 =
           PythStructs.Price({price: int64(100_000_000), conf: uint64(0), expo: int32(-8), publishTime: tFeed0});
       PythStructs.Price memory p1 =
           PythStructs.Price({price: int64(100_000_000), conf: uint64(0), expo: int32(-8), publishTime: tFeed1});


       PythStructs.PriceFeed[] memory parsedFeeds = new PythStructs.PriceFeed[](2);
       parsedFeeds[0] = PythStructs.PriceFeed({id: feed0, price: p0, emaPrice: p0});
       parsedFeeds[1] = PythStructs.PriceFeed({id: feed1, price: p1, emaPrice: p1});


       // Store the synced feed states in MockPyth (Feed 0 at tFeed0, Feed 1 at tFeed1)
       baseMockPyth.setPrice(feed0, int64(100_000_000), int32(-8), tFeed0);
       baseMockPyth.setPrice(feed1, int64(100_000_000), int32(-8), tFeed1);


       bytes[] memory pythUpdateData = new bytes[](1);
       pythUpdateData[0] = hex"01";


       uint64 minPublishTime = commitTime + 1;
       uint64 maxPublishTime = execTime;


       // Mock Pyth's unique historical parser and storage update for this payload
       vm.mockCall(
           address(baseMockPyth),
           abi.encodeWithSelector(
               IPyth.parsePriceFeedUpdatesUnique.selector,
               pythUpdateData,
               feedIds,
               minPublishTime,
               maxPublishTime
           ),
           abi.encode(parsedFeeds)
       );
       vm.mockCall(
           address(baseMockPyth),
           abi.encodeWithSelector(IPyth.updatePriceFeeds.selector, pythUpdateData),
           ""
       );


       // 4. Calling `executeOrderBatch(2, pythUpdateData)` succeeds on Order #1 (which checks each feed
       // against its own `parsedFeeds[i].price.publishTime`), caches `publishTime = tFeed1` (maxPublishTime)
       // and `oldestPublishTime = tFeed0` (minPublishTime), and then REVERTS on Order #2 when cache reuse
       // demands `_requireStoredFeedCoverage(feed0, cache.publishTime)` (`tFeed0 < tFeed1`)!
       vm.expectRevert(
           abi.encodeWithSelector(
               IPletherOracle.PletherOracle__StoredFeedBehind.selector,
               feed0,
               uint256(tFeed0),
               uint256(tFeed1)
           )
       );
       router.executeOrderBatch(2, pythUpdateData);


       // Both Order #1 and Order #2 remain unexecuted because the uncaught revert rolled back the entire batch
       assertEq(router.nextExecuteId(), 1, "Order #1 progress was rolled back by Order #2's cache-reuse revert");
   }
}
```

