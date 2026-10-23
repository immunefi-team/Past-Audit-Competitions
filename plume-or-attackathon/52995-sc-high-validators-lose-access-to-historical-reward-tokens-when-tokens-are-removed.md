# 52995 sc high validators lose access to historical reward tokens when tokens are removed

**Submitted on Aug 14th 2025 at 15:24:43 UTC by @silver\_eth for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network)

* **Report ID:** #52995
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/RewardsFacet.sol>
* **Impacts:** Temporary freezing of funds for at least 24 hours

## Description

### Brief/Intro

When a reward token is removed, while stakers still have access to the reward token up to the time it was removed, validators lose rights to the reward token until it is added again.

### Vulnerability Details

When a reward token is removed from the active set, stakers are still able to withdraw accrued rewards for that token up to the time of removal. However, validators lose the ability to claim their own accrued commission for that token until the token is re-added.

The issue stems from differences in how reward claims are implemented for stakers versus validators:

* Stakers: The withdrawal logic for stakers allows them to withdraw rewards for tokens that were previously active, even if the token has since been removed from the active set.
* Validators: The validator claim logic only permits requesting rewards for currently active reward tokens. Once a token is removed, a validator cannot initiate a claim for that token.

### Impact Details

{% hint style="warning" %}
Validators are effectively DoSed from withdrawing accrued rewards for removed tokens.\
This denial can last indefinitely if the token is never re-added.

If the intention is to block all withdrawals of removed tokens, that intention is not achieved because stakers can still withdraw.
{% endhint %}

## Proof of Concept

{% stepper %}
{% step %}

### Step

Validator V1 earns rewards in token T1 while T1 is active.
{% endstep %}

{% step %}

### Step

Token T1 is removed from the active reward token list.
{% endstep %}

{% step %}

### Step

* Delegators to V1 can still withdraw their share of T1 rewards earned before removal.
* Validator V1 cannot request T1 rewards, because validator claim logic restricts requests to active tokens only.

Result: V1 is unable to claim T1 until/unless T1 is re-added as a reward token.
{% endstep %}
{% endstepper %}

## References

<details>

<summary>Source links</summary>

* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L508>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L126-L133>

</details>
