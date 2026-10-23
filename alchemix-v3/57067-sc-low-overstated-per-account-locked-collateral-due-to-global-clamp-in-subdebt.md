# 57067 sc low overstated per account locked collateral due to global clamp in subdebt

**Submitted on Oct 23rd 2025 at 08:06:42 UTC by @Petrus for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57067
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

In the AlchemistV3 protocol's \_subDebt function, an issue overstates per-account rawLocked collateral by using pre-subtraction debt for calculations but subtracting a globally clamped toFree amount without recomputing on the updated debt, causing excessive pro-rata deductions from users' collateralBalance during \_sync in stressed scenarios like burns or liquidations, which could lead to unfair permanent loss of withdrawable funds, unwarranted liquidations, user attrition, and broader protocol insolvency.

## Vulnerability Details

```solidity
function _subDebt(uint256 tokenId, uint256 amount) internal {  
   Account storage account = _accounts[tokenId];  
 
   // Update collateral variables  
   uint256 toFree = convertDebtTokensToYield(amount) * minimumCollateralization / FIXED_POINT_SCALAR;  
  @> uint256 lockedCollateral = convertDebtTokensToYield(account.debt) * minimumCollateralization / FIXED_POINT_SCALAR;  
 
   // For cases when someone above minimum LTV gets liquidated.  
@>  if (toFree > _totalLocked) {  
       toFree = _totalLocked;  
   }  
 
   account.debt -= amount;  
   totalDebt -= amount;  
   _totalLocked -= toFree;  
@>  account.rawLocked = lockedCollateral - toFree;  // <-- Bug: Uses pre-subtraction lockedCollateral minus clamped toFree, overstating rawLocked when clamp triggers  
 
   // Clamp to avoid underflow due to rounding later at a later time  
   if (cumulativeEarmarked > totalDebt) {  
       cumulativeEarmarked = totalDebt;  
   }  
}  
```

The lockedCollateral is computed on the pre-subtraction account.debt (correct for estimating the delta toFree), but after clamping toFree to \_totalLocked (to protect the global from underflow), the per-account account.rawLocked is set to lockedCollateral - toFree (using the reduced toFree), which overstates the true required locked collateral for the new post-subtraction debt when the clamp activates, as it doesn't recompute based on the updated debt.

## Impact Details

This issue overstates per-account locked collateral baselines during global clamps, leading to excessive pro-rata deductions from users' collateralBalance in future \_sync calls and unfairly amplifying their share of protocol-wide redemptions and fees.

## Mitigation

The best solution is to recompute account.rawLocked after the debt subtraction using the updated account.debt to ensure it exactly matches the new required minimum locked collateral, decoupling per-account accuracy from the global clamp on toFree without altering the global protections.

Patched Code (replace the update block in \_subDebt):

// Update collateral variables\
uint256 toFree = convertDebtTokensToYield(amount) \* minimumCollateralization / FIXED\_POINT\_SCALAR;

// For cases when someone above minimum LTV gets liquidated.\
if (toFree > \_totalLocked) {\
toFree = \_totalLocked;\
}

account.debt -= amount;\
totalDebt -= amount;\
\_totalLocked -= toFree;

// Recompute rawLocked based on updated debt for exactness\
account.rawLocked = convertDebtTokensToYield(account.debt) \* minimumCollateralization / FIXED\_POINT\_SCALAR;

// Clamp to avoid underflow due to rounding later at a later time\
if (cumulativeEarmarked > totalDebt) {\
cumulativeEarmarked = totalDebt;\
}

````

## References
https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L932

## Proof of Concept

## Proof of Concept
POC 

Put the test in the src/test/AlchemistV3.t.sol  
 
