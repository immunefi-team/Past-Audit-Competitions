# 58249 sc low broken two step admin handover in alchemistcurator

**Submitted on Oct 31st 2025 at 17:28:01 UTC by @pikachu0203 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58249
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

The `AlchemistCurator` contract implements a two-step admin handover mechanism, but the implementation is flawed. The `acceptAdminOwnership()` function uses the `onlyAdmin` modifier, which requires the **current admin** to call the function, rather than the **pending admin**. This violates the standard two-step ownership transfer pattern and makes emergency handovers impossible if the current admin becomes unavailable or compromised.

### Affected Scenarios

* **Emergency Response**: When current admin key is compromised and needs immediate transfer
* **Key Loss**: When current admin loses access to their private key
* **Multi-Sig Issues**: When current admin is a multi-sig that becomes unavailable
* **Time-Sensitive Operations**: When admin transfer needs to happen quickly

## Details of the Vulnerability

### Root Cause

The vulnerability exists in the `acceptAdminOwnership()` function:

**File**: `src/AlchemistCurator.sol`

**Vulnerable Code**:

```solidity
// Lines 27-29
function transferAdminOwnerShip(address _newAdmin) external onlyAdmin {
    pendingAdmin = _newAdmin;  // Correctly sets pending admin
}

// Lines 31-35
function acceptAdminOwnership() external onlyAdmin {  // WRONG: Requires current admin
    admin = pendingAdmin;
    pendingAdmin = address(0);
    emit AdminChanged(admin);
}
```

### The Problem

The `onlyAdmin` modifier checks:

```solidity
require(msg.sender == admin, "PD");
```

This means:

1. **Current admin** must call `acceptAdminOwnership()` (wrong)
2. **Pending admin** cannot call `acceptAdminOwnership()` (should be able to)

### Attack Flow

1. **Step 1**: Current admin's key is compromised
   * Admin realizes compromise and quickly calls `transferAdminOwnerShip(secureAddress)`
   * `pendingAdmin` is set to `secureAddress`
   * Transaction succeeds
2. **Step 2**: Compromised admin loses access or is blocked
   * Attacker takes control of admin key
   * Legitimate admin can no longer call functions
   * OR admin key is lost before completing handover
3. **Step 3**: Attempt to complete handover
   * `secureAddress` (pending admin) tries to call `acceptAdminOwnership()`
   * Transaction **reverts with "PD"** because `onlyAdmin` checks `msg.sender == admin`
   * `msg.sender` is `secureAddress`, but `admin` is still the compromised address
   * Handover **permanently stuck**
4. **Result**: Governance deadlock
   * Current admin cannot fix (key compromised/lost)
   * Pending admin cannot accept (wrong modifier)
   * No recovery possible

## Proof of Concept

### Test File

