# \#97632 [SC-Medium] Stale storage-coverage check in PletherOracle batch cache reuse makes executeOrderBatch revert (StoredFeedBehind) on any natural Pyth component skew, breaking keeper batch execution

**Submitted on Nov 2nd 2026 at 14:58:37 UTC by @Magnum for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97632
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

`PletherOracle.updateBatchOrderExecutionPrice` lets a keeper reuse the historical basket resolved for the first order of a batch for every later reuse-eligible order. PR #120 (`81919218`, "synchronize historical Pyth and enforce signed live basket") changed `BasketPrice.publishTime` from the *oldest* component publish time to the *newest* one and added `oldestPublishTime`, but the storage-coverage proof in the reuse branch still requires every Pyth feed in storage to be at least `cache.publishTime` (the newest component). Because the first order stores each feed at its *own* first-post-commit tick, any component that is even one second older than the newest one fails the check and the oracle reverts `PletherOracle__StoredFeedBehind`. `OrderRouterExecutionSidecar._prepareBatchOracle` only converts `HistoricalParserFailure` into a `Pending` result and re-raises everything else, so the entire `OrderRouter.executeOrderBatch` transaction reverts. With six independent Pyth FX feeds, all components publishing in the exact same second is the exception, so in production the batch execution path reverts for essentially every batch that contains a reuse-eligible second order. Keepers lose the gas of every attempt and are forced to fall back to one `executeOrder` per order, each paying the full Pyth update fee.

## Vulnerability Details

**Reuse branch of the batch oracle call** (`packages/perps/src/PletherOracle.sol:262-267`):

```solidity
if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
    // Cache contents are caller-supplied; independently prove storage coverage, not price authenticity.
    for (uint256 i; i < pythFeedIds.length; ++i) {
        _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);   // <-- NEWEST component time
    }
    basket = BasketPrice({
        price: cache.price,
        confidence: cache.confidence,
        publishTime: cache.publishTime,
        oldestPublishTime: cache.oldestPublishTime,
        pythFee: 0
    });
    reusedBasket = true;
}
```

**Coverage proof** (`PletherOracle.sol:645-653`):

```solidity
function _requireStoredFeedCoverage(bytes32 feedId, uint256 requiredPublishTime) internal view {
    uint256 storedPublishTime = pyth.getPriceUnsafe(feedId).publishTime;
    if (storedPublishTime < requiredPublishTime) {
        revert PletherOracle__StoredFeedBehind(feedId, storedPublishTime, requiredPublishTime);
    }
}
```

**How storage gets populated.** The first (non-reusable) order in a batch goes through `_resolveHistoricalOrderBasket` (`PletherOracle.sol:613-628`): it calls `pyth.parsePriceFeedUpdatesUnique(data, feeds, commitTime + 1, ...)`, which yields the *unique* first post-commit tick `p_i` of each feed, then calls `pyth.updatePriceFeeds(payload)` with that same payload. Pyth therefore stores feed `i` at `publishTime = p_i`. `_computeBasketPriceFromFeeds` (`PletherOracle.sol:874-875`) then sets `basket.publishTime = max_i p_i` and `basket.oldestPublishTime = min_i p_i`, and the cache is extended with exactly these values (`PletherOracle.sol:287-296`).

**Reuse eligibility** (`_canReuseHistoricalBatchBasket`, `PletherOracle.sol:974-991`) was updated in PR #120 to require `minReusableCommitTime <= commitTime < cache.oldestPublishTime`. The coverage proof was not updated and still uses `cache.publishTime`.

**Result.** For the second order, every feed `i` with `p_i < max_j p_j` has `stored(i) = p_i < cache.publishTime` and reverts `StoredFeedBehind(feed_i, p_i, max_j p_j)`. The only ways the check passes are (a) zero skew: all six components published in the same second, or (b) some third party already pushed newer signed data for the lagging feeds into Pyth storage before the batch.

**Why the whole batch dies.** `OrderRouterExecutionSidecar._prepareBatchOracle` (`packages/perps/src/OrderRouterExecutionSidecar.sol:1019-1030`):

```solidity
} catch (bytes memory reason) {
    if (reason.length != 68
        || _selector(reason) != IPletherOracle.PletherOracle__HistoricalParserFailure.selector) {
        assembly ("memory-safe") { revert(add(reason, 32), mload(reason)) }   // re-raise
    }
    return (OrderV3Types.PendingReason.OracleFailure, result, state.oracleCache);
}
```

`StoredFeedBehind` is not `HistoricalParserFailure`, so the revert propagates and `executeOrderBatch` reverts atomically, including the already-prepared first order.

