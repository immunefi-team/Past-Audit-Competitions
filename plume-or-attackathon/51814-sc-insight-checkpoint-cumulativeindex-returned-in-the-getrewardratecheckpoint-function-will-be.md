# 51814 sc insight checkpoint cumulativeindex returned in the getrewardratecheckpoint function will be zero

**Submitted on Aug 5th 2025 at 22:46:18 UTC by @oxrex for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51814
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/RewardsFacet.sol>
* **Impacts:** (none explicitly listed)

## Description

### Brief/Intro

The `getRewardRateCheckpoint()` function of the RewardsFacet returns 3 values:

* The timestamp the rate checkpoint was added
* The rate of the checkpoint
* The cumulative amount of rewards in rates paid out thus far for that checkpoint rate

The third and last return in this sequence will be zero as it was never updated across reward accrual functions such as inside `updateRewardPerTokenForValidator()` in the PlumeRewardLogic library.

### Vulnerability Details

Across the codebase, the `checkpoint.cumulativeIndex` returned below is never updated during reward token accrual for the rate.

The intention appears to be that this variable tracks how much rewards have been paid out thus far for this reward token using that reward rate, but since it is never updated, it will remain 0.

```solidity
    function getRewardRateCheckpoint(
        address token,
        uint256 index
    ) external view returns (uint256 timestamp, uint256 rate, uint256 cumulativeIndex) {
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
        if (index >= $.rewardRateCheckpoints[token].length) {
            revert InvalidRewardRateCheckpoint(token, index);
        }
        PlumeStakingStorage.RateCheckpoint memory checkpoint = $.rewardRateCheckpoints[token][index];
@>        return (checkpoint.timestamp, checkpoint.rate, checkpoint.cumulativeIndex);
    }
```

Update the variable pro-rata as reward increases for the reward token inside the `PlumeRewardLogic.updateRewardPerTokenForValidator` function the same way it is done for validators. This will be difficult however given that one must locate the index for the rate in the mapping `mapping(address => RateCheckpoint[]) rewardRateCheckpoints;` or remove the mapping if it isn't being used.

### Impact Details

The return value `checkpoint.cumulativeIndex` from `getRewardRateCheckpoint()` will be 0. This can mislead callers into thinking no rewards have been paid out thus far for that reward rate.

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/RewardsFacet.sol#L709-L719>

## Proof of Concept

{% stepper %}
{% step %}

### Set reward rate

PLUME token reward rate is set via `addRewardToken` of the RewardsFacet with a rate of e.g. `1585490000`.
{% endstep %}

{% step %}

### Let time pass and accrue/claim

7 days go by with users having claimed rewards from day 1 to 7 for the reward rate.
{% endstep %}

{% step %}

### Query checkpoint

Query `getRewardRateCheckpoint` to check the stats. It returns:

* The timestamp when the reward rate was set (correct)
* The rate (correct)
* `cumulativeIndex` = `0` (incorrect).\
  Expected example: `958,904,352,000,000` which is `1585490000 * 604,800` (7 days).
  {% endstep %}
  {% endstepper %}
