# \#97883 [SC-Medium] Keeper batches revert and roll back valid fills whenever a basket's component ticks differ (regression from the newest-component-time change)

**Submitted on Nov 3rd 2026 at 18:18:04 UTC by @W3B_SR_0xki for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97883
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description


## Summary

`OrderRouter.executeOrderBatch` is supposed to price later FIFO orders from the signed historical basket it has just parsed, so a batch pays for one Pyth parse instead of one per order.

The Oracle's reuse branch checks that *every* stored Pyth feed has reached `cache.publishTime`. Since the newest-component-time change, that is the basket's **newest** component time. Pyth stores an update only when it is strictly newer than the stored one. A component that ticked even one second earlier therefore stays below that time, although the same transaction has just proved that feed's coverage against its own tick.

The reuse check then reverts with `StoredFeedBehind`. The Router rethrows the error, and the whole batch reverts, including orders it had already filled.

The trigger is valid, in-policy data: six independently published feeds whose first ticks after the commit fall in different seconds, within the oracle's own 5-second divergence limit. Batches over such data cannot complete. Keepers must fall back to one call per order, paying a full historical parse and Pyth fee each time, or the queued orders sit until they expire.

## Root cause

1. **Fresh branch, order 1: each feed is checked against its own tick.** `_resolveHistoricalOrderBasket` parses the unique first-after-commit ticks, stores them with `pyth.updatePriceFeeds`, and then checks each feed only against its own tick:

   ```solidity
   // PletherOracle.sol:624-626
   for (uint256 i; i < parsedFeeds.length; ++i) {
       _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);
   }
   ```

   The basket it builds stores the newest component time as `publishTime` and the oldest separately:

   ```solidity
   // PletherOracle.sol:874-875
   basket.publishTime = uint64(maxPublishTime);
   basket.oldestPublishTime = uint64(minPublishTime);
   ```

2. **Reuse branch, order 2: every feed is checked against the newest time.**

   ```solidity
   // PletherOracle.sol:262-266  (updateBatchOrderExecutionPrice)
   if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
       for (uint256 i; i < pythFeedIds.length; ++i) {
           _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);   // newest component time
       }
   ```

   `_requireStoredFeedCoverage` (`:645-653`) reverts when `pyth.getPriceUnsafe(feed).publishTime < required`. The feed that ticked at T+4 is stored at T+4, so it fails against T+5.

3. **The Router cannot fall back.**
   - The Router's reuse test (`OrderRouterExecutionSidecar._canReuseHistoricalBatchBasket`) has no coverage term, so it forwards fee 0 to the reuse branch.
   - Its `catch` rethrows every reason except the 68-byte `HistoricalParserFailure` (`OrderRouterExecutionSidecar.sol:1019-1028`).
   - The 100-byte `StoredFeedBehind` therefore reverts the entire `executeOrderBatch` call.

