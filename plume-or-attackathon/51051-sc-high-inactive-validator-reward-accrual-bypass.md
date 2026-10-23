# 51051 sc high inactive validator reward accrual bypass

**Submitted on Jul 30th 2025 at 18:23:58 UTC by @light279 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51051
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/RewardsFacet.sol>
* **Impacts:**
  * Theft of unclaimed yield
  * Protocol insolvency

## Description

### Brief/Intro

Users are able to bypass reward accrual restrictions for inactive validators due to how the `RewardsFacet::setRewardRates` function updates reward rate checkpoints even for inactive validators. This unintended behavior allows users to claim inflated rewards for the duration the validator was inactive, violating the protocol's design intent that no rewards should accrue during such periods.

```javascript
function setRewardRates(
        address[] calldata tokens,
        uint256[] calldata rewardRates_
    ) external onlyRole(PlumeRoles.REWARD_MANAGER_ROLE) {
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();

        if (tokens.length == 0) {
            revert EmptyArray();
        }
        if (tokens.length != rewardRates_.length) {
            revert ArrayLengthMismatch();
        }
        uint16[] memory validatorIds = $.validatorIds;
        for (uint256 i = 0; i < tokens.length; i++) {
            address token_loop = tokens[i];
            uint256 rate_loop = rewardRates_[i];

            if (!$.isRewardToken[token_loop]) {
                revert TokenDoesNotExist(token_loop);
            }
            uint256 maxRate = $.maxRewardRates[token_loop] > 0
                ? $.maxRewardRates[token_loop]
                : MAX_REWARD_RATE;
            if (rate_loop > maxRate) {
                revert RewardRateExceedsMax();
            }

            for (uint256 j = 0; j < validatorIds.length; j++) {
                uint16 validatorId_for_crrc = validatorIds[j];

@>                PlumeRewardLogic.createRewardRateCheckpoint(
                    $,
                    token_loop,
                    validatorId_for_crrc,
                    rate_loop
                );
            }
            $.rewardRates[token_loop] = rate_loop;
        }
        emit RewardRatesSet(tokens, rewardRates_);
    }
```

Note: One might argue that rewards are capped at `validator.slashedAtTimestamp`, which is set when the validator is marked as inactive. However, this does not prevent the user from delaying their claim until after the validator is reactivated or enough time has passed. Additionally, due to the reward rate checkpoint updates triggered by `setRewardRates`, users are still able to accumulate and claim rewards during the inactive period, making the cap ineffective in practice if the user is claiming after validator is made active.

## Vulnerability Details

The protocol allows validators to be marked active/inactive via the `ValidatorFacet::setValidatorStatus` function. When a validator is marked inactive:

* The validator's `slashedAtTimestamp` is set to `block.timestamp`.
* A zero-rate checkpoint is created for all reward tokens, signaling a halt in reward accrual.
* On reactivation, a new reward checkpoint is created with the current global rate, and `slashedAtTimestamp` is cleared.

This works as expected when users interact immediately after the validator goes inactive or active.

{% stepper %}
{% step %}

### Sequence that produces the vulnerability

* A validator is marked inactive, correctly setting a 0 reward rate and `slashedAtTimestamp`.
* A global reward rate is changed via `RewardsFacet::setRewardRates`.
* `setRewardRates` blindly creates new checkpoints for all validators, including inactive ones.
* The reward rate checkpoint history for the inactive validator now includes non-zero rate timestamps that fall within its inactive period.
* If a user does not claim during the inactive period and instead waits until the validator is reactivated, these new checkpoints get used in reward calculation, leading to reward accrual during inactive periods.
  {% endstep %}
  {% endstepper %}

Below is the functional flow after a validator is made active when a user calls claim:

`RewardsFacet::claim(address token,uint16 validatorId)` => `RewardsFacet::_processValidatorRewards` => `PlumeRewardLogic.updateRewardsForValidatorAndToken` => `PlumeRewardLogic::calculateRewardsWithCheckpoints` => `PlumeRewardLogic::_calculateRewardsCore`

In `PlumeRewardLogic::_calculateRewardsCore`: the effective end time is set as `effectiveEndTime = block.timestamp` and it is not updated when the validator was previously inactive (because `slashedAtTimestamp` may have been cleared upon reactivation). As a result, all timestamps between the start and `effectiveEndTime` are used, and any `RateCheckpoint` entries created during the inactive period (by `setRewardRates`) are counted towards rewards.

