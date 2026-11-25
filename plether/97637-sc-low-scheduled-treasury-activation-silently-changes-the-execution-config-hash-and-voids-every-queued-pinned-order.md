# \#97637 [SC-Low] Scheduled treasury activation silently changes the execution-config hash and voids every queued pinned order

**Submitted on Nov 2nd 2026 at 15:04:54 UTC by @W3B_SR_0xki for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97637
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description


**Contracts:** `CfdTreasury.sol` (inherited by `CfdEngine`), `OrderLifecycleBook.sol`, `OrderRecoverySidecar.sol` (via `OrderRouter.clearInvalidOrders`)
**Trigger:** a legitimate, documented owner action (`proposeProtocolTreasury`); no further transaction is needed

## Summary

`OrderLifecycleBook.currentExecutionConfigHash()` hashes the *timestamp-resolved* treasury address, `engine.protocolTreasury()`. Since #130, `CfdTreasury` switches that address to a scheduled successor automatically at `lockedUntil`, with no transaction. The execution-config hash therefore changes by itself at the activation second.

The Router requires every public order to pin the current hash and refuses a zero pin. As a result, every order committed in the preceding execution window becomes a terminal `ConfigMismatch`:
- the trader has paid the open order's execution bounty, but no execution is ever delivered;
- a requested close is never executed, and the position stays open.

This contradicts the protocol's own documented rule that the config hash changes only when the owner sends a finalizing transaction.

## Why this is a bug, not intended design

SECURITY.md:105-107 defines when the execution-config hash may change:

> Each successful Engine/Router finalization therefore changes `OrderLifecycleBook.currentExecutionConfigHash()`. Proposal, cancellation, pause/unpause, and failed finalization do not advance these versions.

So clients and traders are told the hash moves only on an owner's finalizing transaction, which they can observe. A proposal must never move it.

The treasury breaks that rule (`CfdTreasury.sol:45-46, 100-105`):

```solidity
function protocolTreasury() public view returns (address) { return _activeTreasury().account; }
function _activeTreasury() internal view returns (TreasuryConfig storage active) {
    active = _currentTreasury;
    if (_pendingTreasury.account != address(0) && block.timestamp >= _currentTreasury.lockedUntil) {
        active = _pendingTreasury;
    }
}
```

`proposeProtocolTreasury` is a proposal. Its effect on `currentExecutionConfigHash()` arrives later, at `lockedUntil`, with no finalization transaction. Nothing on-chain marks the moment, and no event is emitted.

