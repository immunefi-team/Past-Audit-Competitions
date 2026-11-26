# \#98827 [SC-Medium] Keeper can lose an entire order batch to a revert on a valid mixed-time historical basket due to cache reuse checking stored feeds against the newest component time

**Submitted on Nov 10th 2026 at 12:55:50 UTC by @midkernel for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98827
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Proof of Concept

1. Trader A commits an open order through `OrderRouter` at time `C`.
2. Trader B commits an open order through `OrderRouter` in the same second `C`. Both orders are pending and adjacent in the global FIFO.
3. Pyth publishes the first post-commit ticks of the basket components at different times: five components at `C + 1` and one component at `C + 2`. The gap is 1 second, inside the Oracle's 5 second component divergence limit.
4. Keeper calls `OrderRouter.executeOrderBatch` for both orders with the signed payload that contains those unique ticks and sufficient ETH for the Pyth fees.
5. `PletherOracle` parses the basket for Trader A's order, updates Pyth storage, verifies each stored feed against its own tick time, and returns a cache whose `publishTime` is `C + 2` and whose `oldestPublishTime` is `C + 1`. Trader A's order fills inside the batch.
6. `PletherOracle` selects cache reuse for Trader B's order and requires every stored feed to be at `C + 2` or later. The five feeds stored at `C + 1` fail the check and the Oracle reverts `PletherOracle__StoredFeedBehind(feed, C + 1, C + 2)`.
7. `OrderRouter` does not catch that error. The whole batch transaction reverts, Trader A's fill is rolled back, and the FIFO head still points at Trader A's order.
8. Keeper repeats the batch call later with the same payload and ten times the ETH. The call reverts with the identical error.
9. Keeper calls `OrderRouter.executeOrder` for Trader A's order and then for Trader B's order with the same payload. Both orders fill, which shows the payload is a valid historical basket for each of them.

The complete proof source is one Foundry test file.

