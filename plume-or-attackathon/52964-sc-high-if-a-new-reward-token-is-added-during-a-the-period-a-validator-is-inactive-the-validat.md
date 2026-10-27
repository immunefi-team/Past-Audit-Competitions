# 52964 sc high if a new reward token is added during a the period a validator is inactive the validator will still earn rewards commission for some of the duration in which they were inactive&#x20;

**Submitted on Aug 14th 2025 at 14:05:18 UTC by @silver\_eth for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52964
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/RewardsFacet.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief / Intro

When a new reward token is added while a validator is inactive, the system sets checkpoints for that validator even though it was inactive. Because the validator is made active before settling past rewards when re-activating, the validator (and its stakers) can end up earning rewards and commission for some or all of the duration in which they were inactive.

## Vulnerability Details

{% stepper %}
{% step %}

### Root cause 1

When a new reward token is added, a checkpoint is pushed for every validator, including inactive ones.
{% endstep %}

{% step %}

### Root cause 2

When a validator is made active, the validator status is set active before settling rewards since the last update. As a consequence, the rewards logic treats the validator as having been previously active and accrues commission for the new reward token dating back to the last update.
{% endstep %}

{% step %}

### Resulting effect

* Validator accrues commission for periods when it was actually inactive.
* Validator stakers can claim rewards for the newly added token because *calculateRewardsCore* checks will pass once the validator is active again (e.g., slashtimestamp == 0).
  {% endstep %}
  {% endstepper %}

## Impact Details

* Validators and their stakers gain access to reward tokens that were added while the validator was inactive, enabling theft of unclaimed yield.

## References

* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L289-L305>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L155>

## Proof of Concept

{% stepper %}
{% step %}

### Step: Disable validator

Admin makes a validator inactive.
{% endstep %}

{% step %}

### Step: Add reward token

Admin adds a new reward token while the validator remains inactive. The code pushes a checkpoint for the validator for that reward token.
{% endstep %}

{% step %}

### Step: Re-activate validator

Admin re-activates the validator. During re-activation:

* The validator is set active first.
* Then the system settles/accumulates previous commission for the duration since the validator was last updated.

Because of the earlier checkpointing, this causes the validator to (improperly) accrue commission for the new reward token for the period it was inactive.
{% endstep %}

{% step %}

### Step: Stakers claim

Validator stakers can also claim rewards for the newly added token because *calculateRewardsCore* checks will pass after the validator is active (e.g., slashtimestamp == 0), letting them claim rewards dating before the token was added.
{% endstep %}
{% endstepper %}
