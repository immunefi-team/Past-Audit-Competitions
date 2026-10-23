# 50168 sc insight unused and duplicated functions should be removed from rewardsfacet and stakingfacet

**Submitted on Jul 22nd 2025 at 08:30:52 UTC by @Vanshika for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50168
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/RewardsFacet.sol>

## Description

### Brief / Intro

`RewardsFacet` contains two internal functions that are duplicates. One of the duplicates is never used and should be removed. There are other duplicate unused functions in `StakingFacet`.

### Vulnerability Details

* In `RewardsFacet`, `_earned()` and `_earnedView()` are duplicates of one another.
* `_calculateTotalEarned()` (which calls `earned()`) is a duplicate of `_calculateTotalEarnedView()` (which calls `_earnedView()`). `_calculateTotalEarnedView()` is called by `earned()` and `getClaimableReward()` which are both external and identical — one of them can be safely removed.
* `_calculateTotalEarned()` and `_earned()` are never called; only their duplicates are used, so they should be removed.
* In `StakingFacet`:
  * `_calculateAndClaimAllRewards()` is never used. A similar function `_calculateAndClaimAllRewardsWithCleanup()` is used during `restakeRewards()` and contains all logic in `_calculateAndClaimAllRewards()`, making the former redundant.
  * `_performRestakeWorkflow()` is another internal function that is never called. `_performStakeSetup()` contains all logic in `_performRestakeWorkflow()` and is called during staking and restaking.
  * Other unused functions include `_updateCommissionClaim()`, `_updateRewardClaim()`, and `_updateWithdrawalAmounts()`.

## Impact Details

Multiple duplicate and unused functions across the facets lead to bloated code. Removing dead/duplicate code reduces maintenance burden and improves readability. There is no functional exploit described — the report is an insight to improve code quality.

## Proof of Concept

Some duplicated code from `RewardsFacet` (excerpt):

<details>

<summary>Proof-of-concept code (click to expand)</summary>

```solidity
// --- Internal View Function (_earned) ---
function _earned(address user, address token, uint16 validatorId) internal returns (uint256 rewards) {
    PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
    uint256 userStakedAmount = $.userValidatorStakes[user][validatorId].staked;
    if (userStakedAmount == 0) {
        return $.userRewards[user][validatorId][token];
    }

    (uint256 userRewardDelta,,) =
        PlumeRewardLogic.calculateRewardsWithCheckpoints($, user, validatorId, token, userStakedAmount);
    rewards = $.userRewards[user][validatorId][token] + userRewardDelta;

    return rewards;
}

// --- View-only helper functions ---
function _earnedView(address user, address token, uint16 validatorId) internal view returns (uint256 rewards) {
    PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
    uint256 userStakedAmount = $.userValidatorStakes[user][validatorId].staked;

    if (userStakedAmount == 0) {
        return $.userRewards[user][validatorId][token];
    }

    (uint256 userRewardDelta,,) =
        PlumeRewardLogic.calculateRewardsWithCheckpointsView($, user, validatorId, token, userStakedAmount);

    rewards = $.userRewards[user][validatorId][token] + userRewardDelta;
    return rewards;
}
```

```solidity
function _calculateTotalEarned(address user, address token) internal returns (uint256 totalEarned) {
    PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
    uint16[] memory validatorIds = $.userValidators[user];

    // Sum across all validators
    for (uint256 i = 0; i < validatorIds.length; i++) {
        uint16 validatorId = validatorIds[i];

        // _earned correctly handles all validator states (active, inactive, slashed)
        // by calling calculateRewardsWithCheckpoints, which respects the slashedAtTimestamp.
        totalEarned += _earned(user, token, validatorId);
    }

    return totalEarned;
}

function _calculateTotalEarnedView(address user, address token) internal view returns (uint256 totalEarned) {
    PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
    uint16[] memory validatorIds = $.userValidators[user];

    for (uint256 i = 0; i < validatorIds.length; i++) {
        uint16 validatorId = validatorIds[i];

        // _earnedView correctly handles all validator states (active, inactive, slashed)
        // by calling calculateRewardsWithCheckpointsView, which respects the slashedAtTimestamp.
        totalEarned += _earnedView(user, token, validatorId);
    }

    return totalEarned;
}
```

```solidity
function earned(address user, address token) external view returns (uint256) {
    return _calculateTotalEarnedView(user, token);
}

function getClaimableReward(address user, address token) external view returns (uint256) {
    return _calculateTotalEarnedView(user, token);
}
```

</details>

## Recommendations (implicit)

* Remove unused duplicate functions in `RewardsFacet` (`_earned`, `_calculateTotalEarned`, etc.) and in `StakingFacet` (`_calculateAndClaimAllRewards`, `_performRestakeWorkflow`, `_updateCommissionClaim`, `_updateRewardClaim`, `_updateWithdrawalAmounts`, etc.).
* Keep only the view or non-view variant that is actually used, and ensure external functions are not duplicated (e.g., remove either `earned()` or `getClaimableReward()` if they are semantically identical and redundant).

(Note: No additional code changes are suggested in this report beyond removing unused/duplicate functions to reduce code bloat.)