`packages/perps/test/poc-batch-cache/BatchCacheMixedTime.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {OracleSynchronizationTestBase} from "../perps/shared/OracleSynchronizationTestBase.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice Batch-cache reuse checks every stored Pyth feed against the NEWEST component time of the cached basket,
///         while the fresh historical parse only synchronizes each feed to ITS OWN parsed time. A valid mixed-time
///         unique basket therefore prices the first order and then reverts `StoredFeedBehind` on reuse for the next
///         FIFO order, which rolls back the whole `executeOrderBatch` call.
contract BatchCacheMixedTimeTest is OracleSynchronizationTestBase {

    /// @dev One signed bundle. Feeds 0..4 first publish after `commit` at `commit + 1`; feed 5 first publishes after
    ///      `commit` at `commit + 1 + lag`. Every component is the unique first post-commit update (prev == commit).
    function _mixedTimeUpdates(
        uint64 commit,
        uint64 lag
    ) internal pure returns (MockPyth.FeedUpdate[] memory updates) {
        bytes[] memory data = _payload(commit + 1, commit);
        updates = abi.decode(data[0], (MockPyth.FeedUpdate[]));
        updates[5].price.publishTime = commit + 1 + lag;
    }

    function _encode(
        MockPyth.FeedUpdate[] memory updates
    ) internal pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    function _status(
        uint64 id
    ) internal view returns (OrderV3Types.LifecycleStatus) {
        return router.lifecycleBook().lifecycleStatus(id);
    }

    /// Positive control: equal-time basket. Both same-second orders execute in one batch with ONE historical parse.
    function test_control_EqualTimeBasketBatchReusesCache() public {
        uint64 commit = uint64(block.timestamp);
        uint64 a = _commit(ALICE, 0);
        uint64 b = _commit(BOB, 0);
        _advance(commit + 3);

        uint256 parsesBefore = baseMockPyth.parseUniqueCallCount();
        OrderV3Types.BatchResult memory result =
            router.executeOrderBatch{value: 4 * FEE}(b, _encode(_mixedTimeUpdates(commit, 0)));
        assertEq(result.terminalCount, 2, "both orders terminal");
        assertEq(uint256(_status(a)), uint256(OrderV3Types.LifecycleStatus.Executed));
        assertEq(uint256(_status(b)), uint256(OrderV3Types.LifecycleStatus.Executed));
        assertEq(baseMockPyth.parseUniqueCallCount() - parsesBefore, 1, "second order reused the batch cache");
    }

    /// Finding: the same two orders with a valid mixed-time unique basket (components 1 second apart, limit 5).
    function test_MixedTimeUniqueBasketRevertsWholeBatchOnCacheReuse() public {
        uint64 commit = uint64(block.timestamp);
        uint64 a = _commit(ALICE, 0);
        uint64 b = _commit(BOB, 0);
        _advance(commit + 3);
        bytes[] memory data = _encode(_mixedTimeUpdates(commit, 1));
        bytes32[] memory ids = _synchronizationFeedIds();
        assertEq(pletherOracle.maxComponentPublishTimeDivergence(), 5, "1s component gap is inside policy");

        // 1) The batch covering both orders reverts in full: order A's valid fill is rolled back too.
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector, ids[0], uint256(commit + 1), uint256(commit + 2)
            )
        );
        router.executeOrderBatch{value: 4 * FEE}(b, data);
        assertEq(router.nextExecuteId(), a, "no FIFO progress");
        assertEq(uint256(_status(a)), uint256(OrderV3Types.LifecycleStatus.Pending));
        assertEq(uint256(_status(b)), uint256(OrderV3Types.LifecycleStatus.Pending));

        // 2) Retrying later with more funding gives the identical revert for the same payload.
        _advance(commit + 10);
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector, ids[0], uint256(commit + 1), uint256(commit + 2)
            )
        );
        router.executeOrderBatch{value: 40 * FEE}(b, data);
        assertEq(router.nextExecuteId(), a, "still no FIFO progress");

        // 3) The same payload is a valid historical basket: each order fills when executed alone.
        OrderV3Types.ExecutionResult memory first = router.executeOrder{value: 2 * FEE}(a, data);
        assertEq(uint256(first.status), uint256(OrderV3Types.LifecycleStatus.Executed), "A fills alone");
        assertEq(engine.lastMarkTime(), commit + 2, "mark is the newest component time");
        OrderV3Types.ExecutionResult memory second = router.executeOrder{value: 2 * FEE}(b, data);
        assertEq(uint256(second.status), uint256(OrderV3Types.LifecycleStatus.Executed), "B fills alone");
    }

    /// Oracle-level view of the same defect: the cache the Oracle just produced is rejected by the Oracle.
    function test_OracleRejectsItsOwnFreshlyProvenMixedTimeCache() public {
        uint64 commit = uint64(block.timestamp);
        _advance(commit + 3);
        bytes[] memory data = _encode(_mixedTimeUpdates(commit, 1));
        bytes32[] memory ids = _synchronizationFeedIds();

        IPletherOracle.BatchOrderPriceCache memory cache;
        bool ok;
        (ok,, cache) = pletherOracle.updateBatchOrderExecutionPrice{value: 2 * FEE}(
            address(this), data, _request(commit, false), cache
        );
        assertTrue(ok, "mixed-time unique basket is accepted and synchronized");
        assertTrue(cache.hasHistoricalBasket);
        assertEq(cache.publishTime, commit + 2);
        assertEq(cache.oldestPublishTime, commit + 1);
        // Stored feeds cover every parsed component exactly.
        assertEq(baseMockPyth.getPriceUnsafe(ids[0]).publishTime, commit + 1);
        assertEq(baseMockPyth.getPriceUnsafe(ids[5]).publishTime, commit + 2);

        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector, ids[0], uint256(commit + 1), uint256(commit + 2)
            )
        );
        pletherOracle.updateBatchOrderExecutionPrice(address(this), new bytes[](0), _request(commit, false), cache);
    }

    /// Observed bound: when the payload also carries a newer update for the lagging stored feeds, storage reaches the
    /// newest component time and the same batch fills both orders with one parse.
    function test_bound_NewerUpdateForLaggingFeedsLetsBatchReuseCache() public {
        uint64 commit = uint64(block.timestamp);
        uint64 a = _commit(ALICE, 0);
        uint64 b = _commit(BOB, 0);
        _advance(commit + 3);

        MockPyth.FeedUpdate[] memory unique = _mixedTimeUpdates(commit, 1);
        MockPyth.FeedUpdate[] memory all = new MockPyth.FeedUpdate[](unique.length + 5);
        for (uint256 i; i < unique.length; ++i) {
            all[i] = unique[i];
        }
        // Feeds 0..4 tick again at commit + 2 (prev commit + 1): not unique for this commit, only newer storage.
        for (uint256 i; i < 5; ++i) {
            all[unique.length + i] =
                MockPyth.FeedUpdate(unique[i].id, MockPyth.MockPrice(100_000_000, 100_000, -8, commit + 2, commit + 1));
        }

        uint256 parsesBefore = baseMockPyth.parseUniqueCallCount();
        OrderV3Types.BatchResult memory result = router.executeOrderBatch{value: 4 * FEE}(b, _encode(all));
        assertEq(result.terminalCount, 2, "both orders terminal");
        assertEq(uint256(_status(a)), uint256(OrderV3Types.LifecycleStatus.Executed));
        assertEq(uint256(_status(b)), uint256(OrderV3Types.LifecycleStatus.Executed));
        assertEq(baseMockPyth.parseUniqueCallCount() - parsesBefore, 1, "second order reused the batch cache");
    }

}
```

