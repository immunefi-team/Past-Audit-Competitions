# \#99341 [SC-Medium] Batch execution reverts on a valid basket whose components were published at different seconds

**Submitted on Nov 13th 2026 at 09:22:50 UTC by @krlz for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99341
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

## Brief/Intro

When a later order in `executeOrderBatch` reuses the basket parsed for an earlier order, `PletherOracle` checks every stored Pyth feed against the basket's newest publish time. Each feed was stored at its own publish time, so a basket whose components were published at different seconds always fails the check and the whole batch reverts. The same orders execute one by one with the same payload.

## Vulnerability Details

Within a batch, a later FIFO order may reuse the basket parsed for an earlier order when it was committed before every component of that basket. Before the reuse, the oracle proves that Pyth storage covers the cached basket, but against a different time than the parse that wrote that storage:

```solidity
// Reuse path (PletherOracle.sol#L264-L266): every stored feed against the newest component time
for (uint256 i; i < pythFeedIds.length; ++i) {
    _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
}

// Fresh path that wrote those feeds (PletherOracle.sol#L624-L626): each feed against its own time
for (uint256 i; i < parsedFeeds.length; ++i) {
    _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);
}

// Basket times (PletherOracle.sol#L874-L875)
basket.publishTime = uint64(maxPublishTime);
basket.oldestPublishTime = uint64(minPublishTime);
```

If one component is a second older than the rest, its stored feed sits below `cache.publishTime` and the reuse reverts with `PletherOracle__StoredFeedBehind`. The Router's batch catches only `PletherOracle__HistoricalParserFailure` (`OrderRouterExecutionSidecar.sol#L1019-L1027`), so the revert unwinds the whole batch, including the orders already executed. The documentation lists this revert for a stored feed that is behind (`ORACLE_SYNCHRONIZATION.md#L109`), and here no feed is behind: storage holds exactly the feeds the batch has just written. This is the reuse the documentation describes (`ORACLE_SYNCHRONIZATION.md#L59-L63`), refused only because of the reference time.

