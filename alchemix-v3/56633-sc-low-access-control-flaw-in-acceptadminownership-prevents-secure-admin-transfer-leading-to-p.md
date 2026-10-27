# 56633 sc low access control flaw in acceptadminownership prevents secure admin transfer leading to potential permanent loss of curator control

**Submitted on Oct 18th 2025 at 16:35:52 UTC by @ibrahimatix0x01 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56633
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**

## Description

## Brief/Intro

The AlchemistCurator.acceptAdminOwnership() function incorrectly uses the onlyAdmin modifier, preventing the pending admin from accepting ownership. In a properly implemented two-step ownership transfer, the new admin should be able to confirm they control the address before receiving privileges. Instead, only the current admin can call acceptAdminOwnership(), which defeats the security purpose of the two-step pattern. This creates a risk where admin privileges could be transferred to an incorrect or inaccessible address without the new admin ever confirming control, potentially resulting in permanent loss of curator control over critical protocol parameters including strategy management and vault configuration.

## Vulnerability Details

The AlchemistCurator contract attempts to implement a two-step ownership transfer pattern with transferAdminOwnerShip() and acceptAdminOwnership():

```solidity
function transferAdminOwnerShip(address _newAdmin) external onlyAdmin {
    pendingAdmin = _newAdmin;
}

function acceptAdminOwnership() external onlyAdmin {  // INCORRECT MODIFIER
    admin = pendingAdmin;
    pendingAdmin = address(0);
    emit AdminChanged(admin);
}
```

THE CORE ISSUE: The acceptAdminOwnership() function has the onlyAdmin modifier, meaning only the current admin can call it. This fundamentally breaks the two-step transfer pattern where the pending admin should accept ownership to verify they control the address. EXPECTED IMPLEMENTATION:

```solidity
function acceptAdminOwnership() external {
    require(msg.sender == pendingAdmin, "Not pending admin");
    admin = msg.sender;
    pendingAdmin = address(0);
    emit AdminChanged(admin);
}
```

EVIDENCE THIS IS A BUG (NOT INTENTIONAL DESIGN):

1. Base Contract Uses Single-Step Transfer

The contract inherits from PermissionedProxy.sol, which uses a simple single-step transfer:

```solidity
function setAdmin(address _admin) external onlyAdmin {
    require(_admin != address(0), "zero");
    admin = _admin;
    emit AdminUpdated(_admin);
}
```

If AlchemistCurator wanted the current admin to control everything, it would simply use the inherited setAdmin(). Instead, it deliberately adds complexity with pendingAdmin and two separate functions, indicating an attempt to implement enhanced security through two-step verification.

2. Pattern of Incomplete Implementation

The contract declares multiple unused "pending" variables:

```solidity
address public operator;
address public pendingOperator;  // Declared but never used
address public pendingAdmin;
address public pendingStrategy;  // Declared but never used
```

This strongly suggests incomplete implementation of pending/acceptance patterns throughout the contract.

3. Function Naming Convention

The function is named acceptAdminOwnership() - the word "accept" implies the recipient accepts, not the sender. If the current admin was intended to execute it, it would be named executeAdminTransfer() or finalizeAdminChange().

## Impact Details

PRIMARY IMPACT: The pending admin cannot call acceptAdminOwnership() to accept ownership, completely breaking the two-step verification mechanism that was intended to be implemented. SECURITY CONSEQUENCES:

No Address Verification Before Transfer

The new admin cannot verify they control the private key for the address before receiving admin privileges In a proper two-step transfer, this confirmation step is the entire security benefit - it prevents transfers to wrong addresses

Permanent Loss of Admin Control Risk

If pendingAdmin is set to an incorrect address, the current admin can still complete the transfer. Once transferred, there is no recovery mechanism - the curator admin role would be permanently lost. Example scenarios:

Typo: 0x123...789 instead of 0x123...788 Wrong network: Mainnet address used on L2 deployment Inaccessible multisig: Keys lost or threshold cannot be met Contract address: Address without acceptAdminOwnership() function

Compromised Admin Attack Window

Timeline attack scenario:

Admin calls transferAdminOwnerShip(legitimateNewAdmin) Admin's keys get compromised Attacker calls acceptAdminOwnership() before legitimate admin can Result: Attacker gains curator control, legitimate admin locked out Zero Security Benefit vs. Increased Complexity

The two-step pattern provides no advantage over the inherited PermissionedProxy.setAdmin() single-step function Adds code complexity and gas costs with no security improvement Violates security best practices for ownership transfer (see OpenZeppelin's Ownable2Step pattern)

OPERATIONAL IMPACT ON PROTOCOL: The AlchemistCurator admin role has critical control over:

Strategy Management: Adding/removing MYT yield strategies via setStrategy(), removeStrategy() Risk Parameters: Adjusting absoluteCap and relativeCap for strategies Vault Configuration: Submitting timelock transactions to the Morpho V2 vault Emergency Response: Ability to decrease caps or remove compromised strategies

Loss of curator control would prevent:

Emergency removal of exploited strategies Adjustment of risk parameters during market volatility Addition of new yield opportunities Response to security incidents Protocol parameter updates

While this does not directly result in immediate theft or loss of user funds, it violates fundamental access control security patterns and creates operational risk for protocol governance. SEVERITY CLASSIFICATION: This vulnerability is classified as a Security Best Practices issue affecting admin/governance functionality. It does not directly cause theft or freezing of user funds but represents a critical flaw in access control implementation that could lead to permanent loss of protocol management capabilities.

## RECOMMENDED FIX:

```solidity
function acceptAdminOwnership() external {
    require(msg.sender == pendingAdmin, "Not pending admin");
    admin = msg.sender;
    pendingAdmin = address(0);
    emit AdminChanged(admin);
}
```

Changes needed:

Remove the onlyAdmin modifier Add explicit validation that msg.sender is the pendingAdmin This matches the standard two-step ownership transfer pattern used throughout DeFi (OpenZeppelin's Ownable2Step, Compound's Timelock, etc.)

## Proof of Concept

## Proof of Concept

```solidity
function testPoC_PendingAdminCannotAcceptOwnership() public {
    address newAdmin = address(0x9999);
    
    // Step 1: Current admin initiates transfer
    vm.prank(admin);
    mytCuratorProxy.transferAdminOwnerShip(newAdmin);
    assertEq(mytCuratorProxy.pendingAdmin(), newAdmin);
    
    // Step 2: BUG - pendingAdmin cannot accept (reverts with "PD")
    vm.prank(newAdmin);
    vm.expectRevert(abi.encode("PD")); // Permission Denied
    mytCuratorProxy.acceptAdminOwnership();
    
    // Step 3: Only current admin can accept (defeats purpose)
    vm.prank(admin);
    mytCuratorProxy.acceptAdminOwnership();
    assertEq(mytCuratorProxy.pendingAdmin(), address(0));
    
    // Result: newAdmin has control without ever confirming
    vm.prank(newAdmin);
    mytCuratorProxy.transferAdminOwnerShip(admin); // Works - proves newAdmin is admin
}
```

The test demonstrates:

Pending admin CANNOT call acceptAdminOwnership() (reverts) Only current admin CAN call acceptAdminOwnership() (succeeds) Transfer completes without new admin's confirmation
