# \#97643 [SC-Medium] Batch cache reuse skips the Oracle's stored-feed coverage check, so a staggered basket reverts the whole `executeOrderBatch` instead of falling back to a fresh parse

**Submitted on Nov 2nd 2026 at 15:21:05 UTC by @farismaulana for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97643
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

## Brief/Intro

`executeOrderBatch` reuses the historical basket parsed for an earlier order when a later FIFO order qualifies. The Router decides reuse with a predicate that omits the Oracle's stored-feed coverage requirement. When the cached basket's six components carry different publish times, the Oracle rejects the reuse with `PletherOracle__StoredFeedBehind`. The Router does not fall back to a new historical resolution as `ORACLE_SYNCHRONIZATION.md` describes. It re-raises the error, which reverts the entire batch, including every order the batch had already finalized.

## Vulnerability Details

The cached basket's `publishTime` is the newest component time, and `oldestPublishTime` is the oldest:

```solidity
@>      basket.publishTime = uint64(maxPublishTime);
        basket.oldestPublishTime = uint64(minPublishTime);
```

On the fresh historical parse, after `updatePriceFeeds`, the Oracle checks each feed only against that feed's own parsed time:

```solidity
            pyth.updatePriceFeeds{value: pythFee}(pythUpdateData);
            for (uint256 i; i < parsedFeeds.length; ++i) {
@>              _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);
            }
```

On reuse, `updateBatchOrderExecutionPrice` requires every stored feed to be at or after `cache.publishTime`, which is the newest component:

```solidity
        if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
            // Cache contents are caller-supplied; independently prove storage coverage, not price authenticity.
            for (uint256 i; i < pythFeedIds.length; ++i) {
@>              _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
            }
```

`_requireStoredFeedCoverage` reverts `PletherOracle__StoredFeedBehind(feedId, storedPublishTime, requiredPublishTime)` when the stored time is lower.

The Router-side predicate in `OrderRouterExecutionSidecar._canReuseHistoricalBatchBasket` copies the Oracle's reuse conditions (frozen, `hasHistoricalBasket`, `minReusableCommitTime <= commit < oldestPublishTime`, `commit < publishTime`, `publishTime <= block.timestamp`, settlement window). It has no stored-feed coverage pre-check, so it selects reuse and forwards zero fee:

```solidity
        uint256 fee = 0;
@>      if (!_canReuseHistoricalBatchBasket(oracle, order, state.oracleCache)) {
            fee = oracle.getOrderExecutionFee(pythUpdateData);
```

The catch around the Oracle call tolerates only the 68-byte `PletherOracle__HistoricalParserFailure`. Anything else is re-raised:

```solidity
        } catch (bytes memory reason) {
            if (
@>              reason.length != 68
                    || _selector(reason) != IPletherOracle.PletherOracle__HistoricalParserFailure.selector
            ) {
                assembly ("memory-safe") {
@>                  revert(add(reason, 32), mload(reason))
                }
            }
```

`_prepareBatchOracle` runs in the outer batch frame, not inside the per-item `try`. The revert therefore unwinds the whole `executeOrderBatch` call, including items already executed and finalized earlier in the loop.

Example, with two orders committed in the same second `S`:

1. order A and order B both commit at `S`, adjacent in the FIFO queue.
2. the unique post-commit payload for A carries EUR at `S + 1` and the other five feeds at `S + 3` (inside the 5 s divergence limit).
3. item A: fresh parse, `updatePriceFeeds` stores EUR at `S + 1`, per-feed coverage passes, and A executes and finalizes. The cache becomes `publishTime = S + 3`, `oldestPublishTime = S + 1`.
4. item B: `commitTime = S < oldestPublishTime = S + 1`, so the Router picks reuse with fee 0.
5. the Oracle requires stored EUR `>= S + 3`, finds `S + 1`, and reverts `StoredFeedBehind(EUR, S + 1, S + 3)`.
6. the Router re-raises it, and the whole batch reverts, including A's execution.

The project's design document describes the opposite behaviour for this exact condition. `ORACLE_SYNCHRONIZATION.md:61-63` says reuse "requires all components to follow that order's commit, fit its historical window and satisfy cache coverage. Otherwise the Router requests a new historical resolution." The Router never requests a new resolution, because it never evaluates coverage. The failed-coverage case surfaces as the transaction-level revert listed at `ORACLE_SYNCHRONIZATION.md:109`.

Fix: evaluate stored-feed coverage in the Router predicate, using a view on the Oracle that mirrors `_requireStoredFeedCoverage` against `cache.publishTime`. When coverage fails, treat the order as not reusable and request a fresh historical resolution. That matches the documented flow and keeps the batch prefix intact.

## Impact Details

1. every `executeOrderBatch` range that contains a reuse-eligible pair under staggered component times reverts deterministically, rolling back all orders the batch already executed and its pre-oracle cleanup.
2. the keeper pays the gas of the entire reverted batch on every attempt over that range, and no order in the range settles through the batch path.
3. any trader can create the triggering pair by committing in the same second as another order, because order commits are permissionless.
4. the implemented behaviour contradicts the protocol's own synchronization specification, so keeper operators following `ORACLE_SYNCHRONIZATION.md` expect a fallback that does not exist.

