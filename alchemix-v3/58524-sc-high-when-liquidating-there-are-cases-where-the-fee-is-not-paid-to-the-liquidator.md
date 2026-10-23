# 58524 sc high when liquidating there are cases where the fee is not paid to the liquidator&#x20;

**Submitted on Nov 3rd 2025 at 01:16:50 UTC by @bigbear1229 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58524
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Permanent freezing of unclaimed royalties
  * Since the Fee is not paid to the liquidator, liquidations may not proceed, which can disrupt the normal operation of the system.

## Description

## Brief/Intro

There are cases where the Fee is not paid to the liquidator during liquidation.

## Vulnerability Details

In the \_doLiquidation() function of AlchemistV3.sol, there is a part that pays the Fee to the liquidator.

```solidity
function _doLiquidation(uint256 accountId, uint256 collateralInUnderlying, uint256 repaidAmountInYield)
        internal
        returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying)
    {
        ...
	amountLiquidated = convertDebtTokensToYield(liquidationAmount);
        feeInYield = convertDebtTokensToYield(baseFee);

        // update user balance and debt
        account.collateralBalance = account.collateralBalance > amountLiquidated ? account.collateralBalance - amountLiquidated : 0;
        _subDebt(accountId, debtToBurn);

        // send liquidation amount - fee to transmuter
        TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);

        // send base fee to liquidator if available
        if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
        }

        ...
    }
```

In the \_doLiquidation() function, the amountLiquidated value is obtained from the calculateLiquidation() function. Looking at calculateLiquidation(), we can see that the grossCollateralToSeize value includes the Fee.

```solidity
function calculateLiquidation(
        uint256 collateral,
        uint256 debt,
        uint256 targetCollateralization,
        uint256 alchemistCurrentCollateralization,
        uint256 alchemistMinimumCollateralization,
        uint256 feeBps
    ) public pure returns (uint256 grossCollateralToSeize, uint256 debtToBurn, uint256 fee, uint256 outsourcedFee) {
        ...

        // gross collateral seize = net + fee
        grossCollateralToSeize = debtToBurn + fee;
    }
```

Looking at line 871 of AlchemistV3.sol, we can see that the amountLiquidated value, which already includes the Fee, has been subtracted from account.collateralBalance.

```solidity
// update user balance and debt
account.collateralBalance = account.collateralBalance > amountLiquidated ? account.collateralBalance - amountLiquidated : 0;
```

However, when paying the Fee to the liquidator, there is a check to see if the account.collateralBalance value is greater than the Fee that needs to be paid.

```solidity
if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
	TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
     }
```

Since the fee has already been subtracted from account.collateralBalance, this check becomes unnecessary. Ultimately, if account.collateralBalance is less than feeInYield, the fee cannot be paid.

## Impact Details

Since the Fee is not paid to the liquidator, liquidations may not proceed, which can disrupt the normal operation of the system.

\##Recommended mitigation steps

```solidity
function _doLiquidation(uint256 accountId, uint256 collateralInUnderlying, uint256 repaidAmountInYield)
        internal
        returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying)
    {
        ...
	amountLiquidated = convertDebtTokensToYield(liquidationAmount);
        feeInYield = convertDebtTokensToYield(baseFee);

        // update user balance and debt
<->  account.collateralBalance = account.collateralBalance > amountLiquidated ? account.collateralBalance - amountLiquidated : 0;
<+>	account.collateralBalance = account.collateralBalance > (amountLiquidated - feeInYield) ? account.collateralBalance - (amountLiquidated - feeInYield) : 0;
        _subDebt(accountId, debtToBurn);

        // send liquidation amount - fee to transmuter
        TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);

        // send base fee to liquidator if available
        if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
	    account.collateralBalance -= feeInYield;
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
        }

        ...
    }
```

## Proof of Concept

## Proof of Concept

To identify cases where the fee is not paid to the liquidator, we set the liquidatorFee in AlchemistV3 to a high value and demonstrated the issue. This issue can be verified by adding the following test\_CheckPairAddress() function to the AlchemistV3.t.sol file.

```solidity
function testLiquidate_Unable_To_Claim_Fee() external {
        vm.startPrank(alOwner);
        alchemist.setLiquidatorFee(9100);
        vm.stopPrank();

        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        // just ensureing global alchemist collateralization stays above the minimum required for regular liquidations
        // no need to mint anything
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        uint256 sharesBalance = IERC20(address(vault)).balanceOf(address(yetAnotherExternalUser));
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenIdFor0xBeef, alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization, address(0xbeef));
        vm.stopPrank();

        // modify yield token price via modifying underlying token supply
        (uint256 prevCollateral, uint256 prevDebt,) = alchemist.getCDP(tokenIdFor0xBeef);
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 59 bps or 5.9%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        // ensure initial debt is correct
        vm.assertApproxEqAbs(prevDebt, 180_000_000_000_000_000_018_000, minimumDepositOrWithdrawalLoss);

        // let another user liquidate the previous user position
        vm.startPrank(externalUser);
        uint256 liquidatorPrevTokenBalance = IERC20(address(vault)).balanceOf(address(externalUser));
        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdFor0xBeef);
        uint256 liquidatorAfterTokenBalance = IERC20(address(vault)).balanceOf(address(externalUser));

        console.log("ExpectedBaseFeeInYield : ", feeInYield);
        console.log("RealBaseFeeInYield : ", liquidatorAfterTokenBalance - liquidatorPrevTokenBalance);

        vm.stopPrank();
    }
```
