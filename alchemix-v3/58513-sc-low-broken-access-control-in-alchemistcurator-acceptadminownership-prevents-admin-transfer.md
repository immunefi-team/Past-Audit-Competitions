# 58513 sc low broken access control in alchemistcurator acceptadminownership prevents admin transfer

**Submitted on Nov 2nd 2025 at 23:11:41 UTC by @Tomioka for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58513
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**

## Description

The acceptAdminOwnership function in the AlchemistCurator contract contains a critical access control vulnerability where it uses the `onlyAdmin` modifier instead of allowing the `pendingAdmin` to call it. This breaks the standard two step ownership transfer pattern and makes it impossible to transfer admin privileges to a different address, potentially locking administrative functions if the current admin key is compromised or lost.

### Vulnerability Details

The AlchemistCurator implements a two step admin transfer process:

**Step 1: Propose Transfer** (transferAdminOwnerShip):

```solidity
function transferAdminOwnerShip(address _newAdmin) external onlyAdmin {
    pendingAdmin = _newAdmin;
}
```

**Step 2: Accept Transfer** (acceptAdminOwnership):

```solidity
function acceptAdminOwnership() external onlyAdmin {  //  WRONG MODIFIER
    admin = pendingAdmin;
    pendingAdmin = address(0);
    emit AdminChanged(admin);
}
```

The bug is on line 31, the function uses `onlyAdmin` modifier, which restricts the call to the **current admin**, not the **pending admin**. The standard two step ownership transfer pattern requires:

1. Current admin proposes new admin (`transferAdminOwnerShip`)
2. **Pending admin** accepts the transfer (`acceptAdminOwnership`) CORRECT PATTERN
3. Ownership transfers

However, the current implementation requires:

1. Current admin proposes new admin
2. **Current admin** accepts the transfer BROKEN PATTERN
3. Admin changes to pendingAdmin

**Why This is Broken:**

The two step pattern exists to prevent accidental transfers to wrong addresses. The pending admin must **prove they control the new address** by calling accept. With the current bug:

