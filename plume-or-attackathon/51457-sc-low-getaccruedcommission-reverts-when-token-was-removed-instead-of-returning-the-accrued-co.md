# 51457 sc low getaccruedcommission reverts when token was removed instead of returning the accrued commission

* Report ID: #51457
* Report Type: Smart Contract
* Report severity: Low

Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

{% hint style="warning" %}
Severity: Low — function reverts when it should return a claimable value for a historical reward token.
{% endhint %}

## Description

### Brief/Intro

When a reward token is removed, `ValidatorFacet::getAccruedCommission()` reverts even though the accrued commission can still be claimed.

### Vulnerability Details

Reward tokens can be added and later removed. When adding a reward token, it becomes both a historical reward token and a currently active reward token. When removing a reward token, it remains a historical reward token to allow claiming of rewards for removed reward tokens. This also includes validator commission claims.

However, when a token was added as a reward token, accumulated commission, and was then removed, the `ValidatorFacet::getAccruedCommission()` function reverts even though there is accrued commission that can still be claimed.

### Impact Details

`ValidatorFacet::getAccruedCommission()` reverts when the specified token has been removed, even though the commission can still be claimed.

## References

Code references (kept as-is):

* RewardsFacet::addRewardToken — <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/RewardsFacet.sol#L153-L202>
* RewardsFacet::removeRewardToken — <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c204/plume/src/facets/RewardsFacet.sol#L210-L250>
* ValidatorFacet::getAccruedCommission — <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L957-L959>

## Proof of Concept

{% stepper %}
{% step %}

### Add a reward token

Call `RewardsFacet::addRewardToken()` to add a token as a reward token.

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/RewardsFacet.sol#L153-L202>
{% endstep %}

{% step %}

### Accrue commission

Some time passes and a validator accrues commission for the added reward token.
{% endstep %}

{% step %}

### Remove the reward token

Call `RewardsFacet::removeRewardToken()` to remove the reward token.

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/RewardsFacet.sol#L210-L250>
{% endstep %}

{% step %}

### Observe revert when querying accrued commission

Call `ValidatorFacet::getAccruedCommission()` for that token. The function reverts (see reference below), even though the commission can still be claimed. The function should return the commission for historical tokens as well.

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L957-L959>
{% endstep %}
{% endstepper %}
