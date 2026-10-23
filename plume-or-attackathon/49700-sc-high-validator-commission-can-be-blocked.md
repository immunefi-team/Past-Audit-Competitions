# 49700 sc high validator commission can be blocked

**Submitted on Jul 18th 2025 at 14:15:52 UTC by @Blobism for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #49700
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief/Intro

The validator reward function contains multiple flooring integer divisions to compute the commission delta for the validator since the last update. Anyone can get this update method to be invoked inexpensively, meaning validator commission rewards can be completely blocked for certain tokens.

### Vulnerability Details

The `updateRewardPerTokenForValidator` contains the following lines of code with the flooring divisions:

```solidity
uint256 commissionRateForSegment = getEffectiveCommissionRateAt($, validatorId, oldLastUpdateTime);
uint256 grossRewardForValidatorThisSegment =
    (totalStaked * rewardPerTokenIncrease) / PlumeStakingStorage.REWARD_PRECISION;

// Use regular division (floor) for validator's accrued commission
uint256 commissionDeltaForValidator = (
    grossRewardForValidatorThisSegment * commissionRateForSegment
) / PlumeStakingStorage.REWARD_PRECISION;

if (commissionDeltaForValidator > 0) {
    $.validatorAccruedCommission[validatorId][token] += commissionDeltaForValidator;
}
```

At every iteration of this method, the `validatorLastUpdateTimes` value is set to the new block timestamp for this token, regardless of if the commission delta was nonzero. This means that repeatedly invoking this method at a frequent interval could keep the commission delta as always being zero. Any user can get this method invoked inexpensively, which is where the issue lies.

Recommended fix: Either consider making the `updateRewardPerTokenForValidator` method harder to invoke by any user, or do not update the timestamp when the commission delta is zero (though this may require more consideration of checkpoints).

### Impact Details

Commission reward for a validator can be blocked with this attack.

The attack is more viable if the reward rate for a token is low, since this will require more time for the token rewards to increase to a sufficient level for the flooring division to not be zero. Having a small commission rate and a small total stake on the validator will also help the attacker.

### References

See `plume/src/lib/PlumeRewardLogic.sol`

## Proof of Concept

{% stepper %}
{% step %}

### Step

An attacker stakes with the validator.
{% endstep %}

{% step %}

### Step

The attacker unstakes dust amounts repeatedly, or unstakes then restakes repeatedly.
{% endstep %}

{% step %}

### Step

Each unstake/restake invokes `PlumeRewardLogic.updateRewardsForValidator`, which will internally call `updateRewardPerTokenForValidator`.
{% endstep %}

{% step %}

### Step

The last update time of the validator is updated, but the commission reward delta remains at zero.
{% endstep %}
{% endstepper %}
