# \#99407 [SC-Medium] Inconsistent historical-basket cache validation reverts otherwise executable FIFO order batches

**Submitted on Nov 13th 2026 at 14:37:35 UTC by @manvi for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99407
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description


## Brief / Introduction

While reviewing the historical-basket timestamp handling following Cecuro MED-23, I found an inconsistency between the Oracle's initial validation of a mixed-timestamp Pyth basket and its validation of the same basket when the next FIFO order reuses the batch cache.

A historical basket can be accepted when its components have different publication times within the configured divergence allowance. The first order verifies that each corresponding Pyth feed has been stored through **its own parsed timestamp**. The Oracle then caches the aggregate basket with `publishTime` equal to the **newest** component timestamp. On the next eligible order, cache reuse instead requires **every stored feed** to have reached that newest timestamp. A feed that was legitimately published one second earlier fails the second requirement, even though nothing about its accepted basket changed.

I reproduced this through production-format V3 order requests and `executeOrderBatch`, using the frozen audit commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. With two valid queued orders and an accepted basket whose oldest and newest components differ by only one second, the second order raises `PletherOracle__StoredFeedBehind`. The Router propagates that revert, rolling back the entire transaction, including the first order's otherwise successful execution. The *same payload* executes both orders successfully when I submit them individually.

I wrote five Foundry tests to distinguish the cache-reuse defect from bad price data, invalid orders, and ordinary batch atomicity. **All five passed.** The issue is specifically an avoidable batch-execution failure under acceptable price data, affecting permissionless keeper reliability and FIFO throughput.

**Selected impact: Security Best practices Insight.**

## Vulnerability Details

### 1. Initial historical parsing accepts per-component storage coverage

The Oracle resolves execution pricing through `PletherOracle.updateBatchOrderExecutionPrice()`. For a new historical basket, `_resolveHistoricalOrderBasket()` parses the first qualifying post-commit Pyth ticks and checks their allowed divergence. After updating stored Pyth prices, it verifies storage coverage separately for each parsed feed:

```solidity
for (uint256 i; i < parsedFeeds.length; ++i) {
    _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);
}
```

This is appropriate: if a feed's first qualifying tick was published at `T+4`, its stored timestamp need only be at least `T+4` to cover that feed's parsed price. Another feed in the same valid basket may be published at `T+5`.

The basket computation records the **latest** component time in `basket.publishTime` and the earliest in `basket.oldestPublishTime`. Both belong to the accepted historical basket.

### 2. Cache reuse applies the newest timestamp to every stored feed

For a subsequent FIFO order, `updateBatchOrderExecutionPrice()` first decides whether the cache can be reused using `_canReuseHistoricalBatchBasket()`. If the cache is eligible, the implementation executes:

```solidity
if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
    for (uint256 i; i < pythFeedIds.length; ++i) {
        _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
    }
    // Reuse the previously accepted basket.
}
```

Here `cache.publishTime` is the **newest** component time, not the publication time associated with each Pyth feed. `_requireStoredFeedCoverage()` reverts whenever `pyth.getPriceUnsafe(feedId).publishTime` is below that requested time.

The cache's eligibility predicate checks the later order's commit time against the cached earliest/latest timestamps; it does **not** separately establish that every stored component has reached the latest timestamp. The strict check therefore rejects a state the preceding parsing path deliberately accepted and stored.

This is not a case of corrupt, stale, or manipulated prices. In my test, all six feeds are post-commit, the oldest and newest ticks differ by one second, and the same data is accepted through individual order execution.

### 3. An unexpected cache error reverts the entire Router batch

The Router's `OrderRouterExecutionSidecar._resolveOrderOracle()` calls the Oracle inside a `try/catch`. It handles `PletherOracle__HistoricalParserFailure` as a classified `OracleFailure`, but explicitly rethrows other Oracle errors:

