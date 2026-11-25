# \#99880 [SC-Insight] Senior pending-deposit cancellation is unconditionally blocked, contradicting the whitepaper's promise of pre-maturity cancellation

**Submitted on Nov 15th 2026 at 16:16:57 UTC by @crypt0n1337 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99880
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

# The MED-19 fix made Senior deposits non-cancellable, but WHITEPAPER 4.3 and CANONICAL_ENTRYPOINTS still promise pre-maturity cancellation and a "Senior-capacity escape" — committed Senior escrow is locked contrary to published behavior

## Impact

- **Primary severity:** Low — **Contract fails to deliver promised returns, but doesn't lose value**
- **Secondary reading (argued, weaker):** Medium — *Temporary freezing of funds for at least 1 hour* (across the weekly oracle-freeze window). Presented honestly with caveats; the headline is Low.
- **Relation:** spec-vs-code contradiction / incomplete fix (the fix updated SECURITY.md and WHITEPAPER 4.6 but left two other "promised behavior" documents describing the removed behavior).

## Asset

- `packages/perps/src` tree (Smart Contract), frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

## Affected code and documents (all file:line at HEAD)

- **Code (the denial):** `packages/perps/src/TrancheVault.sol:851-860` — `cancelPendingDeposit` guard; Senior revert at `:856-857` (`TrancheVault__SeniorDepositCommitted`).
- **Stale promise #1:** `packages/perps/WHITEPAPER.md:698-700` (section 4.3 "Synchronized LP epochs are an accounting and liquidity control", header at `:668`).
- **Stale promise #2:** `packages/perps/CANONICAL_ENTRYPOINTS.md:203-205` (LP settlement-hold section).
- **Orphaned "Senior-capacity escape":** `packages/perps/src/HousePool.sol:1393-1396` (areSeniorDepositReservationsWithinLimits still exists but is no longer referenced by the cancel guard; only other reference is interface IHousePool.sol:351).
- **Why the hold is the only healthy exit:** `HousePool.sol:483-485` (Senior admission closes on lpEpochSettlementPaused), `:774-776` (reserveSeniorDeposit reverts while paused), `:785-792` (releaseSeniorDepositReservation deliberately stays callable so escrow retains an exit path).
- **Weekend lock mechanism:** `HousePool.sol:1139-1145` (_entryStatusBlocked returns true while oracleFrozen), consistent with `SECURITY.md:885-886` ("During a frozen entry state the request remains deferred, its activation timestamp remains zero").
- **The consistent, updated doc (code matches this one):** `SECURITY.md:878-880`, `:1199-1203`, `:1394-1395`.

## The contradiction, precisely

### What the published documents promise

`WHITEPAPER.md:698-700` (verbatim):

> "A complete pending deposit is cancellable before maturity. After maturity, cancellation follows the documented epoch rejection, terminal-wipe, impairment, and Senior-capacity escape policy."

This statement is tranche-agnostic ("A complete pending deposit") — it covers Senior. It promises (a) cancellation before maturity for any complete pending deposit, and (b) a Senior-capacity escape path after maturity.

`CANONICAL_ENTRYPOINTS.md:203-205` (verbatim), the integrator-facing reference, under the settlement-hold section:

> "While true, `TrancheView.depositEnabled` can remain true even though activation and new redemption funding are stopped. Existing cancellation rules and funded claims are unchanged."

### What the code does

`TrancheVault.sol:851-860` (HEAD):

```solidity
if (
    (IS_SENIOR || currentLpEpoch() >= requestId) && !queueState.rejected
        && (!IS_SENIOR || !POOL.lpEpochSettlementPaused()) && !_projectedTerminalWipe()
        && !POOL.isSeniorImpairedAfterPendingDepositReconcile()
) {
    if (IS_SENIOR) {
        revert TrancheVault__SeniorDepositCommitted();   // :857
    }
    revert TrancheVault__DepositEpochAlreadyActive();
}
```

