# 56827 sc high missing global earmark reduction in forcerepay&#x20;

**Submitted on Oct 21st 2025 at 02:23:58 UTC by @Petrus for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56827
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

cumulativeEarmarked becomes inflated relative to the actual sum of user earmarks. Subsequent redeem calls (by the transmuter) can over-reduce totalDebt (since redeem subtracts from cumulativeEarmarked without per-account adjustments until \_sync). This could allow excessive debt reduction, leading to undercollateralization globally and incorrect survival ratios in \_sync. Because, during liquidation When an account has earmarked debt, \_liquidate calls \_forceRepay to repay it using the account's collateral. This reduces the account's debt (via \_subDebt) and earmarked but fails to reduce the global cumulativeEarmarked. This decouples per-account earmarks from the global total.

## Vulnerability Details

In the AlchemistV3.sol,

During liquidation,

```solidity
function _liquidate(uint256 accountId) internal returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying) { 

        // Query transmuter and earmark global debt 

        _earmark(); 

        // Sync current user debt before deciding how much needs to be liquidated 

        _sync(accountId); 

 

        Account storage account = _accounts[accountId]; 

 

        // Early return if no debt exists 

        if (account.debt == 0) { 

            return (0, 0, 0); 

        } 

 

        // In the rare scenario where 1 share is worth 0 underlying asset 

        if (IVaultV2(myt).convertToAssets(1e18) == 0) { 

            return (0, 0, 0); 

        } 

 

        // Calculate initial collateralization ratio 

        uint256 collateralInUnderlying = totalValue(accountId); 

        uint256 collateralizationRatio = collateralInUnderlying * FIXED_POINT_SCALAR / account.debt; 

 

        // If account is healthy, nothing to liquidate 

        if (collateralizationRatio > collateralizationLowerBound) { 

            return (0, 0, 0); 

        } 

 

        // Try to repay earmarked debt if it exists 

        uint256 repaidAmountInYield = 0; 

        if (account.earmarked > 0) { 

      @>      repaidAmountInYield = _forceRepay(accountId, account.earmarked); 

        } 

        // If debt is fully cleared, return with only the repaid amount, no liquidation needed, caller receives repayment fee 

        if (account.debt == 0) { 

            feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield); 

            TokenUtils.safeTransfer(myt, msg.sender, feeInYield); 

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

            feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield); 

            TokenUtils.safeTransfer(myt, msg.sender, feeInYield); 

            return (repaidAmountInYield, feeInYield, 0); 

        } 

    } 

```

when an account has earmarked debt, \_liquidate () calls \_forceRepay to repay it using the account's collateral.

In the \_forceRepay (),

```solidity
function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) { 

        if (amount == 0) { 

            return 0; 

        } 

        _checkForValidAccountId(accountId); 

        Account storage account = _accounts[accountId]; 

 

        // Query transmuter and earmark global debt 

        _earmark(); 

 

        // Sync current user debt before deciding how much is available to be repaid 

        _sync(accountId); 

 

        uint256 debt; 

 

        // Burning yieldTokens will pay off all types of debt 

        _checkState((debt = account.debt) > 0); 

 

        uint256 credit = amount > debt ? debt : amount; 

        uint256 creditToYield = convertDebtTokensToYield(credit); 

    @>    _subDebt(accountId, credit); 

 

        // Repay debt from earmarked amount of debt first 

        uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit; 

   @>     account.earmarked -= earmarkToRemove; 

 

        creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield; 

        account.collateralBalance -= creditToYield; 

 

        uint256 protocolFeeTotal = creditToYield * protocolFee / BPS; 

 

        emit ForceRepay(accountId, amount, creditToYield, protocolFeeTotal); 

 

        if (account.collateralBalance > protocolFeeTotal) { 

            account.collateralBalance -= protocolFeeTotal; 

            // Transfer the protocol fee to the protocol fee receiver 

            TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal); 

        } 

 

        if (creditToYield > 0) { 

            // Transfer the repaid tokens from the account to the transmuter. 

            TokenUtils.safeTransfer(myt, address(transmuter), creditToYield); 

        } 

        return creditToYield; 

    } 

 

```

It reduces the account's earmarked debt (account.earmarked -= earmarkToRemove;) and calls \_subDebt to reduce totalDebt, but it does not reduce the global cumulativeEarmarked variable, causing the global earmark total to remain inflated compared to the actual sum of user earmarks.

## Impact Details

