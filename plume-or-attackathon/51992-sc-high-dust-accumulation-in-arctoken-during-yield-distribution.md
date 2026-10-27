# 51992 sc high dust accumulation in arctoken during yield distribution&#x20;

**Submitted on Aug 7th 2025 at 06:21:29 UTC by @Killua for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51992
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>

## Description

### Brief / Intro

In `ArcToken::distributeYield`, the yield distribution mechanism has a logical flaw which causes dust accumulation of yieldTokens in the contract. There is no function to withdraw these dust yield tokens.

### Vulnerability Details

* `distributeYield()` uses a "last holder gets remainder" approach to handle rounding errors, but fails to account for cases where the last holder is yield-restricted (e.g., blacklisted from receiving yields).
* The last holder is intended to receive `lastShare = amount - distributedSum` to ensure complete distribution without dust.
* If the last holder is not allowed to receive yield, this `lastShare` remains in the contract.
* There is no function to withdraw accumulated yieldTokens from the contract.
* Distribution cycles use only the input `amount` parameter and never the contract's existing balance of yieldTokens, so each cycle compounds the problem and increases dust.

### Impact Details

* Not all yield is distributed to holders; some yield becomes stuck in the contract.
* Over time, the dust amount increases with each distribution cycle.
* Without a recovery mechanism, these funds remain stuck.

{% hint style="warning" %}
High-severity: Funds (yieldTokens) become permanently stuck in the contract if recipients are skipped (e.g., yield-restricted). This leads to progressively increasing undistributed yield.
{% endhint %}

## Proof of Concept

<details>

<summary>Test reproducing dust accumulation (expand to view)</summary>

```
function test_YieldDustAccumulation() public {
        // Setup: Give tokens to bob and charlie
        vm.prank(owner);
        token.transfer(bob, 20e18);      // Bob: 20e18
        vm.prank(owner);
        token.transfer(charlie, 20e18);  // Charlie: 20e18
        // Now we have: Owner: 860e18, Alice: 100e18, Bob: 20e18, Charlie: 20e18
        
        // Blacklist charlie (will be skipped in distribution)
        yieldBlacklistModule.addToBlacklist(charlie);
        
        // Record contract balance before distribution
        uint256 contractBalanceBefore = yieldToken.balanceOf(address(token));
   
    uint256 aliceYieldBefore = yieldToken.balanceOf(alice);
    uint256 bobYieldBefore = yieldToken.balanceOf(bob);
    uint256 charlieYieldBefore = yieldToken.balanceOf(charlie);
    

        // Distribute yield 
        //3 cycles of 100e18 yield
       
 for (uint256 i = 0; i < 3; i++) {
        uint256 yieldAmount = 100e18;
        yieldToken.approve(address(token), yieldAmount);
        token.distributeYield(yieldAmount);
     
  // Record balances after distribution
    uint256 contractBalanceAfter = yieldToken.balanceOf(address(token));
    uint256 aliceYieldAfter = yieldToken.balanceOf(alice);
    uint256 bobYieldAfter = yieldToken.balanceOf(bob);
    uint256 charlieYieldAfter = yieldToken.balanceOf(charlie);
    
   

    console.log("=== Net balances after cycle %d ===", i);
    console.log("Alice received: %e", aliceYieldAfter - aliceYieldBefore);
    console.log("Bob received:", bobYieldAfter - bobYieldBefore);
    console.log("Charlie received (should be 0):", charlieYieldAfter - charlieYieldBefore);
    console.log("contract yield token balance remaining:", contractBalanceAfter - contractBalanceBefore);
  
        console.log("Dust accumulated after cycle %d:", i, contractBalanceAfter - contractBalanceBefore);
    }
}
```

</details>

## Notes / Remediation Ideas

* Ensure the "remainder to last holder" logic only applies to eligible yield recipients; if the intended last recipient is ineligible, select the last eligible recipient or redistribute the remainder.
* Alternatively, change distribution logic to consider the contract's existing yieldToken balance (aggregate available yield) and handle rounding such that no tokens are left undistributable.
* Provide an admin recovery function to withdraw or redistribute dust balances (with appropriate access controls and auditing).
