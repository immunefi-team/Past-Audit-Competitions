# \#97795 [SC-Medium] Batch cache reuse checks every Pyth feed against the newest component time, so executeOrderBatch reverts for valid orders when basket components have different publish times

**Submitted on Nov 3rd 2026 at 08:43:36 UTC by @Nyko for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97795
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`OrderRouter.executeOrderBatch` reuses the first order's historical basket for later orders committed before that basket's oldest component. On the reuse path, `PletherOracle` checks that Pyth storage of **every** feed is at least the basket's **newest** component time. The first order only synchronized each feed up to its **own** tick. When the six components were not all published in the same second, the check fails with `PletherOracle__StoredFeedBehind`, the Router re-raises it, and the whole batch reverts, including orders already filled in the same call. The bug was introduced by the MED-7 fix (commit 8191921, PR #120), which changed `publishTime` from the oldest to the newest component time but did not update this check.

## Vulnerability Details

1. A historical basket records two times (`PletherOracle.sol:874-875`):

```solidity
basket.publishTime = uint64(maxPublishTime);
basket.oldestPublishTime = uint64(minPublishTime);
```

2. The first order of a batch parses the payload, writes it to Pyth storage, and checks each feed against its **own** component time (`PletherOracle.sol:623-626`):

```solidity
pyth.updatePriceFeeds{value: pythFee}(pythUpdateData);
for (uint256 i; i < parsedFeeds.length; ++i) {
    _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);
}
```

3. The Router selects reuse for the next order using `oldestPublishTime` (`OrderRouterExecutionSidecar.sol:1084-1085`):

```solidity
commitTime < cache.minReusableCommitTime || commitTime >= cache.oldestPublishTime
    || commitTime >= cache.publishTime
```

4. On the reuse path the Oracle checks **every** feed against the **newest** time (`PletherOracle.sol:263-266`):

```solidity
for (uint256 i; i < pythFeedIds.length; ++i) {
    _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
}
```

`_requireStoredFeedCoverage` reverts when `storedPublishTime < requiredPublishTime` (`:649-651`). Any feed whose first post-commit tick is older than the newest component fails, although storage holds exactly the tick that was parsed and priced.

5. The Router re-raises every Oracle revert except the 68-byte `HistoricalParserFailure`, outside the per-item rollback (`OrderRouterExecutionSidecar.sol:1020-1027`), so the complete `executeOrderBatch` reverts.

Before commit 8191921 the field held the oldest component (`basket.publishTime = uint64(minPublishTime);`), so the comparison at line 265 was correct. PR #120 changed the field to the newest component and added `oldestPublishTime`; its description states that "the oldest is retained for post-commit batch reuse", but line 265 still reads `cache.publishTime`.

The documentation describes a fallback, not a revert: ORACLE_SYNCHRONIZATION.md, "Batch reuse": "Reuse for a later order requires all components to follow that order's commit, fit its historical window and satisfy cache coverage. Otherwise the Router requests a new historical resolution." Here every parsed component is covered in storage; the comparison uses the wrong field.

Trigger conditions: two or more orders in one batch committed before the first basket's oldest component (in practice, orders committed in the same second), and a first post-commit basket whose six FX components do not all share one publish second. Six independent Pyth feeds rarely publish in the same second, and same-second commits are normal under load on Arbitrum. The repository's tests use identical publish times on all feeds, which is why the failure was not caught.

## Impact Details

`executeOrderBatch`, the public keeper entry point for executing queued orders, fails for valid orders and delivers nothing. A retry with the same payload later in the window fails identically, because the revert also undoes the first order's storage synchronization. Keepers must split batches or append extra later ticks to make progress; orders whose keeper keeps batching wait until their deadline (60 s at release) and expire. No funds are lost and orders remain executable one at a time through `executeOrder`.

Selected impact: Low, "Contract fails to deliver promised returns, but doesn't lose value".