For the Senior vault (IS_SENIOR == true):
- `(IS_SENIOR || currentLpEpoch() >= requestId)` is always true — so maturity no longer matters; the guard fires before maturity, directly contradicting WHITEPAPER "cancellable before maturity".
- The old Senior-capacity escape (areSeniorDepositReservationsWithinLimits) was removed from the guard — contradicting WHITEPAPER "Senior-capacity escape policy".
- The only condition that lets a healthy Senior commitment become cancellable is `POOL.lpEpochSettlementPaused()` — i.e. an owner/pauser settlement hold. So CANONICAL's claim that cancellation rules are "unchanged" during a hold is exactly backwards for Senior: the hold is the only moment a healthy Senior commitment becomes refundable (and simultaneously reserveSeniorDeposit is closed, HousePool.sol:774-776).

### Git proof this is an incomplete-fix artifact (not pre-existing, not re-reporting MED-19)

The guard was introduced by PR #123 `d757d0b1` ("fix(perps): resolve LP vault, tranche, and claim findings", 2026-10-22), whose first sub-commit is literally "fix(perps): make senior deposit reservations firm". The pre-fix guard (from `git show d757d0b1 -- TrancheVault.sol`) was:

```solidity
// BEFORE PR #123:
if (currentLpEpoch() >= requestId && !queueState.rejected && !_projectedTerminalWipe()) {
    bool reservationsWithinLimits = !IS_SENIOR || POOL.areSeniorDepositReservationsWithinLimits();
    if (!POOL.isSeniorImpairedAfterPendingDepositReconcile() && reservationsWithinLimits) {
        revert TrancheVault__DepositEpochAlreadyActive();
    }
}
```

i.e. before the fix, the guard only fired after maturity (currentLpEpoch() >= requestId) and did honor the Senior-capacity escape — exactly as WHITEPAPER 4.3 still describes.

PR #123 touched WHITEPAPER.md (6 lines) and CANONICAL_ENTRYPOINTS.md (10 lines) but not these two passages. `git blame` at HEAD shows:
- WHITEPAPER.md:698-700 -> commit 562ab0280, 2026-08-24 (two months before the fix).
- CANONICAL_ENTRYPOINTS.md:203-205 -> commit 153c0f195, 2026-08-25 (before the fix).

The author did update the parallel claim in WHITEPAPER 4.6 (:773, "Accepted reservations are firm commitments and remain activatable after later capacity drift") and rewrote SECURITY.md to the firm model (:878-880: "a complete pending Junior deposit is cancellable before maturity, and an accepted Senior deposit is a firm commitment"). The fix therefore split the Junior/Senior cancellation story everywhere except WHITEPAPER 4.3 and CANONICAL_ENTRYPOINTS, which were left asserting the old, tranche-agnostic, refundable behavior.

## Call sequence (unprivileged, healthy pool, no special state)

1. Senior LP ALICE approves USDG and calls `seniorVault.requestDeposit(1_000_000e6, alice)` -> HousePool.reserveSeniorDeposit, funds escrowed in the vault; requestId targets a FUTURE epoch (currentLpEpoch() < requestId).
2. Before maturity, pool fully healthy (areSeniorDepositReservationsWithinLimits()==true, isSeniorImpairedAfterPendingDepositReconcile()==false, lpEpochSettlementPaused()==false, no projected wipe), ALICE calls `seniorVault.cancelPendingDeposit(requestId)` — which WHITEPAPER 4.3 says is "cancellable before maturity" -> reverts TrancheVault__SeniorDepositCommitted. The identical two calls on juniorVault succeed and return the assets.
3. The escrow remains locked until activation at the next settlement after maturity — normally 5-65 min (the documented epoch delay, WHITEPAPER 4.3), but for a request admitted shortly before the Friday market close it stays locked for the ENTIRE weekend oracle-freeze window (_entryStatusBlocked, HousePool.sol:1143), during which it can be neither cancelled nor activated.
4. None of the four code refund paths is reachable by the LP: settlement hold (owner/pauser), projected terminal wipe (catastrophic pool state), Senior impairment (loss event), or epoch rejection (needs a zero-share quote, unreachable at the 1 USDG MIN_TRANCHE_DEPOSIT_ASSETS).