**Why the existing test suite did not catch it.** `test_BatchReusesBasketWhenEveryComponentFollowsBothCommits` (`packages/perps/test/perps/spec/oracle/OracleSynchronization.t.sol:434-452`) pushes a newer full payload into mock Pyth storage *before* calling the batch (`baseMockPyth.updatePriceFeeds{value: FEE}(newer)` at line 447), which satisfies the coverage check for reasons unrelated to the order's own payload and masks the regression.

**Not affected.** The single-order path `updateOrderExecutionPrice` checks each parsed feed against its own `parsedFeeds[i].price.publishTime` right after storing the same payload (`PletherOracle.sol:624-626`), so it is always satisfiable. The mock Pyth used in the PoC stores each feed at its own publish time (only if newer), which matches real Pyth semantics, so the behavior transfers to mainnet.

## Impact Details

- `OrderRouter.executeOrderBatch`, the primary keeper execution primitive, reverts for every batch in which a second order is reuse-eligible and the six basket components did not all tick in the same second. With six independent Pyth FX feeds this is the normal state of the oracle, so batch execution is effectively unusable in production unless someone first pays for an extra full `updatePriceFeeds` with a later basket.
- Every failed attempt wastes the keeper's gas for parsing and storing the first order (two Pyth update fees' worth of work) and then reverting. Keepers must fall back to one `executeOrder` transaction per order, each paying its own Pyth update fee instead of reusing a basket for free, which raises keeper costs and processes the global FIFO one order per transaction.
- Degraded liveness of the global FIFO under load and of risk-reducing closes queued behind it. Protocol documentation positions batch execution as the keeper's throughput path; that path is broken.
- No attacker action is required: Pyth signs the ticks, so skew cannot be manufactured, but it does not need to be, it occurs naturally. Funds are not lost and not permanently frozen: `executeOrder` still works and `clearInvalidOrders` still expires orders after the window, so the selected impact is **Griefing (damage to the protocol and its keepers without profit motive)**.
- Any direct integrator of `PletherOracle.updateBatchOrderExecutionPrice` is affected the same way (oracle-level reproduction in the PoC).
- Not covered by any listed known issue; the regression was introduced by PR #120 on 2026-10-21, after the Cecuro audit commit `6d6622f2`.

**Recommended fix.** Prove coverage against the time storage is guaranteed to have reached after the proving order's own synchronization, i.e. the oldest component time, which is also the value reuse eligibility is keyed on:

```solidity
// PletherOracle.sol:265
-   _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
+   _requireStoredFeedCoverage(pythFeedIds[i], cache.oldestPublishTime);
```

This is sound because the first order already guaranteed `stored(i) >= p_i >= oldestPublishTime` for every feed, and reuse is only allowed for `commitTime < oldestPublishTime`, so every component remains a strictly post-commit tick. If per-feed exactness is desired, carry `uint64[] componentPublishTimes` in `BatchOrderPriceCache`. Also change `test_BatchReusesBasketWhenEveryComponentFollowsBothCommits` to stop pre-advancing mock Pyth storage so the reuse coverage path is actually exercised.

## References

- `packages/perps/src/PletherOracle.sol` L262-267 (reuse branch), L287-296 (cache extension), L613-628 (`_resolveHistoricalOrderBasket`), L645-653 (`_requireStoredFeedCoverage`), L874-875 (`publishTime = max`, `oldestPublishTime = min`), L974-991 (`_canReuseHistoricalBatchBasket`)
- `packages/perps/src/OrderRouterExecutionSidecar.sol` L1019-1030 (`_prepareBatchOracle` catch)
- `packages/perps/test/perps/spec/oracle/OracleSynchronization.t.sol` L434-452 (masking test)
- Regression origin: https://github.com/Plether-Fi/plether-core/commit/81919218 (PR #120, "synchronize historical Pyth and enforce signed live basket")
- `packages/perps/ORACLE_SYNCHRONIZATION.md` (batch reuse policy)

## Proof of Concept

## Proof of Concept

Foundry test against the competition commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. It extends the repository's own `OracleSynchronizationTestBase` (real `PletherOracle`, `OrderRouter`, `CfdEngine`, `MockPyth` from `packages/shared/test-support`). Mock Pyth storage is never pre-advanced: the only writes come from the orders' own payloads, exactly like a fresh Pyth contract nobody else has pushed to.

### Steps

1. Save the file below as `packages/perps/test/perps/PoC_BatchReuseRevert.t.sol`.
2. Run:

```bash
FOUNDRY_PROFILE=quick forge test --root packages/perps --match-path "test/perps/PoC_BatchReuseRevert.t.sol" -vvv
```

