# 52422 sc low using the current time in geteffectiverewardrateat will result in incorrect reward calculation for an entire duration of a time segment

**Submitted on Aug 10th 2025 at 14:46:28 UTC by @WinSec for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52422
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

Using `block.timestamp`, instead of the older timestamp, in `getEffectiveRewardRateAt` can under/over-accrue rewards for the entire interval when rates change mid-interval. This could result in users and validators being paid less than intended.

## Vulnerability Details

In updateRewardPerTokenForValidator:

```solidity
        uint256 totalStaked = $.validatorTotalStaked[validatorId];
        uint256 oldLastUpdateTime = $.validatorLastUpdateTimes[validatorId][token];

        if (block.timestamp > oldLastUpdateTime) {
            if (totalStaked > 0) {
                uint256 timeDelta = block.timestamp - oldLastUpdateTime; 
                // Get the reward rate effective for the segment ending at block.timestamp 
                PlumeStakingStorage.RateCheckpoint memory effectiveRewardRateChk =
                    getEffectiveRewardRateAt($, token, validatorId, block.timestamp);
                uint256 effectiveRewardRate = effectiveRewardRateChk.rate;

                if (effectiveRewardRate > 0) {
                    uint256 rewardPerTokenIncrease = timeDelta * effectiveRewardRate; 
                    $.validatorRewardPerTokenCumulative[validatorId][token] += rewardPerTokenIncrease;

                    // Accrue commission for the validator for this segment
                    // The commission rate should be the one effective at the START of this segment (oldLastUpdateTime)
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
                }
            }
        }
        // Update last global update time for this validator/token AFTER all calculations for the segment
        $.validatorLastUpdateTimes[validatorId][token] = block.timestamp;
```

`block.timestamp` is passed as an argument to the `getEffectiveRewardRateAt` function which gets the reward rate for the validator and token at the given timestamp (here, `block.timestamp`) instead of using `oldLastUpdateTime`. This is inconsistent with how this function is being used in other parts of the codebase.

For example in `_calculateRewardsCore`:

```solidity
            // What is the validator's RPT at segmentEndTime?
            // This requires calculating the RPT increase *within this specific segment*.
            PlumeStakingStorage.RateCheckpoint memory rewardRateInfoForSegment =
                getEffectiveRewardRateAt($, token, validatorId, segmentStartTime); // Rate at START of segment
            uint256 effectiveRewardRate = rewardRateInfoForSegment.rate;
            uint256 segmentDuration = segmentEndTime - segmentStartTime;
```

`segmentStartTime` is accurately being used here and not `segmentEndTime`.

Using `block.timestamp` (end of segment) in `updateRewardPerTokenForValidator` will use the wrong reward rate for the entire duration of the segment. Hence, `rewardPerTokenIncrease` is calculated incorrectly:

```solidity
                    uint256 rewardPerTokenIncrease = timeDelta * effectiveRewardRate;
                    $.validatorRewardPerTokenCumulative[validatorId][token] += rewardPerTokenIncrease;
```

So, `validatorRewardPerTokenCumulative` is incremented with a wrong value. This incorrect value is used in other calculations as well:

```solidity
                    uint256 commissionRateForSegment = getEffectiveCommissionRateAt($, validatorId, oldLastUpdateTime);
                    uint256 grossRewardForValidatorThisSegment =
                        (totalStaked * rewardPerTokenIncrease) / PlumeStakingStorage.REWARD_PRECISION; //@audit - what if reward has 6 decimals?

                    // Use regular division (floor) for validator's accrued commission
                    uint256 commissionDeltaForValidator = (
                        grossRewardForValidatorThisSegment * commissionRateForSegment
                    ) / PlumeStakingStorage.REWARD_PRECISION; //@audit - why not ceil?

                    if (commissionDeltaForValidator > 0) {
                        $.validatorAccruedCommission[validatorId][token] += commissionDeltaForValidator;
                    }
```

`grossRewardForValidatorThisSegment`, `commissionDeltaForValidator` and hence `validatorAccruedCommission` will be incorrectly updated.

Also, notice that in contrast `getEffectiveCommissionRateAt` in the above code block is using `oldLastUpdateTime`, which is the expected behaviour.

## Impact Details

* If the reward rate changed between `oldLastUpdateTime` and now (`block.timestamp`), using the end-of-segment rate applies the wrong rate to the entire `timeDelta`.
* If end rate < start rate, accrual for the entire interval can be understated or even zeroed.
* If end rate > start rate, accrual can be overstated for the interval.

This can lead to loss of yield for the users and the validator, especially when end rate < start rate. Therefore this issue warrants a high severity.

## References

<https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/plume/src/lib/PlumeRewardLogic.sol#L170-L171>

## Proof of Concept

{% stepper %}
{% step %}

### Step

`updateRewardPerTokenForValidator` is called internally, for example when a new user stakes funds.
{% endstep %}

{% step %}

### Step

`updateRewardPerTokenForValidator` fetches the `getEffectiveRewardRateAt` at the current timestamp instead of the old timestamp.
{% endstep %}

{% step %}

### Step

Rewards calculated will be based on the newer rate than the older one for the entire duration of the time segment.
{% endstep %}

{% step %}

### Step

If the end rate was lower than the start rate, accrual will be understated.
{% endstep %}

{% step %}

### Step

If the end rate was greater, the accrual of rewards will be overstated.
{% endstep %}
{% endstepper %}
