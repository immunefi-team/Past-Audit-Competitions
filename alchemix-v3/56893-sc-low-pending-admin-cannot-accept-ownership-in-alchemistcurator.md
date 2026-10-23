# 56893 sc low pending admin cannot accept ownership in alchemistcurator

**Submitted on Oct 21st 2025 at 15:15:57 UTC by @liae for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56893
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Pending admin cannot accept ownership in `AlchemistCurator` due to use of `onlyAdmin` modifier in `acceptAdminOwnership()` function.

## Vulnerability Details

`AlchemistCurator.sol:acceptAdminOwnership()` is intended to allow the `pendingAdmin` to accept ownership, but this is not possible due to the use of the `onlyAdmin` modifier:

```solidity
function acceptAdminOwnership() external onlyAdmin {
    ...
}
```

## Impact Details

`acceptAdminOwnership()` function does not work as intended, but there is no risk of fund loss.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistCurator.sol#L31>

## Proof of Concept

## Proof of Concept

Add this test case to `src/test/AlchemistCurator.t.sol`:

```solidity
function testAcceptOwnershipFailByPendingAdmin() public {
    address pendingAdmin = address(0x104);
    vm.startPrank(admin);
    mytCuratorProxy.transferAdminOwnerShip(pendingAdmin);
    vm.stopPrank();
    vm.startPrank(pendingAdmin);
    //@audit @note pendingAdmin can not accept ownership
    vm.expectRevert(abi.encode("PD"));
    mytCuratorProxy.acceptAdminOwnership();
    vm.stopPrank();
}
```

Run it with command:

```shell
forge test --mt testAcceptOwnershipFailByPendingAdmin
```
