# 52889 sc high inactive validators accrue rewards for new tokens

**Submitted on Aug 14th 2025 at 01:32:54 UTC by @KlosMitSoss for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52889
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/RewardsFacet.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief/Intro

When a validator is deactivated, the rates for all reward tokens are set to zero. This means that stakers of this validator should not and will not receive any rewards while the validator is inactive. However, when a new reward token is added while a validator is inactive, stakers of that validator will be able to receive rewards from the token addition timestamp once the validator becomes active again.

### Vulnerability Details

The vulnerability stems from the `RewardsFacet::addRewardToken()` function creating positive-rate checkpoints for all validators, including inactive ones, without considering their active status.

When a validator is deactivated via `ValidatorFacet::setValidatorStatus()`, zero-rate checkpoints are correctly created:

```solidity
// Create a zero-rate checkpoint for all reward tokens to signal inactivity start
for (uint256 i = 0; i < rewardTokens.length; i++) {
    PlumeRewardLogic.createRewardRateCheckpoint($, rewardTokens[i], validatorId, 0);
}
```

However, when a new token is added through `addRewardToken()`, it blindly creates positive-rate checkpoints for all validators:

```solidity
// Create a historical record that the rate starts at initialRate for all validators
uint16[] memory validatorIds = $.validatorIds;
for (uint256 i = 0; i < validatorIds.length; i++) {
    uint16 validatorId = validatorIds[i];
    PlumeRewardLogic.createRewardRateCheckpoint($, token, validatorId, initialRate); // Sets rate > 0 for ALL validators
}
```

This creates a checkpoint with `rate = initialRate > 0` for inactive validators, effectively overriding their inactive status for the new token.

The issue manifests during reward calculations in `PlumeRewardLogic::_calculateRewardsCore()`. When a user interacts with the system after validator reactivation, their reward calculation starts from their original stake time (`userValidatorStakeStartTime`), not from when the token was added. The function uses `PlumeRewardLogic::getDistinctTimestamps()` to identify all checkpoint periods and processes each segment:

```solidity
for (uint256 k = 0; k < distinctTimestamps.length - 1; ++k) {
    uint256 segmentStartTime = distinctTimestamps[k];
    uint256 segmentEndTime = distinctTimestamps[k + 1];
    
    // Rate at START of segment
    PlumeStakingStorage.RateCheckpoint memory rewardRateInfoForSegment =
        getEffectiveRewardRateAt($, token, validatorId, segmentStartTime);
    uint256 effectiveRewardRate = rewardRateInfoForSegment.rate;
    
    // Calculate rewards using this rate
    if (effectiveRewardRate > 0 && userStakedAmount > 0) {
        uint256 grossRewardForSegment = 
            (userStakedAmount * rewardPerTokenDeltaForUserInSegment) / PlumeStakingStorage.REWARD_PRECISION;
        // User receives rewards even though validator was inactive
    }
}
```

This enables the following scenario:

{% stepper %}
{% step %}

### Validator deactivation sets zero-rate checkpoints

T1: Validator is deactivated → zero-rate checkpoints created for existing tokens.
{% endstep %}

{% step %}

### New token added while validator inactive

T2: New token added while validator inactive → positive-rate checkpoint created (`rate = initialRate`) for all validators (including inactive ones).
{% endstep %}

{% step %}

### Validator reactivation

T3: Validator reactivated → new positive-rate checkpoint created.
{% endstep %}

{% step %}

### User interaction triggers reward calculation

T4: User interacts (stakes/claims) → reward calculation processes T2-T3 period using `initialRate`, granting rewards for time when validator was inactive.
{% endstep %}
{% endstepper %}

### Impact Details

Due to this issue, users receive unearned rewards equal to `userStakedAmount * initialRate * inactivePeriodDuration`. The protocol loses these reward tokens that should remain in the treasury. Example: when the user has 1 PLUME staked, `initialRate` is 1 token per second and the validator remains inactive for 1 day, they will receive:

```
1e18 * 1 token per second * 86400
```

## References

Code references are provided throughout the report.

## Proof of Concept

{% stepper %}
{% step %}

### Alice stakes with a validator

1. Alice stakes with validator `Id = 1` by calling `StakingFacet::stake()`.\
   (<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/StakingFacet.sol#L257-L269>).
   {% endstep %}

{% step %}

### Validator is deactivated

2. The validator is later deactivated via `ValidatorFacet::setValidatorStatus()`.\
   (<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L271-L286).\\>
   This sets `slashedAtTimestamp` to `block.timestamp` to cap rewards and creates zero-rate checkpoints for all existing reward tokens.
   {% endstep %}

{% step %}

### New reward token added while inactive

3. While the validator remains inactive, `RewardsFacet::addRewardToken()` is called to add a new reward token. The function loops through all `validatorIds`, including inactive ones, and creates reward rate checkpoints with `initialRate` for every validator (<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/RewardsFacet.sol#L193-L196>). This calls `PlumeRewardLogic::updateRewardPerTokenForValidator()`, which only updates `validatorLastUpdateTimes[validatorId][token]` for inactive validators (<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L155-L160>).
   {% endstep %}

{% step %}

### Validator reactivated

4. The validator is reactivated via `ValidatorFacet::setValidatorStatus()`, creating new reward rate checkpoints with the current global rate. (<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L292-L304>).
   {% endstep %}

{% step %}

### Alice claims rewards — calculation includes inactive period

5. Alice claims rewards by calling `RewardsFacet::claim(address token, uint16 validatorId)`, which processes rewards through `RewardsFacet::_processValidatorRewards()` (<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/RewardsFacet.sol#L310-L331>).
6. This calls `PlumeRewardLogic::updateRewardsForValidatorAndToken()` (<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/RewardsFacet.sol#L468>), which invokes `calculateRewardsWithCheckpoints()` (<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L112-L113>). This function calls and then `_calculateRewardsCore()`.
7. `PlumeRewardLogic::updateRewardPerTokenForValidator()` updates the `validatorLastUpdateTimes[validatorId][token]` and `validatorRewardPerTokenCumulative[validatorId][token]` (<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L163-L196>).
8. `PlumeRewardLogic::_calculateRewardsCore()` calculates the rewards from the `userValidatorRewardPerTokenPaidTimestamp[user][validatorId][token]` up to the current `block.timestamp` if the `userValidatorRewardPerTokenPaid[user][validatorId][token]` differs from the `validatorRewardPerTokenCumulative[validatorId][token]` (which it does due to the `updateRewardPerTokenForValidator()` call). In this case, the `userValidatorRewardPerTokenPaidTimestamp[user][validatorId][token]` will be equal to the `userValidatorStakeStartTime` (<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L225-L231>).
9. The function uses `getDistinctTimestamps()` to process reward segments. The segment before token addition has `rate = 0` (correct), but the segment from token addition until validator reactivation uses `initialRate` (incorrect). (<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L294-L358>). As a result, Alice will receive rewards while the validator was inactive.
   {% endstep %}
   {% endstepper %}
