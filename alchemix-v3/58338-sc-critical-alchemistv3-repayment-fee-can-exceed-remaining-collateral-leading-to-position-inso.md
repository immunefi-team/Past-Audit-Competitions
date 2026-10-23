# 58338 sc critical alchemistv3 repayment fee can exceed remaining collateral leading to position insolvency

**Submitted on Nov 1st 2025 at 11:23:00 UTC by @T0nraq for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58338
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Summary

When liquidating an account with earmarked debt, the repayment fee may be larger than the account’s remaining collateral after the force repayment and protocol fee are deducted. The helper `_resolveRepaymentFee` caps the deduction from `collateralBalance`, but it still returns the raw, larger fee value. The caller then transfers this larger amount to the liquidator, overpaying them from protocol-held tokens (i.e., other users’ deposits), leaving the liquidated position at zero collateral and creating an accounting shortfall. This shortfall also can/is not be added to earmarkable debt and cannot be shared globally to restore accounting parity

## Vulnerability Details

### \_resolveRepaymentFee returns raw fee instead of the actually deducted amount

File: `src/AlchemistV3.sol`

```solidity
function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
    Account storage account = _accounts[accountId];
    fee = repaidAmountInYield * repaymentFee / BPS;               // raw fee
    // Deduct only what the account can afford (caps at remaining collateral)
    account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;
    emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);
    return fee;                                                   // BUG: returns raw fee, not deducted amount
}
```

### Liquidation uses the returned (raw) value for transfer

File: `src/AlchemistV3.sol`

```solidity
// If debt is fully cleared after _forceRepay:
feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
TokenUtils.safeTransfer(myt, msg.sender, feeInYield);  // sends raw fee even if more than remaining collateral
```

## Simple example

1. Start with a position that, after force repayment and protocol fee, has 4 MYT collateral left.
2. Repayment fee calculation yields 9 MYT.
3. `_resolveRepaymentFee` deducts only 4 MYT from the account (capped at remaining collateral) but still RETURNS 9 MYT.
4. Liquidation code transfers the returned 9 MYT to the liquidator.

Result: liquidator receives 9 MYT while the account could only afford 4 MYT. The 5 MYT shortfall is taken from protocol-held tokens (others’ deposits), position ends with 0 collateral, and the system is overpaying fees.

## Impact

* Overpayment to liquidators using protocol-held tokens (socializes losses across depositors).
* Liquidated positions end at zero collateral even when they could not afford the full fee.
* Accounting shortfall that compounds over multiple liquidations; can lead to insolvency risk and misleading system metrics.

## Recommended fix (return the actual deducted amount)

Return the amount actually deducted from `collateralBalance`, not the raw fee. Keep calculation simple and safe by capping first, then subtracting, then returning what was paid.

```solidity
function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 feePaid) {
    Account storage account = _accounts[accountId];
    uint256 fee = repaidAmountInYield * repaymentFee / BPS;                // raw
    feePaid = fee > account.collateralBalance ? account.collateralBalance : fee; // cap to affordability
    if (feePaid > 0) {
        account.collateralBalance -= feePaid;
    }
    emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, feePaid);
    return feePaid;                                                        // return what was actually deducted
}
```

## Proof of Concept

## Proof of Concept

* Test file: `src/test/AlchemistV3.t.sol`
* Add test function (snippet below): `testLiquidate_Undercollateralized_Position_With_Earmarked_Debt_Insufficient_Collateral_For_RepaymentFee()`

Run the POC:

```bash
forge test --match-test testLiquidate_Undercollateralized_Position_With_Earmarked_Debt_Insufficient_Collateral_For_RepaymentFee -vvvv
```

