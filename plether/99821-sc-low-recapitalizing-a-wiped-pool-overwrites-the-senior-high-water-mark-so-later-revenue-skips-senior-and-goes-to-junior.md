# \#99821 [SC-Low] Recapitalizing a wiped pool overwrites the Senior high-water mark, so later revenue skips Senior and goes to Junior

**Submitted on Nov 15th 2026 at 12:22:30 UTC by @slowpoke for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99821
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

Brief / Intro

When both tranches are wiped (S = J = 0), Senior keeps its high-water mark (HWM), so the next revenue restores Senior before Junior. An owner recapitalizePool in that state overwrites the HWM with the recap amount and ignores the prior value. A recap smaller than the prior HWM therefore lowers Senior's restoration claim, and the difference goes to Junior.

The PoC shows the effect. With H = 100k, a 30k recap followed by 100k of revenue leaves Senior with 30k and Junior with 100k. With no recap, the same revenue gives Senior 100k and Junior 0. Senior holders end up worse off than if the owner had not recapitalized at all.

Vulnerability details

1. The wiped-pool branch overwrites the HWM. HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent L118–121:

solidity
if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
    state.waterfall.seniorPrincipal += remaining;
    state.waterfall.seniorHighWaterMark =
        recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining; // prior H dropped

HousePool._applyPendingClaimantBucketsLive (L1759–1776) passes the full pending recap as recapitalizationTargetAssets. The new HWM is therefore just the recap amount.

2. The rest of the waterfall preserves the HWM. In the same wiped state:

applyRevenueIntent fills the H − S gap and keeps H;
loss absorption keeps H.

ACCOUNTING_SPEC.md (L337–338) says "Losses do not reduce the Senior high-water mark" and that surplus restores Senior to its HWM before Junior. The recap branch is the only path that lowers H without Senior being paid.

3. Junior takes the lost priority. Once H = S, distributeRevenue sees no Senior gap, so all further revenue accrues to Junior. Junior supply never reaches zero because of the seed floor.

Impact details
Senior holders lose restoration priority. In the PoC, the drop is prior H − recap = 70k, and that value goes to Junior holders instead.
Senior holders can't avoid it. The seed floor keeps Senior supply above zero, and redeeming after a wipe pays nothing.
Recapitalizing makes Senior worse off. The more revenue arrives later, the larger the gap between the recap path and the no-recap path.
PoC (prior H = 100k, then 100k revenue)	Senior HWM	Senior	Junior
Recap 30k (bug)	100k → 30k	30k	100k
No recap (control)	100k	100k	0

Limits: both tranches must be fully wiped, and the owner must recapitalize below the prior HWM. The owner avoids the issue by recapping at least the prior HWM. The function's natspec describes the max(target, amount) rule, but that rule conflicts with the HWM rules in ACCOUNTING_SPEC.md.

Fix: in the wiped branch, set seniorHighWaterMark = max(priorHighWaterMark, recapitalizationTargetAssets, amount).

References
packages/perps/src/libraries/HousePoolPendingPreviewLib.sol L111–137 (wiped branch L118–121), L158–180 (applyRevenueIntent)
packages/perps/src/HousePool.sol L1759–1776
packages/perps/src/CfdEngine.sol L632–653 (recapitalizePool)
packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol L9
packages/perps/ACCOUNTING_SPEC.md L337–338

## Proof of Concept

// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

