# 58061 sc high incorrect collateral and fee check in doliquidation allows liquidator to loose fee&#x20;

**Submitted on Oct 30th 2025 at 11:40:35 UTC by @Bizarro for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58061
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

In `_doLiquidation` function where the contract checks that `collateralBalance >= fee` before transferring the fee to the liquidator(msg.sender). This is incorrect because, in `calculateLiquidation`, the fee is already included in the `liquidationAmount` returned and is removed from the user's collateral.

## Vulnerability Details

The function double-counts the fee in the collateral balance check. The fee is included in the liquidationAmount(already removed from the user's balance), but the contract still checks if the remaining balance is sufficient to pay the fee again.

Path:

1. User position: `collateralBalance = 109`, `liquidationAmount = 100`, `fee = 10`
2. `_doLiquidation` removes 100 from `collateralBalance`, leaving 9.
3. The fee check `collateralBalance >= fee` fails (9<10).
4. Fee transfer to the liquidator does not occur, even though tokens are already removed from the user.

```solidity
    function _doLiquidation(uint256 accountId, uint256 collateralInUnderlying, uint256 repaidAmountInYield)
        internal
        returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying)
    {
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

        // update user balance and debt
@1>         account.collateralBalance = account.collateralBalance > amountLiquidated ? account.collateralBalance - amountLiquidated : 0;
        _subDebt(accountId, debtToBurn);

        // send liquidation amount - fee to transmuter
        TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);

        // send base fee to liquidator if available
@2>        if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
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

    function calculateLiquidation(
        uint256 collateral,
        uint256 debt,
        uint256 targetCollateralization,
        uint256 alchemistCurrentCollateralization,
        uint256 alchemistMinimumCollateralization,
        uint256 feeBps
    ) public pure returns (uint256 grossCollateralToSeize, uint256 debtToBurn, uint256 fee, uint256 outsourcedFee) {


        ---


        // gross collateral seize = net + fee
@>3        grossCollateralToSeize = debtToBurn + fee;
    }
```

* @3 = calculateLiquidation is returning (debtToBurn + fee, , fee,);
* @1 = collateral is reduced by amountLiquidated (debtToBurn + fee converted to yield token)
* @2 = after removing the fee from the collateral, collateralBalance >= feeInYield is required to transfer fee to the liquidator

## Impact Details

If users' collateralBalance is just above the liquidationAmount then removing liquidationAmount leaves only 9 tokens, the subsequent check (collateralBalance >= fee) fails (9 < 10), so the fee transfer to the liquidator does not occur and the funds are stuck in the protocol, even though the fee was already deducted. This means:

* The fee is reduced from the user's collateral but is never transferred to the liquidator.
* Liquidator funds are stuck in the contract.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L1244>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L852>

## Proof of Concept

## Proof of Concept

Paste this test in `test/AlchemistV3.t.sol` and run `forge test --mt testLiquidate_no_fee -vvv`

```solidity
function testLiquidate_no_fee() external {
        vm.startPrank(alchemist.admin());
        alchemist.setLiquidatorFee(9500);
        vm.stopPrank();

        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenIdFor0xBeef, alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / alchemist.minimumCollateralization(), address(0xbeef));
        vm.stopPrank();

        uint256 transmuterPreviousBalance = IERC20(address(vault)).balanceOf(address(transmuterLogic));

        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 900 bps or 9%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 900 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        uint256 liquidatorBalanceBefore =  IERC20(address(vault)).balanceOf(address(externalUser));
        vm.startPrank(externalUser);
        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdFor0xBeef);
        vm.stopPrank(); 
        uint256 liquidatorBalanceAfter = IERC20(address(vault)).balanceOf(address(externalUser));

        assertEq(liquidatorBalanceAfter - liquidatorBalanceBefore, 0);
    }
```

The test will pass showing that the liquidator will not receive any fee.
