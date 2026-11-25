# \#97839 [SC-Medium] `executeOrderBatch` reverts the whole batch when a reused price basket's components carry different publish times and Pyth storage has not been refreshed

**Submitted on Nov 3rd 2026 at 14:19:24 UTC by @Afriauditor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97839
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description


## Summary

`executeOrderBatch` lets later orders in a batch reuse the first order's historical price basket instead of parsing a new one. Before reusing it, the oracle requires every component's *stored* Pyth price to be at least as new as the basket's newest component. The first order's own parse writes each component into storage at its own publish time, so unless something newer has been pushed for the lagging feeds, the check fails whenever the components were published in different seconds — a case the protocol explicitly expects and allows a five-second window for. The `StoredFeedBehind` revert isn't caught by the batch loop, so the entire transaction reverts, including the order that had already executed. The documentation contradicts itself here: one section says a reuse that fails coverage falls back to a fresh price fetch, another says it reverts the transaction; the code reverts. The same orders execute normally one at a time, so no funds are at risk.

Severity: **Low** — the contract fails to deliver documented behaviour and the batch-reuse path doesn't work in its ordinary case, but no value is lost or frozen and the only cost falls on the keeper.

## Description

When an order needs a fresh price, `PletherOracle` parses the keeper's payload, writes it to Pyth storage, and checks each component against its *own* publish time (`PletherOracle.sol:623-626`):

```solidity
pyth.updatePriceFeeds{value: pythFee}(pythUpdateData);
for (uint256 i; i < parsedFeeds.length; ++i) {
    _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);
}
```

When a later order in the same batch reuses that basket, every component is instead checked against `cache.publishTime`, the *newest* component's time (`PletherOracle.sol:262-266`):

```solidity
if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
    for (uint256 i; i < pythFeedIds.length; ++i) {
        _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
    }
```

`_requireStoredFeedCoverage` reverts with `StoredFeedBehind` when the stored time is older than the required one (`645-652`). If one component published at t+4 and the other five at t+5, the first order stores that component at t+4 and the second order's reuse demands t+5 from it. The Router's batch loop only catches `HistoricalParserFailure` from the oracle (`OrderRouterExecutionSidecar.sol:1019-1030`); anything else is re-raised, so the whole `executeOrderBatch` call reverts and the first order returns to the head of the queue.

Components publishing in different seconds is the normal case, not an edge case. The docs note that feeds "publish at different times" (`SECURITY.md:697-698`) and allow a `maxComponentPublishTimeDivergence` of five seconds by default (`ORACLE_SYNCHRONIZATION.md:35`). The check only passes if Pyth storage already holds newer prices for the lagging feeds — because another Pyth consumer on the chain pushed them, or because the keeper pushed them itself. The team's own `test_BatchReusesBasketWhenEveryComponentFollowsBothCommits` passes for exactly that reason: it pushes a newer payload into storage before running the batch (`OracleSynchronization.t.sol:445-446`). Remove that line and the same batch reverts. A keeper that supplies only commit-window ticks, which is all the single-order path needs, hits the revert.

The documentation is inconsistent about what should happen. `ORACLE_SYNCHRONIZATION.md:61-63` says reuse "requires all components to follow that order's commit, fit its historical window and satisfy cache coverage. Otherwise the Router requests a new historical resolution." For the first two conditions that is what happens — `_canReuseHistoricalBatchBasket` falls back to a fresh parse. For cache coverage it is not: the Router never checks coverage before reusing, the oracle enforces it and reverts, and `ORACLE_SYNCHRONIZATION.md:109` separately records that outcome as "reverts the transaction, including earlier batch progress." The code follows line 109, and line 87 still presents reuse as a working path that "costs zero without making a new quote."

Fix: carry each component's publish time in `BatchOrderPriceCache` and check each leg against its own time, as the fresh-fetch path does; or treat a failed coverage check as "not reusable" and fall through to a fresh parse, as lines 61-63 describe — in either case catching `StoredFeedBehind` in `_prepareBatchOracle` so one failed reuse cannot roll back finalized predecessors.

## Impact

Low. The contract fails to deliver documented behaviour, but no value is lost.

- A batch in which a later order reuses the first order's basket reverts whenever the components were published in different seconds and nothing has pushed newer prices for the lagging feeds into Pyth storage. The keeper either executes those orders one per transaction, paying a parse and Pyth fee each time, or arranges for storage to be current first — which costs extra Pyth update units and, for a two-order batch, roughly what reuse was meant to save.
- A failing batch rolls back every order in it, including ones that had already executed; those orders stay queued and are not lost.
- No trader or LP is affected. `executeOrder` accepts the same basket, and liquidations, protection triggers and LP settlement don't use this path.