SECURITY.md:773-775 says identity drift producing `ConfigMismatch` is intentional, but it describes drift caused by finalized owner changes. The treasury has been part of the hash since `e8d9b558` (2026-08-25), when a treasury change required an owner transaction. The no-transaction scheduled activation was added later, in `1209ca8a` (#130, 2026-10-27). That is after the previous audit reports were published in `f50f8482` (#129, 2026-10-26), so neither the design note nor any prior audit covers this combination.

This is a code defect, not an owner's parameter choice. Scheduling a rotation to any address different from the current treasury triggers it, whatever address is chosen. The owner has no way to schedule a rotation that leaves queued orders intact, because the hash change happens at a fixed future second without any transaction they could time or batch.

## Vulnerability details

1. **The treasury resolves by timestamp.** `CfdTreasury.sol:100-105` (above).
2. **The resolved address is hashed into the execution configuration.** `OrderLifecycleBook.sol:172-202`; line 199 is `engine.protocolTreasury()`.
3. **Every public order must pin the current hash, and zero is rejected.** `router/OrderLiquidationBatchLogic.sol:295`:
   ```solidity
   if (bounds.expectedConfigHash == bytes32(0) || bounds.expectedConfigHash != currentConfigHash) revert ...;
   ```
   Traders cannot opt out.
4. **A drifted order is terminal and can be cleared by anyone.** `OrderRouter.clearInvalidOrders` (`OrderRouter.sol:199-204`, delegated to `OrderRecoverySidecar.clearInvalidOrders`) terminalizes config-mismatched orders even behind a live queue head. The reserved bounty is paid to the terminating caller (`OrderLifecycleBook.sol:719`, `recipient = receipt.executor`).

## Impact

At the activation timestamp, every pinned order committed during the preceding execution window fails. The default `maxExecutionWindowSeconds` is 60 s (`router/OrderExecutionOrchestrator.sol:11`).

- **Opens:** the trader has already committed and paid the open order's execution bounty, and receives no execution. In the PoC that is 0.20 USDG per open. The bounty goes to whichever caller terminates the order. A cleaner calling `clearInvalidOrders` gains nothing an ordinary executor would not have been paid; the harm is that the trader paid for a service that is never delivered.
- **Closes:** the requested close is voided. The trader keeps market exposure they had asked to exit and must notice the failure and resubmit.
- **No safe rotation:** every scheduled rotation to a new address does this to the orders queued at that second.

The loss per order is the committed bounty, and exposure is bounded by one execution window, so this is reported as Low.

## Proof of concept

**File:** `packages/perps/test/perps/poc/TreasuryActivationVoidsPinnedOrders.t.sol` (attached).
- **Harness:** the repo's `BasePerpTest`, with the release risk parameters from DEPLOYMENT.md: maintenance 0.1%, initial 0.2%, FAD 3%, execution fee 0.04%, VPI 0.01e18, liquidation charge 0.1%, default 60 s execution window.
- **Contracts:** the real Router, Lifecycle Book, Protection Book and Engine. Only MockPyth and MockSettlementToken (USDG) are mocks.
- **No contract in scope is modified.**

Prerequisite, then run:

```bash
git submodule update --init lib/forge-std lib/openzeppelin-contracts lib/morpho-blue
cd packages/perps
FOUNDRY_TEST=test/perps/poc forge test --match-path test/perps/poc/TreasuryActivationVoidsPinnedOrders.t.sol -vv
```

`test_C3_scheduledTreasuryActivationVoidsQueuedOrders`:
1. Trader A opens a 20,000-token LONG through `commitOrder` and `executeOrder`.
2. The owner calls `setProtocolTreasury(T1, 2 days)`, then `proposeProtocolTreasury(T2, 30 days)`. Activation = `lockedUntil`.
3. At `lockedUntil − 30 s`, A commits a close of the full position, and B and C each commit a 20,000-token open. All are pinned to `h0 = currentExecutionConfigHash()`.
4. At `lockedUntil`, with no transaction, `currentExecutionConfigHash() != h0` and `protocolTreasury() == T2`.
5. Any caller runs `clearInvalidOrders([A, B, C])`. All three end `Failed / ConfigMismatch`, the two open bounties (400,000 atoms) are paid out with no execution, and A's position is still 20,000 tokens.

`test_C3_control_sameOrdersExecuteWithoutActivation`: the same open with no successor scheduled executes normally at the same timestamp.

Output:
```
[PASS] test_C3_control_sameOrdersExecuteWithoutActivation()
[PASS] test_C3_scheduledTreasuryActivationVoidsQueuedOrders()
  config hash changed at activation (no tx)
  orders voided 3
  open bounties paid to cleaner (USDG atoms) 400000
  trader0 lost close; trader1/2 lost (atoms) 200000 200000
```

## Recommendation

The treasury only decides where fees are routed; it changes none of the execution terms a trader pins. Either:

- remove `engine.protocolTreasury()` from `currentExecutionConfigHash()`; or
- hash the stored treasury records (current and pending account plus `lockedUntil`) instead of the timestamp-resolved address. The hash then changes only on an owner transaction, consistent with SECURITY.md:105-107.

If neither is acceptable, document that activation voids pinned orders committed in the preceding execution window, and emit an event at proposal time so clients can stop pinning near `activationTime`.



## Proof of Concept

## Proof of Concept
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// C-3 PoC — a scheduled protocol-treasury activation (documented, legitimate owner action) silently changes
// OrderLifecycleBook.currentExecutionConfigHash() at `lockedUntil` with no transaction (CfdTreasury.sol:101-106;
// OrderLifecycleBook.sol:199 hashes engine.protocolTreasury()). Every queued order pinned to the old hash becomes a
// terminal ConfigMismatch. Any caller sweeps them with clearInvalidOrders and is paid the traders' open bounties for
// doing no execution work; traders' queued closes are voided and their positions stay open.
// README.md: activation "becomes the recipient of future protocol fees"; the lock "applies to future protocol fee
// routing" — it does not say in-flight orders are invalidated.
//
// Configuration: release risk parameters from packages/perps/DEPLOYMENT.md @ a82d4fef — maintenance margin 0.1%
// (maintMarginBps 10), initial margin 0.2% (initMarginBps 20), FAD margin 3%, execution fee 0.04% (engine default
// executionFeeBps 4), VPI 0.01e18, liquidation charge 0.1% with 50% keeper share; default
// maxExecutionWindowSeconds = 60 s (router/OrderExecutionOrchestrator.sol:11).
// Real Router / Lifecycle Book / Protection Book / Engine; only MockPyth and MockSettlementToken (USDG) are mocks.
// Run: git submodule update --init lib/forge-std lib/openzeppelin-contracts lib/morpho-blue
//      cd packages/perps && FOUNDRY_TEST=test/perps/poc forge test --match-path \
//        test/perps/poc/TreasuryActivationVoidsPinnedOrders.t.sol -vv

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console2} from "forge-std/console2.sol";

