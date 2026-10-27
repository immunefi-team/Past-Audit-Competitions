# 57148 sc high mytsharesdeposited variable is not correctly updated during liquidations leading to wrong assumptions and incorrect bad debt calculation in the transmuter&#x20;

**Submitted on Oct 23rd 2025 at 21:36:35 UTC by @Tadev for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57148
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`_mytSharesDeposited` variable represents the total amount of MYT tokens held in positions in the AlchemistV3 contract.

This variable is incremented when a user deposits, and decremented when a user withdraws. It is also decremented by the fee paid to the `protocolFeeReceiver` when a user repays his debt through the `burn` or `repay` functions.

The problem arises because during liquidations, this variable is not updated even though MYT tokens are sent outside of the contract. This happens in multiple places, in `_liquidate`, `_forceRepay` and in `_doLiquidation` functions.

This leads to `_mytSharesDeposited` variable being overestimated forever. The more liquidations / force repay happen, the more the variable will be overestimated.

`_mytSharesDeposited` variable is used to compute the total underlying value in `getTotalUnderlyingValue` function. This function is used by the Transmuter in `claimRedemption` function to compute the `badDebtRatio` in order to determine the actual scaled transmuted amount.

Because `_mytSharesDeposited` variable is overestimated, the bad debt ratio will be inaccurate and underestimated. Hence, the system might not be able to adapt and resolve a bad debt situation.

## Vulnerability Details

The issue lies in the `_liquidate`, `_forceRepay` and `_doLiquidation` functions.

In `_liquidate` function, we can see 2 times the pattern:

```
            feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
            // @audit _mytSharesDeposited should be updated
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
            return (repaidAmountInYield, feeInYield, 0);
```

In `_forceRepay` function:

```
        if (account.collateralBalance > protocolFeeTotal) {
            account.collateralBalance -= protocolFeeTotal;
            // Transfer the protocol fee to the protocol fee receiver
            // @audit _mytSharesDeposited should be updated
            TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
        }

        if (creditToYield > 0) {
            // Transfer the repaid tokens from the account to the transmuter.
            // @audit _mytSharesDeposited should be updated
            TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
        }
```

And in `_doLiquidation` function:

```
        // update user balance and debt
        account.collateralBalance = account.collateralBalance > amountLiquidated ? account.collateralBalance - amountLiquidated : 0;
        _subDebt(accountId, debtToBurn);

        // send liquidation amount - fee to transmuter
        // @audit _mytSharesDeposited should be updated
        TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);

        // send base fee to liquidator if available
        if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
            // @audit _mytSharesDeposited should be updated
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
        }
```

Because `_mytSharesDeposited` is inflated as time passes and liquidations occur, the transmuter will be fed with an incorrect value in `claimRedemption` function:

```
        uint256 denominator = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) > 0
            ? alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance)
            : 1;
        uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10 ** TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator;

        uint256 scaledTransmuted = amountTransmuted;

        if (badDebtRatio > 1e18) {
            scaledTransmuted = amountTransmuted * FIXED_POINT_SCALAR / badDebtRatio;
        }
```

`denominator` will be inflated, `badDebtRatio` will be underestimated, and a potential needed scaling won't be applied.

## Impact Details

The impact of this issue is medium as it breaks internal accounting for `_mytSharesDeposited`, leading to incorrect bad debt calculation by the transmuter and impossibility for the transmuter to adapt and scale transmuted amount to normally resolve a bad dept situation.

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

This proof of concept highlights the path using `_doLiquidation`.

Please copy paste the following test in AlchemistV3.t.sol file:

```
    function testIncorrectMytBalanceAfterLiquidation() external {
        // deposit myt and mint debt tokens
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenIdFor0xBeef, alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization, address(0xbeef));
        vm.stopPrank();

        // update the price of myt
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        uint256 modifiedVaultSupply = (initialVaultSupply * 1200 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        // perform total liquidation
        alchemist.liquidate(tokenIdFor0xBeef);

        (uint256 collateral, uint256 debt, ) = alchemist.getCDP(tokenIdFor0xBeef);

        // debt and collateral are 0 (collateral is 1 because of rounding in `convertToShare` function)
        vm.assertEq(debt, 0);
        vm.assertApproxEqAbs(collateral, 0, 1);

        // mytBalance should be 0 but it's not
        uint256 mytBalance = alchemist.getTotalUnderlyingValue();
        vm.assertEq(mytBalance, 0);
    }
```

This test will revert with the following error:

```
[FAIL: assertion failed: 178571428571428571400000 != 0] testIncorrectMytBalanceAfterLiquidation() (gas: 1141967)
```

Indeed, `_mytSharesDeposited` is not 0 because it has not been decremented enough during liquidation.
