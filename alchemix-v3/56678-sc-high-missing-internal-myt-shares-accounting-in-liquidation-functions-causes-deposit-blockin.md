# 56678 sc high missing internal myt shares accounting in liquidation functions causes deposit blocking and protocol insolvency risk through inflated tvl calculations

**Submitted on Oct 19th 2025 at 09:54:59 UTC by @godwinudo for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56678
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The AlchemistV3.sol contract maintains an internal accounting variable `_mytSharesDeposited` to track total MYT shares deposited as collateral and enforce deposit caps. During liquidations, MYT tokens are transferred out of the contract to the transmuter and liquidators, but `_mytSharesDeposited` is never decremented. This causes the internal tracking to become permanently inflated relative to the actual MYT balance. The inflated tracking blocks legitimate deposits prematurely and, more critically, inflates the Total Value Locked (TVL) calculations used in liquidation logic, making the protocol appear healthier than reality. This can prevent full liquidations during some scenarios, allowing bad debt to accumulate and potentially leading to protocol insolvency.

## Vulnerability Details

The AlchemistV3 contract declares a private state variable that tracks the total MYT shares held as user collateral:

```solidity
/// @dev Total yield tokens deposited
/// This is used to differentiate between tokens deposited into a CDP and balance of the contract
uint256 private _mytSharesDeposited;
```

This variable serves two critical purposes in the protocol. First, it enforces the deposit cap to prevent the contract from accepting more collateral than intended. Second, it calculates the total value locked in the protocol, which directly influences liquidation decisions about whether positions should be partially or fully liquidated.

To understand the issue, we need to first see how the accounting should work. Let me show you the deposit function where `_mytSharesDeposited` is correctly incremented:

```solidity
function deposit(uint256 amount, address recipient, uint256 tokenId) external returns (uint256) {
    _checkArgument(recipient != address(0));
    _checkArgument(amount > 0);
    _checkState(!depositsPaused);
    _checkState(_mytSharesDeposited + amount <= depositCap);

    // Only mint a new position if the id is 0
    if (tokenId == 0) {
        tokenId = IAlchemistV3Position(alchemistPositionNFT).mint(recipient);
        emit AlchemistV3PositionNFTMinted(recipient, tokenId);
    } else {
        _checkForValidAccountId(tokenId);
    }

    _accounts[tokenId].collateralBalance += amount;

    // Transfer tokens from msg.sender now that the internal storage updates have been committed.
    TokenUtils.safeTransferFrom(myt, msg.sender, address(this), amount);
    _mytSharesDeposited += amount;  // ✓ CORRECTLY INCREMENTED

    emit Deposit(amount, tokenId);

    return convertYieldTokensToDebt(amount);
}
```

Notice how after the MYT tokens are transferred into the contract, the code increments `_mytSharesDeposited` by the same amount. This maintains perfect synchronization between the actual token balance and the internal accounting.

Now let me show you the withdrawal function where the accounting is correctly decremented:

```solidity
function withdraw(uint256 amount, address recipient, uint256 tokenId) external returns (uint256) {
    _checkArgument(recipient != address(0));
    _checkForValidAccountId(tokenId);
    _checkArgument(amount > 0);
    _checkAccountOwnership(IAlchemistV3Position(alchemistPositionNFT).ownerOf(tokenId), msg.sender);
    _earmark();

    _sync(tokenId);

    uint256 lockedCollateral = convertDebtTokensToYield(_accounts[tokenId].debt) * minimumCollateralization / FIXED_POINT_SCALAR;
    _checkArgument(_accounts[tokenId].collateralBalance - lockedCollateral >= amount);

    _accounts[tokenId].collateralBalance -= amount;

    // Assure that the collateralization invariant is still held.
    _validate(tokenId);

    // Transfer the yield tokens to msg.sender
    TokenUtils.safeTransfer(myt, recipient, amount);
    _mytSharesDeposited -= amount;  // ✓ CORRECTLY DECREMENTED

    emit Withdraw(amount, tokenId, recipient);

    return amount;
}
```

After transferring MYT tokens out of the contract to the recipient, the code decrements `_mytSharesDeposited` by that same amount. The internal ledger stays perfectly synchronized with reality.

