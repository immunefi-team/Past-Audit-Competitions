# 52084 sc high unstaking before reward token removal leads to incorrect reward accrual on re addition

* Report ID: #52084
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* Impacts:
  * Theft of unclaimed yield
  * Protocol insolvency
* Submitted on Aug 7th 2025 at 19:59:16 UTC by @light279 for Attackathon | Plume Network (<https://immunefi.com/audit-competition/plume-network-attackathon>)

## #52084 \[SC-High] Unstaking Before Reward Token Removal Leads to Incorrect Reward Accrual on Re-addition

### Description

#### Brief / Intro

In the Plume staking system, if a user fully unstakes from a validator before a reward token is removed and later restakes after the token has been removed, user reward pointers are not correctly reinitialized for that historical token. If the removed token is re-added later, the stale user reward pointers cause rewards from the period when the user had no stake to be included in their claim, resulting in incorrect and inflated reward claims.

#### Vulnerability Details

When a user calls StakingFacet::unstake, the system updates their reward data via PlumeRewardLogic.updateRewardsForValidator, setting:

* $.userValidatorRewardPerTokenPaid
* $.userValidatorRewardPerTokenPaidTimestamp

for all historical reward tokens (including those active at the time of unstake, T1). If a reward token is removed at T2, it is removed from the rewardTokens array but user metadata for that token is preserved (no deletion). When the user restakes at T3, `_initializeRewardStateForNewStake()` iterates only over the current rewardTokens array and does not initialize state for removed (historical) tokens. Thus the user's stored pointers for the removed token remain those set at T1.

If the token is re-added at T4 and the user later claims at T5, reward calculation uses the stale pointers (from T1) and includes rewards accrued from T1→T2, during which the user had no stake.

Functional call flow for claiming (simplified): RewardsFacet::claim(token,validatorId) => RewardsFacet::\_processValidatorRewards(user, validatorId) => PlumeRewardLogic.updateRewardsForValidatorAndToken($, user, validatorId, token) => calculateRewardsWithCheckpoints(...) => \_calculateRewardsCore(...)

Excerpt from \_calculateRewardsCore(...) showing use of stale user pointers:

```javascript
function _calculateRewardsCore(
        PlumeStakingStorage.Layout storage $,
        address user,
        uint16 validatorId,
        address token,
        uint256 userStakedAmount,
        uint256 currentCumulativeRewardPerToken
    )
        internal
        view
        returns (
            uint256 totalUserRewardDelta,
            uint256 totalCommissionAmountDelta,
            uint256 effectiveTimeDelta
        )
    {
        uint256 lastUserPaidCumulativeRewardPerToken = $
            .userValidatorRewardPerTokenPaid[user][validatorId][token];
        uint256 lastUserRewardUpdateTime = $
            .userValidatorRewardPerTokenPaidTimestamp[user][validatorId][token];
    ...
```

### Impact Details

* Reward Overpayment: Users can claim unearned rewards for removed tokens corresponding to the period between their full unstake (T1) and the token removal (T2), despite having no stake during that time.

### Proof of Concept

{% stepper %}
{% step %}

### User stakes and earns some rewards

* At time T0, user stakes 100 tokens on validator 1.
  {% endstep %}

{% step %}

### User fully unstakes at time T1

This triggers updateRewardsForValidator(), which sets:

* userValidatorRewardPerTokenPaid\[0xUser]\[1]\[tokenA] to latest cumulative RPT.
* userValidatorRewardPerTokenPaidTimestamp\[0xUser]\[1]\[tokenA] = T1
  {% endstep %}

{% step %}

### tokenA is removed at time T2

* rewardRates\[tokenA] = 0
* tokenA is removed from rewardTokens array
* A final checkpoint with rate 0 is written
  {% endstep %}

{% step %}

### User restakes at time T3 (after tokenA was removed)

* User calls stake/restake which calls \_performStakeSetup.
* Because the user's stake is zero, \_initializeRewardStateForNewStake is called.
* \_initializeRewardStateForNewStake iterates only over current rewardTokens; tokenA is not present.
* Therefore:
  * userValidatorRewardPerTokenPaid\[0xUser]\[1]\[tokenA] remains unchanged (still set at T1).
  * userValidatorRewardPerTokenPaidTimestamp\[0xUser]\[1]\[tokenA] remains unchanged (still T1).

Relevant code:

```javascript
function _initializeRewardStateForNewStake(
        address user,
        uint16 validatorId
    ) internal {
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();

        $.userValidatorStakeStartTime[user][validatorId] = block.timestamp;

        address[] memory rewardTokens = $.rewardTokens;
        for (uint256 i = 0; i < rewardTokens.length; i++) {
            address token = rewardTokens[i];
            if ($.isRewardToken[token]) {
                PlumeRewardLogic.updateRewardPerTokenForValidator(
                    $,
                    token,
                    validatorId
                );

                $.userValidatorRewardPerTokenPaid[user][validatorId][token] = $
                    .validatorRewardPerTokenCumulative[validatorId][token];
                $.userValidatorRewardPerTokenPaidTimestamp[user][validatorId][
                    token
                ] = block.timestamp;
            }
        }
    }
```

{% endstep %}

{% step %}

### tokenA is re-added at time T4

* tokenA is pushed back into rewardTokens.
  {% endstep %}

{% step %}

### User claims rewards at time T5

* updateRewardsForValidatorAndToken(tokenA) is called.
* calculateRewardsWithCheckpoints() -> \_calculateRewardsCore() uses:
  * userValidatorRewardPerTokenPaidTimestamp = T1
  * userValidatorRewardPerTokenPaid = value at T1

Result: reward calculation includes T1 → T2 period even though user had no stake then, causing overpayment.
{% endstep %}
{% endstepper %}