**Environment and revisions.**

> Warning: run this proof only in a disposable local checkout of the pinned release. It deploys contracts on a local in-memory EVM. Do not point it at a live network and do not use real keys.

- Affected and tested revision: `immunefi-team/audit-comp-plether` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, with unmodified production sources under `packages/perps/src`.
- Toolchain: Foundry `forge 1.8.3` (`cae51ad458f6abb64852b7709eb784352429825d`), solc `0.8.35`, the pinned `packages/perps/foundry.toml` profile (`via_ir = true`, optimizer 200 runs). A cold via-IR build takes several minutes and needs roughly 15 GB of memory. The tests run in milliseconds.
- Topology: the test extends `OracleSynchronizationTestBase` and `BasePerpTest` from the same release. They deploy the unmodified production `CfdEngine`, `OrderRouter` (through the release's `LegacyOrderRouterHarness`, which inherits `OrderRouter` and only adds a test commit helper that builds a full V3 request), `OrderRouterExecutionSidecar`, `OrderLifecycleBook`, `MarginClearinghouse`, `HousePool`, both `TrancheVault` instances and `PletherOracle`.
- External stand-ins: the release's `MockPyth` (signed price source, explicit payload mode) and the mock settlement token. `MockPyth` applies the same rules the proof depends on: `parsePriceFeedUpdatesUnique` returns, per feed, the update whose `prevPublishTime` is before the minimum and whose `publishTime` is inside the window, and `updatePriceFeeds` stores a feed only when the update is newer than storage. Execution against the deployed Pyth contract on Arbitrum One is an unverified production assumption of this proof.
- Entry points called: `OrderRouter.commitOrder` (through the harness helper), `OrderRouter.executeOrderBatch`, `OrderRouter.executeOrder` and `PletherOracle.updateBatchOrderExecutionPrice`.

Fixture data, not part of the attack: the accounts `ALICE` and `BOB` are funded with `10_000e6` settlement units each (mock token, $0 actual value), every feed price is `1e8` with confidence `1e5`, the Pyth fee is `1 gwei`, and the basket weights are those of `OracleSynchronizationTestBase`.

**Setup.**

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
cd packages/perps
mkdir -p test/poc-batch-cache
```

Save the source above as `test/poc-batch-cache/BatchCacheMixedTime.t.sol` inside `packages/perps`. No production source is edited.

**Invocation.**

Run from `packages/perps`:

```bash
forge test --match-path 'test/poc-batch-cache/BatchCacheMixedTime.t.sol' -vv
```

**Tests, control and observed bound.**

- `test_control_EqualTimeBasketBatchReusesCache` (positive control): the same two orders with all components at `C + 1`. One batch call fills both orders with a single historical parse, so cache reuse works in this environment.
- `test_MixedTimeUniqueBasketRevertsWholeBatchOnCacheReuse` (finding): steps 1 to 9 above. It asserts the exact `PletherOracle__StoredFeedBehind(feed0, C + 1, C + 2)` revert twice, that `nextExecuteId` still equals Trader A's order and both orders are `Pending`, and that each order then reaches `Executed` through `executeOrder`.
- `test_OracleRejectsItsOwnFreshlyProvenMixedTimeCache` (finding at the Oracle boundary): `PletherOracle.updateBatchOrderExecutionPrice` returns `ok = true` and a cache with `publishTime = C + 2` and `oldestPublishTime = C + 1` for the mixed-time payload. Passing that returned cache back for the same commit reverts `StoredFeedBehind`.
- `test_bound_NewerUpdateForLaggingFeedsLetsBatchReuseCache` (observed bound): when the payload also contains a second update at `C + 2` for the five lagging feeds, Pyth storage reaches `C + 2` for every feed and the same batch fills both orders with one parse.

Evidence provenance: the logs below were recorded by running the setup and invocation above on a fresh clone of the pinned commit, with the proof file extracted from this document. The proof file has SHA-256 `028056c45b61189f1cbc7237a5c6910d1747f0447293bde2efca03198323a228`.

**Cleanup.**

```bash
cd ../../..
rm -rf audit-comp-plether
```

**Demonstrated scope.**

The proof establishes a full revert of `executeOrderBatch` and of the Oracle's own cache reuse for a mixed-time unique basket when Pyth storage holds the older components at their parsed times. It establishes recovery through single-order execution and through a payload that also advances the lagging feeds. No balance changes other than ordinary order settlement occur.

### Logs

```text
Ran 4 tests for test/poc-batch-cache/BatchCacheMixedTime.t.sol:BatchCacheMixedTimeTest
[PASS] test_MixedTimeUniqueBasketRevertsWholeBatchOnCacheReuse() (gas: 10739307)
[PASS] test_OracleRejectsItsOwnFreshlyProvenMixedTimeCache() (gas: 537267)
[PASS] test_bound_NewerUpdateForLaggingFeedsLetsBatchReuseCache() (gas: 5820268)
[PASS] test_control_EqualTimeBasketBatchReusesCache() (gas: 5787208)
Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 48.73ms (64.83ms CPU time)
Ran 1 test suite in 49.91ms (48.73ms CPU time): 4 tests passed, 0 failed, 0 skipped (4 total tests)
```


## Proof of Concept

## Proof of Concept

1. Trader A commits an open order through `OrderRouter` at time `C`.
2. Trader B commits an open order through `OrderRouter` in the same second `C`. Both orders are pending and adjacent in the global FIFO.
3. Pyth publishes the first post-commit ticks of the basket components at different times: five components at `C + 1` and one component at `C + 2`. The gap is 1 second, inside the Oracle's 5 second component divergence limit.
4. Keeper calls `OrderRouter.executeOrderBatch` for both orders with the signed payload that contains those unique ticks and sufficient ETH for the Pyth fees.
5. `PletherOracle` parses the basket for Trader A's order, updates Pyth storage, verifies each stored feed against its own tick time, and returns a cache whose `publishTime` is `C + 2` and whose `oldestPublishTime` is `C + 1`. Trader A's order fills inside the batch.
6. `PletherOracle` selects cache reuse for Trader B's order and requires every stored feed to be at `C + 2` or later. The five feeds stored at `C + 1` fail the check and the Oracle reverts `PletherOracle__StoredFeedBehind(feed, C + 1, C + 2)`.
7. `OrderRouter` does not catch that error. The whole batch transaction reverts, Trader A's fill is rolled back, and the FIFO head still points at Trader A's order.
8. Keeper repeats the batch call later with the same payload and ten times the ETH. The call reverts with the identical error.
9. Keeper calls `OrderRouter.executeOrder` for Trader A's order and then for Trader B's order with the same payload. Both orders fill, which shows the payload is a valid historical basket for each of them.

The complete proof source is one Foundry test file.

`packages/perps/test/poc-batch-cache/BatchCacheMixedTime.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {OracleSynchronizationTestBase} from "../perps/shared/OracleSynchronizationTestBase.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice Batch-cache reuse checks every stored Pyth feed against the NEWEST component time of the cached basket,
///         while the fresh historical parse only synchronizes each feed to ITS OWN parsed time. A valid mixed-time
///         unique basket therefore prices the first order and then reverts `StoredFeedBehind` on reuse for the next
///         FIFO order, which rolls back the whole `executeOrderBatch` call.
contract BatchCacheMixedTimeTest is OracleSynchronizationTestBase {

    /// @dev One signed bundle. Feeds 0..4 first publish after `commit` at `commit + 1`; feed 5 first publishes after
    ///      `commit` at `commit + 1 + lag`. Every component is the unique first post-commit update (prev == commit).
    function _mixedTimeUpdates(
        uint64 commit,
        uint64 lag
    ) internal pure returns (MockPyth.FeedUpdate[] memory updates) {
        bytes[] memory data = _payload(commit + 1, commit);
        updates = abi.decode(data[0], (MockPyth.FeedUpdate[]));
        updates[5].price.publishTime = commit + 1 + lag;
    }

    function _encode(
        MockPyth.FeedUpdate[] memory updates
    ) internal pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    function _status(
        uint64 id
    ) internal view returns (OrderV3Types.LifecycleStatus) {
        return router.lifecycleBook().lifecycleStatus(id);
    }

    /// Positive control: equal-time basket. Both same-second orders execute in one batch with ONE historical parse.
    function test_control_EqualTimeBasketBatchReusesCache() public {
        uint64 commit = uint64(block.timestamp);
        uint64 a = _commit(ALICE, 0);
        uint64 b = _commit(BOB, 0);
        _advance(commit + 3);

        uint256 parsesBefore = baseMockPyth.parseUniqueCallCount();
        OrderV3Types.BatchResult memory result =
            router.executeOrderBatch{value: 4 * FEE}(b, _encode(_mixedTimeUpdates(commit, 0)));
        assertEq(result.terminalCount, 2, "both orders terminal");
        assertEq(uint256(_status(a)), uint256(OrderV3Types.LifecycleStatus.Executed));
        assertEq(uint256(_status(b)), uint256(OrderV3Types.LifecycleStatus.Executed));
        assertEq(baseMockPyth.parseUniqueCallCount() - parsesBefore, 1, "second order reused the batch cache");
    }

    /// Finding: the same two orders with a valid mixed-time unique basket (components 1 second apart, limit 5).
    function test_MixedTimeUniqueBasketRevertsWholeBatchOnCacheReuse() public {
        uint64 commit = uint64(block.timestamp);
        uint64 a = _commit(ALICE, 0);
        uint64 b = _commit(BOB, 0);
        _advance(commit + 3);
        bytes[] memory data = _encode(_mixedTimeUpdates(commit, 1));
        bytes32[] memory ids = _synchronizationFeedIds();
        assertEq(pletherOracle.maxComponentPublishTimeDivergence(), 5, "1s component gap is inside policy");

        // 1) The batch covering both orders reverts in full: order A's valid fill is rolled back too.
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector, ids[0], uint256(commit + 1), uint256(commit + 2)
            )
        );
        router.executeOrderBatch{value: 4 * FEE}(b, data);
        assertEq(router.nextExecuteId(), a, "no FIFO progress");
        assertEq(uint256(_status(a)), uint256(OrderV3Types.LifecycleStatus.Pending));
        assertEq(uint256(_status(b)), uint256(OrderV3Types.LifecycleStatus.Pending));

        // 2) Retrying later with more funding gives the identical revert for the same payload.
        _advance(commit + 10);
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector, ids[0], uint256(commit + 1), uint256(commit + 2)
            )
        );
        router.executeOrderBatch{value: 40 * FEE}(b, data);
        assertEq(router.nextExecuteId(), a, "still no FIFO progress");

        // 3) The same payload is a valid historical basket: each order fills when executed alone.
        OrderV3Types.ExecutionResult memory first = router.executeOrder{value: 2 * FEE}(a, data);
        assertEq(uint256(first.status), uint256(OrderV3Types.LifecycleStatus.Executed), "A fills alone");
        assertEq(engine.lastMarkTime(), commit + 2, "mark is the newest component time");
        OrderV3Types.ExecutionResult memory second = router.executeOrder{value: 2 * FEE}(b, data);
        assertEq(uint256(second.status), uint256(OrderV3Types.LifecycleStatus.Executed), "B fills alone");
    }

    /// Oracle-level view of the same defect: the cache the Oracle just produced is rejected by the Oracle.
    function test_OracleRejectsItsOwnFreshlyProvenMixedTimeCache() public {
        uint64 commit = uint64(block.timestamp);
        _advance(commit + 3);
        bytes[] memory data = _encode(_mixedTimeUpdates(commit, 1));
        bytes32[] memory ids = _synchronizationFeedIds();

        IPletherOracle.BatchOrderPriceCache memory cache;
        bool ok;
        (ok,, cache) = pletherOracle.updateBatchOrderExecutionPrice{value: 2 * FEE}(
            address(this), data, _request(commit, false), cache
        );
        assertTrue(ok, "mixed-time unique basket is accepted and synchronized");
        assertTrue(cache.hasHistoricalBasket);
        assertEq(cache.publishTime, commit + 2);
        assertEq(cache.oldestPublishTime, commit + 1);
        // Stored feeds cover every parsed component exactly.
        assertEq(baseMockPyth.getPriceUnsafe(ids[0]).publishTime, commit + 1);
        assertEq(baseMockPyth.getPriceUnsafe(ids[5]).publishTime, commit + 2);

        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector, ids[0], uint256(commit + 1), uint256(commit + 2)
            )
        );
        pletherOracle.updateBatchOrderExecutionPrice(address(this), new bytes[](0), _request(commit, false), cache);
    }

    /// Observed bound: when the payload also carries a newer update for the lagging stored feeds, storage reaches the
    /// newest component time and the same batch fills both orders with one parse.
    function test_bound_NewerUpdateForLaggingFeedsLetsBatchReuseCache() public {
        uint64 commit = uint64(block.timestamp);
        uint64 a = _commit(ALICE, 0);
        uint64 b = _commit(BOB, 0);
        _advance(commit + 3);

        MockPyth.FeedUpdate[] memory unique = _mixedTimeUpdates(commit, 1);
        MockPyth.FeedUpdate[] memory all = new MockPyth.FeedUpdate[](unique.length + 5);
        for (uint256 i; i < unique.length; ++i) {
            all[i] = unique[i];
        }
        // Feeds 0..4 tick again at commit + 2 (prev commit + 1): not unique for this commit, only newer storage.
        for (uint256 i; i < 5; ++i) {
            all[unique.length + i] =
                MockPyth.FeedUpdate(unique[i].id, MockPyth.MockPrice(100_000_000, 100_000, -8, commit + 2, commit + 1));
        }

        uint256 parsesBefore = baseMockPyth.parseUniqueCallCount();
        OrderV3Types.BatchResult memory result = router.executeOrderBatch{value: 4 * FEE}(b, _encode(all));
        assertEq(result.terminalCount, 2, "both orders terminal");
        assertEq(uint256(_status(a)), uint256(OrderV3Types.LifecycleStatus.Executed));
        assertEq(uint256(_status(b)), uint256(OrderV3Types.LifecycleStatus.Executed));
        assertEq(baseMockPyth.parseUniqueCallCount() - parsesBefore, 1, "second order reused the batch cache");
    }

}
```

**Environment and revisions.**

> Warning: run this proof only in a disposable local checkout of the pinned release. It deploys contracts on a local in-memory EVM. Do not point it at a live network and do not use real keys.

- Affected and tested revision: `immunefi-team/audit-comp-plether` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, with unmodified production sources under `packages/perps/src`.
- Toolchain: Foundry `forge 1.8.3` (`cae51ad458f6abb64852b7709eb784352429825d`), solc `0.8.35`, the pinned `packages/perps/foundry.toml` profile (`via_ir = true`, optimizer 200 runs). A cold via-IR build takes several minutes and needs roughly 15 GB of memory. The tests run in milliseconds.
- Topology: the test extends `OracleSynchronizationTestBase` and `BasePerpTest` from the same release. They deploy the unmodified production `CfdEngine`, `OrderRouter` (through the release's `LegacyOrderRouterHarness`, which inherits `OrderRouter` and only adds a test commit helper that builds a full V3 request), `OrderRouterExecutionSidecar`, `OrderLifecycleBook`, `MarginClearinghouse`, `HousePool`, both `TrancheVault` instances and `PletherOracle`.
- External stand-ins: the release's `MockPyth` (signed price source, explicit payload mode) and the mock settlement token. `MockPyth` applies the same rules the proof depends on: `parsePriceFeedUpdatesUnique` returns, per feed, the update whose `prevPublishTime` is before the minimum and whose `publishTime` is inside the window, and `updatePriceFeeds` stores a feed only when the update is newer than storage. Execution against the deployed Pyth contract on Arbitrum One is an unverified production assumption of this proof.
- Entry points called: `OrderRouter.commitOrder` (through the harness helper), `OrderRouter.executeOrderBatch`, `OrderRouter.executeOrder` and `PletherOracle.updateBatchOrderExecutionPrice`.

Fixture data, not part of the attack: the accounts `ALICE` and `BOB` are funded with `10_000e6` settlement units each (mock token, $0 actual value), every feed price is `1e8` with confidence `1e5`, the Pyth fee is `1 gwei`, and the basket weights are those of `OracleSynchronizationTestBase`.

**Setup.**

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
cd packages/perps
mkdir -p test/poc-batch-cache
```

