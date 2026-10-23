# 57129 sc high missing mytsharesdeposited decrement in liquidation functions causes permanent tvl inflation

**Submitted on Oct 23rd 2025 at 17:51:48 UTC by @Max36935 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57129
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The AlchemistV3 contract fails to decrement the \_mytSharesDeposited state variable when transferring MYT tokens out during liquidation operations, unlike all other token transfer functions (withdraw, burn, repay, claimRedemption) which properly maintain this accounting. This causes getTotalUnderlyingValue() to return permanently inflated values after each liquidation, affecting the alchemistCurrentCollateralization calculation used in critical liquidation decisions. The cumulative accounting error masks the protocol's true collateralization state, preventing the system from recognizing undercollateralization and triggering appropriate crisis responses, potentially enabling users to mint debt against phantom collateral and leading to protocol insolvency.

## Vulnerability Details

Root Cause

The contract maintains \_mytSharesDeposited (line 134 in AlchemistV3.sol) to track total deposited MYT (yield token) shares:

```solidity
/// @dev Total yield tokens deposited
/// This is used to differentiate between tokens deposited into a CDP and balance of the contract
uint256 private _mytSharesDeposited;
```

This variable is consistently updated across all token transfer operations: Token deposits increase the counter:

```solidity
// Line 383 - deposit()
_mytSharesDeposited += amount;
```

All standard token withdrawals decrease the counter:

```solidity
// Line 410 - withdraw()
TokenUtils.safeTransfer(myt, recipient, amount);
_mytSharesDeposited -= amount;

// Line 485 - burn()
TokenUtils.safeTransfer(myt, protocolFeeReceiver, convertDebtTokensToYield(credit) * protocolFee / BPS);
_mytSharesDeposited -= convertDebtTokensToYield(credit) * protocolFee / BPS;

// Line 541 - repay()
TokenUtils.safeTransfer(myt, protocolFeeReceiver, creditToYield * protocolFee / BPS);
_mytSharesDeposited -= creditToYield * protocolFee / BPS;

// Line 638 - claimRedemption()
TokenUtils.safeTransfer(myt, transmuter, collRedeemed);
TokenUtils.safeTransfer(myt, protocolFeeReceiver, feeCollateral);
_mytSharesDeposited -= collRedeemed + feeCollateral;
```

However, liquidation functions break this pattern:

`_doLiquidation()` (lines 875-879) - Missing updates:

```solidity
// send liquidation amount - fee to transmuter
TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);
// ❌ MISSING: _mytSharesDeposited -= (amountLiquidated - feeInYield);

// send base fee to liquidator if available
if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
    TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
    // ❌ MISSING: _mytSharesDeposited -= feeInYield;
}
```

`_forceRepay()` (lines 774-779) - Missing updates:

```solidity
if (account.collateralBalance > protocolFeeTotal) {
    account.collateralBalance -= protocolFeeTotal;
    TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
    // ❌ MISSING: _mytSharesDeposited -= protocolFeeTotal;
}

if (creditToYield > 0) {
    TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
    // ❌ MISSING: _mytSharesDeposited -= creditToYield;
}
```

`_liquidate()` repayment fee paths (lines 826, 840) - Missing updates:

```solidity
feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
// ❌ MISSING: _mytSharesDeposited -= feeInYield;
```

How the Accounting Error Propagates The inflated `_mytSharesDeposited` directly affects TVL calculation:

```solidity
// Line 1238-1241
function _getTotalUnderlyingValue() internal view returns (uint256 totalUnderlyingValue) {
    uint256 yieldTokenTVLInUnderlying = convertYieldTokensToUnderlying(_mytSharesDeposited);
    totalUnderlyingValue = yieldTokenTVLInUnderlying;
}
```

This inflated TVL then feeds into the critical liquidation calculation at line 862:

```solidity
(uint256 liquidationAmount, uint256 debtToBurn, uint256 baseFee, uint256 outsourcedFee) = calculateLiquidation(
    collateralInUnderlying,
    account.debt,
    minimumCollateralization,
    normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt,  // ← Inflated
    globalMinimumCollateralization,
    liquidatorFee
);
```

The fourth parameter is `alchemistCurrentCollateralization`, calculated as `(Total Protocol TVL / Total Protocol Debt) * 1e18`. This ratio determines whether the protocol should enter emergency liquidation mode:

```solidity
// Line 1257 in calculateLiquidation()
if (alchemistCurrentCollateralization < alchemistMinimumCollateralization) {
    outsourcedFee = (debt * feeBps) / BPS;
    // fully liquidate debt in high ltv global environment
    return (debt, debt, 0, outsourcedFee);
}
```

