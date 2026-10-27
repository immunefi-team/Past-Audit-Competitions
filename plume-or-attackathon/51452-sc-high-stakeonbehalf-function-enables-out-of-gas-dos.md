# 51452 sc high stakeonbehalf function enables out of gas dos

**Submitted on Aug 2nd 2025 at 23:43:13 UTC by @KlosMitSoss for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51452
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:**
  * Unbounded gas consumption

## Description

### Brief/Intro

Staking on behalf of another user can be exploited by an attacker who adds enough items to the `userValidators` mapping of any user to cause the gas consumption of function calls to exceed the block gas limit. This results in a denial of service for several functions, preventing them from being called.

### Vulnerability Details

When `StakingFacet::stakeOnBehalf()` is called, which can be done by anyone to stake PLUME to a specific validator on behalf of another user, `_performStakeSetup()` is called within this function:

```solidity
function stakeOnBehalf(uint16 validatorId, address staker) external payable returns (uint256) {
    ... ...
    // Perform all common staking setup for the beneficiary
>>  bool isNewStake = _performStakeSetup(staker, validatorId, stakeAmount);
    ... ...
}
```

This function calls `PlumeValidatorLogic::addStakerToValidator()`, which then pushes the `validatorId` onto the `userValidator` mapping:

```solidity
function addStakerToValidator(PlumeStakingStorage.Layout storage $, address staker, uint16 validatorId) internal {
    ... ...
    if (!$.userHasStakedWithValidator[staker][validatorId]) {
>>  $.userValidators[staker].push(validatorId);
        $.userHasStakedWithValidator[staker][validatorId] = true;
    }
    ... ...
}
```

This can be done for any existing `validatorId`, which means the mapping can be flooded with entries. As a result, when looping over the mapping, it could cause a revert due to the transaction consuming more gas than the block gas limit. This could happen, for example, in `StakingFacet::_calculateAndClaimAllRewardsWithCleanup()`, which is called within `StakingFacet::restakeRewards()`:

```solidity
function _calculateAndClaimAllRewardsWithCleanup(
    address user,
    address targetToken
) internal returns (uint256 totalRewards) {
    ... ...
    // Make a copy to avoid iteration issues
>>  uint16[] memory currentUserValidators = $.userValidators[user];

    // Track validators that might need cleanup after claiming
    uint16[] memory validatorsToCheck = new uint16[](currentUserValidators.length);
    uint256 checkCount = 0;

>>  for (uint256 i = 0; i < currentUserValidators.length; i++) {
        uint16 userValidatorId = currentUserValidators[i];

        uint256 existingRewards = $.userRewards[user][userValidatorId][targetToken];
        uint256 rewardDelta =
            IRewardsGetter(address(this)).getPendingRewardForValidator(user, userValidatorId, targetToken);
        uint256 totalValidatorReward = existingRewards + rewardDelta;

        if (totalValidatorReward > 0) {
            totalRewards += totalValidatorReward;
            PlumeRewardLogic.updateRewardsForValidator($, user, userValidatorId);
            $.userRewards[user][userValidatorId][targetToken] = 0;

            if ($.totalClaimableByToken[targetToken] >= totalValidatorReward) {
                $.totalClaimableByToken[targetToken] -= totalValidatorReward;
            } else {
                $.totalClaimableByToken[targetToken] = 0;
            }

            emit RewardClaimedFromValidator(user, targetToken, userValidatorId, totalValidatorReward);

            // Clear pending rewards flag for this validator and track for cleanup
            PlumeRewardLogic.clearPendingRewardsFlagIfEmpty($, user, userValidatorId);

            // Track this validator for potential relationship cleanup
            validatorsToCheck[checkCount] = userValidatorId;
            checkCount++;
        }
    }
    ... ...
}
```

### Impact Details

This issue causes a denial of service for several functions, including `StakingFacet::restakeRewards()` and `StakingFacet::withdraw()`.

## References

Code references are provided throughout the report (links above).

## Proof of Concept

{% stepper %}
{% step %}

### Stake on behalf to populate user's validators

An attacker calls `StakingFacet::stakeOnBehalf()` to stake PLUME to a specific validator on behalf of Alice. Within this function, `_performStakeSetup()` is called.

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/StakingFacet.sol#L436>
{% endstep %}

{% step %}

### addStakerToValidator pushes validatorId for the beneficiary

`_performStakeSetup()` calls `PlumeValidatorLogic::addStakerToValidator()` which pushes the `validatorId` onto the `userValidators` mapping for Alice.

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/StakingFacet.sol#L224> <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeValidatorLogic.sol#L60>
{% endstep %}

{% step %}

### Looping over flooded mapping causes out-of-gas

When `StakingFacet::restakeRewards()` is called, it invokes `_calculateAndClaimAllRewardsWithCleanup()`, which loops over `$.userValidators[user]`. If that array is large enough (flooded by the attacker), the loop can consume more gas than the block gas limit and revert, causing a denial of service.

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/StakingFacet.sol#L961-L995>
{% endstep %}
{% endstepper %}
