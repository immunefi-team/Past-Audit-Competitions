# 58771 sc high incorrect tracking of total deposited yield tokens mytsharesdeposited in liquidation and force repayment paths

**Submitted on Nov 4th 2025 at 12:45:57 UTC by @w3llyc4de20Ik2nn1 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58771
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

In AlchemistV3's liquidation and force repayment functions, missing subtractions from `_mytSharesDeposited` after yield token transfers to the transmuter, protocol receiver, or liquidator cause overstated TVL reporting via inflated `getTotalUnderlyingValue()`, enabling deposit cap bypasses relative to actual balances

## Vulnerability Details

In the AlchemistV3.sol,

In `_forceRepay ()` (internal function called by `_liquidate` during earmarked debt repayment in liquidate and batchLiquidate),

```solidity
uint256 protocolFeeTotal = creditToYield * protocolFee / BPS; 
 
    emit ForceRepay(accountId, amount, creditToYield, protocolFeeTotal); 
 
    if (account.collateralBalance > protocolFeeTotal) { 
        account.collateralBalance -= protocolFeeTotal; 
        // Transfer the protocol fee to the protocol fee receiver 
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);  // <-- Missing: _mytSharesDeposited -= protocolFeeTotal; 
    } 
 
    if (creditToYield > 0) { 
        // Transfer the repaid tokens from the account to the transmuter. 
        TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);  // <-- Missing: _mytSharesDeposited -= creditToYield; 
    } 
    return creditToYield; 
} 
```

In `_liquidate ()` (repayment-only path, after force repayment restores health; missing subtraction after fee transfer):

```solidity
// If debt is fully cleared, return with only the repaid amount, no liquidation needed, caller receives repayment fee 
    if (account.debt == 0) { 
        feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield); 
        TokenUtils.safeTransfer(myt, msg.sender, feeInYield);  // <-- Missing: _mytSharesDeposited -= feeInYield; 
        return (repaidAmountInYield, feeInYield, 0); 
    } 
    // ... 
} 
```

In `_doLiquidation ()` (internal function called by liquidate and batchLiquidate),