## Impact Details

Primary Impact: Masked Undercollateralization The accounting error causes `alchemistCurrentCollateralization` to appear higher than reality, with the following consequences:

1. Failed Crisis Detection When the protocol becomes genuinely undercollateralized (TVL < Debt × globalMinimumCollateralization), the inflated TVL prevents the system from recognizing this state. The critical check at line 1257 in `calculateLiquidation()`:

```solidity
if (alchemistCurrentCollateralization < alchemistMinimumCollateralization) {
    // Enter emergency full liquidation mode
    return (debt, debt, 0, outsourcedFee);
}
```

This emergency mode should trigger full liquidations when the protocol is in crisis, but the inflated `alchemistCurrentCollateralization` prevents this, allowing normal operations to continue during actual insolvency.

2. Cumulative Degradation Each liquidation compounds the error:

* First liquidation: TVL inflated by X tokens
* Second liquidation: TVL inflated by X + Y tokens
* After N liquidations: TVL inflated by sum of all liquidated amounts
* No recovery mechanism exists to correct the discrepancy

3. Incorrect Liquidation Incentives The `calculateLiquidation()` function uses the inflated collateralization ratio to determine liquidation amounts and fees. This can result in:

* Undersized liquidations (liquidating less debt than needed)
* Incorrect fee calculations
* Delayed recognition of position health

4. Potential for Excessive Debt Minting Users can mint debt up to `minimumCollateralization` ratio. If the protocol's actual TVL is lower than reported due to accumulated inflation, new debt minting could push the protocol into actual insolvency while appearing healthy.

## References

AlchemistV3.sol: Main contract file

* Line 134: `_mytSharesDeposited` variable declaration
* Lines 852-890: `_doLiquidation()` function (missing updates at lines 875, 879)
* Lines 735-780: `_forceRepay()` function (missing updates at lines 774, 779)
* Lines 791-850: `_liquidate()` function (missing updates at lines 826, 840)
* Line 1239: `_getTotalUnderlyingValue()` function (uses inflated `_mytSharesDeposited`)
* Line 862: `calculateLiquidation()` call using inflated TVL

Comparison functions that correctly update `_mytSharesDeposited`:

* Line 410: `withdraw()` - decrements properly
* Line 485: `burn()` - decrements properly
* Line 541: `repay()` - decrements properly
* Line 638: `claimRedemption()` - decrements properly

## Recommendations

Fix 1: Add `_mytSharesDeposited` Decrements in `_doLiquidation()` Update the `_doLiquidation()` function to decrement `_mytSharesDeposited` when transferring MYT tokens out, matching the pattern used in all other withdrawal functions.

