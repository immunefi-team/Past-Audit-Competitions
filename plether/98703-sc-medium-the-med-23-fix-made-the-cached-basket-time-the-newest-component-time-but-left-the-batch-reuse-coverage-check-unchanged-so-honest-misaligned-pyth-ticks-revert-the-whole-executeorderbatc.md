# \#98703 [SC-Medium] the MED-23 fix made the cached basket time the newest component time but left the batch-reuse coverage check unchanged, so honest misaligned Pyth ticks revert the whole `executeOrderBatch` instead of falling back to a fresh resolution

**Submitted on Nov 9th 2026 at 17:17:49 UTC by @farismaulana for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98703
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

when a keeper executes orders in a batch, `PletherOracle` caches the historical basket parsed for the first order and reuses it for later orders committed in the same window. before reusing, it requires every basket feed's stored Pyth publish time to be at least `cache.publishTime`. the fix for the prior audit's MED-23 (Plether-Fi/plether-core PR #120, fix commits `2fca7941` and `7de2f742`, squashed here as `81919218`) made `cache.publishTime` the newest of the six component times, while each feed's stored time is only its own component time. the fix did not update the reuse check, so this is a new bug introduced by that fix. whenever the six FX components carry different publish times, which the oracle explicitly accepts up to `maxComponentPublishTimeDivergence` (5s), the reuse check reverts `PletherOracle__StoredFeedBehind`, the Router re-raises it, and the entire `executeOrderBatch` reverts, rolling back orders that already executed in that call.

the documented behaviour is that reuse requires cache coverage and that "Otherwise the Router requests a new historical resolution". the Router never makes that check, so batch execution fails on honest Pyth data instead of delivering that fallback.

## Vulnerability Details

a fresh historical parse proves storage coverage per feed, each against its own component time:

```solidity
            pyth.updatePriceFeeds{value: pythFee}(pythUpdateData);
            for (uint256 i; i < parsedFeeds.length; ++i) {
@>              _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);
            }
```

the basket built from those feeds then records the newest component as `publishTime` and the oldest as `oldestPublishTime`:

```solidity
@>      basket.publishTime = uint64(maxPublishTime);
        basket.oldestPublishTime = uint64(minPublishTime);
```

but the reuse branch proves every feed against `cache.publishTime`, the newest component:

```solidity
        if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
            // Cache contents are caller-supplied; independently prove storage coverage, not price authenticity.
            for (uint256 i; i < pythFeedIds.length; ++i) {
@>              _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
            }
```

```solidity
        uint256 storedPublishTime = pyth.getPriceUnsafe(feedId).publishTime;
@>      if (storedPublishTime < requiredPublishTime) {
            revert PletherOracle__StoredFeedBehind(feedId, storedPublishTime, requiredPublishTime);
        }
```

before commit `81919218`, the same basket code set `basket.publishTime = uint64(minPublishTime)` (`PletherOracle.sol:691` at `81919218^`) and the reuse loop was the same `cache.publishTime` check, so a cache that passed its own per-feed proof always passed reuse. the commit switched `publishTime` to the max and kept the reuse check unchanged.

the switch to the max is the fix for MED-23 in the prior Cecuro audit. MED-23's remediation reads: "Use a basket freshness marker that advances whenever any component used in pricing advances, preferably maxPublishTime or an explicit basket version/hash". the disposition records the change: "The live basket's progress marker uses the newest component timestamp, so a newer component can advance it even when another component has not updated." PR #120 lists MED-23 as addressed by `2fca7941` and `7de2f742`. that fix changed what `publishTime` means for every consumer, and the batch-reuse coverage check was not updated, so the reuse path now rejects baskets the fix itself is designed to produce. MED-23's root cause, protection gating against a stale progress marker, is closed. this report is about the new revert the fix introduced in batch execution. after the parse, Pyth stores each feed at its own message time (Pyth only overwrites with a newer message), so the lagging feed sits below the max and reuse reverts.

the Router decides whether to reuse with `_canReuseHistoricalBatchBasket`, which checks commit ordering and the settlement window but has no coverage term:

```solidity
        if (
            commitTime < cache.minReusableCommitTime || commitTime >= cache.oldestPublishTime
                || commitTime >= cache.publishTime
        ) {
            return false;
        }
        if (cache.publishTime > block.timestamp) {
            return false;
        }
@>      return uint256(cache.publishTime) <= uint256(commitTime) + oracle.orderSettlementWindow();
```

so it picks reuse, forwards a zero fee, and when the oracle reverts it re-raises everything except a 68-byte `HistoricalParserFailure`:

```solidity
        } catch (bytes memory reason) {
            if (
                reason.length != 68
                    || _selector(reason) != IPletherOracle.PletherOracle__HistoricalParserFailure.selector
            ) {
                assembly ("memory-safe") {
@>                  revert(add(reason, 32), mload(reason))
                }
            }
```

ORACLE_SYNCHRONIZATION.md describes the intended behaviour: "Reuse for a later order requires all components to follow that order's commit, fit its historical window and satisfy cache coverage. Otherwise the Router requests a new historical resolution." the Router enforces the first two conditions and never evaluates the third, so a cache that fails coverage is reused anyway and the batch reverts instead of re-resolving.

concrete case from the PoC:

1. two traders commit opens in the same second `c`. all six feeds are stored at `c`.
2. a keeper calls `executeOrderBatch(2, payload)` with one historical update per feed, component times `[c+1, c+2, c+3, c+3, c+3, c+3]` (a 2s spread, within the 5s divergence cap).
3. order 1 parses fresh, passes its per-feed coverage check, stores feed 0 at `c+1`, and executes (`OrderExecuted(1)` is emitted). the cache now has `publishTime = c+3`.
4. order 2 takes the reuse path. feed 0 is stored at `c+1 < c+3`, so the oracle reverts `PletherOracle__StoredFeedBehind(feed0, c+1, c+3)`.
5. the Router re-raises the revert. the whole call reverts and order 1 is rolled back.

the same payload executes order 1 alone, and executes both orders one at a time through `executeOrder`. with all components at the same time, the batch executes both orders through reuse.

## Impact Details

1. the Router does not deliver the documented batch fallback. ORACLE_SYNCHRONIZATION.md promises that when a cache does not "satisfy cache coverage", "the Router requests a new historical resolution". instead, the reuse path is taken and the whole `executeOrderBatch` reverts.
2. one failing reuse aborts the whole call. in the PoC, order 1 parses, executes and emits `OrderExecuted(1)`, then order 2's reuse reverts and the Router re-raises it, so order 1's execution is rolled back with the batch and both orders stay pending.
3. the condition is the one the MED-23 fix was built for. the oracle accepts components up to 5s apart, and SECURITY.md states that "Requiring simultaneous feed timestamps would impede feeds that publish at different times". any accepted payload whose component times differ triggers the revert while the lagging feed's stored time is still its own component time.
4. batch execution of those orders is unavailable, and the zero-fee reuse the batch path exists for is lost. keepers must fall back to one-order-at-a-time `executeOrder` calls, paying a fresh Pyth update fee per order. the orders keep their first-after-commit prices, so no funds are lost.

this maps to the in-scope Low impact "Contract fails to deliver promised returns, but doesn't lose value": the batch path does not deliver the promised re-resolution fallback, while no value is lost.

## References

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L262-L266

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L623-L626

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L645-L652

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L874-L875

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L996-L1005

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1019-L1027

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1074-L1093

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md#L60-L63

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L697-L698

https://github.com/Plether-Fi/plether-core/pull/120

https://github.com/Plether-Fi/plether-core/commit/2fca7941

https://github.com/Plether-Fi/plether-core/commit/7de2f742b56f58393f4284159d88e10d2abb752e


## Proof of Concept

## Proof of Concept

the test extends the suite's `OracleSynchronizationTestBase`, which deploys the production `PletherOracle`, `OrderRouter`, execution sidecar, `CfdEngine` and `HousePool` over the six basket feeds with the default `orderSettlementWindow = 15` and `maxComponentPublishTimeDivergence = 5`. Pyth is the suite's `MockPyth`, which stores a message only when it is newer than storage, like Pyth's `updatePriceFeeds`.

create `packages/perps/test/perps/poc/BatchReuseCoverage.t.sol` with this content:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {OracleSynchronizationTestBase} from "../shared/OracleSynchronizationTestBase.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice batch historical-basket reuse checks every stored feed against the basket's NEWEST component
///         time (cache.publishTime = maxPublishTime, PletherOracle.sol:874), while creation only proved each feed
///         against its own component time (PletherOracle.sol:623-626). With honest, non-identical component times
///         (within the 5s divergence cap) the router picks reuse (OrderRouterExecutionSidecar.sol:997) and the
///         oracle reverts StoredFeedBehind (PletherOracle.sol:262-266), re-raised by the router (:1019-1025),
///         so the entire executeOrderBatch reverts and order 1's execution is rolled back.
/// @dev Uses the production PletherOracle/OrderRouter/sidecars/CfdEngine/HousePool deployed by BasePerpTest and
///      OracleSynchronizationTestBase. Only Pyth is mocked (MockPyth): updatePriceFeeds stores a message only when it
///      is strictly newer than storage, like Pyth's updateLatestPriceIfNecessary.
contract BatchReuseCoverageTest is OracleSynchronizationTestBase {

    /// One Hermes-style historical payload: one message per feed, every prevPublishTime = commit (unique first tick
    /// after commit), component publish times = commit + offsets[i].
    function _misalignedPayload(
        uint64 commit,
        uint64[6] memory offsets
    ) internal pure returns (bytes[] memory data) {
        data = _payload(commit + 1, commit);
        MockPyth.FeedUpdate[] memory updates = abi.decode(data[0], (MockPyth.FeedUpdate[]));
        for (uint256 i; i < updates.length; ++i) {
            updates[i].price.publishTime = commit + offsets[i];
        }
        data[0] = abi.encode(updates);
    }

    function _skewedOffsets() internal pure returns (uint64[6] memory o) {
        // EUR-like feed ticks 1s after commit, the next at +2s, the rest at +3s: spread 2s <= 5s divergence cap.
        o = [uint64(1), 2, 3, 3, 3, 3];
    }

    function _alignedOffsets() internal pure returns (uint64[6] memory o) {
        o = [uint64(2), 2, 2, 2, 2, 2];
    }

    /// Two ordinary traders commit in the same second; storage is at `commit` (not pre-advanced by anyone).
    function _commitTwoSameSecond() internal returns (uint64 commit) {
        commit = uint64(block.timestamp);
        assertEq(_commit(ALICE, 0), 1);
        assertEq(_commit(BOB, 0), 2);
        bytes32[] memory ids = _synchronizationFeedIds();
        for (uint256 i; i < ids.length; ++i) {
            assertEq(baseMockPyth.getPriceUnsafe(ids[i]).publishTime, commit, "storage not pre-advanced");
        }
        _advance(commit + 4);
    }

    function _assertNothingExecuted() internal view {
        assertEq(router.nextExecuteId(), 1, "FIFO head unchanged");
        (uint256 aliceSize,,,,,,) = engine.positions(ALICE);
        (uint256 bobSize,,,,,,) = engine.positions(BOB);
        assertEq(aliceSize, 0, "order 1 execution rolled back");
        assertEq(bobSize, 0);
        assertEq(uint256(router.lifecycleBook().lifecycleStatus(1)), uint256(OrderV3Types.LifecycleStatus.Pending));
        assertEq(uint256(router.lifecycleBook().lifecycleStatus(2)), uint256(OrderV3Types.LifecycleStatus.Pending));
        assertEq(baseMockPyth.parseUniqueCallCount(), 0, "parse rolled back");
    }

    /// EXPLOIT: honest misaligned components -> whole batch reverts StoredFeedBehind, order 1 rolled back.
    function test_BatchRevertsOnReuseWithMisalignedComponents() public {
        uint64 commit = _commitTwoSameSecond();
        bytes[] memory data = _misalignedPayload(commit, _skewedOffsets());
        bytes32 laggingFeed = _synchronizationFeedIds()[0];

        bytes memory expected = abi.encodeWithSelector(
            IPletherOracle.PletherOracle__StoredFeedBehind.selector, laggingFeed, uint256(commit + 1), uint256(commit + 3)
        );
        (bool success, bytes memory revertData) = address(router).call{value: 4 * FEE}(
            abi.encodeWithSignature("executeOrderBatch(uint64,bytes[])", uint64(2), data)
        );
        console2.log("batch success:", success);
        console2.log("revert data:");
        console2.logBytes(revertData);
        assertFalse(success, "executeOrderBatch must revert");
        assertEq(revertData, expected, "revert is PletherOracle__StoredFeedBehind(feed0, c+1, c+3)");
        _assertNothingExecuted();

        // The same order 1 alone (maxOrderId = 1) executes with the same payload: the revert comes from order 2's reuse.
        OrderV3Types.BatchResult memory r = router.executeOrderBatch{value: 4 * FEE}(1, data);
        assertEq(r.terminalCount, 1, "order 1 alone executes with this payload");
    }

    /// NEGATIVE CONTROL: identical setup, all six components at the same time -> batch executes both via reuse.
    function test_Control_AlignedComponentsBatchSucceeds() public {
        uint64 commit = _commitTwoSameSecond();
        bytes[] memory data = _misalignedPayload(commit, _alignedOffsets());
        OrderV3Types.BatchResult memory r = router.executeOrderBatch{value: 4 * FEE}(2, data);
        console2.log("aligned terminalCount:", r.terminalCount);
        assertEq(r.terminalCount, 2, "both orders execute");
        assertEq(r.nextOrderId, 0);
        assertEq(baseMockPyth.parseUniqueCallCount(), 1, "second order reused the cached basket");
        assertEq(uint256(router.lifecycleBook().lifecycleStatus(1)), uint256(OrderV3Types.LifecycleStatus.Executed));
        assertEq(uint256(router.lifecycleBook().lifecycleStatus(2)), uint256(OrderV3Types.LifecycleStatus.Executed));
    }

    /// SECOND CONTROL: same misaligned payload, executed one order at a time via executeOrder -> both succeed.
    function test_Control_SingleExecuteOrderSucceeds() public {
        uint64 commit = _commitTwoSameSecond();
        bytes[] memory data = _misalignedPayload(commit, _skewedOffsets());
        OrderV3Types.ExecutionResult memory r1 = router.executeOrder{value: 2 * FEE}(1, data);
        OrderV3Types.ExecutionResult memory r2 = router.executeOrder{value: 2 * FEE}(2, data);
        assertEq(uint256(r1.status), uint256(OrderV3Types.LifecycleStatus.Executed), "order 1 executes alone");
        assertEq(uint256(r2.status), uint256(OrderV3Types.LifecycleStatus.Executed), "order 2 executes alone");
        assertEq(baseMockPyth.parseUniqueCallCount(), 2, "each single execution re-parses");
    }

    /// REGRESSION ANALYSIS (oracle level, src unmodified): after an honest parse, every stored feed covers
    /// cache.oldestPublishTime but not cache.publishTime, so a reuse check against the oldest (pre-81919218
    /// semantics, where basket.publishTime was the min) would pass, while the HEAD check against the max reverts.
    function test_CacheCoversOldestButNotNewest() public {
        uint64 commit = uint64(block.timestamp);
        _advance(commit + 4);
        bytes[] memory data = _misalignedPayload(commit, _skewedOffsets());
        IPletherOracle.BatchOrderPriceCache memory empty;
        (bool ok,, IPletherOracle.BatchOrderPriceCache memory cache) =
            pletherOracle.updateBatchOrderExecutionPrice{value: 2 * FEE}(address(this), data, _request(commit, false), empty);
        assertTrue(ok, "honest parse passes the per-feed coverage check");
        assertEq(cache.publishTime, commit + 3, "cache.publishTime = newest component");
        assertEq(cache.oldestPublishTime, commit + 1, "cache.oldestPublishTime = oldest component");

        bytes32[] memory ids = _synchronizationFeedIds();
        for (uint256 i; i < ids.length; ++i) {
            uint256 stored = baseMockPyth.getPriceUnsafe(ids[i]).publishTime;
            assertEq(stored, commit + _skewedOffsets()[i], "stored time == that feed's message time");
            assertGe(stored, cache.oldestPublishTime, "coverage vs oldestPublishTime holds");
        }
        assertLt(baseMockPyth.getPriceUnsafe(ids[0]).publishTime, cache.publishTime, "feed0 behind newest");

        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector, ids[0], uint256(commit + 1), uint256(commit + 3)
            )
        );
        pletherOracle.updateBatchOrderExecutionPrice(address(this), new bytes[](0), _request(commit, false), cache);
    }

}
```

run it from `packages/perps`:

```bash
forge test --match-path test/perps/poc/BatchReuseCoverage.t.sol -vv
```

output:

```bash
[PASS] test_BatchRevertsOnReuseWithMisalignedComponents() (gas: 5526924)
Logs:
  batch success: false
  revert data:
  0xf2aa6d4900000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000065e965f10000000000000000000000000000000000000000000000000000000065e965f3

[PASS] test_CacheCoversOldestButNotNewest() (gas: 441899)
[PASS] test_Control_AlignedComponentsBatchSucceeds() (gas: 4251390)
Logs:
  aligned terminalCount: 2

[PASS] test_Control_SingleExecuteOrderSucceeds() (gas: 4312052)
Suite result: ok. 4 passed; 0 failed; 0 skipped
```

the batch over two orders reverts with `0xf2aa6d49`, which is `PletherOracle__StoredFeedBehind(feed 0x..01, stored 0x65e965f1 = c+1, required 0x65e965f3 = c+3)`. after the call, `nextExecuteId` is still 1, both positions are empty and both orders are `Pending`, so order 1's execution was rolled back with the batch. the same payload executes order 1 alone, executes both orders through single `executeOrder` calls, and with aligned component times the batch executes both orders through reuse (`terminalCount` 2). `test_CacheCoversOldestButNotNewest` shows the cache records `publishTime = c+3` and `oldestPublishTime = c+1`, every stored feed covers `oldestPublishTime`, and only the check against the newest component fails.

