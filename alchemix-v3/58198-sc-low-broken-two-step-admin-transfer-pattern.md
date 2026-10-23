# 58198 sc low broken two step admin transfer pattern

**Submitted on Oct 31st 2025 at 10:23:39 UTC by @teoslaf1 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58198
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Permanent freezing of funds
  * incorrect access control

## Description

### Description

The `acceptAdminOwnership()` function has incorrect access control that prevents the pending admin from accepting ownership while allowing the current admin to complete the transfer unilaterally. This completely defeats the purpose of the two-step transfer pattern, which requires the new admin to explicitly accept ownership to prevent accidental transfers to incorrect or inaccessible addresses.

The function uses `onlyAdmin` modifier, which checks `msg.sender == admin` (the current admin), when it should check `msg.sender == pendingAdmin` (the new pending admin).

### Vulnerable Code

```solidity
function acceptAdminOwnership() external onlyAdmin {  // Wrong modifier
    admin = pendingAdmin;
    pendingAdmin = address(0);
    emit AdminChanged(admin);
}
```

### Impact

* Pending admin cannot accept ownership: The intended recipient is blocked from accepting the transfer
* Current admin retains full control: Can complete the transfer unilaterally without the new admin's consent or confirmation
* Risk of permanent loss: If admin transfers to a wrong/inaccessible address, they can still complete it, permanently losing control
* Defeats security pattern: The two-step transfer becomes a one-step transfer with extra steps
* No validation of new admin: The new address never proves it can interact with the contract

### Recommendation

Change the access control to require the **pending admin** (not current admin) to accept:

```solidity
function acceptAdminOwnership() external {
    require(msg.sender == pendingAdmin, "not pending admin");
    admin = pendingAdmin;
    pendingAdmin = address(0);
    emit AdminChanged(admin);
}
```

## Proof of Concept

## Proof of Concept

Add this to AlchemistCurator.t.sol

```solidity
   function testBrokenAdminTransfer() public {
        address pendingAdmin = address(0x6666666666666666666666666666666666666666);
        
        // Step 1: Current admin initiates transfer to new address
        vm.prank(admin);
        mytCuratorProxy.transferAdminOwnerShip(pendingAdmin);
        
        // Step 2: Pending admin CANNOT accept because onlyAdmin requires current admin
        vm.prank(pendingAdmin);
        vm.expectRevert(abi.encode("PD")); // This should not revert in a proper 2-step transfer
        mytCuratorProxy.acceptAdminOwnership();
        
        // Step 3:Current admin can accept on behalf of pending admin
        vm.prank(admin);
        mytCuratorProxy.acceptAdminOwnership();
        
        // Result: Admin changed without pendingAdmin ever calling accept
        // The pendingAdmin is now the admin, but they never confirmed the transfer
        assertEq(mytCuratorProxy.pendingAdmin(), address(0));
    }
```