3. Observe:
   - `test_PoC_BatchRevertsWithNaturalComponentSkew_SameSecondCommits`: two traders commit at second `T`; the six feeds' first post-commit ticks are `T+1, T+2, T+1, T+3, T+2, T+1` (2 s skew, within the 5 s divergence limit). `executeOrderBatch(2, payload)` reverts `PletherOracle__StoredFeedBehind(feed1, T+1, T+3)`; `nextExecuteId` stays 1, Pyth parse/update counters are rolled back, both orders remain `Pending`, engine mark untouched.
   - `test_PoC_SingleExecutionSucceedsWithSameSkew`: the same two orders and the same payload execute successfully via two `executeOrder` calls (both `Executed`, mark installed at `T+3`, two parses paid).
   - `test_PoC_BatchRevertsForLaterCommitReusingCache`: second commit at `T+1`, ticks `T+2..T+4` → same revert (any commit in `[minReusableCommitTime, oldestPublishTime)` takes the reuse branch).
   - `test_PoC_BatchRevertsWithOneSecondSkewOnOneFeed`: a single component lagging by 1 s is sufficient.
   - `test_PoC_Control_BatchSucceedsOnlyWithZeroSkew`: all six components at `T+2` → batch succeeds with one parse (control case proving the reuse branch is reached and the fault is the coverage check).
   - `test_PoC_OracleLevel_ReuseRevertsOnSkewedStorage`: oracle-only reproduction; returned cache has `publishTime = T+3`, `oldestPublishTime = T+1`, storage feed1 = `T+1`; second call with that cache reverts.
   - `test_PoC_ExpiryCleanupStillWorksAfterBatchRevert`: `clearInvalidOrders` still clears both orders after `maxExecutionWindowSeconds` (bounds the impact to liveness/griefing).

### Result

```
Ran 7 tests for test/perps/PoC_BatchReuseRevert.t.sol:PoC_BatchReuseRevertTest
[PASS] test_PoC_BatchRevertsForLaterCommitReusingCache() (gas: 4356789)
[PASS] test_PoC_BatchRevertsWithNaturalComponentSkew_SameSecondCommits() (gas: 4387264)
[PASS] test_PoC_BatchRevertsWithOneSecondSkewOnOneFeed() (gas: 4353857)
[PASS] test_PoC_Control_BatchSucceedsOnlyWithZeroSkew() (gas: 4223836)
[PASS] test_PoC_ExpiryCleanupStillWorksAfterBatchRevert() (gas: 3940038)
[PASS] test_PoC_OracleLevel_ReuseRevertsOnSkewedStorage() (gas: 413574)
[PASS] test_PoC_SingleExecutionSucceedsWithSameSkew() (gas: 4304149)
Suite result: ok. 7 passed; 0 failed; 0 skipped
```

Key trace excerpt (`-vvv`, first test):

```
PletherOracle::updateBatchOrderExecutionPrice(..., BatchOrderPriceCache({ hasHistoricalBasket: true,
    minReusableCommitTime: 1709794800, price: 100000000, confidence: 100000,
    publishTime: 1709794803, oldestPublishTime: 1709794801 }))
  MockPyth::getPriceUnsafe(0x0000…0001) → Price({ ..., publishTime: 1709794801 })
  ← [Revert] PletherOracle__StoredFeedBehind(0x0000…0001, 1709794801, 1709794803)
← [Revert] OrderRouterExecutionSidecar::executeOrderBatch (re-raised)
← [Revert] OrderRouter::executeOrderBatch
nextExecuteId() → 1 ; parseUniqueCallCount() → 0 ; updatePriceFeedsCallCount() → 0 ; lifecycleStatus(1) → Pending
```