Create a test file at: `src/test/AlchemistCuratorBrokenTwoStepPOC.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import {console} from "forge-std/console.sol";
import {AlchemistCurator} from "../AlchemistCurator.sol";

/**
 * @title AlchemistCuratorBrokenTwoStepPOC
 * @notice Proof of Concept for Broken Two-Step Handover Vulnerability
 */
contract AlchemistCuratorBrokenTwoStepPOC is Test {
    AlchemistCurator public curator;
    
    address public initialAdmin = address(0x1111);
    address public initialOperator = address(0x2222);
    address public newAdmin = address(0x3333);
    address public secureAdmin = address(0x4444);

    function setUp() public {
        // Deploy AlchemistCurator with initial admin and operator
        vm.prank(initialAdmin);
        curator = new AlchemistCurator(initialAdmin, initialOperator);
    }

    /**
     * @notice Test demonstrating pendingAdmin cannot accept ownership
     */
    function test_PendingAdminCannotAccept() public {
        console.log("=== Broken Two-Step Handover Vulnerability POC ===");
        
        // STEP 1: Current admin initiates proper handover
        console.log("\n[STEP 1] Admin initiates handover to newAdmin:");
        vm.prank(initialAdmin);
        curator.transferAdminOwnerShip(newAdmin);
        
        assertEq(curator.pendingAdmin(), newAdmin, "Pending admin should be newAdmin");
        assertEq(curator.admin(), initialAdmin, "Current admin should still be initialAdmin");
        console.log("pendingAdmin set to:", vm.toString(newAdmin));
        console.log("Current admin remains:", vm.toString(initialAdmin));
        
        // STEP 2: Attempt to accept as pendingAdmin (should fail - wrong caller)
        console.log("\n[STEP 2] Attempting handover acceptance:");
        console.log("newAdmin (pendingAdmin) tries to accept...");
        
        vm.prank(newAdmin);
        vm.expectRevert(abi.encode("PD")); // Permission Denied - onlyAdmin modifier fails
        curator.acceptAdminOwnership();
        
        console.log("pendingAdmin CANNOT accept (onlyAdmin modifier blocks it)");
        console.log("   Error: Only current admin can accept (WRONG DESIGN)");
        
        // STEP 3: Only current admin can accept 
        console.log("\n[STEP 3] Current admin accepts (this works, but defeats the purpose):");
        console.log("Initial admin accepts their own replacement...");
        
        vm.prank(initialAdmin);
        curator.acceptAdminOwnership();
        
        assertEq(curator.admin(), newAdmin, "Admin should now be newAdmin");
        console.log("Current admin accepted - admin is now:", vm.toString(newAdmin));
        console.log("But this requires current admin to be available - breaks emergency scenarios");
    }

    /**
     * @notice Test demonstrating emergency handover is impossible
     */
    function test_EmergencyHandoverImpossible() public {
        console.log("\n=== Emergency Handover Scenario ===");
        
        // Scenario: Current admin key is compromised
        console.log("\n[SCENARIO] Current admin key compromised:");
        console.log("Governance wants to transfer admin to secure address immediately");
        
        // Current admin (compromised but still has temporary access) initiates transfer
        vm.prank(initialAdmin);
        curator.transferAdminOwnerShip(secureAdmin);
        
        console.log("Pending admin set to secure address:", vm.toString(secureAdmin));
        console.log("Current admin is still:", vm.toString(initialAdmin));
        
        // PROBLEM: If compromised admin loses access BEFORE accepting:
        console.log("\n[PROBLEM] If compromised admin loses access before accepting:");
        
        // Simulate: compromised admin can no longer call (key stolen/lost)
        // pendingAdmin tries to accept (this should work in proper implementation)
        vm.prank(secureAdmin);
        vm.expectRevert(abi.encode("PD"));
        curator.acceptAdminOwnership();
        
        console.log("pendingAdmin (secureAdmin) cannot accept (reverts with PD)");
        console.log("Compromised admin cannot accept (lost access)");
        console.log("RESULT: Emergency handover is IMPOSSIBLE");
        console.log(" The contract is stuck - no one can complete the handover");
        
        // Verify the state
        assertEq(curator.pendingAdmin(), secureAdmin, "Pending admin should still be secureAdmin");
        assertEq(curator.admin(), initialAdmin, "Admin should still be compromised admin");
        console.log("\nContract state: Handover initiated but CANNOT be completed");
    }

    /**
     * @notice Test demonstrating what happens in key loss scenario
     */
    function test_KeyLossScenario() public {
        console.log("\n=== Key Loss Scenario ===");
        
        // Scenario: Admin realizes key might be compromised, initiates transfer
        vm.prank(initialAdmin);
        curator.transferAdminOwnerShip(secureAdmin);
        
        console.log("Admin initiates handover before losing key access");
        console.log("Pending admin:", vm.toString(secureAdmin));
        
        // Scenario: Admin loses key before accepting
        console.log("\n[SCENARIO] Admin loses key before completing handover:");
        console.log("- Current admin cannot call acceptAdminOwnership() (key lost)");
        console.log("- Pending admin cannot call acceptAdminOwnership() (onlyAdmin blocks it)");
        
        // Attempt from pendingAdmin fails
        vm.prank(secureAdmin);
        vm.expectRevert(abi.encode("PD"));
        curator.acceptAdminOwnership();
        
        console.log("Result: Permanent deadlock - handover cannot complete");
        
        // Verify stuck state
        assertEq(curator.pendingAdmin(), secureAdmin, "Still pending");
        assertEq(curator.admin(), initialAdmin, "Still old admin");
    }
```