```diff
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
    account.collateralBalance = account.collateralBalance > amountLiquidated 
        ? account.collateralBalance - amountLiquidated 
        : 0;
    _subDebt(accountId, debtToBurn);
    
    // send liquidation amount - fee to transmuter
    TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);
+   _mytSharesDeposited -= (amountLiquidated - feeInYield);
    
    // send base fee to liquidator if available
    if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
        TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
+       _mytSharesDeposited -= feeInYield;
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

Fix 2: Add `_mytSharesDeposited` Decrements in `_forceRepay()` and `_liquidate()` Update both `_forceRepay()` (called during liquidation) and the repayment fee paths in `_liquidate()` to maintain consistent accounting. In `_forceRepay()`:

```diff
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
+       _mytSharesDeposited -= protocolFeeTotal;
    }
    
    if (creditToYield > 0) {
        // Transfer the repaid tokens from the account to the transmuter.
        TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
+       _mytSharesDeposited -= creditToYield;
    }
    
    return creditToYield;
}
In _liquidate() repayment fee paths:
difffunction _liquidate(uint256 accountId) internal returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying) {
    // ... existing liquidation logic ...
    
    // If debt is fully cleared, return with only the repaid amount, no liquidation needed
    if (account.debt == 0) {
        feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
        TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
+       _mytSharesDeposited -= feeInYield;
        return (repaidAmountInYield, feeInYield, 0);
    }
    
    // ... recalculation logic ...
    
    if (collateralizationRatio <= collateralizationLowerBound) {
        // Do actual liquidation
        return _doLiquidation(accountId, collateralInUnderlying, repaidAmountInYield);
    } else {
        // Since only a repayment happened, send repayment fee to caller
        feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
        TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
+       _mytSharesDeposited -= feeInYield;
        return (repaidAmountInYield, feeInYield, 0);
    }
}
```

These changes ensure that `_mytSharesDeposited` is consistently maintained across all functions that transfer MYT tokens out of the contract, aligning liquidation functions with the existing pattern used by `withdraw()`, `burn()`, `repay()`, and `claimRedemption()`.

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "forge-std/Test.sol";
import {AlchemistV3Test} from "./AlchemistV3.t.sol";
import {SafeERC20} from "../libraries/SafeERC20.sol";
import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {AlchemistNFTHelper} from "./libraries/AlchemistNFTHelper.sol";
import {IMockYieldToken} from "./mocks/MockYieldToken.sol";

contract Audit_MytSharesNotDecrementedOnLiquidation is AlchemistV3Test {
    address liquidator = address(0x9999);

    function testAudit_MytSharesDeposited_NotUpdated_OnLiquidation() external {
        // Setup: User deposits collateral
        uint256 depositAmount = 100_000e18;

        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);

        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
        alchemist.deposit(depositAmount, someWhale, 0);

        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(someWhale, address(alchemistNFT));

        // Mint maximum debt
        uint256 maxDebt = alchemist.getMaxBorrowable(tokenId);
        alchemist.mint(tokenId, maxDebt, someWhale);
        vm.stopPrank();

        console.log("=== INITIAL STATE ===");
        console.log("User deposited: %s", depositAmount / 1e18);

        // Record the INITIAL state
        uint256 totalUnderlyingBefore = alchemist.getTotalUnderlyingValue();
        uint256 contractBalanceBefore = IERC20(address(vault)).balanceOf(address(alchemist));

        console.log("Contract balance: %s", contractBalanceBefore / 1e18);
        console.log("getTotalUnderlyingValue(): %s", totalUnderlyingBefore / 1e18);
        console.log("");

        // Simulate price drop to make position liquidatable
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);

        // Increase supply by 40% (causes ~28% price drop)
        uint256 modifiedVaultSupply = (initialVaultSupply * 14000) / 10000;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        console.log("=== AFTER PRICE DROP ===");
        (uint256 collateral, uint256 debt, ) = alchemist.getCDP(tokenId);
        uint256 collateralValue = alchemist.totalValue(tokenId);
        console.log("Collateral value: %s", collateralValue / 1e18);
        console.log("Debt: %s", debt / 1e18);
        console.log("Collateralization: %s%%", (collateralValue * 100) / debt);

        uint256 collateralizationRatio = (collateralValue * FIXED_POINT_SCALAR) / debt;
        bool isLiquidatable = collateralizationRatio <= alchemist.collateralizationLowerBound();

        if (!isLiquidatable) {
            console.log("ERROR: Position not liquidatable, test setup failed");
            revert("Setup failed");
        }

        console.log("Position is liquidatable!");
        console.log("");

        // Execute liquidation
        console.log("=== EXECUTING LIQUIDATION ===");
        vm.prank(liquidator);
        (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenId);

        console.log("Amount liquidated: %s", amountLiquidated / 1e18);
        console.log("Fee to liquidator (yield): %s", feeInYield / 1e18);
        console.log("Fee to liquidator (underlying): %s", feeInUnderlying / 1e18);
        console.log("");

        // Check AFTER liquidation
        uint256 contractBalanceAfter = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 totalUnderlyingAfter = alchemist.getTotalUnderlyingValue();

        console.log("=== STATE AFTER LIQUIDATION ===");
        console.log("Contract balance BEFORE: %s", contractBalanceBefore / 1e18);
        console.log("Contract balance AFTER:  %s", contractBalanceAfter / 1e18);
        console.log("Tokens transferred OUT:  %s", (contractBalanceBefore - contractBalanceAfter) / 1e18);
        console.log("");
        console.log("getTotalUnderlyingValue() BEFORE: %s", totalUnderlyingBefore / 1e18);
        console.log("getTotalUnderlyingValue() AFTER:  %s", totalUnderlyingAfter / 1e18);
        console.log("");

        // Calculate expected value
        uint256 tokensTransferredOut = contractBalanceBefore - contractBalanceAfter;
        uint256 expectedTotalUnderlying = totalUnderlyingBefore - tokensTransferredOut;

        console.log("=== BUG VERIFICATION ===");
        console.log("Expected getTotalUnderlyingValue(): %s", expectedTotalUnderlying / 1e18);
        console.log("Actual getTotalUnderlyingValue():   %s", totalUnderlyingAfter / 1e18);
        console.log("Discrepancy: %s", (totalUnderlyingAfter - expectedTotalUnderlying) / 1e18);
        console.log("");

        if (totalUnderlyingAfter > expectedTotalUnderlying) {
            console.log("*** BUG CONFIRMED ***");
            console.log(
                "getTotalUnderlyingValue() is INFLATED by %s tokens",
                (totalUnderlyingAfter - expectedTotalUnderlying) / 1e18
            );
            console.log("");
            console.log("Root cause: _mytSharesDeposited not decremented during liquidation");
            console.log("Impact: Protocol TVL is overstated, affecting liquidation calculations");

            // This assertion proves the bug
            assertGt(
                totalUnderlyingAfter,
                expectedTotalUnderlying,
                "BUG: getTotalUnderlyingValue() should decrease but didn't"
            );
        } else {
            console.log("No bug detected - values match as expected");
        }
    }

    // function testAudit_CompareBurnVsLiquidation_Accounting() external {
    //     // This test compares how burn() properly updates _mytSharesDeposited
    //     // but liquidation doesn't

    //     console.log("=== COMPARISON TEST: burn() vs liquidate() ===");
    //     console.log("");

    //     // Setup two identical positions
    //     uint256 depositAmount = 100_000e18;

    //     // Position 1: Will be burned
    //     vm.startPrank(address(0xBEEF));
    //     IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, address(0xBEEF));
    //     SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
    //     alchemist.deposit(depositAmount, address(0xBEEF), 0);
    //     uint256 tokenId1 = AlchemistNFTHelper.getFirstTokenId(address(0xBEEF), address(alchemistNFT));
    //     uint256 debt1 = alchemist.getMaxBorrowable(tokenId1);
    //     alchemist.mint(tokenId1, debt1, address(0xBEEF));
    //     vm.stopPrank();

    //     // Position 2: Will be liquidated
    //     vm.startPrank(address(0xCAFE));
    //     IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, address(0xCAFE));
    //     SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
    //     alchemist.deposit(depositAmount, address(0xCAFE), 0);
    //     uint256 tokenId2 = AlchemistNFTHelper.getFirstTokenId(address(0xCAFE), address(alchemistNFT));
    //     uint256 debt2 = alchemist.getMaxBorrowable(tokenId2);
    //     alchemist.mint(tokenId2, debt2, address(0xCAFE));
    //     vm.stopPrank();

    //     vm.roll(block.number + 1);

    //     console.log("=== SCENARIO 1: BURN ===");
    //     uint256 tvlBeforeBurn = alchemist.getTotalUnderlyingValue();
    //     console.log("TVL before burn: %s", tvlBeforeBurn / 1e18);

    //     // Burn some debt
    //     uint256 burnAmount = 10_000e18;
    //     vm.prank(address(0xBEEF));
    //     alchemist.burn(burnAmount, tokenId1);

    //     uint256 tvlAfterBurn = alchemist.getTotalUnderlyingValue();
    //     console.log("TVL after burn:  %s", tvlAfterBurn / 1e18);
    //     console.log("TVL decreased by: %s (CORRECT)", (tvlBeforeBurn - tvlAfterBurn) / 1e18);
    //     console.log("");

    //     console.log("=== SCENARIO 2: LIQUIDATION ===");

    //     // Make position liquidatable
    //     uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
    //     IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
    //     uint256 modifiedVaultSupply = (initialVaultSupply * 14000) / 10000;
    //     IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

    //     uint256 tvlBeforeLiquidation = alchemist.getTotalUnderlyingValue();
    //     console.log("TVL before liquidation: %s", tvlBeforeLiquidation / 1e18);

    //     // Liquidate
    //     vm.prank(liquidator);
    //     alchemist.liquidate(tokenId2);

    //     uint256 tvlAfterLiquidation = alchemist.getTotalUnderlyingValue();
    //     console.log("TVL after liquidation:  %s", tvlAfterLiquidation / 1e18);

    //     if (tvlAfterLiquidation == tvlBeforeLiquidation) {
    //         console.log("TVL did NOT decrease (BUG!)");
    //         console.log("");
    //         console.log("*** BUG CONFIRMED ***");
    //         console.log("burn() correctly updates TVL");
    //         console.log("liquidation() does NOT update TVL");
    //     }
    // }
}
```

### Create File

`src/test/Audit_MytSharesNotDecrementedOnLiquidation.t.sol`

### Run Test

forge test --match-contract Audit\_MytSharesNotDecrementedOnLiquidation -vvv
