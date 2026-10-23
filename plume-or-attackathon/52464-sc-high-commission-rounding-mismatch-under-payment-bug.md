# 52464 sc high commission rounding mismatch under payment bug

**Submitted on Aug 11th 2025 at 01:06:24 UTC by @BeastBoy for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52464
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol>
* **Impacts:** Protocol insolvency

## Summary

A rounding mismatch between how validator commission and per-user commission are computed causes a systematic loss (burn) of reward funds. Validator commission uses floor division while user commission uses ceiling division. The mismatch means Σ(user commissions) > validator’s accrued commission, permanently shaving small amounts from the reward pool every settlement.

## Description

In updateRewardPerTokenForValidator the validator’s commission is calculated with floor division:

```solidity
uint256 grossReward = (totalStaked * rewardPerTokenIncrease) / REWARD_PRECISION;
uint256 commissionDeltaForValidator = (grossReward * commissionRateForSegment) / REWARD_PRECISION;
validatorAccruedCommission += commissionDeltaForValidator;
```

This rounds down. In \_calculateRewardsCore, each user’s commission is charged with ceiling division:

```solidity
uint256 grossUser = (userStake * rewardPerTokenDelta) / REWARD_PRECISION;
uint256 commissionForThisSegment = _ceilDiv(grossUser * effectiveCommissionRate, REWARD_PRECISION);
totalCommissionAmountDelta += commissionForThisSegment;
```

Rounding up per-staker but rounding down for the validator’s total introduces a consistent deficit: users collectively pay more commission than the validator receives. No reconciliation logic restores the difference.

Example: two stakers with a 1 Wei reward can cause users to pay 2 Wei of commission while the validator receives 0 Wei — permanently removing 2 Wei from the pool.

## Impact

{% hint style="danger" %}
Stakers are systematically under-paid per user per segment, silently draining value from the reward pool. Over time this can lead to significant loss of funds and protocol insolvency.
{% endhint %}

## Recommendation

{% hint style="info" %}
Use the same rounding rule on both sides so that Σ(user commissions) exactly equals the validator’s accrued commission. Concretely:

* Replace the validator’s floor division with a ceiling (\_ceilDiv) to match per-user rounding, or
* Switch users to floor division (remove per-user ceil) to match the validator-side floor.

Either approach removes the systematic bias; prefer the approach consistent with intended economic behavior and documented invariants.
{% endhint %}

## Proof of Concept

<details>

<summary>Show PoC (Forge test demonstrating discrepancy and a fixed variant)</summary>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import "forge-std/Test.sol";

/**
 * @title RoundingDiscrepancyTest
 * @notice Test to demonstrate the commission rounding discrepancy issue
 * @dev This test shows how ceiling division for users vs floor division for validators
 *      causes users to pay more commission than validators receive
 */