### Test file

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {OracleSynchronizationTestBase} from "./shared/OracleSynchronizationTestBase.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @title PoC: batch cache reuse requires every stored feed to reach the NEWEST component time
contract PoC_BatchReuseRevertTest is OracleSynchronizationTestBase {

    /// @dev Six-component payload whose component i is published at `base + offsets[i]`, each with the same
    ///      signed `prevPublishTime` (strictly before the earliest post-commit tick so Pyth's unique parse accepts it).
    function _skewedPayload(
        uint64 base,
        uint64 prev,
        uint64[6] memory offsets
    ) internal pure returns (bytes[] memory data) {
        bytes32[] memory ids = _synchronizationFeedIds();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            updates[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(100_000_000, 100_000, -8, base + offsets[i], prev)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    /// @dev Realistic cadence skew for a six-feed FX basket, all within the 5 s divergence limit:
    ///      feed1 T+1, feed2 T+2, feed3 T+1, feed4 T+3, feed5 T+2, feed6 T+1. Newest = T+3, oldest = T+1.
    function _realisticSkew() internal pure returns (uint64[6] memory offsets) {
        offsets = [uint64(1), 2, 1, 3, 2, 1];
    }

    /// @notice Two traders commit in the same second. One keeper batch with a correctly signed first-post-commit
    ///         basket reverts as a whole with `StoredFeedBehind`, even though the second order is a textbook reuse.
    function test_PoC_BatchRevertsWithNaturalComponentSkew_SameSecondCommits() public {
        uint64 T = uint64(block.timestamp);
        _commit(ALICE, 0); // order id 1, commitTime T
        _commit(BOB, 0); // order id 2, commitTime T
        _advance(T + 6);

        bytes[] memory historical = _skewedPayload(T, T - 1, _realisticSkew());
        // Storage is NOT pre-advanced: stored feeds are still at T (setUp).
        _assertStoredAtMost(T);

        uint256 parsesBefore = baseMockPyth.parseUniqueCallCount();
        uint256 updatesBefore = baseMockPyth.updatePriceFeedsCallCount();
        uint64 markBefore = engine.lastMarkTime();

        // Order 1 parses + stores (feed1 at T+1 ... feed4 at T+3), cache.publishTime = T+3.
        // Order 2 (commit T < oldest T+1) is reusable -> coverage check demands stored(feed1) >= T+3 -> T+1 -> revert.
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector, bytes32(uint256(1)), uint256(T + 1), uint256(T + 3)
            )
        );
        router.executeOrderBatch{value: 4 * FEE}(2, historical);

        // Whole batch rolled back: nothing executed, Pyth calls undone, both orders still pending.
        assertEq(router.nextExecuteId(), 1, "batch made no progress");
        assertEq(baseMockPyth.parseUniqueCallCount(), parsesBefore, "parse rolled back");
        assertEq(baseMockPyth.updatePriceFeedsCallCount(), updatesBefore, "storage update rolled back");
        assertEq(uint256(router.lifecycleBook().lifecycleStatus(1)), uint256(OrderV3Types.LifecycleStatus.Pending));
        assertEq(uint256(router.lifecycleBook().lifecycleStatus(2)), uint256(OrderV3Types.LifecycleStatus.Pending));
        assertEq(engine.lastMarkTime(), markBefore, "engine mark untouched");
    }

    /// @notice Exactly the same orders and the same signed payload execute fine one by one through `executeOrder`.
    function test_PoC_SingleExecutionSucceedsWithSameSkew() public {
        uint64 T = uint64(block.timestamp);
        _commit(ALICE, 0);
        _commit(BOB, 0);
        _advance(T + 6);

        bytes[] memory historical = _skewedPayload(T, T - 1, _realisticSkew());

        OrderV3Types.ExecutionResult memory r1 = router.executeOrder{value: 2 * FEE}(1, historical);
        assertEq(uint256(r1.status), uint256(OrderV3Types.LifecycleStatus.Executed), "order 1 executed alone");
        OrderV3Types.ExecutionResult memory r2 = router.executeOrder{value: 2 * FEE}(2, historical);
        assertEq(uint256(r2.status), uint256(OrderV3Types.LifecycleStatus.Executed), "order 2 executed alone");

        assertEq(router.nextExecuteId(), 0, "queue drained");
        assertEq(engine.lastMarkTime(), T + 3, "mark installed at the newest component tick");
        assertEq(baseMockPyth.parseUniqueCallCount(), 2, "each single execution pays for its own parse");
        (uint256 sizeAlice,,,,,,) = engine.positions(ALICE);
        (uint256 sizeBob,,,,,,) = engine.positions(BOB);
        assertGt(sizeAlice, 0);
        assertGt(sizeBob, 0);
    }

    /// @notice A later commit that is still covered by the cache (commitTime < oldestPublishTime) hits the same revert.
    function test_PoC_BatchRevertsForLaterCommitReusingCache() public {
        uint64 T = uint64(block.timestamp);
        _commit(ALICE, 0); // commit T
        _advance(T + 1);
        _commit(BOB, 0); // commit T+1
        _advance(T + 7);

        // First post-commit ticks: feed1 T+2, feed2 T+3, feed3 T+2, feed4 T+4, feed5 T+3, feed6 T+2.
        // oldest = T+2 > Bob's commit T+1, so Bob's order is a valid reuse of Alice's basket.
        uint64[6] memory offsets = [uint64(2), 3, 2, 4, 3, 2];
        bytes[] memory historical = _skewedPayload(T, T - 1, offsets);

        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector, bytes32(uint256(1)), uint256(T + 2), uint256(T + 4)
            )
        );
        router.executeOrderBatch{value: 4 * FEE}(2, historical);
        assertEq(router.nextExecuteId(), 1);
    }

    /// @notice Even a 1-second skew on a single component is enough to break the batch.
    function test_PoC_BatchRevertsWithOneSecondSkewOnOneFeed() public {
        uint64 T = uint64(block.timestamp);
        _commit(ALICE, 0);
        _commit(BOB, 0);
        _advance(T + 6);

        uint64[6] memory offsets = [uint64(2), 2, 2, 2, 2, 3]; // only one component lags by 1 s
        bytes[] memory historical = _skewedPayload(T, T - 1, offsets);

        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector, bytes32(uint256(1)), uint256(T + 2), uint256(T + 3)
            )
        );
        router.executeOrderBatch{value: 4 * FEE}(2, historical);
    }

    /// @notice Control: with ZERO component skew the same batch succeeds and reuses the basket.
    function test_PoC_Control_BatchSucceedsOnlyWithZeroSkew() public {
        uint64 T = uint64(block.timestamp);
        _commit(ALICE, 0);
        _commit(BOB, 0);
        _advance(T + 6);

        uint64[6] memory offsets = [uint64(2), 2, 2, 2, 2, 2];
        bytes[] memory historical = _skewedPayload(T, T - 1, offsets);

        OrderV3Types.BatchResult memory result = router.executeOrderBatch{value: 4 * FEE}(2, historical);
        assertEq(result.terminalCount, 2);
        assertEq(result.nextOrderId, 0);
        assertEq(baseMockPyth.parseUniqueCallCount(), 1, "second order reused the cache");
    }

    /// @notice Oracle-level reproduction, independent of the Router: the second call with a reusable cache reverts.
    function test_PoC_OracleLevel_ReuseRevertsOnSkewedStorage() public {
        uint64 T = uint64(block.timestamp);
        _advance(T + 6);
        bytes[] memory historical = _skewedPayload(T, T - 1, _realisticSkew());

        IPletherOracle.BatchOrderPriceCache memory cache;
        bool ok;
        (ok,, cache) = pletherOracle.updateBatchOrderExecutionPrice{value: 2 * FEE}(
            address(this), historical, _request(T, false), cache
        );
        assertTrue(ok);
        assertTrue(cache.hasHistoricalBasket);
        assertEq(cache.publishTime, T + 3, "cache.publishTime is the NEWEST component");
        assertEq(cache.oldestPublishTime, T + 1, "cache.oldestPublishTime is the OLDEST component");
        // Storage now holds each feed at its own tick: feed1 = T+1 < cache.publishTime.
        assertEq(baseMockPyth.getPriceUnsafe(bytes32(uint256(1))).publishTime, T + 1);
        assertEq(baseMockPyth.getPriceUnsafe(bytes32(uint256(4))).publishTime, T + 3);

        // Reuse for an identical commit (the canonical reuse case) reverts.
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector, bytes32(uint256(1)), uint256(T + 1), uint256(T + 3)
            )
        );
        pletherOracle.updateBatchOrderExecutionPrice(address(this), new bytes[](0), _request(T, false), cache);
    }

    /// @notice Consequence bound: orders are not permanently stuck; permissionless expiry cleanup still works.
    function test_PoC_ExpiryCleanupStillWorksAfterBatchRevert() public {
        uint64 T = uint64(block.timestamp);
        _commit(ALICE, 0);
        _commit(BOB, 0);
        _advance(T + 6);

        bytes[] memory historical = _skewedPayload(T, T - 1, _realisticSkew());
        vm.expectRevert();
        router.executeOrderBatch{value: 4 * FEE}(2, historical);

        _advance(T + uint64(router.maxExecutionWindowSeconds()) + 1);
        uint64[] memory ids = new uint64[](2);
        ids[0] = 1;
        ids[1] = 2;
        (, uint256 cleared) = router.clearInvalidOrders(ids);
        assertEq(cleared, 2, "expired orders cleared");
        assertEq(uint256(router.lifecycleBook().lifecycleStatus(1)), uint256(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint256(router.lifecycleBook().lifecycleStatus(2)), uint256(OrderV3Types.LifecycleStatus.Failed));
    }

    function _assertStoredAtMost(
        uint256 bound
    ) internal view {
        bytes32[] memory ids = _synchronizationFeedIds();
        for (uint256 i; i < ids.length; ++i) {
            assertLe(baseMockPyth.getPriceUnsafe(ids[i]).publishTime, bound);
        }
    }

}
```
