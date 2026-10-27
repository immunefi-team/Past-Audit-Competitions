# 58007 sc low pendingadmin cannot call acceptadminownership to accept admin role

**Submitted on Oct 29th 2025 at 23:36:50 UTC by @kaysoft for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58007
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The AlchemistCurator.sol implements 2 step admin transfer. The issue is that after the old admin calls `transferAdminOwnerShip(...)` to transfer the `admin` role, the `pendingAdmin` cannot call the `acceptAdminOwnership()` because `acceptAdminOwnership()` is still access gated with the old admin `onlyOwner` modifier.

## Vulnerability Details

The `onlyOwner` modifier is used as access control to the `acceptAdminOwnership()` function.

This only allows the old admin as the caller of the `acceptAdminOwnership()` function instead of the pendingAdmin.

```solidity
File: AlchemistCurator.sol
function transferAdminOwnerShip(address _newAdmin) external onlyAdmin {
        pendingAdmin = _newAdmin;//@audit old admin transfer to pendingAdmin
    }

    function acceptAdminOwnership() external onlyAdmin {//@audit this is supposed to be called by onlyPendingAdmin(new admin) not onlyAdmin(old admin)
        admin = pendingAdmin;
        pendingAdmin = address(0);
        emit AdminChanged(admin);
    }

```

## Impact Details

* The pendingAdmin that's supposed to accept the admin role as the new admin cannot accept it

## Recommendation

Consider allowing the `pendingAdmin` call the `acceptAdminOwnership()` to accept the admin role instead of the old admin this way.

```diff
-- function acceptAdminOwnership() external onlyAdmin {
++ function acceptAdminOwnership() external  {
++    require(msg.sender == pendingAdmin, "Not pendingAdmin");
        admin = pendingAdmin;
        pendingAdmin = address(0);
        emit AdminChanged(admin);
    }
```

## Proof of Concept

## Proof of Concept

1. Copy and paste the test below to the `src/test/AlchemistCurator.t.sol:AlchemistCuratorTest` test contract
2. Run `forge test --match-test test_pendingAdmin_Cannot_Accept -vvv`
3. This test demonstrates that when pendingAdmin try to call `acceptAdminOwnership(..)` to accept the admin role, the transaction reverts.

```solidity
function test_pendingAdmin_Cannot_Accept() public {
        address alice = makeAddr("AliceJames");

        //1. Old admin transfers admin role to pendingAdmin:alice
        vm.prank(admin);
        mytCuratorProxy.transferAdminOwnerShip(alice);


        //2. Alice who is the pendingAdmin calls acceptAdminOwnership() but it reverts
        vm.prank(alice);
        vm.expectRevert();
        mytCuratorProxy.acceptAdminOwnership();
    }
```
