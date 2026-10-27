# 58399 sc critical precision loss in baddebtratio calculation causes overpayment and dos

**Submitted on Nov 1st 2025 at 23:58:21 UTC by @gizzy for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58399
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Summary

The `badDebtRatio` calculation in `Transmuter.claimRedemption()` suffers from insufficient precision scaling, causing the contract to overpay redemptions during bad debt scenarios. This leads to a Denial of Service (DOS) where the last withdrawer cannot exit their position because the contract attempts to transfer more MYT shares than it actually holds.

## Vulnerability Details

### Root Cause

In `Transmuter.sol:224`, the `badDebtRatio` is calculated as:

```solidity
uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10 ** TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator;
```

This formula only scales by **one** `FIXED_POINT_SCALAR` (10^18), but when used in line 234 to scale down `scaledTransmuted`, it creates a unit mismatch:

```solidity
scaledTransmuted = amountTransmuted * FIXED_POINT_SCALAR / badDebtRatio;
```

The calculation should maintain higher precision by using **two** `FIXED_POINT_SCALAR` factors in the numerator because is a sensitive varible , i will explain how below

### Mathematical Breakdown

Using real values from the POC (treating wei values as dollars for clarity):

#### Scenario Setup

one user

* **Initial Deposit**: 1,000 MYT shares (worth $1,000 at initial price)
* **Debt Minted**: $900.00000000000000090 All is sent to create redemption.
* **Price Crash**: MYT drops 12%
* **Post-Crash Collateral TotalUnderlyingValue**: 1,000 shares = $892.857142857142857000

#### Bad Debt Ratio Calculation

**Current Formula:**

```
totalSyntheticsIssued = $900.00000000000000090
denominator = $892.857142857142857232 (total underlying value)

badDebtRatio = (900.00000000000000090 × 10^18) / 892.857142857142857232
             = 1,008,000,000,000,000,000
             = 1.008 × 10^18
```

#### Impact on scaledTransmuted after full year for easy vasueliztion.

When `scaledTransmuted` is calculated:

**Current :**

```
amountTransmuted = $900,000000000000000090 

scaledTransmuted = (900000000000000000090× 10^18) / 1.008 × 10^18
                 = $892.857142857142857232  
But value of getTotalUnderlyingValue = $892.857142857142857000, so scaledTransmuted > getTotalUnderlyingValue

```

This looks like not much, but the impact occurs when converting to shares in redeem:

```
collRedeemed (shares) = convertToShares($892.857142857142857232)
                      = 1,000.000000000000000259 shares
and the redeem function will try to send this and revert
```

The transmuter contract requests **1,000.000000000000000259 shares** but only has **1,000.000000000000000000 shares** deposited. which will revert . but even of these wei is sent directly to the contract it will still revert with \_mytSharesDeposited underflow becuase is greater than what we stored , so these vaule is continuosly deducted from another users funds if there's another user when redemption happens.

If protocol fee is enables this scales the amount of last user would lose.

if theres a second user that just deposited collateral with even minting debt, redeem transaction will succed but that second user wont be able to withdraw full amount due to lack of funds.

**Overpayment: +259 wei shares ($0.000000000000000232 at crashed price)**

**Correct Formula (with proper scaling):**

```
badDebtRatio = (900.00000000000000090 × 10^18 × 10^18) / 892.857142857142857232
             = 1,008,000,000,000,000,000,262,080,000,000,000,000
             = 1.008000000000000000262080 × 10^36
```

```
scaledTransmuted = (899.999828767123287762 × 10^18 × 10^18) / (1.008000000000000000262080 × 10^36)
                 ≈ $892.857142857142857000 (which is the getTotalUnderlyingValue)

collRedeemed (shares) = convertToShares($892.857142857142857000)
                      = 9.9999e18 shares
```

Just usaged of the transmuter during baddebt triggers this .

While the overpayment is only \~259 wei shares (less than $0.000001), it compounds when:

* Multiple redemptions occur during bad debt
* High-value positions are involved (e.g., $1M position → overpay \~$0.26) In production with realistic values:
* Total collateral: $10,000,000
* 10 users with $1M positions each
* 10% price drop → bad debt state
* Each redemption overpays \~$0.13 in shares
* After 10 redemptions: \~$1.30 missing or more with fees → last user DOS

## Impact

**Denial of Service (DOS)**: The last user to exit their position through claimredemption cannot withdraw their collateral because the contract lacks sufficient MYT shares due to prior overpayments. **Loss of User Funds**: If multiple users try to claim redemptions during bad debt, the contract can become insolvent and it might preventt liquidation because of insuffient funds to transfer to transmuter.

### Likelihood

**High** - This occurs in any scenario where:

* Multiple redemptions exist when bad debt occurs

\##Reference <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/Transmuter.sol#L220C9-L226C10>

## Proof of Concept

## Proof of Concept

Add this test to `src/test/AlchemistV3.t.sol` (<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/test/AlchemistV3.t.sol>):

Run

`forge test --match-test testPOC_BadDebtRation_Precision_loss -vvvv`

```solidity
    function testPOC_BadDebtRation_Precision_loss() external {
        console.log("\n=== POC: Collateral Weight DOS Attack ===\n");

        
        uint256 depositAmount = 1000e18; //981920193698630136722


        // Step 1:  deposits and borrows maximum debt
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
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

        
        // Step 2:  creates redemption with ALL borrowed debt
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), maxBorrowable);
        transmuterLogic.createRedemption(maxBorrowable);
        console.log("\nStep 2:  creates redemption");
        vm.stopPrank();

        vm.roll(block.number +1);
        alchemist.poke(tokenIdAttacker);
        
        // Step 3: Advance time to mature redemption (100% maturity) 
        vm.roll(block.number + 5_256_000);
        console.log("\nStep 3: Fast forward to full maturity (2 years)");
        
        // Step 4: Simulate 12% price crash
        console.log("\nStep 4: PRICE CRASH - MYT drops 12%");
        // Increase mocked supply 10x = price drops to 10% of original
               uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 1200 bps or 12%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 1200 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);
        
        (uint256 collateralBefore, uint256 debtBefore, uint256 earmarkedBefore) = alchemist.getCDP(tokenIdAttacker);
        console.log("\nPosition after price crash:");
        console.log("  Collateral (shares):", collateralBefore);
        console.log("  Collateral value (underlying):", alchemist.totalValue(tokenIdAttacker));
        console.log("  Debt:", debtBefore);
        console.log("  Earmarked:", earmarkedBefore);
        uint256 collateralizationAfterCrash = alchemist.totalValue(tokenIdAttacker) * FIXED_POINT_SCALAR / debtBefore;
        console.log("  Collateralization ratio:", collateralizationAfterCrash / 1e16, "%");
        
 
    
        
        console.log("\nStep 5:  claims redemption after price crash, bug still valid even when liquidated  ");
        console.log("  Claiming redemption... This will attempt to claim redemption but fail because is a single user it will overpay when there are multiple users");
        
        vm.startPrank(address(0xbeef));
        vm.expectRevert();
        transmuterLogic.claimRedemption(1);
        vm.stopPrank();
 
   
    }
```
