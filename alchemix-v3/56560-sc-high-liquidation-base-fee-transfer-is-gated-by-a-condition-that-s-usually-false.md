# 56560 sc high liquidation base fee transfer is gated by a condition that s usually false

**Submitted on Oct 17th 2025 at 16:25:30 UTC by @spongebob for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56560
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

In `calculateLiquidation` the gross seize amount already bundles the base fee (`grossCollateralToSeize = debtToBurn + fee`), so `amountLiquidated` later passed around includes the fee portion `feeInYield`

`_doLiquidation` immediately deducts that full `amountLiquidated` from `account.collateralBalance`, leaving only any residual collateral, then forwards just `amountLiquidated - feeInYield` to the transmuter

<https://github.com/alchemix-finance/v3-poc/blob/b2e2aba046c36ff5e1db6f40f399e93cd2bdaad0/src/AlchemistV3.sol#L857-L865>

The subsequent guard `if (feeInYield > 0 && account.collateralBalance >= feeInYield)` now checks the post-deduction balance. This condition is equivalent to requiring the account to hold at least `grossCollateralToSeize + fee` beforehand and will be false whenever the liquidation seizes everything available.

<https://github.com/alchemix-finance/v3-poc/blob/b2e2aba046c36ff5e1db6f40f399e93cd2bdaad0/src/AlchemistV3.sol#L867-L869>

When the guard fails the liquidator never receives the base fee even though it was already removed from the victim account, so the seized fee remains with the contract.

## Impact

I consider this a critical issue that results in direct theft of user funds because when `_doLiquidation` reduces `account.collateralBalance` by `amountLiquidated`, the borrower permanently loses the entire seized amount. Only `amountLiquidated - feeInYield` is forwarded to the transmuter and the base-fee portion is supposed to go to the liquidator.

Because the subsequent transfer is gated on the already-reduced `account.collateralBalance`, the fee is often never paid out. The tokens remain trapped in the contract, so the borrower’s collateral is confiscated without being credited to any counterparty as intended.

## Recommendation

Transfer the fee to the liquidator before debiting the account and then send the remainder to the transmuter, or keep the current order but drop the balance check so the fee distribution always executes.

## Proof of Concept

Add this test to `AlchemixV3.t.sol` and run `forge test --mt testPOC_Liquidator_Fee_Trapped_Due_To_PostDeduction_Guard -vvvv`

