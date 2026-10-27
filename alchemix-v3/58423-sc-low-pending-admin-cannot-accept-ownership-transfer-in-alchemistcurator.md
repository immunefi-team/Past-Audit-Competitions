# 58423 sc low pending admin cannot accept ownership transfer in alchemistcurator&#x20;

**Submitted on Nov 2nd 2025 at 08:39:40 UTC by @Ekene for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58423
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Pending Admin Cannot Accept Ownership Transfer

## Description

### **Summary**

When the current admin calls `transferAdminOwnerShip()` to set a new `pendingAdmin`, the transfer cannot be completed because the `acceptAdminOwnership()` function is restricted by the `onlyAdmin` modifier. Since the `pendingAdmin` is **not** yet the `admin`, they cannot successfully call this function, effectively breaking the intended admin transfer flow.

### **Impact**

Admin ownership transfer becomes impossible. This could permanently lock administrative privileges to the original admin and prevent future upgrades or configuration changes.

### **Root Cause**

The `acceptAdminOwnership()` function uses the `onlyAdmin` modifier, preventing the `pendingAdmin` from executing it.

### **Recommended Mitigation**

Remove the `onlyAdmin` restriction and explicitly allow the pending admin to accept ownership:

```solidity
function acceptAdminOwnership() external {
    require(msg.sender == pendingAdmin, "Not pending admin");
    admin = pendingAdmin;
    pendingAdmin = address(0);
    emit AdminChanged(admin);
}
```

## Proof of Concept

### **Proof of Concept**

```solidity
function testAdminTransferAndAcceptance() external {
    // Initiate transfer by admin
    vm.startPrank(admin);
    mytCuratorProxy.transferAdminOwnerShip(pendingAdmin);
    vm.stopPrank();

    // Attempt acceptance by pending admin
    vm.startPrank(pendingAdmin);
    vm.expectRevert(abi.encode("PD"));
    mytCuratorProxy.acceptAdminOwnership(); // reverts
    vm.stopPrank();
}
```
