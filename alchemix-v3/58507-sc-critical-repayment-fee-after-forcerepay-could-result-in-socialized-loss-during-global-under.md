# 58507 sc critical repayment fee after forcerepay could result in socialized loss during global undercollateralization

**Submitted on Nov 2nd 2025 at 21:19:12 UTC by @algizsec for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58507
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

When a user gets liquidated, the system first attempts to heal his position by clearing earmarked debt (by forcing a repay). When all debt is cleared a `repaymentFee` should be paid. In case the remaining `account.collateralBalance` cannot cover the full `repaymentFee`, the fee gets socialized between the other users, even though the system could be globally undercollateralized. Instead of covering the liquidation fees from the `feeVault`, `AlchemistV3` uses its own MYT balance to pay the liquidator. Subsequently, this could result in insolvency for withdraws/redemptions from other users.

Furthermore, the behaviour is inconsistent with the way the system pays `feeInYield` during an actual liquidation. During actual liquidation, `feeInYield` is only paid to liquidators if there is enough `account.collateralBalance`. There is an additional bug in this part of the logic which checks the `account.collateralBalance >= feeInYield` after the balance has already been reduced with the fee.

## Vulnerability Details

1. Inside `_resolveRepaymentFee()`:

```solidity
 fee = repaidAmountInYield * repaymentFee / BPS;
account.collateralBalance -= fee > account.collateralBalance ?account.collateralBalance : fee;
```

However, after `_resolveRepaymentFee()` we have:

```solidity
            feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
```

which deduces MYT from the overall balance, but total effective `mytSharesDeposited` remain higher. (Note: there is another unrelated issue where the `mytSharesDeposited` does not get subtracted during `resolveRepaymentFee()`. )

In this scenario, the user could not cover the repayment fee. Even if the system is globally under the `minimumCollateralization`, instead of covering the fee using the `feeVault`, the fees are covered by the other users. This could result in insolvency for late withdraws / redemptions (unless the protocol injects liquidity from the `feeVault`).

2. Inside `_doLiquidation()`:

```solidity
        amountLiquidated = convertDebtTokensToYield(liquidationAmount);         //@audit liquidatedAmount = debtToBurn + baseFee
        feeInYield = convertDebtTokensToYield(baseFee);

        // update user balance and debt
        account.collateralBalance = account.collateralBalance > amountLiquidated ? account.collateralBalance - amountLiquidated : 0;
        _subDebt(accountId, debtToBurn);

        // send liquidation amount - fee to transmuter
        TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);

        // send base fee to liquidator if available
        if (feeInYield > 0 && account.collateralBalance >= feeInYield) {    //@audit if collateralBalance < feeInYield, no fee is sent
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
        }
```

