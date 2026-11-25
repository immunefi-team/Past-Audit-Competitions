# \#98680 [SC-Low] Post-wipeout recapitalization over-credits Senior beyond its high-water mark and inflates the HWM

**Submitted on Nov 9th 2026 at 13:58:57 UTC by @Minette6818 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98680
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

Brief/Intro

After a loss wipes both Senior and Junior principal to zero, a subsequent owner recapitalization is routed by the zero-principal "bootstrap" branch of `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent`, which credits the full settleable recapitalization to Senior principal and raises the Senior high-water mark (HWM) to the full pending target — ignoring a surviving Senior HWM. In the PoC, a $100,000 recap after a wipeout credits Senior $100,000 and sets HWM to $100,000, when the documented rule ("restore Senior only up to its high-water mark, remainder to unassigned") requires Senior $1,000, HWM $1,000, and ~$99,000 parked in `unassignedAssets`. ~$99,000 of the owner's recap is misrouted to Senior, and the inflated HWM subordinates all future Junior revenue to a $100,000 target instead of $1,000. The identical wiped state with a revenue inflow instead respects the HWM, proving the recapitalization path is the outlier.

Vulnerability Details

*Contracts / functions:*

- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol` — `applyClaimantRecapitalizationIntent` (bootstrap branch)
- `packages/perps/src/libraries/HousePoolReconcilePlanLib.sol` — `planReconcile`
- `packages/perps/src/HousePool.sol` — `_reconcile`, `_applyPendingClaimantBucketsLive`
- Contrast: `applyRevenueIntent` (same library, gap-restores correctly)

*Root cause.* The bootstrap branch fires whenever stored Senior + Junior principal are both zero at bucket-application time — including when they were zeroed moments earlier in the same reconcile by loss absorption (or in a prior reconcile), while a nonzero Senior HWM survives, because losses never reduce the HWM. It then credits the full settleable recap to Senior principal and sets HWM = max(full pending target, amount), ignoring the surviving HWM. The same function's non-bootstrap branch (and its own documentation) says a recapitalization restores Senior "only up to its high-water mark" with the remainder to unassigned, never to Junior.

*Trigger (normal recovery flow, no attacker action needed beyond holding Senior shares).* The loss-absorption step caps unassigned to distributable first (by design, ownerless value is first-loss), so any loss large enough to push distributable below stored unassigned zeroes both principals; the next recap then hits the bootstrap branch. Preconditions: (a) a large unassigned balance (created by a prior recap remainder into a healthy/impaired pool — the natural recap use case), (b) a large unreconciled loss, (c) an owner recap. The owner funds the recaps and the protocol misroutes them.

*Step-by-step (PoC trace, 6-decimal USDG atoms):*

1. Healthy pool: Senior = $1,000, Junior = $1,001,000, Senior HWM = $1,000.
2. Owner recapitalizes $500,000 into the healthy pool. Senior is unimpaired, so the whole $500,000 parks in `unassignedAssets`.
3. Catastrophic loss: pool cash drained to $124,000. Reconcile caps unassigned to $124,000 and wipes Senior = Junior = 0. HWM stays ~$1,000.
4. Owner recapitalizes $100,000. Settleable = $100,000.
5. *Actual (buggy):* Senior principal = $100,000, HWM = $100,000, Junior = 0, unassigned stays $124,000.
6. *Correct (per the library's own rule):* Senior = $1,000 (HWM), unassigned = $124,000 + $99,000 = $223,000, HWM = $1,000.
7. Misrouting: $98,999,990,868 atoms (~$99,000) captured by Senior, HWM inflated ~100x, subordinating all future Junior revenue to the $100,000 target.

Proof of Concept

PoC file: `packages/perps/test/perps/poc/PocPool_RecapBootstrapHwm.t.sol`

Run (from the repo root):
```
FOUNDRY_VIA_IR=false forge test --root packages/perps --jobs 1 --match-path "test/perps/poc/PocPool_RecapBootstrapHwm.t.sol"
```
Tests (2 of the 3 tests in this file belong to this finding; the 3rd covers the Junior-deposit finding):

- `test_RecapAfterWipeOverCreditsSeniorBeyondHwm` — builds the wiped state (healthy pool → $500,000 recap parks in unassigned → loss drains cash to $124,000 → reconcile wipes both principals, HWM survives), recaps $100,000, and asserts the buggy outcome: `seniorPrincipal == 100_000e6`, `seniorHighWaterMark == 100_000e6`, `juniorPrincipal == 0`, `unassignedAssets` unchanged at $124,000; emits the misrouted amount `recap2 - hwm0` = 98,999,990,868 atoms (~$99,000).
- `test_RevenueAfterWipeRespectsHwm` — the control: the identical wiped state with a $100,000 revenue inflow instead routes Senior = $1,000 (HWM), Junior = $99,000, HWM unchanged, unassigned untouched. Revenue respects the HWM; recapitalization does not.

Expected result: both tests pass, confirming the misrouting and the revenue/recap asymmetry.

PoC summary in words: deploy the perps test harness, recap a healthy pool so the remainder parks in unassigned, burn pool cash to simulate a catastrophic loss, reconcile to wipe both tranches while the HWM survives, recap again, and observe Senior credited the full amount with the HWM inflated to match — then repeat with revenue instead of recap and observe the correct HWM-respecting routing.

Recommended Fix

In `applyClaimantRecapitalizationIntent`, take the bootstrap branch only when there is no surviving protected entitlement, i.e. only when `seniorHighWaterMark == 0`. When HWM > 0, gap-restore Senior up to the HWM and send the remainder to unassigned — the same routing the existing non-bootstrap branch already implements.

References

- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol` (`applyClaimantRecapitalizationIntent`, `applyRevenueIntent`)
- `packages/perps/src/libraries/HousePoolReconcilePlanLib.sol` (`planReconcile`)
- `packages/perps/src/HousePool.sol` (`_reconcile`, `_applyPendingClaimantBucketsLive`)
- `packages/perps/ACCOUNTING_SPEC.md` (recapitalization / waterfall routing rules)

