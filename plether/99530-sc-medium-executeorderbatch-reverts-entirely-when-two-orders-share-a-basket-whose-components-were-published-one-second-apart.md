# \#99530 [SC-Medium] executeOrderBatch reverts entirely when two orders share a basket whose components were published one second apart

**Submitted on Nov 14th 2026 at 03:44:09 UTC by @tinnohofficial for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99530
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
`executeOrderBatch` reuses the first order's historical Pyth basket for later orders committed before that basket. The reuse path requires every stored feed to be at least as new as the basket's **newest** component, while the parse that created the basket only required each feed at its **own** publish time. A valid basket whose components differ by one second (the Oracle allows five) therefore prices the first order and then reverts the whole transaction on the second, rolling back the first order's completed execution. This contradicts ORACLE_SYNCHRONIZATION.md, which says a cache that does not satisfy coverage is not reused: "Otherwise the Router requests a new historical resolution." The repository's own test of this path expects an unequal-time basket to be reused, and it passes only because it first writes a newer payload to Pyth storage: with that single line removed, the same batch reverts with `PletherOracle__StoredFeedBehind` while both orders execute one at a time with the same payload. In a second PoC with the six release feeds, two traders commit a 100,000-size open in the same block; an equal-time batch executes both, and the batch with a one-second skew reverts, burns 2,394,777 gas, executes nothing, and reverts again 27 seconds later.

## Vulnerability Details
A fresh historical parse builds the basket, writes the payload to Pyth storage and checks each feed against the time that feed was parsed at (`PletherOracle.sol` L613-L626):

```solidity
basket = _computeBasketPriceFromFeeds(PriceMode.OrderExecution, parsedFeeds, maxComponentPublishTimeDivergence);
pyth.updatePriceFeeds{value: pythFee}(pythUpdateData);
for (uint256 i; i < parsedFeeds.length; ++i) {
    _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime); // <-@ each feed, its own time
}
```

The basket's `publishTime` is the newest component and `oldestPublishTime` the oldest (L874-L875). Component times may differ by up to `maxComponentPublishTimeDivergence`, 5 seconds by default (L90, L855-L857).

For the next order of the batch the Oracle reuses that basket when the order was committed before every component (L974-L988). The reuse path then checks storage again, but against a single timestamp (L262-L266):

```solidity
if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
    // Cache contents are caller-supplied; independently prove storage coverage, not price authenticity.
    for (uint256 i; i < pythFeedIds.length; ++i) {
        _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime); // <-@ every feed vs the NEWEST component
    }
```

Storage holds each feed at its own time, as the parse verified moments earlier. Any feed older than the newest component fails `storedPublishTime < requiredPublishTime` and the call reverts `PletherOracle__StoredFeedBehind` (L645-L653).

The Router decides on reuse with the same time predicate and no coverage test (`OrderRouterExecutionSidecar.sol` L997, L1074-L1093). Its `catch` around the Oracle call handles only `PletherOracle__HistoricalParserFailure` and re-throws everything else (L1009-L1030):

```solidity
} catch (bytes memory reason) {
    if (reason.length != 68 || _selector(reason) != IPletherOracle.PletherOracle__HistoricalParserFailure.selector) {
        assembly ("memory-safe") {
            revert(add(reason, 32), mload(reason)) // <-@ StoredFeedBehind reverts the whole batch
        }
    }
```

So the batch never falls back to a new parse for the second order. The whole `executeOrderBatch` call reverts, including orders it had already finalized.

