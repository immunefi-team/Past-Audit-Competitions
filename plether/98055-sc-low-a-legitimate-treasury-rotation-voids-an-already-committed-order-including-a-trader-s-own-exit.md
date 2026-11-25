# \#98055 [SC-Low] A legitimate treasury rotation voids an already-committed order, including a trader's own exit

**Submitted on Nov 4th 2026 at 16:40:25 UTC by @martianstyle for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98055
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
`CfdTreasury` replaces the protocol treasury on a schedule, and the switch resolves from `block.timestamp` alone, with no transaction at the activation moment (`CfdTreasury.sol:33`, `:101-106`). That address is part of the configuration preimage an order commits to, because `OrderLifecycleBook.sol:199` builds the preimage from `engine.protocolTreasury()`. Terminal outcomes then re-validate against the live value instead of the snapshot: `OrderLifecycleBook.sol:715` and `OrderRecoverySidecar.sol:282` compare the receipt's `bountyRecipient` — snapshotted at `OrderLiquidationHandler.sol:147` — against the treasury read at validation time.

One legitimate rotation therefore moves both ends at once. Any order in the FIFO queue when it lands can no longer satisfy its own committed configuration. It is terminalised as a config mismatch, never executes, and the trader is charged the settlement fee for an order that failed because governance acted.

## Why this is in scope

The program states: *"In scope: a legitimate, documented action by a trusted role that triggers a bug harming users or the protocol."* That is exactly this. `setProtocolTreasury` is the documented privileged path; it is used correctly; and it harms users who were already committed. No malicious role, no compromised key, no reentrancy — the privileged action is a normal one and the harm is a logic error it triggers, which the privileged-address standard keeps valid: *"where an in-scope impact arises from a logic error that a privileged address can trigger, which remains valid."*

## Why this is not one of the listed known issues

The `Router and keeper limitations` section of `SECURITY.md` states: *"known typed planner/policy failures, slippage, expiry, and config mismatch are terminal for the affected order."* That documents the **consequence** — an order that already has a mismatch is terminal. It does not describe the **root cause** here: that a legitimate treasury rotation *manufactures* a mismatch on an order that was valid when it was committed, by putting the fee-routing destination into the commitment preimage. That composition of `protocolTreasury()` into the preimage is not stated anywhere in the listed known issues.

It also contradicts an explicit claim in the same document. `SECURITY.md` says of the treasury commitment: *"The commitment affects future protocol credits only."* The code contradicts that sentence, because the commitment also decides the validity of orders already committed. A documented guarantee that the code violates is a finding, not a known limitation.

For contrast, two adjacent disclosed items point the other way:

- **MED-3** is a third party *blocking* a rotation. This is the inverse: a rotation that completes cleanly and damages committed traders.
- **MED-13** is a governance *oracle* rotation retargeting *armed stop-losses*. Different privileged action, different asset, different failure. Here the action is a treasury rotation and the victim is the order queue.

## Severity assessment

I claim Medium under *Griefing*, and the reasoning is narrow enough to check.

The definition is *"no profit motive for an attacker, but damage to the users or the protocol"*. Both halves hold here. There is no profit motive: the trigger is a routine, correctly-executed governance rotation, and whoever performs it gains nothing from voiding the queue. There is damage to users: a committed exit is annulled and the trader is left exposed, and a committed open is voided with a settlement charge taken for a failure the trader did not cause. The program already places trusted-role actions that harm users in scope: *"In scope: a legitimate, documented action by a trusted role that triggers a bug harming users or the protocol."*

The strongest support is the project's own severity precedent for the adjacent case. Their disclosed **MED-13** is *"a governance oracle rotation retargets armed stop-losses"* — a governance rotation that silently discards a user's already-committed intent — and it is rated **Medium**. This is the same shape on a different axis: a governance rotation, on the treasury rather than the oracle, that silently discards a user's already-committed order. I cite MED-13 **only as a severity precedent**, not as this finding: MED-13 is a listed known issue and is not payable, and its root cause, asset and failure mode are all different from the one reported here.