*Eligibility note (why this is NOT a known issue):* Cecuro MED-12 is about revenue being stranded in unassigned when recapitalization restores a claimant path (fix: recompute revenue continuation after recap). This finding is the reverse direction and a different function: the recapitalization branch itself over-credits Senior past the HWM. MED-12's fix comment says "Recapitalization remains senior-first" (priority), not senior-only-beyond-HWM. No existing test covers the zero-principal bootstrap case — the existing `test_ImpairedSeniorFrozenExitScalesHwmByFundedPrincipal` recap case has Senior principal > 0, so it takes the gap-restore branch. This finding is not in the Cecuro report, the SC Audit Studio pre-audit, or the AI scan findings.

## Proof of Concept

Proof of Concept

Step-by-step

1. Start from a healthy pool: Senior = $1,000, Junior = $1,001,000, Senior HWM = $1,000.
2. Owner recapitalizes $500,000 into the healthy pool. Senior is unimpaired, so the whole $500,000 parks in `unassignedAssets`.
3. Catastrophic loss: pool cash is drained to $124,000. Reconcile caps unassigned to $124,000 and wipes Senior = Junior = 0. The Senior HWM survives at ~$1,000.
4. Owner recapitalizes $100,000. Settleable = $100,000.
5. Buggy result: Senior principal = $100,000, HWM = $100,000, Junior = 0, unassigned stays $124,000.
6. Correct result (per the library's own rule): Senior = $1,000 (HWM), unassigned = $223,000, HWM = $1,000.
7. Misrouting: ~$99,000 (98,999,990,868 atoms) captured by Senior, HWM inflated ~100x.
8. Control: repeat the identical wiped state with a $100,000 revenue inflow instead of a recap — Senior is restored only up to the $1,000 HWM, Junior gets $99,000, HWM unchanged. Revenue respects the HWM; recapitalization does not.

How to run

Place this file at `packages/perps/test/perps/poc/PocPool_RecapBootstrapHwm.t.sol` in the repo at commit `a82d4fef`, then run:

`FOUNDRY_VIA_IR=false forge test --root packages/perps --jobs 1 --match-path "test/perps/poc/PocPool_RecapBootstrapHwm.t.sol"`

Expected result: 3 tests pass, confirming the misrouting and the revenue/recap asymmetry.

Note: 2 of the 3 tests in this file belong to this finding (`test_RecapAfterWipeOverCreditsSeniorBeyondHwm` and `test_RevenueAfterWipeRespectsHwm`). The 3rd test (`test_MaturedJuniorDepositLockedDuringSettlementHold`) belongs to a separate finding and will be submitted separately.

PoC code
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";

/// @title PoC: post-wipeout recapitalization over-credits Senior beyond its HWM
/// @notice Finding 1 (Medium): `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent`
///         bootstraps the FULL settleable recapitalization into senior principal and raises the
///         senior high-water mark to the FULL pending target whenever stored senior + junior
///         principal are both zero at application time -- even when a nonzero senior HWM survives
///         from before the wipeout. The documented rule for an impaired senior is "restore only up
///         to its high-water mark, remainder to unassigned; recapitalization never accrues to
///         junior". The bootstrap branch ignores the surviving HWM, so a recap that should restore
///         senior to ~$1,000 (its HWM) and park ~$99,000 in unassignedAssets instead credits
///         senior the full $100,000 and inflates its HWM 100x, subordinating all future junior
///         revenue to the inflated target. The identical state with a REVENUE inflow instead
///         respects the HWM (test 2), proving the recap path is the outlier.
///         The trigger is a normal recovery flow: recap remainder parks in unassigned, a loss
///         wipes both principals (unassigned absorbs the shortfall first by design), then a second
///         recap arrives. No attacker action is needed beyond owning senior shares; the owner
///         funds the recaps, the protocol misroutes them.
///         NOTE: verified passing (3/3 tests in this file) with:
///         FOUNDRY_VIA_IR=false forge test --root packages/perps --jobs 1 --match-path test/perps/poc/PocPool_RecapBootstrapHwm.t.sol -vvv
contract PocPoolRecapBootstrapHwmTest is BasePerpTest {

    address internal constant ALICE = address(0xA11CE);
    address internal constant BOB = address(0xB0B);

    function _reconcile() internal {
        vm.prank(address(seniorVault));
        pool.reconcile();
    }

    /// @dev Builds the wiped state: healthy pool -> recap remainder parks in unassigned ->
    ///      catastrophic loss (pool cash burned, same state a real trader-gain drain produces)
    ///      -> reconcile wipes both principals, HWM survives, unassigned capped to what is left.
    /// @return hwm0 Senior HWM from before the wipe (the entitlement a recap must restore up to).
    /// @return unassignedAfterWipe Unassigned assets surviving the wipe (= remaining distributable).
    function _setupWipedPool() internal returns (uint256 hwm0, uint256 unassignedAfterWipe) {
        // STEP A: owner recapitalizes a HEALTHY pool. Senior is unimpaired, so the whole
        // remainder parks in unassignedAssets (lib-documented recap routing).
        uint256 recap1 = 500_000e6;
        settlementToken.mint(address(this), recap1);
        settlementToken.approve(address(engine), recap1);
        engine.recapitalizePool(recap1);
        _reconcile();
        assertEq(pool.unassignedAssets(), recap1, "recap1 remainder should park in unassigned");
        hwm0 = pool.seniorHighWaterMark();
        assertEq(pool.seniorPrincipal(), hwm0, "senior unimpaired after recap1");
        assertGt(pool.juniorPrincipal(), 0, "junior alive after recap1");

        // STEP B: catastrophic loss. Burn pool cash down to 124,000 USDG; reconcile caps
        // unassigned to the remaining distributable and wipes both principals junior-first.
        uint256 physicalBefore = pool.totalAssets();
        uint256 physicalAfterBurn = 124_000e6;
        settlementToken.burn(address(pool), physicalBefore - physicalAfterBurn);
        _reconcile();
        assertEq(pool.seniorPrincipal(), 0, "wipe must zero senior principal");
        assertEq(pool.juniorPrincipal(), 0, "wipe must zero junior principal");
        assertEq(pool.seniorHighWaterMark(), hwm0, "loss must preserve senior HWM");
        unassignedAfterWipe = pool.unassignedAssets();
        assertEq(unassignedAfterWipe, physicalAfterBurn, "unassigned capped to remaining distributable");
    }

    function test_RecapAfterWipeOverCreditsSeniorBeyondHwm() public {
        (uint256 hwm0, uint256 unassignedAfterWipe) = _setupWipedPool();

        // STEP C: owner recapitalizes again ($100,000). Documented rule: restore senior only
        // up to hwm0 (~$1,000), remainder ~$99,000 to unassigned, HWM unchanged.
        uint256 recap2 = 100_000e6;
        settlementToken.mint(address(this), recap2);
        settlementToken.approve(address(engine), recap2);
        engine.recapitalizePool(recap2);
        _reconcile();

        assertEq(pool.pendingRecapitalizationAssets(), 0, "recap2 must settle");
        // BUG: the zero-principal bootstrap branch credits the FULL settleable amount to
        // senior and raises HWM to the FULL pending target, ignoring the surviving hwm0.
        assertEq(pool.seniorPrincipal(), recap2, "BUG: senior credited full recap, not up to HWM");
        assertEq(pool.seniorHighWaterMark(), recap2, "BUG: HWM inflated to full recap target");
        assertEq(pool.juniorPrincipal(), 0, "junior stays wiped");
        assertEq(
            pool.unassignedAssets(),
            unassignedAfterWipe,
            "BUG: recap remainder did NOT park in unassigned"
        );

        emit log_named_uint("senior HWM before wipe (correct recap cap)", hwm0);
        emit log_named_uint("senior principal after recap2 (actual, buggy)", pool.seniorPrincipal());
        emit log_named_uint("senior HWM after recap2 (actual, buggy)", pool.seniorHighWaterMark());
        emit log_named_uint("misrouted to senior (actual - correct cap)", recap2 - hwm0);
        emit log_named_uint(
            "unassigned after recap2 (actual)", pool.unassignedAssets()
        );
        emit log_named_uint(
            "unassigned after recap2 (correct would be)",
            unassignedAfterWipe + (recap2 - hwm0)
        );
    }

    function test_RevenueAfterWipeRespectsHwm() public {
        (uint256 hwm0, uint256 unassignedAfterWipe) = _setupWipedPool();

        // Same wiped state, but the inflow is REVENUE instead of recapitalization. The
        // revenue router respects the surviving HWM: senior restored only up to hwm0,
        // the remainder accrues to junior, HWM unchanged.
        uint256 revenue = 100_000e6;
        settlementToken.mint(address(pool), revenue);
        vm.prank(address(engine));
        pool.recordClaimantInflow(
            revenue, IHousePool.ClaimantInflowKind.Revenue, IHousePool.ClaimantInflowCashMode.CashArrived
        );
        _reconcile();

        assertEq(pool.pendingTradingRevenueAssets(), 0, "revenue must settle");
        assertEq(pool.seniorPrincipal(), hwm0, "revenue restores senior only up to HWM");
        assertEq(pool.juniorPrincipal(), revenue - hwm0, "revenue remainder accrues to junior");
        assertEq(pool.seniorHighWaterMark(), hwm0, "revenue must not inflate HWM");
        assertEq(pool.unassignedAssets(), unassignedAfterWipe, "revenue must not touch unassigned");
    }

    /// @notice Finding 2 (Low): a matured-but-unactivated JUNIOR deposit has no exit while the
    ///         LP settlement hold is active. The senior cancel guard explicitly opens when
    ///         `lpEpochSettlementPaused` is true (and the release path is documented as
    ///         deliberately callable while paused); the junior guard has no equivalent clause, so
    ///         a junior depositor whose epoch matured under the hold can neither cancel nor
    ///         activate until the owner releases it. Funds are not lost, only locked, and the
    ///         hold is owner-controlled -- hence Low, not Medium.
    function test_MaturedJuniorDepositLockedDuringSettlementHold() public {
        uint256 juniorAmount = 10_000e6;
        uint256 seniorAmount = 10_000e6;

        uint256 juniorReq = _requestDeposit(juniorVault, ALICE, juniorAmount);
        uint256 seniorReq = _requestDeposit(seniorVault, BOB, seniorAmount);

        // Both requests target the same epoch; warp to its start so both are matured.
        uint256 activation = juniorVault.depositEpochStart(juniorReq);
        uint256 seniorActivation = seniorVault.depositEpochStart(seniorReq);
        if (seniorActivation > activation) {
            activation = seniorActivation;
        }
        vm.warp(activation);

        pool.pauseLpEpochSettlement();

        // Junior: matured + healthy + not rejected + not wiped + not impaired -> cancel reverts.
        vm.prank(ALICE);
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        juniorVault.cancelPendingDeposit(juniorReq);

        // Senior: the same state cancels fine under the hold and releases its reservation.
        uint256 bobBefore = settlementToken.balanceOf(BOB);
        vm.prank(BOB);
        uint256 refunded = seniorVault.cancelPendingDeposit(seniorReq);
        assertEq(refunded, seniorAmount, "senior must refund under the hold");
        assertEq(settlementToken.balanceOf(BOB), bobBefore + seniorAmount, "senior funds returned");
        assertEq(pool.reservedSeniorDepositAssets(), 0, "senior reservation released");
        assertGt(
            juniorVault.pendingDepositRequest(juniorReq, ALICE), 0, "junior deposit still locked in escrow"
        );
    }

    function _requestDeposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal returns (uint256 requestId) {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
    }
}
```
