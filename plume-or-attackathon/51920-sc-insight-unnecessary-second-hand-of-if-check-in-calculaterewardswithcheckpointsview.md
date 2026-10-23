# 51920 sc insight unnecessary second hand of if check in calculaterewardswithcheckpointsview&#x20;

**Submitted on Aug 6th 2025 at 15:40:27 UTC by @oxrex for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51920
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/RewardsFacet.sol>

## Description

### Brief/Intro

There is a redundant condition in the `if` check of the `calculateRewardsWithCheckpointsView` function of the `PlumeRewardLogic` library.

### Vulnerability Details

Relevant snippet:

```solidity
function calculateRewardsWithCheckpointsView(
    PlumeStakingStorage.Layout storage $,
    address user,
    uint16 validatorId,
    address token,
    uint256 userStakedAmount
)
    internal
    view
    returns (uint256 totalUserRewardDelta, uint256 totalCommissionAmountDelta, uint256 effectiveTimeDelta)
{
    // This view function simulates what the state-changing `calculateRewardsWithCheckpoints` does.
    // It must accurately calculate the validator's theoretical cumulative reward per token at the
    // current block timestamp, respecting all historical rate changes.

    // 1. Determine the effective time range for the simulation.
    PlumeStakingStorage.ValidatorInfo storage validator = $.validators[validatorId];

    uint256 effectiveEndTime = block.timestamp;
    uint256 tokenRemovalTime = $.tokenRemovalTimestamps[token];
@>       if (tokenRemovalTime > 0 && tokenRemovalTime < effectiveEndTime) { // @audit insight
        effectiveEndTime = tokenRemovalTime;
    }
    if (validator.slashedAtTimestamp > 0 && validator.slashedAtTimestamp < effectiveEndTime) {
        effectiveEndTime = validator.slashedAtTimestamp;
    }

    ...

    // 4. Now that we have the correctly simulated final cumulative RPT, call the core logic.
    return _calculateRewardsCore($, user, validatorId, token, userStakedAmount, simulatedCumulativeRPT);
}
```

The second part of the `if` condition (`&& tokenRemovalTime < effectiveEndTime`) is redundant because `effectiveEndTime` was initialized as `block.timestamp`. The token removal timestamp cannot be in the future relative to `block.timestamp`. Therefore, checking `< effectiveEndTime` is unnecessary once you already check `tokenRemovalTime > 0`.

Suggested minimal change:

```diff
- if (tokenRemovalTime > 0 && tokenRemovalTime < effectiveEndTime) {
+ if (tokenRemovalTime > 0) {
        effectiveEndTime = tokenRemovalTime;
    }
```

Even if the token removal occurred in the current block, `tokenRemovalTime` would be equal to `block.timestamp`, not greater than it, so the extra comparison adds no value.

## Impact Details

The `&& tokenRemovalTime < effectiveEndTime` clause is redundant and does not alter behavior given the current initialization of `effectiveEndTime`. Removing it simplifies the code without changing semantics.

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol#L884-L886>

## Proof of Concept

{% stepper %}
{% step %}

### Step

There are 2 reward tokens pUSD and PLUME
{% endstep %}

{% step %}

### Step

pUSD gets removed at timestamp 555
{% endstep %}

{% step %}

### Step

Functions query `calculateRewardsWithCheckpointsView` of the `PlumeRewardLogic`
{% endstep %}

{% step %}

### Step

The function does an unnecessary second hand check in the `if` condition whereby it determines the `effectiveEndTime` that is relevant for use. But since the `uint256 effectiveEndTime` at the very start above the if check = block.timestamp, it will already be the most recent and cannot be lower than `validator.slashedAtTimestamp` in the first place
{% endstep %}
{% endstepper %}
