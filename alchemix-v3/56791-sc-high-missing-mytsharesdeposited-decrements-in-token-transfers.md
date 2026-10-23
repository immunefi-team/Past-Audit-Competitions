# 56791 sc high missing mytsharesdeposited decrements in token transfers

**Submitted on Oct 20th 2025 at 17:47:43 UTC by @teoslaf1 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56791
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Summary

The `_mytSharesDeposited` state variable is not decremented when MYT tokens are transferred out of the AlchemistV3 contract during liquidations and force repayments. This causes the Total Value Locked (TVL) to be artificially inflated, leading to incorrect collateralization calculations, deposit cap issues, and bad debt ratio miscalculations.

## Vulnerability Details

### Root Cause

`_mytSharesDeposited` tracks the total MYT shares deposited in the contract and is used to calculate TVL:

```solidity
// Line 1244 in AlchemistV3.sol
function _getTotalUnderlyingValue() internal view returns (uint256 totalUnderlyingValue) {
    uint256 yieldTokenTVLInUnderlying = convertYieldTokensToUnderlying(_mytSharesDeposited);
    totalUnderlyingValue = yieldTokenTVLInUnderlying;
}
```

However, when tokens are transferred out during liquidations and force repayments, `_mytSharesDeposited` is not decremented, causing it to diverge from the actual token balance.

### Missing Decrements

#### 1. In `_doLiquidation()` (Lines 880 & 884)

```solidity
// Line 880 - Transfer to transmuter
TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);
// missing ->  _mytSharesDeposited -= (amountLiquidated - feeInYield);

// Line 884 - Transfer fee to liquidator
if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
    TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
    // missing -> _mytSharesDeposited -= feeInYield;
}
```

#### 2. In `_forceRepay()` (Lines 779 & 785)

```solidity
// Line 779 - Protocol fee transfer
if (account.collateralBalance > protocolFeeTotal) {
    account.collateralBalance -= protocolFeeTotal;
    TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
    //  missing -> _mytSharesDeposited -= protocolFeeTotal;
}

// Line 785 - Transmuter transfer
if (creditToYield > 0) {
    TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
    //  missing -> _mytSharesDeposited -= creditToYield;
}
```

## Impact

### Inflated TVL Calculations

* `getTotalUnderlyingValue()` returns inflated values
* Affects all collateralization ratio calculations throughout the protocol
* System appears healthier than reality

## Recommended Mitigation

Add `_mytSharesDeposited` decrements after each token transfer:

### Fix for `_doLiquidation()`

```diff
function _doLiquidation(uint256 accountId, uint256 collateralInUnderlying, uint256 repaidAmountInYield)
    internal
    returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying)
{
    // ... existing code ...

    // send liquidation amount - fee to transmuter
    TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);
+   _mytSharesDeposited -= (amountLiquidated - feeInYield);

    // send base fee to liquidator if available
    if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
        TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
+       _mytSharesDeposited -= feeInYield;
    }

    // ... rest of function ...
}
```

### Fix for `_forceRepay()`

```diff
function _forceRepay(uint256 accountId, uint256 credit) internal returns (uint256 creditToYield) {
    // ... existing code ...

    if (account.collateralBalance > protocolFeeTotal) {
        account.collateralBalance -= protocolFeeTotal;
        // Transfer the protocol fee to the protocol fee receiver
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
+       _mytSharesDeposited -= protocolFeeTotal;
    }

    if (creditToYield > 0) {
        // Transfer the repaid tokens from the account to the transmuter.
        TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
+       _mytSharesDeposited -= creditToYield;
    }

    return creditToYield;
}
```

## Proof of Concept

## Add this to AlchemistV3.t.sol

```solidity
function testPOC_MytSharesDeposited_Not_Decremented_On_Liquidate() external {
    vm.startPrank(someWhale);
    IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
    vm.stopPrank();
    
    uint256 depositAmt = 200_000e18;
    
    // Create healthy position for global collateralization
    vm.startPrank(yetAnotherExternalUser);
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmt * 2);
    alchemist.deposit(depositAmt, yetAnotherExternalUser, 0);
    vm.stopPrank();
    
    // Create position to be liquidated
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmt);
    alchemist.deposit(depositAmt, address(0xbeef), 0);
    uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    alchemist.mint(tokenId, alchemist.totalValue(tokenId) * FIXED_POINT_SCALAR / minimumCollateralization, address(0xbeef));
    vm.stopPrank();
    
    // Crash collateral value
    uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
    uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);
    
    // State before liquidation
    uint256 actualBalanceBefore = vault.balanceOf(address(alchemist));
    uint256 transmuterBalanceBefore = vault.balanceOf(address(transmuterLogic));
    
    // Liquidate
    vm.prank(externalUser);
    alchemist.liquidate(tokenId);
    
    // State after liquidation
    uint256 tvlAfter = alchemist.getTotalUnderlyingValue();
    uint256 actualBalanceAfter = vault.balanceOf(address(alchemist));
    uint256 tokensToTransmuter = vault.balanceOf(address(transmuterLogic)) - transmuterBalanceBefore;
    uint256 tokensTransferredOut = actualBalanceBefore - actualBalanceAfter;
    
    assertGt(tokensTransferredOut, 0, "Tokens transferred out");
    assertGt(tokensToTransmuter, 0, "Tokens sent to transmuter");
    assertGt(tvlAfter, actualBalanceAfter, "BUG: TVL inflated vs actual balance");
    
    console.log("TVL:", tvlAfter);
    console.log("Actual Balance:", actualBalanceAfter);
    console.log("Inflation:", tvlAfter - actualBalanceAfter);
}
```
