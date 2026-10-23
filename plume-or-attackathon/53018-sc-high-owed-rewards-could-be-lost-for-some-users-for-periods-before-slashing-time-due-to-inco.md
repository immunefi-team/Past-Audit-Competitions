# 53018 sc high owed rewards could be lost for some users for periods before slashing time due to incorrect logic&#x20;

Submitted on Aug 14th 2025 at 17:07:58 UTC by @valkvalue for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #53018
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>

Impacts:

* Theft of unclaimed yield
* Permanent freezing of funds

## Description

Brief/Intro\
Rewards could be lost for some users, for owed periods, before slashing time, due to wrong logic.

## Vulnerability Details

Currently `_performSlash` doesn't call `updateRewardPerTokenForValidator` for each token <-> slashed validator pair to ensure the reward cumulative value is properly up to date before the slash.

There is logic in `calculateRewardsWithCheckpoints` intended to handle this case and to accrue and properly increase the global `validatorRewardPerTokenCumulative` prior to calling the core reward calculation. However, that logic assumes `validatorLastUpdateTimes` are correct. An attacker (or some legitimate flow) can cause `validatorLastUpdateTimes` to be updated after the slashing time by calling `updateRewardPerTokenForValidator` in other flows, which sets `validatorLastUpdateTimes` to `block.timestamp` even for slashed validators. The code even contains a comment acknowledging the issue and intentionally avoids calling `updateRewardPerTokenForValidator` in the slashed branch:

```solidity
// Slashed validator case: calculate what cumulative should be up to the slash timestamp.
// We DO NOT call updateRewardPerTokenForValidator here because its logic is incorrect for slashed validators.
```

`updateRewardPerTokenForValidator` sets `validatorLastUpdateTimes` to `block.timestamp` for slashed validators:

Reference: <https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/plume/src/lib/PlumeRewardLogic.sol#L144-L154>

```solidity
function updateRewardPerTokenForValidator(...) internal {
    PlumeStakingStorage.ValidatorInfo storage validator = $.validators[validatorId];
    ...
    if (validator.slashed) {
        // For slashed validators, no further rewards or commission accrue.
        // We just update the timestamp to the current time to mark that the state is "settled" up to now.
        $.validatorLastUpdateTimes[validatorId][token] = block.timestamp;
        if ($.validatorTotalStaked[validatorId] > 0) {
            revert InternalInconsistency("Slashed validator has non-zero totalStaked");
        }
        return;
    }
    ...
}
```

Flows that can trigger this include `RewardsFacet.removeRewardToken` (which settles for all validators, including slashed ones) and `ValidatorFacet.forceSettleValidatorCommission`.

The problematic flow in `calculateRewardsWithCheckpoints` is:

```solidity
function calculateRewardsWithCheckpoints( ...) internal returns {
    if (!validator.slashed) {
        ...
    } else {
        // Slashed validator case: calculate what cumulative should be up to the slash timestamp.
        // We DO NOT call updateRewardPerTokenForValidator here because its logic is incorrect for slashed validators.
        uint256 currentCumulativeRewardPerToken = $.validatorRewardPerTokenCumulative[validatorId][token];
        uint256 effectiveEndTime = validator.slashedAtTimestamp;

        uint256 tokenRemovalTime = $.tokenRemovalTimestamps[token];
        if (tokenRemovalTime > 0 && tokenRemovalTime < effectiveEndTime) {
            effectiveEndTime = tokenRemovalTime;
        }

        uint256 validatorLastUpdateTime = $.validatorLastUpdateTimes[validatorId][token];
        if (effectiveEndTime > validatorLastUpdateTime) {
            uint256 timeSinceLastUpdate = effectiveEndTime - validatorLastUpdateTime;
            if (userStakedAmount > 0) {
                PlumeStakingStorage.RateCheckpoint memory effectiveRewardRateChk = getEffectiveRewardRateAt($, token, validatorId, validatorLastUpdateTime); // Use rate at start of segment
                uint256 effectiveRewardRate = effectiveRewardRateChk.rate;

                if (effectiveRewardRate > 0) {
                    uint256 rewardPerTokenIncrease = timeSinceLastUpdate * effectiveRewardRate;
                    currentCumulativeRewardPerToken += rewardPerTokenIncrease;
                }
            }
        }

        return _calculateRewardsCore($, user, validatorId, token, userStakedAmount, currentCumulativeRewardPerToken);
    }
}
```

Because `updateRewardPerTokenForValidator` can set `validatorLastUpdateTimes[validatorId][token]` to a timestamp after the slashing time, `effectiveEndTime > validatorLastUpdateTime` will be false and the cumulative reward per token will not be increased for the period between the previous last update and the slash. As a result, user rewards covering that period are not accounted for.

When entering `_calculateRewardsCore()`, the following check may return `(0, 0, 0)`:

```solidity
if (
    effectiveEndTime <= lastUserRewardUpdateTime
        || currentCumulativeRewardPerToken <= lastUserPaidCumulativeRewardPerToken
) {
    return (0, 0, 0);
}
```

If a user's `userValidatorRewardPerTokenPaid` was last updated exactly at the `block.timestamp` where the validator was last (incorrectly) updated, then `currentCumulativeRewardPerToken` may be equal to that paid value and thus the function returns zero rewards, even though the user should be owed rewards for the period prior to the slash.

Normal flows like `stake()` and `unstake()` update both the token<->validator pair data and the token<->validator<->user data (including `userValidatorRewardPerTokenPaid` and `validatorRewardPerTokenCumulative`), so it's plausible for a user to have `userValidatorRewardPerTokenPaid` equal to the (incorrectly) stale `validatorRewardPerTokenCumulative`, causing permanent loss of owed rewards.

Example code reference: <https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/plume/src/lib/PlumeRewardLogic.sol#L122-L124>

```solidity
// Update paid pointers AFTER calculating delta to correctly checkpoint the user's state.
$.userValidatorRewardPerTokenPaid[user][validatorId][token] =
    $.validatorRewardPerTokenCumulative[validatorId][token];
$.userValidatorRewardPerTokenPaidTimestamp[user][validatorId][token] = block.timestamp;
```

## Impact Details

Lost yield and permanently frozen yield funds that cannot be claimed by affected users.

## References

* updateRewardPerTokenForValidator snippet: <https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/plume/src/lib/PlumeRewardLogic.sol#L144-L154>

## Proof of Concept

Step-by-step reproduction:

{% stepper %}
{% step %}

### Step

Cumulative reward value for validator isn't updated for some period. Suppose it was last updated at T = 50.
{% endstep %}

{% step %}

### Step

Validator gets slashed at T = 100.
{% endstep %}

{% step %}

### Step

At T = 110, `validatorLastUpdateTime` is (incorrectly) updated to T = 110 via `updateRewardPerTokenForValidator`.
{% endstep %}

{% step %}

### Step

Because `validatorLastUpdateTime` is now > `slashedAtTimestamp`, the slashing flow in `calculateRewardsWithCheckpoints` does not update `validatorRewardPerTokenCumulative` for the period T = 50..100.
{% endstep %}

{% step %}

### Step

A user who last updated their `userValidatorRewardPerTokenPaid` at T = 50 will have `userValidatorRewardPerTokenPaid` equal to the stale cumulative value. When they attempt to claim, `calculateRewardsCore` sees no delta and returns (0, 0, 0), preventing the user from claiming owed rewards.
{% endstep %}
{% endstepper %}

## Additional notes

* The report includes an illustrative image showing the timeline and the missing accounted period: <https://i.imgur.com/3QKHhji.png>
* Do not change or remove the referenced links in this report.