`amountLiquidated` includes `debtToBurn` and `baseFee` (liquidator's fee). The code first deducts the entire `amountLiquidated` from the user, then sends the net (`amountLiquidated - feeInYield`) to the Transmuter. The remaining "slice" corresponds to `feeInYield` and is supposed to be paid to the liquidator. However, the payment of that fee is incorrectly conditioned on the user's post-deduction from `collateralBalance`. If after the deduction, the user's balance is `< feeInYield`, the liquidator gets nothing, even though the fee slice was already taken from the user's funds.

The unpaid fee remains stranded inside the Alchemist contract - accounting-wise removed from the user, but not transferred to anyone.

## Impact Details

During market stress,(e.g when MYT price falls), liquidations can yield repayment fees, which might not be payable by the liquidated users. The shortfall is socialized, which would be acceptable if the system is above the `minimumCollateralization`. If the system is globally undercollateralized, it will let the users bear the losses instead of covering them from the `feeVault`.

Furthermore, the way the system pays liquidators is inconsistent depending on if there is earmark or not. In some edge scenarios without earmarks, liquidators could end up receiving no fees due to a flaw in the logic.

## References

\[resolve repayment fees 1] (<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L825-L826>)

\[resolve repayment fees 2] (<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L903-L904>)

\[feeInYield flaw] (<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L871-L880>)

## Proof of Concept

The PoC demonstrates how a price drop of 10% for MYT could easily result in a socialized loss of magnitude the repayment fee for a force repaid position (2% in this case). For bigger positions, this amount could be significant, amplifying the effect on other users. The PoC finally demonstrates the inability of a user without any debt to withdraw his collateral fully.

1. Add the following test to the `AlchemistV3.t.sol`:

```solidity
function testLiquidate_Undercollateralized_Position_With_Earmarked_Debt_Liquidation_10Percent_Yield_Price_Drop() external {
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        address admin = alchemist.admin();
        vm.startPrank(admin);
        alchemist.setMinimumCollateralization(uint256(FIXED_POINT_SCALAR * FIXED_POINT_SCALAR) / 90e16);
        alchemist.setRepaymentFee(200);
        vm.stopPrank();

        // A user deposits without minting any borrow
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), 100e18);
        alchemist.deposit(100e18, yetAnotherExternalUser, 0);
        uint256 tokenIdForAnotherUser = AlchemistNFTHelper.getFirstTokenId(yetAnotherExternalUser, address(alchemistNFT));

        vm.stopPrank();

        //Beef deposits and borrows the maximum allowed 90%
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), 10000e18 + 100e18);
        alchemist.deposit(10000e18, address(0xbeef), 0);
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        uint256 mintAmount = alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization;

        alchemist.mint(tokenIdFor0xBeef, mintAmount, address(0xbeef));
        vm.stopPrank();

        // Start transmuter redemption to earmark 100% of debt
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
        transmuterLogic.createRedemption(mintAmount);
        vm.stopPrank();

        // // 100% transmutation period elapsed - all debt is earmarked
        vm.roll(block.number + 5_256_000);

        console.log("================== BEFORE PRICE DROP ======================");
        (uint256 collateralBefore, uint256 debtBefore, uint256 earmarkedBefore) = alchemist.getCDP(tokenIdFor0xBeef);
        console.log("Collateral (MYT shares):    ", collateralBefore);
        console.log("Collateral value (ETH):     ", alchemist.totalValue(tokenIdFor0xBeef));
        console.log("Debt:                       ", debtBefore);
        console.log("Earmarked:                  ", earmarkedBefore);

        // MYT price drops by 10%  ->  99% of collateral gets force repayed -> debt is cleared, but collateralBalance not enough to cover all the repayment fee
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        uint256 modifiedVaultSupply = initialVaultSupply * 11000 / 10000; // 10.0% price drop of MYT
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        (collateralBefore, debtBefore,) = alchemist.getCDP(tokenIdFor0xBeef);
        uint256 alchemistCurrentCollateralization = alchemist.normalizeUnderlyingTokensToDebt(alchemist.getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / alchemist.totalDebt(); 
        uint256 alchemistGlobalCollateralization = alchemist.globalMinimumCollateralization();
        
        console.log("================== BEFORE LIQUIDATION ======================");


        //Show that system is globally undercollateralized -> fees should be payed by vault
        console.log("Current alchemist collateralization ratio: ", alchemistCurrentCollateralization);
        console.log("Global alchemist collateralization ratio:  ", alchemistGlobalCollateralization);
        
        uint256 liquidatorBalanceBefore = IERC20(address(vault)).balanceOf(externalUser);

        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdFor0xBeef);
        
        uint256 liquidatorBalanceAfter = IERC20(address(vault)).balanceOf(externalUser);
        uint256 actualFeeReceived = liquidatorBalanceAfter - liquidatorBalanceBefore;

        (uint256 collateralAfter, uint256 debtAfter,) = alchemist.getCDP(tokenIdFor0xBeef);

        console.log("");
        console.log("================== AFTER LIQUIDATION ======================");
        console.log("Assets liquidated:           ", assets);
        console.log("Fee calculated (feeInYield): ", feeInYield);
        console.log("Fee in underlying (outsourcedFee): ", feeInUnderlying);
        console.log("===POSITION===");
        console.log("Collateral left after liquidation: ", collateralAfter);
        console.log("Debt after liquidation:            ", debtAfter);

        // Calculate how much mytSharesDeposited will be left if they were correctly subtracted during the CDP balance deduction
        uint256  mytSharesSubtractedAfterFix = collateralBefore > (assets + feeInYield) ? (assets + feeInYield) : collateralBefore;

        console.log("Total MYT balance:                                    ", alchemist.getTotalDeposited());
        console.log("Total mytSharesDeposited if fix is applied:           ", alchemist._mytSharesDeposited() - mytSharesSubtractedAfterFix);

        // The user with no borrow cannot withdraw all of his collateral since it was partially used as repayment fee
        vm.startPrank(yetAnotherExternalUser);
        vm.expectRevert();
        alchemist.withdraw(100e18, yetAnotherExternalUser, tokenIdForAnotherUser);
        vm.stopPrank();
    }
```

2. Make `_mytSharesDeposited` in AlchemistV3 public for the sake of the PoC
3. Run the test with: `forge test --mt testLiquidate_Undercollateralized_Position_With_Earmarked_Debt_Liquidation_10Percent_Yield_Price_Drop -vv`
4. Output:

```bash
Logs:
  ================== BEFORE PRICE DROP ======================
  Collateral (MYT shares):     10000000000000000000000
  Collateral value (ETH):      10000000000000000000000
  Debt:                        9000000000000000000900
  Earmarked:                   9000000000000000000900
  ================== BEFORE LIQUIDATION ======================
  Current alchemist collateralization ratio:  1020202020202020200
  Global alchemist collateralization ratio:   1111111111111111111
  
  ================== AFTER LIQUIDATION ======================
  Assets liquidated:            9900000000000000010889
  Fee calculated (feeInYield):  198000000000000000217
  Fee in underlying (outsourcedFee):  0
  ===POSITION===
  Collateral left after liquidation:  0
  Debt after liquidation:             0
  Total MYT balance:                                     1999999999999988894
  Total mytSharesDeposited if fix is applied:            100000000000000000000
```