**Why this is not intended behaviour**
1. ORACLE_SYNCHRONIZATION.md L61-L64: "Reuse for a later order requires all components to follow that order's commit, fit its historical window and satisfy cache coverage. Otherwise the Router requests a new historical resolution." The Router never tests coverage and never requests the new resolution.
2. The two coverage checks disagree about the same basket in the same transaction. The parse accepts a feed stored at its own publish time; the reuse path rejects that same stored feed because another component is one second newer. Every component of the cached basket is in storage, so there is nothing for a coverage check to reject.
3. The same document lists a `StoredFeedBehind` revert for "failed cache coverage" (L109). That row describes a cache whose feeds are not in storage. Here coverage holds for every feed, and the revert comes only from comparing against the newest component's time.
4. The repository's own test expects this reuse to work. `test_BatchReusesBasketWhenEveryComponentFollowsBothCommits` (`test/perps/spec/oracle/OracleSynchronization.t.sol` L434-L453) builds a basket with one component at `t+4` and the others at `t+5` and asserts that the batch executes both orders with a single parse ("compatible later order reuses the signed basket"). It only passes because it first calls `baseMockPyth.updatePriceFeeds` with a payload dated `t+6` (L445-L446), which lifts every stored feed above the newest component. The PoC replays that scenario unchanged (both orders execute) and without that one write (the whole batch reverts `StoredFeedBehind(feed, t+4, t+5)`).
5. Unequal component times are an expected input: the Oracle has a dedicated divergence limit for them and the release checklist requires "real target-chain payloads with unequal component times" (ORACLE_SYNCHRONIZATION.md L145).
6. The caller cannot fix it with a different payload. The unique-first-tick rule fixes which update of each feed prices an order, and the batch's own payload stores the older components at their own times, so the same batch call reverts on every retry until some unrelated transaction moves Pyth storage for every feed to the newest component's time or later.

## Impact Details
Griefing: batch execution makes no progress for any pair of reusable orders whose basket has unequal component times, and each attempt costs the caller the full gas of a reverted batch. Two orders are reusable whenever both were committed before the basket's oldest component (three seconds apart in the repository scenario, the same block in the six-feed PoC), which any trader can arrange by committing right after another trader's order.