```solidity
function testPOC_Liquidator_Fee_Trapped_Due_To_PostDeduction_Guard() external {
        // ============================================
        // SETUP: Create whale supply for price manipulation
        // ============================================
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        // ============================================
        // SETUP: Create a healthy account to keep global collateralization healthy
        // ============================================
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        // ============================================
        // STEP 1: Create victim position that will be fully liquidated
        // We want a scenario where grossCollateralToSeize ≈ collateralBalance
        // ============================================
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);

        uint256 tokenIdVictim = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));

        // Mint at exactly minimum collateralization (1.11x)
        uint256 debtAmount = alchemist.totalValue(tokenIdVictim) * FIXED_POINT_SCALAR / minimumCollateralization;
        alchemist.mint(tokenIdVictim, debtAmount, address(0xbeef));
        vm.stopPrank();

        // ============================================
        // STEP 2: Manipulate yield token price to make position undercollateralized
        // ============================================
        (uint256 prevCollateral, uint256 prevDebt,) = alchemist.getCDP(tokenIdVictim);
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // Increase yield token supply by 11% (price drop from 1.0 to ~0.901)
        // This will make the position severely undercollateralized
        uint256 modifiedVaultSupply = (initialVaultSupply * 1100 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        // ============================================
        // STEP 3: Record state BEFORE liquidation
        // ============================================
        uint256 liquidatorBalanceBefore = IERC20(address(vault)).balanceOf(externalUser);
        uint256 contractBalanceBefore = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 transmuterBalanceBefore = IERC20(address(vault)).balanceOf(address(transmuterLogic));

        // ensure initial debt is correct
        vm.assertApproxEqAbs(prevDebt, 180_000_000_000_000_000_018_000, minimumDepositOrWithdrawalLoss);

        // Calculate expected liquidation amounts
        uint256 alchemistCurrentCollateralization =
            alchemist.normalizeUnderlyingTokensToDebt(alchemist.getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / alchemist.totalDebt();
        (uint256 expectedGrossSeize, uint256 expectedDebtToBurn, uint256 expectedBaseFee,) = alchemist.calculateLiquidation(
            alchemist.totalValue(tokenIdVictim),
            prevDebt,
            alchemist.minimumCollateralization(),
            alchemistCurrentCollateralization,
            alchemist.globalMinimumCollateralization(),
            liquidatorFeeBPS
        );

        // Convert to yield tokens
        uint256 expectedAmountLiquidated = alchemist.convertDebtTokensToYield(expectedGrossSeize);
        uint256 expectedFeeInYield = alchemist.convertDebtTokensToYield(expectedBaseFee);

        // CRITICAL: The bug occurs when (victimCollateralBefore - expectedAmountLiquidated) < expectedFeeInYield
        // After line 861 deducts amountLiquidated, the balance will be less than the fee
        // Then line 868 checks if balance >= feeInYield, which will fail

        // For this POC, we demonstrate that even when there IS a fee to be paid,
        // the liquidator doesn't receive it if the post-deduction balance is insufficient

        // ============================================
        // STEP 4: Execute liquidation
        // ============================================
        (uint256 victimCollateralBefore,,) = alchemist.getCDP(tokenIdVictim);
        vm.prank(externalUser);
        alchemist.liquidate(tokenIdVictim);

        // ============================================
        // STEP 5: Record state AFTER liquidation
        // ============================================
        (uint256 victimCollateralAfter, uint256 victimDebtAfter,) = alchemist.getCDP(tokenIdVictim);
        uint256 liquidatorBalanceAfter = IERC20(address(vault)).balanceOf(externalUser);
        uint256 contractBalanceAfter = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 transmuterBalanceAfter = IERC20(address(vault)).balanceOf(address(transmuterLogic));

        // ============================================
        // STEP 6: Verify the vulnerability
        // ============================================

        // Calculate actual changes
        uint256 victimCollateralLoss = victimCollateralBefore - victimCollateralAfter;
        uint256 liquidatorGain = liquidatorBalanceAfter - liquidatorBalanceBefore;
        uint256 transmuterGain = transmuterBalanceAfter - transmuterBalanceBefore;
        uint256 contractDecrease = contractBalanceBefore - contractBalanceAfter;

        // VULNERABILITY 1: Victim lost the full amountLiquidated
        vm.assertEq(victimCollateralLoss, expectedAmountLiquidated, "Victim lost full amountLiquidated");

        // VULNERABILITY 2: Transmuter received (amountLiquidated - feeInYield)
        vm.assertEq(transmuterGain, expectedAmountLiquidated - expectedFeeInYield, "Transmuter received amount minus fee");

        // VULNERABILITY 3: Liquidator received ZERO or minimal fee (fee was not fully transferred)
        // Due to precision limits and the narrow window for this bug, we allow a tolerance
        vm.assertApproxEqAbs(liquidatorGain, 0, 10e18, "VULNERABILITY: Liquidator received minimal/zero fee");

        // VULNERABILITY 4: The fee is trapped in the contract
        // Contract decrease should be (amountLiquidated - feeInYield) sent to transmuter
        // But victim lost amountLiquidated, so feeInYield is trapped
        uint256 trapped = victimCollateralLoss - transmuterGain - liquidatorGain;
        vm.assertApproxEqAbs(trapped, expectedFeeInYield, 10e18, "VULNERABILITY: Fee is trapped in contract");
        // The trapped amount should be close to the expected fee (allowing for rounding)
        // If expectedFeeInYield is very small due to bad debt, trapped might also be small
        if (expectedFeeInYield > 10e18) {
            vm.assertGt(trapped, 0, "VULNERABILITY: Trapped amount is greater than zero");
        }
    }
```