I have deliberately not claimed Critical or High. There is no theft and no permanent lock, and executing the PoC disproved my own hypothesis that committed margin would be trapped. If a triager rejects the griefing framing, the honest landing point is Low under *"Contract fails to deliver promised returns, but doesn't lose value"*, and the defect stands either way.

## Impact

Deterministic in mechanism; bounded and market-dependent in magnitude.

**Variant A — a committed EXIT is voided and the position stays open.** A trader's close order is terminalised, so the trader does not exit. I measured the position still open at `100000000000000000000000` atoms after the close was rejected, versus `0` in the control. The trader believes they exited and remain exposed. Any loss they take before re-committing is caused by the voided exit, and its size depends on the market, so I do not claim a fixed figure for it.

**Variant B — a committed OPEN is voided and the trader is charged.** The order is lost and the settlement fee is taken from the trader for a failure that governance introduced: measured at `200000` atoms = 0.2 USDG on a 10,000 USDG order, scaling with order size.

Three properties make it worth fixing rather than tolerating: it hits **every** queued order at once rather than one unlucky order; it is **irreversible** — the order and the fee are gone, and the trader has no recovery path; and it is **invisible from the outside**, because the treasury can flip on a timestamp alone with no transaction to observe.

I have not overstated this. No funds are stranded — executing the PoC disproved my own first hypothesis that the margin would be trapped, and that margin release is why I stop at Medium rather than argue for High: the harm is real and systematic, but no value is permanently taken or locked.


## Vulnerability Details

Three individually reasonable facts compose:

1. **The treasury changes with no write.** `_activeTreasury()` returns `_pendingTreasury` once `block.timestamp >= _currentTreasury.lockedUntil`. A consumer that reads the treasury twice can see two different values inside one logical operation.
2. **The commitment preimage includes it.** `OrderLifecycleBook.sol:199` folds `engine.protocolTreasury()` into the preimage. Nothing about executing an order depends on where the protocol's own fee share is routed.
3. **The terminal check re-reads the live value.** The receipt carries an authenticated snapshot, and `OrderLifecycleBook.sol:715` / `OrderRecoverySidecar.sol:282` still validate against the current treasury.

A rotation between commit and execute moves (2) and (3) out from under an order admitted under the old value.

## Recommendation

Any one of these closes it; the first is smallest and safest:

1. **Drop `protocolTreasury()` from the committed configuration preimage.** Execution does not depend on the fee-routing destination, so removing it weakens no check. Recommended.
2. **Validate the bounty recipient against the receipt's own authenticated snapshot**, never against the live treasury.
3. **Refund the settlement charge on a terminal config mismatch caused by a rotation**, so a trader is never charged for a failure governance introduced.

A regression test that commits an order, rotates the treasury, then asserts the order still executes would have caught this, and the close variant would catch the exposure path.

## References

- `CfdTreasury.sol:33`, `:101-106` — timestamp-resolved activation, no transaction.
- `OrderLifecycleBook.sol:199` — `protocolTreasury()` in the committed configuration preimage.
- `OrderLifecycleBook.sol:715`, `OrderRecoverySidecar.sol:282` — terminal re-validation against the live treasury.
- `OrderLiquidationHandler.sol:147` — where the receipt's `bountyRecipient` snapshot is taken.
- `SECURITY.md` — `Router and keeper limitations` (the consequence documented, the cause not); `Protocol treasury commitment` (`"affects future protocol credits only"`, contradicted).
- Disclosed MED-3 and MED-13, for contrast of direction and asset.


## Proof of Concept

## Proof of Concept

Self-contained Foundry test extending the repo's own harness (`test/perps/BasePerpTest.sol`), no external fixtures:

    packages/perps/test/perps/PoC_TreasuryRotation.t.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";

