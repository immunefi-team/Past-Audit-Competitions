# 56830 sc low broken admin ownership transfer logic acceptadminownership requires current admin instead of pending admin blocking role claim&#x20;

**Submitted on Oct 21st 2025 at 04:04:05 UTC by @akioniace for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56830
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

**AlchemistCurator::acceptAdminOwnership()** is incorrectly restricted by `onlyAdmin` modifier. After the current Admin calls `transferAdminOwnership(newAdmin)`, the `pendingAdmin` cannot call `acceptAdminOwnership()` to complete the transfer because accept function require caller to already to be as current Admin. As implemented, only the current admin can accept the pending admin, which defeats the two step handover flow.

## Vulnerability Details

The Root cause is that **AlchemistCurator::acceptAdminOwnership()** function is declared as:

```solidity
// FILE: https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/AlchemistCurator.sol?utm_source=immunefi#L31-L35

    function acceptAdminOwnership() external onlyAdmin { // @audit-issue : current admin, should be pendingAdmin
        admin = pendingAdmin;
        pendingAdmin = address(0);
        emit AdminChanged(admin);
    }
```

Because of `onlyAdmin`, only the existing `admin` can call **acceptAdminOwnersup()**. The intended two-step transfer pattern requires the `pendingAdmin` to be able to opt in and claim the role. The **correct guard** is that caller must be equal `pendingAdmin`, not `admin`.

## Impact Details

The acceptAdminOwnership() function is incorrectly restricted to the current admin, preventing the pending admin from claiming ownership. This breaks the two-step ownership transfer logic.

## References

To achieve two step ownership look at the openzeppelin Ownable2Step process: <https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable2Step.sol#L60-L67>

Broken Logic Code Link - <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol?utm\\_source=immunefi#L31-L35>

## Proof of Concept

## Proof of Concept

Copy Paste this function into **v3-poc/src/test/AlchemistCurator.t.sol**:

```solidity
    function testPendingAdminCannotAcceptOwnership() public {
        address newAdmin = makeAddr("Pending Trusted Admin"); // new trusted admin

        vm.startPrank(admin);
        mytCuratorProxy.transferAdminOwnerShip(newAdmin); // set pending admin 
        assertEq(mytCuratorProxy.pendingAdmin(), newAdmin);
        console.log("///////////////////////////////////////////////");
        console.log("Pending Admin: ", mytCuratorProxy.pendingAdmin());
        console.log("///////////////////////////////////////////////");
        vm.stopPrank();

        vm.startPrank(newAdmin); // pending admin tries to accept ownership
        vm.expectRevert(abi.encode("PD"));
        mytCuratorProxy.acceptAdminOwnership(); // this will fail
        vm.stopPrank();
    }
```

run with command:

```cmd
forge test --match-path src/test/AlchemistCurator.t.sol --match-test testPendingAdminCannotAcceptOwnership -vvvv
```

**Here is the Logs From the Test:**

```cmd
Ran 1 test for src/test/AlchemistCurator.t.sol:AlchemistCuratorTest
[PASS] testPendingAdminCannotAcceptOwnership() (gas: 55607)
Logs:
  ///////////////////////////////////////////////
  Pending Admin:  0x8f8E50A4614F9E558A5897b3B0AA8A6FEDb9d391
  ///////////////////////////////////////////////

Traces:
  [55607] AlchemistCuratorTest::testPendingAdminCannotAcceptOwnership()
    ├─ [0] VM::addr(<pk>) [staticcall]
    │   └─ ← [Return] Pending Trusted Admin: [0x8f8E50A4614F9E558A5897b3B0AA8A6FEDb9d391]
    ├─ [0] VM::label(Pending Trusted Admin: [0x8f8E50A4614F9E558A5897b3B0AA8A6FEDb9d391], "Pending Trusted Admin")
    │   └─ ← [Return]
    ├─ [0] VM::startPrank(0x4444444444444444444444444444444444444444)
    │   └─ ← [Return]
    ├─ [25525] MockAlchemistCurator::transferAdminOwnerShip(Pending Trusted Admin: [0x8f8E50A4614F9E558A5897b3B0AA8A6FEDb9d391])
    │   └─ ← [Return]
    ├─ [521] MockAlchemistCurator::pendingAdmin() [staticcall]
    │   └─ ← [Return] Pending Trusted Admin: [0x8f8E50A4614F9E558A5897b3B0AA8A6FEDb9d391]
    ├─ [0] console::log("///////////////////////////////////////////////") [staticcall]
    │   └─ ← [Stop]
    ├─ [521] MockAlchemistCurator::pendingAdmin() [staticcall]
    │   └─ ← [Return] Pending Trusted Admin: [0x8f8E50A4614F9E558A5897b3B0AA8A6FEDb9d391]
    ├─ [0] console::log("Pending Admin: ", Pending Trusted Admin: [0x8f8E50A4614F9E558A5897b3B0AA8A6FEDb9d391]) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("///////////////////////////////////////////////") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [0] VM::startPrank(Pending Trusted Admin: [0x8f8E50A4614F9E558A5897b3B0AA8A6FEDb9d391])
    │   └─ ← [Return]
    ├─ [0] VM::expectRevert(custom error 0xf28dceb3:  ` PD)
    │   └─ ← [Return]
    ├─ [782] MockAlchemistCurator::acceptAdminOwnership()
    │   └─ ← [Revert] PD
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    └─ ← [Stop]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.02ms (952.28µs CPU time)

Ran 1 test suite in 17.19ms (3.02ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
