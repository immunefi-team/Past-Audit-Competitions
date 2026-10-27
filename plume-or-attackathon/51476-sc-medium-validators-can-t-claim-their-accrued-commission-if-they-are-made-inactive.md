# 51476 sc medium validators can t claim their accrued commission if they are made inactive

**Submitted on Aug 3rd 2025 at 07:41:23 UTC by @WinSec for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51476
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief/Intro

Validators lose their accrued commission if they are made inactive.

### Vulnerability Details

`requestCommissionClaim` has a check:

```solidity
        if (!validator.active || validator.slashed) {
            revert ValidatorInactive(validatorId);
        }
```

Which means if the validator is not active then they can't claim commission. This is contradictory to the rules that apply to users:

when users call `claim` it calls `_validateValidatorForClaim`:

```solidity
    function _validateValidatorForClaim(
        uint16 validatorId
    ) internal view {//@audit-ok only cheks if the
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();

        if (!$.validatorExists[validatorId]) {
            revert ValidatorDoesNotExist(validatorId);
        }
        // Allow claims from slashed validators - users should be able to claim preserved rewards
        // Only reject if validator doesn't exist
    }
```

Users are allowed to claim from slashed validators as well as inactive validators, as long as the validator exists. But validators who have accrued commission and aren't malicious and not slashed — if they are made inactive — cannot claim their commission and thus lose it unless reactivated.

### Impact Details

Validators lose their accrued commission.

### References

* <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol#L507>

## Proof of Concept

{% stepper %}
{% step %}

### Step

Validator is active and earning commission on stakes for multiple reward periods.
{% endstep %}

{% step %}

### Step

Validator accumulates significant commission (e.g., 1000 USDC) over time but hasn't claimed yet.
{% endstep %}

{% step %}

### Step

Admin marks validator as inactive for legitimate reasons (maintenance, compliance, protocol upgrade).
{% endstep %}

{% step %}

### Step

Validator attempts to claim their rightfully earned commission via `requestCommissionClaim`.
{% endstep %}

{% step %}

### Step

Transaction reverts with `ValidatorInactive(validatorId)` despite commission being earned while active.
{% endstep %}

{% step %}

### Step

Users can still claim rewards from the same inactive validator without any restrictions.
{% endstep %}

{% step %}

### Step

Validator's earned commission becomes permanently inaccessible unless reactivated.
{% endstep %}
{% endstepper %}
