# 58447 sc critical unfair collateral loss through socialized redemption costs

**Submitted on Nov 2nd 2025 at 12:36:43 UTC by @MahdiKarimi for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58447
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Protocol insolvency

## Description

## Overview

In AlchemistV3, the redemption process is designed such that the cost of redemption (collateral and protocol fees) is shared among all debt holders, while the benefit (debt reduction) is granted only to users with previously earmarked debt.

This creates an imbalance: if a user has no earmarked debt, their collateral still decays during a redemption, even though neither their debt nor earmark changes. This happens because, during the earmarking phase, earmarked debt is distributed across all existing debt holders. If a new user later joins the system by depositing collateral and minting debt, they start with zero earmarked balance. When a redemption then occurs, the collateral decay (representing redemption costs) is applied globally, causing the new user’s collateral to decrease—despite having no debt reduced or earmarked cleared.

As a result, users can lose collateral value without receiving any benefit, effectively subsidizing the redemptions of others.

***

## Technical Description

### Flawed Accounting Relationship

The issue stems from the contract’s **global weight-based decay model**. Two key weights control balance updates during redemptions:

| Weight                                | Purpose                                             | Scope                                      |
| ------------------------------------- | --------------------------------------------------- | ------------------------------------------ |
| `_collateralWeight`                   | Tracks collateral and fee removal after redemptions | Global (applies to all users)              |
| `_redemptionWeight`, `_earmarkWeight` | Track debt earmarking and redemption decay          | Per-user (applies only to earmarked users) |

When a redemption occurs, `redeem()` reduces total collateral and updates `_collateralWeight` based on the redeemed amount. During the next `_sync()`, every user’s collateral balance is scaled down using this updated weight — even if they had **no earmarked debt** and thus **no debt relief**.

Meanwhile, the actual **debt reduction** in `_sync()` depends entirely on a user’s personal earmarked balance. If a user’s `earmarked == 0`, their debt remains unchanged, despite their collateral being reduced through the global decay.

The outcome:

> Collateral losses are *socialized* across all users, but the debt benefit is *privatized* to earmarked users.

***

## Demonstration (PoC)

A minimal test, `test_PoC_SocializedDecay_UnfairCollateralLoss()` which I have included can be added to`AlchemistV3.t.sol`, verifies this behavior.

### Scenario

1. **User A**
   * Deposits `100e18` collateral
   * Mints `90e18` debt
   * `20e18` of this debt is earmarked
2. **User B (victim)**
   * Deposits `100e18` collateral
   * Mints `90e18` debt
   * Has **no** earmarked debt
3. **Redemption phase**
   * The transmuter redeems `20e18` of User A’s debt.
   * `_collateralWeight` increases to reflect total collateral withdrawn (principal + fee).
4. **Sync phase**
   * User B calls `poke()`, triggering `_sync()`
   * His collateral balance is reduced due to global `_collateralWeight`, but his debt remains unchanged (since `earmarked = 0`).

### Output Summary

```
User B Debt Before:  90e18
User B Debt After:   90e18
User B Collateral Before: 100e18
User B Collateral After:   90e18
```

User B’s **debt is constant**, yet they lose **10e18 collateral**. That 10e18 was effectively seized to fund the redemption of User A’s earmarked debt.

***

## Impact

This is direct loss of collateral for users

## Root Cause Summary

| Mechanism                                         | Problem                                                        | Consequence                                  |
| ------------------------------------------------- | -------------------------------------------------------------- | -------------------------------------------- |
| `_collateralWeight` (global)                      | Applies redemption costs to all users                          | Collateral loss for unrelated users          |
| `_redemptionWeight` / `_earmarkWeight` (per-user) | Restrict debt relief to earmarked accounts                     | Benefit limited to selected users            |
| `_sync()`                                         | Uses global decay for collateral but per-user weights for debt | Accounting mismatch between loss and benefit |

***

## Recommended Fix

1. During `_sync()`, calculate `collateralToRemove` as a function of each user’s **personal redeemedTotal** (the portion of their earmarked cleared).
2. Apply decay locally, ensuring:
   * Users without earmarked debt do **not** lose collateral.
   * Debt and collateral adjustments remain symmetrical.

## Proof of Concept

## Proof of Concept

