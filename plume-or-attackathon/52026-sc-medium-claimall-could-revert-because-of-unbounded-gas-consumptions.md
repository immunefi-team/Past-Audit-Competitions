# 52026 sc medium claimall could revert because of unbounded gas consumptions

* Submitted on Aug 7th 2025 at 11:51:26 UTC by @WinSec for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)
* Report ID: #52026
* Report Type: Smart Contract
* Report severity: Medium
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/RewardsFacet.sol>
* Impacts:
  * Unbounded gas consumption

## Description

### Brief / Intro

Once a validator is added the `validatorExists` mapping is set to true and there is no way to unset it even if the validator is slashed. The `claimAll()` function iterates over all reward tokens and, for each token, iterates over all validators associated with the user — and additional loops run when clearing and cleaning up validator relationships. If validators accumulate over time (and some are slashed but not removed), `claimAll()` can run out of gas and revert.

<details>

<summary>Vulnerability Details (expand)</summary>

`addValidator` sets the validator status to active and also updates the `validatorExists` mapping:

```solidity
validator.active = true;
$.validatorExists[validatorId] = true;
```

`setValidatorStatus` toggles the status to inactive, but when a validator is slashed it is not set inactive nor removed from the `validatorExists` mapping, so the `validatorExists` entry for a slashed validator remains `true`.

`claimAll` iterates over all tokens and calls `_processAllValidatorRewards(msg.sender, token)`, which itself iterates over all validator IDs for the user and calls `_processValidatorRewards` for each:

```solidity
function claimAll() external nonReentrant returns (uint256[] memory) {
    PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
    address[] memory tokens = $.rewardTokens;
    uint256[] memory claims = new uint256[](tokens.length);

    // Process each token
    for (uint256 i = 0; i < tokens.length; i++) {
        address token = tokens[i];

        // Process rewards from all active validators for this token
        uint256 totalReward = _processAllValidatorRewards(msg.sender, token);

        // Finalize claim if there are rewards
        if (totalReward > 0) {
            _finalizeRewardClaim(token, totalReward, msg.sender);
            claims[i] = totalReward;
            emit RewardClaimed(msg.sender, token, totalReward);
        }
    }

    // Clear pending flags for all validators after claiming all tokens
    uint16[] memory validatorIds = $.userValidators[msg.sender];
    _clearPendingRewardFlags(msg.sender, validatorIds);

    // Clean up validator relationships for validators with no remaining involvement
    PlumeValidatorLogic.removeStakerFromAllValidators($, msg.sender);

    return claims;
}
```

`_processAllValidatorRewards` contains a loop like:

```solidity
for (uint256 i = 0; i < validatorIds.length; i++) {
    uint16 validatorId = validatorIds[i];

    // The underlying reward processing logic correctly handles all validator states
    // (active, inactive, slashed) by respecting the relevant timestamps

    uint256 rewardFromValidator = _processValidatorRewards(user, validatorId, token);
    totalReward += rewardFromValidator;
}
```

And `removeStakerFromAllValidators` iterates again:

```solidity
function removeStakerFromAllValidators(PlumeStakingStorage.Layout storage $, address staker) internal {
    // Make a copy to avoid iteration issues when removeStakerFromValidator is called
    uint16[] memory userAssociatedValidators = $.userValidators[staker];

    for (uint256 i = 0; i < userAssociatedValidators.length; i++) {
        uint16 validatorId = userAssociatedValidators[i];
        if ($.userValidatorStakes[staker][validatorId].staked == 0) {
            removeStakerFromValidator($, staker, validatorId);
        }
    }
}
```

In total there are multiple nested/serial loops iterating over all user-associated validators (including slashed ones). As the number of validators per user grows, these loops — combined with state updates — can cause unbounded gas consumption and make `claimAll` revert.

</details>

## Impact Details

claimAll may revert due to excessive gas consumption when a user has many associated validators (including slashed ones that remain in `$.userValidators` and/or `validatorExists`).

## Proof of Concept

{% stepper %}
{% step %}

### Step

User stakes with many validators over time (e.g., 50 different validators). Each `stake()` call adds the validator to `$.userValidators[user]`.
{% endstep %}

{% step %}

### Step

Many validators get slashed over time (e.g., 30 of them). `_performSlash()` clears `$.validatorStakers[validatorId]` but leaves `$.userValidators[user]` unchanged.
{% endstep %}

{% step %}

### Step

User calls `claimAll()`. The call iterates over all tokens and, for each token, iterates over all 50 validators (including the 30 slashed). Additional loops run to clear pending flags and to remove staker-relationships.
{% endstep %}

{% step %}

### Step

Combined iterations and state updates exceed block gas limits and `claimAll()` reverts with an out-of-gas error, making the function effectively unusable for that user.
{% endstep %}
{% endstepper %}

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/RewardsFacet.sol#L359-L387>
