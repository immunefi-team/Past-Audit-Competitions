# 58002 sc low missing submitremovestrategy function&#x20;

**Submitted on Oct 29th 2025 at 22:45:10 UTC by @Coachmike for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58002
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The issue is found in the removeStrategy(address adapter, address myt) function in the AlchemistCurator contract. This function is intended to remove a strategy (adapter) from the associated Morpho Vault V2 (MYT), but it fails to do so

## Vulnerability Details

The issue triggers in the line where removeStrategy () calls \_setStrategy(adapter, myt, true), which directly invokes vault.removeAdapter(adapter) without a prior submit call to queue the timelock. This causes a revert in VaultV2's timelocked() guard.

Walkthrough of the Issue

```solidity
function removeStrategy(address adapter, address myt) external onlyOperator {  // <-- Caller (operator) calls this 

    require(adapter != address(0), "INVALID_ADDRESS"); 

    require(myt != address(0), "INVALID_ADDRESS"); 

    _setStrategy(adapter, myt, true); // remove  // <-- Jumps directly to execution (skips submit!) 

} 
```

Function passes requires. Calls \_setStrategy(adapter, myt, true).

Unlike submitSetStrategy (for adds), there's no vault.submit(data) to queue the timelock.

```solidity

function _setStrategy(address adapter, address myt, bool remove) internal { 
    adapterToMYT[adapter] = myt;  // <-- Sets mapping (succeeds, but irrelevant to bug) 
    IVaultV2 vault = _vault(adapter);  // <-- Resolves to MYT vault (succeeds) 
    if (remove) { 
        vault.removeAdapter(adapter);  // <-- Calls VaultV2.removeAdapter(adapter) – BUG TRIGGERS HERE 
    } else { 
        vault.addAdapter(adapter); 
    } 
    emit StrategySet(adapter, myt);  // <-- Never reached due to revert 
} 

 

```

Inside VaultV2.sol: removeAdapter (Timelock Check Fails)

```solidity

function removeAdapter(address account) external {  // <-- Called as myt.removeAdapter(adapter) 
    timelocked();  // <-- INTERNAL CALL: This reverts! (See below) 
    if (isAdapter[account]) { 
        for (uint256 i = 0; i < adapters.length; i++) { 
            if (adapters[i] == account) { 
                adapters[i] = adapters[adapters.length - 1]; 
                adapters.pop(); 
                break; 
            } 
        } 
        isAdapter[account] = false; 
    } 
    emit EventsLib.RemoveAdapter(account);  // <-- Never reached 
} 

```

```solidity

function timelocked() internal {  // <-- Called by removeAdapter 
    bytes4 selector = bytes4(msg.data);  // <-- selector = removeAdapter.selector (e.g., 0x12345678) 
    require(executableAt[msg.data] != 0, ErrorsLib.DataNotTimelocked());  // <-- REVERTS HERE: executableAt[msg.data] == 0 
    require(block.timestamp >= executableAt[msg.data], ErrorsLib.TimelockNotExpired()); 
    require(!abdicated[selector], ErrorsLib.Abdicated()); 
    executableAt[msg.data] = 0;  // <-- Clears pending (never reached) 
    emit EventsLib.Accept(selector, msg.data);  // <-- Never reached 
} 

```

selector = bytes4(msg.data) extracts removeAdapter selector.

executableAt\[msg.data] checks for pending submission. Value: 0 (no prior submit(data) call was made for this exact calldata: abi.encodeCall(IVaultV2.removeAdapter, (adapter))).

Revert: require(executableAt\[msg.data] != 0, ErrorsLib.DataNotTimelocked()) fails with "DataNotTimelocked".

Reproduction Steps :

Deploy VaultV2 with curator set to the AlchemistCurator instance.

Set a timelock > 0 for removeAdapter.selector.

As operator, call removeStrategy(adapter, myt) for a valid added adapter.

Reverts with DataNotTimelocked().

## Soln

Add submitRemoveStrategy(address adapter, address myt) external onlyOperator mirroring submitSetStrategy, but using abi.encodeCall(IVaultV2.removeAdapter, adapter). Emit SubmitRemoveStrategy. Then removeStrategy can execute post-timelock like setStrategy.

```solidity

function submitRemoveStrategy(address adapter, address myt) external onlyOperator { 
    require(adapter != address(0), "INVALID_ADDRESS"); 
    require(myt != address(0), "INVALID_ADDRESS"); 
    bytes memory data = abi.encodeCall(IVaultV2.removeAdapter, adapter); 
    IVaultV2(vault).submit(data);   
 
    emit SubmitRemoveStrategy(adapter, myt);   
} 
```

## Impact Details