| Repository scenario (orders at t and t+3; one component at t+4, others at t+5) | Result |
|---|---|
| `executeOrderBatch` after a payload dated t+6 was written to Pyth storage (control, the repository's test) | 2 terminal, one parse |
| `executeOrderBatch` without that write | reverts `StoredFeedBehind(feed, t+4, t+5)`, head unchanged, first order rolled back |
| `executeOrder` for each order, same payload | both `Executed` |

| Six release feeds: two traders commit in the same block, same block of execution | Result |
|---|---|
| `executeOrder` for each, skewed payload (control) | both `Executed` |
| `executeOrderBatch`, equal-time payload (control) | 2 terminal, both executed |
| `executeOrderBatch`, payload with one feed 1 s newer | reverts `StoredFeedBehind(feed, T+1, T+2)` |
| gas burned by the reverted batch call | 2,394,777 |
| queue head and trader A's position after the revert | head unchanged, no position (execution rolled back) |
| same batch 27 s later | reverts again, no progress |

- Trader A's order is individually executable and was finalized inside the batch, yet its execution is rolled back because trader B's order followed it. Until someone executes the orders one at a time, both stay pending and expire at the end of their execution window.
- A keeper that groups contiguous ready orders into one batch call gets the same revert on every retry, because the payload is determined by the orders' commit time.
- Ordinary close orders carry no execution bounty, so no outside keeper is paid to execute them singly.

**Recommendation.** In the reuse path check each feed against the time it had in the cached basket (or against `cache.oldestPublishTime`), as the parse does. Alternatively have the Router fall back to a new historical resolution when reuse coverage fails, as the document describes, instead of re-throwing.

## References
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L262-L266

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L613-L626

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L645-L653

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L974-L988

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L997-L1030

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1074-L1093

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md?plain=1#L59-L64

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/oracle/OracleSynchronization.t.sol#L434-L453

## Proof of Concept

# Proof of concept

Two Foundry test files for the repository at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. Both use only public Router entry points of the unmodified contracts with the repository's own test harness and its Pyth mock.

- `BatchReuseUnequalTimes.t.sol` runs on the repository's own oracle-synchronization fixture (`test/perps/shared/OracleSynchronizationTestBase.sol`) and replays the repository's own reuse scenario, `test_BatchReusesBasketWhenEveryComponentFollowsBothCommits` in `test/perps/spec/oracle/OracleSynchronization.t.sol`, once unchanged and once without its single write of a newer payload to Pyth storage.
- `BatchCacheCoverage.t.sol` deploys the Oracle with the six release feeds, weights, base prices and inversions and the release risk parameters (maintenance 0.1%, initial 0.2%, execution fee 0.04%, VPI factor 1%, basket confidence cap 10 bp, adverse-confidence multiplier 25%, Senior limit 40M USDG / 80%).

## Run
Save both files below in `packages/perps/test/poc/`, then from the repository root:

```
forge test --root packages/perps --match-path 'test/poc/Batch*.t.sol' -vv
```

## Output
```
Ran 2 tests for test/poc/BatchReuseUnequalTimes.t.sol:BatchReuseUnequalTimesTest
[PASS] test_Control_RepositoryScenario_NewerStoragePresent_BatchExecutesBoth() (gas: 4262708)
Logs:
  newer payload already in Pyth storage: batch terminalCount 2
[PASS] test_RepositoryScenario_WithoutNewerStorage_WholeBatchReverts() (gas: 5992803)
Logs:
  no newer payload in Pyth storage: batch reverted StoredFeedBehind; queue head still 1
  single executions with the same payload (2 = Executed): 2 2
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished
Ran 1 test for test/poc/BatchCacheCoverage.t.sol:BatchCacheCoverageTest
[PASS] test_BatchRevertsWhileSingleExecutionsSucceed() (gas: 12408949)
Logs:
  control single A status (2=Executed) 2
  control single B status (2=Executed) 2
  control equal-time batch terminalCount 2
  batch with 1s component skew: reverted with PletherOracle__StoredFeedBehind; head still 1
  gas burned by the reverted batch call 2394777
  retry at commit+30s: reverted again; head still 1
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished
Ran 2 test suites: 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

## What the tests show
`BatchReuseUnequalTimes.t.sol` (ALICE commits at t, BOB at t+3; the basket has one component published at t+4 and the others at t+5, so every component follows both commits):
1. `test_Control_RepositoryScenario_NewerStoragePresent_BatchExecutesBoth`: unchanged repository scenario. A payload dated t+6 is written to Pyth storage first. `executeOrderBatch` executes both orders with one parse.
2. `test_RepositoryScenario_WithoutNewerStorage_WholeBatchReverts`: the same scenario without that write. `executeOrderBatch` reverts with `PletherOracle__StoredFeedBehind(feed, t+4, t+5)`. The queue head is still order 1 and ALICE has no position. `executeOrder` then executes each order on its own with the same payload.

`BatchCacheCoverage.t.sol` (two traders commit a 100,000-size open in the same block at time T; five feeds are published at T+1 and one at T+2):
1. Control: `executeOrder` for each order with that payload. Both are `Executed`.
2. Control: `executeOrderBatch` with an equal-time payload (all feeds at T+1). Both orders are executed in one call.
3. `executeOrderBatch` with the one-second skew reverts with `PletherOracle__StoredFeedBehind(feed, T+1, T+2)`. The queue head does not move, trader A has no position, the caller's ETH is returned and 2,394,777 gas is burned.
4. The same call 27 seconds later reverts again with no progress.

## Test file 1: BatchReuseUnequalTimes.t.sol
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {OracleSynchronizationTestBase} from "../perps/shared/OracleSynchronizationTestBase.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice The repository's own reuse test (`test_BatchReusesBasketWhenEveryComponentFollowsBothCommits` in
///         test/perps/spec/oracle/OracleSynchronization.t.sol) expects a batch to reuse a basket whose components
///         were published one second apart. It writes a newer payload to Pyth storage before the batch. This file
///         runs the same scenario twice on the repository's own fixture: once unchanged, once without that write.
contract BatchReuseUnequalTimesTest is OracleSynchronizationTestBase {

    /// @dev Unchanged copy of the repository scenario: a newer payload is in Pyth storage, the batch reuses the
    ///      unequal-time basket and executes both orders with one parse.
    function test_Control_RepositoryScenario_NewerStoragePresent_BatchExecutesBoth() public {
        (bytes[] memory historical, uint64 firstCommit) = _twoOrdersAndUnequalTimeBasket();
        bytes[] memory newer = _pricedPayload(firstCommit + 6, firstCommit + 5, 102_000_000);
        baseMockPyth.updatePriceFeeds{value: FEE}(newer);

        OrderV3Types.BatchResult memory result = router.executeOrderBatch{value: 2 * FEE}(2, historical);
        assertEq(result.terminalCount, 2);
        assertEq(result.nextOrderId, 0);
        assertEq(baseMockPyth.parseUniqueCallCount(), 1, "second order reused the signed basket");
        assertEq(engine.lastMarkPrice(), 101_000_000);
        console2.log("newer payload already in Pyth storage: batch terminalCount", uint256(result.terminalCount));
    }

    /// @dev Same scenario with the single storage write removed. Nothing else differs.
    function test_RepositoryScenario_WithoutNewerStorage_WholeBatchReverts() public {
        (bytes[] memory historical, uint64 firstCommit) = _twoOrdersAndUnequalTimeBasket();

        bytes32 olderFeed = _synchronizationFeedIds()[0];
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector,
                olderFeed,
                uint256(firstCommit + 4),
                uint256(firstCommit + 5)
            )
        );
        router.executeOrderBatch{value: 2 * FEE}(2, historical);
        assertEq(router.nextExecuteId(), 1, "no order executed, including the first");
        (uint256 aliceSize,,,,,,) = engine.positions(ALICE);
        assertEq(aliceSize, 0, "first order's execution rolled back");
        console2.log("no newer payload in Pyth storage: batch reverted StoredFeedBehind; queue head still", uint256(1));

        // Each order executes on its own with the very same payload.
        OrderV3Types.ExecutionResult memory first = router.executeOrder{value: 2 * FEE}(1, historical);
        OrderV3Types.ExecutionResult memory second = router.executeOrder{value: 2 * FEE}(2, historical);
        assertEq(uint8(first.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(uint8(second.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        console2.log("single executions with the same payload (2 = Executed):", uint8(first.status), uint8(second.status));
    }

    /// @dev ALICE commits at t, BOB at t+3. The basket has one component at t+4 and the rest at t+5, so every
    ///      component follows both commits.
    function _twoOrdersAndUnequalTimeBasket() private returns (bytes[] memory historical, uint64 firstCommit) {
        firstCommit = uint64(block.timestamp);
        _commit(ALICE, 0);
        _advance(firstCommit + 3);
        _commit(BOB, 0);
        _advance(firstCommit + 6);

        historical = _pricedPayload(firstCommit + 5, firstCommit, 101_000_000);
        MockPyth.FeedUpdate[] memory updates = abi.decode(historical[0], (MockPyth.FeedUpdate[]));
        updates[0].price.publishTime = firstCommit + 4;
        historical[0] = abi.encode(updates);
    }

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

}
```

## Test file 2: BatchCacheCoverage.t.sol
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Batch historical-cache reuse reverts the whole batch when the cached basket has unequal component
///         publish times, even though each order is individually executable at the same unique basket.
contract BatchCacheCoverageTest is BasePerpTest {

    address internal constant KEEPER = address(0xBEEF01);
    address internal constant TRADER_A = address(0xA11CE);
    address internal constant TRADER_B = address(0xB0B0B);

    uint256 internal constant LIVE_TIME = 1_793_725_200; // Tue 2026-11-03 17:00 UTC (live FX session).
    uint256 internal constant PRICE = 1e8;
    uint256 internal constant SIZE = 100_000e18; // 100k tokens ~ 100k USDG notional.
    uint256 internal constant MARGIN = 5_000e6;

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

    function setUp() public override {
        super.setUp();
        PletherOracle six = new PletherOracle(
            address(engine), address(pool), address(baseMockPyth), _feedIds(), _weights(), _basePrices(), _inversions()
        );
        routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig({pletherOracle: address(six)}));
        vm.warp(routerAdmin.oracleConfigActivationTime());
        routerAdmin.finalizeOracleConfig();
        pletherOracle = six;
        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.basketMaxConfidenceRatioBps = 10;
        config.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(config);

        _fundTrader(TRADER_A, 50_000e6);
        _fundTrader(TRADER_B, 50_000e6);
        vm.warp(LIVE_TIME);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory initialMark = _equalTick(PRICE, uint64(vm.getBlockTimestamp()), uint64(vm.getBlockTimestamp() - 1));
        vm.prank(KEEPER);
        router.updateMarkPrice(initialMark);
        assertEq(engine.executionFeeBps(), 4);
        assertFalse(engine.isFadWindow());
        assertFalse(engine.isOracleFrozen());
    }

    /// @dev Two opens committed in the same block share commitTime T. The unique post-T basket has five components
    ///      at T+1 and one at T+2 (a realistic one-second publish skew, well inside the 5 s divergence limit).
    function test_BatchRevertsWhileSingleExecutionsSucceed() public {
        vm.warp(vm.getBlockTimestamp() + 10);
        vm.roll(vm.getBlockNumber() + 1);
        uint64 commitTime = uint64(vm.getBlockTimestamp());
        uint64 idA = _commitOpen(TRADER_A, CfdTypes.Side.LONG);
        uint64 idB = _commitOpen(TRADER_B, CfdTypes.Side.SHORT);
        assertEq(idB, idA + 1);
        OrderV3Types.PendingIntent memory pa = router.lifecycleBook().pendingIntent(idA);
        assertEq(pa.timing.commitTimestamp, commitTime);

        bytes[] memory skewed = _skewedTick(PRICE, commitTime);
        bytes[] memory equal = _equalTick(PRICE, commitTime + 1, commitTime);
        vm.warp(commitTime + 3);
        vm.roll(vm.getBlockNumber() + 1);
        uint256 fee = pletherOracle.getOrderExecutionFee(skewed);
        vm.deal(KEEPER, 1 ether);

        uint256 snap = vm.snapshotState();

        // CONTROL 1: single executions with the SAME skewed payload both execute at the unique basket.
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory ra = router.executeOrder{value: fee}(idA, skewed);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory rb = router.executeOrder{value: fee}(idB, skewed);
        console2.log("control single A status (2=Executed)", uint256(ra.status));
        console2.log("control single B status (2=Executed)", uint256(rb.status));
        assertEq(uint8(ra.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(uint8(rb.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        vm.revertToState(snap);

        // CONTROL 2: batch with an equal-time payload executes both (cache reused for B).
        uint256 snap2 = vm.snapshotState();
        vm.prank(KEEPER);
        OrderV3Types.BatchResult memory okBatch = router.executeOrderBatch{value: 2 * fee}(idB, equal);
        console2.log("control equal-time batch terminalCount", uint256(okBatch.terminalCount));
        assertEq(okBatch.terminalCount, 2);
        vm.revertToState(snap2);

        // TEST: batch with the one-second-skewed payload reverts entirely; A's finalized work is rolled back too.
        bytes32 laggingFeed = _feedIds()[0];
        uint256 keeperEthBefore = KEEPER.balance;
        uint256 gasBefore = gasleft();
        vm.prank(KEEPER);
        (bool ok, bytes memory ret) =
            address(router).call{value: 2 * fee}(abi.encodeCall(router.executeOrderBatch, (idB, skewed)));
        uint256 gasUsed = gasBefore - gasleft();
        assertFalse(ok, "skewed batch must revert");
        assertEq(
            ret,
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector,
                laggingFeed,
                uint256(commitTime + 1),
                uint256(commitTime + 2)
            ),
            "revert reason is the reuse coverage check"
        );
        assertEq(KEEPER.balance, keeperEthBefore, "ETH returned by the revert");
        assertEq(router.nextExecuteId(), idA, "nothing finalized: head unchanged after batch revert");
        (uint256 sizeA,,,,,,) = engine.positions(TRADER_A);
        assertEq(sizeA, 0, "trader A's execution was rolled back with the batch");
        console2.log("batch with 1s component skew: reverted with PletherOracle__StoredFeedBehind; head still", idA);
        console2.log("gas burned by the reverted batch call", gasUsed);

        // The same batch keeps reverting for the whole execution window: the payload is fixed by the unique
        // first-tick rule, so a retry cannot pick different component times.
        vm.warp(commitTime + 30);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(KEEPER);
        (ok,) = address(router).call{value: 2 * fee}(abi.encodeCall(router.executeOrderBatch, (idB, skewed)));
        assertFalse(ok, "retry 27 seconds later reverts the same way");
        assertEq(router.nextExecuteId(), idA, "still no progress");
        console2.log("retry at commit+30s: reverted again; head still", idA);
    }

    function _commitOpen(address account, CfdTypes.Side side) internal returns (uint64 id) {
        uint256 target = side == CfdTypes.Side.LONG ? 1 : type(uint256).max;
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(router.nextCommitId())),
            side: side,
            sizeDelta: SIZE,
            marginDelta: MARGIN,
            targetPrice: target,
            isClose: false,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp + 60),
                executionWindowSeconds: 60,
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
        vm.prank(account);
        id = router.commitOrder(request);
    }

    function _equalTick(uint256 basketPrice, uint64 publishTime, uint64 prevPublishTime)
        internal pure returns (bytes[] memory data)
    {
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            ticks[i] = _feedUpdate(i, basketPrice, publishTime, prevPublishTime);
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    /// @dev Feeds 0..4 first publish at commit+1; feed 5 (lowest weight) first publishes at commit+2.
    function _skewedTick(uint256 basketPrice, uint64 commitTime) internal pure returns (bytes[] memory data) {
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 5; ++i) {
            ticks[i] = _feedUpdate(i, basketPrice, commitTime + 1, commitTime);
        }
        ticks[5] = _feedUpdate(5, basketPrice, commitTime + 2, commitTime - 1);
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    function _feedUpdate(uint256 i, uint256 basketPrice, uint64 publishTime, uint64 prevPublishTime)
        internal pure returns (MockPyth.FeedUpdate memory)
    {
        bool[] memory inverted = _inversions();
        uint256 base = _rawPrice(i);
        uint256 raw = inverted[i] ? base * 1e8 / basketPrice : base * basketPrice / 1e8;
        uint64 conf = uint64(raw / 10_000); // 1 bp per component
        return MockPyth.FeedUpdate(
            _feedIds()[i], MockPyth.MockPrice(int64(uint64(raw)), conf, int32(-8), publishTime, prevPublishTime)
        );
    }

    function _feedIds() internal pure returns (bytes32[] memory ids) {
        ids = new bytes32[](6);
        ids[0] = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
        ids[1] = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
        ids[2] = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
        ids[3] = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
        ids[4] = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
        ids[5] = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;
    }

    function _weights() internal pure returns (uint256[] memory w) {
        w = new uint256[](6);
        w[0] = 576e15;
        w[1] = 136e15;
        w[2] = 119e15;
        w[3] = 91e15;
        w[4] = 42e15;
        w[5] = 36e15;
    }

    function _basePrices() internal pure returns (uint256[] memory x) {
        x = new uint256[](6);
        x[0] = 117_500_000;
        x[1] = 638_000;
        x[2] = 134_480_000;
        x[3] = 72_880_000;
        x[4] = 10_860_000;
        x[5] = 126_100_000;
    }

    function _inversions() internal pure returns (bool[] memory inv) {
        inv = new bool[](6);
        inv[1] = true;
        inv[3] = true;
        inv[4] = true;
        inv[5] = true;
    }

    function _rawPrice(uint256 i) internal pure returns (uint256) {
        if (i == 0) return 117_500_000;
        if (i == 1) return uint256(1e16) / 638_000;
        if (i == 2) return 134_480_000;
        if (i == 3) return uint256(1e16) / 72_880_000;
        if (i == 4) return uint256(1e16) / 10_860_000;
        return uint256(1e16) / 126_100_000;
    }
}
```
