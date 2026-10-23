# 53039 sc high rewards and commissions accrued in the interval before a slash might be lost

**Submitted on Aug 14th 2025 at 17:50:39 UTC by @a16 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #53039
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief/Intro

When a validator becomes slashed, all the rewards (and the associated commissions) that were accrued up until that point should still be accounted for. However, since *currentCumulativeRewardPerToken* might not be properly incremented in such scenario, rewards and commissions might be lost.

### Vulnerability Details

The core issue is that when `updateRewardPerTokenForValidator()` is called for slashed/inactive validators, the *currentCumulativeRewardPerToken* will not be incremented to account for the time that passed between the last update and the slash/deactivation. This means that a user that was synced with *currentCumulativeRewardPerToken* on the last update would remain synced even after the call to `updateRewardPerTokenForValidator()`.

If that user later calls `claim()`, which will eventually trigger the `_calculateRewardsCore()` function, `userValidatorRewardPerTokenPaid[user][validatorId][token]` would be the same as `currentCumulativeRewardPerToken` (even though the timestamps are not synced), and the:

```
if (
    effectiveEndTime <= lastUserRewardUpdateTime
        || currentCumulativeRewardPerToken <= lastUserPaidCumulativeRewardPerToken
) {
    return (0, 0, 0);
}
```

shortcut would skip the calculation all together (although it shouldn't, as the accrued rewards between the last update and the slash were not accrued yet).

As noted in the source, `currentCumulativeRewardPerToken` is also not incremented by `updateRewardPerTokenForValidator()` in the case of an inactive validator. The key difference is that when a validator becomes inactive, `_settleCommissionForValidatorUpToNow()` is called, which in turn calls `updateRewardPerTokenForValidator()` and this happens before the validator becomes inactive, so `currentCumulativeRewardPerToken` is correctly updated.

This potential problem is mentioned in line 390 in `PlumeRewardLogic.sol`:

// We DO NOT call updateRewardPerTokenForValidator here because its logic is incorrect for slashed validators.

The problem is that if `updateRewardPerTokenForValidator()` was already called before `calculateRewardsWithCheckpoints()` was called (for example, if the relevant token was removed), `validatorLastUpdateTimes[validatorId][token]` would be incremented to pass `effectiveEndTime` and `currentCumulativeRewardPerToken` would never be incremented.

Note that the validator also loses commission for that interval between the last update and the slash since `updateRewardPerTokenForValidator()` skips that part for slashed validators.

### Impact Details

Rewards and commissions earned by users and validators for the interval between the last update and the actual slash might not be accounted for.

### Suggestions

Make sure that `currentCumulativeRewardPerToken` is updated, and consider updating the state for all (validator, reward tokens) pairs before actually slashing that validator.

## Proof of Concept

{% stepper %}
{% step %}
User stakes to a validator.

User \_u stakes some amount to Validator \_v.
{% endstep %}

{% step %}
User syncs rewards for a token.

User \_u calls `claim()` for reward token *rt*. The internal function `_processValidatorRewards()` calls `updateRewardsForValidatorAndToken()`, which syncs `userValidatorRewardPerTokenPaid[_u][_v][_rt]` with `validatorRewardPerTokenCumulative[_v][_rt]`.
{% endstep %}

{% step %}
Validator gets slashed.

Some time passes, and Validator \_v gets slashed.
{% endstep %}

{% step %}
Reward token removal triggers update but not cumulative increment.

Reward token \_rt is removed. This call iterates over all validators (including the slashed ones) and calls `updateRewardPerTokenForValidator(_rt, _v)`, which updates `validatorLastUpdateTimes[_v][_rt]` without incrementing `validatorRewardPerTokenCumulative[_v][_rt]`.
{% endstep %}

{% step %}
User claims later and accrual is skipped.

User \_u calls `claim()`. This triggers `calculateRewardsWithCheckpoints()`, which attempts to apply the "patch" that fixes `currentCumulativeRewardPerToken` but incorrectly skips it since `effectiveEndTime > validatorLastUpdateTime` is false. `_calculateRewardsCore()` returns early and no rewards are actually accrued.
{% endstep %}
{% endstepper %}
