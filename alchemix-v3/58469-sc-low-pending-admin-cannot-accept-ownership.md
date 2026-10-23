# 58469 sc low pending admin cannot accept ownership

**Submitted on Nov 2nd 2025 at 14:30:25 UTC by @ihtishamsudo for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58469
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The acceptAdminOwnership() function in AlchemistCurator.sol uses an incorrect access control modifier (onlyAdmin), preventing the intended new admin from accepting ownership transfer.

## Vulnerability Details

The acceptAdminOwnership() function is protected by the onlyAdmin modifier, which restricts access to only the current admin:

```solidity
function acceptAdminOwnership() external onlyAdmin {
    admin = pendingAdmin;
    pendingAdmin = address(0);
    emit AdminChanged(admin);
}
```

## Impact Details

The pendingAdmin cannot call acceptAdminOwnership() (reverts with "PD") Only the current admin can call this function

## References

The pendingAdmin cannot call acceptAdminOwnership() (reverts with "PD") Only the current admin can call this function The current admin can complete the transfer unilaterally without the new admin's consent

## Proof of Concept

## Proof of Concept

Include this test in AlchemistCurator.t.sol and execute `forge test --match-test "testAcceptAdminOwnershipBug" -vvv`

````solidity
function testAcceptAdminOwnershipBug() public {
        // Setup: Admin transfers ownership to a new admin
        address newAdmin = address(0x5555555555555555555555555555555555555555);
        
        vm.prank(admin);
        mytCuratorProxy.transferAdminOwnerShip(newAdmin);
        
        // Bug: pendingAdmin cannot accept ownership because of onlyAdmin modifier
        vm.prank(newAdmin);
        vm.expectRevert(abi.encode("PD")); // Will revert with "PD" (Permission Denied)
        mytCuratorProxy.acceptAdminOwnership();
        
    }

- **Test log**

```bash
Ran 1 test for src/test/AlchemistCurator.t.sol:AlchemistCuratorTest
[PASS] testAcceptAdminOwnershipBug() (gas: 40937)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 7.95ms (2.48ms CPU time)

Ran 1 test suite in 26.62ms (7.95ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
````