## Harm (release config: Senior cap 40M / 80%, Senior coupon baseCarryBps=500)

A Senior request (1 USDG up to remaining capacity) is non-refundable escrow from admission to activation, contrary to the two documents a user or integrator relies on. An integrator implementing CANONICAL_ENTRYPOINTS will ship a "cancel pending deposit" action that always reverts for Senior in normal operation, and will fail to surface the one time-limited Senior refund right that actually exists (during a settlement hold). The LP's capital sits idle (no coupon accrual, no ability to react to news) for the lock duration — ~65 min ordinarily, ~48h across a weekend. No value is lost (the escrow ultimately activates into the Senior shares the LP requested), which is why the primary classification is Low "fails to deliver promised returns". The Medium temp-freeze reading applies to the weekend case (lock >> 1h, not user-reversible) but is weaker because the frozen window is environmental/time-gated (not attacker-triggerable) and the capital converts to the LP's intended Senior position rather than being stranded.

## Why this is NOT a known issue and NOT a duplicate

- Not MED-19 as written / not re-reporting the fix. MED-19 = "refundable senior reservations allow temporary capacity denial" (capacity parking); its fix = make reservations firm. This report does NOT argue the reservations should be refundable. It reports that the fix was INCOMPLETE on the documentation side: two shipped "promised behavior" documents still describe the pre-fix refundable behavior, which denies integrators/LPs a documented capability. This is the program's explicit reportable class ("the fix ... is INCOMPLETE" / spec-vs-code contradiction with a behavioral consequence).
- Not the pinned SECURITY.md Known Limitation (SECURITY.md:1394-1395: "accepted senior reservations are grandfathered ... an explicit settlement hold, projected terminal wipe, Senior impairment, or epoch rejection allows escrow recovery"). That limitation pins the firm behavior; it does NOT cover the fact that WHITEPAPER 4.3 and CANONICAL_ENTRYPOINTS still promise the opposite. The contradiction between those two documents and the code/SECURITY.md is unpinned.
- Distinct from report C (#99689, recap-remainder-freeze). C's root is a recapitalization remainder stranded in unassignedAssets that closes the tranche gate protocol-wide; it requires a prior loss + an owner recapitalizePool action, and it treats the SeniorDepositCommitted firm behavior as correct (building a freeze on top of it). S1-10 requires no loss, no recap, no owner action — a healthy pool — and its root is purely the documentation-vs-code contradiction. Different root, trigger, and mechanism.
- Distinct from L2/I1 (controller-consent). Those concern requestDeposit/requestRedeem accepting an arbitrary controller without consent — an authorization defect. S1-10 concerns the cancellation guard vs. the published cancellation promise. Unrelated roots.
- Distinct from reports A (#99680 protection) and B (HWM recap). Unrelated subsystems.

## Proof of Concept

## Proof of concept

Lane `D:\Pentest\bb\plether\pocw5` (copied from poc2: out/, cache/, foundry.toml); test `pocw5/test/SeniorNonCancellable.t.sol`, inheriting harn/BasePerpTest.sol. Default base funds Junior with 1,000,000e6 and configures Senior capacity (maxSeniorExposureAssets = type(uint256).max-1, maxSeniorShareBps = 9999), so the pool is healthy with Senior capacity open.

Run:
```
cd D:\Pentest\bb\plether\pocw5
C:\Users\pcher\.foundry\bin\forge.exe test --offline --match-path test/SeniorNonCancellable.t.sol -vv
```

Result: 3 passed; 0 failed.

| Test | What it shows | Result |
|---|---|---|
| test_SeniorPreMaturityCancelReverts_healthyPool | Healthy pool, currentLpEpoch() < requestId (pre-maturity), reservations within limits, not impaired, not paused — seniorVault.cancelPendingDeposit reverts TrancheVault__SeniorDepositCommitted; escrow stays pending and unactivated. Directly contradicts WHITEPAPER 4.3 "cancellable before maturity". | PASS |
| test_JuniorPreMaturityCancelSucceeds_control | Identical Junior request is cancellable pre-maturity: juniorVault.cancelPendingDeposit returns the full 1,000,000e6 to the LP. Proves the promised behavior genuinely exists and only the Senior branch was removed. | PASS |
| test_SeniorCommittedLockedAcrossFrozenWindow | Senior request admitted while markets open; warp into the weekly oracle-freeze window — canAcceptTrancheDeposits(senior)==false, canSettleDepositEntries()==false; cancel reverts at window open and still reverts 12h later; escrow unchanged and claimableDepositRequest==0 throughout. Demonstrates the >= 1h lock. | PASS |

The code+document citations above are self-sufficient for the Low spec-vs-code claim; the PoC strengthens it by demonstrating the revert in a fully healthy, pre-maturity state and the extended lock across the frozen window.

## Suggested fix

Either (a) restore the documented behavior for Senior (allow pre-maturity cancellation and/or the capacity escape), or (b) update WHITEPAPER.md:698-700 and CANONICAL_ENTRYPOINTS.md:203-205 to match the firm model already stated in SECURITY.md:878-880 and WHITEPAPER.md:773. Option (b) also requires correcting CANONICAL's "Existing cancellation rules ... are unchanged" to note that a settlement hold is what enables Senior escrow recovery.


### Test (pocw5/test/SeniorNonCancellable.t.sol), 3/3 pass offline
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "harn/BasePerpTest.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";

/// @title S1-10 - Senior pre-maturity cancellation is denied, contradicting WHITEPAPER 4.3 / CANONICAL_ENTRYPOINTS
/// @notice WHITEPAPER.md:698-699 promises, tranche-agnostically, "A complete pending deposit is cancellable before
///         maturity", and :700 promises a "Senior-capacity escape policy" after maturity. The MED-19 fix (PR #123,
///         d757d0b1) rewrote TrancheVault.cancelPendingDeposit so a committed Senior deposit reverts
///         TrancheVault__SeniorDepositCommitted whenever the pool is healthy (not rejected, settlement not paused,
///         no projected terminal wipe, Senior not impaired) - including BEFORE maturity - and it deleted the
///         areSeniorDepositReservationsWithinLimits() escape from the guard. SECURITY.md was updated to the firm
///         model; WHITEPAPER 4.3 and CANONICAL_ENTRYPOINTS:203-205 were left promising the old refundable behavior.
///         Local deployment only (MockSettlementToken / MockPyth), release-representative senior caps.
contract SeniorNonCancellableTest is BasePerpTest {

    address internal constant ALICE = address(0xA11CE); // Senior LP
    address internal constant BOB = address(0xB0B); // Junior LP (control)

    function setUp() public override {
        super.setUp();
        _refreshMark();
    }

    function _refreshMark() internal {
        vm.prank(address(router));
        engine.updateMarkPrice(1e8, uint64(block.timestamp));
    }

    function _requestSenior(address lp, uint256 assets) internal returns (uint256 epochId) {
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(seniorVault), assets);
        epochId = seniorVault.requestDeposit(assets, lp);
        vm.stopPrank();
    }

    function _requestJunior(address lp, uint256 assets) internal returns (uint256 epochId) {
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), assets);
        epochId = juniorVault.requestDeposit(assets, lp);
        vm.stopPrank();
    }

    /// PRIMARY: healthy pool, pre-maturity -> the documented Senior cancellation is DENIED.
    function test_SeniorPreMaturityCancelReverts_healthyPool() public {
        uint256 capacity = seniorVault.maxRequestDeposit(ALICE);
        uint256 assets = 1_000_000e6;
        assertGt(capacity, assets, "senior capacity open in a healthy pool");

        uint256 epochId = _requestSenior(ALICE, assets);

        // The request targets a FUTURE epoch: it is pre-maturity in the whitepaper's sense.
        assertLt(seniorVault.currentLpEpoch(), epochId, "request is pre-maturity");
        // The pool is fully healthy: none of the four code refund conditions is met.
        assertTrue(pool.areSeniorDepositReservationsWithinLimits(), "reservations within limits (the removed escape)");
        assertFalse(pool.isSeniorImpairedAfterPendingDepositReconcile(), "senior not impaired");
        assertFalse(pool.lpEpochSettlementPaused(), "settlement not paused");
        assertEq(seniorVault.pendingDepositRequest(epochId, ALICE), assets, "escrow is pending");

        // WHITEPAPER 4.3:698-699 says this is cancellable before maturity. The code reverts.
        vm.prank(ALICE);
        vm.expectRevert(TrancheVault.TrancheVault__SeniorDepositCommitted.selector);
        seniorVault.cancelPendingDeposit(epochId);

        // The escrow stays locked; no refund path is reachable by ALICE.
        assertEq(seniorVault.pendingDepositRequest(epochId, ALICE), assets, "escrow still locked, no refund");
        assertEq(seniorVault.claimableDepositRequest(epochId, ALICE), 0, "not yet activated");
    }

    /// CONTROL: the identical Junior request IS cancellable pre-maturity -> code matches the whitepaper for Junior,
    /// proving the promised behavior exists and only the Senior branch was silently removed.
    function test_JuniorPreMaturityCancelSucceeds_control() public {
        uint256 assets = 1_000_000e6;
        uint256 epochId = _requestJunior(BOB, assets);
        assertLt(juniorVault.currentLpEpoch(), epochId, "request is pre-maturity");

        uint256 balBefore = settlementToken.balanceOf(BOB);
        vm.prank(BOB);
        uint256 refunded = juniorVault.cancelPendingDeposit(epochId);

        assertEq(refunded, assets, "junior refund equals the deposit");
        assertEq(settlementToken.balanceOf(BOB) - balBefore, assets, "junior assets returned to BOB");
        assertEq(juniorVault.pendingDepositRequest(epochId, BOB), 0, "junior pending cleared");
    }

    /// ESCALATION: a Senior request admitted while markets are open stays locked across the weekly oracle-freeze
    /// window. Cancellation reverts and activation is deferred for the whole frozen window (>> 1 hour), so the
    /// escrow can be neither reclaimed nor activated. Demonstrates the >= 1h "temporary freezing" reading.
    function test_SeniorCommittedLockedAcrossFrozenWindow() public {
        uint256 assets = 1_000_000e6;
        uint256 epochId = _requestSenior(ALICE, assets);
        assertLt(seniorVault.currentLpEpoch(), epochId, "request is pre-maturity");

        // Find a frozen Saturday strictly after the current time (1_710_021_600 is a known frozen Saturday).
        uint256 fz = 1_710_021_600;
        while (fz - 12 hours <= block.timestamp) {
            fz += 7 days;
        }

        // Markets closed from Friday night: assert frozen 12h before the Saturday anchor and seed a valid frozen mark.
        vm.warp(fz - 12 hours);
        assertTrue(engine.isOracleFrozen(), "oracle frozen (weekend window open)");
        vm.prank(address(router));
        engine.updateMarkPrice(1e8, uint64(fz - 12 hours));

        // Entries are deferred for the whole frozen window: no admission and no activation.
        assertFalse(pool.canAcceptTrancheDeposits(true), "senior admission closed while frozen");
        assertFalse(pool.canSettleDepositEntries(), "deposit-entry settlement deferred while frozen");

        // The committed Senior escrow cannot be cancelled at the start of the window...
        vm.prank(ALICE);
        vm.expectRevert(TrancheVault.TrancheVault__SeniorDepositCommitted.selector);
        seniorVault.cancelPendingDeposit(epochId);

        // ...nor 12 hours later (still frozen) -> the lock persists far beyond the 1h threshold.
        vm.warp(fz);
        assertTrue(engine.isOracleFrozen(), "still frozen 12h later");
        vm.prank(ALICE);
        vm.expectRevert(TrancheVault.TrancheVault__SeniorDepositCommitted.selector);
        seniorVault.cancelPendingDeposit(epochId);

        // Escrow unchanged and never activated across the window.
        assertEq(seniorVault.pendingDepositRequest(epochId, ALICE), assets, "escrow locked across frozen window");
        assertEq(seniorVault.claimableDepositRequest(epochId, ALICE), 0, "not activated across frozen window");
    }
}

```