The check was correct when it was added. Commit 64c6ea90 (PR #103) introduced it while `publishTime` was the oldest component time. Commit 81919218 (PR #120) made `publishTime` the newest component time and added `oldestPublishTime`, but left this comparison on `publishTime`. The repository's test of this case, `test_BatchReusesBasketWhenEveryComponentFollowsBothCommits`, passes only because it pushes a newer tick into Pyth storage before the batch (`OracleSynchronization.t.sol#L445-L446`).

## Impact Details

No order, position or fund is at risk, since each order still executes with the same payload through `executeOrder`. The keeper loses the gas of the reverted batch, about 2.28M gas in the PoC, and has to fall back to single-order execution for every queue segment whose first post-commit tick has components published at different seconds.

## Recommendation

Compare the stored feeds with the basket's oldest component time, the time the check used when it was added. With this change the PoC batch reuses the basket and executes both orders, and the repository's oracle tests still pass:

```diff
-                _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
+                _requireStoredFeedCoverage(pythFeedIds[i], cache.oldestPublishTime);
```

## Link to Proof of Concept

https://gist.github.com/KirillZiborov/a1ac405ff5762764ed5a0d9a3684f90a

## Proof of Concept

## Proof of Concept

This Insight matters to keepers that execute the queue in batches. It applies whenever a batch holds a later order committed before every component of an earlier order's basket and those components carry different publish times, which the oracle accepts within its 5 s divergence limit.

`testBatchRevertsOnReusedMixedTimeBasket` runs on the repository's oracle-synchronization fixture (six feeds) with the release risk and Router parameters.

1. Two traders commit opens in the same second `t`.
2. The first tick after `t` has one feed published at `t + 1` and the other five at `t + 2`.
3. A keeper calls `executeOrderBatch` over both orders with that tick. The second order reuses the first order's basket, and the call reverts with `PletherOracle__StoredFeedBehind(feed, t + 1, t + 2)`. Neither order executes.
4. The keeper calls `executeOrder` for each order with the same payload, and both execute.

`testControlUniformBasketBatchExecutes` runs the same batch with a basket whose components share one publish time, and the second order reuses it. `testForkBatchRevertsOnReusedMixedTimeBasket` reproduces the revert on the live v2.0.0 deployment on Arbitrum Sepolia at block 325381938 with the live basket prices.

Save the code as `packages/perps/test/perps/PoC.t.sol` and run `forge test --root packages/perps --match-path test/perps/PoC.t.sol -vv`. The fork test reads `ARBITRUM_SEPOLIA_RPC_URL`.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;

import {Test, console} from "forge-std/Test.sol";
import {IERC20} from "forge-std/interfaces/IERC20.sol";
import {OracleSynchronizationTestBase} from "./shared/OracleSynchronizationTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";

/// @notice The live Plether perps v2.0.0 deployment on Arbitrum Sepolia, forked locally at a pinned block.
/// @dev Addresses and runtime code hashes come from the public release manifest
///      (deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0). Every in-scope contract is the deployed one;
///      only the Pyth price-update contract is replaced by the repository's own Pyth test double, so that the test can
///      supply price updates. Nothing is broadcast.
abstract contract LivePerpsFork is Test {

    uint256 internal constant FORK_BLOCK = 325_381_938;

    IERC20 internal constant L_USDG = IERC20(0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D);
    MarginClearinghouse internal constant L_CLEARINGHOUSE =
        MarginClearinghouse(0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A);
    CfdEngine internal constant L_ENGINE = CfdEngine(0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA);
    HousePool internal constant L_POOL = HousePool(0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e);
    PletherOracle internal constant L_ORACLE = PletherOracle(0x4f2c62A0E03657F52C633c19c11f88d8F0897Bb2);
    OrderRouter internal constant L_ROUTER = OrderRouter(0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444);
    address internal constant L_LIFECYCLE_BOOK = 0x327d5fd04e17D96DD5C4F845A0b4256A9e19645C;
    address internal constant L_PROTECTION_BOOK = 0x81E3b03Cf29865b6c0f85925839567e2Bc61073D;
    address internal constant L_PYTH = 0x0B73614636C855Bf23F342F307FB981A3e47f42B;

    /// @dev Arbitrum's L2 block number, which the Router reads from the ArbSys precompile for its same-block rule.
    uint256 internal arbBlock;

    bytes32[] internal liveFeedIds;
    int64[] internal livePrices;
    uint64[] internal liveConfs;
    int32[] internal liveExpos;

    /// @dev Forks the live deployment, checks that the deployed code is the released (audited) build, records the
    ///      live Pyth prices of the six basket feeds, then installs the Pyth test double at the Pyth address.
    function _selectLiveFork() internal {
        vm.createSelectFork(vm.envString("ARBITRUM_SEPOLIA_RPC_URL"), FORK_BLOCK);
        assertEq(block.chainid, 421_614, "Arbitrum Sepolia");
        assertEq(address(L_USDG).codehash, 0x12233a5f6b0abf6a0f8447e56f9507afdd142a78cc42ebb0d041a3997bd299f6);
        assertEq(address(L_CLEARINGHOUSE).codehash, 0xc8e4965893dbd7073303303e9d5ab9c775aabcc1259635e4993fc7fdb0681398);
        assertEq(address(L_ENGINE).codehash, 0xa0f8e5be6e622a5a83e46e88f6b012757ef924cf613a50cba52a2b5c66b76a6b);
        assertEq(address(L_POOL).codehash, 0x5e684cc82bfc5b697a24c80ef26910cfbe25efbe6fb2ca184c421808b5ced957);
        assertEq(address(L_ORACLE).codehash, 0x1ecb1ca760e7ba51d958697eeefe06ff7ba1a3ad162b3e7978af3df30393bc13);
        assertEq(address(L_ROUTER).codehash, 0x6c65de67ef2824572537e3641a71100100c49d909b196c43e5d271312534d3cf);
        assertEq(L_LIFECYCLE_BOOK.codehash, 0xc3f34d5c32716e1818e1ae1cf5e14892a639bc35f15894610e619afe648ba554);
        assertEq(L_PROTECTION_BOOK.codehash, 0xe15987a99e503d6bbd5d871090b9bfea06383bb37fa7af8dfd9cf457fdc4eed5);
        assertEq(
            L_ROUTER.executionSidecar().codehash, 0xfc72fcbe4496d0d5356e96d259536cd4edcb926dd7fe1473a4275ddfb09cb57e
        );
        assertEq(
            L_ROUTER.liquidationBatchSidecar().codehash,
            0x8420be926293c70a880b5f12c07674861563727a04aca954323004855807c8f7
        );
        assertEq(address(L_ROUTER.pletherOracle()), address(L_ORACLE), "router oracle");
        assertEq(address(L_ORACLE.pyth()), L_PYTH, "oracle pyth");

        for (uint256 i; i < 6; ++i) {
            bytes32 id = L_ORACLE.pythFeedIds(i);
            (bool ok, bytes memory ret) = L_PYTH.staticcall(abi.encodeWithSignature("getPriceUnsafe(bytes32)", id));
            require(ok, "live pyth read");
            (int64 price, uint64 conf, int32 expo,) = abi.decode(ret, (int64, uint64, int32, uint256));
            liveFeedIds.push(id);
            livePrices.push(price);
            liveConfs.push(conf);
            liveExpos.push(expo);
        }

        MockPyth double = new MockPyth();
        vm.etch(L_PYTH, address(double).code);
        // The double's own scalar state lives in slots 0..11; clear what the real Pyth proxy kept there.
        for (uint256 slot; slot < 12; ++slot) {
            vm.store(L_PYTH, bytes32(slot), bytes32(0));
        }
        for (uint256 i; i < 6; ++i) {
            MockPyth(L_PYTH).setPrice(liveFeedIds[i], livePrices[i], liveConfs[i], liveExpos[i], vm.getBlockTimestamp() - 60);
        }
        arbBlock = FORK_BLOCK;
        _mockArbBlock();
    }

    /// @dev The local fork has no ArbSys precompile behaviour that advances with the test's blocks; serve the L2 block
    ///      number that the test advances, one per step.
    function _mockArbBlock() internal {
        vm.mockCall(address(100), abi.encodeWithSignature("arbBlockNumber()"), abi.encode(arbBlock));
    }

    /// @dev A Pyth update carrying the live prices of the six basket feeds at the given publish time.
    function _feedUpdates(
        uint64 publishTime,
        uint64 prevPublishTime
    ) internal view returns (MockPyth.FeedUpdate[] memory updates) {
        updates = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            updates[i] = MockPyth.FeedUpdate(
                liveFeedIds[i],
                MockPyth.MockPrice(livePrices[i], liveConfs[i], liveExpos[i], publishTime, prevPublishTime)
            );
        }
    }

    function _livePayload(
        uint64 publishTime,
        uint64 prevPublishTime
    ) internal view returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = abi.encode(_feedUpdates(publishTime, prevPublishTime));
    }

    /// @dev Refreshes the live engine mark at the current second, as the oracle worker or any caller does.
    function _liveRefresh() internal {
        L_ROUTER.updateMarkPrice(_livePayload(uint64(vm.getBlockTimestamp()), uint64(vm.getBlockTimestamp() - 1)));
    }

    function _liveFund(
        address account,
        uint256 amount
    ) internal {
        deal(address(L_USDG), account, amount);
        vm.startPrank(account);
        L_USDG.approve(address(L_CLEARINGHOUSE), amount);
        L_CLEARINGHOUSE.deposit(account, amount);
        vm.stopPrank();
    }

    /// @dev A V3 open request with unbounded execution limits.
    function _liveOpenRequest(
        bytes32 clientOrderId,
        CfdTypes.Side side,
        uint256 sizeDelta,
        uint256 marginDelta
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = clientOrderId;
        request.side = side;
        request.sizeDelta = sizeDelta;
        request.marginDelta = marginDelta;
        request.targetPrice = side == CfdTypes.Side.LONG ? 1 : type(uint256).max;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
            executionWindowSeconds: uint32(L_ROUTER.maxExecutionWindowSeconds()),
            allowedExecutionModes: 7,
            expectedConfigHash: IOrderLifecycleBook(L_LIFECYCLE_BOOK).currentExecutionConfigHash(),
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
    }

    function _liveCommitOpen(
        address account,
        bytes32 clientOrderId,
        CfdTypes.Side side,
        uint256 sizeDelta,
        uint256 marginDelta
    ) internal returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request = _liveOpenRequest(clientOrderId, side, sizeDelta, marginDelta);
        vm.prank(account);
        orderId = L_ROUTER.commitOrder(request);
    }

    function _step(
        uint256 seconds_
    ) internal {
        vm.warp(vm.getBlockTimestamp() + seconds_);
        vm.roll(vm.getBlockNumber() + 1);
        ++arbBlock;
        _mockArbBlock();
    }

}