```solidity
} catch (bytes memory reason) {
    if (
        reason.length != 68
            || _selector(reason) != IPletherOracle.PletherOracle__HistoricalParserFailure.selector
    ) {
        assembly ("memory-safe") {
            revert(add(reason, 32), mload(reason))
        }
    }
    return (OrderV3Types.PendingReason.OracleFailure, result, state.oracleCache);
}
```

`PletherOracle__StoredFeedBehind` is not `HistoricalParserFailure`, so it is propagated to the outer batch frame. The first order's settlement, Engine mark update, Pyth storage updates, and order lifecycle changes are rolled back with the failed batch. This is the observed result in the test, not merely an inferred consequence.

### 4. Reproduction with correctly ordered historical ticks

I used the Oracle synchronization fixture's six-feed basket and its mock Pyth implementation, with the production V3 Router request format. The two trader commitments and their qualifying historical basket have these timestamps:

| Item | Timestamp | Meaning |
|---|---|---|
| Alice commits | `T` | First FIFO order |
| Bob commits | `T+3` | Second FIFO order |
| Feed 0 tick | `T+4` | Oldest accepted component |
| Feeds 1–5 ticks | `T+5` | Newest accepted components |
| Batch execution | `T+6` | Both historical updates are available |

All feed updates use `T` as their previous publication time in the fixture; both historical ticks are after **both** commitments, and their one-second divergence is permitted. The first order installs the basket and caches `oldestPublishTime = T+4`, `publishTime = T+5`.

The second order qualifies for cache reuse because its commit time `T+3` is before `T+4` and `T+5`. The reuse coverage check then requests `T+5` for feed 0, whose stored time is `T+4`, causing the exact revert:

```text
PletherOracle__StoredFeedBehind(feed0, T+4, T+5)
```

Since the revert occurs in the outer batch Oracle preparation, the transaction preserves neither order's execution; both remain pending. In contrast, calling `executeOrder()` twice with the same data causes two independent historical parses and completes both orders.

### 5. Why the existing test coverage does not catch this case

The repository has `test_BatchReusesBasketWhenEveryComponentFollowsBothCommits()` in `test/perps/spec/oracle/OracleSynchronization.t.sol`. That test also constructs a valid mixed-time basket, but **preloads newer stored prices at `T+6`** before invoking the batch. Because the stored timestamp of each feed is already newer than the cached latest component, the strict coverage check succeeds.

My preloading control reproduces that successful behavior. The separate no-preload case establishes the missing path: a valid mixed-time basket is parsed, installed, and immediately rejected when reused within the *same batch*. This is also consistent with the direct Oracle test: the Oracle generates a successful cache, then fails to reuse the cache it just returned.

### 6. Relationship to Cecuro MED-23

Cecuro MED-23 concerned historical-basket timestamp alignment in protection-trigger handling. The frozen code records the newest component timestamp as the basket's publication time. The observation here is **not** a claim that the newest-component timestamp should be reverted to the oldest time. The issue is the second-step storage-coverage check applying that aggregate maximum to each individual feed without confirming it is available.

The previously reviewed historical-parser path and the cache-reuse path impose inconsistent requirements. My test isolates that inconsistency independently of protection triggering or changes to the market price.

## Impact Details

The immediate effect is an avoidable failure of `executeOrderBatch()` for multiple otherwise executable FIFO orders when the Pyth components have acceptable but nonidentical publication timestamps and storage has not been preloaded to the newest time.

- **No settlement progress:** the whole batch reverts, including the first valid order that was already prepared for execution.
- **Permissionless keeper impact:** the caller spends gas on a batch that cannot complete with that valid mixed-time payload; repeated batching with the same storage conditions repeats the failure.
- **Operational workaround:** executing the two orders individually succeeds with the identical payload, and preloading sufficiently new per-feed prices also avoids the failure. That workaround is why I am reporting a concrete batch-liveness and validation issue, rather than claiming that funds are frozen or permanently inaccessible.

