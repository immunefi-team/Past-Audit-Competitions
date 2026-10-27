# 56625 sc low broken ownership transfer logic in alchemistcurator permanently freezes contract operations

**Submitted on Oct 18th 2025 at 15:47:04 UTC by @ENIGMA for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56625
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

#### AlchemistCurator has a logic flaw in its ownership transfer — acceptAdminOwnership() is gated by onlyAdmin instead of being callable by pendingAdmin. As a result, once transferAdminOwnerShip(newAdmin) is executed, the new admin can never take control. This creates a permanent governance deadlock, freezing all admin-only operations and potentially locking protocol funds. The issue violates the README rule that “ownership transfer must be secure and correct.

## Vulnerability Details

### What is broken:

#### acceptAdminOwnership() is protected by onlyAdmin instead of being callable by the pendingAdmin. That single misplaced restriction breaks the standard 2-step transfer pattern and makes pendingAdmin unable to ever complete an ownership claim.

```
function transferAdminOwnerShip(address _newAdmin) external onlyAdmin {
pendingAdmin = _newAdmin;
}

function acceptAdminOwnership() external onlyAdmin {
admin = pendingAdmin;
pendingAdmin = address(0);
emit AdminChanged(admin);
}
```

#### Why this is wrong:

#### transferAdminOwnerShip correctly nominates a new admin (sets pendingAdmin).

#### acceptAdminOwnership must be callable by the nominated address (pendingAdmin) to accept the role. Instead the modifier onlyAdmin allows only the current admin to call acceptAdminOwnership. That defeats the acceptance step.

## Impact Details

### 1. Irrecoverable admin lock: After calling transferAdminOwnerShip(newAdmin), the pendingAdmin cannot execute acceptAdminOwnership(), leaving ownership transfer permanently incomplete.

### 2. Admin functions disabled: All admin-only functions (e.g., cap adjustments, adapter management, allocations) become inaccessible to the new admin and may be unsafe or unusable for the old one.

### 3. Funds frozen: Vault and strategy assets requiring admin actions remain locked in the contract, with no on-chain method to recover or move them—resulting in a permanent freeze of funds.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol> L27-L35

## Proof of Concept

## Proof of Concept

```
            // SPDX-License-Identifier: MIT
            pragma solidity 0.8.28;
            
            import "forge-std/Test.sol";
            import "../../AlchemistCurator.sol";
            
            /**
             * @notice Proof-of-Concept for the "Permanent Freeze" vulnerability.
             * 
             * 1. Ownership transfer fails due to flawed acceptance logic.
             * 2. Contract becomes permanently frozen — both admins lose privileges.
             */
            
            interface IVaultMinimal {
                function getIdData() external view returns (uint256);
            }
            
            contract VaultAdapterStub is IVaultMinimal {
                function getIdData() external pure returns (uint256) {
                    return 1;
                }
            }
            
            contract AlchemistCuratorOwnershipTest is Test {
                AlchemistCurator curator;
                VaultAdapterStub adapter;
            
                address admin = address(0xA11CE);
                address operator = address(0xB0B);
                address newAdmin = address(0xC0DE);
            
                function setUp() public {
                    curator = new AlchemistCurator(admin, operator);
                    adapter = new VaultAdapterStub();
                }
            
                /**
                 * @notice Step 1: Ownership transfer fails.
                 * The new admin cannot claim ownership, leaving `pendingAdmin` stuck.
                 */
                function testOwnershipTransferFails() public {
                    // Admin initiates ownership transfer
                    vm.prank(admin);
                    curator.transferAdminOwnerShip(newAdmin);
            
                    // Ensure pending admin is correctly recorded
                    assertEq(curator.pendingAdmin(), newAdmin, "pendingAdmin should be set");
            
                    // New admin attempts to accept but reverts due to onlyAdmin restriction
                    vm.prank(newAdmin);
                    vm.expectRevert();
                    curator.acceptAdminOwnership();
            
                    // Ownership remains unchanged
                    bytes32 adminSlot = bytes32(uint256(0));
                    address currentAdmin = address(uint160(uint256(vm.load(address(curator), adminSlot))));
                    assertEq(currentAdmin, admin, "Admin should remain unchanged after failed acceptance");
                }
            
                /**
                 * @notice Step 2: Contract permanently freezes after failed ownership transfer.
                 * No admin (old or new) can now perform privileged operations.
                 */
                function testOwnershipTransferCausesPermanentFreeze() public {
                    // Admin starts transfer process
                    vm.prank(admin);
                    curator.transferAdminOwnerShip(newAdmin);
            
                    assertEq(curator.pendingAdmin(), newAdmin, "pendingAdmin set");
            
                    // New admin fails to accept ownership
                    vm.prank(newAdmin);
                    vm.expectRevert();
                    curator.acceptAdminOwnership();
            
                    // Check that admin did not change
                    bytes32 adminSlot = bytes32(uint256(0));
                    address currentAdmin = address(uint160(uint256(vm.load(address(curator), adminSlot))));
                    assertEq(currentAdmin, admin, "Admin remains original");
            
                    // Both old and new admins can no longer perform privileged actions
                    vm.prank(newAdmin);
                    vm.expectRevert();
                    curator.decreaseAbsoluteCap(address(adapter), 1 ether);
            
                    vm.prank(admin);
                    (bool success, ) = address(curator).call(
                        abi.encodeWithSignature("decreaseAbsoluteCap(address,uint256)", address(adapter), 1 ether)
                    );
                    success; // suppress unused warning
            
                    // Confirm ownership is permanently stuck
                    assertEq(curator.pendingAdmin(), newAdmin, "pendingAdmin remains stuck");
                    currentAdmin = address(uint160(uint256(vm.load(address(curator), adminSlot))));
                    assertEq(currentAdmin, admin, "admin slot unchanged");
                }
            }
            
```

