# 58442 sc high liquidation breaks core accounting invariant missing cumulativeearmarked update in forcerepay causes permanent state drift

**Submitted on Nov 2nd 2025 at 12:05:26 UTC by @Bear36435 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58442
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The \_forceRepay() function, which is called during liquidation, updates individual account earmarked amounts but fails to update the global cumulativeEarmarked variable. This creates a permanent state inconsistency that violates the protocol's core invariant: cumulativeEarmarked must always equal the sum of all individual account.earmarked amounts. Each liquidation increases this discrepancy, causing the earmarking system to operate with corrupted accounting data. Over time, this state drift will cause the transmuter redemption mechanism to malfunction, as it relies on accurate cumulativeEarmarked values to properly allocate collateral for redemptions.

## Vulnerability Details

Root Cause:

In AlchemistV3.sol, the repay() function correctly maintains the relationship between individual account earmarked amounts and the global cumulativeEarmarked counter:

```solidity
   
    // AlchemistV3.sol Lines 522-526 - CORRECT IMPLEMENTATION
uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
account.earmarked -= earmarkToRemove;
uint256 earmarkPaidGlobal = cumulativeEarmarked > earmarkToRemove ? earmarkToRemove : cumulativeEarmarked;
cumulativeEarmarked -= earmarkPaidGlobal; // Global state updated
   
```

However, the \_forceRepay() function, which is called during liquidation, only updates the individual account's earmarked amount but completely omits the global cumulativeEarmarked update:

```solidity

   // AlchemistV3.sol Lines 760-762 - BUGGY IMPLEMENTATION
uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
account.earmarked -= earmarkToRemove; //@audit Individual account updated
// @audit MISSING: cumulativeEarmarked -= earmarkPaidGlobal; - Global state NOT updated
```

The Critical Invariant

The protocol maintains a critical accounting invariant throughout the codebase:

Invariant: cumulativeEarmarked == Σ(all account.earmarked values)

This invariant is essential because:

1. The transmuter uses cumulativeEarmarked to calculate global earmark weights
2. The \_earmark() function checks cumulativeEarmarked against totalEarmarkableDebt to determine capacity
3. Redemption accounting relies on accurate cumulativeEarmarked values

Execution Flow Leading to Bug

1. User creates a position and borrows alUSD
2. Another user creates a transmuter redemption, triggering earmarking via \_earmark()
3. Both the individual account.earmarked and global cumulativeEarmarked are incremented
4. Position becomes undercollateralized due to debt accrual or collateral value drop
5. Liquidator calls liquidate() -> calls \_forceRepay() internally
6. \_forceRepay() removes debt and decrements account.earmarked
7. BUG: cumulativeEarmarked is NOT decremented
8. Invariant is now permanently violated

Contradiction to Intended Behavior

The code clearly shows the intended behavior in repay() where both local and global states are updated together. The \_forceRepay() function should follow the exact same pattern since it performs the same logical operation (removing earmarked debt), just in a forced context during liquidation. The inconsistency between these two implementations is a clear bug.

## Impact Details

Primary Impact: Contract Fails to Deliver Promised Returns

The earmarking mechanism is a core feature that enables users to receive their collateral back through the transmuter over time. The corrupted cumulativeEarmarked value causes several operational failures:

1. Incorrect Earmark Weight Calculations

The transmuter calculates redemption weights based on cumulativeEarmarked:

```solidity
  // Weight of redemption amount / total earmarked debt
uint256 weight = amount * FIXED_POINT_SCALAR / cumulativeEarmarked;
```

With an inflated cumulativeEarmarked, each redemption receives less weight than it should, causing users to receive collateral at a slower rate than intended.

2. Premature Earmark Capacity Limits

The \_earmark() function checks if new earmarks exceed capacity:

```solidity
 
   if (cumulativeEarmarked + amountToEarmark > totalEarmarkableDebt) {
    // Limit reached
}
```

An inflated cumulativeEarmarked means this limit is reached prematurely, blocking legitimate earmarks even when actual earmarked amounts are well below the limit.

3. Accumulating State Drift

The discrepancy grows with each liquidation:

• 1 liquidation with 10k earmarked debt -> 10k drift.

• 10 liquidations -> 100k drift.

• 100 liquidations -> 1M drift.

This is permanent and cannot be corrected without a contract upgrade.

