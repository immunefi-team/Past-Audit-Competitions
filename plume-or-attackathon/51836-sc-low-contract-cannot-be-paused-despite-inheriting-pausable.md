# 51836 sc low contract cannot be paused despite inheriting pausable

**Submitted on Aug 6th 2025 at 05:17:17 UTC by @Finlooz4 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network)

* **Report ID:** #51836
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol>
* **Impacts:**
  * Missing implementation of expected functionality

## Description

### Brief/Intro

The `TellerWithMultiAssetSupportPredicateProxy` contract inherits from OpenZeppelin's `Pausable` but does not implement any functions to actually pause or unpause the contract. As a result, although the contract performs pause checks before allowing deposits, there is no mechanism to change the paused state. This renders the pausable feature ineffective and prevents the contract from being halted in emergencies or during maintenance.

### Vulnerability Details

OpenZeppelin's `Pausable` provides internal functions `_pause()` and `_unpause()` to manage the paused state. These functions are internal and must be invoked by the inheriting contract (typically via public/external functions protected by an access control modifier). In this contract, no public or external functions call `_pause()` or `_unpause()`, so the paused state cannot be toggled by any actor, including the contract owner.

Examples from the contract:

```solidity
// In deposit:
if (paused()) {
    revert TellerWithMultiAssetSupportPredicateProxy__Paused();
}

// In depositAndBridge:
if (paused()) {
    revert TellerWithMultiAssetSupportPredicateProxy__Paused();
}
```

The presence of these checks indicates the contract was intended to be pausable, but without functions to set the paused state, these checks are effectively inert. The default state is unpaused, so `paused()` will always return false unless `_pause()` is called — which cannot happen given the current implementation.

## Impact Details

Because the contract cannot be paused:

* In the event of a security vulnerability or critical bug, the contract cannot be halted to stop further interactions, allowing an exploit to continue unchecked.
* Maintenance or upgrades that require halting operations cannot be performed safely, risking inconsistencies or errors during those procedures.
* Overall, the inability to pause reduces the protocol's reactive and defensive capabilities during incidents.

## References

```solidity
// Inherits Pausable but NO functions to change pause state
contract TellerWithMultiAssetSupportPredicateProxy is ... Pausable {...}

// Pause checks that always pass (since contract can't be paused)
if (paused()) { // Always false
    revert TellerWithMultiAssetSupportPredicateProxy__Paused();
}
```

## Proof of Concept

{% stepper %}
{% step %}

### Describe the exploit scenario

Vulnerability is present in `deposit()` (and similar functions). An attacker weaponizes this logic to drain the teller vault.
{% endstep %}

{% step %}

### Admin attempts mitigation

An admin attempts to pause the contract to limit the attack scope. The contract cannot be paused due to missing pause/unpause implementations.
{% endstep %}

{% step %}

### Consequence

Attacker continues to execute the attack and drain funds while the contract remains unpausable.
{% endstep %}
{% endstepper %}