* Current admin can transfer to themselves (pointless)
* Current admin CANNOT transfer to a different address (the new admin can't accept)
* The `pendingAdmin` cannot call `acceptAdminOwnership()` due to access control

**Complete Attack Scenario:**

While this isn't an "attack" in the malicious sense, it creates a critical operational failure:

1. Current admin wants to transfer to new multisig/address
2. Admin calls `transferAdminOwnerShip(newAddress)`
3. `pendingAdmin` is set to `newAddress`
4. New admin attempts to call `acceptAdminOwnership()`
5. **Transaction REVERTS** - caller is not current admin
6. Admin transfer is **permanently stuck**

**Real World Impact:**

This bug has severe consequences in production:

* **Safe key rotation impossible**: Cannot perform secure admin rotation using the two-step pattern
* **Multisig migration blocked**: Cannot safely upgrade from EOA to multisig
* **Compromise scenario**: Cannot use the intended emergency transfer mechanism
* **DAO governance broken**: Cannot safely transfer to governance contracts

*Note: While the two step process is broken, admin transfer is still possible via the unsafe `setAdmin()` function, but this bypasses all safety mechanisms.*

**Root Cause:**

The bug stems from incorrect modifier application. The correct implementation should check that `msg.sender == pendingAdmin`, not `msg.sender == admin`. This is evident from other similar contracts in the ecosystem (OpenZeppelin's Ownable2Step, etc.).

**Comparison with Correct Implementation:**

OpenZeppelin's Ownable2Step (correct pattern):

```solidity
function acceptOwnership() public virtual {
    address sender = _msgSender();
    require(pendingOwner() == sender, "Ownable2Step: caller is not the new owner");
    _transferOwnership(sender);
}
```

AlchemistCurator (broken pattern):

```solidity
function acceptAdminOwnership() external onlyAdmin {  // Should be: msg.sender == pendingAdmin
    admin = pendingAdmin;
    pendingAdmin = address(0);
    emit AdminChanged(admin);
}
```

**Alternative Path Exists:**

While the two step transfer is broken, there **IS an alternative path** to transfer admin via the `setAdmin()` function inherited from `PermissionedProxy`:

```solidity
function setAdmin(address _admin) external onlyAdmin {
    admin = _admin;
    emit AdminUpdated(_admin);
}
```

This allows the current admin to directly set a new admin without acceptance. However, this defeats the safety purpose of the two step pattern and creates centralized risk.

### Impact Details

**Primary Impact: Safe Admin Transfer Broken**

* The intended two step admin transfer pattern fails
* `pendingAdmin` cannot accept ownership due to access control restrictions
* Forces use of unsafe direct transfer method (`setAdmin()`)

**Secondary Impact: Security and Operational Risks**

* **Key compromise**: Cannot perform secure admin rotation using two step pattern
* **Multisig upgrade**: Cannot safely migrate from EOA to multisig
* **DAO transition**: Cannot safely decentralize to governance contracts
* **Centralized risk**: Must rely on unsafe direct transfer method

**Affected Functionality:**

The following critical admin-only functions can only be accessed by the current admin (no safe rotation possible):

* `transferAdminOwnerShip` (line 27) : Setting strategy caps
* `submitSetStrategy` (line 37) : Adding new yield strategies
* `setStrategy` (line 43) : Strategy management
* `removeStrategy` (line 49) : Emergency strategy removal
* `decreaseAbsoluteCap` (line 73) : Risk management
* `decreaseRelativeCap` (line 83) : Risk management
* `increaseAbsoluteCap` (line 117) : Capacity expansion
* `increaseRelativeCap` (line 122) : Capacity expansion

*Note: Admin can still be changed via `setAdmin()`, but this is unsafe and bypasses the two step safety mechanism.*

**Affected Users:**

* **Protocol administrators**: Cannot perform secure admin rotation; must use unsafe direct method
* **DAO members**: Cannot safely transition to decentralized control
* **Security team**: Cannot use intended emergency transfer mechanism
* **All users**: Exposed to centralized risk due to unsafe admin transfer method

### References

* Vulnerable accept function: src/AlchemistCurator.sol:31-35
* Transfer initiation (works correctly): src/AlchemistCurator.sol:27-29
* Comparison with OpenZeppelin standard: Ownable2Step.sol

## Proof of Concept

## Proof of Concept

Run the PoC

Execute this command in your terminal: forge test --match-path test/CuratorAdminAccept.t.sol -vvv

```solidity
pragma solidity 0.8.28;

import {Test} from "../lib/forge-std/src/Test.sol";
import {AlchemistCurator} from "../src/AlchemistCurator.sol";

/// @title Curator admin acceptance bug – pending admin can't accept, current admin can force
contract CuratorAdminAccept is Test {
    address admin = address(0xA11CE);
    address operator = address(0x0B0B);
    address newAdmin = address(0xB0B);
    address third = address(0xC0C0);

    AlchemistCurator curator;

    function setUp() public {
        vm.startPrank(admin);
        curator = new AlchemistCurator(admin, operator);
        vm.stopPrank();
    }

    function test_PendingAdminCannotAccept_CurrentAdminCanForce() public {
        // Step 1: current admin nominates a pending admin
        vm.prank(admin);
        curator.transferAdminOwnerShip(newAdmin);

        // Step 2: pending admin tries to accept -> should revert due to onlyAdmin modifier
        vm.startPrank(newAdmin);
        vm.expectRevert(bytes("PD"));
        curator.acceptAdminOwnership();
        vm.stopPrank();

        // Step 3: current admin can call accept and unilaterally set admin to the pending admin
        vm.prank(admin);
        curator.acceptAdminOwnership();

        // Step 4: Verify that admin role moved to newAdmin by checking onlyAdmin-gated behavior
        //         setAdmin is onlyAdmin in PermissionedProxy; A (old admin) shouldn't be able to call it anymore,
        //         but newAdmin should be able to.
        vm.expectRevert(bytes("PD"));
        vm.prank(admin);
        curator.setAdmin(third);

        // Now as newAdmin, setAdmin should succeed.
        vm.prank(newAdmin);
        curator.setAdmin(third);

        // After changing admin to `third`, newAdmin should no longer have admin powers.
        vm.expectRevert(bytes("PD"));
        vm.prank(newAdmin);
        curator.setOperator(address(0xDEAD), true);

        // And the new admin (third) should have admin powers.
        vm.prank(third);
        curator.setOperator(address(0xBEEF), true);
    }
}
```

## Recommended Mitigation

Replace the `onlyAdmin` modifier with a proper check for `pendingAdmin`:

```solidity
function acceptAdminOwnership() external {
    require(msg.sender == pendingAdmin, "Not pending admin");
    require(pendingAdmin != address(0), "No pending admin");

    admin = pendingAdmin;
    pendingAdmin = address(0);
    emit AdminChanged(admin);
}
```

This allows the `pendingAdmin` to accept the transfer while preventing unauthorized calls.
