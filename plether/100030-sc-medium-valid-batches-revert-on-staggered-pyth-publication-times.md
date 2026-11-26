# \#100030 [SC-Medium] Valid batches revert on staggered Pyth publication times

**Submitted on Nov 16th 2026 at 05:21:47 UTC by @gandu for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100030
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
Two valid delayed orders can fail as a batch when their signed Pyth components publish a few seconds apart. The first order accepts the basket, but reuse for the second applies a stricter timestamp condition and reverts the whole transaction. On mainnet, this wastes keeper gas and delays a valid trader order, potentially until another executor retries it individually.

## Vulnerability Details
1. Delayed-order pricing uses the first eligible post-commit Pyth update for each currency-basket component.
2. The six components may have different publication times if their spread stays within the configured divergence limit.
3. During batch execution, the Router can keep the basket resolved for the first order and offer it to a later eligible order.
4. The first historical resolution checks each stored feed against that feed's own parsed publication time.
5. In the PoC, Alice commits at t, Mallory at t+1, one component publishes at t+2, and the other five at t+4.
6. Every update follows both commitments, and the two-second separation is within the permitted five-second component divergence.
7. Alice's order resolves successfully. The resulting cache records the basket's newest publication time, t+4.
8. The Router accepts that cache for Mallory's order, but updateBatchOrderExecutionPrice requires all stored feeds to cover the cache's t+4 time.
9. The component stored at t+2 fails the new requirement with StoredFeedBehind although it was valid for the first resolution.
10. The revert unwinds the entire batch, including Alice's valid execution. Separate single-order calls with the same signed payload succeed, and an equal-time control succeeds as a batch.

~~~solidity
for (uint256 i; i < pythFeedIds.length; ++i) {
    _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
}
~~~

## Impact Details
The demonstrated result is a failed batch transaction: the keeper pays gas, Alice's valid order remains at the FIFO head, and both orders remain pending. A new executor can recover by processing them separately. If no successful retry occurs before an order deadline, an affected order may expire, but the PoC proves immediate disruption, not guaranteed expiry or loss of trader funds. This supports confirmed Medium Griefing without claiming a permanent queue lock.




## Proof of Concept

## Proof of Concept
Place Oracle_BatchCacheCoverageRevert.t.sol at packages/perps/test/poc/. The attack test checks StoredFeedBehind, unchanged pending orders, and successful separate executions. The control makes component publication times equal and confirms batch reuse succeeds. No in-scope source contract is changed.

~~~bash
forge test --root packages/perps --offline --match-path 'test/poc/Oracle_BatchCacheCoverageRevert.t.sol' -vv
~~~

The submitted report records 2 passed, 0 failed. This suite was not rerun for this rewrite.


```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {OracleSynchronizationTestBase} from "../perps/shared/OracleSynchronizationTestBase.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice Batch cache reuse checks every stored feed against the NEWEST component time, while a fresh
///         historical resolution checks each feed against its own parsed time. With honest unequal component
///         times and a payload carrying only the commit-window ticks, the Router decides to reuse (time rules
///         pass, fee 0) and the Oracle reverts StoredFeedBehind, which the batch re-throws, reverting the
///         whole batch including the first, already-valid execution. The same orders execute fine one by one.
contract Oracle_BatchCacheCoverageRevert is OracleSynchronizationTestBase {

    function test_BatchRevertsOnHonestUnequalComponentTimesButSingleExecutionsSucceed() public {
        uint64 firstCommit = uint64(block.timestamp);
        uint64 aliceId = _commit(ALICE, 0);
        _advance(firstCommit + 1);
        uint64 bobId = _commit(BOB, 0); // later order, commit strictly before the oldest component
        _advance(firstCommit + 6);

        // First post-commit ticks: feed0 publishes at +2, feeds1..5 publish at +4 (2s apart, < 5s divergence).
        bytes[] memory historical = _payload(firstCommit + 4, firstCommit);
        MockPyth.FeedUpdate[] memory updates = abi.decode(historical[0], (MockPyth.FeedUpdate[]));
        updates[0].price.publishTime = firstCommit + 2;
        historical[0] = abi.encode(updates);

        uint256 keeperBefore = address(this).balance;
        // Funding for two independent resolutions is supplied; reuse would need none.
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector,
                updates[0].id,
                uint256(firstCommit + 2),
                uint256(firstCommit + 4)
            )
        );
        router.executeOrderBatch{value: 4 * FEE}(bobId, historical);

        // Nothing progressed: both orders still pending, Alice's valid fill rolled back.
        assertEq(router.nextExecuteId(), aliceId, "batch made no progress");
        assertEq(uint256(router.lifecycleBook().lifecycleStatus(aliceId)), uint256(OrderV3Types.LifecycleStatus.Pending));
        assertEq(uint256(router.lifecycleBook().lifecycleStatus(bobId)), uint256(OrderV3Types.LifecycleStatus.Pending));
        (uint256 aliceSize,,,,,,) = engine.positions(ALICE);
        assertEq(aliceSize, 0);
        assertEq(address(this).balance, keeperBefore, "reverted call returns ETH; only gas is lost");

        // The identical payload executes both orders when submitted one at a time (fresh per-feed coverage).
        OrderV3Types.ExecutionResult memory r1 = router.executeOrder{value: 2 * FEE}(aliceId, historical);
        assertEq(uint256(r1.status), uint256(OrderV3Types.LifecycleStatus.Executed));
        OrderV3Types.ExecutionResult memory r2 = router.executeOrder{value: 2 * FEE}(bobId, historical);
        assertEq(uint256(r2.status), uint256(OrderV3Types.LifecycleStatus.Executed));
    }

    /// @notice Control: with equal component times the same two-order batch succeeds via reuse.
    function test_ControlEqualComponentTimesBatchSucceeds() public {
        uint64 firstCommit = uint64(block.timestamp);
        _commit(ALICE, 0);
        _advance(firstCommit + 1);
        uint64 bobId = _commit(BOB, 0);
        _advance(firstCommit + 6);
        bytes[] memory historical = _payload(firstCommit + 4, firstCommit);
        OrderV3Types.BatchResult memory result = router.executeOrderBatch{value: 4 * FEE}(bobId, historical);
        assertEq(result.terminalCount, 2);
        assertEq(baseMockPyth.parseUniqueCallCount(), 1);
    }

}

```

