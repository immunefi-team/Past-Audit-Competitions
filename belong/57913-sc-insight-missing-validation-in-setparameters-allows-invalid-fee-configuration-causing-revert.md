# 57913 sc insight missing validation in setparameters allows invalid fee configuration causing reverts in paytovenue&#x20;

**Submitted on Oct 29th 2025 at 12:43:55 UTC by @DoD4uFN for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57913
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>

## Description

### Brief/Intro

The `setParameters` function in `BelongCheckIn.sol` allows the contract owner to configure fee parameters without any validation. This includes `_fees.platformSubsidyPercentage` and `_fees.processingFeePercentage`, which are later assumed to maintain a specific relationship. If misconfigured such that `processingFeePercentage > platformSubsidyPercentage`, any call to `payToVenue` using LONG tokens will revert, causing a denial of service for LONG-based payments.

## Vulnerability Details

The payment flow in `payToVenue` depends on the assumption that the platform subsidy percentage is greater than or equal to the processing fee percentage. This invariant is not enforced anywhere, including the owner-only configuration function `setParameters`.

When a customer pays using LONG tokens, the code calculates:

```solidity
uint256 subsidyMinusFees =
    _storage.fees.platformSubsidyPercentage.calculateRate(customerInfo.amount)
    - _storage.fees.processingFeePercentage.calculateRate(customerInfo.amount);
```

If `_fees.processingFeePercentage` > `_fees.platformSubsidyPercentage`, the subtraction will underflow (or revert depending on compiler/version), causing `payToVenue` to revert. This disables all LONG-token payments.

There is no validation ensuring:

```
platformSubsidyPercentage >= processingFeePercentage
```

leaving the system vulnerable to misconfiguration.

## Impact Details

If `processingFeePercentage` is set higher than `platformSubsidyPercentage`, every call to `payToVenue` with LONG tokens will revert. No direct fund loss is described, but the vulnerability fully breaks a core payment pathway.

## References

* `BelongCheckIn.sol` - setParameters\
  <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol#L344-L350>
* `BelongCheckIn.sol` - \_setParameters\
  <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol#L597-L612>
* `BelongCheckIn.sol` - payToVenue - platform subsidy calculations\
  <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol#L472-L475>

## Proof of Concept

{% stepper %}
{% step %}

### Step 1 — Owner sets invalid fees

Owner calls `setParameters` with an invalid configuration:

```solidity
Fees memory invalidFees = Fees({
    referralCreditsAmount: 3,
    affiliatePercentage: 1000,
    longCustomerDiscountPercentage: 3000,
    platformSubsidyPercentage: 300,
    processingFeePercentage: 301, // invalid: greater than platformSubsidyPercentage
    buybackBurnPercentage: 5000
});
```

{% endstep %}

{% step %}

### Step 2 — Customer attempts LONG token payment

A customer calls `payToVenue()` with any valid `CustomerInfo` indicating a LONG token payment.
{% endstep %}

{% step %}

### Step 3 — Transaction reverts

The transaction reverts due to the subtraction:

```solidity
uint256 subsidyMinusFees = 
    _storage.fees.platformSubsidyPercentage.calculateRate(customerInfo.amount) -
    _storage.fees.processingFeePercentage.calculateRate(customerInfo.amount);
```

Because `processingFeePercentage > platformSubsidyPercentage`, this results in an underflow/revert and breaks the payment flow.
{% endstep %}
{% endstepper %}
