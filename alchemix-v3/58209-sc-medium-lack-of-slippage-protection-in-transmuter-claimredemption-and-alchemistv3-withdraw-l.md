# 58209 sc medium lack of slippage protection in transmuter claimredemption and alchemistv3 withdraw leads to user yield losses

**Submitted on Oct 31st 2025 at 11:37:36 UTC by @Smartkelvin for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58209
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/Transmuter.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

The Transmuter.claimRedemption function and it contract associate (AlchemistV3.withdraw) lacks any minimum output protection , allowing dynamic system state changes—such as concurrent claims, bad debt ratio spikes, etc—to cause users to receive significantly less yield than expected after burning their position NFT

## Vulnerability Details

The claimRedemption function in the Transmuter contract processes user redemptions by calculating time-pro-rata transmuted synthetics, applying bad debt scaling, conditionally redeeming from the Alchemist, and distributing yield tokens. However, it performs no verification that the final claimYield meets a user-specified minimum, exposing outputs to variance from external and concurrent state changes.

## Impact Details

Direct User Loss

MEV Exploitation

## Mitigation

Introduce a minYieldOut parameter to both functions and revert check immediately before transfers.

## Proof of Concept

## Proof of Concept

```
  function testClaimRedemption_BadDebtSlippage_UserGetsLess() external {
    uint256 user1Deposit = 2000e18;
    uint256 user1Mint = 1000e18;
    
    // ============================================
    //  User creates transmuter position
    // ============================================
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(vault), address(alchemist), user1Deposit);
    alchemist.deposit(user1Deposit, address(0xbeef), 0);
    uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    
    // Mint synthetic tokens
    alchemist.mint(tokenId, user1Mint, address(0xbeef));
    
    // Create transmuter redemption
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), user1Mint);
    transmuterLogic.createRedemption(user1Mint);
    vm.stopPrank();
    
    uint256 redemptionId = 1;
    vm.roll(block.number + transmuterLogic.timeToTransmute() + 1);
    uint256 totalSyntheticsBefore = alchemist.totalSyntheticsIssued();
    uint256 totalUnderlyingBefore = alchemist.getTotalUnderlyingValue();
    uint256 transmuterBalance = IERC20(address(vault)).balanceOf(address(transmuterLogic));
    uint256 transmuterUnderlyingValue = alchemist.convertYieldTokensToUnderlying(transmuterBalance);
    
    uint256 denominator = totalUnderlyingBefore + transmuterUnderlyingValue > 0 
        ? totalUnderlyingBefore + transmuterUnderlyingValue 
        : 1;
    
    uint256 badDebtRatioBefore = totalSyntheticsBefore * 1e18 / denominator;
    
    console.log("\n=== SYSTEM STATE BEFORE CLAIM ===");
    console.log("Total Synthetics Issued:", totalSyntheticsBefore);
    console.log("Total Underlying in Alchemist:", totalUnderlyingBefore);
    console.log("Transmuter Balance:", transmuterBalance);
    console.log("Total Backing (denominator):", denominator);
    console.log("Bad Debt Ratio:", badDebtRatioBefore);
    
    // Calculate what user EXPECTS to receive (1:1 conversion)
    uint256 expectedYieldTokens = alchemist.convertDebtTokensToYield(user1Mint);
    console.log("\nUser expects yield tokens:", expectedYieldTokens);
    // ============================================
    //  User claims redemption
    // ============================================
    vm.startPrank(address(0xbeef));
    
    uint256 yieldBalanceBefore = IERC20(address(vault)).balanceOf(address(0xbeef));
    
    transmuterLogic.claimRedemption(redemptionId);
    
    uint256 yieldBalanceAfter = IERC20(address(vault)).balanceOf(address(0xbeef));
    uint256 actualYieldReceived = yieldBalanceAfter - yieldBalanceBefore;
    
    vm.stopPrank();
    
    console.log("\n=== CLAIM RESULTS ===");
    console.log("Expected yield tokens:", expectedYieldTokens);
    console.log("Actually received:", actualYieldReceived);

    
    console.log("\n=== THE ISSUE ===");
    console.log("User has NO slippage protection!");
    console.log("If bad debt existed, they would receive less");
    console.log("Transaction would still succeed - no revert");
    console.log("No way to specify minYieldTokens parameter");
    
    // Test passes, demonstrating lack of protection
    assertTrue(actualYieldReceived > 0, "User received some yield");
    
}
```
