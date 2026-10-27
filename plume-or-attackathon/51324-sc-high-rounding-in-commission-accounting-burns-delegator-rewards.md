# 51324 sc high rounding in commission accounting burns delegator rewards

**Submitted on Aug 1st 2025 at 17:33:58 UTC by @Rhaydden for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51324
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol>
* **Impacts:** Theft of unclaimed yield

## Description

### Brief/Intro

The staking reward logic credits validator commission using floor division but debits each delegator using ceiling division. When ≥2 stakers exist, the sum of per-user charges always exceeds the amount credited to the validator, and the excess “dust” is lost forever. Over time this silently steals part of the users’ unclaimed yield and breaks the protocol's accounting.

In the validator path, `updateRewardPerTokenForValidator` floors the aggregate commission so the validator is never over credited:

```solidity
commissionDeltaForValidator =
    (grossRewardForValidatorThisSegment * commissionRate)
/* FLOOR */    / REWARD_PRECISION;
```

In the user path, `_calculateRewardsCore` ceilings each delegator’s commission so the validator is never underpaid:

```solidity
commissionForThisSegment =
    _ceilDiv(grossRewardForSegment * commissionRate,
/* CEIL */   REWARD_PRECISION);
```

These two choices together break conservation of tokens when more than one staker exists:

ceil(user₁) + ceil(user₂) + … + ceil(userₙ) ≥ floor(user₁+user₂+…+userₙ)

The excess created by the per-user ceilings is not recorded anywhere. Comments in the code hint at rounding intent, but the contract never reconciles the difference.

### Vulnerability Details

In PlumeRewardLogic.sol:

```solidity
// Validator accrual – floor (line ~185)
uint256 commissionDeltaForValidator =
        (grossRewardForValidatorThisSegment * commissionRate)
/* FLOOR */ / REWARD_PRECISION;

// Delegator charge – ceil (line ~348)
uint256 commissionForThisSegment =
        _ceilDiv(grossRewardForSegment * effectiveCommissionRate,
/* CEIL */   REWARD_PRECISION);
```

Mathematically:

Σ ceil(user\_i × r / P) ≥ floor(Σ user\_i × r / P)