Save the source above as `test/poc-batch-cache/BatchCacheMixedTime.t.sol` inside `packages/perps`. No production source is edited.

**Invocation.**

Run from `packages/perps`:

```bash
forge test --match-path 'test/poc-batch-cache/BatchCacheMixedTime.t.sol' -vv
```

**Tests, control and observed bound.**

- `test_control_EqualTimeBasketBatchReusesCache` (positive control): the same two orders with all components at `C + 1`. One batch call fills both orders with a single historical parse, so cache reuse works in this environment.
- `test_MixedTimeUniqueBasketRevertsWholeBatchOnCacheReuse` (finding): steps 1 to 9 above. It asserts the exact `PletherOracle__StoredFeedBehind(feed0, C + 1, C + 2)` revert twice, that `nextExecuteId` still equals Trader A's order and both orders are `Pending`, and that each order then reaches `Executed` through `executeOrder`.
- `test_OracleRejectsItsOwnFreshlyProvenMixedTimeCache` (finding at the Oracle boundary): `PletherOracle.updateBatchOrderExecutionPrice` returns `ok = true` and a cache with `publishTime = C + 2` and `oldestPublishTime = C + 1` for the mixed-time payload. Passing that returned cache back for the same commit reverts `StoredFeedBehind`.
- `test_bound_NewerUpdateForLaggingFeedsLetsBatchReuseCache` (observed bound): when the payload also contains a second update at `C + 2` for the five lagging feeds, Pyth storage reaches `C + 2` for every feed and the same batch fills both orders with one parse.

