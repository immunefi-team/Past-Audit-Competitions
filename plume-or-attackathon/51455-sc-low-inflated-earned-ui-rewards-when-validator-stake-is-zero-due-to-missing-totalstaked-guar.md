# 51455 sc low inflated earned ui rewards when validator stake is zero due to missing totalstaked guard in view logic

**Submitted on Aug 3rd 2025 at 00:12:53 UTC by @Rhaydden for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51455
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

`calculateRewardsWithCheckpointsView` keeps accruing reward-per-token during any period where a validator’s `totalStaked` is zero, whereas the state-changing path (`updateRewardPerTokenForValidator`) correctly stops accrual. The mismatch causes `earned()` and all UI displays to show rewards users can never claim.

## Vulnerability Details

In `PlumeRewardLogic.sol` (view path):

```solidity
// VIEW path
uint256 rptIncreaseInSegment = segmentDuration * rateForSegment;
simulatedCumulativeRPT += rptIncreaseInSegment;
```

The loop runs for every time segment with a non zero reward rate. There is no check that the validator actually has stake during the segment.

Opposite of what the state-changing path does:

```solidity
// STATE-CHANGING path
if (totalStaked > 0) {
    uint256 rewardPerTokenIncrease = timeSinceLastUpdate * effectiveRewardRate;
    $.validatorRewardPerTokenCumulative[validatorId][token] += rewardPerTokenIncrease;
}
```

When `totalStaked == 0`, the cumulative value on chain is frozen and only `validatorLastUpdateTimes` is advanced. Because the view function starts its simulation from that stored timestamp, it integrates reward rate across the zero-stake window, inflating `simulatedCumulativeRPT`. `_calculateRewardsCore` then computes an overstated user delta, so `earned()` differs from the value that will be paid by `claim()`.

Conditions required for the discrepancy:

{% stepper %}
{% step %}
Validator is active but has momentarily zero stake (all delegators withdrew).
{% endstep %}

{% step %}
Reward rate for the token > 0.
{% endstep %}

{% step %}
No transaction hits `updateRewardPerTokenForValidator` during that interval (common when there is no stake).
{% endstep %}
{% endstepper %}

Every second that passes, the gap between displayed and claimable rewards widens.

## Impact Details

{% hint style="info" %}
Low – contract fails to deliver promised returns, but doesn't lose value. Users see larger rewards than they can claim.
{% endhint %}

## Fix

Add a stake guard identical to the state-changing path:

```solidity
if ($.validatorTotalStaked[validatorId] > 0 && rateForSegment > 0) {
    simulatedCumulativeRPT += segmentDuration * rateForSegment;
}
```

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L907-L909>

## Proof of Concept

{% stepper %}
{% step %}
Add validator V with reward rate R > 0.
{% endstep %}

{% step %}
Delegate 100 tokens to V. Wait 1 hour; call `earned(user, V)` -> returns E1.
{% endstep %}

{% step %}
User withdraws entire stake (totalStake becomes 0). Do nothing else for 2 hours.
{% endstep %}

{% step %}
Call `earned(user, V)` again -> returns E2 where `E2 - E1 ≈ 2 h * R`.
{% endstep %}

{% step %}
Now call `claim(user, V)`; the user only receives E1 (plus small rounding), proving the extra 2 hours of rewards were never claimable.
{% endstep %}
{% endstepper %}
