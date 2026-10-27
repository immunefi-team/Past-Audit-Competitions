# 57127 sc low pending admin should call the function instead of admin&#x20;

**Submitted on Oct 23rd 2025 at 17:42:45 UTC by @SAAJ for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57127
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary

`pendingAdmin` will not be able to call `acceptAdminOwnership` due to `onlyAdmin` modifier.

## Vulnerability Details

`AlchemistCurator` function allows transferring of contract admin ownership through function `transferAdminOwnerShip`, the process is however a two step process. In the first step the `admin` call function with `_newAdmin` to set as `pendingAdmin`.

```
    function transferAdminOwnerShip(address _newAdmin) external onlyAdmin {
        pendingAdmin = _newAdmin;
    }
```

The second step involves accepting of ownership by the `pendingAdmin` which however restrict the call by `pendingAdmin` as it is callable only by `admin` due to `onlyAdmin` modifier.

```
    function acceptAdminOwnership() external onlyAdmin {
        admin = pendingAdmin;
        pendingAdmin = address(0);
        emit AdminChanged(admin);
    }
```

This enforces the authority to present `admin` to call both function and does not allow the `pendingAdmin` to call the `acceptAdminOwnership` function.

## Code Reference

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistCurator.sol#L31>

## Recommendations

The recommendation is made to remove the `onlyAdmin` modifier and implementing check that allows `pendingAdmin` to call the `acceptAdminOwnership` function.

```diff
-    function acceptAdminOwnership() external onlyAdmin {
+    function acceptAdminOwnership() external {
+       require(msg.sender == pendingAdmin, "PD");
        admin = pendingAdmin;
        pendingAdmin = address(0);
        emit AdminChanged(admin);
    }
```

## Proof of Concept

## Proof of Concept

Here is a test demonstrating the issue of restriction made making `pendingAdmin` to call the `acceptAdminOwnership` function.

First we call the function `transferAdminOwnerShip` with `admin making the call the transaction process normally, but when we try to call the` acceptAdminOwnership`function with the`pendingAdmin\`\`.

```
//  forge t --mc AlchemistCuratorTest --mt testTransferAdminOwnerShip -vvv
    function testTransferAdminOwnerShip() public {
        address pendingAdmin = makeAddr('NewAdmin');

        vm.startPrank(admin);
        mytCuratorProxy.transferAdminOwnerShip(pendingAdmin);
    }

//  forge t --mc AlchemistCuratorTest --mt testAcceptAdminOwnership -vvv
    function testAcceptAdminOwnership() public {
        testTransferAdminOwnerShip();

        address pendingAdmin = makeAddr('NewAdmin');

        vm.startPrank(pendingAdmin);
        mytCuratorProxy.acceptAdminOwnership();
    }
```

The result clearly shows the `testAcceptAdminOwnership` revert with reason `PD` showing `pendingAdmin` unable to call the function `acceptAdminOwnership` of contract `AlchemistCurator`.

```
Ran 1 test for src/test/AlchemistCurator.t.sol:AlchemistCuratorTest
[FAIL: PD] testAcceptAdminOwnership() (gas: 47192)
Traces:
  [47192] AlchemistCuratorTest::testAcceptAdminOwnership()
    ├─ [0] VM::addr(<pk>) [staticcall]
    │   └─ ← [Return] pendingAdmin: [0x2541e375f4cD6799E173d5f4968581ffb685e62F]
    ├─ [0] VM::label(pendingAdmin: [0x2541e375f4cD6799E173d5f4968581ffb685e62F], "NewAdmin")
    │   └─ ← [Return]
    ├─ [0] VM::startPrank(0x4444444444444444444444444444444444444444)
    │   └─ ← [Return]
    ├─ [25525] MockAlchemistCurator::transferAdminOwnerShip(pendingAdmin: [0x2541e375f4cD6799E173d5f4968581ffb685e62F])
    │   └─ ← [Return]
    ├─ [0] VM::addr(<pk>) [staticcall]
    │   └─ ← [Return] pendingAdmin: [0x2541e375f4cD6799E173d5f4968581ffb685e62F]
    ├─ [0] VM::label(pendingAdmin: [0x2541e375f4cD6799E173d5f4968581ffb685e62F], "NewAdmin")
    │   └─ ← [Return]
    ├─ [0] VM::startPrank(pendingAdmin: [0x2541e375f4cD6799E173d5f4968581ffb685e62F])
    │   └─ ← [Return]
    ├─ [782] MockAlchemistCurator::acceptAdminOwnership()
    │   └─ ← [Revert] PD
    └─ ← [Revert] PD

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 4.62ms (948.50µs CPU time)

Ran 1 test suite in 47.70ms (4.62ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in src/test/AlchemistCurator.t.sol:AlchemistCuratorTest
[FAIL: PD] testAcceptAdminOwnership() (gas: 47192)

Encountered a total of 1 failing tests, 0 tests succeeded
```
