# 50691 sc insight no validator limit can lead to dos

**Submitted on Jul 27th 2025 at 15:23:05 UTC by @PotEater for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50691
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/RewardsFacet.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

In the `RewardsFacet.sol` contract, in the function `claimAll`, there is a for loop which iterates through a list of validatorIds (in the internal function \_processAllValidatorRewards), this list has no limit and if grows too large, it will cause a Denial of Service and admin will have to remove validators.

## Vulnerability Details

The function iterates through `validatorIds` array, no limit is enforced to this array, this means there may be that much validators so the function runs to OOG (out of gas) error.

Code snippet:

{% code title="RewardsFacet.sol (excerpt)" %}

```solidity
    function claimAll() external nonReentrant returns (uint256[] memory) {
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
        address[] memory tokens = $.rewardTokens;
        uint256[] memory claims = new uint256[](tokens.length);

        // Process each token
        for (uint256 i = 0; i < tokens.length; i++) { // @audit-issue exponential looping
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

{% endcode %}

What makes this issue even more probable is the fact that the `claimAll` function iterates through `token.length` and on each iteration calls internal function `_processAllValidatorRewards` which creates an exponential looping issue and is even more vulnerable to the out of gas error when too many validators.

Code snippet:

{% code title="RewardsFacet.sol (excerpt)" %}

```solidity
    function _processAllValidatorRewards(address user, address token) internal returns (uint256 totalReward) {
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();

        uint16[] memory validatorIds = $.userValidators[user];

        for (uint256 i = 0; i < validatorIds.length; i++) {
            uint16 validatorId = validatorIds[i];

            // The underlying reward processing logic correctly handles all validator states
            // (active, inactive, slashed) by respecting the relevant timestamps

            uint256 rewardFromValidator = _processValidatorRewards(user, validatorId, token);
            totalReward += rewardFromValidator;
        }

        return totalReward;
    }
```

{% endcode %}

So the function becomes a gas BOMB, because there is exponential looping together with unlimited validators possible.

## Impact Details

{% hint style="warning" %}
Temporary Denial of Service: the contract can run out of gas when `claimAll` is called with many reward tokens and the user has many validators, requiring admin intervention to remove validators so claims can succeed.
{% endhint %}

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/RewardsFacet.sol#L542>

## Proof of Concept

{% stepper %}
{% step %}

### Reproduction requirements

Too many validators present for a user.
{% endstep %}

{% step %}

### Steps

* Call `claimAll`.
* Assume there are 10 reward tokens; `claimAll` iterates 10 times (tokens.length).
* Each iteration calls `_processAllValidatorRewards`, which iterates over the user's validators.
  {% endstep %}

{% step %}

### Outcome

If there are many validators (e.g., 10 validators and 10 tokens → 100 iterations, or more in real scenarios), the transaction can run out of gas and revert, causing a temporary denial of service for claims.
{% endstep %}
{% endstepper %}
