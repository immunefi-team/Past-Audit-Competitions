# 49726 sc insight there is a redundant zero address check in the validatorfacet sol that is obsolete and could never be true

**Submitted on Jul 18th 2025 at 18:41:02 UTC by @avoloder for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #49726
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impacts:** Confusion and reduced maintainability

## Description

### Brief/Intro

There is an if branch inside `setValidatorAddresses()` function that can never be true and can lead to confusion.

### Vulnerability Details

The following code snippet contains an unreachable branch due to a prior condition:

```solidity
// Update L2 Withdraw Address if provided and different
if (newL2WithdrawAddress != address(0) && newL2WithdrawAddress != validator.l2WithdrawAddress) {
    if (newL2WithdrawAddress == address(0)) {
        // Add specific check for zero address
        revert ZeroAddress("newL2WithdrawAddress");
    }
    validator.l2WithdrawAddress = newL2WithdrawAddress;
}
```

The outer `if` requires `newL2WithdrawAddress != address(0)` to be true. The inner `if (newL2WithdrawAddress == address(0))` can therefore never be true, making the inner check redundant and potentially confusing.

### Impact Details

The main impact is confusion and reduced maintainability. The redundant check signals that the code may not have been carefully reviewed or updated, increasing cognitive load for future developers and potentially leading to misunderstanding of requirements.

## References

* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L365-L436>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L397-L399>

{% hint style="warning" %}
Consider removing the unreachable inner check, or reworking the surrounding logic if the intent was to explicitly guard against zero addresses in all entry paths. This will improve clarity and maintainability.
{% endhint %}

<details>

<summary>Proof of Concept</summary>

A validator admin tries to update validator addresses and wants to keep the existing `l2WithdrawAddress`. They pass `address(0)` as the parameter `newL2WithdrawAddress`. Because the outer `if` prevents entering the branch when `newL2WithdrawAddress == address(0)`, the inner `revert ZeroAddress("newL2WithdrawAddress")` is not reachable — demonstrating the redundancy.

</details>