### Why This Is Severe Despite No Fund Loss

1. **Breaks Core Promise:** Protocol promises redemption mechanism
2. **Permanent Damage:** Cannot self-heal, requires upgrade
3. **Silent Failure:** No revert, just quietly breaks
4. **Compounds:** Gets exponentially worse over time
5. **No Workaround:** Users have no alternative redemption path

Affected Users

• Redemption creators: Receive collateral slower than expected • Position holders: May be unable to earmark when they should be able to • Protocol operators: Cannot rely on accurate accounting data • Liquidators: Indirectly affected as liquidations trigger the bug

## Recommended Mitigation Steps

Add the missing cumulativeEarmarked update to the \_forceRepay() function, mirroring the correct implementation in repay():

```solidity

    // AlchemistV3.sol Lines 760-762
function _forceRepay(...) internal {
    // ... existing code ...
    
    uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
    account.earmarked -= earmarkToRemove;
    
    // ADD THESE LINES:
    uint256 earmarkPaidGlobal = cumulativeEarmarked > earmarkToRemove 
        ? earmarkToRemove 
        : cumulativeEarmarked;
    cumulativeEarmarked -= earmarkPaidGlobal;
    
    // ... rest of function ...
}

```

This ensures that both the individual account state and the global state remain synchronized, preserving the critical invariant that cumulativeEarmarked == Σ(all account.earmarked values).

## Proof of Concept

## Proof of Concept

Paste this function inside the AlchemistV3.t.sol

