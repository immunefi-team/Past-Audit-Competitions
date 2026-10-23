# 51842 sc high unclaimed staker rewards lost when admin clears validator records without checking pending rewards

**Submitted on Aug 6th 2025 at 07:43:59 UTC by @farman1094 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51842
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ManagementFacet.sol>
* **Impacts:** Permanent freezing of funds

## Description

### Brief / Intro

The function `ManagementFacet::adminBatchClearValidatorRecords` allows an administrator to delete user records for a validator without checking for or processing any pending rewards. This results in users permanently losing access to rewards they legitimately earned before a slashing event or before the record was cleared.

### Vulnerability Details

When a validator is slashed, a staker may lose some or all of their staked amount, but can still claim any rewards accrued up to the `slashAtTimestamp`. However, the `ManagementFacet::adminBatchClearValidatorRecords` function is used to remove user records for a validator without verifying whether those users have unclaimed rewards. As a result, important user state is updated:

```
function adminBatchClearValidatorRecords(address[] calldata users, address validator) external onlyAdmin {
...
   $.userValidatorStakes[user][slashedValidatorId].staked = 0;
                    // Decrement user's global stake
                    if ($.stakeInfo[user].staked >= userActiveStakeToClear) {
                        $.stakeInfo[user].staked -= userActiveStakeToClear;
...
}
```

This breaks the protocol’s intended safety for stakers: even if their stake is slashed, their pending rewards up to `slashedAtTimestamp` should remain claimable. Clearing state without checking pending rewards directly causes permanent loss of those rewards.

The reward update function returns early when the user's stake is zero:

```
// PlumeRewardLogic::updateRewardsForValidatorAndToken
     if (userStakedAmount == 0) {
            // If user has no stake, there's nothing to calculate. We still need to update the user's "paid" pointers
            // to the latest global state to prevent incorrect future calculations.
            // First, ensure the validator's state is up-to-date.
            if (!$.validators[validatorId].slashed) {
                updateRewardPerTokenForValidator($, token, validatorId);
            }
            $.userValidatorRewardPerTokenPaid[user][validatorId][token] =
                $.validatorRewardPerTokenCumulative[validatorId][token];
            $.userValidatorRewardPerTokenPaidTimestamp[user][validatorId][token] = block.timestamp;
            return;
```

Because of this early return, rewards accrued between the user's last update and `slashedAtTimestamp` are effectively lost.

### Impact Details

This vulnerability results in direct, irreversible loss of user rewards. Even if the user loses their staked funds due to slashing, they are still entitled to rewards legitimately earned from `lastUpdatedTime` to `slashedAtTimestamp`. The admin clearing logic does not check for pending rewards before wiping state, causing permanent loss of those reward funds.

## Proof of Concept

{% stepper %}
{% step %}

### Step 1 — Setup

Assume Alice is a staker who has deposited funds with a validator. The validator is actively generating rewards, and Alice accrues unclaimed rewards over time; her per-validator state hasn't been updated to reflect the most recent accruals.
{% endstep %}

{% step %}

### Step 2 — Validator is slashed

The validator is slashed. According to protocol rules, Alice should still be able to claim her rewards up to `slashedAtTimestamp`. The reward calculator confirms this by capping the effective end time at `slashedAtTimestamp`:

```
// calculateRewardsWithCheckpoints
  uint256 currentCumulativeRewardPerToken = $.validatorRewardPerTokenCumulative[validatorId][token];
  uint256 effectiveEndTime = validator.slashedAtTimestamp;
  ...
  return _calculateRewardsCore($, user, validatorId, token, userStakedAmount, currentCumulativeRewardPerToken);
```

{% endstep %}

{% step %}

### Step 3 — Admin clears validator records

An admin calls `ManagementFacet::adminBatchClearValidatorRecords` to clear the slashed validator's records. The function:

* Removes Alice’s stake record for the slashed validator.
* Sets Alice’s stake to zero for that validator.
* Decrements Alice’s global stake by the cleared amount.

Relevant excerpt:

```
if (userActiveStakeToClear > 0) {
    $.userValidatorStakes[user][slashedValidatorId].staked = 0;
    // Decrement user's global stake
    if ($.stakeInfo[user].staked >= userActiveStakeToClear) {
        $.stakeInfo[user].staked -= userActiveStakeToClear;
    } else {
        $.stakeInfo[user].staked = 0;
    }
```

{% endstep %}

{% step %}

### Step 4 — Attempt to claim later

When Alice later tries to claim rewards (e.g., via `RewardsFacet::claim`), the claim flow calls `_processValidatorRewards` -> `PlumeRewardLogic::updateRewardsForValidatorAndToken`. Since Alice's per-validator stake is now zero, the update function returns early and does not calculate rewards:

```
// updateRewardsForValidatorAndToken
if (userStakedAmount == 0) {
    // ...
    if (!$.validators[validatorId].slashed) {
        updateRewardPerTokenForValidator($, token, validatorId);
    }
    $.userValidatorRewardPerTokenPaid[user][validatorId][token] =
        $.validatorRewardPerTokenCumulative[validatorId][token];
    $.userValidatorRewardPerTokenPaidTimestamp[user][validatorId][token] = block.timestamp;
    return;
}
```

All unclaimed rewards accrued between Alice's last reward update and `slashedAtTimestamp` are therefore not computed nor credited.
{% endstep %}

{% step %}

### Step 5 — Result

Alice loses any unclaimed rewards from the period up to `slashedAtTimestamp` permanently. There is no mechanism in the admin clear function to preserve or process pending rewards before wiping user state.
{% endstep %}
{% endstepper %}

## References / Affected Files

* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ManagementFacet.sol>

(End of report)
