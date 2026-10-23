# 57345 sc high missing cumulativeearmarked decrement in forcerepay breaks earmarking invariant leading to unfair redemption burden distribution

**Submitted on Oct 25th 2025 at 11:57:20 UTC by @Smartkelvin for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57345
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The \_forceRepay() function, called during liquidations to repay earmarked debt using a user's collateral, fails to decrement the global cumulativeEarmarked state variable despite reducing the user's local account.earmarked. This breaks the critical accounting invariant that cumulativeEarmarked should equal the sum of all users' account.earmarked values. This causes the protocol to understate available unearmarked debt, throttling new earmarking operations, and creates an unfair distribution of redemption burdens where non-liquidated users bear disproportionate losses during transmuter redemptions

## Vulnerability Details

The Accounting Invariant The protocol maintains a critical invariant:

`cumulativeEarmarked = Σ(all account.earmarked values)` This invariant ensures accurate tracking of how much debt has been earmarked for redemption across all position.

In \_forceRepay() (in AlchemistV3.sol)

```
  function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
    // ... earmark and sync ...
    
    uint256 credit = amount > debt ? debt : amount;
    uint256 creditToYield = convertDebtTokensToYield(credit);
    _subDebt(accountId, credit);  // Reduces totalDebt and applies clamp

    // Repay debt from earmarked amount of debt first
    uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
    account.earmarked -= earmarkToRemove;  // ✓ Local earmark reduced
    
    // ❌ MISSING: Global cumulativeEarmarked decrement
    
    creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield;
    account.collateralBalance -= creditToYield;
    
    // ...  ...
}
```

Compare this with the correct implementation in repay()

```
  function repay(uint256 amount, uint256 recipientTokenId) public returns (uint256) {
    // ...  ...
    
    uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
    account.earmarked -= earmarkToRemove;  // ✓ Local earmark reduced

    uint256 earmarkPaidGlobal = cumulativeEarmarked > earmarkToRemove ? earmarkToRemove : cumulativeEarmarked;
    cumulativeEarmarked -= earmarkPaidGlobal;  // ✓ Global earmark reduced correctly
    
    // ... rest of func...
}
```

The \_subDebt() function includes a safety clamp:

```
if (cumulativeEarmarked > totalDebt) {
    cumulativeEarmarked = totalDebt;
}
```

However, this only prevents cumulativeEarmarked from exceeding totalDebt, not from being overstated relative to actual user earmarks. This persists and accumulates with each liquidation.

## Impact Details

Throttled Earmarking of New Debt The \_earmark() function calculates available unearmarked debt as: `uint256 liveUnearmarked = totalDebt - cumulativeEarmarked;`

When `cumulativeEarmarked` is overstated, `liveUnearmarked` is understated, causing:

* Less new debt gets earmarked than should be
* New borrowers receive inadequate earmark coverage
* The transmuter queue doesn't grow proportionally with new debt

**Example:**

Actual State:

* Total debt: 200e18
* Actual sum of earmarks: 50e18
* Should earmark: 150e18
* cumulativeEarmarked: 140e18 (overstated by 90e18 from liquidations)
* liveUnearmarked: 200 - 140 = 60e18
* Only 60e18 will be earmarked instead of 150e18
* 90e18 of debt is incorrectly excluded from earmarking

### 2. Unfair Redemption Burden Distribution

### 3. Compounding Effect

Each liquidation that triggers `_forceRepay` adds to the desynchronization:

* First liquidation: +90e18
* Second liquidation: +50e18
* Total : 140e18

## Proof of Concept

## Proof of Concept

add the test function to the test file of the contract

```
function testForceRepay_CumulativeEarmarked_Smartkelvin_BugProof() external {
    vm.startPrank(someWhale);
    IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
    vm.stopPrank();

    // Setup - deposit from two users
    vm.startPrank(yetAnotherExternalUser);
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
    alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
    vm.stopPrank();

    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
    alchemist.deposit(depositAmount, address(0xbeef), 0);
    uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    alchemist.mint(tokenIdFor0xBeef, alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization, address(0xbeef));
    vm.stopPrank();

    // Create transmuter position to enable earmarking
    vm.startPrank(address(0xdad));
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 90e18);
    transmuterLogic.createRedemption(90e18);
    vm.stopPrank();

    // Fast forward to allow earmarking
    vm.roll(block.number + 5_256_000);
    alchemist.poke(tokenIdFor0xBeef);

    // Record BEFORE state
    uint256 cumulativeBefore = alchemist.cumulativeEarmarked();
    (, uint256 debtBefore, uint256 earmarkedBefore) = alchemist.getCDP(tokenIdFor0xBeef);

    console.log("\n=== BEFORE LIQUIDATION ===");
    console.log("cumulativeEarmarked:", cumulativeBefore);
    console.log("account.earmarked:", earmarkedBefore);
    console.log("debt:", debtBefore);

    // Price crash to make position liquidatable
    uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
    uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

    // Execute liquidation (triggers _forceRepay internally)
    vm.startPrank(externalUser);
    alchemist.liquidate(tokenIdFor0xBeef);
    vm.stopPrank();

    // Record AFTER state
    uint256 cumulativeAfter = alchemist.cumulativeEarmarked();
    (,, uint256 earmarkedAfter) = alchemist.getCDP(tokenIdFor0xBeef);

    console.log("\n=== AFTER LIQUIDATION ===");
    console.log("cumulativeEarmarked:", cumulativeAfter);
    console.log("account.earmarked:", earmarkedAfter);

    // Calculate the desync
    uint256 earmarkReduction = earmarkedBefore > earmarkedAfter ? earmarkedBefore - earmarkedAfter : 0;
    console.log("\n=== BUG VERIFICATION ===");
    console.log("Local earmark reduced by:", earmarkReduction);
    
    if (earmarkReduction > 0) {
        uint256 expectedCumulative = cumulativeBefore > earmarkReduction ? cumulativeBefore - earmarkReduction : 0;
        
        console.log("Expected cumulativeEarmarked:", expectedCumulative);
        console.log("Actual cumulativeEarmarked:", cumulativeAfter);
        
        uint256 desync = cumulativeAfter > expectedCumulative ? 
                         cumulativeAfter - expectedCumulative : 
                         expectedCumulative - cumulativeAfter;
        
        console.log("DESYNC:", desync);
        
        //  PROOF: Instead of asserting they're equal (which would fail),
        // we assert they're NOT equal to prove the bug exists
        if (desync > 1e18) {
            console.log("\n>>> BUG CONFIRMED <<<");
            console.log("Global cumulativeEarmarked was NOT decremented in _forceRepay");
            console.log("This proves the accounting invariant is broken");
            
            // For a bug-proving test, we want THIS assertion to PASS
            // It asserts that the bug condition exists
            assertTrue(
                cumulativeAfter > expectedCumulative,
                "TEST PROVES BUG: cumulativeEarmarked is overstated after _forceRepay"
            );
            
            // Also verify the exact amount of the desync
            assertEq(
                desync,
                earmarkReduction,
                "Desync should equal the earmark reduction that was missed"
            );
        } else {
            // If desync is small, the bug doesn't exist (or was fixed)
            fail("Bug NOT found - cumulativeEarmarked was properly decremented");
        }
    } else {
        fail("No earmark reduction occurred - test setup issue");
    }
}
```