This bug prevents operators from removing adapters (strategies) from the vault, permanently locking in potentially risky or outdated allocations and exposing the vault to unmitigated security or inefficiency risks.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistCurator.sol#L49>

## Proof of Concept

## Proof of Concept

## POC

```solidity
// SPDX-License-Identifier: MIT 

pragma solidity 0.8.28; 

 

import {Test} from "forge-std/Test.sol"; 

import {VaultV2} from "../../lib/vault-v2/src/VaultV2.sol"; 

import {IVaultV2} from "../../lib/vault-v2/src/interfaces/IVaultV2.sol"; 

import {TestERC20} from "./mocks/TestERC20.sol"; 

import {MockYieldToken} from "./mocks/MockYieldToken.sol"; 

import {MockMYTStrategy} from "./mocks/MockMYTStrategy.sol"; 

import {MockAlchemistCurator} from "./mocks/MockAlchemistCurator.sol"; 

import {MYTTestHelper} from "./libraries/MYTTestHelper.sol"; 

import {IMYTStrategy} from "../interfaces/IMYTStrategy.sol"; 

 

/** 

 * @title RemoveStrategyBugTest 

 * @notice Demonstrates that removeStrategy() fails because it skips the required submit() step 

 * @dev This test proves the bug without any manipulation or mocking beyond the standard test setup 

 *  

 * RUN BOTH TESTS TO SEE THE BUG: 

 * 1. testRemoveStrategyFailsDueToMissingSubmit - Shows the bug (reverts with DataNotTimelocked) 

 * 2. testCorrectRemovalRequiresSubmitFirst - Shows the correct way (works when submit is called first) 

 */ 

contract RemoveStrategyBugTest is Test { 

    using MYTTestHelper for *; 

 

    MockAlchemistCurator public mytCuratorProxy; 

    VaultV2 public vault; 

    address public operator = address(0x2222222222222222222222222222222222222222); 

    address public admin = address(0x4444444444444444444444444444444444444444); 

    address public mockVaultCollateral; 

    address public mockStrategyYieldToken; 

    MockMYTStrategy public mytStrategy; 

 

    function setUp() public { 

        mockVaultCollateral = address(new TestERC20(100e18, uint8(18))); 

        mockStrategyYieldToken = address(new MockYieldToken(mockVaultCollateral)); 

 

        vm.startPrank(admin); 

        mytCuratorProxy = new MockAlchemistCurator(admin, operator); 

        vault = MYTTestHelper._setupVault(mockVaultCollateral, admin, address(mytCuratorProxy)); 

        mytStrategy = MYTTestHelper._setupStrategy( 

            address(vault),  

            mockStrategyYieldToken,  

            admin,  

            "MockToken",  

            "MockTokenProtocol",  

            IMYTStrategy.RiskClass.LOW 

        ); 

        vm.stopPrank(); 

         

        // Set a timelock for removeAdapter to enforce the timelock requirement 

        // The curator (mytCuratorProxy) must be the one to submit 

        vm.startPrank(address(mytCuratorProxy)); 

        bytes memory timelockData = abi.encodeCall( 

            IVaultV2.increaseTimelock,  

            (IVaultV2.removeAdapter.selector, 1 days) 

        ); 

        vault.submit(timelockData); 

        vm.warp(block.timestamp + vault.timelock(IVaultV2.increaseTimelock.selector)); 

        vault.increaseTimelock(IVaultV2.removeAdapter.selector, 1 days); 

        vm.stopPrank(); 

    } 

 

    /** 

     * @notice This test demonstrates the bug: removeStrategy reverts with "DataNotTimelocked" 

     * @dev Steps: 

     *      1. Operator successfully adds strategy via submitSetStrategy + setStrategy (works) 

     *      2. Operator attempts to remove strategy via removeStrategy (FAILS) 

     *      3. The failure occurs because removeStrategy calls vault.removeAdapter() directly 

     *         without first calling vault.submit() to queue the timelock 

     */ 

    function testRemoveStrategyFailsDueToMissingSubmit() public { 

        // Step 1: Successfully add the strategy (this works correctly) 

        vm.startPrank(operator); 

         

        // Submit the add operation (queues timelock) 

        mytCuratorProxy.submitSetStrategy(address(mytStrategy), address(vault)); 

         

        // Fast forward past the timelock 

        bytes memory addData = abi.encodeCall(IVaultV2.addAdapter, address(mytStrategy)); 

        bytes4 addSelector = bytes4(addData); 

        vm.warp(block.timestamp + vault.timelock(addSelector)); 

         

        // Execute the add operation (completes successfully) 

        mytCuratorProxy.setStrategy(address(mytStrategy), address(vault)); 

         

        // Verify strategy was added 

        assertTrue(vault.isAdapter(address(mytStrategy)), "Strategy should be added"); 

         

        // Step 2: Attempt to remove the strategy - THIS WILL REVERT 

        // The bug: removeStrategy() calls vault.removeAdapter() directly without submit() 

        // Expected revert: "DataNotTimelocked" from VaultV2.timelocked() 

        vm.expectRevert(); 

        mytCuratorProxy.removeStrategy(address(mytStrategy), address(vault)); 

         

        vm.stopPrank(); 

         

        // Verify strategy is still in vault (removal failed) 

        assertTrue(vault.isAdapter(address(mytStrategy)), "Strategy still present after failed removal"); 

    } 

 

   

} 

```

