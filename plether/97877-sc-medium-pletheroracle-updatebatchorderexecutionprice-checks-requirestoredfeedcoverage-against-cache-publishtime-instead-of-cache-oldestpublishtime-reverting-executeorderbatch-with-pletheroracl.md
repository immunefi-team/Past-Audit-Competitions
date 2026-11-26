# \#97877 [SC-Medium] PletherOracle.updateBatchOrderExecutionPrice Checks _requireStoredFeedCoverage Against cache.publishTime Instead of cache.oldestPublishTime, Reverting executeOrderBatch With PletherOracle__StoredFeedBehind on Valid Multi-Feed Skew

**Submitted on Nov 3rd 2026 at 17:55:19 UTC by @alpha064 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97877
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

# `PletherOracle.updateBatchOrderExecutionPrice` Checks `_requireStoredFeedCoverage` Against `cache.publishTime` Instead of `cache.oldestPublishTime`, Reverting `executeOrderBatch` With `PletherOracle__StoredFeedBehind` on Valid Multi-Feed Skew

## Summary & Post-Audit Regression Provenance

The final pre-competition security audit (`audits/plether-ai-audit-report-2026-10-26.pdf`) evaluated commit `6d6622f2871dbdfc4904c20da2e1f7662f5763b6` (`fix(perps): use net risk depth and fingerprint calendar changes (#118)`, October 15, 2026). Six days later, post-audit commit `819192182833597641504b38cce7751bbcacecb8` (`fix(perps): synchronize historical Pyth and enforce signed live basket (#120)`, sub-commit `fix(perps): require post-commit feeds for batch reuse`, October 21, 2026 — direct ancestor of the in-scope competition commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`) modified `packages/perps/src/PletherOracle.sol` to track both `basket.publishTime` (`maxPublishTime`, the newest component feed timestamp across the 6 FX feeds) and `basket.oldestPublishTime` (`minPublishTime`, the oldest component feed timestamp across the 6 FX feeds) in `BatchOrderPriceCache`.

Per `packages/perps/ORACLE_SYNCHRONIZATION.md` (`lines 35, 48-50, 59-63, 109`), historical order execution allows the 6 component FX feed timestamps $t_0, \dots, t_5$ to diverge by up to `maxComponentPublishTimeDivergence` (`5` seconds by default), and batch cache reuse is explicitly designed to reuse a resolved historical basket for subsequent orders in `OrderRouter.executeOrderBatch` whenever every component feed follows the subsequent order's `commitTime` (`request.commitTime < cache.oldestPublishTime`).

When Order 1 in `OrderRouter.executeOrderBatch` resolves a historical basket where component timestamps are not all identical (for example, 1 feed is at $t_{\min}$ and the other 5 feeds are at $t_{\max} = t_{\min} + 1$, a valid 1-second divergence $\le 5\text{s}$):
1. `PletherOracle._resolveHistoricalOrderBasket()` (`lines 618-626`) computes `basket.publishTime = t_max` and `basket.oldestPublishTime = t_min`, calls `pyth.updatePriceFeeds(pythUpdateData)`, and verifies per-component storage coverage against each feed's actual publish time:
   ```solidity
   pyth.updatePriceFeeds{value: pythFee}(pythUpdateData);
   for (uint256 i; i < parsedFeeds.length; ++i) {
       _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);
   }
   ```
   After this step, the lagging feed's stored timestamp in Pyth is $t_{\min}$, while the other 5 feeds are at $t_{\max}$.
2. `PletherOracle.updateBatchOrderExecutionPrice()` (`lines 281-290`) caches `nextCache` with `publishTime = t_max` and `oldestPublishTime = t_min`.
3. When Order 2 in the same `executeOrderBatch` call has `request.commitTime < cache.oldestPublishTime`, both `OrderRouterExecutionSidecar._canReuseHistoricalBatchBasket()` (`lines 1074-1093`) and `PletherOracle._canReuseHistoricalBatchBasket()` (`lines 546-563`) return `true`.
4. However, inside the cache-reuse branch of `PletherOracle.updateBatchOrderExecutionPrice()` (`lines 262-266`), the loop checks `_requireStoredFeedCoverage` against `cache.publishTime` ($t_{\max}$) instead of `cache.oldestPublishTime` ($t_{\min}$):
   ```solidity
   if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
       // Cache contents are caller-supplied; independently prove storage coverage, not price authenticity.
       for (uint256 i; i < pythFeedIds.length; ++i) {
           _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
       }
   ```
5. Because the lagging feed in Pyth storage is at $t_{\min} < t_{\max}$, `_requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime)` reverts with `PletherOracle__StoredFeedBehind(feedId, t_min, t_max)`.
6. In `OrderRouterExecutionSidecar._batchOracle()` (`lines 1009-1029`), the `try oracle.updateBatchOrderExecutionPrice` `catch` block only catches `PletherOracle__HistoricalParserFailure.selector` (`reason.length == 68`) and re-throws all other errors via `revert(add(reason, 32), mload(reason))`, **reverting the entire `executeOrderBatch` transaction and rolling back Order 1 as well**.

### Why the Existing Unit Test Missed This Bug

In `packages/perps/test/perps/spec/oracle/OracleSynchronization.t.sol` (`lines 434-453`), `test_BatchReusesBasketWhenEveryComponentFollowsBothCommits` was specifically written to test batch cache reuse with a skewed multi-feed basket (`updates[0].price.publishTime = firstCommit + 3`, `updates[1..5].price.publishTime = firstCommit + 4`). However, lines 445-446 of that test accidentally pre-populated `baseMockPyth` storage with a future update (`firstCommit + 6`) before calling `executeOrderBatch`:

```solidity
bytes[] memory newer = _pricedPayload(firstCommit + 6, firstCommit + 5, 102_000_000);
baseMockPyth.updatePriceFeeds{value: FEE}(newer);
```

Because `baseMockPyth` already held `firstCommit + 6 > firstCommit + 4` in storage before `executeOrderBatch` ran, `storedPublishTime` was artificially `firstCommit + 6`, masking the `storedPublishTime (firstCommit + 3) < cache.publishTime (firstCommit + 4)` revert at `PletherOracle.sol:265`.

---

## Vulnerable Code

### 1. `packages/perps/src/PletherOracle.sol` (`lines 245-296`, `613-629`, and `645-653`)

Link: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L245-L296

```solidity
    function updateBatchOrderExecutionPrice(
        address refundRecipient,
        bytes[] calldata pythUpdateData,
        OrderExecutionRequest calldata request,
        BatchOrderPriceCache calldata cache
    )
        external
        payable
        override
        nonReentrant
        returns (bool ok, PriceSnapshot memory snapshot, BatchOrderPriceCache memory nextCache)
    {
        nextCache = cache;
        PolicySnapshot memory policy = _policyForMode(PriceMode.OrderExecution);

        BasketPrice memory basket;
        bool reusedBasket;
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
        } else {
            (basket, ok) = _resolveOrderExecutionBasket(refundRecipient, pythUpdateData, request, policy);
            if (!ok) {
                snapshot.updateFee = basket.pythFee;
                return (false, snapshot, nextCache);
            }
            if (!policy.oracleFrozen) {
                nextCache = BatchOrderPriceCache({
                    hasHistoricalBasket: true,
                    minReusableCommitTime: request.commitTime,
                    oldestPublishTime: basket.oldestPublishTime,
                    publishTime: basket.publishTime,
                    price: basket.price,
                    confidence: basket.confidence
                });
            }
        }