```solidity

   function testBug_ForceRepay_Missing_CumulativeEarmarked_Update() external {
    uint256 amount = 200_000e18;
    
    vm.startPrank(someWhale);
    IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
    vm.stopPrank();

    vm.startPrank(yetAnotherExternalUser);
    SafeERC20.safeApprove(address(vault), address(alchemist), amount * 2);
    alchemist.deposit(amount, yetAnotherExternalUser, 0);
    vm.stopPrank();

   
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
    alchemist.deposit(amount, address(0xbeef), 0);
    uint256 tokenIdVictim = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    uint256 mintAmount = alchemist.totalValue(tokenIdVictim) * FIXED_POINT_SCALAR / minimumCollateralization;
    alchemist.mint(tokenIdVictim, mintAmount, address(0xbeef));
    // Transfer minted tokens to 0xdad for redemption
    SafeERC20.safeTransfer(address(alToken), address(0xdad), mintAmount);
    vm.stopPrank();

    vm.startPrank(anotherExternalUser);
    SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
    alchemist.deposit(amount, anotherExternalUser, 0);
    uint256 tokenIdSecond = AlchemistNFTHelper.getFirstTokenId(anotherExternalUser, address(alchemistNFT));
    alchemist.mint(tokenIdSecond, mintAmount, anotherExternalUser);
    // Transfer minted tokens to 0xdad for redemption
    SafeERC20.safeTransfer(address(alToken), address(0xdad), mintAmount);
    vm.stopPrank();

    vm.startPrank(address(0xdad));
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount * 2);
    transmuterLogic.createRedemption(mintAmount * 2);
    vm.stopPrank();

   
    vm.roll(block.number + (5_256_000 * 60 / 100));

    alchemist.poke(tokenIdVictim);
    alchemist.poke(tokenIdSecond);

    // Verify: Both positions have earmarked debt
    (, uint256 debtVictimBefore, uint256 earmarkedVictimBefore) = alchemist.getCDP(tokenIdVictim);
    (, uint256 debtSecondBefore, uint256 earmarkedSecondBefore) = alchemist.getCDP(tokenIdSecond);
    uint256 cumulativeEarmarkedBefore = alchemist.cumulativeEarmarked();

    console.log("=== BEFORE LIQUIDATION ===");
    console.log("Victim earmarked:", earmarkedVictimBefore);
    console.log("Second account earmarked:", earmarkedSecondBefore);
    console.log("cumulativeEarmarked:", cumulativeEarmarkedBefore);
    console.log("Sum of earmarked:", earmarkedVictimBefore + earmarkedSecondBefore);
    
    // Invariant should hold before liquidation
    assertApproxEqAbs(
        cumulativeEarmarkedBefore, 
        earmarkedVictimBefore + earmarkedSecondBefore,
        2,
        "Invariant should hold before liquidation"
    );

   
    uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
    // Increase yield token supply by 5.9% (price drop)
    uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

   
    // This internally calls _forceRepay() which has the bug
    vm.prank(externalUser);
    alchemist.liquidate(tokenIdVictim);

    alchemist.poke(tokenIdVictim);
    alchemist.poke(tokenIdSecond);

    // Get state after liquidation
    (, uint256 debtVictimAfter, uint256 earmarkedVictimAfter) = alchemist.getCDP(tokenIdVictim);
    (, uint256 debtSecondAfter, uint256 earmarkedSecondAfter) = alchemist.getCDP(tokenIdSecond);
    uint256 cumulativeEarmarkedAfter = alchemist.cumulativeEarmarked();

    console.log("\n=== AFTER LIQUIDATION ===");
    console.log("Victim earmarked:", earmarkedVictimAfter);
    console.log("Second account earmarked:", earmarkedSecondAfter);
    console.log("cumulativeEarmarked:", cumulativeEarmarkedAfter);
    console.log("Sum of earmarked:", earmarkedVictimAfter + earmarkedSecondAfter);
    console.log("\n=== BUG DETECTED ===");
    console.log("Earmarked removed from victim:", earmarkedVictimBefore - earmarkedVictimAfter);
    console.log("cumulativeEarmarked reduction:", cumulativeEarmarkedBefore - cumulativeEarmarkedAfter);
    
    uint256 earmarkedReduction = earmarkedVictimBefore > earmarkedVictimAfter ? 
        earmarkedVictimBefore - earmarkedVictimAfter : 0;
    uint256 cumulativeReduction = cumulativeEarmarkedBefore > cumulativeEarmarkedAfter ?
        cumulativeEarmarkedBefore - cumulativeEarmarkedAfter : 0;
    
    console.log("DISCREPANCY:", earmarkedReduction > cumulativeReduction ? 
        earmarkedReduction - cumulativeReduction : 0);

    // BUG PROOF: The invariant is now broken
    uint256 sumOfEarmarkedAfter = earmarkedVictimAfter + earmarkedSecondAfter;
    
    // This assertion will PASS, proving the bug exists
    // The sum of individual earmarked amounts is LESS than cumulativeEarmarked
    assertTrue(
        cumulativeEarmarkedAfter > sumOfEarmarkedAfter,
        "BUG PROVEN: cumulativeEarmarked is higher than sum of account.earmarked"
    );
    
    // Calculate the exact discrepancy
    uint256 discrepancy = cumulativeEarmarkedAfter - sumOfEarmarkedAfter;
    
    // The discrepancy should equal the earmarked debt that was force-repaid
    assertApproxEqAbs(
        discrepancy,
        earmarkedReduction,
        2,
        "Discrepancy should equal the force-repaid earmarked amount"
    );
    
    console.log("\n=== INVARIANT VIOLATION CONFIRMED ===");
    console.log("Expected cumulativeEarmarked:", sumOfEarmarkedAfter);
    console.log("Actual cumulativeEarmarked:", cumulativeEarmarkedAfter);
    console.log("Violation amount:", discrepancy);
}
```

\##Test Output

```
Siddique:~/Audit/v3-poc$ 
Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] testBug_ForceRepay_Missing_CumulativeEarmarked_Update() (gas: 4323339)
Logs:
  === BEFORE LIQUIDATION ===
  Victim earmarked: 108000000000000000010800
  Second account earmarked: 108000000000000000010800
  cumulativeEarmarked: 216000000000000000021600
  Sum of earmarked: 216000000000000000021600
  
=== AFTER LIQUIDATION ===
  Victim earmarked: 0
  Second account earmarked: 108000000000000000010800
  cumulativeEarmarked: 216000000000000000021600
  Sum of earmarked: 108000000000000000010800
  
=== BUG DETECTED ===
  Earmarked removed from victim: 108000000000000000010800
  cumulativeEarmarked reduction: 0
  DISCREPANCY: 108000000000000000010800
  
=== INVARIANT VIOLATION CONFIRMED ===
  Expected cumulativeEarmarked: 108000000000000000010800
  Actual cumulativeEarmarked: 216000000000000000021600
  Violation amount: 108000000000000000010800

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 26.38ms (10.35ms CPU time)

Ran 1 test suite in 29.01ms (26.38ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