/// @notice Finding A PoC: recapitalizing a fully wiped pool overwrites the Senior high-water mark with the recap
///         target. A recap below the prior HWM lowers Senior's restoration priority, and later revenue that should
///         restore Senior goes to Junior instead.
///         Run: forge test --match-contract RecapLowersSeniorHwmPoC -vv
contract RecapLowersSeniorHwmPoC is BasePerpTest {

    uint256 constant RECAP = 30_000e6;
    uint256 constant REVENUE = 100_000e6;

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 100_000e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 100_000e6;
    }

    function _mark() internal {
        _ensureLaterOracleTime();
        vm.prank(address(router));
        engine.updateMarkPrice(1e8, SafeCast.toUint64(block.timestamp));
    }

    function _reconcile() internal {
        _mark();
        vm.prank(address(juniorVault));
        pool.reconcile();
    }

    /// @dev Both tranches lose all backing. Losses keep the Senior HWM (S = J = 0, H = 100k); seed supply stays live.
    function _wipeBothTranches() internal returns (uint256 priorH) {
        settlementToken.burn(address(pool), settlementToken.balanceOf(address(pool)));
        _reconcile();
        assertEq(pool.seniorPrincipal(), 0, "senior wiped");
        assertEq(pool.juniorPrincipal(), 0, "junior wiped");
        assertGt(seniorVault.totalSupply(), 0, "senior holders remain (seed floor)");
        assertGt(juniorVault.totalSupply(), 0, "junior holders remain (seed floor)");
        priorH = pool.seniorHighWaterMark();
        assertEq(priorH, 100_000e6, "losses keep the Senior HWM");
    }

    /// @dev Owner recapitalization through the real entry point.
    function _recap(
        uint256 amount
    ) internal {
        settlementToken.mint(address(this), amount);
        settlementToken.approve(address(engine), amount);
        engine.recapitalizePool(amount);
    }

    /// @dev Trading revenue as the Engine records it when traders lose to the pool (cash arrives with the call).
    function _revenue(
        uint256 amount
    ) internal {
        settlementToken.mint(address(pool), amount);
        vm.prank(address(engine));
        pool.recordClaimantInflow(
            amount, IHousePool.ClaimantInflowKind.Revenue, IHousePool.ClaimantInflowCashMode.CashArrived
        );
    }

    /// Bug: recap 30k after the wipe, then 100k of revenue.
    /// H drops 100k -> 30k, so the revenue skips Senior and goes to Junior.
    function test_A_RecapBelowPriorHwm_LowersSeniorHwm() public {
        uint256 priorH = _wipeBothTranches();

        _recap(RECAP);
        _reconcile();
        emit log_named_decimal_uint("Senior HWM before recap", priorH, 6);
        emit log_named_decimal_uint("Senior HWM after recap", pool.seniorHighWaterMark(), 6);
        assertEq(pool.seniorPrincipal(), RECAP, "recap restores S = 30k");
        assertEq(pool.seniorHighWaterMark(), RECAP, "HWM overwritten with the recap target");
        assertLt(pool.seniorHighWaterMark(), priorH, "HWM lowered below its prior value");

        _revenue(REVENUE);
        _reconcile();
        emit log_named_decimal_uint("bug:     Senior principal", pool.seniorPrincipal(), 6);
        emit log_named_decimal_uint("bug:     Junior principal", pool.juniorPrincipal(), 6);
        assertEq(pool.seniorPrincipal(), RECAP, "Senior gets none of the revenue");
        assertEq(pool.juniorPrincipal(), REVENUE, "all revenue goes to Junior");
    }

    /// Control: the same wipe and the same 100k of revenue with no recap at all.
    /// The HWM is preserved, so the revenue restores Senior first.
    function test_A_Control_NoRecap_SeniorRestoredFirst() public {
        _wipeBothTranches();

        _revenue(REVENUE);
        _reconcile();
        emit log_named_decimal_uint("control: Senior principal", pool.seniorPrincipal(), 6);
        emit log_named_decimal_uint("control: Junior principal", pool.juniorPrincipal(), 6);
        assertEq(pool.seniorHighWaterMark(), 100_000e6, "HWM preserved");
        assertEq(pool.seniorPrincipal(), REVENUE, "revenue restores Senior to its HWM");
        assertEq(pool.juniorPrincipal(), 0, "Junior gets nothing until Senior is whole");
    }
}

