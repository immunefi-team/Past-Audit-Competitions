# 52996 sc high users can claim rewards for newly added reward tokens even when the validator they staked for was inactive during some time interval&#x20;

**Submitted on Aug 14th 2025 at 15:41:04 UTC by @swarun for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #52996
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* Impacts: Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief / Intro

Suppose a validator was inactive for some time interval and during that time interval a new token got added. Users shouldn't receive rewards when the validator is inactive, but they will receive rewards when a new reward token was added during the inactive interval. This causes users to receive rewards for time intervals where there was no accumulation of rewards, which should not happen.

### Vulnerability Details

When a new reward token is added while a validator is inactive, the logic creates reward checkpoints for all validators (including the inactive one) at the addition time. Later, when the validator is reactivated, a new checkpoint is created again at the activation time. Because the reward calculation logic splits time into segments based on checkpoints, this can result in a segment that covers the inactive interval with a non-zero effective reward rate (from the token addition checkpoint), producing rewards to stakers even though the validator was inactive during that time.

### Impact Details

Users receive more rewards than they are eligible for.

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L252>

## Proof of Concept

{% stepper %}
{% step %}

### Setup: validator made inactive while users are staked

Suppose the admin sets a validator inactive (without slashing) while users remain staked. The function used:

```solidity
function setValidatorStatus(
    uint16 validatorId,
    bool newActiveStatus
) external onlyRole(PlumeRoles.ADMIN_ROLE) _validateValidatorExists(validatorId) {
    PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
    PlumeStakingStorage.ValidatorInfo storage validator = $.validators[validatorId];

    // Prevent activating an already slashed validator through this function
    if (newActiveStatus && validator.slashed) {
        revert ValidatorAlreadySlashed(validatorId);
    }

    bool currentStatus = validator.active;

    // If status is actually changing
    if (currentStatus != newActiveStatus) {
        address[] memory rewardTokens = $.rewardTokens;

        // If going INACTIVE: settle validator commission and record timestamp
        if (!newActiveStatus && currentStatus) {
            // Settle commission for validator using current rates
            PlumeRewardLogic._settleCommissionForValidatorUpToNow($, validatorId);

            // Record when the validator became inactive (reuse slashedAtTimestamp field)
            // This allows existing reward logic to cap rewards at this timestamp
            validator.slashedAtTimestamp = block.timestamp;

            // Create a zero-rate checkpoint for all reward tokens to signal inactivity start
            for (uint256 i = 0; i < rewardTokens.length; i++) {
                PlumeRewardLogic.createRewardRateCheckpoint($, rewardTokens[i], validatorId, 0);
            }

            // NOTE: User rewards will be settled naturally when users interact
            // (stake, unstake, claim, etc.) due to the timestamp cap in reward logic
        }

        // Update the status
        validator.active = newActiveStatus;

        // If going ACTIVE: reset timestamps and clear the timestamp cap
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
    }

    emit ValidatorStatusUpdated(validatorId, newActiveStatus, validator.slashed);
}
```

* Validator becomes inactive at t = 0 with users still staked.
* Validator remains inactive until t = 2 days.
  {% endstep %}

{% step %}

### Add a new reward token during inactivity

At t = 1 day (while the validator is inactive), a new reward token is added via:

```solidity
function addRewardToken(
    address token,
    uint256 initialRate,
    uint256 maxRate
) external onlyRole(PlumeRoles.REWARD_MANAGER_ROLE) {
    PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
    if (token == address(0)) {
        revert ZeroAddress("token");
    }
    if ($.isRewardToken[token]) {
        revert TokenAlreadyExists();
    }
    if (initialRate > maxRate) {
        revert RewardRateExceedsMax();
    }

    // Prevent re-adding a token in the same block it was removed to avoid checkpoint overwrites.
    if ($.tokenRemovalTimestamps[token] == block.timestamp) {
        revert CannotReAddTokenInSameBlock(token);
    }

    // Add to historical record if it's the first time seeing this token.
    if (!$.isHistoricalRewardToken[token]) {
        $.isHistoricalRewardToken[token] = true;
        $.historicalRewardTokens.push(token);
    }

    uint256 additionTimestamp = block.timestamp;

    // Clear any previous removal timestamp to allow re-adding
    $.tokenRemovalTimestamps[token] = 0;

    $.rewardTokens.push(token);
    $.isRewardToken[token] = true;
    $.maxRewardRates[token] = maxRate;
    $.rewardRates[token] = initialRate; // Set initial global rate
    $.tokenAdditionTimestamps[token] = additionTimestamp;

    // Create a historical record that the rate starts at initialRate for all validators
    uint16[] memory validatorIds = $.validatorIds;
    for (uint256 i = 0; i < validatorIds.length; i++) {
        uint16 validatorId = validatorIds[i];
        PlumeRewardLogic.createRewardRateCheckpoint($, token, validatorId, initialRate);
    }

    emit RewardTokenAdded(token);
    if (maxRate > 0) {
        emit MaxRewardRateUpdated(token, maxRate);
    }
}
```

* This creates a reward-rate checkpoint for every validator (including the inactive one) at t = 1 day with a non-zero rate.
  {% endstep %}