contract BatchReuseMixedTimesLocalPoC is OracleSynchronizationTestBase {

    address internal constant KEEPER = address(0xCEE9);

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

    /// @dev The fixture's setUp is final; the release policy is applied at the start of each test.
    function _releasePolicy() internal {
        // Release router/oracle policy (DeployPerpsArbitrumSepolia: adverse multiplier 2,500, minimum open 1,000 USDG).
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500;
        config.minOpenNotionalAssets = 1000e6;
        _setRouterConfig(config);
        // The governance delays above end on a Sunday (oracle-frozen weekend); trade on the following Monday.
        vm.warp(vm.getBlockTimestamp() + 1 days);
        vm.roll(vm.getBlockNumber() + 1);
        assertFalse(engine.isFadWindow() || engine.isOracleFrozen(), "live market");
        vm.deal(KEEPER, 1 ether);
        vm.label(KEEPER, "keeper");
    }

    /// Two orders committed in the same second. The first post-commit tick has one component published one second
    /// before the other five, inside the 5 s divergence limit.
    function _mixedTimeBasket(
        uint64 commit
    ) internal pure returns (bytes[] memory data) {
        data = _payload(commit + 2, commit);
        MockPyth.FeedUpdate[] memory updates = abi.decode(data[0], (MockPyth.FeedUpdate[]));
        updates[0].price.publishTime = commit + 1;
        data[0] = abi.encode(updates);
    }

    function testBatchRevertsOnReusedMixedTimeBasket() public {
        _releasePolicy();
        uint64 commit = uint64(vm.getBlockTimestamp());
        uint64 first = _commit(ALICE, 0);
        uint64 second = _commit(BOB, 0);
        assertEq(router.lifecycleBook().orderTiming(second).commitTimestamp, commit, "same commit second");
        _advance(commit + 3);
        bytes[] memory data = _mixedTimeBasket(commit);

        // The keeper's batch over both orders reverts as a whole on the reused basket.
        uint256 gasBefore = gasleft();
        vm.prank(KEEPER);
        (bool ok, bytes memory reason) =
            address(router).call{value: 2 * FEE}(abi.encodeCall(router.executeOrderBatch, (second, data)));
        uint256 gasBurned = gasBefore - gasleft();
        assertFalse(ok, "batch reverted");
        assertEq(
            reason,
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector, bytes32(uint256(1)), commit + 1, commit + 2
            ),
            "reverted on the stored feed published one second before the basket's newest component"
        );
        assertEq(router.nextExecuteId(), first, "no order progressed, not even the first one");
        console.log("gas burned by the reverted batch", gasBurned);

        // The very same payload executes each order on its own: the basket was valid all along.
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r1 = router.executeOrder{value: 2 * FEE}(first, data);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r2 = router.executeOrder{value: 2 * FEE}(second, data);
        assertEq(uint256(r1.status), uint256(OrderV3Types.LifecycleStatus.Executed), "first executes alone");
        assertEq(uint256(r2.status), uint256(OrderV3Types.LifecycleStatus.Executed), "second executes alone");
    }

    /// Control: a basket whose components share one publish time is reused and the batch executes both orders.
    function testControlUniformBasketBatchExecutes() public {
        _releasePolicy();
        uint64 commit = uint64(vm.getBlockTimestamp());
        _commit(ALICE, 0);
        uint64 second = _commit(BOB, 0);
        _advance(commit + 3);
        vm.prank(KEEPER);
        OrderV3Types.BatchResult memory result =
            router.executeOrderBatch{value: 2 * FEE}(second, _payload(commit + 2, commit));
        assertEq(result.terminalCount, 2, "both executed");
        assertEq(baseMockPyth.parseUniqueCallCount(), 1, "the second order reused the first order's basket");
    }

}