```javascript
function _calculateRewardsCore(
        PlumeStakingStorage.Layout storage $,
        address user,
        uint16 validatorId,
        address token,
        uint256 userStakedAmount,
        uint256 currentCumulativeRewardPerToken
    )
        internal
        view
        returns (uint256 totalUserRewardDelta, uint256 totalCommissionAmountDelta, uint256 effectiveTimeDelta)
    {
        uint256 lastUserPaidCumulativeRewardPerToken = $.userValidatorRewardPerTokenPaid[user][validatorId][token];
        uint256 lastUserRewardUpdateTime = $.userValidatorRewardPerTokenPaidTimestamp[user][validatorId][token];

        if (lastUserRewardUpdateTime == 0) {
            // This is the first reward calculation for this user/validator/token.
            // Start the calculation from when the user's stake began.
            // The checkpoint system will correctly apply a zero rate for any period before the token was added.
            lastUserRewardUpdateTime = $.userValidatorStakeStartTime[user][validatorId];

            if (lastUserRewardUpdateTime == 0 && $.userValidatorStakes[user][validatorId].staked > 0) {
                
                uint256 fallbackTime = block.timestamp;

                // If validator is slashed, cap fallback time at slash timestamp
                PlumeStakingStorage.ValidatorInfo storage validator = $.validators[validatorId];
                if (validator.slashedAtTimestamp > 0 && validator.slashedAtTimestamp < fallbackTime) {
                    fallbackTime = validator.slashedAtTimestamp;
                }

                lastUserRewardUpdateTime = fallbackTime;
            }
        }

        // For recently reactivated validators, don't calculate rewards
        // from before the reactivation time to prevent retroactive accrual
        uint256 validatorLastUpdateTime = $.validatorLastUpdateTimes[validatorId][token];

        // For slashed/inactive validators, cap the calculation period at the timestamp
        PlumeStakingStorage.ValidatorInfo storage validator = $.validators[validatorId];
@>        uint256 effectiveEndTime = block.timestamp;

        // Check token removal timestamp
        uint256 tokenRemovalTime = $.tokenRemovalTimestamps[token];
        if (tokenRemovalTime > 0 && tokenRemovalTime < effectiveEndTime) {
            effectiveEndTime = tokenRemovalTime;
        }

        // Then check validator slash/inactive timestamp
@>        if (validator.slashedAtTimestamp > 0) {
            if (validator.slashedAtTimestamp < effectiveEndTime) {
                effectiveEndTime = validator.slashedAtTimestamp;
            }
        }

        // If no time has passed or user hasn't earned anything yet (e.g. paid index is already current)
        if (
            effectiveEndTime <= lastUserRewardUpdateTime
                || currentCumulativeRewardPerToken <= lastUserPaidCumulativeRewardPerToken
        ) {
            return (0, 0, 0);
        }

        effectiveTimeDelta = effectiveEndTime - lastUserRewardUpdateTime; // This is the total duration of interest

        uint256[] memory distinctTimestamps =
            getDistinctTimestamps($, validatorId, token, lastUserRewardUpdateTime, effectiveEndTime);

...........................
```

## Impact Details

* Inflated rewards: Users can earn rewards for periods where their validator was supposed to be inactive and reward accrual should have been paused.
* Protocol inconsistency: This violates the designed reward logic and creates unfair advantage for users staking with previously inactive validators.

## Proof of Concept

{% stepper %}
{% step %}
Validator A is marked inactive at t1 — sets `slashedAtTimestamp = t1`.
{% endstep %}

{% step %}
At t2 > t1, call is made to `RewardsFacet::setRewardRates` for a particular reward token — this sets new rate checkpoints for all validators including A.
{% endstep %}

{% step %}
At t3 > t2, Validator A is reactivated — clears `slashedAtTimestamp` and sets new checkpoint.
{% endstep %}

{% step %}
At t4 > t3, user calls `claim()`:

* `_calculateRewardsCore()` fetches checkpoints from t1 to t4.
* Includes the non-zero checkpoint at t2, even though A was inactive.
* User receives inflated rewards for \[t2, t3], violating protocol intent.
  {% endstep %}
  {% endstepper %}
