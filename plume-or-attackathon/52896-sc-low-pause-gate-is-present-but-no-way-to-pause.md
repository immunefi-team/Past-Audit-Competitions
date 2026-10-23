# 52896 sc low pause gate is present but no way to pause

**Submitted on Aug 14th 2025 at 06:36:51 UTC by @hulkvision for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network)

* **Report ID:** #52896
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds
  * “Pause” gate is present but no way to pause (operational risk / incident response)

## Description

### Brief/Intro

The contract owner is unable to activate the emergency pause mechanism, rendering the contract unstoppable during an active exploit or operational failure.

### Vulnerability Details

The `TellerWithMultiAssetSupportPredicateProxy` contract inherits from OpenZeppelin's `Pausable` utility and implements the necessary checks to block functions when paused. However, it critically omits the external `pause()` and `unpause()` functions required for the owner to actually trigger the paused state. This oversight renders the entire emergency stop feature non-functional, meaning if a separate vulnerability were discovered and exploited, the owner would be powerless to halt the contract and prevent further financial losses.

```solidity
// File: TellerWithMultiAssetSupportPredicateProxy.sol

// L78-L80
function deposit(...) ... {
    if (paused()) { // <--- Check is present
        revert TellerWithMultiAssetSupportPredicateProxy__Paused();
    }
    ...
}

// L133-L135
function depositAndBridge(...) ... {
    if (paused()) { // <--- Check is present
        revert TellerWithMultiAssetSupportPredicateProxy__Paused();
    }
    ...
}
```

This demonstrates a clear intent to allow the owner to pause the contract. However, the contract source code is missing the corresponding control functions that would call the internal `_pause()` and `_unpause()` functions from the Pausable contract.

Without these functions, there is no way for the owner or any other party to transition the contract into a "paused" state. The security mechanism is incomplete and therefore inoperable.

{% hint style="danger" %}
Impact: If any of the vulnerabilities in this contract were actively exploited, the owner would have no way to stop the attacker from draining funds or causing further damage through this proxy.
{% endhint %}

## References

* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/0ee676b5715075c26db6706960fd49ab59b587fc/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol#L78-L80>
* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/0ee676b5715075c26db6706960fd49ab59b587fc/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol#L133-L135>

## Proof of Concept

{% stepper %}
{% step %}

### Reproduction step 1

The owner identifies a critical vulnerability being exploited through the `deposit` / `depositAndBridge` functions.
{% endstep %}

{% step %}

### Reproduction step 2

The owner attempts to pause the contract but finds there is no external `pause()`/`unpause()` function available, so the contract cannot be paused.
{% endstep %}
{% endstepper %}