### This is a regression introduced by a fix
- Commit `64c6ea90` (#103, "synchronize stored Pyth feeds during historical execution") added the coverage check `_requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime)`. At that time, `basket.publishTime = uint64(minPublishTime)`: the **oldest** component time. So the check was correct.
- Commit `81919218` (#120) switched to `basket.publishTime = uint64(maxPublishTime)`. This is the "progress marker uses the newest component timestamp" change described in the Cecuro MED-23 fix. The same commit:
  - added `oldestPublishTime`;
  - moved the reuse *predicate* to `commitTime >= cache.oldestPublishTime`;
  - left the coverage check at `:265` on `cache.publishTime`, which now holds the newest time.

### Why this is not intended behaviour
- **The documented fallback is a fresh resolution, not a revert.** ORACLE_SYNCHRONIZATION.md:61-63: reuse "requires all components to follow that order's commit, fit its historical window and satisfy cache coverage. **Otherwise the Router requests a new historical resolution.**"
- **Reuse is meant to happen inside proven coverage.** SECURITY.md:649: batch execution may reuse a parsed basket for later FIFO orders that fall "within its proven coverage". Here the coverage *was* just proven per feed (`:625`); only the stricter re-check at `:265` fails.
- **The repo's own reuse test needs an undocumented extra step.** `test_BatchReusesBasketWhenEveryComponentFollowsBothCommits` (`test/perps/spec/oracle/OracleSynchronization.t.sol:434-455`) uses exactly this tick shape: feed 1 at +4, the rest at +5. It passes only because lines 445-446 first push a *newer* update for every feed into Pyth storage. No keeper documentation asks keepers to do that.
- **The docs flag this case as untested.** ORACLE_SYNCHRONIZATION.md:123-124 and :145: "the pinned live-basket fixture has equal component times … Release qualification must include real target-chain payloads with unequal component times."
- **The failure table does not cover it.** ORACLE_SYNCHRONIZATION.md:109 ("`StoredFeedBehind` … or failed cache coverage → reverts the transaction") describes storage that genuinely lacks coverage. Here the cache comes from the same transaction and from valid data, and fails only because the check compares against the wrong timestamp.

None of the known-issue sources mention this: the Cecuro report, the SC Audit Studio pre-audit, the AI scan, the Cantina and collaborative reports, or SECURITY.md Known Limitations.

## Conditions under which it triggers

All of the following. Condition 2 is met whenever the first post-commit ticks of the six independently published feeds fall in different seconds. The oracle accepts up to 5 s of spread, and ORACLE_SYNCHRONIZATION.md:123-124 and :145 require release payloads with unequal component times to be qualified.

1. A keeper calls `executeOrderBatch` over at least two orders, and a later order's `commitTime` is before the oldest component tick of the earlier order's basket and within the settlement window. In practice, several orders were committed close together.
2. The first-after-commit ticks of the six feeds are not all in the same second. Each Pyth feed publishes independently, and the oracle accepts up to `maxComponentPublishTimeDivergence` (5 s) of spread.
3. Pyth storage does not already hold a newer update than the basket's newest component time for every lagging feed.

## Impact
**Category:** Low. The contract fails to deliver a promised function (one signed basket pricing later FIFO orders in a batch), but loses no value.

- **The batch reverts as a whole.** The keeper's gas is wasted, and orders that had already filled earlier in the same call are rolled back. The PoC asserts that order 1's position is undone.
- **Batch reuse no longer saves anything.** Keepers must fall back to `executeOrder` per order, with one full historical parse and one Pyth fee per order. The PoC shows two parses where the batch would use one.
- **Orders can lapse.** A keeper that relies on batch execution leaves the queued orders pending, and they expire after the execution window (60 s by default), so users' orders fail.
- **Executing orders one at a time is not a fix.** `executeOrderBatch` is the protocol's documented batching entrypoint, and basket reuse is its advertised saving (ORACLE_SYNCHRONIZATION.md:59-64). The fallback costs one full historical parse and one Pyth fee per order. A batch that hits this path also rolls back fills it had already completed.
- No funds are lost. This is a liveness and efficiency failure of a documented protocol feature. It is filed as Low.

## Proof of concept

Place the file below at `packages/perps/test/perps/spec/oracle/PoC_B1_BatchReuseCoverage.t.sol` and run, from the repository root:

```
git submodule update --init --recursive
forge test --root packages/perps --match-path test/perps/spec/oracle/PoC_B1_BatchReuseCoverage.t.sol -vv
```

**What the PoC uses:**
- The repo's own `OracleSynchronizationTestBase`: the 6-feed oracle and MockPyth.
- MockPyth's `_applyUpdate` (`packages/shared/test-support/MockPyth.sol:200`) stores an update only when it is strictly newer than the stored one. Production Pyth `updatePriceFeeds` applies the same rule. That rule is what stops the lagging feed from catching up within the call.
- Release risk parameters.
- Orders committed through the production V3 `OrderRouter.commitOrder(OrderRequest)`, and executed through the production `executeOrderBatch` and `executeOrder`.
- No in-scope contract is modified. Only Pyth and USDG are mocks.

**Tests:**
1. `test_B1_validBatchRevertsWhenComponentTicksDiffer`: two orders, one signed payload with feed 1 at T+4 and feeds 2-6 at T+5. The batch reverts with `StoredFeedBehind(feed1, T+4, T+5)`, both orders stay pending, and order 1's fill is undone.
2. `test_B1_samePayloadSucceedsOrderByOrder`: the same payload executes both orders one by one, which proves the data is valid and in policy. It costs two parses.
3. `test_B1_equalTicksBatchWorks`: the control. With equal component times the identical batch reuses the basket and fills both orders with one parse.
4. `test_B1_onlyAnExtraNewerPushMakesTheBatchWork`: the batch works only after an extra push of a newer update for every feed, which is the step the repo's own test adds.


## Recommendation

Compare each stored feed against the oldest component time. This restores the check's meaning from before #120: every feed was stored at a tick at least as new as the oldest component.

```diff
-                _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
+                _requireStoredFeedCoverage(pythFeedIds[i], cache.oldestPublishTime);
```

Optionally, also require that at least one feed reaches `cache.publishTime`. Alternatively, add the coverage condition to the Router's `_canReuseHistoricalBatchBasket`, so that a failed reuse falls back to a fresh resolution as ORACLE_SYNCHRONIZATION.md:61-63 describes. Add a test with unequal component times and no extra pre-push.

## Proof of Concept

## Proof of Concept
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// PoC: OrderRouter.executeOrderBatch reverts (including already-valid earlier orders) whenever the historical basket
// it just parsed has unequal component publish times. PletherOracle.updateBatchOrderExecutionPrice's reuse branch
// requires every stored Pyth feed to reach cache.publishTime, which is the NEWEST component time
// (PletherOracle.sol:265 vs :874). Pyth only stores strictly newer updates, so a component that ticked earlier stays
// below that time even though the same transaction just proved its coverage against its own tick (:625).
//
// Uses the repo's own OracleSynchronizationTestBase (6-feed oracle, MockPyth with real "store only if newer" rules).
// Orders are committed through the production V3 OrderRouter.commitOrder(OrderRequest); execution uses the production
// executeOrderBatch / executeOrder. Only Pyth (MockPyth) and USDG are mocks. Release risk parameters.

import {OracleSynchronizationTestBase} from "../../shared/OracleSynchronizationTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

contract PoC_B1_BatchReuseCoverage is OracleSynchronizationTestBase {

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    /// @dev LONG open of 100 lots through the production V3 commit (no legacy adapter).
    function _commitV3(
        address trader
    ) internal returns (uint64 orderId) {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("PoC-B1", router.nextCommitId()));
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = 10_000e18;
        request.marginDelta = 1000e6;
        request.targetPrice = 1; // no price limit for a LONG open
        request.isClose = false;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + window),
            executionWindowSeconds: uint32(window),
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
        vm.prank(trader);
        orderId = router.commitOrder(request);
    }

    /// @dev Two orders 3 s apart, executed at T+6 with ONE signed six-feed payload: feed 1 ticks at T+4, feeds 2-6 at
    ///      T+5 (1 s apart, inside the 5 s divergence limit). Every tick follows both commits and fits both windows.
    function _twoOrdersAndPayload(
        bool unequalTicks
    ) internal returns (uint64 firstCommit, uint64 aliceId, uint64 bobId, bytes[] memory historical) {
        firstCommit = uint64(vm.getBlockTimestamp());
        aliceId = _commitV3(ALICE);
        _advance(firstCommit + 3);
        bobId = _commitV3(BOB);
        _advance(firstCommit + 6);
        historical = _payload(firstCommit + 5, firstCommit);
        if (unequalTicks) {
            MockPyth.FeedUpdate[] memory updates = abi.decode(historical[0], (MockPyth.FeedUpdate[]));
            updates[0].price.publishTime = firstCommit + 4;
            historical[0] = abi.encode(updates);
        }
    }

    /// @notice The bug: a keeper batch over valid, in-policy data reverts in the reuse coverage check, and the revert
    ///         also rolls back order 1, which the same transaction had already executed successfully.
    function test_B1_validBatchRevertsWhenComponentTicksDiffer() public {
        (uint64 t, uint64 aliceId, uint64 bobId, bytes[] memory historical) = _twoOrdersAndPayload(true);
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector,
                _synchronizationFeedIds()[0],
                uint256(t + 4),
                uint256(t + 5)
            )
        );
        router.executeOrderBatch{value: 2 * FEE}(bobId, historical);
        // Nothing executed: both orders are still pending and Alice's order is still the FIFO head.
        assertEq(router.nextExecuteId(), aliceId, "order 1 rolled back");
        assertEq(router.pendingOrderCounts(ALICE), 1);
        assertEq(router.pendingOrderCounts(BOB), 1);
        (uint256 aliceSize,,,,,,) = engine.positions(ALICE);
        assertEq(aliceSize, 0, "valid fill of order 1 was undone");
    }

    /// @notice Same payload, same orders: executing them one by one succeeds, so the data is valid and in policy.
    ///         Only the batch reuse branch rejects it.
    function test_B1_samePayloadSucceedsOrderByOrder() public {
        (, uint64 aliceId, uint64 bobId, bytes[] memory historical) = _twoOrdersAndPayload(true);
        OrderV3Types.ExecutionResult memory first = router.executeOrder{value: 2 * FEE}(aliceId, historical);
        OrderV3Types.ExecutionResult memory second = router.executeOrder{value: 2 * FEE}(bobId, historical);
        assertEq(uint8(first.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(uint8(second.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(baseMockPyth.parseUniqueCallCount(), 2, "two full historical parses, two Pyth fees");
    }

    /// @notice Control: with equal component times the identical batch reuses the basket and executes both orders.
    ///         Unequal component times (normal for six independently published feeds) are the only trigger.
    function test_B1_equalTicksBatchWorks() public {
        (,, uint64 bobId, bytes[] memory historical) = _twoOrdersAndPayload(false);
        OrderV3Types.BatchResult memory result = router.executeOrderBatch{value: 2 * FEE}(bobId, historical);
        assertEq(result.terminalCount, 2);
        assertEq(baseMockPyth.parseUniqueCallCount(), 1, "second order reused the basket");
    }

    /// @notice The repo's own reuse test only passes because it first stores a NEWER update for every feed
    ///         (OracleSynchronization.t.sol:445-446). Without that extra, undocumented pre-push, the batch reverts.
    function test_B1_onlyAnExtraNewerPushMakesTheBatchWork() public {
        (uint64 t,, uint64 bobId, bytes[] memory historical) = _twoOrdersAndPayload(true);
        baseMockPyth.updatePriceFeeds{value: FEE}(_payload(t + 6, t + 5));
        OrderV3Types.BatchResult memory result = router.executeOrderBatch{value: 2 * FEE}(bobId, historical);
        assertEq(result.terminalCount, 2);
    }

}
```