I have not assigned a fabricated gas cost or a timed-freeze duration. The measured result is the deterministic **batch revert**, with a passing individual-execution control and two successful batch controls. The finding matters where keepers use batch execution to settle multiple pending orders at one historical basket, especially when different currency feeds publish at slightly different moments.

### Recommended remediation

The Oracle should not declare a cache reusable based solely on aggregate timestamps and then revert because some component lacks **per-feed** storage coverage.

A robust implementation can either retain the accepted publication timestamp for each component and verify coverage against that component's timestamp, or treat a coverage mismatch as a cache miss and perform a fresh historical parse. The latter must preserve the current fee accounting, post-commit uniqueness, expiration windows, and basket provenance checks; it must not accept unverified caller-supplied cache data.

I would also add a regression test for a mixed-time historical basket **without preloaded newer Pyth prices**, with two commit times both preceding its oldest component. The expected result is successful batch execution or a safe fresh-parse fallback, not an unclassified revert that discards the entire valid batch.

## References

- [Affected Oracle batch function, `PletherOracle.sol:245–299`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L245-L299)
- [Historical parser and per-feed coverage, `PletherOracle.sol:596–629`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L596-L629)
- [Coverage guard and revert, `PletherOracle.sol:645–653`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L645-L653)
- [Newest/oldest component times, `PletherOracle.sol:845–875`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L845-L875)
- [Cache eligibility, `PletherOracle.sol:974–990`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L974-L990)
- [Router's catch/rethrow, `OrderRouterExecutionSidecar.sol:995–1029`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L995-L1029)
- [Existing preloaded-price control, `OracleSynchronization.t.sol:434–452`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/oracle/OracleSynchronization.t.sol#L434-L452)
- [Router/oracle batching semantics, `SECURITY.md:647–678`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L647-L678)
- [Cecuro audit, including MED-23](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)
- [Plether competition scope and known-issue requirements](https://immunefi.com/audit-competition/plether/scope/)
- [Plether competition Insight reporting terms](https://immunefi.com/audit-competition/plether/information/)


## Proof of Concept

## Environment and scope

- **Commit:** `a82d4fefb02768e5b3bddcc97dee371eee24fea8`
- **Test file:** `packages/perps/poc/PletherOracleMixedTimeCacheV3_Final.t.sol`
- **In-scope behavior:** `PletherOracle` batch-cache handling and `OrderRouterExecutionSidecar` error propagation
- **Execution:** Repository Foundry suite, Solidity `0.8.35`; production-format `OrderV3Types.OrderRequest` and the regular Router `executeOrder` / `executeOrderBatch` entrypoints
- **Test dependencies:** `OracleSynchronizationTestBase` supplies funded trader accounts, the local Engine/Router fixture, mock USDG and `MockPyth`. The production contracts in `packages/perps/src` are unmodified.

The tests exercise ordinary permissionless execution with valid mock post-commit Pyth history, rather than injecting invalid storage or impersonating the Oracle. Only the out-of-scope price source and test collateral are mocked, as in the repository's existing test suite.

## Reproduction steps

1. At fixture time `T`, Alice commits an ordinary 10,000-unit V3 LONG. Three seconds later, Bob commits the same request. Both are valid and remain adjacent in the global FIFO queue.
2. Advance time to `T+6`. Construct a historical update payload with feed 0 at `T+4` and all remaining feeds at `T+5`; each has previous publication time `T`. Every tick is after both commitments, and the one-second divergence is valid.
3. Call `router.executeOrderBatch{value: 2 * FEE}(bobId, data)`. The first order successfully constructs the historical basket and its cache. The second order's reuse attempt reverts with `PletherOracle__StoredFeedBehind(feed0, T+4, T+5)`.
4. Assert rollback of `nextExecuteId`, the Engine mark, mock Pyth parse counter and both traders' positions. Both order lifecycles remain `Pending`.
5. Repeat the same setup in separate clean Foundry tests: the same data succeeds with two individual `executeOrder` calls; uniform feed times succeed in a batch with one parse; preloading newer stored feed prices also succeeds; and a direct Oracle call confirms that its newly returned cache fails its own reuse-coverage check.

## Five PoC tests

| Test | Observation |
|---|---|
| `test_PoC_ValidMixedTimeBasketRevertsEntireBatch` | The valid mixed-time basket triggers `StoredFeedBehind`, reverting both orders and the Engine mark |
| `test_Control_IdenticalMixedTimeBasketWorksWithSingleOrders` | The identical payload executes both orders through individual V3 calls |
| `test_Control_UniformTickBasketSucceedsInBatch` | Uniform component timestamps permit one-parse batch execution |
| `test_Control_PreloadedNewerStoredPricesAvoidRevert` | Newer stored feed timestamps satisfy the stricter reuse check and let the batch complete |
| `test_Oracle_OriginalParseSucceedsButOwnCacheCannotBeReused` | The Oracle accepts the original basket but cannot reuse the resulting cache without additional storage data |

## Run the tests

Create `packages/perps/poc/PletherOracleMixedTimeCacheV3_Final.t.sol` with the full Solidity source below, then run:

```bash
cd /workspaces/audit-comp-plether/packages/perps
export PATH="$HOME/.foundry/bin:$PATH"

EXPECTED=a82d4fefb02768e5b3bddcc97dee371eee24fea8
ACTUAL=$(git rev-parse HEAD)
if [ "$ACTUAL" != "$EXPECTED" ]; then
  echo "Wrong audited commit: $ACTUAL" >&2
  exit 1
fi

set -o pipefail

export FOUNDRY_TEST=poc
export FOUNDRY_OUT=out-oracle-mixed-v3
export FOUNDRY_CACHE_PATH=cache-oracle-mixed-v3

forge fmt poc/PletherOracleMixedTimeCacheV3_Final.t.sol
forge test --offline \
  --match-path 'poc/PletherOracleMixedTimeCacheV3_Final.t.sol' \
  --match-contract '^PletherOracleMixedTimeCacheV3Test$' \
  -vvv 2>&1 | tee oracle-mixed-time-v3-final.log

grep -E 'PASS|FAIL|Suite result|POC:|CONTROL:|ORACLE:' oracle-mixed-time-v3-final.log

git diff -- src/
```

## Observed Foundry output

The following is the successful run supplied from the Codespace, not a projected result:

```text
Ran 5 tests for poc/PletherOracleMixedTimeCacheV3_Final.t.sol:PletherOracleMixedTimeCacheV3Test
[PASS] test_Control_IdenticalMixedTimeBasketWorksWithSingleOrders() (gas: 4332296)
Logs:
  CONTROL: identical mixed-time data executed both orders individually

[PASS] test_Control_PreloadedNewerStoredPricesAvoidRevert() (gas: 4268848)
Logs:
  CONTROL: newer stored Pyth prices avoid cache-coverage revert

[PASS] test_Control_UniformTickBasketSucceedsInBatch() (gas: 4241376)
Logs:
  CONTROL: uniform-time basket executes both orders with one parse

[PASS] test_Oracle_OriginalParseSucceedsButOwnCacheCannotBeReused() (gas: 2952356)
Logs:
  ORACLE: accepted historical basket fails its own cache-reuse check

[PASS] test_PoC_ValidMixedTimeBasketRevertsEntireBatch() (gas: 4418633)
Logs:
  POC: valid mixed-time basket reverted complete FIFO batch
  POC: both pending orders and original mark preserved

Suite result: ok. 5 passed; 0 failed; 0 skipped; finished in 100.65ms (74.47ms CPU time)
Ran 1 test suite in 211.00ms (100.65ms CPU time): 5 tests passed, 0 failed, 0 skipped (5 total tests)
```

## Full Solidity PoC

The report copy below preserves the five exercised test cases and assertions. For readability, only the `console2.log` message text was synchronized with the captured Codespace log; this does not change test behavior or the executable assertions.

**File:** `packages/perps/poc/PletherOracleMixedTimeCacheV3_Final.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {OracleSynchronizationTestBase} from "../test/perps/shared/OracleSynchronizationTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Mixed-time valid basket cache-reuse regression diagnostic.
/// @dev Production V3 requests and Router entrypoints; external Pyth and test USDG are mocked.
///      A passing test demonstrates batch liveness failure, not fund loss.
contract PletherOracleMixedTimeCacheV3Test is OracleSynchronizationTestBase {
    uint256 private clientId;

    function _v3Commit(address trader) internal returns (uint64 orderId) {
        uint32 window = uint32(router.maxExecutionWindowSeconds());
        OrderV3Types.OrderRequest memory req = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(++clientId),
            side: CfdTypes.Side.LONG,
            sizeDelta: 10_000e18,
            marginDelta: 1_000e6,
            targetPrice: 1,
            isClose: false,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp()) + uint64(window),
                executionWindowSeconds: window,
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
            })
        });
        vm.prank(trader);
        orderId = router.commitOrder(req);
        assertGt(orderId, 0, "V3 commitment missing");
    }

    function _pair() internal returns (uint64 t0, uint64 aliceId, uint64 bobId) {
        t0 = uint64(vm.getBlockTimestamp());
        aliceId = _v3Commit(ALICE);
        _advance(t0 + 3);
        assertEq(vm.getBlockTimestamp(), t0 + 3, "second commit time incorrect");
        bobId = _v3Commit(BOB);
        _advance(t0 + 6);
        assertEq(vm.getBlockTimestamp(), t0 + 6, "execution time incorrect");
        assertEq(aliceId + 1, bobId, "FIFO adjacent orders expected");
    }

    function _mixed(uint64 t0) internal pure returns (bytes[] memory data) {
        data = _payload(t0 + 5, t0);
        MockPyth.FeedUpdate[] memory updates = abi.decode(data[0], (MockPyth.FeedUpdate[]));
        updates[0].price.publishTime = t0 + 4;
        data[0] = abi.encode(updates);
    }

    function test_PoC_ValidMixedTimeBasketRevertsEntireBatch() public {
        (uint64 t0, uint64 aliceId, uint64 bobId) = _pair();
        bytes[] memory data = _mixed(t0);
        uint64 initialMark = engine.lastMarkTime();
        uint256 initialParses = baseMockPyth.parseUniqueCallCount();
        bytes32 laggingId = _synchronizationFeedIds()[0];

        vm.expectRevert(abi.encodeWithSelector(
            IPletherOracle.PletherOracle__StoredFeedBehind.selector,
            laggingId, uint256(t0 + 4), uint256(t0 + 5)
        ));
        router.executeOrderBatch{value: 2 * FEE}(bobId, data);

        assertEq(router.nextExecuteId(), aliceId, "First settled order must roll back");
        assertEq(engine.lastMarkTime(), initialMark, "Engine mark must roll back");
        assertEq(baseMockPyth.parseUniqueCallCount(), initialParses, "Pyth parse must roll back");
        assertEq(uint8(router.lifecycleBook().lifecycleStatus(aliceId)),
            uint8(OrderV3Types.LifecycleStatus.Pending));
        assertEq(uint8(router.lifecycleBook().lifecycleStatus(bobId)),
            uint8(OrderV3Types.LifecycleStatus.Pending));
        (uint256 sizeAlice,,,,,,) = engine.positions(ALICE);
        (uint256 sizeBob,,,,,,) = engine.positions(BOB);
        assertEq(sizeAlice, 0);
        assertEq(sizeBob, 0);
        console2.log("POC: valid mixed-time basket reverted complete FIFO batch");
        console2.log("POC: both pending orders and original mark preserved");
    }

    function test_Control_IdenticalMixedTimeBasketWorksWithSingleOrders() public {
        (uint64 t0, uint64 aliceId, uint64 bobId) = _pair();
        bytes[] memory data = _mixed(t0);
        OrderV3Types.ExecutionResult memory a = router.executeOrder{value: 2 * FEE}(aliceId, data);
        OrderV3Types.ExecutionResult memory b = router.executeOrder{value: 2 * FEE}(bobId, data);
        assertEq(uint8(a.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(uint8(b.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(router.nextExecuteId(), 0);
        assertEq(baseMockPyth.parseUniqueCallCount(), 2);
        (uint256 sizeAlice,,,,,,) = engine.positions(ALICE);
        (uint256 sizeBob,,,,,,) = engine.positions(BOB);
        assertEq(sizeAlice, 10_000e18);
        assertEq(sizeBob, 10_000e18);
        console2.log("CONTROL: identical mixed-time data executed both orders individually");
    }

    function test_Control_UniformTickBasketSucceedsInBatch() public {
        (uint64 t0,, uint64 bobId) = _pair();
        OrderV3Types.BatchResult memory result =
            router.executeOrderBatch{value: 2 * FEE}(bobId, _payload(t0 + 5, t0));
        assertEq(result.terminalCount, 2);
        assertEq(result.nextOrderId, 0);
        assertEq(baseMockPyth.parseUniqueCallCount(), 1);
        console2.log("CONTROL: uniform-time basket executes both orders with one parse");
    }

    function test_Control_PreloadedNewerStoredPricesAvoidRevert() public {
        (uint64 t0,, uint64 bobId) = _pair();
        baseMockPyth.updatePriceFeeds{value: FEE}(_payload(t0 + 6, t0 + 5));
        OrderV3Types.BatchResult memory result =
            router.executeOrderBatch{value: 2 * FEE}(bobId, _mixed(t0));
        assertEq(result.terminalCount, 2);
        assertEq(result.nextOrderId, 0);
        assertEq(baseMockPyth.parseUniqueCallCount(), 1);
        console2.log("CONTROL: newer stored Pyth prices avoid cache-coverage revert");
    }

    function test_Oracle_OriginalParseSucceedsButOwnCacheCannotBeReused() public {
        (uint64 t0,,) = _pair();
        IPletherOracle.BatchOrderPriceCache memory empty;
        (bool ok,, IPletherOracle.BatchOrderPriceCache memory cache) =
            pletherOracle.updateBatchOrderExecutionPrice{value: 2 * FEE}(
                address(this), _mixed(t0), _request(t0, false), empty
            );
        assertTrue(ok);
        assertTrue(cache.hasHistoricalBasket);
        assertEq(cache.oldestPublishTime, t0 + 4);
        assertEq(cache.publishTime, t0 + 5);
        bytes32 id0 = _synchronizationFeedIds()[0];
        assertEq(baseMockPyth.getPriceUnsafe(id0).publishTime, t0 + 4);

        vm.expectRevert(abi.encodeWithSelector(
            IPletherOracle.PletherOracle__StoredFeedBehind.selector,
            id0, uint256(t0 + 4), uint256(t0 + 5)
        ));
        pletherOracle.updateBatchOrderExecutionPrice(
            address(this), new bytes[](0), _request(t0 + 3, false), cache
        );
        console2.log("ORACLE: accepted historical basket fails its own cache-reuse check");
    }
}
```

## Result interpretation

The tests reproduce an implementation-level mismatch in accepted basket vs cached-basket coverage. They prove a transaction-level batch revert, complete rollback, and successful alternate execution paths. No privileged operation or manipulated protocol storage is necessary in the test fixture.

The test does **not** measure direct asset theft, protocol insolvency, or any minimum one-hour unavailability period. Those are not the impacts claimed in this Insight. The specific recommendation is to make the cache eligibility and per-component coverage checks agree, or to fall back to an independently validated parse instead of reverting a valid batch.
