# #46534 \[SC-Insight] Missing Validation to Prevent Self-Assignment of Work Address

**Submitted on Jun 1st 2025 at 08:54:07 UTC by @elyas6126 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46534
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/AgentOwnerRegistry.sol>
* **Impacts:**

## Description

## Brief/Intro

The `setWorkAddress` function in the `AgentOwnerRegistry` contract lacks validation to prevent a management address from setting itself as the work address. This breaks the intended separation between management and operational addresses, potentially leading to confusion in the system's address hierarchy and undermining the designed access control patterns.

***

## Issue Details

The `setWorkAddress` function allows agent managers to assign a work address for operational activities separate from their management address. However, the function does not validate that the work address differs from the caller's address (management address). This allows a manager to set their own address as both the management and work address, which defeats the purpose of having separate roles.

**Relevant code snippet:**

```solidity
function setWorkAddress(address _ownerWorkAddress)
    external
{
    require(isWhitelisted(msg.sender), "agent not whitelisted");
    require(_ownerWorkAddress == address(0) || workToMgmtAddress[_ownerWorkAddress] == address(0),
        "work address in use");
    // Missing: require(_ownerWorkAddress != msg.sender, "Work address cannot be management address");
    
    // rest of function logic...
}
```

The absence of this check allows the same address to serve dual roles, which contradicts the system's design of separating management and operational responsibilities.

***

## Impact Details

This missing validation could result in:

* Breakdown of the intended address separation model, where management and work addresses should serve different purposes
* Potential confusion in off-chain systems or UI components that expect distinct management and work addresses
* Undermining of access control patterns that rely on the separation between these two address types
* Possible operational complications when the same address is used for both management decisions and routine operations

While this issue does not directly lead to loss of funds or protocol failure, it compromises the architectural integrity of the agent management system and could cause confusion in system operations.

***

## References

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/AgentOwnerRegistry.sol#L53>

## Proof of Concept

## Proof of Concept

````typescript

## Proof of Concept

Here's a simple PoC demonstrating that a manager can set their own address as the work address:

```typescript
it("should allow manager to set themselves as work address (demonstrates issue)", async () => {
    const manager = agentOwner1;
    
    // First whitelist the manager
    await agentOwnerRegistry.addAddressToWhitelist(manager, { from: governance });
    
    // Manager sets their own address as work address (should not be allowed)
    await agentOwnerRegistry.setWorkAddress(manager, { from: manager });
    
    // Verify that manager is now both management and work address
    const workAddress = await agentOwnerRegistry.getWorkAddress(manager);
    assert.equal(workAddress, manager, "Manager successfully set themselves as work address - demonstrates the issue");
    
    // This breaks the intended separation between management and work addresses
    assert.equal(manager, workAddress, "Same address serves as both management and work address");
});
````

## Mitigation

To address this issue and improve code quality, add the following require checks at the beginning of the `setWorkAddress` function:

```solidity
require(msg.sender != _ownerWorkAddress, "Work address can not be management address")
```
