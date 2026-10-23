# 49915 sc low misleading event emission in createwhitelistrestrictions function in restrictionsfactory contract

**Submitted on Jul 20th 2025 at 14:25:54 UTC by @AasifUsmani for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #49915
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/restrictions/RestrictionsFactory.sol>
* **Impacts:** Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The `createWhitelistRestrictions` function in `RestrictionsFactory.sol` emits misleading event data when the actual admin of the created restrictions module differs from the function caller, leading to incorrect off-chain tracking and potential confusion about module ownership.

### Vulnerability Details

**Location**\
`RestrictionsFactory.sol - createWhitelistRestrictions() function (lines 67-91)`

**Root Cause**

The function always emits `msg.sender` as the "owner" in the `RestrictionsCreated` event, regardless of who actually receives admin privileges in the deployed restrictions module:

```solidity
function createWhitelistRestrictions(address admin) external returns (address) {
    // Actual admin logic: use provided admin or fallback to msg.sender
    bytes memory initData = abi.encodeWithSelector(
        WhitelistRestrictions.initialize.selector, 
        admin != address(0) ? admin : msg.sender
    );
    
    address proxy = _deployProxy(address(implementation), initData);
    
    // Always emits msg.sender as owner, not the actual admin!
    emit RestrictionsCreated(proxy, msg.sender, address(implementation), "Whitelist");
}
```

**The Problem**

The event field labeled as "owner" doesn't represent the actual admin/owner of the restrictions module, creating a disconnect between emitted data and contract reality.

## Impact Details

**Misleading Off-Chain Data**\
Off-chain systems, indexers, and user interfaces will incorrectly display module ownership information:

```solidity
// Scenario: Token owner delegates deployment to avoid gas fees
// Alice (token owner) asks Bob (deployer) to create restrictions with Alice as admin

createWhitelistRestrictions(alice); // Bob calls this function

// Reality: Alice becomes admin of the restrictions module
// Event: RestrictionsCreated(proxy, bob, implementation, "Whitelist")
// Problem: Event shows Bob as "owner" but Alice has actual control
```

**Data Integrity Issues**

1. Incorrect Analytics: Dashboards will show wrong creator/owner statistics
2. User Confusion: Users checking events will see incorrect ownership information
3. Audit Trail Problems: Security monitoring systems will track wrong addresses as module controllers

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/restrictions/RestrictionsFactory.sol#L87>

## Proof of Concept

{% stepper %}
{% step %}

### Step

Call `createWhitelistRestrictions` with different `msg.sender` and `admin` parameters.
{% endstep %}

{% step %}

### Step

Observe the emitted event: it will show `msg.sender` as the "owner" instead of the `admin` address that actually received control.
{% endstep %}
{% endstepper %}
