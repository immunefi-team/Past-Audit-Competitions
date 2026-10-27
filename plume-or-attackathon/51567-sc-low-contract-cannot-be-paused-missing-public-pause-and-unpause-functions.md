# 51567 sc low contract cannot be paused missing public pause and unpause functions

**Submitted on Aug 4th 2025 at 04:22:59 UTC by @MMophule for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51567
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol>
* **Impacts:**
  * Unable to Pause Contract

## Description

### Brief/Intro

The contract inherits from the OpenZeppelin `Pausable` contract, but does not implement the required public `pause()` and `unpause()` functions. OpenZeppelin's `Pausable` only provides internal `_pause()` and `_unpause()` functions, so the inheriting contract must expose public functions (usually access-controlled) that call those internals.

### Vulnerability Details

Because there are no public `pause()` and `unpause()` functions implemented, the contract cannot be paused via the standard mechanisms provided by OpenZeppelin's `Pausable`.

### Impact Details

During potential hacks or black swan events, being unable to pause the contract can prevent administrators from mitigating further harm. This was assessed as medium in impact.

## Proof of Concept

The contract and any subcontracts inheriting from it do not include public functions that call `_pause()` and `_unpause()`, so pause/unpause functionality is effectively unavailable.

## Suggested Fix (implicit)

Ensure the contract exposes appropriately access-controlled public functions that call `_pause()` and `_unpause()` (for example, `pause()` and `unpause()` guarded by an admin/owner role).
