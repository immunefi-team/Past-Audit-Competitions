# 56383 sc low the alchemistcurator acceptadminownership can t be called by the pending admin and if the function is called without pending admin the admin rigths will be lost

**Submitted on Oct 15th 2025 at 10:39:24 UTC by @EagleEye for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56383
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `AlchemistCurator::transferAdminOwnerShip` and `AlchemistCurator::acceptAdminOwnership` functions contain a logical error that prevents the pending admin from accepting the ownership. Additionally, if the current admin calls `acceptAdminOwnership` before setting a valid pending admin address, the contract’s admin rights will be permanently lost by setting the admin to the zero address.

## Vulnerability Details

The `AlchemistCurator::acceptAdminOwnership` function is restricted by the `onlyAdmin` modifier. This means only the current admin can call it. As a result, the pending admin (the intended new admin) has no way to accept ownership, making the ownership transfer process ineffective. Also, if the current admin calls `AlchemistCurator::acceptAdminOwnership` before any pending admin is set, then the new admin will be assigned to the zero address. Since the `onlyAdmin` modifier restricts critical functions to admin, and no one controls the zero address, all administrative control over the contract will be lost permanently.

```
@>      function transferAdminOwnerShip(address _newAdmin) external onlyAdmin {
        pendingAdmin = _newAdmin;
    }

    function acceptAdminOwnership() external onlyAdmin {
        admin = pendingAdmin;
 @>  pendingAdmin = address(0);
        emit AdminChanged(admin);
    }
```

## Impact Details

The ownership transfer mechanism is non-functional. The expected behavior is that the new admin should accept the admin role, but the implementation doesn't do that. Control can never be properly transferred to a new admin, breaking expected administrative processes. Also, if the admin calls the `acceptOwnership` without pending admin, the new admin will be the zero address and the contract becomes irreversibly locked, as no valid account can perform admin-only operations.

## References

<https://github.com/alchemix-finance/v3-poc/blob/b2e2aba046c36ff5e1db6f40f399e93cd2bdaad0/src/AlchemistCurator.sol#L27-L35>

## Recommendation

Remove the `onlyAdmin` modifier from the `acceptOwnership` function and add a check to ensure that there is a pending admin and the pending admin is the caller:

```
function acceptAdminOwnership() external {
        require(msg.sender == pendingAdmin, "Not pending admin");
        admin = pendingAdmin;
        pendingAdmin = address(0);
        emit AdminChanged(admin);
    }
```

## Proof of Concept

## Proof of Concept

The fist `testChangeAdmin` test shows that the pending admin is not able to accept the ownership, because of the `onlyAdmin` modifier of the `acceptAdminOwnership` function. The second test `testAdminCallsAcceptOwnershipWithoutPendingAdmin` shows that if the current admin calls the `acceptAdminOwnership` function without pending admin, the new admin will be the zero address and the admin rights of the contracts will be lost. This is because the `acceptAdminOwnership` function doesn't check if there is a pending admin:

```
contract AlchemistV3Tests is StdInvariant, Test {
    
    AlchemistCurator public curator;
    PermissionedProxy public proxy;
    address public admin = address(1);
    address public operator = address(2);

    function setUp() public {
        proxy = new PermissionedProxy(admin, operator);
        curator = new AlchemistCurator(admin, operator);
    }

    function testChangeAdmin() public {
        vm.prank(admin);
        curator.transferAdminOwnerShip(address(3));
        //Pending admin can't accept the ownership
        vm.prank(address(3));
        vm.expectRevert();
        curator.acceptAdminOwnership();
    }

    function testAdminCallsAcceptOwnershipWithoutPendingAdmin() public {
        //If the admin calls by mistake the acceptAdminOwnership function without a pending admin, the admin will be set to address(0) and the admin rights will be lost
        vm.startPrank(admin);
        curator.acceptAdminOwnership();
    }
}
```

The results:

```
[36640] AlchemistV3Tests::testChangeAdmin()
    ├─ [0] VM::prank(ECRecover: [0x0000000000000000000000000000000000000001])
    │   └─ ← [Return] 
    ├─ [24699] AlchemistCurator::transferAdminOwnerShip(RIPEMD-160: [0x0000000000000000000000000000000000000003])
    │   └─ ← [Stop] 
    ├─ [0] VM::prank(RIPEMD-160: [0x0000000000000000000000000000000000000003])
    │   └─ ← [Return] 
    ├─ [0] VM::expectRevert(custom error 0xf4844814)
    │   └─ ← [Return] 
    ├─ [507] AlchemistCurator::acceptAdminOwnership()
    │   └─ ← [Revert] revert: PD
    └─ ← [Stop] 

[19025] AlchemistV3Tests::testAdminCallsAcceptOwnershipWithoutPendingAdmin()
    ├─ [0] VM::startPrank(ECRecover: [0x0000000000000000000000000000000000000001])
    │   └─ ← [Return] 
    ├─ [8797] AlchemistCurator::acceptAdminOwnership()
    │   ├─ emit AdminChanged(admin: 0x0000000000000000000000000000000000000000)
    │   └─ ← [Stop] 
    └─ ← [Stop] 

```
