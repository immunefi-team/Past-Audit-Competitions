# 56714 sc high accounting invariant violation in forcerepay leads to protocol insolvency

**Submitted on Oct 19th 2025 at 20:44:55 UTC by @HandsomeEarthworm6 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56714
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The \_forceRepay function in AlchemistV3 fails to update the global cumulativeEarmarked counter when repaying a user's earmarked debt during liquidation. This breaks a critical accounting invariant that the protocol depends on for all debt management operations.

The Invariant: cumulativeEarmarked = Σ(all users' account.earmarked)

This invariant is fundamental to the protocol's earmarking and redemption system, which manages how debt transitions from normal to earmarked status over time, and how redemptions are distributed fairly across all borrowers.

## Vulnerability Details

In the \_forceRepay function , when earmarked debt is repaid, only the user's individual account.earmarked is decremented, but the global cumulativeEarmarked counter is not updated:

function \_forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) { if (amount == 0) { return 0; } \_checkForValidAccountId(accountId); Account storage account = \_accounts\[accountId];

```
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
    account.earmarked -= earmarkToRemove;  //  @audit --User counter reduced

// @audit --MISSING: cumulativeEarmarked -= earmarkToRemove;


    creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield;
    account.collateralBalance -= creditToYield;

    uint256 protocolFeeTotal = creditToYield * protocolFee / BPS;

    emit ForceRepay(accountId, amount, creditToYield, protocolFeeTotal);

    if (account.collateralBalance > protocolFeeTotal) {
        account.collateralBalance -= protocolFeeTotal;
        // Transfer the protocol fee to the protocol fee receiver
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
    }

    if (creditToYield > 0) {
        // Transfer the repaid tokens from the account to the transmuter.
        TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
    }
    return creditToYield;
}
```

## Impact Details

Once the invariant is broken, the protocol suffers cascading failures:

1.Incorrect Unearmarked Debt Calculation: The \_earmark() function calculates available unearmarked debt as:

uint256 liveUnearmarked = totalDebt - cumulativeEarmarked; If cumulativeEarmarked is inflated (not decremented during force repay), liveUnearmarked becomes artificially low or even underflows to zero, preventing any new debt from being earmarked.

2.Broken Weight System: The earmark weight is calculated as:

\_earmarkWeight += PositionDecay.WeightIncrement(amount, liveUnearmarked); With incorrect liveUnearmarked, the entire weight system (earmark weight, redemption weight, collateral weight, survival accumulator) becomes corrupted, affecting all CDP sync operations.

3.Redemption Failures: The redeem() function caps redemptions at:

uint256 liveEarmarked = cumulativeEarmarked; if (amount > liveEarmarked) amount = liveEarmarked;

```
   The system believes more debt is earmarked than actually exists, causing incorrect redemption distributions.

4. User Fund Lock-- Users with healthy collateral may be unable to borrow because the system incorrectly believes all debt is already earmarked, blocking normal protocol operations.

5. Protocol Insolvency- Over time, as more liquidations occur, the discrepancy compounds, potentially rendering the entire protocol insolvent as accounting diverges further from reality.

This is a critical severity vulnerability because:
- It breaks a fundamental accounting invariant
- It affects core protocol functionality (borrowing, earmarking, redemptions)
- It causes permanent state corruption that compounds over time
- It can lead to protocol insolvency
- It is triggered by normal protocol operations (liquidations)

## References
n/a

## Proof of Concept

## Proof of Concept

add this test to AlchemistV3.t.sol  and the run forge test --match-test testForceRepay_DirectCall_Isolated  -vvv, to observe the that AlchemistV3 fails to update the global cumulativeEarmarked counter when repaying a user's earmarked debt during liquidation.This test triggers liquidation which inturns triggers force repay and the we have users earmarked debt being subtracted but not the cumulative earmarked .



function testForceRepay_DirectCall_Isolated() external {
    // Setup user with earmarked debt
    vm.startPrank(address(0xbeef));
    uint256 depositAmount = 100e18;
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
    alchemist.deposit(depositAmount, address(0xbeef), 0);
    uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    
    // Mint at minimum collateralization (90% LTV = 111.11% collateralization)
    uint256 debtAmount = alchemist.totalValue(tokenId) * FIXED_POINT_SCALAR / minimumCollateralization;
    alchemist.mint(tokenId, debtAmount, address(0xbeef));
    vm.stopPrank();
    
    console.log("Initial debt:", debtAmount);
    console.log("Initial collateral value:", alchemist.totalValue(tokenId));
    
    // Create redemption to earmark 50% of debt
    vm.startPrank(anotherExternalUser);
    uint256 redemptionAmount = debtAmount / 2;
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), redemptionAmount);
    transmuterLogic.createRedemption(redemptionAmount);
    vm.stopPrank();
    
    // Fast forward and sync to apply earmark
    vm.roll(block.number + 5_256_000);
    alchemist.poke(tokenId);
    
    (uint256 collateralBefore, uint256 debtBefore, uint256 earmarkedBefore) = alchemist.getCDP(tokenId);
    uint256 cumulativeEarmarkedBefore = alchemist.cumulativeEarmarked();
    
    console.log("\n=== BEFORE LIQUIDATION ===");
    console.log("Collateral:", collateralBefore);
    console.log("Debt:", debtBefore);
    console.log("Earmarked:", earmarkedBefore);
    console.log("Cumulative Earmarked:", cumulativeEarmarkedBefore);
    console.log("Collateralization Lower Bound:", alchemist.collateralizationLowerBound());
    
    // Verify we have earmarked debt
    assertTrue(earmarkedBefore > 0, "Should have earmarked debt");
    assertTrue(cumulativeEarmarkedBefore >= earmarkedBefore, "Global should track user earmark");
    
    // Drop price to make position undercollateralized
    // Current: 100 collateral, debt 90, ratio = 111.11%
    // Target: Drop to ~104% (below 105.26% lower bound)
    // If debt is 90 and we want collateral value = 93.6, price should be 0.936
    // Price = underlying / supply, so to decrease price, we INCREASE supply
    // Current: price = 1e24 / 1e24 = 1.0
    // Target: price = 1e24 / X = 0.936, so X = 1e24 / 0.936 = 1.068376... e24
    uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
    
    // Increase supply by ~6.8% to drop price to ~0.936
    uint256 modifiedVaultSupply = (initialVaultSupply * 1069) / 1000; // 106.9% of original
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);
    
    // Verify position is now undercollateralized
    uint256 newCollateralValue = alchemist.totalValue(tokenId);
    uint256 currentRatio = newCollateralValue * FIXED_POINT_SCALAR / debtBefore;
    console.log("\n=== AFTER PRICE DROP ===");
    console.log("New collateral value:", newCollateralValue);
    console.log("Current ratio:", currentRatio);
    console.log("Is undercollateralized:", currentRatio < alchemist.collateralizationLowerBound());
    
    // Ensure it's actually undercollateralized
    require(currentRatio < alchemist.collateralizationLowerBound(), "Position must be undercollateralized");
    
    // This should now trigger liquidation via _forceRepay
    vm.startPrank(externalUser);
    (uint256 amountLiquidated, uint256 feeInYield,) = alchemist.liquidate(tokenId);
    vm.stopPrank();
    
    (uint256 collateralAfter, uint256 debtAfter, uint256 earmarkedAfter) = alchemist.getCDP(tokenId);
    uint256 cumulativeEarmarkedAfter = alchemist.cumulativeEarmarked();
    
    console.log("\n=== AFTER LIQUIDATION ===");
    console.log("Collateral:", collateralAfter);
    console.log("Debt:", debtAfter);
    console.log("Earmarked:", earmarkedAfter);
    console.log("Cumulative Earmarked:", cumulativeEarmarkedAfter);
    console.log("Amount liquidated:", amountLiquidated);
    console.log("Fee:", feeInYield);
    
    uint256 earmarkedReduced = earmarkedBefore - earmarkedAfter;
    uint256 cumulativeReduced = cumulativeEarmarkedBefore - cumulativeEarmarkedAfter;
    
    console.log("\n=== COMPARISON ===");
    console.log("User earmarked reduced by:", earmarkedReduced);
    console.log("Global cumulative reduced by:", cumulativeReduced);
    console.log("MATCH?", earmarkedReduced == cumulativeReduced);
    
    // THE BUG: These should match but won't if _forceRepay doesn't update cumulativeEarmarked
    if (earmarkedReduced > 0) {
        assertEq(cumulativeReduced, earmarkedReduced, "BUG FOUND: cumulativeEarmarked not properly updated in _forceRepay");
    }
}
```
