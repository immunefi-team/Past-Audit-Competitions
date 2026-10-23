# 51162 sc low missing pause control implementation in tellerwithmultiassetsupportpredicateproxy

**Submitted on Jul 31st 2025 at 17:22:43 UTC by @TeamJosh for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51162
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The `TellerWithMultiAssetSupportPredicateProxy` contract inherits from OpenZeppelin’s `Pausable` contract, intending to provide an emergency stop mechanism for sensitive user-facing operations like `deposit` and `depositAndBridge`.

However, the contract does not implement any mechanism to trigger the internal `_pause()` or `_unpause()` functions, rendering the `paused()` check in these functions ineffective.

### Vulnerability Details

The following guard appears in `deposit()` and `depositAndBridge()`:

```solidity
if (paused()) {
    revert TellerWithMultiAssetSupportPredicateProxy__Paused();
}
```

These checks are intended to prevent interaction when the system is paused. However, there is no public or internal function in the contract that calls `_pause()` or `_unpause()`, which are required to actually change the pause state.

Key missing components:

* No `pause()` or `unpause()` function callable by the owner.
* No emergency pause mechanism in case of malicious activity or discovered vulnerabilities.

As a result, the `paused()` state will always return `false`, meaning the guard is never actually enforced, and emergency control is effectively broken.

## Impact Details

* Inability to pause user-facing entry points (`deposit` and `depositAndBridge`) in case of exploits or bugs.

## Proof of Concept

```solidity
// Try pausing the contract (this function doesn’t exist)

proxy.pause(); // Error: function does not exist

// Confirm that deposit is always enabled despite claiming to be pausable

bool pausedState = proxy.paused(); // always false

proxy.deposit(...); // always succeeds, even in supposed paused state
```

## References

(Add any relevant links to documentation or code)
