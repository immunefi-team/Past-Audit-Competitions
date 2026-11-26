# \#97766 [SC-Low] Protection create and replace revert after any live order fills until the next mark refresh, because the MED-28 fix rejects a state its own guard accepts

**Submitted on Nov 3rd 2026 at 02:35:08 UTC by @Tradi3 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97766
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

## Brief/Intro

After any live order execution, `createPositionProtection`, `replacePositionProtection` and `commitOpenOrderWithProtection` revert with `PletherOracle__PriceOutOfOrder` until some account calls `OrderRouter.updateMarkPrice`. A trader whose open has just filled cannot add a stop-loss during that window. PR #120 added a branch that accepts this exact state, and PR #124 then placed a call ahead of it that reverts first, so the branch can no longer be reached on this path. No funds are lost, and any caller can clear the state by refreshing the mark.

**Target:** `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`: `packages/perps/src/PositionProtectionBook.sol`, `packages/perps/src/PletherOracle.sol`

## Vulnerability Details

Historical order execution moves the Engine mark forward without moving the accepted live basket. `_resolveHistoricalOrderBasket` parses the post-commit tick and never writes `_acceptedBasket` (`PletherOracle.sol:596-643`), while the Router advances `lastMarkTime`. `ORACLE_SYNCHRONIZATION.md:150-151` states this: "historical execution may advance the Engine without advancing the accepted live basket".

`_acceptedMarkInstalled()` (`PositionProtectionBook.sol:888-895`) was written for that state and returns true when `acceptedTime < markTime`.

`_freshProtectionObservation()` now reaches it too late:

1. Line 880 calls `oracle.getLatestPrice(PriceMode.MarkRefresh)`.
2. That view builds its snapshot with publish-order enforcement (`PletherOracle.sol:578`, `:892-896`) and reverts `PriceOutOfOrder` when the accepted basket is older than `engine.lastMarkTime()`.
3. Line 883 checks `_acceptedMarkInstalled()`, which is never reached in that state.

So on this path the `acceptedTime < markTime` and `!accepted` branches are dead. The three callers are `PositionProtectionBook.sol:229` (create), `:295` (replace) and `:463` (attached open).

#124 fixed Cecuro MED-28. Its disposition reads: "The integrated synchronization guard also rejects arming when the accepted oracle mark is ahead of, or inconsistent with, the Engine mark." An accepted mark that is behind the Engine mark is neither, and `_acceptedMarkInstalled()` accepts it. The `getLatestPrice` call added by the same fix rejects it anyway.

The parent-fill path at `:598` calls `_acceptedMarkInstalled()` directly, so an attached open still arms when it fills. A trader can therefore hold an armed protection that cannot be edited right after that same fill.

`triggerPositionProtection` is not affected, because the trigger path carries its own price update.

## Impact Details

Every live fill creates this state, so it follows the most common trader action: open, then set a stop-loss. The call reverts until a mark refresh lands. During that window a trader cannot add or move a stop-loss or take-profit, and cannot commit an open with protection attached.

Limits:

- No user funds are lost by the revert itself, and positions and reserves are untouched.
- Any account can call `updateMarkPrice` with signed Pyth data, and the existing reference keepers refresh the mark. Create succeeds right after that.
- How often this hits depends on how often the mark is refreshed. Measured on the v1.2.3 trading competition on Arbitrum Sepolia (13-25 September 2026, 16,566 orders), from public events: the dedicated mark updater ran about every 33 s (median interarrival 33 s, p99 64 s). At least 112 of the 520 successful protection creates and replaces (21.5%) were sent after an order execution and before the next accepted mark update. At the pin those calls revert. This is a lower bound, because the measurement also counted order receipts as mark updates, and at the pin those do not move the accepted basket.
- It affects live and FAD execution, where historical parsing runs. Protection creation is rejected in frozen hours anyway.

## Recommended fix

In `_freshProtectionObservation()`, read the accepted basket without publish-order enforcement, and when `acceptedTime < markTime` use the cached Engine mark and `markTime` as the current observation. That keeps the check PR #124 added against crossing prices when the accepted basket is current, and it restores the state PR #120 allows. Alternatively, move the `_acceptedMarkInstalled()` check ahead of the oracle read and branch on it.

Add a test that calls `createPositionProtection` straight after a historical execution with no mark refresh. The existing protection tests refresh the mark first, which is why this path passes in the suite.

