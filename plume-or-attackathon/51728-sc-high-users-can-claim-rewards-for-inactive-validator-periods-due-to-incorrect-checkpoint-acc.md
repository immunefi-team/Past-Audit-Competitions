# 51728 sc high users can claim rewards for inactive validator periods due to incorrect checkpoint accrual&#x20;

Submitted on Aug 5th 2025 at 11:02:41 UTC by @farman1094 for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #51728
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/RewardsFacet.sol>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief / Intro

A logic flaw in the reward calculation mechanism allows users to claim staking rewards for periods when their validator was inactive due to improper handling of reward rate checkpoints.

### Vulnerability Details

When a validator’s status is set to inactive, a checkpoint with a zero reward rate is correctly pushed to the checkpoint array to ensure no rewards are calculated during inactivity:

```solidity
// ValidatorFacet::setValidatorStatus
  // Record when the validator became inactive (reuse slashedAtTimestamp field)
                // This allows existing reward logic to cap rewards at this timestamp
                validator.slashedAtTimestamp = block.timestamp;

                // Create a zero-rate checkpoint for all reward tokens to signal inactivity start
                for (uint256 i = 0; i < rewardTokens.length; i++) {
                    PlumeRewardLogic.createRewardRateCheckpoint($, rewardTokens[i], validatorId, 0);
                }
```

However, subsequent calls to functions such as `setRewardRates` or `setMaxRewardRate` push new checkpoints for all validators by calling `createRewardRateCheckpoint`, regardless of whether a validator is inactive or slashed. This causes non-zero reward-rate checkpoints to be added for periods when the validator should not be accruing rewards:

```solidity
// RewardsFacet::setRewardRates
    uint16[] memory validatorIds = $.validatorIds;
        for (uint256 i = 0; i < tokens.length; i++) {
            address token_loop = tokens[i];
            uint256 rate_loop = rewardRates_[i];

            if (!$.isRewardToken[token_loop]) {
                revert TokenDoesNotExist(token_loop);
            }
            uint256 maxRate = $.maxRewardRates[token_loop] > 0 ? $.maxRewardRates[token_loop] : MAX_REWARD_RATE;
            if (rate_loop > maxRate) {
                revert RewardRateExceedsMax();
            }

            for (uint256 j = 0; j < validatorIds.length; j++) {
                uint16 validatorId_for_crrc = validatorIds[j];

                PlumeRewardLogic.createRewardRateCheckpoint($, token_loop, validatorId_for_crrc, rate_loop);
            }
            $.rewardRates[token_loop] = rate_loop;
        }
```

When the validator is reactivated, `validator.slashedAtTimestamp` is cleared (set to 0), removing the cap that should limit reward accrual to the inactivity end time. The reactivation call also pushes a checkpoint restoring the global reward rate:

```solidity
// ValidatorFacet::setValidatorStatus
         if (newActiveStatus && !currentStatus) {
                // Create a new checkpoint to restore the reward rate, signaling activity resumes
                for (uint256 i = 0; i < rewardTokens.length; i++) {
                    address token = rewardTokens[i];
                    $.validatorLastUpdateTimes[validatorId][token] = block.timestamp;
                    uint256 currentGlobalRate = $.rewardRates[token];
                    PlumeRewardLogic.createRewardRateCheckpoint($, token, validatorId, currentGlobalRate);
                }
                // Clear the timestamp since validator is active again (unless actually slashed)
                if (!validator.slashed) {
                    validator.slashedAtTimestamp = 0;
                }
            }
```

As a result, when a user claims rewards after the validator is reactivated, the ending time will be the current `block.timestamp`, and non-zero reward-rate checkpoints exist for the interval when the validator was supposed to be inactive. The reward calculation therefore includes these inactive periods, allowing overpayment.

## Impact Details

{% hint style="warning" %}
This vulnerability allows users to illegitimately claim rewards for durations when their validator was not performing work (inactive). Rewards are misallocated, enabling some users to claim rewards they shouldn’t, which reduces the pool available for others (or leads to overpayment of total rewards).
{% endhint %}

## Suggested Fix

{% hint style="info" %}
Add a check in `PlumeRewardLogic::createRewardRateCheckpoint` (or in the callers before pushing checkpoints) to avoid creating non-zero reward-rate checkpoints for validators that have a non-zero `validator.slashedAtTimestamp` (i.e., validators that are inactive/slashed). This ensures zero-rate checkpoints remain effective for the inactive period and prevents accidental restoration of non-zero rates during inactivity.
{% endhint %}

## Proof of Concept

{% stepper %}
{% step %}

### Step

Assume a user has staked with a Validator. The Validator is initially active and earning rewards.
{% endstep %}

{% step %}

### Step

The protocol calls `setValidatorStatus(validatorId, false)` to set the Validator inactive. This sets `validator.slashedAtTimestamp = block.timestamp` and pushes a zero-rate checkpoint for all tokens. Rewards accrual should stop from this timestamp onward.

```solidity
// ValidatorFacet::setValidatorStatus
  // Record when the validator became inactive (reuse slashedAtTimestamp field)
                // This allows existing reward logic to cap rewards at this timestamp
                validator.slashedAtTimestamp = block.timestamp;

                // Create a zero-rate checkpoint for all reward tokens to signal inactivity start
                for (uint256 i = 0; i < rewardTokens.length; i++) {
                    PlumeRewardLogic.createRewardRateCheckpoint($, rewardTokens[i], validatorId, 0);
                }
```

{% endstep %}

{% step %}

### Step

The protocol later calls `setRewardRates(...)` or `setMaxRewardRate(...)` (which iterate over all validators). These calls invoke `PlumeRewardLogic.createRewardRateCheckpoint`, pushing new non-zero reward-rate checkpoints for every validator, including the inactive one.
{% endstep %}

{% step %}

### Step

The validator’s checkpoint history now contains non-zero reward-rate checkpoints that overlap the period it was supposed to be inactive.
{% endstep %}

{% step %}

### Step

The protocol calls `setValidatorStatus(validatorId, true)` to reactivate the validator. This clears `validator.slashedAtTimestamp` and pushes another checkpoint restoring the current global reward rate.

```solidity
// ValidatorFacet::setValidatorStatus
         if (newActiveStatus && !currentStatus) {
                // Create a new checkpoint to restore the reward rate, signaling activity resumes
                for (uint256 i = 0; i < rewardTokens.length; i++) {
                    address token = rewardTokens[i];
                    $.validatorLastUpdateTimes[validatorId][token] = block.timestamp;
                    uint256 currentGlobalRate = $.rewardRates[token];
                    PlumeRewardLogic.createRewardRateCheckpoint($, token, validatorId, currentGlobalRate);
                }
                // Clear the timestamp since validator is active again (unless actually slashed)
                if (!validator.slashed) {
                    validator.slashedAtTimestamp = 0;
                }
            }
```

{% endstep %}

{% step %}

### Step

A user who staked with that Validator calls `RewardsFacet::claim`. The reward calculation sums rewards across all distinct checkpoint periods from the last claim to the current time. Because non-zero checkpoints exist for the inactive period, the user receives rewards for that period.
{% endstep %}
{% endstepper %}

Result: The user receives rewards that should not have been payable for the inactive period, misallocating the rewards pool.

***
