# 51479 sc high inaccurate reward calculation post validator slashing due to premature timestamp update on token removal

**Submitted on Aug 3rd 2025 at 08:15:40 UTC by @light279 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51479
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/RewardsFacet.sol>
* **Impacts:**
  * Theft of unclaimed yield
  * Protocol insolvency

## Description

## Brief/Intro

The Plume staking and reward distribution mechanism includes a vulnerability wherein users may lose a portion of their eligible rewards if a reward token is removed after the validator they delegated to has been slashed. Although the system is designed to allow users to claim rewards accrued until the validator’s slashing time, an interaction between slashing and token removal updates internal timestamps in a way that prevents proper reward calculation.

## Vulnerability Details

When a validator is slashed through `ValidatorFacet::slashValidator`, a `validatorToSlash.slashedAtTimestamp` is set and the validator becomes inactive. Normally, users should be able to claim their pending rewards accrued until the slashing occurred. The claim process flows through several functions:

* `RewardsFacet::claim(address token,uint16 validatorId)` → `RewardsFacet::_processValidatorRewards` → `PlumeRewardLogic.updateRewardsForValidatorAndToken` → `PlumeRewardLogic::calculateRewardsWithCheckpoints`

In the slashed branch of `PlumeRewardLogic::calculateRewardsWithCheckpoints` the code calculates the effective end time as the slashed timestamp (and respects a token removal timestamp if earlier), then uses `validatorLastUpdateTime` to determine whether to add an additional segment of rewards between `validatorLastUpdateTime` and that effective end time:

```javascript
function calculateRewardsWithCheckpoints(
    PlumeStakingStorage.Layout storage $,
    address user,
    uint16 validatorId,
    address token,
    uint256 userStakedAmount
)
    internal
    returns (
        uint256 totalUserRewardDelta,
        uint256 totalCommissionAmountDelta,
        uint256 effectiveTimeDelta
    )
{
    PlumeStakingStorage.ValidatorInfo storage validator = $.validators[
        validatorId
    ];

    if (!validator.slashed) {
        // Normal case: update and use the updated cumulative.
        updateRewardPerTokenForValidator($, token, validatorId);
        uint256 finalCumulativeRewardPerToken = $
            .validatorRewardPerTokenCumulative[validatorId][token];
        return
            _calculateRewardsCore(
                $,
                user,
                validatorId,
                token,
                userStakedAmount,
                finalCumulativeRewardPerToken
            );
    } else {
        // Slashed validator case: calculate what cumulative should be up to the slash timestamp.
        // We DO NOT call updateRewardPerTokenForValidator here because its logic is incorrect for slashed validators.
        uint256 currentCumulativeRewardPerToken = $
            .validatorRewardPerTokenCumulative[validatorId][token];
        uint256 effectiveEndTime = validator.slashedAtTimestamp;

        uint256 tokenRemovalTime = $.tokenRemovalTimestamps[token];
        if (tokenRemovalTime > 0 && tokenRemovalTime < effectiveEndTime) {
            effectiveEndTime = tokenRemovalTime;
        }

        uint256 validatorLastUpdateTime = $.validatorLastUpdateTimes[
            validatorId
        ][token];

        if (effectiveEndTime > validatorLastUpdateTime) {
            uint256 timeSinceLastUpdate = effectiveEndTime -
                validatorLastUpdateTime;

            if (userStakedAmount > 0) {
                PlumeStakingStorage.RateCheckpoint
                    memory effectiveRewardRateChk = getEffectiveRewardRateAt(
                        $,
                        token,
                        validatorId,
                        validatorLastUpdateTime
                    ); // Use rate at start of segment
                uint256 effectiveRewardRate = effectiveRewardRateChk.rate;

                if (effectiveRewardRate > 0) {
                    uint256 rewardPerTokenIncrease = timeSinceLastUpdate *
                        effectiveRewardRate;
                    currentCumulativeRewardPerToken += rewardPerTokenIncrease;
                }
            }
        }

        return
            _calculateRewardsCore(
                $,
                user,
                validatorId,
                token,
                userStakedAmount,
                currentCumulativeRewardPerToken
            );
    }
}
```

