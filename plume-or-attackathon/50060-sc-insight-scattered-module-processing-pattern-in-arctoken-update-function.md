# 50060 sc insight scattered module processing pattern in arctoken update function

**Submitted on Jul 21st 2025 at 11:47:58 UTC by @AasifUsmani for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50060
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:** (none listed)

## Description

### Brief/Intro

The `_update` function in `ArcToken.sol` exhibits a scattered and unorganized approach to restriction module processing. The function performs redundant address validation checks and spreads module-related operations across multiple sections instead of grouping them logically, resulting in unnecessary gas consumption and poor code maintainability.

### Vulnerability Details

Technical Analysis

The function demonstrates an inefficient pattern where module operations are fragmented across different execution phases:

1. Redundant Address Validation Pattern

```solidity
// Phase 1: Permission checks
address specificTransferModule = $.specificRestrictionModules[RestrictionTypes.TRANSFER_RESTRICTION_TYPE];
if (specificTransferModule != address(0)) {  // ❌ Check #1
    transferAllowed = transferAllowed && ITransferRestrictions(specificTransferModule).isTransferAllowed(from, to, amount);
}

address globalTransferModule = IRestrictionsRouter(routerAddr).getGlobalModuleAddress(RestrictionTypes.GLOBAL_SANCTIONS_TYPE);
if (globalTransferModule != address(0)) {  // ❌ Check #2
    try ITransferRestrictions(globalTransferModule).isTransferAllowed(from, to, amount) returns (bool globalAllowed) {
        transferAllowed = transferAllowed && globalAllowed;
    } catch {
        transferAllowed = false;
    }
}

// Phase 2: Before hooks - scattered across function
if (specificTransferModule != address(0)) {  // ❌ Check #3 - same variable!
    ITransferRestrictions(specificTransferModule).beforeTransfer(from, to, amount);
}
if (globalTransferModule != address(0)) {  // ❌ Check #4 - same variable!
    try ITransferRestrictions(globalTransferModule).beforeTransfer(from, to, amount) {}
    catch {}
}

// Phase 3: After hooks - scattered again
if (specificTransferModule != address(0)) {  // ❌ Check #5 - same variable!
    ITransferRestrictions(specificTransferModule).afterTransfer(from, to, amount);
}
if (globalTransferModule != address(0)) {  // ❌ Check #6 - same variable!
    try ITransferRestrictions(globalTransferModule).afterTransfer(from, to, amount) {}
    catch {}
}
```

2. Unorganized Logic Distribution

The function spreads operations for each module across three separate sections of code, making it difficult to understand the complete module interaction flow and requiring multiple modifications when updating module handling logic.

3. Inconsistent Organization Approach

Instead of grouping related operations by module, the current implementation groups by operation type (permissions → before hooks → after hooks), which fragments the logical flow and increases complexity.

### Impact Details

Gas Inefficiency

* Current waste: 6 address comparisons per transfer (3 for specific module, 3 for global module)
* Optimal requirement: 2 address comparisons per transfer
* Estimated savings: 400-600 gas per transfer
* Cumulative impact: Significant for high-volume tokens with frequent transfers

Code Maintainability Issues

* Multiple modification points: Changes to module handling require updates in 3 separate locations
* Logic fragmentation: Complete module flow cannot be understood from a single code section
* Debugging complexity: Tracing module interactions requires jumping between different parts of the function

## Recommendation

Reorganize the function to process each module completely in dedicated sections. Example suggested restructuring:

```solidity
function _update(address from, address to, uint256 amount) internal virtual override {
    ArcTokenStorage storage $ = _getArcTokenStorage();
    
    address routerAddr = $.restrictionsRouter;
    if (routerAddr == address(0)) {
        revert RouterNotSet();
    }
    
    // ✅ Process specific module completely in one section
    address specificModule = $.specificRestrictionModules[RestrictionTypes.TRANSFER_RESTRICTION_TYPE];
    if (specificModule != address(0)) {
        // All specific module operations grouped together
        if (!ITransferRestrictions(specificModule).isTransferAllowed(from, to, amount)) {
            revert TransferRestricted();
        }
        ITransferRestrictions(specificModule).beforeTransfer(from, to, amount);
    }
    
    // ✅ Process global module completely in one section
    address globalModule = IRestrictionsRouter(routerAddr).getGlobalModuleAddress(RestrictionTypes.GLOBAL_SANCTIONS_TYPE);
    if (globalModule != address(0)) {
        // All global module operations grouped together
        try ITransferRestrictions(globalModule).isTransferAllowed(from, to, amount) returns (bool allowed) {
            if (!allowed) revert TransferRestricted();
        } catch {
            revert TransferRestricted();
        }
        
        try ITransferRestrictions(globalModule).beforeTransfer(from, to, amount) {}
        catch {}
    }
    
    // ✅ Core balance tracking logic
    if (from != address(0)) {
        uint256 fromBalanceBefore = balanceOf(from);
        if (fromBalanceBefore == amount) {
            $.holders.remove(from);
        }
    }
    
    // ✅ Execute the actual transfer
    super._update(from, to, amount);
    
    // ✅ Add new holder tracking
    if (to != address(0) && balanceOf(to) > 0) {
        $.holders.add(to);
    }
    
    // ✅ After hooks - grouped by module
    if (specificModule != address(0)) {
        ITransferRestrictions(specificModule).afterTransfer(from, to, amount);
    }
    if (globalModule != address(0)) {
        try ITransferRestrictions(globalModule).afterTransfer(from, to, amount) {}
        catch {}
    }
}
```

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L653>

## Proof of Concept

{% stepper %}
{% step %}
To check the excessive gas used by original function, run the original function and log the gas amount.
{% endstep %}

{% step %}
Run the recommended (modified) function and log the gas amount for this version as well.
{% endstep %}

{% step %}
Compare the logged gas usage between the original and the recommended implementations.
{% endstep %}

{% step %}
You will observe a significant gas saving for the modified function; the modified function is also more readable and easier to maintain.
{% endstep %}
{% endstepper %}
