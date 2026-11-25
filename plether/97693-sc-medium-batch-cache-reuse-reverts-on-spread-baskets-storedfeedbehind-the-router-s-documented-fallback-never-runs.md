# \#97693 [SC-Medium] Batch cache reuse reverts on spread baskets (StoredFeedBehind); the router's documented fallback never runs

**Submitted on Nov 2nd 2026 at 17:56:57 UTC by @jacku705 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97693
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary

Batch order execution — a documented core keeper path — **reverts on the normal case**. When the oracle parses a multi-feed basket whose component ticks are not perfectly synchronised (a "spread" basket: `oldestPublishTime < publishTime`, the ordinary state of six independently-ticking FX feeds), the resulting batch price cache can never be reused: the reuse path demands stored Pyth prices at the basket's *newest* component time for *every* feed, a condition the cache's own synchronisation never establishes. The router predicts reuse successfully, forwards a zero oracle fee, and the oracle's `StoredFeedBehind` error — which the router's error handling does not convert into its documented fallback — propagates and **reverts the entire batch transaction**, discarding all earlier progress in that batch.

## Root cause

1. **Impossible coverage demand (oracle).** `PletherOracle.updateBatchOrderExecutionPrice` reuses a cached basket only after `_requireStoredFeedCoverage(feedId, cache.publishTime)` succeeds for every feed (`src/PletherOracle.sol:265`, `:645`) — i.e. every feed's *stored* Pyth price must be at the basket's newest component time. But parsing a basket synchronises each feed only to **its own** tick time, and the cache stores no per-feed times. A spread basket therefore fails its own reuse check immediately after the parse that created it, and keeps failing until unrelated later Pyth activity happens to advance every lagging feed past the basket's newest time.
2. **Router predicts reuse without the coverage check.** `OrderRouterExecutionSidecar._canReuseHistoricalBatchBasket` (`src/OrderRouterExecutionSidecar.sol:1074`) checks only times (commit window vs cache times). It concludes reuse is possible, forwards `updateFee = 0`, and calls the batch path.
3. **The documented fallback never runs.** The sidecar's try/catch converts only `HistoricalParserFailure` into a pending stop. `PletherOracle__StoredFeedBehind` is a different error: it propagates and reverts the whole batch transaction. `ORACLE_SYNCHRONIZATION.md`'s failure table promises "Otherwise the Router requests a new historical resolution" — that fallback is unreachable for this failure mode.

## Impact (proven by PoC at oracle level)

- Control: a zero-spread basket (both feeds at the same tick) parses once and reuses cleanly — no new parse, zero fee. This is why the existing test suite passes: it only exercises aligned baskets.
- Spread basket (feeds at 994 and 990 — a 4-second spread, inside the 5-second divergence limit): the first parse succeeds and creates the cache; stored feed times are 994 and 990 respectively. A second order whose commit time passes **every** router-side reuse check (commit 985 ≥ minReusable 980, < oldest 990, < publish 994, within the 15-second window) then reverts with `PletherOracle__StoredFeedBehind(FEED_B, 990, 994)`.

Consequences in production: any batch containing a spread basket — the normal case, not an edge case — cannot use the cache-reuse path at all, and a keeper submitting a batch that reaches the reuse branch loses the entire transaction, including orders already processed earlier in the same batch. Workarounds exist (single executions re-parse; sufficiently delayed batches self-heal as Pyth storage advances), which is why this is graded Medium rather than High: it is a liveness/economic DoS on the batch execution path, with no direct fund loss.

**Honest limit of the proof:** the revert is demonstrated end-to-end at the oracle level (the exact call the router makes, with the exact cache the first parse produced). The final hop — that this error reverts the *router's* batch transaction rather than being caught — is established by direct code reading of the sidecar's catch scope and its direct batch-loop call, not by a separate full-router PoC. The report is submitted on that basis, stated openly.

## Proof of concept

