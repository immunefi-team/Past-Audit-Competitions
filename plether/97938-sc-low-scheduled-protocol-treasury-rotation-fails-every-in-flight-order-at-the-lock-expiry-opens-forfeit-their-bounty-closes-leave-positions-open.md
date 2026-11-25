# \#97938 [SC-Low] Scheduled protocol-treasury rotation fails every in-flight order at the lock expiry: opens forfeit their bounty, closes leave positions open

**Submitted on Nov 4th 2026 at 00:30:07 UTC by @vortez for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97938
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary

Since PR #130, `CfdEngine.protocolTreasury()` is resolved from `block.timestamp`: a successor queued with `proposeProtocolTreasury` becomes effective at the exact lock expiry, with no transaction. `OrderLifecycleBook.currentExecutionConfigHash()` includes `engine.protocolTreasury()`, so the execution-config hash changes at that second. Every external order must pin that hash, so every order still in flight at the expiry fails terminally with `ConfigMismatch`. Opens forfeit their reserved execution bounty to the executor, and closes leave the position open. Nothing a trader pinned changes; only the recipient of protocol fee credits does. This repeats at every scheduled rotation, the documented way to rotate the treasury.

## Technical details

1. **Timestamp-resolved treasury.** `CfdTreasury._activeTreasury()` (`CfdTreasury.sol:101-106`):
   ```solidity
   active = _currentTreasury;
   if (_pendingTreasury.account != address(0) && block.timestamp >= _currentTreasury.lockedUntil) {
       active = _pendingTreasury;
   }
   ```
   `protocolTreasury()` (`:45-47`) returns this value. `SECURITY.md:121-127`: "At the exact expiry, the latest pending address becomes effective automatically … without a confirmation or keeper transaction." `SECURITY.md:151`: "There is no autonomous activation event at expiry."

2. **The treasury is part of the execution-config hash.** `OrderLifecycleBook.currentExecutionConfigHash()` (`OrderLifecycleBook.sol:172-203`) encodes `engine.protocolTreasury()` (`:199`). The returned hash therefore changes at the expiry timestamp without any storage write.

3. **Every external order pins the hash, and a mismatch is terminal.** A fresh external request must supply the current nonzero hash (`SECURITY.md:754-762`; `router/OrderLiquidationBatchLogic.sol:296`). At processing time, `OrderRouterExecutionSidecar._preOracleAction` (`:907-915`) and the execution item (`:404-412`) compare `book.currentExecutionConfigHash()` with `pending.bounds.expectedConfigHash` and settle a terminal `ConfigMismatch`. Orders live up to `maxExecutionWindowSeconds` (60 s by default, `router/OrderExecutionOrchestrator.sol:11`), so every order committed in roughly the last minute before the expiry is caught.

4. **Terminal failure pays the open bounty to the executor.** `OrderRouter._settleTerminalBounty` (`OrderRouter.sol:320-345`) takes the reserved order bounty and credits it to the processing executor (`engine.creditBounty`). Open bounties are 1 bp of notional, clamped to 0.01–0.20 USDG (`router/OrderRouterBase.sol:72-74`). Closes carry no bounty, but the close does not happen.

**Why the existing design does not cover this.** `SECURITY.md:766-775` says "finalized policy drift" producing `ConfigMismatch` is intentional, and lists the treasury among the hashed fields. That rationale fits governance finalizations, which are explicit, event-emitting transactions that change execution policy. A scheduled treasury activation differs on both points:

- **It is not a transaction.** It happens silently at a timestamp, so no transaction or event marks the boundary.
- **It changes nothing about execution.** PR #130 itself states "Fee amounts, LP economics, and junior-vault maintenance-fee recipients are unchanged". Only the clearinghouse account that receives future protocol credits changes.

Neither PR #130 nor `SECURITY.md` mentions that a rotation invalidates in-flight orders, so integrators and users get no warning.

## Impact

Selected impact: **Griefing (no profit motive for an attacker, but damage to the users or protocol)**.

At every scheduled treasury rotation, which is the documented governance flow (`setProtocolTreasury(treasury, lock)` → `proposeProtocolTreasury(successor, lock)`, locks of 1 s to 90 days):