A strict inequality occurs whenever fractions exist, producing up to (n − 1) wei of “dust” per segment (n = #stakers). This dust is:

* Removed from delegators’ gross rewards;
* Not added to `validatorAccruedCommission`;
* Not reflected in `totalClaimableByToken`;

Because reward distribution runs every epoch for every validator, the loss compounds and eventually makes:

contract.balance > totalClaimableByToken + Σ validatorAccruedCommission

breaking conservation of tokens.

## Impact Details

High (Theft of unclaimed yield)

* Delegators receive less reward than they earned.
* Validator cannot withdraw the difference, so funds accumulate as unclaimable dust.
* Over many epochs with thousands of stakers, the loss can become material, effectively burning user yield.

## Fix

* Simpler fix: use floor everywhere (i.e., floor per-user commission as well).
* Alternative: keep per-user ceil but credit the rounding dust to the validator (i.e., reconcile per-segment rounding delta and add it to `validatorAccruedCommission` or `totalClaimableByToken`).

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L185-L187>

## Proof of Concept

<details>

<summary>Solidity test demonstrating the rounding mismatch (expand to view)</summary>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import "forge-std/Test.sol";
import "forge-std/console.sol";
import "../src/lib/PlumeRewardLogic.sol";
import "../src/lib/PlumeStakingStorage.sol";

/**
 * @title CommissionRoundingBugTest
 * @notice Test to demonstrate the rounding mismatch between validator commission (floor division)
 *         and user commission charges (ceiling division)
 */
contract CommissionRoundingBugTest is Test {
    // Constants and storage
    uint256 constant REWARD_PRECISION = PlumeStakingStorage.REWARD_PRECISION; 
    uint256 constant COMMISSION_RATE = REWARD_PRECISION / 10; // 10% commission

    // Simulated users and validators
    address validatorAdmin = address(0x1);
    address alice = address(0x2);
    address bob = address(0x3);
    address carol = address(0x4);
    uint16 validatorId = 1;
    address rewardToken = address(0x10);

    // Track balances for verification
    uint256 totalGrossRewards;
    uint256 totalValidatorCommission;
    uint256 totalUserNetRewards;
    uint256 expectedLostDust;

    function setUp() public {
        // Initial setup - no specific setup needed for this test
    }

    // Helper to simulate validator commission calculation using floor division
    function calculateValidatorCommission(uint256 grossAmount) public view returns (uint256) {
        // Simulates how updateRewardPerTokenForValidator calculates commission
        return (grossAmount * COMMISSION_RATE) / REWARD_PRECISION;
    }

    // Helper to simulate user commission calculation using ceiling division
    function calculateUserCommission(uint256 grossAmount) public view returns (uint256) {
        // Simulates how _calculateRewardsCore calculates commission
        if (grossAmount == 0) return 0;
        return (grossAmount * COMMISSION_RATE + REWARD_PRECISION - 1) / REWARD_PRECISION;
    }

    function testCommissionRoundingMismatch() public {
        // SCENARIO: We'll create rewards for 3 users where each reward amount
        // produces a fractional commission amount that gets rounded differently

        // A deliberately chosen value to demonstrate the rounding issue
        uint256 baseAmount = REWARD_PRECISION / 9; // This produces fractional commission

        // User rewards (each slightly different to show consistent loss pattern)
        uint256 aliceGross = baseAmount;
        uint256 bobGross = baseAmount + 1;
        uint256 carolGross = baseAmount - 1;

        totalGrossRewards = aliceGross + bobGross + carolGross;
        console.log("Total gross rewards:", totalGrossRewards);

        // PART 1: Calculate validator commission using floor division (as in the contract)
        uint256 aggregateValidatorCommission = calculateValidatorCommission(totalGrossRewards);
        totalValidatorCommission = aggregateValidatorCommission;
        console.log("Validator commission (floor):", aggregateValidatorCommission);

        // PART 2: Calculate per-user commission using ceiling division (as in the contract)
        uint256 aliceCommission = calculateUserCommission(aliceGross);
        uint256 bobCommission = calculateUserCommission(bobGross);
        uint256 carolCommission = calculateUserCommission(carolGross);
        
        uint256 totalUserCommissionCharges = aliceCommission + bobCommission + carolCommission;
        console.log("User 1 commission (ceil):", aliceCommission);
        console.log("User 2 commission (ceil):", bobCommission);
        console.log("User 3 commission (ceil):", carolCommission);
        console.log("Total user commission charges:", totalUserCommissionCharges);

        // PART 3: Calculate net rewards for users
        uint256 aliceNet = aliceGross - aliceCommission;
        uint256 bobNet = bobGross - bobCommission;
        uint256 carolNet = carolGross - carolCommission;
        totalUserNetRewards = aliceNet + bobNet + carolNet;
        console.log("Total net user rewards:", totalUserNetRewards);

        // PART 4: Calculate accounting discrepancy
        expectedLostDust = totalUserCommissionCharges - aggregateValidatorCommission;
        console.log("Expected 'dust' loss:", expectedLostDust);

        // PART 5: Verify the accounting issue exists
        uint256 actualTotalTokens = totalValidatorCommission + totalUserNetRewards;
        console.log("Tokens accounted for:", actualTotalTokens);
        console.log("Original total tokens:", totalGrossRewards);
        
        // The key assertion: there's a discrepancy due to the rounding mismatch
        assertTrue(
            actualTotalTokens < totalGrossRewards, 
            "Accounting error: tokens have been lost"
        );
        
        // Verify the exact amount of dust loss
        assertEq(
            totalGrossRewards - actualTotalTokens,
            expectedLostDust, 
            "Lost dust should match the expected amount"
        );
        
        // Demonstrate how many users can result in even more dust loss
        console.log("");
        console.log("DUST LOSS SCALING DEMONSTRATION");
        
        uint256[] memory userCounts = new uint256[](3);
        userCounts[0] = 10;
        userCounts[1] = 100;
        userCounts[2] = 1000;
        
        for (uint i = 0; i < userCounts.length; i++) {
            uint256 userCount = userCounts[i];
            uint256 singleUserGross = baseAmount;
            uint256 totalGross = singleUserGross * userCount;
            
            uint256 validatorCommTotal = calculateValidatorCommission(totalGross);
            uint256 userCommTotal = calculateUserCommission(singleUserGross) * userCount;
            uint256 dustLoss = userCommTotal - validatorCommTotal;
            
            console.log("With", userCount, "users: dust loss =", dustLoss);
        }
        
        // PART 6: Demonstrate one possible fix (using floor division everywhere)
        console.log("");
        console.log("SOLUTION DEMONSTRATION: Floor division everywhere");
        
        uint256 fixedAliceCommission = (aliceGross * COMMISSION_RATE) / REWARD_PRECISION; // floor
        uint256 fixedBobCommission = (bobGross * COMMISSION_RATE) / REWARD_PRECISION; // floor
        uint256 fixedCarolCommission = (carolGross * COMMISSION_RATE) / REWARD_PRECISION; // floor
        
        uint256 fixedTotalUserCommission = fixedAliceCommission + fixedBobCommission + fixedCarolCommission;
        uint256 fixedUserNetRewards = aliceGross - fixedAliceCommission + bobGross - fixedBobCommission + carolGross - fixedCarolCommission;
        
        console.log("Fixed validator commission:", aggregateValidatorCommission);
        console.log("Fixed total user commission charges:", fixedTotalUserCommission);
        console.log("Fixed net user rewards:", fixedUserNetRewards);
        console.log("Total tokens accounted for:", aggregateValidatorCommission + fixedUserNetRewards);
        console.log("Original total tokens:", totalGrossRewards);
        
        // Verify the fix restores proper accounting
        assertEq(
            aggregateValidatorCommission + fixedUserNetRewards,
            totalGrossRewards,
            "Fixed solution should preserve all tokens"
        );
    }
}
```

</details>

## Logs

<details>

<summary>Execution logs (expand to view)</summary>

```
Ran 1 test for test/CommissionRoundingBug.t.sol:CommissionRoundingBugTest
[PASS] testCommissionRoundingMismatch() (gas: 110332)
Logs:
  Total gross rewards: 333333333333333333
  Validator commission (floor): 33333333333333333
  User 1 commission (ceil): 11111111111111112
  User 2 commission (ceil): 11111111111111112
  User 3 commission (ceil): 11111111111111111
  Total user commission charges: 33333333333333335
  Total net user rewards: 299999999999999998
  Expected 'dust' loss: 2
  Tokens accounted for: 333333333333333331
  Original total tokens: 333333333333333333
  
  DUST LOSS SCALING DEMONSTRATION
  With 10 users: dust loss = 9
  With 100 users: dust loss = 90
  With 1000 users: dust loss = 900
  
  SOLUTION DEMONSTRATION: Floor division everywhere
  Fixed validator commission: 33333333333333333
  Fixed total user commission charges: 33333333333333333
  Fixed net user rewards: 300000000000000000
  Total tokens accounted for: 333333333333333333
  Original total tokens: 333333333333333333

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 8.22ms (3.46ms CPU time)
```

</details>
