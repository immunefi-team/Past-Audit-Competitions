# 52436 sc low getaccruedcommission could return an inaccurate value

**Submitted on Aug 10th 2025 at 17:06:50 UTC by @a16 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52436
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Temporary freezing of funds for at least 1 hour

## Description

### Brief / Intro

The `getAccruedCommission()` function is supposed to return the amount of commission accrued by a validator for a single token, but it does not account for changes that occurred since the last on-chain update.

### Vulnerability Details

Commission accrues over time as a percentage of rewards. The public function `getAccruedCommission()` returns `validatorAccruedCommission[validatorId][token]`, but the true accrued commission (as computed in `requestCommissionClaim()`) may be larger because `requestCommissionClaim()` updates `validatorAccruedCommission[validatorId][token]` by calling `_settleCommissionForValidatorUpToNow()`, which internally calls `updateRewardPerTokenForValidator()` and increments `validatorAccruedCommission[validatorId][token]`. If time has passed since the last update, the value returned by `getAccruedCommission()` can be considerably smaller than the actual amount that would be settled on-chain.

### Impact Details

Validators calling this public function might get an incorrect impression of the amount of commission accrued so far, leading them to delay claiming rewards or otherwise misjudge available commission.

## Proof of Concept

{% stepper %}
{% step %}

### Observer checks accrued commission

Validator 1 calls `getAccruedCommission()` which returns a relatively low value. Validator 1 decides to wait another day before checking again.
{% endstep %}

{% step %}

### Value appears unchanged

Validator 1 calls `getAccruedCommission()` again and still sees the same low value, because no on-chain update has run to settle accrued commission.
{% endstep %}

{% step %}

### Claim reveals higher settled amount

Validator 1 calls `requestCommissionClaim()` and observes that the amount actually received is larger than what `getAccruedCommission()` previously reported, since `requestCommissionClaim()` triggers settlement logic that updates the accrued commission up to now.
{% endstep %}
{% endstepper %}
