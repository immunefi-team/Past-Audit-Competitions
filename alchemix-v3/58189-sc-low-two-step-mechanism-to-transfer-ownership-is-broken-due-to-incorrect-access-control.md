# 58189 sc low two step mechanism to transfer ownership is broken due to incorrect access control

**Submitted on Oct 31st 2025 at 09:24:19 UTC by @KKam86 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58189
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Implementation of two-step mechanism to transfer ownership in `AlchemistCurator` contract doesn't work as intended because function `acceptAdminOwnership()` have access control modifier `onlyAdmin`. In result only current admin can call `acceptAdminOwnership()`. This function should be called by new pending admin to accept new privileges.

## Vulnerability Details

Two-step mechanism to transfer ownership consists of two functions which must be called sequentially in correct order:

1. Current admin transfers his privileges (ownership) to another address by calling `transferAdminOwnerShip()`. This new address is assigned to `pendingAdmin` variable. This address must accept new privileges.
2. New pending admin must accept ownership by making a call to `acceptAdminOwnership()`. This function should be reserved only to `pendingAdmin`.

However second step of transfering ownership process in `AlchemistCurator` is wrongly implemented. Function `acceptAdminOwnership()` have access control modifier `onlyAdmin` and in result only current admin can call this function:

```solidity
    modifier onlyAdmin() {
        require(msg.sender == admin, "PD"); // admin is the current admin in AlchemistCurator contract
        _;
    }
```

As an example, OpenZeppelin's `Ownable2Step` contract implements two-step mechanism for transfering ownership in the following, correct way:

1. Current owner calls `transferOwnership()` and propose new pending admin:

```solidity

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
```

2. New pending admin accepts the ownership transfer by making a call to `acceptOwnership()`:

```solidity

    /**
     * @dev The new owner accepts the ownership transfer.
     */
    function acceptOwnership() public virtual {
        address sender = _msgSender();
    >>  if (pendingOwner() != sender) {
            revert OwnableUnauthorizedAccount(sender);
        }
        _transferOwnership(sender);
    }


    /**
     * @dev Transfers ownership of the contract to a new account (`newOwner`) and deletes any pending owner.
     * Internal function without access restriction.
     */
    function _transferOwnership(address newOwner) internal virtual override {
        delete _pendingOwner;
        super._transferOwnership(newOwner);
    }
```

Also `transferOwnership()` function emit additional event `OwnershipTransferStarted` to signal the begining of ownership transfer process.

## Impact Details

1. No protection against common mistakes, such as transfers of ownership to incorrect accounts, or to contracts that are unable to interact with the permission system.
2. Risk of permanent DOS. If current admin accepts wrong address (which is unable to interact with `AlchemistCurator` contract) by mistake, then all functions with `onlyAdmin` modifier will always revert. There is no way to undo the changes (only redeploying the new `AlchemistCurator` contract).

## References

<https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable2Step.sol>

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol?utm\\_source=immunefi#L31>

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/PermissionedProxy.sol#L17-L20>

## Proof of Concept

## Proof of Concept

Place following test in `AlchemistCurator.t.sol` file:

```solidity
    function testAcceptAdminOwnershipPendingAdminAccessRevert() public {
        address newAdmin = makeAddr("newAdmin");
        vm.prank(admin);
        mytCuratorProxy.transferAdminOwnerShip(newAdmin);
        assertEq(mytCuratorProxy.pendingAdmin(), newAdmin);
        // New pending admin tries to accept ownership
        vm.startPrank(newAdmin);
        vm.expectRevert(abi.encode("PD"));
        mytCuratorProxy.acceptAdminOwnership();
        vm.stopPrank();
    }
```

Output:

```solidity
Ran 1 test for src/test/AlchemistCurator.t.sol:AlchemistCuratorTest
[PASS] testAcceptAdminOwnershipPendingAdminAccessRevert() (gas: 47377)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 145.55ms (21.00ms CPU time)
```
