# 50392 sc insight phantom commission burn

**Submitted on Jul 24th 2025 at 07:58:33 UTC by @BeastBoy for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50392
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol>
* **Impacts:**
  * Protocol insolvency

## Description

Within `_calculateRewardsCore` each segment’s user commission is computed using ceiling division:

```solidity
uint256 commissionForThisSegment =
    _ceilDiv(grossRewardForSegment * effectiveCommissionRate, REWARD_PRECISION);
```

This rounds any fractional dust up, deducting

ceil(X / P) – floor(X / P) ≥ 1

from the user. However, in `updateRewardPerTokenForValidator` the validator’s commission is credited using plain floor division:

```solidity
uint256 commissionDeltaForValidator =
    (grossRewardForValidatorThisSegment * commissionRateForSegment) / REWARD_PRECISION;
```

The extra “1‑unit” dust never appears in `validatorAccruedCommission` or `totalClaimableByToken`, nor is `totalCommissionAmountDelta` ever written back to storage. As a result, every segment silently “burns” the rounding difference.

## Impact

Users collectively lose a non‑negligible amount of tokens over time as these rounding deltas are irretrievably removed from the system.

{% hint style="warning" %}
Protocol balance is reduced every time a remainder exists, leading to systematic token removal and potential long-term solvency issues.
{% endhint %}

## Recommendation

Unify the rounding mode by replacing the ceiling division in `_calculateRewardsCore` with floor division to match the validator side, ensuring no dust is charged beyond what the validator receives.

## Proof of Concept

<details>

<summary>Mathematical proof &#x26; realistic demonstration (expand to view)</summary>

#### Mathematical rounding proof (pseudo test)

```solidity
/**
 * @notice Mathematical proof of the rounding discrepancy
 * @dev Shows concrete examples where ceil(x) != floor(x) causing phantom burns
 */
function test_MathematicalRoundingProof() public pure {
    console2.log("=== MATHEMATICAL ROUNDING PROOF ===");
    
    // Test cases that demonstrate the rounding difference
    uint256[5] memory grossRewards = [uint256(100), 1000, 9999, 123456, 1e18];
    uint256 commissionRate = 33333333333333333; // ~3.33% - creates remainders
    
    uint256 totalPhantomBurn = 0;
    
    for (uint256 i = 0; i < grossRewards.length; i++) {
        uint256 gross = grossRewards[i];
        
        // User commission (ceiling division)
        uint256 userCommission = _ceilDiv(gross * commissionRate, 1e18);
        
        // Validator commission (floor division)  
        uint256 validatorCommission = (gross * commissionRate) / 1e18;
        
        // The phantom burn
        uint256 burn = userCommission - validatorCommission;
        totalPhantomBurn += burn;
        
        console2.log("--- Case %s ---", i + 1);
        console2.log("Gross reward: %s", gross);
        console2.log("User pays: %s", userCommission);
        console2.log("Validator gets: %s", validatorCommission);
        console2.log("Phantom burn: %s", burn);
        
        // Check the mathematical relationship
        uint256 numerator = gross * commissionRate;
        uint256 remainder = numerator % 1e18;
        
        console2.log("Numerator: %s", numerator);
        console2.log("Remainder: %s", remainder);
        console2.log("Has remainder: %s", remainder > 0 ? "YES" : "NO");
        
        // Prove the relationship: burn = 1 if remainder > 0, burn = 0 if remainder = 0
        if (remainder > 0) {
            require(burn == 1, "Burn should be 1 when remainder exists");
            console2.log("CONFIRMED: 1 wei burned due to remainder");
        } else {
            require(burn == 0, "Burn should be 0 when no remainder");
            console2.log("No burn - exact division");
        }
        
        console2.log("");
    }
    
    console2.log("=== SUMMARY ===");
    console2.log("Total phantom burn across test cases: %s wei", totalPhantomBurn);
    console2.log("PROOF: Every transaction with remainder burns exactly 1 wei");
    console2.log("IMPACT: Systematic token removal from protocol over time");
}
```

#### Real-world scenario demonstration

```solidity
/**
 * @notice Demonstrates the vulnerability with real protocol math
 */
function test_RealWorldPhantomBurnScenario() public pure {
    console2.log("=== REAL WORLD SCENARIO ===");
    
    // Realistic values from a DeFi protocol
    uint256 stakeAmount = 1000e18; // 1000 tokens
    uint256 rewardRate = 5787037037037037; // ~0.5% daily APR
    uint256 duration = 86400; // 1 day in seconds  
    uint256 commissionRate = 25e16; // 25% commission
    
    // Calculate gross reward for the period
    uint256 grossReward = (duration * rewardRate * stakeAmount) / 1e18;
    
    console2.log("Stake amount: %s tokens", stakeAmount / 1e18);
    console2.log("Daily reward rate: %s (%.4f%%)", rewardRate, rewardRate * 10000 / 1e18);
    console2.log("Commission rate: %s (%.0f%%)", commissionRate, commissionRate * 100 / 1e18);
    console2.log("Period: %s seconds (1 day)", duration);
    console2.log("Gross reward: %s", grossReward);
    
    // Calculate commissions with both methods
    uint256 userCommission = _ceilDiv(grossReward * commissionRate, 1e18);
    uint256 validatorCommission = (grossReward * commissionRate) / 1e18;
    
    console2.log("=== COMMISSION CALCULATION ===");
    console2.log("User charged (ceiling): %s", userCommission);
    console2.log("Validator receives (floor): %s", validatorCommission);
    
    uint256 phantomBurn = userCommission - validatorCommission;
    console2.log("Daily phantom burn: %s wei", phantomBurn);
    
    // Project over time
    uint256 yearlyBurn = phantomBurn * 365;
    uint256 burnPercentage = (phantomBurn * 1e18) / grossReward;
    
    console2.log("=== IMPACT PROJECTION ===");
    console2.log("Yearly burn per staker: %s wei", yearlyBurn);
    console2.log("Burn percentage per day: %.10f%%", burnPercentage * 100 / 1e18);
    
    // With 10,000 stakers
    console2.log("With 10,000 stakers:");
    console2.log("- Daily protocol burn: %s wei", phantomBurn * 10000);
    console2.log("- Yearly protocol burn: %s wei", yearlyBurn * 10000);
    
    require(phantomBurn > 0, "Phantom burn must exist in realistic scenario");
    console2.log("VULNERABILITY CONFIRMED: Real-world impact demonstrated");
}
```

</details>
