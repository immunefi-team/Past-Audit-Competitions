# 52588 sc high retroactive reward accrual for newly added tokens when validator was inactive&#x20;

**Submitted on Aug 11th 2025 at 19:47:43 UTC by @light279 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52588
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/RewardsFacet.sol>
* **Impacts:**
  * Theft of unclaimed yield
  * Protocol insolvency

## Description

### Brief / Intro

When a validator is turned inactive the system records a timestamp to cap rewards. If a new reward token is added while the validator is inactive, the contract creates reward checkpoints for all validators (including the inactive one). After the validator is reactivated, users who had already claimed rewards up to the inactive timestamp can later claim the newly added token’s rewards — and the reward calculation mistakenly includes the period when the validator was inactive and the user should not have earned rewards.

In short: users may receive retroactive rewards for a token that was added while the validator was inactive.

## Vulnerability Details

The issue is best understood as a sequence of events:

{% stepper %}
{% step %}

### Validator made inactive and users settled

* Validator is made inactive at time `T` — `slashedAtTimestamp` (used as an inactive cap) is set and users can claim rewards up to `T`.
* User claims all existing-token rewards up to `T` (storage pointers for those tokens are advanced to `T` or otherwise settled).
  {% endstep %}

{% step %}

### New reward token added while validator is inactive

* A new reward token `X` is added while the validator is still inactive.
* `RewardsFacet::addRewardToken` creates an initial checkpoint (with `initialRate`) for every validator, including this inactive validator at time `T'`.
  {% endstep %}

{% step %}

### Validator reactivated

* Validator is reactivated at time `T1`.
* Reactivation resets/creates validator-level checkpoints and `validatorLastUpdateTimes` for tokens.
  {% endstep %}

{% step %}

### User claims token X after reactivation

* User claims token `X` after reactivation (time `T2`, T2 > T1).
* Functional flow: `RewardsFacet::claim(address token,uint16 validatorId)` => `RewardsFacet::_processValidatorRewards` => `PlumeRewardLogic.updateRewardsForValidatorAndToken` => `PlumeRewardLogic::calculateRewardsWithCheckpoints` => `PlumeRewardLogic::_calculateRewardsCore`.
* The code updates the validator cumulative RPT for token `X` (from reactivation → `T2`), then calls `_calculateRewardsCore`.
* Because the user never had any per-token paid-pointer for `X` (token did not exist at time `T`), `lastUserRewardUpdateTime` is set to the stake start of the user — and the distinct timestamp logic includes the token-addition / inactive period window (`T' → T1`).
  {% endstep %}

{% step %}

### Result

* The calculation includes rewards for the interval when the validator was inactive and token `X` had been added (`T' → T1`).
* As a result, the user may receive tokens they should not have accrued.
  {% endstep %}
  {% endstepper %}

### Root causes

* `RewardsFacet::addRewardToken` creates checkpoints for inactive validators without taking inactive status into account.

Relevant code excerpt:

```javascript
function addRewardToken(
        address token,
        uint256 initialRate,
        uint256 maxRate
    ) external onlyRole(PlumeRoles.REWARD_MANAGER_ROLE) {
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
        if (token == address(0)) {
            revert ZeroAddress("token");
        }
        if ($.isRewardToken[token]) {
            revert TokenAlreadyExists();
        }
        if (initialRate > maxRate) {
            revert RewardRateExceedsMax();
        }

        // Prevent re-adding a token in the same block it was removed to avoid checkpoint overwrites.
        if ($.tokenRemovalTimestamps[token] == block.timestamp) {
            revert CannotReAddTokenInSameBlock(token);
        }

        // Add to historical record if it's the first time seeing this token.
        if (!$.isHistoricalRewardToken[token]) {
            $.isHistoricalRewardToken[token] = true;
            $.historicalRewardTokens.push(token);
        }

        uint256 additionTimestamp = block.timestamp;

        // Clear any previous removal timestamp to allow re-adding
        $.tokenRemovalTimestamps[token] = 0;

        $.rewardTokens.push(token);
        $.isRewardToken[token] = true;
        $.maxRewardRates[token] = maxRate;
        $.rewardRates[token] = initialRate; // Set initial global rate
        $.tokenAdditionTimestamps[token] = additionTimestamp;

        // Create a historical record that the rate starts at initialRate for all validators
        uint16[] memory validatorIds = $.validatorIds;
@>        for (uint256 i = 0; i < validatorIds.length; i++) {
@>            uint16 validatorId = validatorIds[i];
@>            PlumeRewardLogic.createRewardRateCheckpoint(
@>                $,
@>                token,
@>                validatorId,
@>                initialRate
@>            );
@>        }

        emit RewardTokenAdded(token);
        if (maxRate > 0) {
            emit MaxRewardRateUpdated(token, maxRate);
        }
    }
```

## Impact Details

Unintended token distribution / inflation — users can receive rewards for periods where the validator was inactive.

## Proof of Concept

The PoC is shown as a step sequence:

{% stepper %}
{% step %}

### T — validator inactive & users settled

* Validator `V` is made inactive via `setValidatorStatus(V, false)` at time `T`.
* `validator.slashedAtTimestamp` (used as cap) is set to `T`.
* User `U` claims all existing token rewards up to `T` (storage pointers updated so U has no pending rewards for those tokens).
  {% endstep %}

{% step %}

### T' — add new token while inactive

* Admin calls `addRewardToken(tokenX, initialRate, maxRate)`.
* `addRewardToken` runs `createRewardRateCheckpoint` for all validators, including `V`. A checkpoint for `tokenX` exists with timestamp `T' (> T)` although `V` was inactive.
  {% endstep %}

{% step %}

### T1 — validator reactivated

* Admin calls `setValidatorStatus(V, true)`. This resets `validatorLastUpdateTimes` and creates new checkpoints signalling activity resumes.
  {% endstep %}

{% step %}

### T2 — user claims tokenX after reactivation

* `U` calls `claim(tokenX, V)` (or a function that processes rewards for `tokenX`).
* `updateRewardsForValidatorAndToken` → `calculateRewardsWithCheckpoints` → `updateRewardPerTokenForValidator` updates cumulative RPT from activation → `T2`.
* `_calculateRewardsCore` computes `lastUserRewardUpdateTime` for `tokenX`: since `U` never had a per-token paid timestamp for `tokenX` (token was added after their earlier claim), the code uses the stake start time of the user and collects distinct timestamps including `T' → T1`.
* The resulting `totalUserRewardDelta` includes reward segments covering `T' → T1` (the period when validator was inactive) — rewards that should have been capped/zero.
  {% endstep %}

{% step %}

### Result

* `U` receives tokens for an interval they should not have been entitled to (the validator was inactive then).
  {% endstep %}
  {% endstepper %}
