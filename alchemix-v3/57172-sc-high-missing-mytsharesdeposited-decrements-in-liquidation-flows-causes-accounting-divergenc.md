# 57172 sc high missing mytsharesdeposited decrements in liquidation flows causes accounting divergence

**Submitted on Oct 24th 2025 at 03:59:42 UTC by @Smartkelvin for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57172
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The AlchemistV3 contract fails to decrement the internal \_mytSharesDeposited variable when yield tokens are transferred out during liquidation operations (\_doLiquidation and \_forceRepay). This accounting error causes \_mytSharesDeposited to become inflated relative to the actual MYT token balance held by the contract. As a consequence, the deposit cap check becomes overly restrictive (preventing legitimate deposits even when capacity exists), the Total Value Locked (TVL) calculation via \_getTotalUnderlyingValue() becomes overstated, and the bad debt ratio used by the Transmuter for redemption payouts is understated, potentially allowing users to extract more value than entitled during system insolvency

## Vulnerability Details

The \_mytSharesDeposited variable is designed to track the total yield tokens deposited into CDPs, as indicated by its comment

```
  /// This is used to differentiate between tokens deposited into a CDP and balance of the contract
uint256 private _mytSharesDeposited;
```

Throughout the codebase, \_mytSharesDeposited is correctly updated in most token transfer operations:

Correct implementations:

deposit(): Increments \_mytSharesDeposited when tokens enter

withdraw(): Decrements \_mytSharesDeposited when tokens leave

burn(): Decrements \_mytSharesDeposited for protocol fee deduction

repay(): Decrements \_mytSharesDeposited for protocol fee deduction

redeem(): Decrements \_mytSharesDeposited for all outbound transfers

However, liquidation flows are missing these decrements:

```
  function _doLiquidation(uint256 accountId, uint256 collateralInUnderlying, uint256 repaidAmountInYield)
    internal
    returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying)
{
    Account storage account = _accounts[accountId];
    
    // ... liquidation calculation ...
    
    amountLiquidated = convertDebtTokensToYield(liquidationAmount);
    feeInYield = convertDebtTokensToYield(baseFee);
    
    // Tokens are deducted from collateralBalance (which was counted in _mytSharesDeposited)
    account.collateralBalance = account.collateralBalance > amountLiquidated 
        ? account.collateralBalance - amountLiquidated 
        : 0;
    _subDebt(accountId, debtToBurn);
    
    // Transfers OUT of contract
    TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);
    
    if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
        TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
    }
    
    // ❌ MISSING: _mytSharesDeposited -= (amountLiquidated - feeInYield);
    // ❌ MISSING: _mytSharesDeposited -= feeInYield;
    
    emit Liquidated(accountId, msg.sender, amountLiquidated + repaidAmountInYield, feeInYield, feeInUnderlying);
    return (amountLiquidated + repaidAmountInYield, feeInYield, feeInUnderlying);
}
```

```
  _forceRepay()

function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
    // ... validation and calculation ...
    
    uint256 credit = amount > debt ? debt : amount;
    uint256 creditToYield = convertDebtTokensToYield(credit);
    _subDebt(accountId, credit);
    
    // Tokens taken FROM collateralBalance (which was counted in _mytSharesDeposited)
    creditToYield = creditToYield > account.collateralBalance 
        ? account.collateralBalance 
        : creditToYield;
    account.collateralBalance -= creditToYield;
    
    uint256 protocolFeeTotal = creditToYield * protocolFee / BPS;
    
    if (account.collateralBalance > protocolFeeTotal) {
        account.collateralBalance -= protocolFeeTotal;
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
        // ❌ MISSING: _mytSharesDeposited -= protocolFeeTotal;
    }
    
    if (creditToYield > 0) {
        TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
        // ❌ MISSING: _mytSharesDeposited -= creditToYield;
    }
    return creditToYield;
}
```

The inflated \_mytSharesDeposited impacts three critical area

Deposit Cap Check (AlchemistV3.sol)

```
 function deposit(uint256 amount, address recipient, uint256 tokenId) external returns (uint256) {
    _checkState(_mytSharesDeposited + amount <= depositCap);
    // ... rest of function
}
```