contract TreasuryActivationVoidsPinnedOrdersPoC is BasePerpTest {

    address internal constant T1 = address(0x7EA1);
    address internal constant T2 = address(0x7EA2);
    address internal constant CLEANER = address(0xC1EA);
    address internal constant KEEPER = address(0xE0EC);
    address[3] internal traders = [address(0xA1), address(0xA2), address(0xA3)];

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

    function _status(
        uint64 id
    ) internal view returns (OrderV3Types.LifecycleStatus s, OrderV3Types.TerminalReason r) {
        OrderV3Types.TerminalOutcome memory o = router.lifecycleBook().terminalOutcome(id);
        return (o.status, o.reason);
    }

    function test_C3_scheduledTreasuryActivationVoidsQueuedOrders() public {
        _startRecordingLogs();
        for (uint256 i = 0; i < 3; i++) {
            _fundTrader(traders[i], 50_000e6);
        }

        // trader 0 already holds a position it will try to close at the activation minute
        vm.prank(traders[0]);
        uint64 openId = router.commitOrder(CfdTypes.Side.LONG, 20_000e18, 2_000e6, 0, false);
        bytes[] memory d0 = _mockPythUpdateData(1e8);
        vm.prank(KEEPER);
        router.executeOrder(openId, d0);
        (uint256 heldSize,,,,,,) = engine.positions(traders[0]);
        assertEq(heldSize, 20_000e18, "position opened");

        // Owner: documented, legitimate treasury commitment + scheduled successor (CfdTreasury.sol:70-89, :124-147)
        engine.setProtocolTreasury(T1, 2 days);
        uint256 L = engine.protocolTreasuryLockedUntil();
        engine.proposeProtocolTreasury(T2, 30 days);
        (address pending, uint256 activation,) = engine.pendingProtocolTreasury();
        assertEq(pending, T2);
        assertEq(activation, L);

        // Traders act normally in the minute before activation, pinning the hash the Router hands them
        vm.warp(L - 30);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(address(router));
        engine.updateMarkPrice(1e8, uint64(vm.getBlockTimestamp()));
        bytes32 h0 = router.lifecycleBook().currentExecutionConfigHash();
        uint64[] memory ids = new uint64[](3);
        uint256 cap = engine.CAP_PRICE();
        vm.prank(traders[0]); // close the held position
        ids[0] = router.commitOrder(CfdTypes.Side.LONG, 20_000e18, 0, cap, true);
        vm.prank(traders[1]);
        ids[1] = router.commitOrder(CfdTypes.Side.SHORT, 20_000e18, 2_000e6, 0, false);
        vm.prank(traders[2]);
        ids[2] = router.commitOrder(CfdTypes.Side.LONG, 20_000e18, 2_000e6, 0, false);
        uint256 bounties = router.getAccountReservations(traders[1]).executionBountyAssets
            + router.getAccountReservations(traders[2]).executionBountyAssets;
        uint256[3] memory before;
        for (uint256 i = 0; i < 3; i++) {
            before[i] = clearinghouse.balanceAssets(traders[i]);
        }

        // Activation instant: no transaction of any kind
        vm.warp(L);
        vm.roll(vm.getBlockNumber() + 1);
        bytes32 h1 = router.lifecycleBook().currentExecutionConfigHash();
        assertTrue(h1 != h0, "config hash changed at activation with no transaction");
        assertEq(engine.protocolTreasury(), T2);

        // Anyone sweeps every queued order, ahead of execution, and is paid for it
        uint256 cleanerBefore = clearinghouse.balanceAssets(CLEANER);
        vm.prank(CLEANER);
        (, uint256 cleared) = router.clearInvalidOrders(ids);
        assertEq(cleared, 3, "all three orders voided");
        for (uint256 i = 0; i < 3; i++) {
            (OrderV3Types.LifecycleStatus s, OrderV3Types.TerminalReason r) = _status(ids[i]);
            assertEq(uint8(s), uint8(OrderV3Types.LifecycleStatus.Failed));
            assertEq(uint8(r), uint8(OrderV3Types.TerminalReason.ConfigMismatch));
        }
        uint256 cleanerGain = clearinghouse.balanceAssets(CLEANER) - cleanerBefore;
        assertEq(cleanerGain, bounties, "cleaner collected the open bounties without executing anything");
        (heldSize,,,,,,) = engine.positions(traders[0]);
        assertEq(heldSize, 20_000e18, "trader 0's close was voided; position still open");

        console2.log("config hash changed at activation (no tx)");
        console2.log("orders voided", cleared);
        console2.log("open bounties paid to cleaner (USDG atoms)", cleanerGain);
        console2.log("trader0 lost close; trader1/2 lost (atoms)", before[1] - clearinghouse.balanceAssets(traders[1]),
            before[2] - clearinghouse.balanceAssets(traders[2]));
    }

    /// Control: identical orders one second earlier relative to activation execute normally.
    function test_C3_control_sameOrdersExecuteWithoutActivation() public {
        _startRecordingLogs();
        _fundTrader(traders[1], 50_000e6);
        engine.setProtocolTreasury(T1, 2 days);
        uint256 L = engine.protocolTreasuryLockedUntil();
        // no successor scheduled
        vm.warp(L - 30);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(address(router));
        engine.updateMarkPrice(1e8, uint64(vm.getBlockTimestamp()));
        vm.prank(traders[1]);
        uint64 id = router.commitOrder(CfdTypes.Side.SHORT, 20_000e18, 2_000e6, 0, false);
        bytes[] memory d = _mockPythUpdateData(1e8);
        vm.warp(L);
        vm.prank(KEEPER);
        router.executeOrder(id, d);
        (OrderV3Types.LifecycleStatus s, OrderV3Types.TerminalReason r) = _status(id);
        assertEq(uint8(s), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(uint8(r), uint8(OrderV3Types.TerminalReason.Executed));
    }

}
```