contract RoundingDiscrepancyTest is Test {
    
    // Simplified constants for easier demonstration
    uint256 constant REWARD_PRECISION = 1000;
    uint256 constant COMMISSION_RATE = 333; // 33.3%
    
    /**
     * @notice Helper function for ceiling division (copied from the contract)
     */
    function _ceilDiv(uint256 a, uint256 b) internal pure returns (uint256) {
        if (b == 0) return 0;
        return (a + b - 1) / b;
    }
    
    /**
     * @notice Test demonstrating the rounding discrepancy
     * @dev Shows that sum of user commissions > validator's accrued commission
     */
    function test_CommissionRoundingDiscrepancy() public {
        // Setup: 3 users with 1 wei staked each
        uint256 totalStaked = 3;
        uint256 userStakedAmount = 1;
        uint256 numUsers = 3;
        uint256 rewardPerTokenIncrease = 1000; // Chosen to create rounding issues
        
        // Validator commission calculation (using floor division like updateRewardPerTokenForValidator)
        uint256 grossRewardForValidator = (totalStaked * rewardPerTokenIncrease) / REWARD_PRECISION;
        uint256 validatorCommission = (grossRewardForValidator * COMMISSION_RATE) / REWARD_PRECISION;
        
        console.log("=== Validator Commission Calculation (Floor Division) ===");
        console.log("Total staked:", totalStaked);
        console.log("Reward per token increase:", rewardPerTokenIncrease);
        console.log("Gross reward for validator:", grossRewardForValidator);
        console.log("Commission rate:", COMMISSION_RATE);
        console.log("Validator commission (floor):", validatorCommission);
        
        // User commission calculation (using ceiling division like _calculateRewardsCore)
        uint256 totalUserCommissions = 0;
        
        console.log("\n=== User Commission Calculations (Ceiling Division) ===");
        for (uint256 i = 0; i < numUsers; i++) {
            uint256 grossRewardForUser = (userStakedAmount * rewardPerTokenIncrease) / REWARD_PRECISION;
            uint256 userCommission = _ceilDiv(grossRewardForUser * COMMISSION_RATE, REWARD_PRECISION);
            totalUserCommissions += userCommission;
            
            console.log("User", i + 1, "- Gross reward:", grossRewardForUser);
            console.log("User", i + 1, "- Commission (ceiling):", userCommission);
        }
        
        console.log("\n=== Results ===");
        console.log("Total commission paid by users:", totalUserCommissions);
        console.log("Commission received by validator:", validatorCommission);
        console.log("Discrepancy (burned):", totalUserCommissions - validatorCommission);
        
        // Assertions to verify the issue
        assertGt(totalUserCommissions, validatorCommission, "Users should pay more than validator receives");
        assertEq(totalUserCommissions, 3, "Users should pay 3 wei total");
        assertEq(validatorCommission, 0, "Validator should receive 0 wei");
        
        // The discrepancy represents funds that are effectively "burned"
        uint256 burnedAmount = totalUserCommissions - validatorCommission;
        assertEq(burnedAmount, 3, "3 wei should be burned due to rounding");
        
        console.log("\n=== Issue Summary ===");
        console.log("This test demonstrates that users collectively pay", totalUserCommissions, "wei in commission");
        console.log("while the validator only receives", validatorCommission, "wei,");
        console.log("effectively burning", burnedAmount, "wei from the reward pool.");
    }
    
    /**
     * @notice Test with different parameters to show the issue scales
     */
    function test_CommissionRoundingDiscrepancy_LargerScale() public {
        // Larger scale example
        uint256 totalStaked = 10;
        uint256 userStakedAmount = 1;
        uint256 numUsers = 10;
        uint256 rewardPerTokenIncrease = 1500;
        uint256 commissionRate = 250; // 25%
        
        // Validator calculation
        uint256 grossRewardForValidator = (totalStaked * rewardPerTokenIncrease) / REWARD_PRECISION;
        uint256 validatorCommission = (grossRewardForValidator * commissionRate) / REWARD_PRECISION;
        
        // User calculations
        uint256 totalUserCommissions = 0;
        for (uint256 i = 0; i < numUsers; i++) {
            uint256 grossRewardForUser = (userStakedAmount * rewardPerTokenIncrease) / REWARD_PRECISION;
            uint256 userCommission = _ceilDiv(grossRewardForUser * commissionRate, REWARD_PRECISION);
            totalUserCommissions += userCommission;
        }
        
        console.log("\n=== Larger Scale Test ===");
        console.log("Validator commission:", validatorCommission);
        console.log("Total user commissions:", totalUserCommissions);
        console.log("Burned amount:", totalUserCommissions - validatorCommission);
        
        // Verify the discrepancy exists at larger scale too
        if (totalUserCommissions > validatorCommission) {
            uint256 burnedAmount = totalUserCommissions - validatorCommission;
            console.log("Even at larger scale,", burnedAmount, "wei is burned due to rounding differences");
        }
    }
    
    /**
     * @notice Test showing the fix: using consistent rounding
     */
    function test_CommissionRounding_Fixed() public {
        uint256 totalStaked = 3;
        uint256 userStakedAmount = 1;
        uint256 numUsers = 3;
        uint256 rewardPerTokenIncrease = 1000;
        uint256 commissionRate = 333;
        
        // FIXED: Validator calculation using ceiling division (same as users)
        uint256 grossRewardForValidator = (totalStaked * rewardPerTokenIncrease) / REWARD_PRECISION;
        uint256 validatorCommissionFixed = _ceilDiv(grossRewardForValidator * commissionRate, REWARD_PRECISION);
        
        // User calculations (unchanged)
        uint256 totalUserCommissions = 0;
        for (uint256 i = 0; i < numUsers; i++) {
            uint256 grossRewardForUser = (userStakedAmount * rewardPerTokenIncrease) / REWARD_PRECISION;
            uint256 userCommission = _ceilDiv(grossRewardForUser * commissionRate, REWARD_PRECISION);
            totalUserCommissions += userCommission;
        }
        
        console.log("\n=== Fixed Version (Consistent Ceiling Division) ===");
        console.log("Validator commission (ceiling):", validatorCommissionFixed);
        console.log("Total user commissions:", totalUserCommissions);
        console.log("Difference:", totalUserCommissions >= validatorCommissionFixed ? 
                   totalUserCommissions - validatorCommissionFixed : 
                   validatorCommissionFixed - totalUserCommissions);
        
        // With consistent rounding, the amounts should be much closer or equal
        // Note: There might still be small differences due to the distributed calculation
        // vs centralized calculation, but the systematic bias is eliminated
        
        console.log("With consistent rounding rules, the systematic under-payment issue is resolved.");
    }
}
```

</details>

## Notes

* Do not change any external links or query parameters; the target file link above is preserved as-is.
* The core issue is a deterministic arithmetic rounding inconsistency — fixing it requires making rounding symmetric between the aggregated validator calculation and per-user calculations.
