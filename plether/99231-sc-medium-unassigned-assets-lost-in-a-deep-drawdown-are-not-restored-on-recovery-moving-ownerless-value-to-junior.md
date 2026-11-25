# \#99231 [SC-Medium] Unassigned Assets Lost in a Deep Drawdown Are Not Restored on Recovery, Moving Ownerless Value to Junior

**Submitted on Nov 12th 2026 at 13:59:28 UTC by @Hacken for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99231
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Vulnerability Details

unassignedAssets holds ownerless value, such as a recapitalization remainder, which only governance can assign through assignUnassignedAssets. LP claims are reconciled against distributable equity minus this bucket, so a loss hits senior and junior principal first. Once LP principal is gone, any further loss caps the bucket at the remaining distributable equity:
```solidity
// HousePoolReconcilePlanLib.sol

plan.state.unassignedAssets =
    state.unassignedAssets > snapshot.distributable ? snapshot.distributable : state.unassignedAssets;
```
The cap permanently lowers the stored balance, and no path restores it. On recovery, the reconcile treats all equity above the reduced bucket as LP value. It restores senior up to the high-water mark and gives the rest to junior:

```solidity
// HousePoolReconcilePlanLib.sol

uint256 distributableToClaims = snapshot.distributable > plan.state.unassignedAssets
    ? snapshot.distributable - plan.state.unassignedAssets
    : 0;
```
A loss and an equal recovery therefore end in a different allocation than the one they started from. The part of the bucket that was drawn down ends up with junior.

Example:

Start: S = J = 1,000 and unassigned = 500.
A loss of 2,300 leaves 200 of equity. LP principal goes to zero, and unassigned is capped to 200.
A full recovery of 2,300 is distributed as S = 1,000, J = 1,300, unassigned = 200.
Junior receives 300 of ownerless value.
A shallower loss of 1,800 never reaches the bucket and is fully symmetric.

## Impact Details
Ownerless value held for governance assignment is moved to junior holders after any drawdown deep enough to reach it. No LP loses funds. The issue is a misallocation of value that governance was meant to assign.

## Recommendation
Track the amount drawn down from unassignedAssets and restore it before recovery revenue reaches the waterfall. Alternatively, document that unassigned assets rank below LP claims on recovery.



## Proof of Concept

## Proof of Concept
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

/// @notice PoC: a drawdown deep enough to reach `unassignedAssets` permanently shrinks the bucket, and an equal
///         recovery hands the drawn-down ownerless value to junior instead of restoring the bucket.
/// @dev Uses BasePerpTest defaults, which match the release risk parameters (maxSkewRatio 0.4e18, baseCarryBps 500).
contract AuditUnassignedDrawdownPoC is BasePerpTest {

    address internal constant TRADER = address(0x7EAD);
    /// @dev Carry and senior coupon accrue over the few seconds between steps; allow a sub-cent tolerance.
    uint256 internal constant TOL = 0.01e6;

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0; // only the 1,000 USDG senior and 1,000 USDG junior seeds
    }

    function _mark(uint256 price) internal {
        // vm.getBlockTimestamp avoids the stale block.timestamp that via-IR can return after vm.warp.
        uint256 last = engine.lastMarkTime();
        uint256 nowTs = vm.getBlockTimestamp();
        uint256 t = (nowTs > last ? nowTs : last) + 1;
        vm.warp(t);
        vm.prank(address(router));
        engine.updateMarkPrice(price, uint64(t));
        vm.prank(address(juniorVault));
        pool.reconcile();
    }

    function test_PoC_drawdownMovesUnassignedAssetsToJunior() public {
        // 1. Start: S = J = 1,000 and 5,000 of ownerless value from a governance recapitalization
        uint256 recap = 5000e6;
        settlementToken.mint(address(this), recap);
        settlementToken.approve(address(engine), recap);
        engine.recapitalizePool(recap); // senior is at its high-water mark, so all of it stays unassigned
        vm.prank(address(juniorVault));
        pool.reconcile();

        assertEq(pool.seniorPrincipal(), 1000e6, "start senior");
        assertEq(pool.juniorPrincipal(), 1000e6, "start junior");
        assertEq(pool.seniorHighWaterMark(), 1000e6, "start senior HWM");
        assertEq(pool.unassignedAssets(), recap, "start unassigned");
        uint256 juniorBefore = pool.juniorPrincipal();
        uint256 unassignedBefore = pool.unassignedAssets();

        // 2. A trader opens a 2,500 USDG long at 1.00 (36% of depth, inside the 40% skew limit)
        uint256 notionalUsd = 2500;
        uint256 margin = notionalUsd * 1e6;
        settlementToken.mint(TRADER, margin);
        vm.startPrank(TRADER);
        settlementToken.approve(address(clearinghouse), margin);
        clearinghouse.deposit(TRADER, margin);
        vm.stopPrank();
        CfdTypes.Order memory order = CfdTypes.Order({
            account: TRADER,
            sizeDelta: notionalUsd * 1e18,
            marginDelta: margin,
            targetPrice: 1e8,
            commitTime: uint64(block.timestamp),
            commitBlock: uint64(block.number),
            orderId: 0,
            side: CfdTypes.Side.LONG,
            isClose: false
        });
        uint256 depth = pool.totalAssets();
        vm.prank(address(router));
        engine.processOrderTyped(order, 1e8, depth, uint64(block.timestamp));

        // 3. Drawdown: the mark moves 96% in the trader's favour, an LP loss of about 2,400.
        //    2,000 of LP principal absorbs the first 2,000; the remaining ~400 reaches the bucket and caps it.
        _mark(0.04e8);
        assertEq(pool.juniorPrincipal(), 0, "junior wiped");
        assertEq(pool.seniorPrincipal(), 0, "senior wiped");
        uint256 cappedUnassigned = pool.unassignedAssets();
        assertLt(cappedUnassigned, unassignedBefore, "drawdown capped the ownerless bucket");
        assertApproxEqAbs(unassignedBefore - cappedUnassigned, 400e6, TOL, "about 400 of ownerless value drawn down");

        // 4. Full recovery: the mark returns to the entry price, so the trader's gain disappears
        _mark(1e8);

        // Senior is restored only to its high-water mark; the bucket is NOT restored; junior gets the difference.
        assertApproxEqAbs(pool.seniorPrincipal(), 1000e6, TOL, "senior back to HWM");
        assertEq(pool.unassignedAssets(), cappedUnassigned, "bucket never restored on recovery");
        assertApproxEqAbs(pool.juniorPrincipal(), juniorBefore + 400e6, TOL, "junior received ~400 of ownerless value");

        // Same mark and same pool cash as in step 1, yet ~400 of ownerless value moved from the bucket to junior.
        uint256 movedToJunior = pool.juniorPrincipal() - juniorBefore;
        assertApproxEqAbs(movedToJunior, unassignedBefore - pool.unassignedAssets(), TOL, "value moved bucket -> junior");
        emit log_named_decimal_uint("unassigned before drawdown", unassignedBefore, 6);
        emit log_named_decimal_uint("unassigned after recovery ", pool.unassignedAssets(), 6);
        emit log_named_decimal_uint("junior before drawdown    ", juniorBefore, 6);
        emit log_named_decimal_uint("junior after recovery     ", pool.juniorPrincipal(), 6);
    }
}
```
