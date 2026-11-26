# \#99824 [SC-High] Reconcile ordering lets same-pass revenue fill the Senior gap before the recap, sending Junior's revenue to unassigned assets

**Submitted on Nov 15th 2026 at 12:28:39 UTC by @slowpoke for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99824
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

Brief / Intro

When Senior is impaired (S < H), an owner recapitalization is queued as a pending bucket and applied at the next reconcile. If ordinary trading revenue arrives before that reconcile, both settle in one pass, and HousePool._reconcile distributes the revenue first. The revenue closes the Senior gap, and the recap then finds no gap and goes to unassigned assets (U), because recap never accrues to Junior. In two passes (recap, then revenue), the recap closes the gap and the revenue goes to Junior.

So the same events produce different allocations depending only on whether a reconcile ran in between. The PoC shows the effect with a 40k gap, a 40k recap and 2k of revenue:

two passes: Junior +2k, U 0;
one pass: Junior 0, U +2k.
Vulnerability details

1. Plan revenue is distributed before the pending recap. HousePool._reconcile (L1555–1599):

solidity
plan = HousePoolReconcilePlanLib.planReconcile(state, snapshot, pendingBucketAssets, ...); // L1568: revenue fills H − S
_setWaterfallState(plan.state.waterfall);
...
_applyPendingClaimantBucketsLive(settleableClaimantBuckets, ...);                         // L1598: recap applied last

planReconcile removes the pending buckets from distributable (L68–70) and passes the rest to distributeRevenue (L98), which fills H − S first.

2. Recap surplus never reaches Junior. In HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent (L123–136), with the gap already closed, gap == 0, so all of remaining goes to unassignedAssets.

3. The window is easy to hit. Recap is credited only at the next reconcile, and two things often delay it:

settleLpEpoch reverts while the pool is degraded (_requireWithdrawalsLive, L1868), which is the usual state after the impairment that called for the recap;
_emitLpEpochSettled reverts with NoLpEpochProgress (L1184) on quiet pools, which rolls back the reconcile.

Any trade in that window becomes same-pass revenue.

Impact details
Junior loses revenue. Junior's share of the revenue moves to U. The loss is capped at min(recap, revenue, gap). Total value is conserved.
U blocks entries. U > 0 blocks deposit requests and activation (HousePoolTrancheGateLib L28) until the owner calls assignUnassignedAssets(false, receiver).
Recovery skips existing Junior holders. assignUnassignedAssets(false, receiver) mints new Junior shares to one receiver. In the PoC the receiver ends up with more than 99.9% of Junior supply. Wiped Junior holders do not get their pro-rata share back.
PoC (gap 40k, recap 40k, revenue 2k)	Senior	Junior	Unassigned
Two passes (recap, then revenue)	+40k	+2k	0
One pass (revenue, then recap)	+40k	0	+2k

Limits: requires S < H, a pending recap, and revenue that hasn't been reconciled yet. The owner can avoid it by sizing the recap net of previewed revenue.

Fix: apply the pending recap bucket before distributing plan revenue, or let recap surplus continue into Junior once the Senior gap is closed.

References
packages/perps/src/HousePool.sol L1555–1599 (_reconcile; plan L1568, buckets L1598), L660–700 (assignUnassignedAssets), L1184, L1868
packages/perps/src/libraries/HousePoolReconcilePlanLib.sol L47–98 (pending subtraction L68–70, distributeRevenue L98)
packages/perps/src/libraries/HousePoolPendingPreviewLib.sol L111–137
packages/perps/src/libraries/HousePoolTrancheGateLib.sol L28

## Proof of Concept

// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Finding C PoC: when trading revenue and a pending recapitalization settle in the same reconcile pass,
///         revenue fills the Senior gap first and the recap lands in unassigned assets (U). Junior loses revenue it
///         receives when the two settle in separate passes.
///         Run: forge test --match-contract RecapOrderingUnassignedPoC -vv
contract RecapOrderingUnassignedPoC is BasePerpTest {

    address internal trader = address(0xA11CE);

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 100_000e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 100_000e6;
    }

    function _recap(
        uint256 amount
    ) internal {
        settlementToken.mint(address(this), amount);
        settlementToken.approve(address(engine), amount);
        engine.recapitalizePool(amount);
    }

    /// @dev Same `_reconcile` that `settleLpEpoch` runs.
    function _reconcile() internal {
        vm.prank(address(juniorVault));
        pool.reconcile();
    }

    /// @dev Senior impaired (S 60k < H 100k), Junior wiped but seed supply live.
    function _impairSenior() internal {
        settlementToken.burn(address(pool), 140_000e6);
        _reconcile();
        assertEq(pool.seniorPrincipal(), 60_000e6, "setup S");
        assertEq(pool.seniorHighWaterMark(), 100_000e6, "setup H");
        assertEq(pool.juniorPrincipal(), 0, "setup J");
        assertGt(juniorVault.totalSupply(), 0, "junior seed supply live");
    }

    /// @dev Ordinary trading: a LONG opens at 1.00 and closes at 1.10 (LONG profits when the basket falls),
    ///      so the pool keeps the trader's loss. With S > 0 this is plain distributable value, not a bucket.
    function _traderLosesToPool() internal returns (uint256 poolGain) {
        _fundTrader(trader, 10_000e6);
        uint256 before = pool.totalAssets();
        _open(trader, CfdTypes.Side.LONG, 20_000e18, 5000e6, 1e8);
        _ensureLaterOracleTime();
        _close(trader, CfdTypes.Side.LONG, 20_000e18, 1.1e8);
        poolGain = pool.totalAssets() - before;
        assertEq(pool.pendingTradingRevenueAssets(), 0, "revenue is not bucketed while S > 0");
    }

    /// Same events in the same order (recap, then trader loss). The only difference is whether a reconcile happens
    /// between them. The allocation should not depend on that.
    function test_C_SamePassRevenueDivertsJuniorShareToUnassigned() public {
        _impairSenior();
        uint256 gap = pool.seniorHighWaterMark() - pool.seniorPrincipal();

        uint256 snap = vm.snapshotState();

        // Path A — two passes: recap, reconcile, trader loss, reconcile.
        _recap(gap);
        _reconcile();
        uint256 gainA = _traderLosesToPool();
        _reconcile();
        uint256 sA = pool.seniorPrincipal();
        uint256 jA = pool.juniorPrincipal();
        uint256 uA = pool.unassignedAssets();

        vm.revertToState(snap);

        // Path B — one pass: recap, trader loss, a single reconcile.
        _recap(gap);
        uint256 gainB = _traderLosesToPool();
        _reconcile();
        uint256 sB = pool.seniorPrincipal();
        uint256 jB = pool.juniorPrincipal();
        uint256 uB = pool.unassignedAssets();

        console2.log("revenue to pool (A, B)", gainA, gainB);
        console2.log("two passes  S, J, U", sA, jA, uA);
        console2.log("one pass    S, J, U", sB, jB, uB);

        assertGt(gainA, 0, "trader loss reached the pool");
        assertApproxEqAbs(gainA, gainB, 10, "identical economics on both paths");
        assertEq(sA, sB, "Senior identical");
        assertEq(sA, pool.seniorHighWaterMark(), "Senior restored on both paths");
        assertApproxEqAbs(jA, gainA, 10, "two passes: Junior gets the revenue");
        assertEq(uA, 0, "two passes: nothing unassigned");
        assertEq(jB, 0, "one pass: Junior gets nothing");
        assertApproxEqAbs(uB, gainB, 10, "one pass: Junior's revenue became unassigned");
    }

    /// Follow-through: U blocks entries until the owner assigns it. With J == 0, the receiver of a Junior assignment
    /// takes essentially the whole Junior tranche; existing Junior holders do not get their pro-rata share back.
    function test_C_AssignmentHandsJuniorRevenueToReceiver() public {
        _impairSenior();
        _recap(pool.seniorHighWaterMark() - pool.seniorPrincipal());
        uint256 revenue = _traderLosesToPool();
        _reconcile();
        assertApproxEqAbs(pool.unassignedAssets(), revenue, 10, "precondition: revenue is in U");
        assertFalse(pool.canAcceptTrancheDeposits(false), "U blocks entries");

        address receiver = address(0xBEEF);
        uint256 supplyBefore = juniorVault.totalSupply();
        pool.assignUnassignedAssets(false, receiver);
        uint256 recvShares = juniorVault.balanceOf(receiver);
        uint256 supplyAfter = juniorVault.totalSupply();
        console2.log("junior supply before / receiver shares / after", supplyBefore, recvShares, supplyAfter);
        assertGt(recvShares * 1000, supplyAfter * 999, "receiver owns > 99.9% of Junior");
        assertApproxEqAbs(pool.juniorPrincipal(), revenue, 10, "Junior principal = diverted revenue");
    }
}