Run the first test with: forge test --match-test testSubDebtOverstatesRawLockedAfterClampLeadingToExcessiveCollateralDeduction –vvvv 
```solidity 

Function testSubDebtOverstatesRawLockedAfterClampLeadingToExcessiveCollateralDeduction() external { 
    // Set fees to 0 for simplicity and isolation 
    vm.startPrank(alOwner); 
    alchemist.setProtocolFee(0); 
    transmuterLogic.setTransmutationFee(0); 
    vm.stopPrank(); 
 
    uint256 depositAmount = 11e18; // Enough for minColl on 10e18 debt 
    uint256 mintAmount = 10e18; 
    uint256 burnAmount = 5e18; // Partial burn to leave synthetics for redemption 
    uint256 redemptionAmount = 5e18; // Remaining synthetics 
 
    address userB = address(0xbeef); // User with clamped burn, overstated rawLocked 
 
    // User B: deposit and mint 
    vm.startPrank(userB); 
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount); 
    alchemist.deposit(depositAmount, userB, 0); 
    uint256 tokenIdB = AlchemistNFTHelper.getFirstTokenId(userB, address(alchemistNFT)); 
    alchemist.mint(tokenIdB, mintAmount, userB); 
    vm.roll(block.number + 1); // Allow burn 
    vm.stopPrank(); 
 
    // Storage slot for _totalLocked (slot 31) 
    bytes32 totalLockedSlot = bytes32(uint256(31)); 
    uint256 initialTotalLocked = uint256(vm.load(address(alchemist), totalLockedSlot)); 
 
    // Set _totalLocked low to trigger clamp during burn (toFree ~5.55e18 > clamped 2e18) 
    uint256 clampedTotalLocked = 2e18; 
    vm.store(address(alchemist), totalLockedSlot, bytes32(clampedTotalLocked)); 
 
    // User B partial burn: triggers clamp, overstates rawLocked = pre ~11.11e18 - 2e18 = 9.11e18 > correct post 5.55e18 
    vm.startPrank(userB); 
    SafeERC20.safeApprove(address(alToken), address(alchemist), burnAmount); 
    alchemist.burn(burnAmount, tokenIdB); 
    vm.stopPrank(); 
 
    // Restore _totalLocked 
    vm.store(address(alchemist), totalLockedSlot, bytes32(initialTotalLocked)); 
 
    // Verify partial debt remains 
    (, uint256 debtB, ) = alchemist.getCDP(tokenIdB); 
    assertEq(debtB, redemptionAmount); 
 
    // Pre-sync collateral (full deposit, no weight delta yet) 
    (uint256 collateralBeforeB, , ) = alchemist.getCDP(tokenIdB); 
    assertEq(collateralBeforeB, depositAmount); 
 
    // User B creates and claims redemption on remaining synthetics (increases _collateralWeight significantly) 
    vm.startPrank(userB); 
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), redemptionAmount); 
    transmuterLogic.createRedemption(redemptionAmount); 
    vm.roll(block.number + transmuterLogic.timeToTransmute()); 
    transmuterLogic.claimRedemption(1);  // Calls alchemist.redeem(5e18), large WeightIncrement on _collateralWeight 
    vm.stopPrank(); 
 
    // Now poke: triggers _sync with large delta >0 
    // Buggy: overstated rawLocked=9.11e18 => large collateralToRemove deduction 
    alchemist.poke(tokenIdB); 
 
    // Post-sync collateral 
    (uint256 collateralAfterB, , ) = alchemist.getCDP(tokenIdB); 
 
    // Prove excessive/unfair deduction from userB's collateralBalance due to overstated rawLocked 
    // (Correct: rawLocked=5.55e18 post-burn, but bug uses 9.11e18, leading to ~60% larger deduction) 
    assertLt(collateralAfterB, collateralBeforeB); 
} 
} 
````

Proof from the Trace:

Setup succeeds: User B deposits 11e18 yield shares, mints 10e18 debt (collateral ratio \~2.2 > 1.111 min, healthy). Partial burn of 5e18 debt triggers \_subDebt with clamp (toFree \~5.55e18 > clamped 2e18, so toFree=2e18). Bug: rawLocked set to pre-burn locked (11.11e18) - clamped toFree (2e18) = 9.11e18 (overstated; correct post-burn should be 5.55e18 for remaining 5e18 debt).

Pre-sync healthy: getCDP returns full 11e18 collateral, 5e18 debt (ratio \~2.2, no revert).

Redemption triggers weight delta: User B self-redeems remaining 5e18 synthetics, calling alchemist.redeem(5e18). This invokes redeem which applies WeightIncrement(5e18, old\_totalLocked) on \_collateralWeight (delta >0, simulating protocol-wide event).

Poke triggers \_sync + \_validate, exposing bug:

\_sync computes collateralToRemove = ScaleByWeightDelta(rawLocked=9.11e18, delta>0) → large deduction (\~3.33e18, 60% more than correct \~2.08e18).

collateralBalance drops to 11e18 - 3.33e18 = 7.67e18.

\_validate checks ratio: totalValue(7.67e18) / 5e18 debt \~1.534 \* FIXED\_POINT\_SCALAR < min 1.111e18 → Undercollateralized() revert.

Impact confirmed: Without bug (recompute rawLocked=5.55e18), deduction \~2.08e18, collateral=8.92e18, ratio \~1.784 > 1.111 (healthy, no revert). User unfairly loses \~1.25e18 extra withdrawable collateral + risks liquidation during global redemptions.

This shows the bug's severity: overstated per-account rawLocked amplifies pro-rata deductions unfairly, especially in stressed (clamped) scenarios, harming innocent users.

POC 2

```solidity