- **Every config-pinned order in flight fails terminally.** That is roughly every order committed in the last ~60 s before the expiry, across all accounts. In the PoC: two opens and one close.
- **Traders who were exiting stay exposed.** The close does not execute and the position stays open. The trader must notice the failure and resubmit at a later price, which can matter in a fast market or near the liquidation threshold.
- **Traders who were opening lose the trade and the reserved bounty.** The bounty (≤ 0.20 USDG per order) is credited to the executor. The trade must be re-entered at a later price.
- **The boundary is predictable, but only off-chain.** `pendingProtocolTreasury()` exposes the activation time, but users and front-ends that pin `currentExecutionConfigHash()` get no on-chain signal that it is about to change.

No LP or protocol funds are lost, and no attacker needs to act. The damage falls on traders and is caused by the protocol's own scheduled transition. Per the program rules, a legitimate and documented action of a trusted role that triggers a bug is in scope. Here the owner only uses the documented rotation flow, correctly.

## Recommended fix

Remove `engine.protocolTreasury()` from `currentExecutionConfigHash()` (`OrderLifecycleBook.sol:199`). The treasury identity does not affect any value a trader pins, and every protocol credit already resolves `protocolTreasury()` at settlement time (for example `OrderLifecycleBook.sol:715`), so removing it does not weaken any binding a trader relies on.

If the treasury must remain bound, two alternatives:

- **Explicit activation.** Make the activation explicit: a permissionless `activateProtocolTreasury()` that calls `_syncTreasury()` and emits an event, and hash a treasury version that only advances in that call. The boundary then becomes an observable transaction.
- **Pre-activation hash.** Accept the pre-activation hash for orders whose `commitTime` is earlier than the activation timestamp.

## Severity justification

- **Impact category.** Griefing: damage to users without an attacker profit motive. Every user with a pinned order in the roughly 60 s before each scheduled rotation has it terminally failed. Exits do not execute, so positions stay exposed, and opens also lose their bounty. The damage recurs with every rotation and needs no attacker.
- **Trusted role.** The owner performs only the documented rotation flow. The program states that a legitimate, documented action of a trusted role that triggers a bug is in scope.
- **Not a known issue.** The behavior is not in the known-issues list, in the prior audits (Cecuro, SC Audit Studio, scan `d4541b49`), or in the `SECURITY.md` Known Limitations. `CfdTreasury` was added in PR #130 after the Cecuro audit commit `6d6622f2`. Each piece is documented separately: automatic activation (`SECURITY.md:121-127`) and the treasury being part of the hash (`:766-775`). What is not documented, in `SECURITY.md` or in PR #130, is that combining them invalidates every in-flight order at each rotation. PR #130 describes the user-facing consequence only as "Future credits switch at the scheduled expiry … Fee amounts, LP economics … are unchanged".
- **Magnitude, stated plainly.** No LP or protocol funds are lost. The direct loss is ≤ 0.20 USDG per open; the main harm is failed trades, including failed exits, at a protocol-scheduled moment.


## Proof of Concept

## Proof of Concept

Runs in the repository's Foundry suite at the commit in scope (`a82d4fefb02768e5b3bddcc97dee371eee24fea8`). It inherits only `test/perps/BasePerpTest.sol`, uses the real Router path (`router.commitOrder` / `router.executeOrder` with the suite's mock Pyth payloads) and the release risk parameters from `packages/perps/DEPLOYMENT.md`. No in-scope contract is modified.

### How to run

Save the file below as `packages/perps/test/perps/audit/TreasuryRotationPoC.t.sol` (it imports `../BasePerpTest.sol`) and run:

```
cd packages/perps
forge test --match-path test/perps/audit/TreasuryRotationPoC.t.sol -vv
```

### What each test shows

- **`test_control_executedBeforeExpiry` (control).** The owner locks treasury A for 7 days and schedules treasury B with `proposeProtocolTreasury`. Alice and Bob commit opens 30 s before the expiry, a keeper executes them 10 s before, and both positions open.
- **`test_scheduledRotationFailsInFlightOrders`.** Same schedule. Alice and Bob commit opens 20 s before the expiry, pinned to the current execution-config hash. The clock reaches the expiry with no transaction: `protocolTreasury()` becomes B and `currentExecutionConfigHash()` changes. The keeper processes both orders. Each returns `Failed` / `ConfigMismatch`, no position opens, each trader loses the 0.20 USDG reserved bounty, and the keeper is credited 0.40 USDG.
- **`test_scheduledRotationFailsInFlightClose`.** Carol opens 1 h before the expiry, then commits a full close 20 s before it. At the expiry the close returns `Failed` / `ConfigMismatch` and Carol's position remains open.

