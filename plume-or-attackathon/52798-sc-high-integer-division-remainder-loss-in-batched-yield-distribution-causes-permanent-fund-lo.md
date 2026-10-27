# 52798 sc high integer division remainder loss in batched yield distribution causes permanent fund lock

**Submitted on Aug 13th 2025 at 09:17:55 UTC by @ZeroExRes for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52798
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:** Permanent freezing of funds

## Description

### Brief/Intro

The `distributeYieldWithLimit()` function contains a flaw where integer division remainders from each batch are permanently lost and locked in the contract, while the regular `distributeYield()` function correctly handles these remainders. This creates an accumulating fund loss mechanism that can result in significant monetary losses over time, with no recovery method available.

### Vulnerability Details

The issue stems from different remainder handling between two yield distribution functions:

`distributeYield()` (Works Correctly):

```solidity
// Processes all holders except last
for (uint256 i = 0; i < lastProcessedIndex; i++) {
    uint256 share = (amount * holderBalance) / effectiveTotalSupply;
    // ... distribute share
    distributedSum += share;
}

// CRITICAL: Last holder gets ALL remainder
uint256 lastShare = amount - distributedSum;  // ← Captures remainder
yToken.safeTransfer(lastHolder, lastShare);
```

`distributeYieldWithLimit()` (Loses Remainder):

```solidity
for (uint256 i = 0; i < batchSize; i++) {
    uint256 share = (totalAmount * holderBalance) / effectiveTotalSupply;
    // ... distribute share  
    amountDistributed += share;
}
// ← NO remainder handling - funds permanently stuck!
```

### Impact Details

Each batched distribution loses integer division remainders from every user calculation, with no recovery mechanism available. Even though there is no significant loss for a particular user, these per-user remainders accumulate into non-trivial amounts with each distribution, creating systematic fund loss that compounds over the contract's operational lifetime.

{% hint style="danger" %}
Permanent freezing of funds: the remainders accumulated per batch are not returned to recipients nor tracked for later distribution, so funds become permanently locked in the contract balance.
{% endhint %}

## Proof of Concept

Add to ArcToken.t.sol:

{% code title="ArcToken.t.sol" %}

```solidity
function test_YieldRemainderLoss_Comparison() public {    
    // Setup: Create 3 equal holders to guarantee remainder
    token.transfer(bob, 100e18);    // Bob: 100e18
    token.transfer(charlie, 100e18); // Charlie: 100e18
    // Alice already has: 100e18 
    // Owner now has: 700e18
    
    uint256 yieldAmount = 99;
    
    console.log("Setup - Token balances:");
    console.log("  Owner:", token.balanceOf(owner) / 1e18, "tokens");
    console.log("  Alice:", token.balanceOf(alice) / 1e18, "tokens");
    console.log("  Bob:", token.balanceOf(bob) / 1e18, "tokens");
    console.log("  Charlie:", token.balanceOf(charlie) / 1e18, "tokens");
    console.log("Yield to distribute:", yieldAmount);
    console.log("");

    // ========== TEST 1: Regular Distribution ==========
    console.log("=== TEST 1: distributeYield() ===");
    
    yieldToken.approve(address(token), yieldAmount);
    uint256 contractBefore1 = yieldToken.balanceOf(address(token));
    uint256 ownerBefore1 = yieldToken.balanceOf(owner);
    
    token.distributeYield(yieldAmount);
    
    uint256 contractAfter1 = yieldToken.balanceOf(address(token));
    uint256 ownerAfter1 = yieldToken.balanceOf(owner);
    uint256 aliceAfter1 = yieldToken.balanceOf(alice);
    uint256 bobAfter1 = yieldToken.balanceOf(bob);
    uint256 charlieAfter1 = yieldToken.balanceOf(charlie);
    
    uint256 ownerReceived1 = ownerAfter1 - (ownerBefore1 - yieldAmount);
    uint256 totalDistributed1 = ownerReceived1 + aliceAfter1 + bobAfter1 + charlieAfter1;
            
    console.log("Contract balance change:", contractAfter1 - contractBefore1);
    console.log("Total distributed:", totalDistributed1);
    console.log("Funds lost:", yieldAmount - totalDistributed1);
    console.log("");

    // ========== TEST 2: Batched Distribution ==========
    console.log("=== TEST 2: distributeYieldWithLimit() ===");
    
    yieldToken.mint(owner, yieldAmount);
    yieldToken.approve(address(token), yieldAmount);
    uint256 contractBefore2 = yieldToken.balanceOf(address(token));
    
    uint256 totalDistributed2 = 0;
    uint256 nextIndex = 0;
    uint256 batchCount = 0;
    
    do {
        (uint256 newNextIndex, , uint256 batchDistributed) = 
            token.distributeYieldWithLimit(yieldAmount, nextIndex, 2);
        totalDistributed2 += batchDistributed;
        nextIndex = newNextIndex;
        batchCount++;
        console.log("Batch", batchCount, "distributed:", batchDistributed);
    } while (nextIndex != 0);
    
    uint256 contractAfter2 = yieldToken.balanceOf(address(token));
    
    console.log("Contract balance change:", contractAfter2 - contractBefore2);
    console.log("Total distributed:", totalDistributed2);
    console.log("Funds lost:", yieldAmount - totalDistributed2);
    console.log("");

    // ========== CRITICAL COMPARISON ==========
    uint256 regularLoss = yieldAmount - totalDistributed1;
    uint256 batchedLoss = yieldAmount - totalDistributed2;
    uint256 extraStuckFunds = (contractAfter2 - contractBefore2) - (contractAfter1 - contractBefore1);
    
    console.log("Regular distribution lost:", regularLoss);
    console.log("Batched distribution lost:", batchedLoss);
    console.log("Extra funds stuck from batching:", extraStuckFunds);

    // Assertions to prove the issue
    assertEq(regularLoss, 0, "Regular distribution should lose no funds");
    assertGt(batchedLoss, 0, "Batched distribution should lose funds due to remainder bug");
    assertGt(contractAfter2, contractAfter1, "More funds stuck after batched distribution");
}
```

{% endcode %}

## References

Mentioned above

***

If you want, I can:

* Suggest concrete fixes (e.g., accumulate distributed sum per batch and forward remainder to a designated recipient or track leftover for later distribution), or
* Produce a small patch diff for ArcToken.sol implementing correct remainder handling in distributeYieldWithLimit().