Now let me show you where the accounting breaks down. The liquidation process has two internal functions that transfer MYT tokens out of the contract but completely fail to update `_mytSharesDeposited`. First, let's look at the `_forceRepay` function, which is called during liquidations to handle early repayment of earmarked debt:

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
    _subDebt(accountId, credit);

    // Repay debt from earmarked amount of debt first
    uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
    account.earmarked -= earmarkToRemove;

    creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield;
    account.collateralBalance -= creditToYield;

    uint256 protocolFeeTotal = creditToYield * protocolFee / BPS;

    emit ForceRepay(accountId, amount, creditToYield, protocolFeeTotal);

    if (account.collateralBalance > protocolFeeTotal) {
        account.collateralBalance -= protocolFeeTotal;
        // Transfer the protocol fee to the protocol fee receiver
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
        // _mytSharesDeposited NOT decremented 
    }

    if (creditToYield > 0) {
        // Transfer the repaid tokens from the account to the transmuter.
        TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
        // _mytSharesDeposited NOT decremented
    }
    return creditToYield;
}
```

Look carefully at those two `TokenUtils.safeTransfer` calls. The function sends `protocolFeeTotal` amount of MYT to the protocol fee receiver and `creditToYield` amount to the transmuter. These are real token transfers that permanently remove MYT from the AlchemistV3 contract's balance. However, there is no corresponding `_mytSharesDeposited -= ...` statement after either transfer. The internal accounting variable still believes these tokens are in the contract when they are not.

The second location where this bug occurs is in the `_doLiquidation` function, which handles the actual collateral seizure during liquidations:

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
    account.collateralBalance = account.collateralBalance > amountLiquidated ? account.collateralBalance - amountLiquidated : 0;
    _subDebt(accountId, debtToBurn);

    // send liquidation amount - fee to transmuter
    TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);
    // _mytSharesDeposited NOT decremented

    // send base fee to liquidator if available
    if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
        TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
        // _mytSharesDeposited NOT decremented
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

Again, look at the two `TokenUtils.safeTransfer` calls that send MYT tokens. The function transfers `amountLiquidated - feeInYield` to the transmuter and `feeInYield` to the liquidator. These are real token movements that reduce the contract's actual MYT balance, but there is no `_mytSharesDeposited` decrement anywhere in this function. The internal ledger remains unchanged even though tokens have permanently left the contract.

To understand when this bug manifests, we need to look at when liquidations occur. The public `liquidate` function is the entry point:

```solidity
function liquidate(uint256 accountId) external override returns (uint256 yieldAmount, uint256 feeInYield, uint256 feeInUnderlying) {
    _checkForValidAccountId(accountId);
    (yieldAmount, feeInYield, feeInUnderlying) = _liquidate(accountId);
    if (yieldAmount > 0) {
        return (yieldAmount, feeInYield, feeInUnderlying);
    } else {
        // no liquidation amount returned, so no liquidation happened
        revert LiquidationError();
    }
}
```

This calls the internal `_liquidate` function, which determines whether a position is eligible for liquidation and handles the multi-step process:

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
        repaidAmountInYield = _forceRepay(accountId, account.earmarked);  // Bug occurs here
    }
    
    // If debt is fully cleared, return with only the repaid amount, no liquidation needed
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
        return _doLiquidation(accountId, collateralInUnderlying, repaidAmountInYield);  //  Bug occurs here 
    } else {
        // Since only a repayment happened, send repayment fee to caller
        feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
        TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
        return (repaidAmountInYield, feeInYield, 0);
    }
}
```

The liquidation process first attempts to repay any earmarked debt through `_forceRepay`, and if the position is still undercollateralized after that, it proceeds to the actual liquidation through `_doLiquidation`. Both of these functions, as we have seen, transfer MYT tokens out without updating `_mytSharesDeposited`.

## Impact Details

First, it affects the deposit cap enforcement. Deposits are incorrectly rejected because the internal accounting is out of sync with reality.

This function uses the inflated `_mytSharesDeposited` value to calculate the total underlying value of all collateral in the protocol. Because `_mytSharesDeposited` was not decremented during liquidation, this function returns a value that is higher than the actual collateral value.

```solidity
function _getTotalUnderlyingValue() internal view returns (uint256 totalUnderlyingValue) {
    uint256 yieldTokenTVLInUnderlying = convertYieldTokensToUnderlying(_mytSharesDeposited);
    totalUnderlyingValue = yieldTokenTVLInUnderlying;
}
```