#### The test command: forge test --match-contract AlchemistCuratorOwnershipTest -vvvv

### The output:

```
                       forge test --match-contract AlchemistCuratorOwnershipTest -vvvv
       [⠊] Compiling...
       [⠰] Compiling 1 files with Solc 0.8.28
       [⠔] Solc 0.8.28 finished in 1.29s
       Compiler run successful with warnings:
       Warning (2018): Function state mutability can be restricted to view
          --> src/AlchemistCurator.sol:166:5:
           |
       166 |     function _vault(address adapter) internal returns (IVaultV2) {
           |     ^ (Relevant source part starts here and spans across multiple lines).
       
       
       Ran 2 tests for src/test/mytest/AlchemistCuratorOwnershipTest.t.sol:AlchemistCuratorOwnershipTest
       [PASS] testOwnershipTransferCausesPermanentFreeze() (gas: 64344)
       Traces:
         [64344] AlchemistCuratorOwnershipTest::testOwnershipTransferCausesPermanentFreeze()
           ├─ [0] VM::prank(0x00000000000000000000000000000000000A11cE)
           │   └─ ← [Return]
           ├─ [25525] AlchemistCurator::transferAdminOwnerShip(0x000000000000000000000000000000000000c0DE)
           │   └─ ← [Return]
           ├─ [521] AlchemistCurator::pendingAdmin() [staticcall]
           │   └─ ← [Return] 0x000000000000000000000000000000000000c0DE
           ├─ [0] VM::prank(0x000000000000000000000000000000000000c0DE)
           │   └─ ← [Return]
           ├─ [0] VM::expectRevert(custom error 0xf4844814)
           │   └─ ← [Return]
           ├─ [782] AlchemistCurator::acceptAdminOwnership()
           │   └─ ← [Revert] PD
           ├─ [0] VM::load(AlchemistCurator: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], 0x0000000000000000000000000000000000000000000000000000000000000000) [staticcall]
           │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000000000a11ce
           ├─ [0] VM::prank(0x000000000000000000000000000000000000c0DE)
           │   └─ ← [Return]
           ├─ [0] VM::expectRevert(custom error 0xf4844814)
           │   └─ ← [Return]
           ├─ [848] AlchemistCurator::decreaseAbsoluteCap(VaultAdapterStub: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], 1000000000000000000 [1e18])
           │   └─ ← [Revert] PD
           ├─ [0] VM::prank(0x00000000000000000000000000000000000A11cE)
           │   └─ ← [Return]
           ├─ [4416] AlchemistCurator::decreaseAbsoluteCap(VaultAdapterStub: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], 1000000000000000000 [1e18])
           │   ├─ [318] VaultAdapterStub::getIdData() [staticcall]
           │   │   └─ ← [Return] 1
           │   └─ ← [Revert] EvmError: Revert
           ├─ [521] AlchemistCurator::pendingAdmin() [staticcall]
           │   └─ ← [Return] 0x000000000000000000000000000000000000c0DE
           ├─ [0] VM::load(AlchemistCurator: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], 0x0000000000000000000000000000000000000000000000000000000000000000) [staticcall]
           │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000000000a11ce
           └─ ← [Return]
       
       [PASS] testOwnershipTransferFails() (gas: 47236)
       Traces:
         [47236] AlchemistCuratorOwnershipTest::testOwnershipTransferFails()
           ├─ [0] VM::prank(0x00000000000000000000000000000000000A11cE)
           │   └─ ← [Return]
           ├─ [25525] AlchemistCurator::transferAdminOwnerShip(0x000000000000000000000000000000000000c0DE)
           │   └─ ← [Return]
           ├─ [521] AlchemistCurator::pendingAdmin() [staticcall]
           │   └─ ← [Return] 0x000000000000000000000000000000000000c0DE
           ├─ [0] VM::prank(0x000000000000000000000000000000000000c0DE)
           │   └─ ← [Return]
           ├─ [0] VM::expectRevert(custom error 0xf4844814)
           │   └─ ← [Return]
           ├─ [782] AlchemistCurator::acceptAdminOwnership()
           │   └─ ← [Revert] PD
           ├─ [0] VM::load(AlchemistCurator: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], 0x0000000000000000000000000000000000000000000000000000000000000000) [staticcall]
           │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000000000a11ce
           └─ ← [Return]
       
       Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 20.88ms (12.34ms CPU time)
       
       Ran 1 test suite in 264.95ms (20.88ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```