Evidence provenance: the logs below were recorded by running the setup and invocation above on a fresh clone of the pinned commit, with the proof file extracted from this document. The proof file has SHA-256 `028056c45b61189f1cbc7237a5c6910d1747f0447293bde2efca03198323a228`.

**Cleanup.**

```bash
cd ../../..
rm -rf audit-comp-plether
```

**Demonstrated scope.**

The proof establishes a full revert of `executeOrderBatch` and of the Oracle's own cache reuse for a mixed-time unique basket when Pyth storage holds the older components at their parsed times. It establishes recovery through single-order execution and through a payload that also advances the lagging feeds. No balance changes other than ordinary order settlement occur.

### Logs

```text
Ran 4 tests for test/poc-batch-cache/BatchCacheMixedTime.t.sol:BatchCacheMixedTimeTest
[PASS] test_MixedTimeUniqueBasketRevertsWholeBatchOnCacheReuse() (gas: 10739307)
[PASS] test_OracleRejectsItsOwnFreshlyProvenMixedTimeCache() (gas: 537267)
[PASS] test_bound_NewerUpdateForLaggingFeedsLetsBatchReuseCache() (gas: 5820268)
[PASS] test_control_EqualTimeBasketBatchReusesCache() (gas: 5787208)
Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 48.73ms (64.83ms CPU time)
Ran 1 test suite in 49.91ms (48.73ms CPU time): 4 tests passed, 0 failed, 0 skipped (4 total tests)
```