contract BatchReuseMixedTimesForkPoC is LivePerpsFork {

    address internal constant ALICE = address(0xA11CE);
    address internal constant BOB = address(0xB0B);
    address internal constant KEEPER = address(0xCEE9);

    function testForkBatchRevertsOnReusedMixedTimeBasket() public {
        _selectLiveFork();
        assertEq(L_ROUTER.nextExecuteId(), 0, "live queue empty at the pinned block");
        _step(1);
        _liveRefresh();
        _liveFund(ALICE, 10_000e6);
        _liveFund(BOB, 10_000e6);
        _step(1);

        // Two traders commit opens in the same second.
        uint64 commit = uint64(vm.getBlockTimestamp());
        uint256 size = 20 * CfdTypes.SIZE_QUANTUM; // 2,000 tokens, about 1,934 USDG notional at the live mark
        uint64 first = _liveCommitOpen(ALICE, keccak256("alice-1"), CfdTypes.Side.LONG, size, 500e6);
        uint64 second = _liveCommitOpen(BOB, keccak256("bob-1"), CfdTypes.Side.SHORT, size, 500e6);
        _step(3);

        // First post-commit tick of the live basket: EUR published at commit + 1, the other five at commit + 2.
        MockPyth.FeedUpdate[] memory updates = _feedUpdates(commit + 2, commit);
        updates[0].price.publishTime = commit + 1;
        bytes[] memory data = new bytes[](1);
        data[0] = abi.encode(updates);

        assertEq(L_ROUTER.lifecycleBook().orderTiming(second).commitTimestamp, commit, "same commit second");
        vm.prank(KEEPER);
        (bool ok, bytes memory reason) =
            address(L_ROUTER).call(abi.encodeCall(L_ROUTER.executeOrderBatch, (second, data)));
        assertFalse(ok, "live batch reverted");
        assertEq(
            reason,
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector, liveFeedIds[0], commit + 1, commit + 2
            )
        );
        assertEq(L_ROUTER.nextExecuteId(), first, "no live order progressed");

        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r1 = L_ROUTER.executeOrder(first, data);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r2 = L_ROUTER.executeOrder(second, data);
        assertEq(uint256(r1.status), uint256(OrderV3Types.LifecycleStatus.Executed), "first executes alone");
        assertEq(uint256(r2.status), uint256(OrderV3Types.LifecycleStatus.Executed), "second executes alone");
    }

}
```