File: `packages/perps/test/competition/L1OracleDivergenceAndCache.t.sol` (Appendix). The tests for this report are the two `test_C2_*` tests (the file's `test_C3_*` tests cover a separate delayed-order candidate and are not part of this report). Run from the perps project root:

```
forge test --root packages/perps --match-contract L1OracleDivergenceAndCacheTest --match-test test_C2 --threads 1 -vv
```

Verified results (run three times independently — lane run, coordinator central run, and a final independent re-run on 2 Nov 2026 — identical outcome each time; full-file run 5/5 including the unrelated C3 tests):

```
[PASS] test_C2_Control_AlignedCacheReuseWorks() (gas: 798060)
[PASS] test_C2_SpreadCacheReuse_RevertsStoredFeedBehind() (gas: 802650)
```

## Suggested remediation

Either (a) store per-feed publish times in `BatchOrderPriceCache` and require stored coverage only at each feed's own cached tick time, or (b) treat `StoredFeedBehind` like `HistoricalParserFailure` in the router sidecar so the documented fallback (fresh historical resolution) actually runs. (a) fixes the root cause; (b) is the minimal change that restores the documented behaviour.

## Appendix — full PoC source (`L1OracleDivergenceAndCache.t.sol`)

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Lane L1 (Oracle & Index) — second-pass candidates, distinct from the pre-existing
// test/competition/Lane1.t.sol (whose C1 = above-cap capped-base execution shift, and
// whose K-tests kill production-basket precision, LOW-36, cache forgery, calendar).
//
// C2: Batch cache reuse demands stored-feed coverage at the basket's NEWEST component
//     time for EVERY feed (PletherOracle.updateBatchOrderExecutionPrice ->
//     _requireStoredFeedCoverage(feedId, cache.publishTime)), but the cache's own
//     synchronization only guarantees each feed's OWN tick time, and the cache stores
//     no per-feed times. A spread basket (oldest < newest, the normal case for six
//     independently-ticking FX feeds) therefore fails reuse immediately after the very
//     parse that created it, unless unrelated later Pyth activity advanced every feed
//     past the basket's newest time. The Router predicts reuse without checking
//     coverage (OrderRouterExecutionSidecar._canReuseHistoricalBatchBasket checks only
//     times), forwards fee = 0, and only converts HistoricalParserFailure into a
//     pending stop; StoredFeedBehind propagates and reverts the whole batch
//     transaction, wiping earlier batch progress (ORACLE_SYNCHRONIZATION.md failure
//     table). The documented fallback ("Otherwise the Router requests a new historical
//     resolution") never runs for this case.
//
// C3: A delayed order's execution price is the FIRST Pyth tick after commit (unique
//     parse enforces prevPublishTime < commitTime + 1). If those first ticks span more
//     than maxComponentPublishTimeDivergence (5s), _computeBasketPriceFromFeeds reverts
//     PublishTimeDivergence inside the historical-parse try block, which the oracle
//     wraps as HistoricalParserFailure: single execution reverts outright, batch
//     execution stops with OracleFailure. No later tick can rescue the order, because
//     any later tick's prevPublishTime is >= commitTime + 1 and fails uniqueness
//     (PriceFeedNotFoundWithinRange -> ok = false, order stays pending). The order is
//     therefore unexecutable until its executionDeadline, with margin/bounty locked.
//     This is not attacker-triggerable (tick timing is a market-data property), so it
//     is graded Low; C2 is graded Medium (batch path is a documented core keeper path).

import {
    PletherOracleEngineMock,
    PletherOracleVaultMock
} from "../perps/spec/oracle/PletherOracleMocks.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {Test} from "forge-std/Test.sol";

contract L1OracleDivergenceAndCacheTest is Test {

    bytes32 internal constant FEED_A = bytes32(uint256(1));
    bytes32 internal constant FEED_B = bytes32(uint256(2));

    MockPyth internal pyth;
    PletherOracleEngineMock internal engine;
    PletherOracleVaultMock internal vault;
    PletherOracle internal oracle;

    bytes32[] internal feedIds;
    uint256[] internal weights;
    uint256[] internal basePrices;
    bool[] internal inversions;

    receive() external payable {}

    function setUp() public {
        pyth = new MockPyth();
        pyth.setSynchronizeLegacyUniquePrices(true);
        engine = new PletherOracleEngineMock();
        vault = new PletherOracleVaultMock();
        engine.setOrderRouter(address(this));

        feedIds.push(FEED_A);
        feedIds.push(FEED_B);
        weights.push(0.5e18);
        weights.push(0.5e18);
        basePrices.push(1e8);
        basePrices.push(1e8);
        inversions.push(false);
        inversions.push(false);

        oracle = new PletherOracle(address(engine), address(vault), address(pyth), feedIds, weights, basePrices, inversions);
    }

    function _data() internal pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = hex"01";
    }

    function _request(uint64 commitTime, CfdTypes.Side side, bool isClose, bool revertOnUnavailable)
        internal
        pure
        returns (IPletherOracle.OrderExecutionRequest memory)
    {
        return IPletherOracle.OrderExecutionRequest({
            commitTime: commitTime,
            targetPrice: 0,
            side: side,
            isClose: isClose,
            revertOnHistoricalUnavailable: revertOnUnavailable
        });
    }

    // ---- C2 control: a zero-spread basket reuses fine (this is why existing tests pass) ----
    function test_C2_Control_AlignedCacheReuseWorks() public {
        vm.warp(1000);
        pyth.setUniquePrice(FEED_A, int64(100_000_000), uint64(10_000), int32(-8), 995, 980);
        pyth.setUniquePrice(FEED_B, int64(100_000_000), uint64(10_000), int32(-8), 995, 980);

        IPletherOracle.BatchOrderPriceCache memory empty;
        (bool ok, , IPletherOracle.BatchOrderPriceCache memory cache) = oracle.updateBatchOrderExecutionPrice(
            address(this), _data(), _request(980, CfdTypes.Side.LONG, false, false), empty
        );
        assertTrue(ok, "first batch parse resolves");
        assertEq(cache.publishTime, 995, "cache newest time");
        assertEq(cache.oldestPublishTime, 995, "zero spread control");

        uint256 parses = pyth.parseUniqueCallCount();
        (bool ok2, IPletherOracle.PriceSnapshot memory reused,) = oracle.updateBatchOrderExecutionPrice(
            address(this), new bytes[](0), _request(991, CfdTypes.Side.LONG, false, false), cache
        );
        assertTrue(ok2, "aligned cache reuses");
        assertEq(reused.updateFee, 0, "reuse costs no fee");
        assertEq(pyth.parseUniqueCallCount(), parses, "reuse performs no new parse");
    }

    // ---- C2: spread basket (A@994, B@990) fails reuse straight after its own sync ----
    function test_C2_SpreadCacheReuse_RevertsStoredFeedBehind() public {
        vm.warp(1000);
        // First ticks after commit 980, spread 4s (within the 5s divergence limit).
        pyth.setUniquePrice(FEED_A, int64(100_000_000), uint64(10_000), int32(-8), 994, 980);
        pyth.setUniquePrice(FEED_B, int64(100_000_000), uint64(10_000), int32(-8), 990, 980);

        IPletherOracle.BatchOrderPriceCache memory empty;
        (bool ok, , IPletherOracle.BatchOrderPriceCache memory cache) = oracle.updateBatchOrderExecutionPrice(
            address(this), _data(), _request(980, CfdTypes.Side.LONG, false, false), empty
        );
        assertTrue(ok, "first batch parse resolves");
        assertEq(cache.publishTime, 994, "cache newest time");
        assertEq(cache.oldestPublishTime, 990, "cache oldest time");
        // The sync wrote each feed's OWN tick to Pyth storage (real-Pyth semantics,
        // reproduced by the test-support mock's legacy sync): B is at 990, not 994.
        assertEq(pyth.getPriceUnsafe(FEED_A).publishTime, 994, "stored A equals its tick");
        assertEq(pyth.getPriceUnsafe(FEED_B).publishTime, 990, "stored B equals its tick");

        // Second order committed at 985: passes every Router-side reuse time check
        // (985 >= minReusable 980, 985 < oldest 990, 985 < publish 994,
        // 994 <= 985 + 15 window), so the Router would forward fee = 0 and reuse.
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector, FEED_B, uint256(990), uint256(994)
            )
        );
        oracle.updateBatchOrderExecutionPrice(
            address(this), new bytes[](0), _request(985, CfdTypes.Side.LONG, false, false), cache
        );
    }

    // ---- C3 control: aligned first ticks execute ----
    function test_C3_Control_AlignedFirstTickExecutes() public {
        vm.warp(1000);
        pyth.setUniquePrice(FEED_A, int64(100_000_000), uint64(10_000), int32(-8), 993, 980);
        pyth.setUniquePrice(FEED_B, int64(100_000_000), uint64(10_000), int32(-8), 993, 980);

        (bool ok, IPletherOracle.PriceSnapshot memory snap) =
            oracle.updateOrderExecutionPrice(address(this), _data(), _request(980, CfdTypes.Side.LONG, false, false));
        assertTrue(ok, "aligned first tick executes");
        assertEq(snap.markPrice, 100_000_000, "neutral mark");
    }

    // ---- C3: first ticks spread 6s (> 5s limit) -> execution reverts, not ok=false ----
    // Coordinator central-run correction (2 Nov): on this direct oracle call the RAW
    // PletherOracle__PublishTimeDivergence error surfaces (mode 0, min 989, max 995,
    // limit 5), not the HistoricalParserFailure wrapper the first draft expected.
    // The C3 conclusion is unchanged: the order cannot execute on its first ticks,
    // and the next test proves later ticks cannot rescue it.
    function test_C3_FirstTickDivergence_RevertsPublishTimeDivergence() public {
        vm.warp(1000);
        // Window for commit 980 is (980, 995]; first ticks A@995, B@989: spread 6 > 5.
        pyth.setUniquePrice(FEED_A, int64(100_000_000), uint64(10_000), int32(-8), 995, 980);
        pyth.setUniquePrice(FEED_B, int64(100_000_000), uint64(10_000), int32(-8), 989, 980);

        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__PublishTimeDivergence.selector,
                uint256(0), // mode (order execution)
                uint256(989), // min publish time
                uint256(995), // max publish time
                uint256(5) // maxComponentPublishTimeDivergence limit
            )
        );
        oracle.updateOrderExecutionPrice(address(this), _data(), _request(980, CfdTypes.Side.LONG, false, false));
    }

    // ---- C3 no-escape: later aligned ticks are non-unique -> ok=false, order stays pending ----
    function test_C3_LaterTicksAreNonUnique_OrderStaysPending() public {
        vm.warp(1000);
        pyth.setUniquePrice(FEED_A, int64(100_000_000), uint64(10_000), int32(-8), 995, 980);
        pyth.setUniquePrice(FEED_B, int64(100_000_000), uint64(10_000), int32(-8), 989, 980);
        // First-tick attempt reverts (divergence); shown in the previous test. Now a
        // keeper retries with later, mutually aligned ticks whose prevPublishTimes are
        // already inside the window (>= commitTime + 1 = 981), so uniqueness fails.
        pyth.setUniquePrice(FEED_A, int64(100_000_000), uint64(10_000), int32(-8), 993, 990);
        pyth.setUniquePrice(FEED_B, int64(100_000_000), uint64(10_000), int32(-8), 993, 989);

        (bool ok, IPletherOracle.PriceSnapshot memory snap) =
            oracle.updateOrderExecutionPrice(address(this), _data(), _request(980, CfdTypes.Side.LONG, false, false));
        assertFalse(ok, "later ticks cannot satisfy uniqueness; order stays pending");
        assertEq(snap.price, 0, "no execution price produced");
    }
}
```

## Proof of Concept

// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Lane L1 (Oracle & Index) — second-pass candidates, distinct from the pre-existing
// test/competition/Lane1.t.sol (whose C1 = above-cap capped-base execution shift, and
// whose K-tests kill production-basket precision, LOW-36, cache forgery, calendar).
//
// C2: Batch cache reuse demands stored-feed coverage at the basket's NEWEST component
//     time for EVERY feed (PletherOracle.updateBatchOrderExecutionPrice ->
//     _requireStoredFeedCoverage(feedId, cache.publishTime)), but the cache's own
//     synchronization only guarantees each feed's OWN tick time, and the cache stores
//     no per-feed times. A spread basket (oldest < newest, the normal case for six
//     independently-ticking FX feeds) therefore fails reuse immediately after the very
//     parse that created it, unless unrelated later Pyth activity advanced every feed
//     past the basket's newest time. The Router predicts reuse without checking
//     coverage (OrderRouterExecutionSidecar._canReuseHistoricalBatchBasket checks only
//     times), forwards fee = 0, and only converts HistoricalParserFailure into a
//     pending stop; StoredFeedBehind propagates and reverts the whole batch
//     transaction, wiping earlier batch progress (ORACLE_SYNCHRONIZATION.md failure
//     table). The documented fallback ("Otherwise the Router requests a new historical
//     resolution") never runs for this case.
//
// C3: A delayed order's execution price is the FIRST Pyth tick after commit (unique
//     parse enforces prevPublishTime < commitTime + 1). If those first ticks span more
//     than maxComponentPublishTimeDivergence (5s), _computeBasketPriceFromFeeds reverts
//     PublishTimeDivergence inside the historical-parse try block, which the oracle
//     wraps as HistoricalParserFailure: single execution reverts outright, batch
//     execution stops with OracleFailure. No later tick can rescue the order, because
//     any later tick's prevPublishTime is >= commitTime + 1 and fails uniqueness
//     (PriceFeedNotFoundWithinRange -> ok = false, order stays pending). The order is
//     therefore unexecutable until its executionDeadline, with margin/bounty locked.
//     This is not attacker-triggerable (tick timing is a market-data property), so it
//     is graded Low; C2 is graded Medium (batch path is a documented core keeper path).

import {
    PletherOracleEngineMock,
    PletherOracleVaultMock
} from "../perps/spec/oracle/PletherOracleMocks.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {Test} from "forge-std/Test.sol";

contract L1OracleDivergenceAndCacheTest is Test {

    bytes32 internal constant FEED_A = bytes32(uint256(1));
    bytes32 internal constant FEED_B = bytes32(uint256(2));

    MockPyth internal pyth;
    PletherOracleEngineMock internal engine;
    PletherOracleVaultMock internal vault;
    PletherOracle internal oracle;

    bytes32[] internal feedIds;
    uint256[] internal weights;
    uint256[] internal basePrices;
    bool[] internal inversions;

    receive() external payable {}

    function setUp() public {
        pyth = new MockPyth();
        pyth.setSynchronizeLegacyUniquePrices(true);
        engine = new PletherOracleEngineMock();
        vault = new PletherOracleVaultMock();
        engine.setOrderRouter(address(this));

        feedIds.push(FEED_A);
        feedIds.push(FEED_B);
        weights.push(0.5e18);
        weights.push(0.5e18);
        basePrices.push(1e8);
        basePrices.push(1e8);
        inversions.push(false);
        inversions.push(false);

        oracle = new PletherOracle(address(engine), address(vault), address(pyth), feedIds, weights, basePrices, inversions);
    }

    function _data() internal pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = hex"01";
    }

    function _request(uint64 commitTime, CfdTypes.Side side, bool isClose, bool revertOnUnavailable)
        internal
        pure
        returns (IPletherOracle.OrderExecutionRequest memory)
    {
        return IPletherOracle.OrderExecutionRequest({
            commitTime: commitTime,
            targetPrice: 0,
            side: side,
            isClose: isClose,
            revertOnHistoricalUnavailable: revertOnUnavailable
        });
    }

    // ---- C2 control: a zero-spread basket reuses fine (this is why existing tests pass) ----
    function test_C2_Control_AlignedCacheReuseWorks() public {
        vm.warp(1000);
        pyth.setUniquePrice(FEED_A, int64(100_000_000), uint64(10_000), int32(-8), 995, 980);
        pyth.setUniquePrice(FEED_B, int64(100_000_000), uint64(10_000), int32(-8), 995, 980);

        IPletherOracle.BatchOrderPriceCache memory empty;
        (bool ok, , IPletherOracle.BatchOrderPriceCache memory cache) = oracle.updateBatchOrderExecutionPrice(
            address(this), _data(), _request(980, CfdTypes.Side.LONG, false, false), empty
        );
        assertTrue(ok, "first batch parse resolves");
        assertEq(cache.publishTime, 995, "cache newest time");
        assertEq(cache.oldestPublishTime, 995, "zero spread control");

        uint256 parses = pyth.parseUniqueCallCount();
        (bool ok2, IPletherOracle.PriceSnapshot memory reused,) = oracle.updateBatchOrderExecutionPrice(
            address(this), new bytes[](0), _request(991, CfdTypes.Side.LONG, false, false), cache
        );
        assertTrue(ok2, "aligned cache reuses");
        assertEq(reused.updateFee, 0, "reuse costs no fee");
        assertEq(pyth.parseUniqueCallCount(), parses, "reuse performs no new parse");
    }

    // ---- C2: spread basket (A@994, B@990) fails reuse straight after its own sync ----
    function test_C2_SpreadCacheReuse_RevertsStoredFeedBehind() public {
        vm.warp(1000);
        // First ticks after commit 980, spread 4s (within the 5s divergence limit).
        pyth.setUniquePrice(FEED_A, int64(100_000_000), uint64(10_000), int32(-8), 994, 980);
        pyth.setUniquePrice(FEED_B, int64(100_000_000), uint64(10_000), int32(-8), 990, 980);

        IPletherOracle.BatchOrderPriceCache memory empty;
        (bool ok, , IPletherOracle.BatchOrderPriceCache memory cache) = oracle.updateBatchOrderExecutionPrice(
            address(this), _data(), _request(980, CfdTypes.Side.LONG, false, false), empty
        );
        assertTrue(ok, "first batch parse resolves");
        assertEq(cache.publishTime, 994, "cache newest time");
        assertEq(cache.oldestPublishTime, 990, "cache oldest time");
        // The sync wrote each feed's OWN tick to Pyth storage (real-Pyth semantics,
        // reproduced by the test-support mock's legacy sync): B is at 990, not 994.
        assertEq(pyth.getPriceUnsafe(FEED_A).publishTime, 994, "stored A equals its tick");
        assertEq(pyth.getPriceUnsafe(FEED_B).publishTime, 990, "stored B equals its tick");

        // Second order committed at 985: passes every Router-side reuse time check
        // (985 >= minReusable 980, 985 < oldest 990, 985 < publish 994,
        // 994 <= 985 + 15 window), so the Router would forward fee = 0 and reuse.
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector, FEED_B, uint256(990), uint256(994)
            )
        );
        oracle.updateBatchOrderExecutionPrice(
            address(this), new bytes[](0), _request(985, CfdTypes.Side.LONG, false, false), cache
        );
    }

    // ---- C3 control: aligned first ticks execute ----
    function test_C3_Control_AlignedFirstTickExecutes() public {
        vm.warp(1000);
        pyth.setUniquePrice(FEED_A, int64(100_000_000), uint64(10_000), int32(-8), 993, 980);
        pyth.setUniquePrice(FEED_B, int64(100_000_000), uint64(10_000), int32(-8), 993, 980);

        (bool ok, IPletherOracle.PriceSnapshot memory snap) =
            oracle.updateOrderExecutionPrice(address(this), _data(), _request(980, CfdTypes.Side.LONG, false, false));
        assertTrue(ok, "aligned first tick executes");
        assertEq(snap.markPrice, 100_000_000, "neutral mark");
    }

    // ---- C3: first ticks spread 6s (> 5s limit) -> execution reverts, not ok=false ----
    // Coordinator central-run correction (2 Nov): on this direct oracle call the RAW
    // PletherOracle__PublishTimeDivergence error surfaces (mode 0, min 989, max 995,
    // limit 5), not the HistoricalParserFailure wrapper the first draft expected.
    // The C3 conclusion is unchanged: the order cannot execute on its first ticks,
    // and the next test proves later ticks cannot rescue it.
    function test_C3_FirstTickDivergence_RevertsPublishTimeDivergence() public {
        vm.warp(1000);
        // Window for commit 980 is (980, 995]; first ticks A@995, B@989: spread 6 > 5.
        pyth.setUniquePrice(FEED_A, int64(100_000_000), uint64(10_000), int32(-8), 995, 980);
        pyth.setUniquePrice(FEED_B, int64(100_000_000), uint64(10_000), int32(-8), 989, 980);

        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__PublishTimeDivergence.selector,
                uint256(0), // mode (order execution)
                uint256(989), // min publish time
                uint256(995), // max publish time
                uint256(5) // maxComponentPublishTimeDivergence limit
            )
        );
        oracle.updateOrderExecutionPrice(address(this), _data(), _request(980, CfdTypes.Side.LONG, false, false));
    }

    // ---- C3 no-escape: later aligned ticks are non-unique -> ok=false, order stays pending ----
    function test_C3_LaterTicksAreNonUnique_OrderStaysPending() public {
        vm.warp(1000);
        pyth.setUniquePrice(FEED_A, int64(100_000_000), uint64(10_000), int32(-8), 995, 980);
        pyth.setUniquePrice(FEED_B, int64(100_000_000), uint64(10_000), int32(-8), 989, 980);
        // First-tick attempt reverts (divergence); shown in the previous test. Now a
        // keeper retries with later, mutually aligned ticks whose prevPublishTimes are
        // already inside the window (>= commitTime + 1 = 981), so uniqueness fails.
        pyth.setUniquePrice(FEED_A, int64(100_000_000), uint64(10_000), int32(-8), 993, 990);
        pyth.setUniquePrice(FEED_B, int64(100_000_000), uint64(10_000), int32(-8), 993, 989);

        (bool ok, IPletherOracle.PriceSnapshot memory snap) =
            oracle.updateOrderExecutionPrice(address(this), _data(), _request(980, CfdTypes.Side.LONG, false, false));
        assertFalse(ok, "later ticks cannot satisfy uniqueness; order stays pending");
        assertEq(snap.price, 0, "no execution price produced");
    }
}