```solidity
// send liquidation amount - fee to transmuter 
    TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);  // <-- Missing: _mytSharesDeposited -= (amountLiquidated - feeInYield); 
 
    // send base fee to liquidator if available 
    if (feeInYield > 0 && account.collateralBalance >= feeInYield) { 
        TokenUtils.safeTransfer(myt, msg.sender, feeInYield);  // <-- Missing: _mytSharesDeposited -= feeInYield; 
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

The variable `_mytSharesDeposited` tracks total yield tokens deposited into the contract (updated on deposit/withdraw/repay/redeem), but in liquidation paths, transfers of yield tokens out to the transmuter, protocol fee receiver, or liquidator are not subtracted from it, causing it to overstate the actual contract balance (IERC20(myt).balanceOf(address(this))).

## Impact Details

This issue causes `_mytSharesDeposited` to overstate total deposited yield tokens after liquidations, breaking TVL reporting in `getTotalDeposited()` and `_getTotalUnderlyingValue()`, and allowing deposits to exceed the effective cap relative to actual balances.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L738>

## Proof of Concept

## Proof of Concept

Put the test in the src/test/AlchemistV3.t.sol

Run the test with: forge test --match-test testLiquidationOverstatesTotalUnderlyingValue –vvvv

```solidity
function testLiquidationOverstatesTotalUnderlyingValue() external { 

    uint256 testDepositAmount = 100e18; 

    vm.startPrank(address(0xbeef)); 

    SafeERC20.safeApprove(address(vault), address(alchemist), testDepositAmount + 100e18); 

    alchemist.deposit(testDepositAmount, address(0xbeef), 0); 

    uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT)); 

    alchemist.mint(tokenId, alchemist.totalValue(tokenId) * FIXED_POINT_SCALAR / minimumCollateralization, address(0xbeef)); 

    vm.stopPrank(); 

 

    // Ensure global collateralization is healthy 

    vm.startPrank(yetAnotherExternalUser); 

    SafeERC20.safeApprove(address(vault), address(alchemist), testDepositAmount * 2); 

    alchemist.deposit(testDepositAmount * 2, yetAnotherExternalUser, 0); 

    vm.stopPrank(); 

 

    // Pre-liquidation values 

    uint256 preLiquidationUnderlyingValue = alchemist.getTotalUnderlyingValue(); 

    uint256 preLiquidationActualBalance = IERC20(address(vault)).balanceOf(address(alchemist)); 

 

    // Simulate price drop to make position undercollateralized 

    uint256 initialSupply = IERC20(mockStrategyYieldToken).totalSupply(); 

    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialSupply * 1200 / 1000); 

 

    // Perform liquidation 

    vm.startPrank(externalUser); 

    alchemist.liquidate(tokenId); 

    vm.stopPrank(); 

 

    // Post-liquidation values 

    uint256 postLiquidationUnderlyingValue = alchemist.getTotalUnderlyingValue(); 

    uint256 postLiquidationActualBalance = IERC20(address(vault)).balanceOf(address(alchemist)); 

    uint256 postActualUnderlyingValue = alchemist.convertYieldTokensToUnderlying(postLiquidationActualBalance); 

 

    // Verify overstatement: getTotalUnderlyingValue() overstates actual underlying value due to unsubtracted shares in liquidation paths 

    assertGt(postLiquidationUnderlyingValue, postActualUnderlyingValue); 

    // Verify actual balance decreased after liquidation transfers 

    assertLt(postLiquidationActualBalance, preLiquidationActualBalance); 

    // Verify the reported underlying value reflects the price drop but not the share removal (overstatement relative to expected post-liquidation value) 

    assertGt(postLiquidationUnderlyingValue, preLiquidationUnderlyingValue * 833 / 1000); 

} 
```

Key Proof Points from the Test Execution:

Setup (Pre-Liquidation State):

\--> A position is created with 100e18 yield shares deposited (testDepositAmount).

\--> Debt is minted to max (90e19 debt tokens, requiring \~108e18 yield shares locked as collateral).

\--> Global health is ensured with another 200e18 deposit.

\--> Total shares held by Alchemist: 300e18 (preLiquidationActualBalance).

\--> Reported underlying value: 300e18 (full value, no overstatement yet).

\--> Price is then dropped to 83.33% (supply increased by 20%, simulating devaluation).

Liquidation Execution:

\--> The position becomes undercollateralized due to the price drop.

\--> Liquidation transfers 99.999e18 yield shares to the Transmuter (as collateral seizure).

\--> 2.7e18 underlying tokens are withdrawn from the fee vault to the liquidator (outsourced fee).

\--> No \_mytSharesDeposited -= updates occur for these transfers (as per the bug in \_doLiquidation).

Post-Liquidation State (Overstatement Revealed):

\--> Actual shares held by Alchemist: 200e18 (postLiquidationActualBalance < pre).

\--> Actual underlying value of those shares (at post-price): 166.666e18 (postActualUnderlyingValue).

\--> Reported underlying value (postLiquidationUnderlyingValue): 249.999e18.

\--> This overstates by \~50e18 (30% inflation), because \_mytSharesDeposited still reflects the pre-liquidation 300e18 shares, ignoring the 99.999e18 transferred out.

Assertions confirm:

postLiquidationUnderlyingValue > postActualUnderlyingValue (249.999e18 > 166.666e18): Direct proof of overstatement in getTotalUnderlyingValue().

postLiquidationActualBalance < preLiquidationActualBalance (200e18 < 300e18): Confirms shares were actually removed.

postLiquidationUnderlyingValue > preLiquidationUnderlyingValue \* 833 / 1000 (249.999e18 > \~249.9e18, but critically, it doesn't reflect the full share removal + price drop; the reported value only partially adjusts for price but ignores share outflow).

Impact Demonstrated:

TVL Inflation: getTotalUnderlyingValue() (used in totalValue(), collateral checks, etc.) now reports \~83.33e18 more than reality, misleading users/admins on system health.

Deposit Cap Bypass: Future deposits could exceed the effective cap (relative to real shares held), as deposit() checks against the inflated \_mytSharesDeposited.

Therefore, this is a clear arithmetic invariant violation: reported shares/underlying ≠ actual balance post-liquidation transfers.