Recommended fix: in the reuse branch compare against `cache.oldestPublishTime` (or each feed's own cached component time):

```solidity
_requireStoredFeedCoverage(pythFeedIds[i], cache.oldestPublishTime);
```

or have the Router skip reuse and request a new historical resolution when coverage would fail, as the documentation describes.

## References

- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L263-L266
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L623-L626
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L874-L875
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1020-L1027
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1084-L1085
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md
- Plether-Fi/plether-core PR #120 (commit 8191921), fix for Cecuro MED-7

## Proof of Concept

## Proof of Concept

The test uses the repository's own `OracleSynchronizationTestBase` (real Router, execution sidecar and PletherOracle; `MockPyth` only supplies Pyth updates). No in-scope contract is modified.

Steps:

1. Save the file below as `packages/perps/test/perps/poc/BatchCoverageNewestTime.t.sol` at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8.
2. Run from the repository root: `forge test --root packages/perps --match-path test/perps/poc/BatchCoverageNewestTime.t.sol -vv` (setting `FOUNDRY_VIA_IR=false` makes the compile much faster).
3. Expected: 5 passed.

What the tests show:

- `test_BatchRevertsStoredFeedBehindOnMixedFirstTicks`: ALICE and BOB commit in the same second. In the first post-commit payload, feed 1 ticks at commit+2 and feeds 2..6 at commit+1 (all valid unique ticks). `executeOrderBatch` reverts with `PletherOracle__StoredFeedBehind(feed 2, commit+1, commit+2)`. ALICE's fill is rolled back and both orders stay pending. The same batch reverts again at commit+12.
- `test_Control_SameOrdersExecuteIndividually`: the same orders with the same payload execute one by one through `executeOrder`; both fill.
- `test_Control_EqualComponentTimesBatchSucceeds`: with equal component times the batch reuses the cached basket and fills both orders.
- `test_RouteAround_SplitBatch` and `test_RouteAround_AppendLaterTick`: the two workarounds a keeper needs today.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {OracleSynchronizationTestBase} from "../shared/OracleSynchronizationTestBase.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice Batch cache reuse checks Pyth storage of every feed against the NEWEST component time
///         (PletherOracle.sol:265, cache.publishTime = maxPublishTime at :874), while the first order of the batch
///         only synchronizes each feed up to its OWN tick (:623-626). Two orders committed in the same second and a
///         first-tick basket whose component seconds differ make the reused order revert StoredFeedBehind; the
///         Router re-raises it (OrderRouterExecutionSidecar.sol:1020-1027) and the whole executeOrderBatch reverts.
contract BatchCoverageNewestTimeTest is OracleSynchronizationTestBase {

    /// @dev First post-commit ticks: feed 1 first ticks at commit+2 (no aggregate in second commit+1), feeds 2..6 at
    ///      commit+1. Every update is a valid unique tick (previous publish time = commit < commit+1).
    function _mixedFirstTicks(
        uint64 commit
    ) internal pure returns (bytes[] memory data) {
        data = _payload(commit + 1, commit);
        MockPyth.FeedUpdate[] memory u = abi.decode(data[0], (MockPyth.FeedUpdate[]));
        u[0].price.publishTime = commit + 2;
        data[0] = abi.encode(u);
    }

    /// @dev Victim and a second trader (any account) commit in the same second.
    function _sameSecondPair() internal returns (uint64 commit, uint64 a, uint64 b) {
        commit = uint64(block.timestamp);
        a = _commit(ALICE, 0);
        b = _commit(BOB, 0);
    }

    function _status(
        uint64 id
    ) internal view returns (OrderV3Types.LifecycleStatus) {
        return router.lifecycleBook().lifecycleStatus(id);
    }

    function _behind(
        uint64 commit
    ) internal pure returns (bytes memory) {
        return abi.encodeWithSelector(
            IPletherOracle.PletherOracle__StoredFeedBehind.selector,
            bytes32(uint256(2)),
            uint256(commit + 1),
            uint256(commit + 2)
        );
    }

    /// Bug: the batch reverts with StoredFeedBehind although storage covers every parsed component's own tick, the
    /// executed prefix (order a) is rolled back, and a retry later inside the settlement window fails identically.
    function test_BatchRevertsStoredFeedBehindOnMixedFirstTicks() public {
        (uint64 commit, uint64 a, uint64 b) = _sameSecondPair();
        _advance(commit + 3);
        bytes[] memory data = _mixedFirstTicks(commit);
        uint256 parses = baseMockPyth.parseUniqueCallCount();
        uint256 storedFeed2 = baseMockPyth.getPriceUnsafe(bytes32(uint256(2))).publishTime;

        vm.expectRevert(_behind(commit));
        router.executeOrderBatch{value: 4 * FEE}(b, data);

        assertEq(router.nextExecuteId(), a, "prefix rolled back: order a is still the head");
        assertEq(uint256(_status(a)), uint256(OrderV3Types.LifecycleStatus.Pending), "a pending");
        assertEq(uint256(_status(b)), uint256(OrderV3Types.LifecycleStatus.Pending), "b pending");
        (uint256 aliceSize,,,,,,) = engine.positions(ALICE);
        assertEq(aliceSize, 0, "a did not fill");
        assertEq(baseMockPyth.parseUniqueCallCount(), parses, "parse rolled back");
        assertEq(
            baseMockPyth.getPriceUnsafe(bytes32(uint256(2))).publishTime, storedFeed2, "storage sync rolled back"
        );

        // Deterministic: the revert also undid a's storage sync, so the same batch fails again later in the window.
        _advance(commit + 12);
        vm.expectRevert(_behind(commit));
        router.executeOrderBatch{value: 4 * FEE}(b, data);
    }

    /// Control 1: the same two orders, the same payload, executed one by one, both fill.
    function test_Control_SameOrdersExecuteIndividually() public {
        (uint64 commit, uint64 a, uint64 b) = _sameSecondPair();
        _advance(commit + 3);
        bytes[] memory data = _mixedFirstTicks(commit);

        router.executeOrder{value: 2 * FEE}(a, data);
        router.executeOrder{value: 2 * FEE}(b, data);

        assertEq(uint256(_status(a)), uint256(OrderV3Types.LifecycleStatus.Executed), "a executed");
        assertEq(uint256(_status(b)), uint256(OrderV3Types.LifecycleStatus.Executed), "b executed");
        (uint256 aliceSize,,,,,,) = engine.positions(ALICE);
        (uint256 bobSize,,,,,,) = engine.positions(BOB);
        assertGt(aliceSize, 0);
        assertGt(bobSize, 0);
        assertEq(engine.lastMarkTime(), commit + 2, "mark at the newest component");
    }

    /// Control 2: identical flow with equal component times; the batch reuses the cache and fills both orders.
    function test_Control_EqualComponentTimesBatchSucceeds() public {
        (uint64 commit, uint64 a, uint64 b) = _sameSecondPair();
        _advance(commit + 3);
        bytes[] memory data = _payload(commit + 1, commit);
        uint256 parses = baseMockPyth.parseUniqueCallCount();

        OrderV3Types.BatchResult memory result = router.executeOrderBatch{value: 4 * FEE}(b, data);

        assertEq(result.terminalCount, 2);
        assertEq(result.nextOrderId, 0);
        assertEq(baseMockPyth.parseUniqueCallCount(), parses + 1, "order b reused the cached basket");
        assertEq(uint256(_status(a)), uint256(OrderV3Types.LifecycleStatus.Executed));
        assertEq(uint256(_status(b)), uint256(OrderV3Types.LifecycleStatus.Executed));
    }

    /// a keeper routes around by bounding the batch at a (b then parses fresh in a new call).
    function test_RouteAround_SplitBatch() public {
        (uint64 commit, uint64 a, uint64 b) = _sameSecondPair();
        _advance(commit + 3);
        bytes[] memory data = _mixedFirstTicks(commit);

        OrderV3Types.BatchResult memory first = router.executeOrderBatch{value: 2 * FEE}(a, data);
        assertEq(first.terminalCount, 1);
        OrderV3Types.BatchResult memory second = router.executeOrderBatch{value: 2 * FEE}(b, data);
        assertEq(second.terminalCount, 1);
        assertEq(uint256(_status(b)), uint256(OrderV3Types.LifecycleStatus.Executed));
    }

    /// appending any later tick of every feed to the payload makes a's synchronization store it, so the
    /// reuse coverage holds and the single batch fills both orders at the first-tick price.
    function test_RouteAround_AppendLaterTick() public {
        (uint64 commit, uint64 a, uint64 b) = _sameSecondPair();
        _advance(commit + 3);
        MockPyth.FeedUpdate[] memory first = abi.decode(_mixedFirstTicks(commit)[0], (MockPyth.FeedUpdate[]));
        MockPyth.FeedUpdate[] memory later = abi.decode(_payload(commit + 3, commit + 2)[0], (MockPyth.FeedUpdate[]));
        MockPyth.FeedUpdate[] memory both = new MockPyth.FeedUpdate[](first.length + later.length);
        for (uint256 i; i < first.length; ++i) {
            both[i] = first[i];
            both[first.length + i] = later[i];
        }
        bytes[] memory data = new bytes[](1);
        data[0] = abi.encode(both);

        OrderV3Types.BatchResult memory result = router.executeOrderBatch{value: 4 * FEE}(b, data);
        assertEq(result.terminalCount, 2);
        assertEq(uint256(_status(a)), uint256(OrderV3Types.LifecycleStatus.Executed));
        assertEq(engine.lastMarkTime(), commit + 2, "fill still uses the first post-commit basket");
    }

}
```