## POC 2

```solidity
// SPDX-License-Identifier: MIT 

pragma solidity 0.8.28; 

 

import {Test} from "forge-std/Test.sol"; 

import {VaultV2} from "../../lib/vault-v2/src/VaultV2.sol"; 

import {ERC20Mock} from "../../lib/vault-v2/test/mocks/ERC20Mock.sol"; 

import {IVaultV2} from "../../lib/vault-v2/src/interfaces/IVaultV2.sol"; 

import {TestYieldToken} from "./mocks/TestYieldToken.sol"; 

import {TestERC20} from "./mocks/TestERC20.sol"; 

import {TokenUtils} from "../libraries/TokenUtils.sol"; 

import {MockYieldToken} from "./mocks/MockYieldToken.sol"; 

import {IMockYieldToken} from "./mocks/MockYieldToken.sol"; 

import {MockMYTStrategy} from "./mocks/MockMYTStrategy.sol"; 

import {MYTTestHelper} from "./libraries/MYTTestHelper.sol"; 

import {IMYTStrategy} from "../interfaces/IMYTStrategy.sol"; 

import {AlchemistCurator} from "../AlchemistCurator.sol"; 

import {ErrorsLib} from "../../lib/vault-v2/src/libraries/ErrorsLib.sol"; 

 

contract AlchemistCuratorTest is Test { 

    using MYTTestHelper for *; 

 

    AlchemistCurator public mytCurator; 

    VaultV2 public vault; 

    address public operator = address(0x2222222222222222222222222222222222222222); // default operator 

    address public admin = address(0x4444444444444444444444444444444444444444); // DAO OSX 

    address public mockVaultCollateral = address(new TestERC20(100e18, uint8(18))); 

    address public mockStrategyYieldToken = address(new MockYieldToken(mockVaultCollateral)); 

    uint256 public defaultStrategyAbsoluteCap = 200 ether; 

    uint256 public defaultStrategyRelativeCap = 1e18; // 100% 

    MockMYTStrategy public mytStrategy; 

 

    function setUp() public { 

        vm.startPrank(admin); 

        mytCurator = new AlchemistCurator(admin, operator); 

        vault = MYTTestHelper._setupVault(mockVaultCollateral, admin, address(mytCurator)); 

        mytStrategy = MYTTestHelper._setupStrategy(address(vault), mockStrategyYieldToken, admin, "MockToken", "MockTokenProtocol", IMYTStrategy.RiskClass.LOW); 

        vm.stopPrank(); 

    } 

 

    function testRemoveStrategyRevertsDueToMissingSubmitAndTimelock() public { 

        // Arrange: Submit and set strategy (add adapter) successfully to establish valid state for removal attempt 

        vm.startPrank(operator); 

        mytCurator.submitSetStrategy(address(mytStrategy), address(vault)); 

 

        // Fast forward timelock for addAdapter 

        bytes4 addAdapterSelector = IVaultV2.addAdapter.selector; 

        uint256 timelockDuration = vault.timelock(addAdapterSelector); 

        vm.warp(block.timestamp + timelockDuration); 

 

        mytCurator.setStrategy(address(mytStrategy), address(vault)); 

        vm.stopPrank(); 

 

        // Verify adapter is added (isAdapter true in vault) 

        assertTrue(vault.isAdapter(address(mytStrategy))); 

 

        // Act & Assert: Attempt removal directly without prior submit (timelock), expect revert from VaultV2.timelocked() 

        // This demonstrates the bug: removeStrategy calls vault.removeAdapter directly, skipping submit/data queuing, 

        // causing executableAt[msg.data] == 0 and revert with "DataNotTimelocked", preventing strategy removal 

        // Impact: Operators cannot remove risky/outdated strategies, locking vault allocations permanently 

        vm.startPrank(operator); 

        vm.expectRevert(ErrorsLib.DataNotTimelocked.selector); 

        mytCurator.removeStrategy(address(mytStrategy), address(vault)); 

        vm.stopPrank(); 

 

        // Verify adapter remains added (removal failed as expected) 

        assertTrue(vault.isAdapter(address(mytStrategy))); 

    } 

} 
```