### Output

```
[PASS] test_control_executedBeforeExpiry()
[PASS] test_scheduledRotationFailsInFlightClose()
  carol position after her close failed 20000000000000000000000
[PASS] test_scheduledRotationFailsInFlightOrders()
  treasury A -> B at expiry; hash changed with no transaction
  alice / bob bounty forfeited (USDG atoms) 200000 200000
  keeper credited (USDG atoms) 400000
Suite result: ok. 3 passed; 0 failed; 0 skipped
```

### Test file: `TreasuryRotationPoC.t.sol`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console} from "forge-std/console.sol";

import {BasePerpTest} from "../BasePerpTest.sol";

/// A scheduled protocol-treasury rotation changes `currentExecutionConfigHash()` at the exact lock expiry with no
/// transaction, so every config-pinned order still in flight fails terminally with ConfigMismatch and opening orders
/// forfeit their execution bounty. Uses only the repository's BasePerpTest and the real Router path.
contract TreasuryRotationPoCTest is BasePerpTest {

    address internal alice = address(0xA11CE);
    address internal bob = address(0xB0B0);
    address internal keeper = address(0xEEEE);
    address internal treasuryA = address(0x7A0A);
    address internal treasuryB = address(0x7B0B);
    uint256 internal constant LOCK = 7 days;

    /// Release parameters from packages/perps/DEPLOYMENT.md.
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

    function _openRequest(
        uint256 salt
    ) internal view returns (OrderV3Types.OrderRequest memory req) {
        req.clientOrderId = keccak256(abi.encode("treasury-rotation", salt));
        req.side = CfdTypes.Side.LONG;
        req.sizeDelta = 20_000e18;
        req.marginDelta = 5_000e6;
        req.targetPrice = 1; // LONG open: no practical slippage limit
        req.isClose = false;
        req.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp) + 60,
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
        });
    }

    function _refreshMark() internal {
        bytes[] memory data = _mockPythUpdateData(1e8);
        vm.deal(address(this), 10 ether);
        router.updateMarkPrice{value: 1 ether}(data);
    }

    function _scheduleRotation() internal returns (uint256 activation) {
        // Documented governance flow: lock treasury A, queue treasury B to take over at the lock expiry.
        engine.setProtocolTreasury(treasuryA, LOCK);
        activation = engine.protocolTreasuryLockedUntil();
        engine.proposeProtocolTreasury(treasuryB, LOCK);
    }

    function _commitOpens() internal returns (uint64 aliceOrder, uint64 bobOrder) {
        _refreshMark();
        _fundTrader(alice, 50_000e6);
        _fundTrader(bob, 50_000e6);
        OrderV3Types.OrderRequest memory ra = _openRequest(1);
        vm.prank(alice);
        aliceOrder = router.commitOrder(ra);
        OrderV3Types.OrderRequest memory rb = _openRequest(2);
        vm.prank(bob);
        bobOrder = router.commitOrder(rb);
    }

    function _execute(
        uint64 orderId
    ) internal returns (OrderV3Types.ExecutionResult memory res) {
        vm.roll(block.number + 1);
        bytes[] memory data = _mockPythUpdateData(1e8);
        vm.deal(keeper, 10 ether);
        vm.prank(keeper);
        res = router.executeOrder{value: 1 ether}(orderId, data);
    }

    /// Control: identical schedule; orders committed and executed before the expiry open normally.
    function test_control_executedBeforeExpiry() public {
        uint256 activation = _scheduleRotation();
        vm.warp(activation - 30);
        (uint64 a, uint64 b) = _commitOpens();
        vm.warp(activation - 10);
        assertEq(uint256(_execute(a).status), uint256(OrderV3Types.LifecycleStatus.Executed), "alice not executed");
        assertEq(uint256(_execute(b).status), uint256(OrderV3Types.LifecycleStatus.Executed), "bob not executed");
        (uint256 sizeA,,,,,,) = engine.positions(alice);
        (uint256 sizeB,,,,,,) = engine.positions(bob);
        assertEq(sizeA, 20_000e18);
        assertEq(sizeB, 20_000e18);
    }

    /// Orders committed 20 s before the expiry fail with ConfigMismatch once the clock passes it, with no transaction.
    function test_scheduledRotationFailsInFlightOrders() public {
        uint256 activation = _scheduleRotation();
        vm.warp(activation - 20);
        bytes32 hashBefore = router.lifecycleBook().currentExecutionConfigHash();
        (uint64 a, uint64 b) = _commitOpens();
        uint256 aliceBefore = clearinghouse.balanceAssets(alice);
        uint256 bobBefore = clearinghouse.balanceAssets(bob);
        uint256 keeperBefore = clearinghouse.balanceAssets(keeper);

        vm.warp(activation); // the lock expires; nobody sends a transaction
        bytes32 hashAfter = router.lifecycleBook().currentExecutionConfigHash();
        assertEq(engine.protocolTreasury(), treasuryB, "successor not effective");
        assertTrue(hashBefore != hashAfter, "config hash unchanged");

        OrderV3Types.ExecutionResult memory ra = _execute(a);
        OrderV3Types.ExecutionResult memory rb = _execute(b);
        assertEq(uint256(ra.status), uint256(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint256(ra.terminalReason), uint256(OrderV3Types.TerminalReason.ConfigMismatch));
        assertEq(uint256(rb.status), uint256(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint256(rb.terminalReason), uint256(OrderV3Types.TerminalReason.ConfigMismatch));

        (uint256 sizeA,,,,,,) = engine.positions(alice);
        (uint256 sizeB,,,,,,) = engine.positions(bob);
        assertEq(sizeA, 0, "alice position opened");
        assertEq(sizeB, 0, "bob position opened");

        uint256 aliceLost = aliceBefore - clearinghouse.balanceAssets(alice);
        uint256 bobLost = bobBefore - clearinghouse.balanceAssets(bob);
        uint256 keeperGained = clearinghouse.balanceAssets(keeper) - keeperBefore;
        console.log("treasury A -> B at expiry; hash changed with no transaction");
        console.log("alice / bob bounty forfeited (USDG atoms)", aliceLost, bobLost);
        console.log("keeper credited (USDG atoms)", keeperGained);
        assertGt(aliceLost, 0, "no bounty forfeited");
        assertEq(keeperGained, aliceLost + bobLost, "bounties not credited to the executor");
    }

    /// A trader trying to exit is also affected: a pinned close in flight at the expiry fails and the position stays open.
    function test_scheduledRotationFailsInFlightClose() public {
        uint256 activation = _scheduleRotation();
        // carol opens well before the expiry
        address carol = address(0xCA201);
        vm.warp(activation - 1 hours);
        _refreshMark();
        _fundTrader(carol, 50_000e6);
        OrderV3Types.OrderRequest memory open = _openRequest(3);
        vm.prank(carol);
        uint64 openId = router.commitOrder(open);
        assertEq(uint256(_execute(openId).status), uint256(OrderV3Types.LifecycleStatus.Executed), "open failed");

        // 20 s before the expiry carol commits a full close pinned to the current hash
        vm.warp(activation - 20);
        _refreshMark();
        OrderV3Types.OrderRequest memory close = _openRequest(4);
        close.isClose = true;
        close.marginDelta = 0;
        close.targetPrice = type(uint256).max; // LONG close: no practical slippage limit
        close.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        vm.prank(carol);
        uint64 closeId = router.commitOrder(close);

        vm.warp(activation); // no transaction
        OrderV3Types.ExecutionResult memory rc = _execute(closeId);
        assertEq(uint256(rc.status), uint256(OrderV3Types.LifecycleStatus.Failed), "close not failed");
        assertEq(uint256(rc.terminalReason), uint256(OrderV3Types.TerminalReason.ConfigMismatch), "not ConfigMismatch");
        (uint256 size,,,,,,) = engine.positions(carol);
        console.log("carol position after her close failed", size);
        assertEq(size, 20_000e18, "position should still be open");
    }

}
```

