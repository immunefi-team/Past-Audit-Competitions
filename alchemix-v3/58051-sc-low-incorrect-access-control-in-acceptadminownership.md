# 58051 sc low incorrect access control in acceptadminownership&#x20;

**Submitted on Oct 30th 2025 at 09:55:07 UTC by @dldLambda for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58051
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The acceptAdminOwnership() function is incorrectly protected by the onlyAdmin modifier. This prevents the pending admin from claiming ownership, rendering the two-step ownership transfer mechanism broken. As a result:

The current admin is the only one who can call acceptAdminOwnership(). If the current admin loses access (key loss, multisig failure, etc.), no one can ever become the new admin. The contract becomes permanently unmanageable.

## Vulnerability Details

Let's take a closer look at what the problem is.

onlyAdmin modifier:

```
    modifier onlyAdmin() {
        require(msg.sender == admin, "PD");
        _;
    }
```

That is, to call a function with this modifier, the caller must have admin status.

Two-step transfer of access rights:

```
    // ===== Admin Management =====
    function transferAdminOwnerShip(address _newAdmin) external onlyAdmin {
        pendingAdmin = _newAdmin;
    }

    function acceptAdminOwnership() external onlyAdmin {
        admin = pendingAdmin;
        pendingAdmin = address(0);
        emit AdminChanged(admin);
    }
```

The main problem is that function acceptAdminOwnership is implemented incorrectly. When called, it requires that the caller already be an admin, but he cannot be one because he is still a pendingAdmin.

It's a vicious circle: to call a function and gain admin rights, you need to be an admin, but you can't do that because you're not an admin.

Here's what a proper implementation looks like, for example (openzeppelin-contracts):

```
   /**
     * @dev Starts the ownership transfer of the contract to a new account. Replaces the pending transfer if there is one.
     * Can only be called by the current owner.
     *
     * Setting `newOwner` to the zero address is allowed; this can be used to cancel an initiated ownership transfer.
     */
    function transferOwnership(address newOwner) public virtual override onlyOwner {
        _pendingOwner = newOwner;
        emit OwnershipTransferStarted(owner(), newOwner);
    }

    /**
     * @dev Transfers ownership of the contract to a new account (`newOwner`) and deletes any pending owner.
     * Internal function without access restriction.
     */
    function _transferOwnership(address newOwner) internal virtual override {
        delete _pendingOwner;
        super._transferOwnership(newOwner);
    }

    /**
     * @dev The new owner accepts the ownership transfer.
     */
    function acceptOwnership() public virtual {
        address sender = _msgSender();
        if (pendingOwner() != sender) {
            revert OwnableUnauthorizedAccount(sender);
        }
        _transferOwnership(sender);
    }
```

## Impact Details

Ownership transfer is permanently broken — the two-step admin transfer mechanism fails at the final step. If the current admin becomes unavailable (lost key, team departure, multisig failure), no one can ever assume control.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistCurator.sol#L27>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistCurator.sol#L31>

<https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable2Step.sol>

## Proof of Concept

## Proof of Concept

1. add getters to MockAlchemistCurator:

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {AlchemistCurator} from "../../AlchemistCurator.sol";

contract MockAlchemistCurator is AlchemistCurator {
    constructor(address _admin, address _operator) 
        AlchemistCurator(_admin, _operator) 
    {}

    
    function getAdmin() external view returns (address) {
        return admin;  
    }

    function getPendingAdmin() external view returns (address) {
        return pendingAdmin;  
    }
}
```

2. add this test and run:

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import {MockAlchemistCurator} from "./mocks/MockAlchemistCurator.sol";

contract AcceptAdminOwnershipTest is Test {
    MockAlchemistCurator public curator;

    address public admin = address(0xA1);
    address public operator = address(0xB2); 
    address public pending = address(0xB1);
    address public rando = address(0xC1);

    event AdminChanged(address indexed newAdmin);

    function setUp() public {
        
        curator = new MockAlchemistCurator(admin, operator);
    }

    function test_acceptAdminOwnership_Success() public {
        vm.prank(admin);
        curator.transferAdminOwnerShip(pending);

        vm.prank(pending);
        vm.expectEmit(true, true, false, true);
        emit AdminChanged(pending);  
        curator.acceptAdminOwnership();

        assertEq(curator.getAdmin(), pending);
        assertEq(curator.getPendingAdmin(), address(0));
    }
}
```

And it will obviously fall under the current implementation, since it is impossible to accept the rights:

```
    ├─ [782] MockAlchemistCurator::acceptAdminOwnership()
    │   └─ ← [Revert] PD
```

If you replace the implementation with the correct one, the test passes.