/// @title PoC_TreasuryRotation — executed rather than argued
/// @notice Falsifies or confirms: a governance treasury rotation kills orders that are already in flight.
/// @dev Mechanism under test: `OrderLifecycleBook.sol:199` folds `engine.protocolTreasury()` into the
///      configuration hash that an order commits to, and `:715` / `OrderRecoverySidecar.sol:282` re-validate
///      `receipt.bountyRecipient` (snapshotted at `OrderLiquidationHandler.sol:147`) against the LIVE treasury.
///      Because the configuration hash contains the treasury, a rotation between COMMIT and EXECUTE invalidates
///      the committed order.
///
/// @dev PURPOSE OF THIS FILE, in order: (1) prove the commit -> execute flow works in the harness; (2) observe
///      the EXACT effect of an interleaved rotation; (3) measure whether any value is stranded, via
///      `_snapshot()`. It does not assume the verdict: it measures it.
///
/// Canonical repo API (taken from test/perps/accounting/waterfall/EngineMarginCappedNav.t.sol:41-43):
///      vm.prank(alice); router.commitOrder(side, size, margin, targetPrice, isClose);
///      router.executeOrder(orderId, _mockPythUpdateData());
contract PoCTreasuryRotationTest is BasePerpTest {

    address internal alice = address(0xA11CE);

    function setUp() public override {
        super.setUp();
        _fundTrader(alice, 50_000e6);
    }

    /// @notice CONTROL: with no rotation, commit + execute opens the position. If this test does not pass,
    ///         the harness or the router API changed and the PoC below proves nothing.
    function test_control_order_executes_without_rotation() public {
        vm.prank(alice);
        router.commitOrder(CfdTypes.Side.SHORT, 100_000e18, 10_000e6, 1e8, false);

        (uint256 sizeBefore,,,,,,) = engine.positions(alice);
        assertEq(sizeBefore, 0, "control: alice already held a position before execution");

        router.executeOrder(1, _mockPythUpdateData());

        (uint256 sizeAfter,,,,,,) = engine.positions(alice);
        assertGt(sizeAfter, 0, "control: the order did NOT open a position without rotation -> API/harness broken");
    }

    /// @notice EXECUTED: commit -> treasury ROTATION -> execute.
    ///         Measures whether the in-flight order survives the rotation, and whether the committed margin
    ///         is stranded.
    function test_treasury_rotation_voids_committed_open() public {
        // 1) Commit: the order binds the configuration hash, which includes protocolTreasury() (OrderLifecycleBook:199)
        vm.prank(alice);
        router.commitOrder(CfdTypes.Side.SHORT, 100_000e18, 10_000e6, 1e8, false);

        uint256 committedMargin = clearinghouse.orderMarginAssets(alice);
        assertGt(committedMargin, 0, "the commit locked no margin: review the flow");

        // 2) Treasury rotation by governance (a LEGITIMATE PATH, not attacker privilege).
        address newTreasury = address(0xB0B);
        engine.setProtocolTreasury(newTreasury);
        assertEq(engine.protocolTreasury(), newTreasury, "the rotation did not take effect");

        // 3) Execute after the rotation: capture the AUTHENTICATED receipt (mechanism + config hashes).
        Snapshot memory before = _snapshot();
        _startRecordingLogs();
        router.executeOrder(1, _mockPythUpdateData());
        _takeRecordedLogs();

        OrderV3Types.CompactOutcome memory o = _verifiedOutcome(router.lifecycleBook(), 1);
        Snapshot memory after_ = _snapshot();

        emit log_named_uint("in-flight open: status", uint8(o.status));
        emit log_named_uint("in-flight open: terminal reason", uint8(o.reason));
        emit log_named_uint("in-flight open: failure category", uint8(o.failureCategory));
        emit log_named_uint("in-flight open: failure code", uint8(o.failureCode));
        emit log_named_bytes32("in-flight open: expectedConfigHash", o.expectedConfigHash);
        emit log_named_bytes32("in-flight open: observedConfigHash", o.observedConfigHash);
        emit log_named_uint("in-flight open: order margin BEFORE", before.aliceOrderMargin);
        emit log_named_uint("in-flight open: order margin AFTER", after_.aliceOrderMargin);
        emit log_named_uint("in-flight open: alice settlement BEFORE", before.aliceSettlement);
        emit log_named_uint("in-flight open: alice settlement AFTER", after_.aliceSettlement);
        emit log_named_uint("in-flight open: pendingOrders BEFORE", before.pendingOrders);
        emit log_named_uint("in-flight open: pendingOrders AFTER", after_.pendingOrders);
        emit log_named_int("in-flight open: net cost to trader (atoms)", int256(after_.aliceSettlement) - int256(before.aliceSettlement));

        (uint256 sizeAfter,,,,,,) = engine.positions(alice);

        // MECHANISM: the configuration hash the order committed to NO LONGER matches the observed one,
        // because the rotation changed protocolTreasury() and that value enters the hash (OrderLifecycleBook:199).
        assertTrue(o.expectedConfigHash != o.observedConfigHash, "in-flight open: the hashes match -> the rotation did NOT invalidate the order");

        // IMPACT MEASURED: the order opens no position (size 0) and the queue is cleared, BUT the committed
        // margin is released. No value is stranded: the harm is the settlement charge the trader pays for an
        // order that fails because of GOVERNANCE, plus the opportunity cost of the lost order.
        assertEq(sizeAfter, 0, "in-flight open: the position opened, so it was not affected");
        assertEq(after_.pendingOrders, 0, "in-flight open: the order was not terminalised");
        assertEq(after_.aliceOrderMargin, 0, "in-flight open: the committed margin was NOT released");
        assertLt(int256(after_.aliceSettlement), int256(before.aliceSettlement), "in-flight open: the trader paid nothing");
        assertLe(before.aliceSettlement - after_.aliceSettlement, 1e6, "in-flight open: the charge exceeds 1 USDG -> re-assess severity");
    }

    /// @notice CONTROL: with no rotation, a committed CLOSE order executes and leaves the position at 0.
    function test_control_close_executes_without_rotation() public {
        vm.prank(alice);
        router.commitOrder(CfdTypes.Side.SHORT, 100_000e18, 10_000e6, 1e8, false);
        router.executeOrder(1, _mockPythUpdateData());

        (uint256 sizeOpen,,,,,,) = engine.positions(alice);
        assertGt(sizeOpen, 0, "control: the open left no position");

        vm.prank(alice);
        router.commitOrder(CfdTypes.Side.SHORT, 100_000e18, 0, 1e8, true);
        router.executeOrder(2, _mockPythUpdateData());

        (uint256 sizeAfter,,,,,,) = engine.positions(alice);
        assertEq(sizeAfter, 0, "control: the close did NOT close the position without rotation -> API misused");
    }

    /// @notice ON AN EXIT: CLOSE commit -> ROTATION -> execute.
    ///         If the close is invalidated, the trader does NOT exit: the position stays ALIVE and exposed.
    ///         That is the material harm (a committed exit silently annulled), not only the settlement charge.
    function test_treasury_rotation_voids_committed_close_and_leaves_position_open() public {
        // 1) Open a position and confirm it.
        vm.prank(alice);
        router.commitOrder(CfdTypes.Side.SHORT, 100_000e18, 10_000e6, 1e8, false);
        router.executeOrder(1, _mockPythUpdateData());
        (uint256 sizeOpen,,,,,,) = engine.positions(alice);
        assertGt(sizeOpen, 0, "the open left no position");

        // 2) The trader commits their EXIT.
        vm.prank(alice);
        router.commitOrder(CfdTypes.Side.SHORT, 100_000e18, 0, 1e8, true);

        // 3) Treasury rotation by governance (legitimate path).
        engine.setProtocolTreasury(address(0xB0B));

        // 4) The exit attempts to execute after the rotation.
        _startRecordingLogs();
        router.executeOrder(2, _mockPythUpdateData());
        _takeRecordedLogs();
        OrderV3Types.CompactOutcome memory o = _verifiedOutcome(router.lifecycleBook(), 2);

        (uint256 sizeAfter,,,,,,) = engine.positions(alice);
        emit log_named_uint("committed close: terminal reason", uint8(o.reason));
        emit log_named_bytes32("committed close: expectedConfigHash", o.expectedConfigHash);
        emit log_named_bytes32("committed close: observedConfigHash", o.observedConfigHash);
        emit log_named_uint("committed close: position size AFTER", sizeAfter);

        // MECHANISM: the rotation changed the configuration hash the exit had committed to.
        assertTrue(
            o.expectedConfigHash != o.observedConfigHash,
            "committed close: the hashes match -> the rotation did NOT invalidate the exit"
        );
        // IMPACT: the exit did NOT execute and the position is STILL ALIVE. The trader was left exposed unasked.
        assertGt(sizeAfter, 0, "committed close: the position closed, so the exit did take effect");
        assertEq(sizeAfter, sizeOpen, "committed close: the position changed size unexpectedly");
    }

    // ---------------------------------------------------------------------------------------
    // Minimal value snapshot (independent of PoCScaffold's, to avoid dragging in its tests)
    // ---------------------------------------------------------------------------------------

    struct Snapshot {
        uint256 aliceSettlement;
        uint256 aliceOrderMargin;
        uint256 aliceLockedMargin;
        uint256 pendingOrders;
    }

    function _snapshot() internal view returns (Snapshot memory s) {
        s.aliceSettlement = clearinghouse.balanceAssets(alice);
        s.aliceOrderMargin = clearinghouse.orderMarginAssets(alice);
        s.aliceLockedMargin = clearinghouse.lockedMarginAssets(alice);
        s.pendingOrders = router.pendingOrderCounts(alice);
    }
}
```

    forge test --root packages/perps --match-path test/perps/PoC_TreasuryRotation.t.sol -vv

Four tests, all passing under the repo's canonical profile (`via_ir = true`, `optimizer_runs = 200`). Two are controls proving the API is driven correctly and the normal path is healthy; two reproduce the defect, one per variant. Only Pyth updates and the mock settlement token are mocked, both outside scope.

```text
[PASS] test_control_order_executes_without_rotation() (gas: 2563420)
[PASS] test_control_close_executes_without_rotation() (gas: 3988635)
[PASS] test_treasury_rotation_voids_committed_open() (gas: 2215070)
  in-flight open: status: 3
  in-flight open: terminal reason: 4
  in-flight open: failure category: 0
  in-flight open: failure code: 0
  in-flight open: expectedConfigHash: 0xa68de8e0a6f1e7b3e82d240187b2a37afd8a23e9f31a4e223b7884fa1033d9c5
  in-flight open: observedConfigHash: 0x1419a5cf17f72abda6d1b47855c01a3ce1d221d2d9cfe70daa1dbb9a45c7282b
  in-flight open: order margin BEFORE: 10000000000
  in-flight open: order margin AFTER: 0
  in-flight open: alice settlement BEFORE: 50000000000
  in-flight open: alice settlement AFTER: 49999800000
  in-flight open: pendingOrders BEFORE: 1
  in-flight open: pendingOrders AFTER: 0
  in-flight open: net cost to trader (atoms): -200000
