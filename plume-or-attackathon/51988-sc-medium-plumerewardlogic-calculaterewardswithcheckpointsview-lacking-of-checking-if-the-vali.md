# 51988 sc medium plumerewardlogic calculaterewardswithcheckpointsview lacking of checking if the validator is inactive but not slashed&#x20;

* Report ID: #51988
* Report Type: Smart Contract
* Report severity: Medium
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol>
* Submitted on Aug 7th 2025 at 04:05:20 UTC by @jasonxiale for Attackathon | Plume Network: <https://immunefi.com/audit-competition/plume-network-attackathon>

## Description

Brief/Intro

The comments for `PlumeRewardLogic.calculateRewardsWithCheckpointsView` state that this view function "simulates what the state-changing `calculateRewardsWithCheckpoints` does" and "must accurately calculate the validator's theoretical cumulative reward per token at the current block timestamp, respecting all historical rate changes."

However, the current implementation of `PlumeRewardLogic.calculateRewardsWithCheckpointsView` fails to consider the case where a validator is inactive but not slashed, which can lead to using stale cumulative reward-per-token values and therefore incorrect reward calculations.

Relevant source pointers:

* calculateRewardsWithCheckpoints (state-changing): <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L374-L418>
* updateRewardPerTokenForValidator: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L135-L197>
* calculateRewardsWithCheckpointsView: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L862-L917>

## Vulnerability Details

{% stepper %}
{% step %}
In the state-changing function `PlumeRewardLogic.calculateRewardsWithCheckpoints`, when the validator is not slashed the function will call `PlumeRewardLogic.updateRewardPerTokenForValidator`.

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L374-L418>
{% endstep %}

{% step %}
In `PlumeRewardLogic.updateRewardPerTokenForValidator`, if the validator is inactive but not slashed, the function returns early (does not update cumulative RPT).

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L135-L197> (see L155-L160)
{% endstep %}

{% step %}
Because of step 2, the stored value `$.validatorRewardPerTokenCumulative[validatorId][token]` will not be updated; the subsequent code in the state-changing flow uses this updated value when computing rewards.

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L386-L387>
{% endstep %}

{% step %}
The view function `PlumeRewardLogic.calculateRewardsWithCheckpointsView` attempts to simulate cumulative RPT over time. But it only adjusts for slashed validators (it clamps effectiveEndTime to slashedAtTimestamp) and then starts from the stored cumulative value:

Code excerpt:

```
    // calculate effectiveEndTime when the validator is slashed    
    if (validator.slashedAtTimestamp > 0 && validator.slashedAtTimestamp < effectiveEndTime) {
        effectiveEndTime = validator.slashedAtTimestamp;
    }

    // 2. Start with the last known, stored cumulative value and its timestamp.
    uint256 simulatedCumulativeRPT = $.validatorRewardPerTokenCumulative[validatorId][token];
    uint256 lastUpdateTime = $.validatorLastUpdateTimes[validatorId][token];

    // 3. If time has passed since the last update, simulate the RPT increase segment by segment.
    if (effectiveEndTime > lastUpdateTime) {
        ...
    }

    // 4. Now that we have the correctly simulated final cumulative RPT, call the core logic.
    return _calculateRewardsCore($, user, validatorId, token, userStakedAmount, simulatedCumulativeRPT);
```

Because the view does not check whether the validator is inactive (but not slashed), it may use a stale `simulatedCumulativeRPT` when simulating rewards. Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L862-L917>
{% endstep %}
{% endstepper %}

## Impact

Because `PlumeRewardLogic.calculateRewardsWithCheckpointsView` can return an incorrect reward delta for inactive-but-not-slashed validators (using stale cumulative RPT), functions that rely on that view may react incorrectly.

Example: `PlumeRewardLogic.clearPendingRewardsFlagIfEmpty` uses `calculateRewardsWithCheckpointsView`. If the view reports that pending rewards are non-empty (when they should be empty) the cleanup function can return early, which forces callers to traverse a loop every time and results in increased/unbounded gas consumption for users.

References:

* clearPendingRewardsFlagIfEmpty: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L813-L848> (usage at L838-L840)
* Calling path example: StakingFacet -> \_calculateAndClaimAllRewardsWithCleanup -> clearPendingRewardsFlagIfEmpty: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/StakingFacet.sol#L451-L495> and <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/StakingFacet.sol#L953C14-L1003> Overall effect: unnecessary repeated loop iterations, leading to higher gas consumption for users; in worst cases, unbounded gas usage.

## Proof of Concept

{% stepper %}
{% step %}
Trigger flow that leads to cleanup checks:

* Call `StakingFacet.restakeRewards`. This invokes `_calculateAndClaimAllRewardsWithCleanup`. Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/StakingFacet.sol#L451-L495> and L468
  {% endstep %}

{% step %}
Inside `_calculateAndClaimAllRewardsWithCleanup`, `clearPendingRewardsFlagIfEmpty` is invoked. Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/StakingFacet.sol#L953C14-L1003> and L989
{% endstep %}

{% step %}
Because `clearPendingRewardsFlagIfEmpty` calls `PlumeRewardLogic.calculateRewardsWithCheckpointsView`, which does not handle inactive-but-not-slashed validators, the view can return incorrect results causing the cleanup function to return early (see L840), forcing repeated loop traversal on subsequent calls and wasting gas. References: clearPendingRewardsFlagIfEmpty: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L813-L848> and the loop: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L826-L843>
{% endstep %}
{% endstepper %}

## References

* PlumeRewardLogic.sol: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol>
* Specific commit/context used in the report: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol>

(End of report)