## References

- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L862-L895
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L571-L580
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L878-L903
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md#L150-L151


## Proof of Concept

## Proof of Concept

This Insight is valuable whenever a trader sets or moves a stop-loss or take-profit between an order fill and the next mark refresh, which on the measured testnet history was at least one call in five.

Add the file below as `packages/perps/test/perps/poc/ProtectionAfterFill.t.sol`. No `src` changes. Run from `packages/perps` on Forge 1.5.1:

```bash
FOUNDRY_PROFILE=quick forge test --offline --match-path test/perps/poc/ProtectionAfterFill.t.sol -vv
```

It uses release parameters and the repo fixture. Only Pyth and the settlement token are mocks. From one snapshot, the first arm calls create and an attached open straight after a live fill, and both revert with `PriceOutOfOrder`. The control refreshes the mark and both succeed. The test also shows an attached open that arms on fill and then cannot be edited.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdClosePreview} from "@plether/perps/CfdClosePreview.sol";
import {CfdEngineAdmin} from "@plether/perps/CfdEngineAdmin.sol";
import {CfdEnginePlanTypes} from "@plether/perps/CfdEnginePlanTypes.sol";
import {CfdMath} from "@plether/perps/CfdMath.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {OrderRouterAdmin} from "@plether/perps/OrderRouterAdmin.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {PositionProtectionBook} from "@plether/perps/PositionProtectionBook.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {AccountLensViewTypes} from "@plether/perps/interfaces/AccountLensViewTypes.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {ICfdOrderPolicyEvaluator} from "@plether/perps/interfaces/ICfdOrderPolicyEvaluator.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionBook} from "@plether/perps/interfaces/IPositionProtectionBook.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PerpsViewTypes} from "@plether/perps/interfaces/PerpsViewTypes.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MarketCalendarLib} from "@plether/perps/libraries/MarketCalendarLib.sol";
import {SolvencyAccountingLib} from "@plether/perps/libraries/SolvencyAccountingLib.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {StdStorage, stdStorage} from "forge-std/StdStorage.sol";