## References

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L262-L266

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L618-L626

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L645-L653

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L874-L875

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L974-L991

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L292-L301

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L995-L1028

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1074-L1093

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md#L58-L64

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md#L105-L110


## Proof of Concept

## Proof of Concept

this insight applies whenever `executeOrderBatch` reaches an order that committed after the previously parsed order but before the earliest post-commit Pyth tick in that order's basket (`A.commitTime <= B.commitTime < oldestPublishTime`), which any trader produces by committing in the same second as another order, and that basket's six feeds carry different publish times. every batch over such a range reverts and rolls back its already-executed orders.

the test uses the repository's `OracleSynchronizationTestBase` fixture. two orders commit before the oldest component tick, and the post-commit payload publishes feed 0 at `+4` and the remaining feeds at `+5`. a uniform-time control runs the same flow.

create `packages/perps/test/perps/spec/oracle/BatchCacheCoverageRevert.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {OracleSynchronizationTestBase} from "../../shared/OracleSynchronizationTestBase.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

contract BatchCacheCoverageRevertTest is OracleSynchronizationTestBase {

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

    function test_staggeredBasketReuseRevertsWholeBatch() public {
        uint64 firstCommit = uint64(block.timestamp);
        _commit(ALICE, 0);
        _advance(firstCommit + 3);
        _commit(BOB, 0);
        _advance(firstCommit + 6);

        bytes[] memory historical = _pricedPayload(firstCommit + 5, firstCommit, 101_000_000);
        MockPyth.FeedUpdate[] memory updates = abi.decode(historical[0], (MockPyth.FeedUpdate[]));
        updates[0].price.publishTime = firstCommit + 4;
        bytes32 laggingFeed = updates[0].id;
        historical[0] = abi.encode(updates);

        uint64 headBefore = router.nextExecuteId();
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector,
                laggingFeed,
                uint256(firstCommit + 4),
                uint256(firstCommit + 5)
            )
        );
        router.executeOrderBatch{value: 2 * FEE}(2, historical);

        // The first order, which the batch had executed before reaching the second, was rolled back.
        assertEq(router.nextExecuteId(), headBefore, "batch prefix rolled back: first order still at the queue head");
    }

    function test_control_uniformBasketReuseSucceeds() public {
        uint64 firstCommit = uint64(block.timestamp);
        _commit(ALICE, 0);
        _advance(firstCommit + 3);
        _commit(BOB, 0);
        _advance(firstCommit + 6);

        bytes[] memory historical = _pricedPayload(firstCommit + 5, firstCommit, 101_000_000);
        OrderV3Types.BatchResult memory result = router.executeOrderBatch{value: 2 * FEE}(2, historical);
        assertEq(result.terminalCount, 2, "uniform basket: batch executes both orders");
        assertEq(baseMockPyth.parseUniqueCallCount(), 1, "second order reused the cached basket");
    }

}
```

run from the repository root:

```bash
forge test --root packages/perps --match-path test/perps/spec/oracle/BatchCacheCoverageRevert.t.sol -vv
```

```bash
Ran 2 tests for test/perps/spec/oracle/BatchCacheCoverageRevert.t.sol:BatchCacheCoverageRevertTest
[PASS] test_control_uniformBasketReuseSucceeds() (gas: 4224894)
[PASS] test_staggeredBasketReuseRevertsWholeBatch() (gas: 4373911)
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

the `-vvvv` trace of the staggered case shows the first order opening inside the batch, then the second order taking the reuse branch with the cache's newest component as the required time, and the revert unwinding the whole batch:

```bash
    │   │   │   │   │   ├─ emit PositionOpened(account: 0x00000000000000000000000000000000000A11cE, side: 0, sizeDelta: 10000000000000000000000 [1e22], price: 100980000 [1.009e8], marginDelta: 1000000000 [1e9])
    ...
    │   │   ├─ [17860] PletherOracle::updateBatchOrderExecutionPrice(..., OrderExecutionRequest({ commitTime: 1709794803 [1.709e9], ... }), BatchOrderPriceCache({ hasHistoricalBasket: true, minReusableCommitTime: 1709794800 [1.709e9], price: 101000000 [1.01e8], confidence: 100000 [1e5], publishTime: 1709794805 [1.709e9], oldestPublishTime: 1709794804 [1.709e9] }))
    │   │   │   ├─ [1619] MockPyth::getPriceUnsafe(0x0000000000000000000000000000000000000000000000000000000000000001) [staticcall]
    │   │   │   └─ ← [Revert] PletherOracle__StoredFeedBehind(0x0000000000000000000000000000000000000000000000000000000000000001, 1709794804 [1.709e9], 1709794805 [1.709e9])
    │   │   └─ ← [Revert] PletherOracle__StoredFeedBehind(0x0000000000000000000000000000000000000000000000000000000000000001, 1709794804 [1.709e9], 1709794805 [1.709e9])
    │   └─ ← [Revert] PletherOracle__StoredFeedBehind(0x0000000000000000000000000000000000000000000000000000000000000001, 1709794804 [1.709e9], 1709794805 [1.709e9])
```

with the same payload, the staggered basket rolls back the order the batch had already executed and reverts the whole call, while the uniform-time control executes both orders in one batch with a single historical parse.

