# 51712 sc insight yield distribution will revert if global module doesn t implement iyieldrestrictions

* **Submitted on:** Aug 5th 2025 at 08:05:57 UTC by @WinSec for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)
* **Report ID:** #51712
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:** Theft of unclaimed yield

## Description

### Brief/Intro

If the global module set in the router does not implement IYieldRestrictions (or the call fails), the developer attempted to handle that case with a try/catch. However, due to how Solidity handles function selector mismatches, the call can revert before reaching the catch block. As a result, `distributeYield` can revert entirely, preventing yield distribution for all holders.

### Vulnerability Details

`distributeYield` distributes yield to ArcToken holders while skipping restricted accounts. It relies on `_isYieldAllowed` at multiple points in `distributeYield`. Example loop:

```solidity
for (uint256 i = 0; i < holderCount; i++) {
    address holder = $.holders.at(i);
    if (_isYieldAllowed(holder)) {
        effectiveTotalSupply += balanceOf(holder);
    }
}
```

The `_isYieldAllowed` implementation checks both a `specificYieldModule` and a `globalYieldModule`. Focusing on the global module:

```solidity
address globalYieldModule = IRestrictionsRouter(routerAddr).getGlobalModuleAddress(RestrictionTypes.GLOBAL_SANCTIONS_TYPE);
if (globalYieldModule != address(0)) {
    try IYieldRestrictions(globalYieldModule).isYieldAllowed(account) returns (bool globalAllowed) {
        allowed = allowed && globalAllowed;
    } catch {
        // If global module doesn't implement IYieldRestrictions or call fails, treat as restricted?
        // Or handle based on specific global module design.
        // Current: Assume allowed if call fails/not implemented (less restrictive).
    }
}
```

Comments indicate the developer intended to treat the account as allowed if the call fails or is not implemented. However, if the contract set as `globalYieldModule` does not implement `isYieldAllowed`, the external call can revert due to a function selector mismatch, and Solidity's try/catch will not catch that — causing the entire `distributeYield` transaction to revert.

This is analogous to the issue described here: <https://github.com/sherlock-audit/2024-04-teller-finance-judging/issues/178>

### Impact Details

Yield will not be distributed to any holders while the misconfigured global module remains set, causing potential loss/theft of unclaimed yield or denial of yield distribution.

## Proof of Concept

{% stepper %}
{% step %}

### Step 1: Admin sets a wrong contract as global yield module

Call:

```
router.updateGlobalModuleImplementation(GLOBAL_SANCTIONS_TYPE, wrongContract)
```

Where `wrongContract` does not implement `isYieldAllowed()`.
{% endstep %}

{% step %}

### Step 2: Call distributeYield()

`distributeYield()` is invoked to distribute yield to token holders. Execution iterates through holders:

```solidity
for (uint256 i = 0; i < holderCount; i++) {
    address holder = $.holders.at(i);
    if (_isYieldAllowed(holder)) {
        effectiveTotalSupply += balanceOf(holder);
    }
}
```

{% endstep %}

{% step %}

### Step 3: First call to \_isYieldAllowed reaches the global module check

Attempted call:

```solidity
address globalYieldModule = IRestrictionsRouter(routerAddr).getGlobalModuleAddress(RestrictionTypes.GLOBAL_SANCTIONS_TYPE);
if (globalYieldModule != address(0)) {
    try IYieldRestrictions(globalYieldModule).isYieldAllowed(account) returns (bool globalAllowed) {
        allowed = allowed && globalAllowed;
    } catch {
        // Intended fallback if call fails/not implemented
    }
}
```

Because `wrongContract` lacks the `isYieldAllowed` selector, the call cannot be dispatched.
{% endstep %}

{% step %}

### Step 4: Call fails with function-not-found error that bypasses try/catch

Solidity's try/catch does not handle function selector mismatches in this context — the external call reverts before the catch block runs.
{% endstep %}

{% step %}

### Step 5: Entire distributeYield transaction reverts

The revert stops the whole `distributeYield` execution and no yield is distributed to any holders. Distribution attempts will continue to fail until the global module is fixed or removed.
{% endstep %}
{% endstepper %}

## References

* Related issue: <https://github.com/sherlock-audit/2024-04-teller-finance-judging/issues/178>