This issue causes cumulativeEarmarked to overstate total earmarked debt, leading to excessive debt reductions in redeem calls and potential global undercollateralization drift during liquidations.

## soln

In the \_forceRepay () after account.earmarked -= earmarkToRemove;, add:

```solidity

uint256 globalEarmarkPaid = cumulativeEarmarked > earmarkToRemove ? earmarkToRemove : cumulativeEarmarked; 
cumulativeEarmarked -= globalEarmarkPaid; 
```

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L738>

## Proof of Concept

## Proof of Concept

Add the test in the src/test/AlchemistV3.t.sol

And run with forge test --match-test testLiquidation\_ForceRepay\_Does\_Not\_Reduce\_CumulativeEarmarked -vvvv

```solidity
function testLiquidation_ForceRepay_Does_Not_Reduce_CumulativeEarmarked() external {
    vm.startPrank(someWhale);
    IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
    vm.stopPrank();

    // Setup: Create a healthy account to maintain system collateralization
    vm.startPrank(yetAnotherExternalUser);
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
    alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
    vm.stopPrank();

    // Setup: Create the main account that will be liquidated
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
    alchemist.deposit(depositAmount, address(0xbeef), 0);
    uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    uint256 mintAmount = alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization;
    alchemist.mint(tokenIdFor0xBeef, mintAmount, address(0xbeef));
    vm.stopPrank();

    // Setup: Create a redemption to earmark debt
    vm.startPrank(anotherExternalUser);
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
    transmuterLogic.createRedemption(mintAmount);
    vm.stopPrank();

    // Advance to allow earmarking to occur
    vm.roll(block.number + 5_256_000 / 2);

    // Force earmarking by calling poke
    alchemist.poke(tokenIdFor0xBeef);

    // Get state before liquidation
    (uint256 collateralBefore, uint256 debtBefore, uint256 earmarkedBefore) = alchemist.getCDP(tokenIdFor0xBeef);
    uint256 cumulativeEarmarkedBefore = alchemist.cumulativeEarmarked();

    // Verify earmarked debt exists before liquidation
    require(earmarkedBefore > 0, "Earmarked debt should exist before liquidation");
    require(cumulativeEarmarkedBefore > 0, "cumulativeEarmarked should be > 0 before liquidation");

    // Trigger undercollateralization by dropping yield token price
    uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
    uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

    // Perform liquidation (which internally calls _forceRepay if earmarked debt exists)
    vm.startPrank(externalUser);
    alchemist.liquidate(tokenIdFor0xBeef);
    vm.stopPrank();

    // Get state after liquidation
    (uint256 collateralAfter, uint256 debtAfter, uint256 earmarkedAfter) = alchemist.getCDP(tokenIdFor0xBeef);
    uint256 cumulativeEarmarkedAfter = alchemist.cumulativeEarmarked();

    // Calculate what was repaid
    uint256 earmarkedRepaid = earmarkedBefore - earmarkedAfter;
    uint256 cumulativeEarmarkedReduction = cumulativeEarmarkedBefore >= cumulativeEarmarkedAfter 
        ? cumulativeEarmarkedBefore - cumulativeEarmarkedAfter 
        : 0;


    
    assertTrue(earmarkedRepaid > 0, "Account earmarked debt should be repaid during liquidation");
    assertTrue(
        cumulativeEarmarkedReduction < earmarkedRepaid,
        "BUG CONFIRMED: cumulativeEarmarked reduction is less than account earmarked repayment - global state not properly updated in _forceRepay"
    );
}


}

```

My POC brief explaination of the issue The Invariant Violation:

Before liquidation: earmarkedBefore = 90e22 and cumulativeEarmarkedBefore = 90e22 After liquidation: earmarkedAfter = 0 but cumulativeEarmarkedAfter = 90e22

The Bug: When \_forceRepay() is called during liquidation:

It correctly reduces account.earmarked from 90e22 to 0 It does not reduce cumulativeEarmarked - it stays at 90e22

What should happen: Both should decrease by the same amount (90e22), resulting in:

earmarkedAfter = 0 cumulativeEarmarkedAfter = 0

What actually happens:

earmarkedAfter = 0 cumulativeEarmarkedAfter = 90e22 (unchanged)

The test assertion assertTrue(cumulativeEarmarkedReduction < earmarkedRepaid) passes because 0 < 90e22 is true, which mathematically proves the global counter was not updated. Impact: The global cumulativeEarmarked is now inflated relative to the sum of individual user earmarks. On the next redemption or earmarking operation, this stale value will cause incorrect debt calculations.
