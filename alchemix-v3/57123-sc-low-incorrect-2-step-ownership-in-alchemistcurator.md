# 57123 sc low incorrect 2 step ownership in alchemistcurator

**Submitted on Oct 23rd 2025 at 17:21:45 UTC by @JoeMama for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57123
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The current implementation of `acceptAdminOwnership()` allows any current admin to accept ownership on behalf of the pendingAdmin, instead of requiring the pending admin to explicitly accept it.

This defeats the purpose of the two-step ownership transfer, which is meant to prevent wrong address ownership transfers. The acceptance should come only from the pending admin.

## Vulnerability Details

```
    function transferAdminOwnerShip(address _newAdmin) external onlyAdmin {
        pendingAdmin = _newAdmin;
    }

    function acceptAdminOwnership() external onlyAdmin {
        admin = pendingAdmin;
        pendingAdmin = address(0);
        emit AdminChanged(admin);
    }
```

## Impact Details

The `acceptAdminOwnership` function is meant to verify that the new admin address actually exists. However, since the new admin must now call `acceptAdminOwnership` to complete the transfer, there’s a risk that the specified address doesn’t exist or isn’t accessible. If ownership is transferred to such an address, the curator contracts would become unusable because no one would be able to operate them.

## Recommendation:

Update the function to require that only the pending admin can call it:

```
function transferAdminOwnerShip(address _newAdmin) external onlyAdmin {
      pendingAdmin = _newAdmin;
 }

function acceptAdminOwnership() external { 
      require(pendingAdmin == msg.sender); 
      admin = pendingAdmin;
      pendingAdmin = address(0);
      emit AdminChanged(admin);
}
```

## Proof of Concept

## Proof of Concept

Please add these 2 test to the existing AlchemistCuratorTest.

```
    function testTransferOwnershipPendingCannotAccept() public {
        address pendingAdmin = makeAddr("pendingAmin");

        vm.prank(admin);
        mytCuratorProxy.transferAdminOwnerShip(pendingAdmin);

        vm.expectRevert(abi.encode("PD"));
        vm.prank(pendingAdmin);
        mytCuratorProxy.acceptAdminOwnership(); // pending admin should be able to call this but it reverts.
    }

    function testTransferOwnershipToNonExistingAddress() public {
   
        vm.prank(admin);
        mytCuratorProxy.transferAdminOwnerShip(address(0));

        vm.prank(admin);
        // the owner of the address is now address 0 meaning the contract cannot be used anymore.
        mytCuratorProxy.acceptAdminOwnership(); 
    }
```