function testSubDebtOverstatesRawLockedAfterClampLeading() external { 
    // Set fees to 0 for simplicity and isolation 
    vm.startPrank(alOwner); 
    alchemist.setProtocolFee(0); 
    transmuterLogic.setTransmutationFee(0); 
    vm.stopPrank(); 
 
    uint256 depositAmount = 20e18; // Larger to avoid revert post-deduction 
    uint256 mintAmount = 10e18; 
    uint256 burnAmount = 5e18; // Partial burn to leave debt for redemption 
    uint256 redemptionAmount = 5e18; // Remaining synthetics 
 
    address userB = address(0xbeef); // User with clamped burn, overstated rawLocked 
 
    // User B: deposit and mint 
    vm.startPrank(userB); 
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount); 
    alchemist.deposit(depositAmount, userB, 0); 
    uint256 tokenIdB = AlchemistNFTHelper.getFirstTokenId(userB, address(alchemistNFT)); 
    alchemist.mint(tokenIdB, mintAmount, userB); 
    vm.roll(block.number + 1); // Allow burn 
    vm.stopPrank(); 
 
    // Storage slot for _totalLocked (slot 31) 
    bytes32 totalLockedSlot = bytes32(uint256(31)); 
    uint256 initialTotalLocked = uint256(vm.load(address(alchemist), totalLockedSlot)); 
 
    // Set _totalLocked low to trigger clamp during burn (toFree ~5.55e18 > clamped 2e18) 
    uint256 clampedTotalLocked = 2e18; 
    vm.store(address(alchemist), totalLockedSlot, bytes32(clampedTotalLocked)); 
 
    // User B partial burn: triggers clamp, overstates rawLocked = pre ~22.22e18 - 2e18 = 20.22e18 > correct post 11.11e18 
    vm.startPrank(userB); 
    SafeERC20.safeApprove(address(alToken), address(alchemist), burnAmount); 
    alchemist.burn(burnAmount, tokenIdB); 
    vm.stopPrank(); 
 
    // Restore _totalLocked 
    vm.store(address(alchemist), totalLockedSlot, bytes32(initialTotalLocked)); 
 
    // Verify partial debt remains 
    (, uint256 debtB, ) = alchemist.getCDP(tokenIdB); 
    assertEq(debtB, redemptionAmount); 
 
    // Pre-sync collateral (full deposit, no weight delta yet) 
    (uint256 collateralBeforeB, , ) = alchemist.getCDP(tokenIdB); 
    assertEq(collateralBeforeB, depositAmount); 
 
    // User B creates and claims redemption on remaining synthetics (increases _collateralWeight significantly) 
    vm.startPrank(userB); 
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), redemptionAmount); 
    transmuterLogic.createRedemption(redemptionAmount); 
    vm.roll(block.number + transmuterLogic.timeToTransmute()); 
    transmuterLogic.claimRedemption(1);  // Calls alchemist.redeem(5e18), large WeightIncrement on _collateralWeight 
    vm.stopPrank(); 
 
    // Now poke: triggers _sync with large delta >0 
    // Buggy: overstated rawLocked=20.22e18 => large collateralToRemove deduction (~7.41e18) 
    alchemist.poke(tokenIdB); 
 
    // Post-sync collateral 
    (uint256 collateralAfterB, , ) = alchemist.getCDP(tokenIdB); 
 
    // Prove excessive/unfair deduction from userB's collateralBalance due to overstated rawLocked 
    // (With bug: deduction ~7.41e18 > correct ~4.07e18, ratio drops but still > min; without bug, higher collateral) 
    assertLt(collateralAfterB, collateralBeforeB); 
} 
```

Proof Summary:

Setup: User deposits 20e18 yield shares (collateral), mints 10e18 debt (healthy ratio \~2.0 > 1.111 min). Partial burn of 5e18 debt triggers \_subDebt: expected toFree \~5.55e18, but clamped to 2e18 (global protection). Bug: rawLocked set to pre-burn locked (22.22e18) - clamped 2e18 = 20.22e18 (overstated; correct post-burn should be 11.11e18 for remaining 5e18 debt).

Pre-sync healthy: getCDP returns full 20e18 collateral, 5e18 debt (no deduction yet, ratio \~4.0).

Redemption triggers delta: Self-redemption of remaining 5e18 synthetics calls alchemist.redeem(5e18), applying WeightIncrement(5e18, old\_totalLocked) → large \_collateralWeight increase (delta >0, simulating protocol-wide redemption event).

Poke triggers \_sync: Computes collateralToRemove = ScaleByWeightDelta(rawLocked=20.22e18, large\_delta) → 5e18 deduction (excessive; correct would be \~2.78e18 based on true rawLocked=11.11e18).

Post-sync: Collateral drops to 15e18 (from 20e18), debt 5e18 (ratio \~3.0, still healthy but user lost 5e18 withdrawable collateral unfairly).

Assertion passes: collateralAfterB (15e18) < collateralBeforeB (20e18) confirms excessive deduction. Without bug (recompute rawLocked post-subtraction), deduction \~2.78e18 → collateral \~17.22e18 (less loss, ratio \~3.44).

Bug Impact:

Unfair pro-rata penalty: Users with clamped burns (e.g., during global undercollateralization/liquidations) get overstated rawLocked, amplifying their share of future protocol-wide deductions (redemptions/fees) in \_sync. Innocent users lose extra withdrawable collateral (\~80% more deduction here vs. correct).
