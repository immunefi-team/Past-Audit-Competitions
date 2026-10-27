# 58520 sc low pending admin cannot accept ownership

**Submitted on Nov 2nd 2025 at 23:57:48 UTC by @acnologiac for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58520
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The acceptAdminOwnership() function in AlchemistCurator is implemented incorrectly, allowing the current admin to complete ownership transfers . This breaks the intended security of the two-step ownership transfer pattern, risking permanent loss of contract control, governance paralysis, and potential exploitation.

## Vulnerability Details

The AlchemistCurator contract implements a two-step admin ownership transfer mechanism intended to provide an extra layer of security by requiring the pending admin to actively accept ownership. The acceptAdminOwnership function uses the onlyAdmin modifier, which restricts execution to the current admin. However, in a proper two-step transfer pattern, the pending admin (the new admin receiving ownership) should be the one calling this function to accept ownership. This breaks the security model, as the new admin never has to prove control over their address or actively consent to taking ownership. As a result, the current admin can transfer control to any address, including incorrect or non-functional addresses, without any confirmation from the intended recipient.

The two-step transfer pattern exists to ensure the new admin must prove they control the address by signing a transaction and the new admin must actively accept, showing awareness of the responsibility. The broken implementation allows the current admin to unilaterally transfer ownership without any of these safeguards.

## Impact Details

This vulnerability introduces severe risks that can make the contract become permanently non-functional for all governance operations as the curator cannot add or remove strategies, transfer admin again or set operators.

## References

```solidity
function acceptAdminOwnership() external onlyAdmin {
    admin = pendingAdmin;
    pendingAdmin = address(0);
    emit AdminChanged(admin);
}

function transferAdminOwnerShip(address _newAdmin) external onlyAdmin {
        pendingAdmin = _newAdmin;
    }
```

## Proof of Concept

## Proof of Concept

The test should be ran in the AlchemistCurator.t.sol found in the test folder using this command; `forge test --match-test testPendingAdminCannotAccept -vv`

```solidity
function testPendingAdminCannotAccept() public {
    address newAdmin = address(0x6666666666666666666666666666666666666666);

    // Admin nominates newAdmin
    vm.startPrank(admin);
    mytCuratorProxy.transferAdminOwnerShip(newAdmin);
    vm.stopPrank();

    // pending (newAdmin) attempts to accept — should revert because onlyAdmin blocks it
    vm.startPrank(newAdmin);
    vm.expectRevert(abi.encode("PD")); // Permission Denied from onlyAdmin
    mytCuratorProxy.acceptAdminOwnership();
    vm.stopPrank();

    // Ensure nothing changed: admin still the original (read storage slot 0)
    address currentAdmin = address(uint160(uint256(vm.load(
        address(mytCuratorProxy),
        bytes32(uint256(0))
    ))));
    assertEq(currentAdmin, admin, "admin unexpectedly changed");
    // pendingAdmin still set
    assertEq(mytCuratorProxy.pendingAdmin(), newAdmin, "pendingAdmin unexpectedly cleared");
}
```
