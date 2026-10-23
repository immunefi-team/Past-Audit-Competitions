# 52719 sc medium inactive validators blocked from claiming commissions despite passed timelock

* Report ID: #52719
* Report Type: Smart Contract
* Report severity: Medium
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* Impacts:
  * Theft of unclaimed yield

## Brief / Intro

When a commission claim for a validator and token is requested, a timelock is started, after which the request can be finalized. However, both requesting and finalizing are disallowed for inactive validators, even if the timelock had already passed before the validator became inactive. This restriction is stricter than for slashed validators, which is inconsistent and does not make sense.

## Vulnerability Details

Validators can be either deactivated or slashed. Both events set the `slashedAtTimestamp` to the current `block.timestamp`. However, slashing a validator is far more severe: it means that all other validators voted to slash it due to malicious behavior. By contrast, deactivation simply means the validator is no longer active, without implying any wrongdoing.

Therefore, it is illogical to completely block commission claims for deactivated validators while still allowing certain commission claims for slashed validators.

Relevant code excerpts:

When a validator is slashed, the commission claim can still be finalized if the timelock had already passed before the validator was slashed:

```solidity
        if (validator.slashed && readyTimestamp >= validator.slashedAtTimestamp) {
            revert ValidatorInactive(validatorId);
        }
```

In contrast, deactivated validators cannot finalize their requests at all, regardless of whether the timelock had already passed:

```solidity
        // For a non-slashed validator, simply require it to be active to finalize a claim.
        if (!validator.slashed && !validator.active) {
            revert ValidatorInactive(validatorId);
        }
```

This creates an inconsistent and stricter rule for deactivated (inactive) validators compared to slashed validators.

## Impact Details

{% hint style="warning" %}
Inactive validators are permanently unable to claim their commissions, even if the timelock had already passed while they were active. This restriction is stricter than the rules for slashed validators, which is inconsistent and unfair. This can lead to loss/theft of unclaimed yield.
{% endhint %}

## References

Code references are provided throughout the report (links retained as-is):

* ValidatorFacet.sol (target): <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* PlumeRewardLogic.sol lines referenced in PoC: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L189-L191>
* Request commission claim: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L500-L539>
* Request timestamp set: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L531>
* Requesting blocked for slashed or inactive: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L513-L515>
* setValidatorStatus (deactivation): <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L252-L308>
* Deactivation sets slashedAtTimestamp: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L277>
* finalizeCommissionClaim checks: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L566-L575>

## Proof of Concept

{% stepper %}
{% step %}

### Step

Validator A accrues commissions.

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L189-L191>
{% endstep %}

{% step %}

### Step

The admin of validator A requests a commission claim by calling ValidatorFacet::requestCommissionClaim().

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L500-L539>

This sets the `requestTimestamp` to the current `block.timestamp`: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L531>

Note: Requesting a claim is not possible for slashed or inactive validators: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L513-L515>
{% endstep %}

{% step %}

### Step

After the timelock has passed, the validator is deactivated by calling ValidatorFacet::setValidatorStatus().

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L252-L308>

Deactivation sets the `slashedAtTimestamp` to the current `block.timestamp`: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L277>
{% endstep %}

{% step %}

### Step

When the validator admin tries to finalize the commission claim by calling ValidatorFacet::finalizeCommissionClaim(), the call reverts even though the timelock had already passed while the validator was still active.

Reference (finalize checks): <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L566-L575>

If the validator had been slashed instead, the call would not revert: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L566-L570>

This demonstrates the inconsistency: deactivated validators cannot finalize even if the timelock expired before deactivation, while slashed validators may finalize depending on timestamps.
{% endstep %}
{% endstepper %}
