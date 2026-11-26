# \#98508 [SC-Medium] executeOrderBatch reverts when a later order reuses a basket whose feeds have different publish times

**Submitted on Nov 8th 2026 at 09:51:32 UTC by @hungviet276 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98508
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

**Brief/Intro**
I found an issue with historical price reuse in OrderRouter.executeOrderBatch. When the first order in a batch executes, the router caches its historical basket so a later order can reuse it. The reuse path checks every stored Pyth feed against cache.publishTime, which is now the newest timestamp in the basket. However, the first order only makes sure each stored feed is at least as recent as its own parsed timestamp. If one feed is a second behind the others, the second order reverts with PletherOracle__StoredFeedBehind. The batch router doesn’t handle this error, so the entire transaction reverts and the first order is rolled back as well. I reproduced this with two valid orders: both execute individually through executeOrder, but executeOrderBatch fails when it processes them together. No funds are lost, but batch execution breaks for ordinary, valid orders.

**Vulnerability Details**
The issue is in PletherOracle.updateBatchOrderExecutionPrice, lines 262–266:

if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
    for (uint256 i; i < pythFeedIds.length; ++i) {
        _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
    }
    // ... then builds the basket from the cache

The problem is the timestamp passed to _requireStoredFeedCoverage.

When the first order resolves its historical price, _resolveHistoricalOrderBasket parses the feed updates, calls pyth.updatePriceFeeds, and checks storage coverage using each feed’s own timestamp:

_requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);

This doesn’t guarantee that all six stored feeds have reached the newest timestamp in the basket.

The basket calculation stores both timestamps:

publishTime: newest component timestamp
oldestPublishTime: oldest component timestamp
The protocol allows the component timestamps to differ by up to maxComponentPublishTimeDivergence (5 seconds by default).

For example, suppose the first feed has a timestamp of t+4, while the other five have t+5. The first order resolves successfully. After synchronization, Pyth storage still has the first feed at t+4, which is valid. The second order then reuses that basket, and the reuse path calls:

_requireStoredFeedCoverage(feed0, t + 5);

But feed0 is only at t+4, so the call reverts:

PletherOracle__StoredFeedBehind(feed0, t + 4, t + 5)

Why the whole batch reverts
I traced the error back through OrderRouterExecutionSidecar._prepareBatchOracle (lines 1019–1030). The catch block only handles PletherOracle__HistoricalParserFailure with the expected 68-byte revert data. Other errors are re-thrown:

if (
    reason.length != 68
        || _selector(reason) != IPletherOracle.PletherOracle__HistoricalParserFailure.selector
) {
    assembly ("memory-safe") {
        revert(add(reason, 32), mload(reason))
    }
}

StoredFeedBehind has three arguments, so its revert data is 100 bytes. It doesn’t match the handled error. The exception propagates out of executeOrderBatch and reverts the entire transaction, including the first order’s execution. Oracle preparation happens outside the per-order rollback frame, so the router’s normal item isolation doesn’t prevent this.

Sending additional ETH doesn’t fix it. Once the router selects the reuse path, it forwards zero Pyth fee for that order, and the same coverage check still fails.

Regression from PR #120
I checked the history of this code.

PR #103 (64c6ea9) introduced the storage coverage check using cache.publishTime. At that point, publishTime was the oldest component timestamp (basket.publishTime = uint64(minPublishTime)).

PR #120 (8191921), which addressed MED-23, changed publishTime to the newest component timestamp and added oldestPublishTime. The cache eligibility logic was updated to use oldestPublishTime, but the storage coverage check was left unchanged.

That makes the old check too strict for a basket with different component timestamps. The first order can produce a valid cached basket that the second order cannot reuse.

This also doesn’t match ORACLE_SYNCHRONIZATION.md. Its Batch reuse section says reuse must “satisfy cache coverage. Otherwise the Router requests a new historical resolution.” Here, reuse eligibility passes, but the storage coverage check fails afterward. The transaction reverts instead of falling back to a fresh resolution or returning a batch stop reason.

Existing test coverage
The existing test test_BatchReusesBasketWhenEveryComponentFollowsBothCommits uses a basket with mixed component timestamps. However, it first calls baseMockPyth.updatePriceFeeds with newer prices for all feeds, which makes the storage coverage check pass.

Without that extra update, the same test setup reproduces the issue. Historical execution itself doesn’t advance all stored feeds to the newest component timestamp. Live basket updates don’t provide that guarantee either, since they use parsePriceFeedUpdates, which doesn’t write prices into Pyth storage.

Conditions
I reproduced the issue when:

At least two valid orders are included in the same batch.
The later order is eligible to reuse the first order’s historical basket.
The basket contains different component timestamps.
At least one feed’s stored timestamp is still older than the newest component timestamp (nobody pushed a newer price for it in between).
No malicious action or unusual configuration is needed. Two orders committed in the same second can reach this state. Batches without cache reuse, or with identical timestamps across all feeds, are unaffected.

Suggested fix
The coverage check should use the oldest timestamp:

- _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
+ _requireStoredFeedCoverage(pythFeedIds[i], cache.oldestPublishTime);

This matches the minimum storage coverage guaranteed by the first order’s synchronization.

The existing _canReuseHistoricalBatchBasket checks should remain unchanged, especially:

commitTime < cache.oldestPublishTime

That check ensures all components are newer than the later order’s commit.

I tried the one-line change locally, and all 241 existing oracle spec tests under test/perps/spec/oracle/*.t.sol passed.

Another possible fix is to resolve a fresh historical basket when the stored feeds don’t cover the cached basket, instead of reverting the entire batch.

**Impact Details**
This affects the public OrderRouter.executeOrderBatch entrypoint.

A keeper can submit a batch of valid orders and lose the entire transaction because one stored Pyth feed is behind the newest feed timestamp. Even if the first order has already executed within the transaction, that execution is rolled back.

The orders remain Pending and can still be executed individually through executeOrder. I didn’t find a direct loss of user funds. The impact is that batch execution becomes unreliable for otherwise valid orders, and the keeper spends gas on reverted transactions.

As additional context, the reference keeper in plether-app (Keeper.hs, submitIntent) uses executeOrderBatch when two or more orders are ready. I didn’t find a fallback to individual execution in that path. The keeper itself is out of scope, and the PoC doesn’t depend on its implementation.

I consider this a Low-severity issue matching the in-scope impact “Contract fails to deliver promised returns, but doesn’t lose value”. It breaks the expected batch execution behavior without directly losing funds, and single-order execution remains available.

**References**
PletherOracle.sol: cache reuse
PletherOracle.sol: historical synchronization
PletherOracle.sol: basket timestamps
OrderRouterExecutionSidecar.sol: error handling
OrderRouterExecutionSidecar.sol: cache reuse eligibility
ORACLE_SYNCHRONIZATION.md: Batch reuse
CANONICAL_ENTRYPOINTS.md: keeper batch execution
PR #103 commit
PR #120 commit



## Proof of Concept

**Proof of Concept**
The PoC was run against commit a82d4fefb02768e5b3bddcc97dee371eee24fea8. No contract in scope needs to be modified.

The test extends the repository’s OracleSynchronizationTestBase. It uses the real Oracle, Router, Engine and pool contracts, with a mocked Pyth implementation. All order operations go through the public Router entrypoints (commitOrder, executeOrderBatch, executeOrder).

Steps
1. Check out the repository at commit a82d4fe.
2. Save the PoC source below as packages/perps/test/perps/spec/oracle/BatchReuseCoveragePoC.t.sol.
3. Run:
cd packages/perps
FOUNDRY_VIA_IR=false forge test --match-path test/perps/spec/oracle/BatchReuseCoveragePoC.t.sol -vv

FOUNDRY_VIA_IR=false only speeds up compilation.

Result from my local run:

[PASS] test_Control_BatchSucceedsWhenStorageAlreadyNewer()
[PASS] test_PoC_BatchEntrypointWorksWithoutReuse()
[PASS] test_PoC_CompatibleBatchRevertsOnSelfInflictedCoverageGap()
[PASS] test_PoC_SameOrdersExecuteIndividually()
[PASS] test_PoC_SameSecondCommitsBatchReverts()
Suite result: ok. 5 passed; 0 failed; 0 skipped

What the tests show
CompatibleBatchRevertsOnSelfInflictedCoverageGap: Alice commits at t, Bob at t+3. In the historical payload, feed0 is at t+4 and the other feeds are at t+5. The second order fails with StoredFeedBehind(feed0, t+4, t+5), and both orders are reverted and stay Pending. Sending 100 times the fee doesn’t change the result.
SameSecondCommitsBatchReverts: two orders committed in the same second trigger the same failure, with a one-second feed timestamp difference.
SameOrdersExecuteIndividually: both orders execute successfully with executeOrder and the same payload.
BatchEntrypointWorksWithoutReuse: processing one order per batch works with the same historical payload.
Control_BatchSucceedsWhenStorageAlreadyNewer: the batch succeeds when Pyth storage is updated in advance, as in the existing repository test setup.
The -vvvv trace shows the first order going through oracle synchronization and finalization before the second order fails:

MockPyth::parsePriceFeedUpdatesUnique{value: 1000000000}(...)
MockPyth::updatePriceFeeds{value: 1000000000}(...)
CfdEngine::updateMarkPrice(101000000, 1709794802)
emit OrderFinalized(orderId: 1, account: 0x...A11cE, ...)
← [Revert] PletherOracle__StoredFeedBehind(0x...04, 1709794801, 1709794802)

Because the outer transaction reverts, the first order’s finalization does not persist.

PoC source
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {OracleSynchronizationTestBase} from "../../shared/OracleSynchronizationTestBase.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

contract BatchReuseCoveragePoC is OracleSynchronizationTestBase {

    function _pricedPayload(
        uint64 tick,
        uint64 previous,
        int64 price
    ) private pure returns (bytes[] memory data) {
        data = _payload(tick, previous);
        MockPyth.FeedUpdate[] memory updates = abi.decode(data[0], (MockPyth.FeedUpdate[]));
        for (uint256 i; i < updates.length; ++i) {
            updates[i].price.price = price;
        }
        data[0] = abi.encode(updates);
    }

    function _fixture() private returns (uint64 firstCommit, bytes[] memory historical) {
        firstCommit = uint64(block.timestamp);
        _commit(ALICE, 0);
        _advance(firstCommit + 3);
        _commit(BOB, 0);
        _advance(firstCommit + 6);

        // Feed 0 ticks at +4, feeds 1..5 at +5.
        historical = _pricedPayload(firstCommit + 5, firstCommit, 101_000_000);
        MockPyth.FeedUpdate[] memory updates = abi.decode(historical[0], (MockPyth.FeedUpdate[]));
        updates[0].price.publishTime = firstCommit + 4;
        historical[0] = abi.encode(updates);
    }

    function test_Control_BatchSucceedsWhenStorageAlreadyNewer() public {
        (uint64 firstCommit, bytes[] memory historical) = _fixture();
        baseMockPyth.updatePriceFeeds{value: FEE}(_pricedPayload(firstCommit + 6, firstCommit + 5, 102_000_000));

        OrderV3Types.BatchResult memory result = router.executeOrderBatch{value: 2 * FEE}(2, historical);
        assertEq(result.terminalCount, 2);
        assertEq(result.nextOrderId, 0);
    }

    function test_PoC_CompatibleBatchRevertsOnSelfInflictedCoverageGap() public {
        (uint64 firstCommit, bytes[] memory historical) = _fixture();
        bytes32 feed0 = _synchronizationFeedIds()[0];

        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector, feed0, firstCommit + 4, firstCommit + 5
            )
        );
        router.executeOrderBatch{value: 4 * FEE}(2, historical);

        assertEq(router.nextExecuteId(), 1);
        assertEq(uint256(router.lifecycleBook().lifecycleStatus(1)), uint256(OrderV3Types.LifecycleStatus.Pending));
        assertEq(uint256(router.lifecycleBook().lifecycleStatus(2)), uint256(OrderV3Types.LifecycleStatus.Pending));

        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector, feed0, firstCommit + 4, firstCommit + 5
            )
        );
        router.executeOrderBatch{value: 100 * FEE}(2, historical);
    }

    function test_PoC_SameSecondCommitsBatchReverts() public {
        uint64 t = uint64(block.timestamp);
        _commit(ALICE, 0);
        _commit(BOB, 0);
        _advance(t + 3);

        bytes[] memory historical = _pricedPayload(t + 2, t, 101_000_000);
        MockPyth.FeedUpdate[] memory updates = abi.decode(historical[0], (MockPyth.FeedUpdate[]));
        updates[3].price.publishTime = t + 1;
        historical[0] = abi.encode(updates);

        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector, _synchronizationFeedIds()[3], t + 1, t + 2
            )
        );
        router.executeOrderBatch{value: 4 * FEE}(2, historical);
        assertEq(router.nextExecuteId(), 1, "no order executed");
    }

    function test_PoC_BatchEntrypointWorksWithoutReuse() public {
        (, bytes[] memory historical) = _fixture();
        OrderV3Types.BatchResult memory first = router.executeOrderBatch{value: 2 * FEE}(1, historical);
        assertEq(first.terminalCount, 1);
        OrderV3Types.BatchResult memory second = router.executeOrderBatch{value: 2 * FEE}(2, historical);
        assertEq(second.terminalCount, 1);
        assertEq(router.nextExecuteId(), 0);
    }

    function test_PoC_SameOrdersExecuteIndividually() public {
        (, bytes[] memory historical) = _fixture();
        router.executeOrder{value: 2 * FEE}(1, historical);
        router.executeOrder{value: 2 * FEE}(2, historical);
        assertEq(uint256(router.lifecycleBook().lifecycleStatus(1)), uint256(OrderV3Types.LifecycleStatus.Executed));
        assertEq(uint256(router.lifecycleBook().lifecycleStatus(2)), uint256(OrderV3Types.LifecycleStatus.Executed));
        assertEq(router.nextExecuteId(), 0);
    }
}