{% step %}

### Validator reactivated and another checkpoint created

At t = 2 days the admin reactivates the validator. The active transition creates another checkpoint for all reward tokens at t = 2 days and resets timestamp caps:

```solidity
// If going ACTIVE: reset timestamps and clear the timestamp cap
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

* Now the newly added token has two checkpoints for this validator: one at t = 1 day (added) and one at t = 2 days (reactivation).
  {% endstep %}

{% step %}

### Reward accumulation logic and how it miscomputes for the inactive interval

Important notes:

* While the validator was inactive, the system prevents updating $.validatorRewardPerTokenCumulative\[validatorId]\[token] due to timestamp caps (no reward accumulation while inactive).
* Later, after reactivation and time passing, the cumulative RPT is updated, and a user triggers a claim.

Claim flow (relevant parts):

```solidity
function claim(address token, uint16 validatorId) external nonReentrant returns (uint256) {
    _validateTokenForClaim(token, msg.sender);
    _validateValidatorForClaim(validatorId);

    uint256 reward = _processValidatorRewards(msg.sender, validatorId, token);

    if (reward > 0) {
        _finalizeRewardClaim(token, reward, msg.sender);
    }

    PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
    PlumeRewardLogic.clearPendingRewardsFlagIfEmpty($, msg.sender, validatorId);
    PlumeValidatorLogic.removeStakerFromValidator($, msg.sender, validatorId);

    return reward;
}
```

Main reward processing:

```solidity
function _processValidatorRewards(
    address user,
    uint16 validatorId,
    address token
) internal returns (uint256 reward) {
    PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();

    // Settle pending rewards for this specific user/validator/token combination.
    // This updates both $.userRewards and $.totalClaimableByToken consistently.
    PlumeRewardLogic.updateRewardsForValidatorAndToken($, user, validatorId, token);

    // Now that rewards are settled, the full claimable amount is in storage.
    reward = $.userRewards[user][validatorId][token];

    if (reward > 0) {
        _updateUserRewardState(user, validatorId, token);
        emit RewardClaimedFromValidator(user, token, validatorId, reward);
    }
}
```

* updateRewardsForValidatorAndToken ultimately calls core calculation logic which splits the period since the user's last update into segments based on distinct checkpoint timestamps (rate or commission changes).
  {% endstep %}

{% step %}

### How the segments cause incorrect rewards

The core loop computing rewards iterates segments between distinct timestamps (checkpoints). Simplified snippet:

```solidity
for (uint256 k = 0; k < distinctTimestamps.length - 1; ++k) {
    uint256 segmentStartTime = distinctTimestamps[k];
    uint256 segmentEndTime = distinctTimestamps[k + 1];

    if (segmentEndTime <= segmentStartTime) {
        continue;
    }

    uint256 rptAtSegmentStart;
    if (k == 0) {
        rptAtSegmentStart = lastUserPaidCumulativeRewardPerToken;
    } else {
        rptAtSegmentStart = rptTracker;
    }

    PlumeStakingStorage.RateCheckpoint memory rewardRateInfoForSegment =
        getEffectiveRewardRateAt($, token, validatorId, segmentStartTime); // Rate at START of segment
    uint256 effectiveRewardRate = rewardRateInfoForSegment.rate;
    uint256 segmentDuration = segmentEndTime - segmentStartTime;

    uint256 rptIncreaseInSegment = 0;
    if (effectiveRewardRate > 0 && segmentDuration > 0) {
        rptIncreaseInSegment = segmentDuration * effectiveRewardRate;
    }

    uint256 rptAtSegmentEnd = rptAtSegmentStart + rptIncreaseInSegment;
    uint256 rewardPerTokenDeltaForUserInSegment = rptAtSegmentEnd - rptAtSegmentStart;

    if (rewardPerTokenDeltaForUserInSegment > 0 && userStakedAmount > 0) {
        uint256 grossRewardForSegment =
            (userStakedAmount * rewardPerTokenDeltaForUserInSegment) / PlumeStakingStorage.REWARD_PRECISION;

        uint256 effectiveCommissionRate = getEffectiveCommissionRateAt($, validatorId, segmentStartTime);
        uint256 commissionForThisSegment =
            _ceilDiv(grossRewardForSegment * effectiveCommissionRate, PlumeStakingStorage.REWARD_PRECISION);

        if (grossRewardForSegment >= commissionForThisSegment) {
            totalUserRewardDelta += (grossRewardForSegment - commissionForThisSegment);
        }

        totalCommissionAmountDelta += commissionForThisSegment;
    }
    rptTracker = rptAtSegmentEnd;
}
```

* In the scenario: distinctTimestamps includes t = 1 day (token addition) and t = 2 days (validator reactivated).
* For the segment \[1 day, 2 day], getEffectiveRewardRateAt at segmentStartTime (1 day) returns a non-zero rate (token was added).
* segmentDuration = 1 day, userStakedAmount > 0, so gross reward is calculated and added to the user's rewards even though the validator was inactive during that segment.
* This yields extraneous rewards for the inactive interval.
  {% endstep %}
  {% endstepper %}