contract ProtectionAfterFillTest is BasePerpTest {

    using stdStorage for StdStorage;

    IPositionProtectionBook internal protectionBook;
    IPositionProtectionActions internal protectionActions;
    IPositionProtectionViews internal protectionViews;
    CfdClosePreview internal closePreviewContract;

    address internal constant ALICE = address(0xA11CE);
    address internal constant BOB = address(0xB0B);
    address internal constant CAROL = address(0xCA201);
    address internal constant KEEPER = address(0xBEEF);
    address internal constant LP_SENIOR = address(0x5E10);
    address internal constant LP_JUNIOR = address(0x4010);

    // Friday 2024-03-08 21:30:00 UTC (16:30 EST, FAD start) and 22:00:00 UTC (17:00 EST, Frozen start)
    uint256 internal constant FRIDAY_FAD_START = 1_709_933_400;
    uint256 internal constant FRIDAY_FROZEN_START = 1_709_935_200;
    // Sunday 2024-03-10 21:00:00 UTC (17:00 EDT after US DST start at 07:00 UTC)
    uint256 internal constant SUNDAY_FROZEN_END_DST = 1_710_104_400;
    // Friday 2024-03-22 21:00:00 UTC (17:00 EDT, Frozen start after setUp timelocks)
    uint256 internal constant LIVE_FRIDAY_FROZEN_START = 1_711_141_200;
    /// @dev Override BasePerpTest to use exact DEPLOYMENT.md initial release parameters.
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
        return 400_000e6;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 600_000e6;
    }

    function setUp() public override {
        super.setUp();

        // Set release HousePool Senior caps: 40M USDG and 8000 bps (80%).
        _setSeniorCapacity(40_000_000e6, 8000);

        // Set release Router & Oracle parameters: minOpenNotionalAssets = 1000e6, adverseConfidenceMultiplierBps = 2500.
        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.minOpenNotionalAssets = 1000e6;
        rc.adverseConfidenceMultiplierBps = 2500;
        rc.basketMaxConfidenceRatioBps = 10;
        rc.maxPendingOrders = 5;
        rc.positionProtectionTriggerBountyAssets = 200_000;
        _setRouterConfig(rc);

        // Ensure executionFeeBps = 4, frozenCloseSpreadBps = 50, settlementBufferBps = 25.
        ICfdEngineAdminHost.EngineRiskConfig memory ec = ICfdEngineAdminHost.EngineRiskConfig({
            riskParams: _riskParams(),
            executionFeeBps: 4,
            frozenCloseSpreadBps: 50,
            settlementBufferBps: 25
        });
        engineAdmin.proposeRiskConfig(ec);
        uint256 t = engineAdmin.riskConfigActivationTime();
        vm.warp(t);
        assertEq(vm.getBlockTimestamp(), t);
        engineAdmin.finalizeRiskConfig();

        // Warp forward (never backward!) to a clean Monday live window after all timelocks: Monday 2024-03-18 10:00:00 UTC.
        uint256 startTs = 1_710_756_000;
        assertGt(startTs, vm.getBlockTimestamp());
        vm.warp(startTs);
        assertEq(vm.getBlockTimestamp(), startTs);
        protectionBook = router.positionProtectionBook();
        protectionActions = IPositionProtectionActions(address(protectionBook));
        protectionViews = IPositionProtectionViews(address(protectionBook));
        closePreviewContract = new CfdClosePreview();

        _refreshLiveMark(1e8, 0);

        _fundTrader(ALICE, 100_000e6);
        _fundTrader(BOB, 100_000e6);
        _fundTrader(CAROL, 100_000e6);
    }

    // =========================================================================
    // HELPERS
    // =========================================================================

    function _refreshLiveMark(
        uint256 price,
        uint64 conf
    ) internal {
        uint256 t = vm.getBlockTimestamp();
        if (t <= engine.lastMarkTime()) {
            t = uint256(engine.lastMarkTime()) + 1;
            vm.warp(t);
            assertEq(vm.getBlockTimestamp(), t);
        }
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint64 prev = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] =
                MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), conf, -8, uint64(t), prev));
        }
        bytes[] memory updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
        router.updateMarkPrice(updateData);
    }

    function _historicalUpdateDataAt(
        uint256 price,
        uint64 conf,
        uint64 publishTime,
        uint64 prevPublishTime
    ) internal returns (bytes[] memory updateData) {
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), int64(uint64(price)), conf, int32(-8), publishTime, prevPublishTime
        );
        updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
    }

    function _defaultBounds(
        bool isClose
    ) internal view returns (OrderV3Types.ExecutionBounds memory bounds) {
        bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
            executionWindowSeconds: 60,
            allowedExecutionModes: 1 | 2 | 4,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: isClose ? 0 : type(uint256).max,
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

    function _makeOrderRequest(
        bytes32 clientOrderId,
        CfdTypes.Side side,
        uint256 sizeDelta,
        uint256 marginDelta,
        uint256 targetPrice,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory req) {
        req = OrderV3Types.OrderRequest({
            clientOrderId: clientOrderId,
            side: side,
            sizeDelta: sizeDelta,
            marginDelta: isClose ? 0 : marginDelta,
            targetPrice: targetPrice,
            isClose: isClose,
            bounds: _defaultBounds(isClose)
        });
    }

    function _protectionParams(
        uint256 tp,
        uint256 sl
    ) internal pure returns (PositionProtectionTypes.PositionProtectionParams memory params) {
        params.takeProfitTriggerPrice = tp;
        params.stopLossTriggerPrice = sl;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    // =========================================================================
    // LEAD 1: PositionProtectionBook._freshProtectionObservation() reverts with
    // PletherOracle__PriceOutOfOrder after any historical order execution
    // =========================================================================

    function test_ProtectionRevertsAfterLiveOrderExecution() public {
        uint256 t = vm.getBlockTimestamp();
        uint64 initialAcceptedTime = engine.lastMarkTime();
        assertEq(initialAcceptedTime, uint64(t));

        // Step 1: Alice commits and executes a normal opening order via OrderRouter (as documented in trader-quickstart.md).
        OrderV3Types.OrderRequest memory openReq =
            _makeOrderRequest(keccak256("alice-open-1"), CfdTypes.Side.LONG, 10_000e18, 2000e6, 1, false);
        vm.prank(ALICE);
        uint64 openOrderId = router.commitOrder(openReq);
        t += 2;
        vm.warp(t);
        assertEq(vm.getBlockTimestamp(), t);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory execData = _historicalUpdateDataAt(1e8, 0, uint64(t - 1), initialAcceptedTime);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory execRes = router.executeOrder(openOrderId, execData);
        assertEq(uint8(execRes.status), uint8(OrderV3Types.LifecycleStatus.Executed));

        // Historical order execution advanced engine.lastMarkTime() to t - 1, while PletherOracle's _acceptedBasket is still at initialAcceptedTime.
        (bool accepted,, uint64 acceptedTime) = pletherOracle.getAcceptedBasketMark();
        assertTrue(accepted);
        assertEq(acceptedTime, initialAcceptedTime);
        assertEq(engine.lastMarkTime(), uint64(t - 1));
        assertLt(acceptedTime, engine.lastMarkTime());

        uint256 snap = vm.snapshotState();

        // Arm A (Baseline after order execution): Alice tries to add TP/SL to her new position (createPositionProtection),
        // or Bob tries to commit an open order with attached TP/SL (commitOpenOrderWithProtection).
        // Both revert with PletherOracle__PriceOutOfOrder(acceptedTime, markTime).
        uint64 markTime = engine.lastMarkTime();
        vm.prank(ALICE);
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, markTime
            )
        );
        protectionActions.createPositionProtection(_protectionParams(90_000_000, 110_000_000));

        OrderV3Types.OrderRequest memory bobOpenReq =
            _makeOrderRequest(keccak256("bob-attached-open"), CfdTypes.Side.SHORT, 10_000e18, 2000e6, 2e8, false);
        vm.prank(BOB);
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, markTime
            )
        );
        protectionActions.commitOpenOrderWithProtection(bobOpenReq, _protectionParams(110_000_000, 90_000_000));

        // Arm B (Control from the exact same snapshot: synchronize _acceptedBasket via updateMarkPrice):
        vm.revertToState(snap);
        _refreshLiveMark(1e8, 0);
        (, , uint64 syncedAcceptedTime) = pletherOracle.getAcceptedBasketMark();
        assertEq(syncedAcceptedTime, engine.lastMarkTime());

        vm.prank(ALICE);
        uint64 aliceProtId = protectionActions.createPositionProtection(_protectionParams(90_000_000, 110_000_000));
        assertEq(
            uint8(protectionViews.getPositionProtection(aliceProtId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Armed)
        );

        // Reconstruct bobOpenReq after _refreshLiveMark(1e8, 0) so submitBy uses the updated timestamp:
        bobOpenReq = _makeOrderRequest(keccak256("bob-attached-open"), CfdTypes.Side.SHORT, 10_000e18, 2000e6, 2e8, false);
        vm.prank(BOB);
        (uint64 bobParentId, uint64 bobProtId) =
            protectionActions.commitOpenOrderWithProtection(bobOpenReq, _protectionParams(110_000_000, 90_000_000));
        assertEq(
            uint8(protectionViews.getPositionProtection(bobProtId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.PendingOpen)
        );

        // Execute Bob's attached open: afterOrderTerminal arms Bob's protection (because _acceptedMarkInstalled() allows acceptedTime < markTime),
        // but Bob immediately cannot edit his armed TP/SL (replacePositionProtection reverts with PletherOracle__PriceOutOfOrder)!
        t = vm.getBlockTimestamp() + 2;
        vm.warp(t);
        assertEq(vm.getBlockTimestamp(), t);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory bobExecData = _historicalUpdateDataAt(1e8, 0, uint64(t - 1), syncedAcceptedTime);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory bobRes = router.executeOrder(bobParentId, bobExecData);
        assertEq(uint8(bobRes.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(
            uint8(protectionViews.getPositionProtection(bobProtId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Armed)
        );

        uint64 bobMarkTime = engine.lastMarkTime();
        vm.prank(BOB);
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__PriceOutOfOrder.selector, syncedAcceptedTime, bobMarkTime
            )
        );
        protectionActions.replacePositionProtection(bobProtId, _protectionParams(115_000_000, 85_000_000));
    }

    // =========================================================================
    // LEAD 2: Open / Increase Resulting Position Margin Carve-Out, VPI Rebate
    // Routing, and Margin Call Simulator Entry Cap
    // =========================================================================

}
```

Output on a clean clone at the pin, Forge 1.5.1:

```text
[PASS] test_ProtectionRevertsAfterLiveOrderExecution() (gas: 7190355)
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