TVL Calculation (AlchemistV3.sol)

```
 function _getTotalUnderlyingValue() internal view returns (uint256 totalUnderlyingValue) {
    uint256 yieldTokenTVLInUnderlying = convertYieldTokensToUnderlying(_mytSharesDeposited);
    return yieldTokenTVLInUnderlying;
}
```

Bad Debt Ratio in Transmuter (Transmuter.sol)

```
  uint256 denominator = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance);
uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10**TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator;
```

## Impact Details

Deposit Cap Bypass Prevention (DoS)

After liquidations occur, \_mytSharesDeposited remains inflated while the actual MYT balance decreases. This may causes the deposit cap check to incorrectly reject valid deposits

Overstated TVL Calculations

The \_getTotalUnderlyingValue() function directly uses the inflated \_mytSharesDeposited

```
 function _getTotalUnderlyingValue() internal view returns (uint256) {
    return convertYieldTokensToUnderlying(_mytSharesDeposited);
}
```

Understated Bad Debt Ratio in Transmuter The Transmuter uses getTotalUnderlyingValue() (which relies on \_mytSharesDeposited) to calculate the bad debt ratio for scaling redemption payouts:

```
 uint256 denominator = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance);
uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10**decimals / denominator;

if (badDebtRatio > 1e18) {
    scaledTransmuted = amountTransmuted * FIXED_POINT_SCALAR / badDebtRatio;
}
```

## Proof of Concept

## Proof of Concept

```
   add this below pocs to the AlchemistV3.t.sol test file



function testLiquidation_Breaking_smartkelvin_mytBug() external {
    vm.startPrank(someWhale);
    IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
    vm.stopPrank();

    // Set tight deposit cap
    address actualAdmin = alchemist.admin();
    vm.prank(actualAdmin);
    alchemist.setDepositCap(depositAmount * 2);

    // Fill up to exactly the cap
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
    alchemist.deposit(depositAmount, address(0xbeef), 0);
    uint256 tokenId1 = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    alchemist.mint(tokenId1, alchemist.totalValue(tokenId1) * FIXED_POINT_SCALAR / minimumCollateralization, address(0xbeef));
    vm.stopPrank();

    vm.startPrank(yetAnotherExternalUser);
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
    alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
    vm.stopPrank();

    // Now at cap - further deposits should fail
    vm.startPrank(address(0xdead));
    SafeERC20.safeApprove(address(vault), address(alchemist), 1e18);
    vm.expectRevert(); // Should fail - at cap
    alchemist.deposit(1e18, address(0xdead), 0);
    vm.stopPrank();

    // Liquidate to free up actual space
    uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
    uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

    uint256 balanceBefore = IERC20(address(vault)).balanceOf(address(alchemist));
    
    vm.startPrank(externalUser);
    (uint256 amountLiquidated, uint256 feeInYield,) = alchemist.liquidate(tokenId1);
    vm.stopPrank();

    uint256 balanceAfter = IERC20(address(vault)).balanceOf(address(alchemist));
    uint256 tokensFreed = balanceBefore - balanceAfter;
    
    console.log("Tokens freed from contract:", tokensFreed);
    console.log("Amount to transmuter:", amountLiquidated - feeInYield);
    
    // Even though actual tokens left the contract,
    // we still cannot deposit because _mytSharesDeposited wasn't decremented
    vm.startPrank(address(0xdead));
    uint256 attemptDeposit = tokensFreed / 2; // Try to use half the freed space
    SafeERC20.safeApprove(address(vault), address(alchemist), attemptDeposit);
    
    // This SHOULD succeed since actual tokens left, but WILL FAIL due to bug
    vm.expectRevert(); // Bug: deposit cap check uses stale _mytSharesDeposited
    alchemist.deposit(attemptDeposit, address(0xdead), 0);
    
    console.log("CONFIRMED: Cannot deposit despite actual space available");
    console.log("Root cause: _mytSharesDeposited not decremented in _doLiquidation");
    
    vm.stopPrank();
}


```
