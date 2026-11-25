# \#97618 [SC-Insight] Matured LP deposit requests have no cancellation path while activation is deferred — extend the Senior hold-escape to all deferral causes and both tranches

**Submitted on Nov 2nd 2026 at 14:39:05 UTC by @L4rp for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97618
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**

## Description

Recommendation

In TrancheVault.cancelPendingDeposit, allow a matured deposit request to be cancelled whenever HousePool reports that entry activation is blocked (paused() || oracleFrozen || degradedMode || deficit != 0 — the same predicate _entriesMaySettle already evaluates), for both tranches, releasing the Senior capacity reservation exactly as the existing lpEpochSettlementPaused escape does. Alternatively, defer redemption funding whenever entries are deferred so holders and pending depositors are treated symmetrically. Either way, document the resulting lock in the LP-facing docs.

Observation

Two individually documented rules combine into an undocumented multi-day lock:

TrancheVault.sol:851-860 — once currentLpEpoch() >= requestId, cancelPendingDeposit reverts (DepositEpochAlreadyActive for Junior, SeniorDepositCommitted for Senior) unless the epoch was rejected, a terminal wipe or Senior impairment is projected, or — Senior only — lpEpochSettlementPaused() is set. It does not consult paused(), oracleFrozen, degradedMode, or the deficit state.
HousePool.sol:903-912, 1139-1145 — _settleLpEpoch defers deposit activation in exactly those four states; with nothing else to process, settleLpEpoch reverts HousePool__NoLpEpochProgress (or HousePool__DegradedMode).
HousePool.sol:854-856, 882-884 — redemption funding proceeds in the same frozen settlement at the 25 / 75 bps frozen fee.

A request that matures into any deferral window therefore can neither activate nor be withdrawn. For the weekly FX freeze that is a measured 47 h 15 min lock on Junior escrow (Friday 22:00 UTC → Sunday 21:15 UTC in the PoC calendar), during which existing share holders can still redeem. Under an owner pause() or degraded mode the lock has no time bound. Junior has no escape at all; Senior escapes only the explicit hold, not the recurring freeze.

The partial Senior escape shows the intended policy is “an exit exists when settlement is blocked”; it is implemented for one tranche and one of four causes. SECURITY.md Known Limitations acknowledges that “redemption funding is allowed to proceed while entry activation is deferred” — this Insight is about the missing cancellation path and its asymmetric implementation, which no document states, and which an LP cannot anticipate before requesting.

Why it matters

LPs who request late on Friday are, in effect, making a binding ~48-hour commitment with no exit and no disclosure of that fact, while share holders retain exit liquidity in the same window. If that commitment is intended, it should be stated where LPs can read it; if not, the escape hatch that already exists should cover all deferral causes.

## Proof of Concept

Step-by-step (weekend variant)

Friday 21:20 UTC (live, pre-cutoff): LP requests a 100,000 USDG Junior deposit; escrow is pulled; the request targets the 22:00 UTC epoch.
22:00 UTC: FX calendar freezes the oracle; the epoch has matured.
LP calls cancelPendingDeposit → reverts TrancheVault__DepositEpochAlreadyActive.
Anyone calls settleLpEpoch → entries deferred; with nothing else to process it reverts HousePool__NoLpEpochProgress. Nothing claimable; escrow still pending.
Saturday 12:00 UTC: same result.
In the same frozen window an existing Junior holder requests a redemption; the frozen settlement funds it (75 bps fee) while reporting the matured deposit as deferred. Holder exits; depositor remains locked.
Sunday 21:15 UTC: first live settlement activates the deposit at reopening NAV. Lock duration: 47 h 15 min, no exit.

Variants in the test: Senior request (reverts SeniorDepositCommitted; lpEpochSettlementPaused() is false so the hold-escape does not apply); owner pause() after maturity (lock lasts until unpause()); degraded mode (settlement reverts HousePool__DegradedMode; flag set via stdstore because the organic latch requires a real pool loss — disclosed).

File
packages/perps/test/perps/spec/lp/MaturedDepositStuckWhileEntriesDeferred.t.sol — uses the repository’s HousePoolAsyncTestBase fixture; no contract or existing test file is modified. 5 tests, all pass on Foundry 1.5.1 at commit a82d4fe; hosting directory re-runs green (243/243); full non-fork suite passes (2,549 / 0).

solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// PoC for finding A-1: a deposit request whose epoch has matured can neither activate nor be cancelled while entry
// activation is deferred (oracle frozen / pool paused / degraded mode). TrancheVault.cancelPendingDeposit refuses once
// currentLpEpoch() >= requestId (assuming activation is imminent) while HousePool._settleLpEpoch defers entries in
// exactly those states — and, with nothing else to process, the settlement call itself reverts (NoLpEpochProgress /
// DegradedMode). Junior has no escape at all; Senior escapes only the explicit settlement hold. Meanwhile existing
// share holders can still redeem in the same frozen window (the asymmetry).
//
// Copy to packages/perps/test/perps/spec/lp/ and run:
//   forge test --root packages/perps --match-path "*MaturedDepositStuck*" -vvv

import {HousePoolAsyncTestBase} from "../../shared/HousePoolTestBase.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {StdStorage, stdStorage} from "forge-std/StdStorage.sol";

contract MaturedDepositStuckWhileEntriesDeferredTest is HousePoolAsyncTestBase {

    using stdStorage for StdStorage;

    address internal constant LP = address(0x1A1A);
    uint256 internal constant DEPOSIT = 100_000e6;

    // Calendar (March 2024; New York on standard time until Sun Mar 10 07:00 UTC, so the Friday FX boundary is
    // 22:00 UTC and the Sunday reopen is 21:00 UTC). Fixture SETUP_TIMESTAMP is Mon 2024-03-04 06:00 UTC.
    uint256 internal constant FRI_MIDNIGHT = 1_709_856_000; // Fri 2024-03-08 00:00 UTC
    uint256 internal constant FRI_REQUEST = FRI_MIDNIGHT + 21 hours + 20 minutes; // live, pre-FAD (21:30), pre-cutoff
    uint256 internal constant FRI_FROZEN = FRI_MIDNIGHT + 22 hours + 30 minutes; // epoch matured, oracle frozen
    uint256 internal constant SUN_REOPEN = FRI_MIDNIGHT + 2 days + 21 hours + 15 minutes; // Sun 21:15 UTC live again

    function _refreshMark() internal {
        vm.prank(address(router));
        engine.updateMarkPrice(1e8, uint64(block.timestamp));
    }

    function _expectCancelBlocked(
        TrancheVault vault,
        uint256 requestId,
        bytes4 selector
    ) internal {
        vm.expectRevert(selector);
        vm.prank(LP);
        vault.cancelPendingDeposit(requestId, LP, LP);
    }

    /// EXPECTED: an LP whose matured deposit cannot be activated can withdraw their escrowed USDG, or activation
    ///           happens. (SECURITY.md:878-880 documents pre-maturity cancellation and a hold-escape for matured
    ///           Senior requests; ACCOUNTING_SPEC.md:554 says frozen mode "defers" activation. Nothing documents a
    ///           multi-day lock with no exit.)
    /// ACTUAL:   for the entire FX weekend freeze the request is neither activatable (settlement reverts
    ///           NoLpEpochProgress, zero claimable) nor cancellable (DepositEpochAlreadyActive); escrow is locked for
    ///           ~47 hours and then force-activated at Sunday NAV.
    function test_PoC_JuniorDepositMaturedIntoFrozenWindowIsStuck() public {
        vm.warp(FRI_REQUEST);
        _refreshMark();
        assertFalse(engine.isOracleFrozen(), "Friday 21:20 UTC is live");
        assertTrue(pool.canAcceptTrancheDeposits(false), "junior entry open while live");

        uint256 requestId = _requestAsyncDeposit(juniorVault, LP, DEPOSIT);
        assertEq(juniorVault.depositEpochStart(requestId), FRI_MIDNIGHT + 22 hours, "targets the 22:00 UTC epoch");
        assertEq(settlementToken.balanceOf(LP), 0, "escrow pulled");

        vm.warp(FRI_FROZEN);
        assertTrue(engine.isOracleFrozen(), "weekend freeze has begun");
        assertGe(juniorVault.currentLpEpoch(), requestId, "epoch matured");

        // 1. cannot cancel
        _expectCancelBlocked(juniorVault, requestId, TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);

        // 2. cannot activate: frozen-mode settlement defers entries; with nothing else to process it reverts outright
        vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
        pool.settleLpEpoch(0, 0);
        assertEq(juniorVault.claimableDepositRequest(requestId, LP), 0, "nothing claimable");
        assertEq(juniorVault.pendingDepositRequest(requestId, LP), DEPOSIT, "still pending in escrow");

        // 3. ... for the whole weekend. Retry mid-Saturday.
        vm.warp(FRI_MIDNIGHT + 1 days + 12 hours);
        assertTrue(engine.isOracleFrozen());
        _expectCancelBlocked(juniorVault, requestId, TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
        pool.settleLpEpoch(0, 0);
        assertEq(juniorVault.claimableDepositRequest(requestId, LP), 0);

        // 4. Sunday reopen: the first live settlement force-activates at Sunday NAV. The LP never had a choice.
        vm.warp(SUN_REOPEN);
        assertFalse(engine.isOracleFrozen(), "market live again Sunday 17:15 NY");
        _refreshMark();
        IHousePool.LpEpochSettlementResult memory result = pool.settleLpEpoch(0, 0);
        assertFalse(result.entriesDeferred, "entries activate once live");
        assertGt(juniorVault.claimableDepositRequest(requestId, LP), 0, "now claimable");
        assertEq(SUN_REOPEN - (FRI_MIDNIGHT + 22 hours), 47 hours + 15 minutes, "escrow locked ~47h with no exit");
    }

    /// The asymmetry: in the same frozen window an EXISTING holder can request and get funded a redemption (frozen fee
    /// applies) in a settlement that simultaneously reports the matured deposit as deferred.
    function test_PoC_HoldersCanStillRedeemInTheSameFrozenWindow() public {
        vm.warp(FRI_REQUEST);
        _refreshMark();
        uint256 requestId = _requestAsyncDeposit(juniorVault, LP, DEPOSIT);

        vm.warp(FRI_FROZEN);
        assertTrue(engine.isOracleFrozen());

        // Existing Junior holder (this contract, funded in setUp) redeems some shares during the freeze.
        uint256 shares = juniorVault.balanceOf(address(this)) / 10;
        assertGt(shares, 0);
        uint256 redeemId = juniorVault.requestRedeem(shares, address(this), address(this));
        vm.warp(pool.lpEpochStart(redeemId));
        assertTrue(engine.isOracleFrozen(), "still frozen at redeem maturity");

        uint256 balBefore = settlementToken.balanceOf(address(this));
        IHousePool.LpEpochSettlementResult memory result = pool.settleLpEpoch(0, 0);
        assertGt(result.juniorFundedAssets, 0, "redemption funded while frozen");
        assertTrue(result.entriesDeferred, "matured deposit deferred in the very same settlement");
        uint256 claimable = juniorVault.claimableRedeemRequest(redeemId, address(this));
        assertGt(claimable, 0);
        juniorVault.claimRedeem(redeemId, claimable, address(this), address(this));
        assertGt(settlementToken.balanceOf(address(this)), balBefore, "holder exited during freeze");

        // ... while the matured depositor is still locked in.
        _expectCancelBlocked(juniorVault, requestId, TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        assertEq(juniorVault.claimableDepositRequest(requestId, LP), 0);
        assertEq(juniorVault.pendingDepositRequest(requestId, LP), DEPOSIT);
    }

    /// Senior variant: same window, Senior request. The Senior hold-escape (lpEpochSettlementPaused) does not cover the
    /// recurring frozen window, so Senior is stuck too (different selector).
    function test_PoC_SeniorDepositMaturedIntoFrozenWindowIsStuck() public {
        vm.warp(FRI_REQUEST);
        _refreshMark();
        assertTrue(pool.canAcceptTrancheDeposits(true), "senior entry open while live");
        uint256 requestId = _requestAsyncDeposit(seniorVault, LP, DEPOSIT);

        vm.warp(FRI_FROZEN);
        assertTrue(engine.isOracleFrozen());
        assertFalse(pool.lpEpochSettlementPaused(), "no explicit hold - the Senior escape hatch is closed");

        _expectCancelBlocked(seniorVault, requestId, TrancheVault.TrancheVault__SeniorDepositCommitted.selector);
        vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
        pool.settleLpEpoch(0, 0);
        assertEq(seniorVault.claimableDepositRequest(requestId, LP), 0);
        assertEq(seniorVault.pendingDepositRequest(requestId, LP), DEPOSIT);
    }

    /// Admin variant: an owner/pauser pause() after maturity. Pause "defers deposit activation" (documented) but the
    /// matured depositor also cannot cancel for as long as the pause lasts. Unpause is owner-only, no expiry.
    function test_PoC_PauseAfterMaturityLocksMaturedDepositor() public {
        _refreshMark();
        uint256 requestId = _requestAsyncDeposit(juniorVault, LP, DEPOSIT);
        vm.warp(juniorVault.depositEpochStart(requestId));
        _refreshMark();

        pool.pause(); // owner (this contract)

        _expectCancelBlocked(juniorVault, requestId, TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
        pool.settleLpEpoch(0, 0);
        assertEq(juniorVault.claimableDepositRequest(requestId, LP), 0);

        // Only the owner can end it.
        pool.unpause();
        IHousePool.LpEpochSettlementResult memory result = pool.settleLpEpoch(0, 0);
        assertFalse(result.entriesDeferred);
        assertGt(juniorVault.claimableDepositRequest(requestId, LP), 0);
    }

    /// Degraded-mode variant (latch simulated via storage write; the real latch needs raw E < L after a close or
    /// liquidation). While degraded, every settlement reverts and the matured depositor cannot cancel; only the owner's
    /// clearDegradedMode() releases it.
    function test_PoC_DegradedModeLocksMaturedDepositor() public {
        _refreshMark();
        uint256 requestId = _requestAsyncDeposit(juniorVault, LP, DEPOSIT);
        vm.warp(juniorVault.depositEpochStart(requestId));
        _refreshMark();

        stdstore.target(address(engine)).sig("degradedMode()").checked_write(true);
        assertTrue(engine.degradedMode());

        _expectCancelBlocked(juniorVault, requestId, TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        vm.expectRevert(IHousePool.HousePool__DegradedMode.selector);
        pool.settleLpEpoch(0, 0);
        assertEq(juniorVault.claimableDepositRequest(requestId, LP), 0);

        // Owner clears degraded mode (simulated) -> activation proceeds.
        stdstore.target(address(engine)).sig("degradedMode()").checked_write(false);
        IHousePool.LpEpochSettlementResult memory result = pool.settleLpEpoch(0, 0);
        assertFalse(result.entriesDeferred);
        assertGt(juniorVault.claimableDepositRequest(requestId, LP), 0);
    }

}

Reproduction

git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
foundryup --version v1.5.1
# place MaturedDepositStuckWhileEntriesDeferred.t.sol in packages/perps/test/perps/spec/lp/
forge test --root packages/perps --match-path "*MaturedDepositStuck*" -vvv

Expected output

[PASS] test_PoC_JuniorDepositMaturedIntoFrozenWindowIsStuck()
[PASS] test_PoC_HoldersCanStillRedeemInTheSameFrozenWindow()
[PASS] test_PoC_SeniorDepositMaturedIntoFrozenWindowIsStuck()
[PASS] test_PoC_PauseAfterMaturityLocksMaturedDepositor()
[PASS] test_PoC_DegradedModeLocksMaturedDepositor()
Suite result: ok. 5 passed; 0 failed; 0 skipped

Notes

Verify the cancel predicate ignores deferral states: sed -n '851,860p' packages/perps/src/TrancheVault.sol; grep -n "oracleFrozen\|degradedMode\|paused()" packages/perps/src/TrancheVault.sol shows none feed cancelPendingDeposit.
Verify the deferral causes: sed -n '903,912p;1139,1145p' packages/perps/src/HousePool.sol.
Verify redemptions still fund while frozen: grep -n "_settlementFeeBps" packages/perps/src/HousePool.sol (the frozen flag selects a fee, it does not gate funding).
Line numbers refer to commit a82d4fe.