However, if `RewardsFacet::removeRewardToken` is called after the validator is slashed, it internally calls `PlumeRewardLogic.updateRewardPerTokenForValidator`, which updates:

```javascript
$.validatorLastUpdateTimes[validatorId][token] = block.timestamp;
```

even for slashed validators. Example snippet:

```javascript
function updateRewardPerTokenForValidator(
    PlumeStakingStorage.Layout storage $,
    address token,
    uint16 validatorId
) internal {
    PlumeStakingStorage.ValidatorInfo storage validator = $.validators[
        validatorId
    ]; // Get validator info

    // --- REORDERED SLASHED/INACTIVE CHECKS ---
    // Check for slashed state FIRST since slashed validators are also inactive
    if (validator.slashed) {
        // For slashed validators, no further rewards or commission accrue.
        // We just update the timestamp to the current time to mark that the state is "settled" up to now.
        $.validatorLastUpdateTimes[validatorId][token] = block.timestamp;

        // Add a defensive check: A slashed validator should never have any stake. If it does, something is
        // wrong with the slashing logic itself.
        if ($.validatorTotalStaked[validatorId] > 0) {
            revert InternalInconsistency(
                "Slashed validator has non-zero totalStaked"
            );
        }
        return;
    }
    ...
}
```

Because `validatorLastUpdateTime` is updated to a time after `slashedAtTimestamp`, subsequent calls to `calculateRewardsWithCheckpoints` will find:

```
if (effectiveEndTime > validatorLastUpdateTime)
```

to be false (effectiveEndTime == slashedAtTimestamp < validatorLastUpdateTime), preventing the additional reward-per-token increase from being calculated for the period between the previous `validatorLastUpdateTime` and the slashing time. This results in user rewards accrued up to the slashing moment becoming unclaimable.

## Impact Details

{% hint style="danger" %}
This results in loss of unclaimed user rewards that should have been claimable up to the slashing timestamp. Since the tokens are not stolen but rendered unclaimable due to incorrect accounting, the impact classification is:

High — Theft of unclaimed yield

This affects all delegators of a validator that has been slashed and subsequently had its reward token removed, causing unexpected reward loss without any direct malicious activity.
{% endhint %}

## Proof of Concept

{% stepper %}
{% step %}

### Setup and Normal Operation

1. User stakes tokens with validator V1.
2. Rewards accumulate normally. User claims up to time t where rewards are updated for validator and user.
   {% endstep %}

{% step %}

### Slashing

3. Validator V1 is slashed; `slashedAtTimestamp = T1` is set, with T1 > t.
4. The user does not claim immediately.
   {% endstep %}

{% step %}

### Token Removal After Slashing

5. `removeRewardToken(token)` is called after T1, which:
   * Calls `updateRewardPerTokenForValidator()` internally.
   * Updates `validatorLastUpdateTimes[V1][token] = block.timestamp = T2` (T2 > T1).
     {% endstep %}

{% step %}

### Claim After Token Removal

6. The user now tries to claim rewards.
7. In `calculateRewardsWithCheckpoints`, `effectiveEndTime = slashedAtTimestamp = T1`.
8. The condition `if (effectiveEndTime > validatorLastUpdateTime)` becomes false because validatorLastUpdateTime = T2 > T1.
9. No `rewardPerTokenIncrease` is calculated for the interval up to the slash, and the call to `_calculateRewardsCore` receives an outdated `currentCumulativeRewardPerToken`.
10. As a result, rewards that should have been claimable up to T1 are not awarded to the user.
    {% endstep %}
    {% endstepper %}

## Notes / Observations

* The bug arises from updating `validatorLastUpdateTimes` when handling token removal for already-slashed validators. This update should not advance the last-update time beyond the `slashedAtTimestamp` used for reward calculation, because it effectively erases the period for which rewards should be computed at claim time.
* The flow where `updateRewardPerTokenForValidator` is invoked during token removal leads to the incorrect timestamp progression.