```

Link: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L613-L653

```solidity
        try pyth.parsePriceFeedUpdatesUnique{value: pythFee}(
            pythUpdateData, pythFeedIds, minPublishTime, maxPublishTime
        ) returns (
            PythStructs.PriceFeed[] memory parsedFeeds
        ) {
            basket = _computeBasketPriceFromFeeds(
                PriceMode.OrderExecution, parsedFeeds, maxComponentPublishTimeDivergence
            );
            // Reverts in the success body are NOT caught by the historical-parse catch below.
            // Keep the parsed basket as the pricing source even if storage already contains newer prices.
            pyth.updatePriceFeeds{value: pythFee}(pythUpdateData);
            for (uint256 i; i < parsedFeeds.length; ++i) {
                _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);
            }
            basket.pythFee = executionFunding;
            ok = true;
        } catch (bytes memory reason) {
...
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

### 2. `packages/perps/src/OrderRouterExecutionSidecar.sol` (`lines 990-1030`)

Link: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L990-L1030

```solidity
    function _batchOracle(
        IOrderRouterExecutionHost host,
        CfdTypes.Order memory order,
        bytes[] calldata pythUpdateData,
        BatchExecutionState memory state
    )
        private
        returns (
            OrderV3Types.PendingReason stopReason,
            OracleResult memory result,
            IPletherOracle.BatchOrderPriceCache memory nextCache
        )
    {
        IPletherOracle oracle = IPletherOracle(host.pletherOracle());
        bool reuse = _canReuseHistoricalBatchBasket(oracle, order, state.oracleCache);
        uint256 fee = reuse ? 0 : oracle.getOrderExecutionFee(pythUpdateData);
        if (fee != 0) {
            // Each non-reused historical parse plus stored-feed update is funded once from the batch msg.value budget.
            // slither-disable-next-line msg-value-loop
            uint256 suppliedValue = msg.value;
            if (suppliedValue < state.oracleFundingTotal + fee) {
                revert IPletherOracle.PletherOracle__InsufficientFee(suppliedValue, state.oracleFundingTotal + fee);
            }
        }
        IPletherOracle.PriceSnapshot memory snapshot;
        IPletherOracle.OrderExecutionRequest memory oracleRequest = _oracleRequest(order, false);
        bool ok;
        try oracle.updateBatchOrderExecutionPrice{value: fee}(
            state.executor, pythUpdateData, oracleRequest, state.oracleCache
        ) returns (
            bool resolved,
            IPletherOracle.PriceSnapshot memory resolvedSnapshot,
            IPletherOracle.BatchOrderPriceCache memory resolvedCache
        ) {
            ok = resolved;
            snapshot = resolvedSnapshot;
            nextCache = resolvedCache;
        } catch (bytes memory reason) {
            if (
                reason.length != 68
                    || _selector(reason) != IPletherOracle.PletherOracle__HistoricalParserFailure.selector
            ) {
                assembly ("memory-safe") {
                    revert(add(reason, 32), mload(reason))
                }
            }
            // The reverted Oracle call returned its funding to the Router; the outer refund handles it once.
            return (OrderV3Types.PendingReason.OracleFailure, result, state.oracleCache);
        }
```

---

## Impact

- **`OrderRouter.executeOrderBatch` Reverts and Rolls Back Prior Executed Orders on Any Skewed Multi-Feed Basket**:
  In a 6-feed FX basket (`EUR`, `JPY`, `GBP`, `CAD`, `SEK`, `CHF`), component Pyth feeds routinely publish 1–5 seconds apart (`0 < maxPublishTime - minPublishTime <= maxComponentPublishTimeDivergence`). Whenever Order 1 in a batch resolves such a basket (updating Pyth storage to `[t_0, ..., t_5]` where $\min_i t_i < \max_i t_i$) and a subsequent order in the batch satisfies `commitTime < cache.oldestPublishTime`, `OrderRouterExecutionSidecar._batchOracle` takes the `reuse = true` path (`fee = 0`) and `PletherOracle.updateBatchOrderExecutionPrice` reverts with `PletherOracle__StoredFeedBehind`, aborting the entire `executeOrderBatch` transaction and rolling back all preceding orders in the batch.

---

## Recommended Fix

In `PletherOracle.updateBatchOrderExecutionPrice()` (`line 265`), validate stored feed coverage against `cache.oldestPublishTime` (the minimum component timestamp guaranteed to be covered by every feed in the cached basket):

```diff
         if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
             // Cache contents are caller-supplied; independently prove storage coverage, not price authenticity.
             for (uint256 i; i < pythFeedIds.length; ++i) {
-                _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
+                _requireStoredFeedCoverage(pythFeedIds[i], cache.oldestPublishTime);
             }
```


## Proof of Concept

# Proof of Concept — `PletherOracle.updateBatchOrderExecutionPrice` Reverts With `PletherOracle__StoredFeedBehind` on Valid Multi-Feed Skew

## Test File

Save the following test file as `packages/perps/test/perps/spec/oracle/PletherOracleBatchCacheSkewDoS.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {OracleSynchronizationTestBase} from "../../shared/OracleSynchronizationTestBase.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

contract PletherOracleBatchCacheSkewDoSTest is OracleSynchronizationTestBase {

    function test_BatchCacheReuseRevertsOnValidMultiFeedSkewWhenPythStorageIsNotPreSeededInFuture() public {
        uint64 firstCommit = uint64(block.timestamp);
        _commit(ALICE, 0);
        _advance(firstCommit + 1);
        _commit(BOB, 0);
        _advance(firstCommit + 5);

        // Construct a valid 6-feed historical basket where feed 0 is at firstCommit + 3 and feeds 1..5 are at
        // firstCommit + 4 (1s skew <= maxComponentPublishTimeDivergence = 5s; all feeds > both commits).
        bytes[] memory historical = _payload(firstCommit + 4, firstCommit);
        MockPyth.FeedUpdate[] memory updates = abi.decode(historical[0], (MockPyth.FeedUpdate[]));
        updates[0].price.publishTime = firstCommit + 3;
        for (uint256 i; i < updates.length; ++i) {
            updates[i].price.price = 101_000_000;
        }
        historical[0] = abi.encode(updates);

        // Unlike test_BatchReusesBasketWhenEveryComponentFollowsBothCommits (which artificially pre-populated
        // baseMockPyth at firstCommit + 6), stored Pyth feeds are at firstCommit before the batch executes.
        // Order 1 succeeds and updates stored feed 0 to firstCommit + 3 and feeds 1..5 to firstCommit + 4,
        // caching cache.publishTime = firstCommit + 4 and cache.oldestPublishTime = firstCommit + 3.
        // Order 2 reuses the cache and checks _requireStoredFeedCoverage(feed[0], cache.publishTime = firstCommit + 4),
        // reverting the entire batch with PletherOracle__StoredFeedBehind!
        bytes32 laggingFeedId = _synchronizationFeedIds()[0];
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector,
                laggingFeedId,
                firstCommit + 3,
                firstCommit + 4
            )
        );
        router.executeOrderBatch{value: 2 * FEE}(2, historical);

        // Both ALICE's and BOB's orders remain unexecuted in the queue because the revert rolled back Order 1.
        assertEq(router.nextExecuteId(), 1);
        (uint256 aliceSize,,,,,,) = engine.positions(ALICE);
        (uint256 bobSize,,,,,,) = engine.positions(BOB);
        assertEq(aliceSize, 0);
        assertEq(bobSize, 0);
    }

}
```

## Command to Run

```bash
forge test --root packages/perps --match-path test/perps/spec/oracle/PletherOracleBatchCacheSkewDoS.t.sol -vv
```

## Verbatim Output

```text
Ran 1 test for test/perps/spec/oracle/PletherOracleBatchCacheSkewDoS.t.sol:PletherOracleBatchCacheSkewDoSTest
[PASS] test_BatchCacheReuseRevertsOnValidMultiFeedSkewWhenPythStorageIsNotPreSeededInFuture() (gas: 5048240)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 17.73ms (6.21ms CPU time)
```

