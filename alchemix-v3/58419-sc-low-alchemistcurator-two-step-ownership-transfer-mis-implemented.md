# 58419 sc low alchemistcurator two step ownership transfer mis implemented

**Submitted on Nov 2nd 2025 at 07:13:54 UTC by @Y4nhu1 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58419
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**

## Description

## Brief/Intro

Using only the `onlyAdmin` modifier in a two-step ownership transfer could result in setting the admin to an unavailable account.

## Vulnerability Details

The `acceptAdminOwnership` function in the AlchemistCurator contract prevents pending admin acceptance by using the modifier `onlyAdmin` instead of ensuring that `msg.sender` matches the `pendingAdmin`.

```solidity
function acceptAdminOwnership() external onlyAdmin {
    admin = pendingAdmin;
    pendingAdmin = address(0);
    emit AdminChanged(admin);
}
```

## Impact Details

The advantage of the two-step ownership transfer is that it can prevent the owner from being set to an incorrect account. However, if only the admin is allowed to call function `acceptAdminOwnership`, this advantage is lost, as the admin could be set to an unavailable address that can’t interact with this contract anymore.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistCurator.sol#L31>

## Proof of Concept

## Proof of Concept

`forge test --mt "testAcceptAdminOwnershipAccessControl" -vvv`

```diff
diff --git a/src/test/AlchemistCurator.t.sol b/src/test/AlchemistCurator.t.sol
index a21c4c4..bb92c2a 100644
--- a/src/test/AlchemistCurator.t.sol
+++ b/src/test/AlchemistCurator.t.sol
@@ -191,6 +191,21 @@ contract AlchemistCuratorTest is Test {
         mytCuratorProxy.acceptAdminOwnership();
     }
 
+    function testAcceptAdminOwnershipAccessControl() public {
+        address pendingAdmin = makeAddr("pendingAdmin");
+
+        vm.prank(admin);
+        mytCuratorProxy.transferAdminOwnerShip(pendingAdmin);
+
+        // pending admin can't accept admin ownership
+        vm.prank(pendingAdmin);
+        vm.expectRevert(abi.encode("PD"));
+        mytCuratorProxy.acceptAdminOwnership();
+
+        vm.prank(admin);
+        mytCuratorProxy.acceptAdminOwnership();
+    }
+
     function testSetStrategyUnauthorizedAccessRevert() public {
         vm.expectRevert(abi.encode("PD"));
         mytCuratorProxy.setStrategy(address(mytStrategy), address(vault));
```