This inflated TVL is then used in the liquidation logic:

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
        normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt,  // Uses inflated TVL
        globalMinimumCollateralization,
        liquidatorFee
    );
    // ... rest of function
}
```

The fourth parameter to `calculateLiquidation` is the protocol-wide collateralization ratio, which is calculated by dividing the total underlying value (from `_getTotalUnderlyingValue`) by the total debt. Because the TVL is inflated, this ratio is artificially high. Now let's look at what `calculateLiquidation` does with this value:

```solidity
function calculateLiquidation(
    uint256 collateral,
    uint256 debt,
    uint256 targetCollateralization,
    uint256 alchemistCurrentCollateralization,
    uint256 alchemistMinimumCollateralization,
    uint256 feeBps
) public pure returns (uint256 grossCollateralToSeize, uint256 debtToBurn, uint256 fee, uint256 outsourcedFee) {
    if (debt >= collateral) {
        outsourcedFee = (debt * feeBps) / BPS;
        // fully liquidate debt if debt is greater than collateral
        return (collateral, debt, 0, outsourcedFee);
    }

    if (alchemistCurrentCollateralization < alchemistMinimumCollateralization) {
        outsourcedFee = (debt * feeBps) / BPS;
        // fully liquidate debt in high ltv global environment
        return (debt, debt, 0, outsourcedFee);
    }

    // fee is taken from surplus = collateral - debt
    uint256 surplus = collateral > debt ? collateral - debt : 0;
    fee = (surplus * feeBps) / BPS;

    // collateral remaining for margin‐restore calc
    uint256 adjCollat = collateral - fee;

    // compute m*d  (both plain units)
    uint256 md = (targetCollateralization * debt) / FIXED_POINT_SCALAR;

    // if md <= adjCollat, nothing to liquidate
    if (md <= adjCollat) {
        return (0, 0, fee, 0);
    }

    // numerator = md - adjCollat
    uint256 num = md - adjCollat;

    // denom = m - 1
    uint256 denom = targetCollateralization - FIXED_POINT_SCALAR;

    // debtToBurn = (num * FIXED_POINT_SCALAR) / denom
    debtToBurn = (num * FIXED_POINT_SCALAR) / denom;

    // gross collateral seize = net + fee
    grossCollateralToSeize = debtToBurn + fee;
}
```

Look at the second if statement. When the protocol-wide collateralization ratio is below the minimum threshold, the function returns a full liquidation where all the debt is burned and the outsourced fee is charged. This is the emergency mode that is supposed to protect the protocol when it is in danger of insolvency. However, because `alchemistCurrentCollateralization` is calculated using the inflated TVL, this condition will evaluate to false even when the real collateralization is dangerously low. The protocol will proceed to a partial liquidation when it should be doing a full liquidation, leaving bad debt in the system.

## Proof of Concept

## Proof of Concept

Add this to the AlchemistV3.t.sol test contract and run

```solidity
   function testLiquidation_MissingMytSharesDepositedAccounting() public {
    vm.startPrank(alOwner);
    alchemist.setProtocolFee(protocolFee);
    alchemist.setLiquidatorFee(liquidatorFeeBPS);
    alchemist.setRepaymentFee(100);
    vm.stopPrank();

    address victim = externalUser;
    address liquidator = anotherExternalUser;
    uint256 depositAmt = 100_000e18;
    
    vm.startPrank(someWhale);
    IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
    vm.stopPrank();
    
    vm.startPrank(victim);
    IERC20(address(vault)).approve(address(alchemist), depositAmt);
    alchemist.deposit(depositAmt, victim, 0);
    uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(victim, address(alchemistNFT));
    
    uint256 maxBorrow = alchemist.getMaxBorrowable(tokenId);
    uint256 borrowAmount = (maxBorrow * 98) / 100; // 98% of max to get closer to edge
    alchemist.mint(tokenId, borrowAmount, victim);
    vm.stopPrank();

    uint256 balanceBefore = IERC20(address(vault)).balanceOf(address(alchemist));
    uint256 tvlBefore = alchemist.getTotalUnderlyingValue();
    
    // 15% loss: increase supply by ~1765 bps
    _manipulateYieldTokenPrice(1765);

    (,uint256 debt,) = alchemist.getCDP(tokenId);
    uint256 ltvAfterLoss = (debt * 100 * FIXED_POINT_SCALAR) / alchemist.totalValue(tokenId);
    assertTrue(ltvAfterLoss > 95 * FIXED_POINT_SCALAR, "Position must be liquidatable");

    vm.prank(liquidator);
    (uint256 amountLiquidated,,) = alchemist.liquidate(tokenId);

    uint256 balanceAfter = IERC20(address(vault)).balanceOf(address(alchemist));
    uint256 tvlAfter = alchemist.getTotalUnderlyingValue();

    uint256 mytOutflow = balanceBefore - balanceAfter;
    assertTrue(mytOutflow > 0, "MYT left contract");
    
    uint256 depositCap = balanceAfter + 50_000e18;
    vm.prank(alOwner);
    alchemist.setDepositCap(depositCap);
    
    address newUser = yetAnotherExternalUser;
    uint256 newDeposit = 40_000e18;
    assertTrue(balanceAfter + newDeposit < depositCap, "Capacity exists");
    
    vm.startPrank(newUser);
    IERC20(address(vault)).approve(address(alchemist), newDeposit);
    vm.expectRevert(IllegalState.selector);
    alchemist.deposit(newDeposit, newUser, 0);
    vm.stopPrank();
    
    uint256 expectedTvlDecrease = alchemist.convertYieldTokensToUnderlying(mytOutflow);
    uint256 expectedTvlAfter = tvlBefore - expectedTvlDecrease;
    assertTrue(tvlAfter > expectedTvlAfter, "TVL inflated");
    
    console.log("\n=== BUG CONFIRMED ===");
    console.log("MYT outflow:", mytOutflow);
    console.log("TVL inflation:", ((tvlAfter - expectedTvlAfter) * 100) / expectedTvlAfter, "%");
}
```