The behaviour at line 109 is documented, which is why this is a Low about an internal contradiction and a non-functional optimisation, not a liveness or funds issue.



## Proof of Concept


The PoC reuses the exact fixture of the team's `test_BatchReusesBasketWhenEveryComponentFollowsBothCommits`: two queued orders and one signed basket in which one component published one second before the other five. The only difference between the control and the failing test is the team's pre-push of newer prices into Pyth storage. A third test shows the single-order path accepting the same basket.

Place at `packages/perps/test/perps/poc/PoC_BatchReuseMixedTimestamps.t.sol` and run:

```
forge test --root packages/perps --match-path test/perps/poc/PoC_BatchReuseMixedTimestamps.t.sol -vv
```

Result at `a82d4fe`: 3 passed, 0 failed (reproduced independently; file attached).

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {OracleSynchronizationTestBase} from "../shared/OracleSynchronizationTestBase.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

contract PoC_BatchReuseMixedTimestamps is OracleSynchronizationTestBase {

    function _twoOrdersAndMixedBasket() internal returns (bytes[] memory historical, uint64 firstCommit) {
        firstCommit = uint64(block.timestamp);
        _commit(ALICE, 0); // order 1
        _advance(firstCommit + 3);
        _commit(BOB, 0); // order 2
        _advance(firstCommit + 6);

        // One signed basket after both commits: legs 1-5 at t+5, leg 0 at t+4 (1 s skew, inside the 5 s default).
        historical = _pricedPayload(firstCommit + 5, firstCommit, 101_000_000);
        MockPyth.FeedUpdate[] memory updates = abi.decode(historical[0], (MockPyth.FeedUpdate[]));
        updates[0].price.publishTime = firstCommit + 4;
        historical[0] = abi.encode(updates);
    }

    /// Control: the team's test as written. Passes only because Pyth storage is pushed to t+6 first.
    function test_Control_TeamFixtureWithPrePush_BatchReuses() public {
        (bytes[] memory historical, uint64 firstCommit) = _twoOrdersAndMixedBasket();
        bytes[] memory newer = _pricedPayload(firstCommit + 6, firstCommit + 5, 102_000_000);
        baseMockPyth.updatePriceFeeds{value: FEE}(newer); // <- storage refresh the batch itself does not perform

        OrderV3Types.BatchResult memory result = router.executeOrderBatch{value: 2 * FEE}(2, historical);
        assertEq(result.terminalCount, 2, "both orders execute from one parse");
        assertEq(result.nextOrderId, 0, "queue drained");
    }

    /// Same batch without the refresh: order 2's reuse demands t+5 from leg 0, which storage holds at t+4.
    function test_NaturalFlow_SameBatchWithoutPrePush_RevertsEntirely() public {
        (bytes[] memory historical, uint64 firstCommit) = _twoOrdersAndMixedBasket();
        bytes32 laggingLeg = _synchronizationFeedIds()[0];

        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector, laggingLeg, firstCommit + 4, firstCommit + 5
            )
        );
        router.executeOrderBatch{value: 2 * FEE}(2, historical);

        assertEq(router.nextExecuteId(), 1, "order 1 is back at the head: its execution was rolled back too");
    }

    /// The single-order path accepts the very same basket.
    function test_SinglePath_AcceptsTheSameMixedBasket() public {
        (bytes[] memory historical,) = _twoOrdersAndMixedBasket();

        OrderV3Types.ExecutionResult memory first = router.executeOrder{value: 2 * FEE}(1, historical);
        OrderV3Types.ExecutionResult memory second = router.executeOrder{value: 2 * FEE}(2, historical);

        assertEq(uint256(first.status), uint256(OrderV3Types.LifecycleStatus.Executed), "order 1 executes alone");
        assertEq(uint256(second.status), uint256(OrderV3Types.LifecycleStatus.Executed), "order 2 executes alone");
        assertEq(router.nextExecuteId(), 0, "queue drained, one Pyth parse per order");
    }

    function _pricedPayload(uint64 tick, uint64 previous, int64 price) private pure returns (bytes[] memory data) {
        data = _payload(tick, previous);
        MockPyth.FeedUpdate[] memory updates = abi.decode(data[0], (MockPyth.FeedUpdate[]));
        for (uint256 i; i < updates.length; ++i) {
            updates[i].price.price = price;
        }
        data[0] = abi.encode(updates);
    }
}
```