```solidity
/**
     * collateral decay is "socialized" while debt decay is "individualized".
     * This allows a user's collateral to be drained to pay for the redemption
     * of another user's debt.
     *
     * Scenario:
     * 1. User 1 (address(0xbeef)) deposits 100e18 collateral and mints 100e18 debt.
     * 2. An earmark of 20e18 occurs (mocked via transmuter).
     * 3. User 2 (externalUser) deposits 100e18 collateral and mints 100e18 debt.
     * 4. A redemption of 20e18 occurs (called by transmuter).
     * 5. User 2's position is synced.
     *
     * Expected Result:
     * - User 2's debt should be unchanged (100e18).
     * - User 2's collateral should be *reduced* as it was unfairly used
     * to cover the socialized cost of User 1's redemption.
     */
    function test_PoC_SocializedDecay_UnfairCollateralLoss() public {
        // --- 0. Define Users from setUp ---
        address user1 = address(0xbeef);
        address user2_victim = externalUser;
        uint256 tokenId_1;
        uint256 tokenId_2;
        // Max LTV based on minimumCollateralization = 1.111...
        uint256 mintAmount = 90e18; // 90% LTV

        // --- 1. User 1 Setup ---
        vm.startPrank(user1);
        SafeERC20.safeApprove(address(vault), address(alchemist), 100e18);
        // Deposit 100e18 myt
        alchemist.deposit(100e18, user1, 0);
        tokenId_1 = AlchemistNFTHelper.getFirstTokenId(user1, address(alchemistNFT)); // Get the tokenId
        // Mint 90e18 debt
        alchemist.mint(tokenId_1, mintAmount, user1);
        vm.stopPrank();

        // --- 2. Earmark for User 1 ---
        vm.roll(block.number + 1); // Advance block so earmark can run.

        // Get the block numbers for the mock call
        uint256 lastEarmarkBlock = alchemist.lastEarmarkBlock(); 
        uint256 currentBlock = block.number; // The block 'poke' will execute in.

        // Mock the transmuter's queryGraph call to return 20e18
        // We mock the *specific* call _earmark will make
        vm.mockCall(
            address(transmuterLogic),
            abi.encodeWithSelector(ITransmuter.queryGraph.selector, lastEarmarkBlock + 1, currentBlock),
            abi.encode(20e18)
        );

        // Poke User 1 to trigger _earmark() and _sync()
        alchemist.poke(tokenId_1);

        // Clear the mock call so it doesn't affect User 2's mint
        vm.clearMockedCalls();

        // Check that global state was updated by _earmark
        assertEq(alchemist.cumulativeEarmarked(), 20e18, "Global cumulativeEarmarked was not updated by _earmark");
        
        (,, uint256 earmarked1) = alchemist.getCDP(tokenId_1);
        assertEq(earmarked1, 20e18, "User 1 was not earmarked");

        // --- 3. User 2 (Victim) Setup ---
        vm.startPrank(user2_victim);
        SafeERC20.safeApprove(address(vault), address(alchemist), 100e18);
        // Deposit 100e18 myt
        alchemist.deposit(100e18, user2_victim, 0);
        tokenId_2 = AlchemistNFTHelper.getFirstTokenId(user2_victim, address(alchemistNFT)); // Get the tokenId
        // Mint 90e18 debt
        alchemist.mint(tokenId_2, mintAmount, user2_victim);
        vm.stopPrank();

        // --- 4. Get "Before" State for User 2 ---
        // We sync User 2's position to get a clean "before" state.
        alchemist.poke(tokenId_2);
        (uint256 collateralBefore, uint256 debtBefore, ) = alchemist.getCDP(tokenId_2);
        (uint256 collateralBeforeUser1, uint256 debtBeforeUser1, ) = alchemist.getCDP(tokenId_1);


        // Sanity check
        assertEq(debtBefore, mintAmount, "User 2 initial debt is incorrect");

        // --- 5. Trigger Redemption ---
        vm.roll(block.number + 1); // Advance block so redemption can run

        // Prank as the transmuter to call redeem
        vm.prank(address(transmuterLogic));
        alchemist.redeem(20e18); // Redeem the 20e18 that was earmarked

        // --- 6. Sync User 2 (Victim) ---
        // This poke will trigger _sync() for User 2, applying the
        // socialized collateral decay from the redemption.
        alchemist.poke(tokenId_2);

        // --- 7. Get "After" State & Assert ---
        (uint256 collateralAfter, uint256 debtAfter, ) = alchemist.getCDP(tokenId_2);
        (uint256 collateralAfterUser1, uint256 debtAfterUser1, ) = alchemist.getCDP(tokenId_1);


        // --- ASSERTIONS ---
        console.log("--- Socialized Decay PoC ---");
        console.log("User 2 Debt Before:     %s", debtBefore);
        console.log("User 2 Debt After:      %s", debtAfter);
        console.log("---------------------------------");
        console.log("User 2 Collateral Before: %s", collateralBefore);
        console.log("User 2 Collateral After:  %s", collateralAfter);
        console.log("Collateral Lost by User 2:  %s", collateralBefore - collateralAfter);
        console.log("---------------------------------");
        console.log("User 1 Debt Before:      %s", debtBeforeUser1);
        console.log("User 1 Collateral Before:  %s", collateralBeforeUser1);
        console.log("User 1 Debt After:      %s", debtAfterUser1);
        console.log("User 1 Collateral After:  %s", collateralAfterUser1);

        // CRITICAL ASSERTION 1: User 2's debt is unchanged
        // They did not have any debt earmarked, so they get no benefit.
        assertEq(debtAfter, mintAmount, "PoC FAILED: User 2's debt changed.");

        // CRITICAL ASSERTION 2: User 2's collateral *decreased*
        // Their collateral was taken to pay for User 1's redemption.
        assertTrue(
            collateralAfter < collateralBefore,
            "PoC FAILED: User 2's collateral did not decrease."
        );
    }
```