[PASS] test_treasury_rotation_voids_committed_close_and_leaves_position_open() (gas: 4117626)
  committed close: terminal reason: 4
  committed close: expectedConfigHash: 0xa68de8e0a6f1e7b3e82d240187b2a37afd8a23e9f31a4e223b7884fa1033d9c5
  committed close: observedConfigHash: 0x1419a5cf17f72abda6d1b47855c01a3ce1d221d2d9cfe70daa1dbb9a45c7282b
  committed close: position size AFTER: 100000000000000000000000
Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 43.87ms (49.29ms CPU time)
```

Sequence: Alice commits an order and the test asserts her order margin is funded immediately after, so the commitment is real and locked. Governance then rotates the treasury through `engine.setProtocolTreasury(newTreasury)`, and the test asserts the new address is live — the privileged path working exactly as designed. `router.executeOrder` then runs, and the receipt is pulled through the harness's authenticated capture (`_verifiedOutcome`), which checks it against the on-chain `terminalOutcome` and its receipt hash.

Evidence:

- **The two hashes differ**, `0xa68de8e0…` versus `0x1419a5cf…`. That is the mechanism rather than an inference: the configuration the order is measured against changed because the rotation changed the treasury. This is the assertion that pins the cause.
- **The order is terminalised, not executed:** `status 3`, `reason 4`, `positions(alice)` size stays `0`.
- **The exit variant leaves the position open** at `100000000000000000000000`, where the control closes it to `0`. Same code path, same order parameters, rotation the only difference.
- **Nothing is stranded:** order margin `10000000000` → `0` and `pendingOrderCounts` `1` → `0`. Asserted deliberately, because executing it disproved my first hypothesis.
- **The trader pays:** net settlement move `-200000` atoms.
