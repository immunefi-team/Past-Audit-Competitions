# 49647 sc low pausable functions are not exposed

**Submitted on Jul 17th 2025 at 21:27:36 UTC by @rajkaur for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #49647
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol>

{% hint style="info" %}
Brief / Intro

The `TellerWithMultiAssetSupportPredicateProxy` contract inherits OpenZeppelin's `Pausable` contract to support an emergency stop. However, it does not expose external functions to call the internal `_pause()` and `_unpause()` methods, so the owner cannot activate or deactivate the pause state.
{% endhint %}

## Vulnerability Details

The contract correctly checks `paused()` to guard critical functions, indicating the developer intended the contract to be pausable. Example:

```solidity
function deposit(
)
    external
{
    if (paused()) {
        revert TellerWithMultiAssetSupportPredicateProxy__Paused();
    }
}
```

However, OpenZeppelin's `_pause()` and `_unpause()` functions are `internal`. The contract does not provide external/public functions (e.g., `pause()` / `unpause()`) wrapped with appropriate owner access control to call these internal functions. As a result, there is no mechanism for the owner to trigger the pause or unpause behavior.

## Impact Details

If an emergency occurs, the protocol would not be able to pause the contract, defeating the intended emergency-stop protection.

## Proof of Concept

The PoC is the absence of external pause/unpause wrappers combined with usage of `paused()` checks. The snippet above demonstrates the contract relies on `paused()` but provides no way for the owner to change that state.

Suggested remediation (not altering vulnerability text): add owner-restricted external functions that call `_pause()` and `_unpause()`, for example:

```solidity
function pause() external onlyOwner {
    _pause();
}

function unpause() external onlyOwner {
    _unpause();
}
```

(Do not change existing access control semantics; ensure the same owner/role restrictions used across the codebase are applied.)