```solidity
    function testLiquidate_Undercollateralized_Position_With_Earmarked_Debt_Insufficient_Collateral_For_RepaymentFee() external {
        vm.startPrank(alOwner);
        alchemist.setRepaymentFee(500);
        vm.stopPrank();
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        // just ensureing global alchemist collateralization stays above the minimum required for regular liquidations
        // no need to mint anything
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        uint256 mintAmount = alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization;

        alchemist.mint(tokenIdFor0xBeef, mintAmount, address(0xbeef));
        vm.stopPrank();

        // Need to start a transmutator deposit, to start earmarking debt
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
        transmuterLogic.createRedemption(mintAmount);
        vm.stopPrank();

        uint256 transmuterPreviousBalance = IERC20(address(vault)).balanceOf(address(transmuterLogic));

        // skip to a future block. Lets say 100% of the way through the transmutation period (5_256_000 blocks)
        vm.roll(block.number + (5_256_000));

        // Earmarked debt should be 100% of the total debt
        (uint256 prevCollateral, uint256 prevDebt, uint256 earmarked) = alchemist.getCDP(tokenIdFor0xBeef);
        require(earmarked == prevDebt, "Earmarked debt should be 60% of the total debt");

        // modify yield token price via modifying underlying token supply
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
        uint256 liquidatorPrevUnderlyingBalance = IERC20(vault.asset()).balanceOf(address(externalUser));
        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdFor0xBeef);
        (uint256 depositedCollateral, uint256 debt,) = alchemist.getCDP(tokenIdFor0xBeef);

        uint256 repaymentFee = alchemist.convertDebtTokensToYield(earmarked) * 500 / BPS;

        console.log("compare", repaymentFee, depositedCollateral);
        console.log(repaymentFee > depositedCollateral);
        console.log(depositedCollateral == 0);

        vm.stopPrank();

        // ensure debt is reduced only by the repayment of max earmarked amount
        vm.assertApproxEqAbs(debt, prevDebt - earmarked, minimumDepositOrWithdrawalLoss);

        // ensure depositedCollateral is reduced only by the repayment of max earmarked amount
        // vm.assertApproxEqAbs(depositedCollateral, prevCollateral - alchemist.convertDebtTokensToYield(earmarked) - repaymentFee, minimumDepositOrWithdrawalLoss);

        // ensure assets is equal to repayment of max earmarked amount
        vm.assertApproxEqAbs(assets, alchemist.convertDebtTokensToYield(earmarked), minimumDepositOrWithdrawalLoss);

        // ensure liquidator fee is correct (i.e. only repayment fee, since only a repayment is done)
        vm.assertApproxEqAbs(feeInYield, repaymentFee, 1e18);
        vm.assertEq(feeInUnderlying, 0);

        // liquidator gets correct amount of fee, i.e. repayment fee > 0
        _validateLiquidiatorState(
            externalUser,
            liquidatorPrevTokenBalance,
            liquidatorPrevUnderlyingBalance,
            feeInYield,
            feeInUnderlying,
            assets,
            alchemist.convertDebtTokensToYield(earmarked)
        );

        vm.assertEq(alchemistFeeVault.totalDeposits(), 10_000 ether);

        // transmuter recieves the liquidation amount in yield token minus the fee
        vm.assertApproxEqAbs(
            IERC20(address(vault)).balanceOf(address(transmuterLogic)), transmuterPreviousBalance + alchemist.convertDebtTokensToYield(earmarked), 1e18
        );

        uint256 mytSharesDeposited = alchemist._mytSharesDeposited();
        uint256 actualVaultBalance = IERC20(address(vault)).balanceOf(address(alchemist)) + IERC20(address(vault)).balanceOf(address(transmuterLogic));
        
        console.log("=== ACCOUNTING BUG DEMONSTRATION ===");
        console.log("_mytSharesDeposited:", mytSharesDeposited);
        console.log("Actual vault balance:", actualVaultBalance);
        console.log("Discrepancy:", mytSharesDeposited - actualVaultBalance);

        // show that fee cut out of other users liqudity and now the entire system is insolvent
        assertTrue(mytSharesDeposited > actualVaultBalance, "BUG CONFIRMED: _mytSharesDeposited accounting is higher than actual vault balance after force repay");
    }
```
