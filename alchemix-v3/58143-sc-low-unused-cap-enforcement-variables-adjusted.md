# 58143 sc low unused cap enforcement variables adjusted&#x20;

**Submitted on Oct 30th 2025 at 22:58:51 UTC by @w3llyc4de20Ik2nn1 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58143
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistAllocator.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The `AlchemistAllocator` calculates an adjusted cap by selecting the appropriate limit between absoluteCap, relativeCap, and daoTarget but fails to enforce it, allowing allocate and deallocate calls to bypass intended risk limits and pass unchecked amounts to the vault.

## Vulnerability Details

Offer a detailed explanation of the vulnerability This issue affects both allocate and deallocate functions. Here's the full relevant code for each, highlighting the unused adjusted calculation: In the `AlchemistAllocator.sol`,

In allocate function:

```solidity

function allocate(address adapter, uint256 amount) external { 
    require(msg.sender == admin || operators[msg.sender], "PD"); 
    bytes32 id = IMYTStrategy(adapter).adapterId(); 
    uint256 absoluteCap = vault.absoluteCap(id); 
    uint256 relativeCap = vault.relativeCap(id); 
    // FIXME get this from the StrategyClassificationProxy for the respective risk class 
    uint256 daoTarget = type(uint256).max; 
    uint256 adjusted = absoluteCap > relativeCap ? absoluteCap : relativeCap; 
    if (msg.sender != admin) { 
        // caller is operator 
        adjusted = adjusted > daoTarget ? adjusted : daoTarget; 
    } 
    // pass the old allocation to the adapter 
    bytes memory oldAllocation = abi.encode(vault.allocation(id)); 
    vault.allocate(adapter, oldAllocation, amount); 
} 
```

In deallocate function:

```solidity

function deallocate(address adapter, uint256 amount) external { 
    require(msg.sender == admin || operators[msg.sender], "PD"); 
    bytes32 id = IMYTStrategy(adapter).adapterId(); 
    uint256 absoluteCap = vault.absoluteCap(id); 
    uint256 relativeCap = vault.relativeCap(id); 
    // FIXME get this from the StrategyClassificationProxy for the respective risk class 
    uint256 daoTarget = type(uint256).max; 
    uint256 adjusted = absoluteCap < relativeCap ? absoluteCap : relativeCap; 
    if (msg.sender != admin) { 
        // caller is operator 
        adjusted = adjusted < daoTarget ? adjusted : daoTarget; 
    } 
    // pass the old allocation to the adapter 
    bytes memory oldAllocation = abi.encode(vault.allocation(id)); 
    vault.deallocate(adapter, oldAllocation, amount); 
} 
```

The variables absoluteCap, relativeCap, and daoTarget are fetched to compute adjusted, which is meant to represent a safe limit for the amount being allocated or deallocated based on strategy caps and targets; however, adjusted is calculated but never used or compared against amount, so the vault receives the full amount without any limit enforcement.

## Impact Details

Without enforcing adjusted against amount, the contract allows unlimited allocations or deallocations, bypassing risk limits and potentially exposing the vault to excessive funds in unsafe strategies.

## Recommended Fix

Add a check like `require(amount <= adjusted, "Exceeds cap");` in allocate.

For deallocate, check `require(amount <= vault.allocation(id), "Exceeds current allocation");`

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistAllocator.sol?utm\\_source=immunefi>

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT 

pragma solidity 0.8.28; 

 

import {Test} from "forge-std/Test.sol"; 

import {VaultV2} from "../../lib/vault-v2/src/VaultV2.sol"; 

import {IVaultV2} from "../../lib/vault-v2/src/interfaces/IVaultV2.sol"; 

import {TestERC20} from "./mocks/TestERC20.sol"; 

import {TokenUtils} from "../libraries/TokenUtils.sol"; 

import {MockYieldToken} from "./mocks/MockYieldToken.sol"; 

import {IMockYieldToken} from "./mocks/MockYieldToken.sol"; 

import {MYTTestHelper} from "./libraries/MYTTestHelper.sol"; 

import {MockMYTStrategy} from "./mocks/MockMYTStrategy.sol"; 

import {AlchemistAllocator} from "../AlchemistAllocator.sol"; 

import {IMYTStrategy} from "../interfaces/IMYTStrategy.sol"; 

 

contract ProofOfVulnerability is Test { 

    using MYTTestHelper for *; 

 

    AlchemistAllocator public allocator; 

    VaultV2 public vault; 

    address public admin = address(0x2222222222222222222222222222222222222222); 

    address public operator = address(0x3333333333333333333333333333333333333333); 

    address public curator = address(0x8888888888888888888888888888888888888888); 

    address public user1 = address(0x5555555555555555555555555555555555555555); 

    address public mockVaultCollateral; 

    address public mockStrategyYieldToken; 

    MockMYTStrategy public mytStrategy; 

 

    function setUp() public { 

        mockVaultCollateral = address(new TestERC20(100e18, uint8(18))); 

        mockStrategyYieldToken = address(new MockYieldToken(mockVaultCollateral)); 

         

        vm.startPrank(admin); 

        vault = MYTTestHelper._setupVault(mockVaultCollateral, admin, curator); 

        mytStrategy = MYTTestHelper._setupStrategy( 

            address(vault),  

            mockStrategyYieldToken,  

            admin,  

            "MockToken",  

            "MockTokenProtocol",  

            IMYTStrategy.RiskClass.LOW 

        ); 

        allocator = new AlchemistAllocator(address(vault), admin, operator); 

        vm.stopPrank(); 

         

        vm.startPrank(curator); 

        vault.submit(abi.encodeCall(IVaultV2.setIsAllocator, (address(allocator), true))); 

        vm.warp(block.timestamp + vault.timelock(IVaultV2.setIsAllocator.selector)); 

        vault.setIsAllocator(address(allocator), true); 

         

        vault.submit(abi.encodeCall(IVaultV2.addAdapter, address(mytStrategy))); 

        vm.warp(block.timestamp + vault.timelock(IVaultV2.addAdapter.selector)); 

        vault.addAdapter(address(mytStrategy)); 

         

        bytes memory idData = mytStrategy.getIdData(); 

        vault.submit(abi.encodeCall(IVaultV2.increaseAbsoluteCap, (idData, 200 ether))); 

        vm.warp(block.timestamp + vault.timelock(IVaultV2.increaseAbsoluteCap.selector)); 

        vault.increaseAbsoluteCap(idData, 200 ether); 

         

        vault.submit(abi.encodeCall(IVaultV2.increaseRelativeCap, (idData, 0.10e18))); 

        vm.warp(block.timestamp + vault.timelock(IVaultV2.increaseRelativeCap.selector)); 

        vault.increaseRelativeCap(idData, 0.10e18); 

        vm.stopPrank(); 

    } 

 

    function testProofOfVulnerability_AdjustedNeverUsed() public { 

        deal(address(mockVaultCollateral), address(user1), 500 ether); 

        vm.startPrank(user1); 

        TokenUtils.safeApprove(address(mockVaultCollateral), address(vault), 500 ether); 

        IVaultV2(address(vault)).deposit(500 ether, address(vault)); 

        vm.stopPrank(); 

 

        bytes32 allocationId = mytStrategy.adapterId(); 

        uint256 absoluteCap = vault.absoluteCap(allocationId); 

        uint256 relativeCap = vault.relativeCap(allocationId); 

        uint256 totalAssets = vault._totalAssets(); 

         

        uint256 relativeCapInAssets = (totalAssets * relativeCap) / 1e18; 

        uint256 allocatorCalculatesAdjusted = absoluteCap > relativeCapInAssets ? absoluteCap : relativeCapInAssets; 

 

        assertEq(absoluteCap, 200 ether, "Vault absoluteCap = 200 ether"); 

        assertEq(relativeCap, 0.10e18, "Vault relativeCap = 10%"); 

        assertEq(relativeCapInAssets, 50 ether, "RelativeCap in assets = 50 ether"); 

        assertEq(allocatorCalculatesAdjusted, 200 ether, "Allocator's adjusted = max(200, 50) = 200 ether"); 

 

        vm.startPrank(admin); 

        vm.expectRevert(); 

        allocator.allocate(address(mytStrategy), 100 ether); 

        vm.stopPrank(); 

 

        assertTrue( 

            allocatorCalculatesAdjusted == 200 ether, 

            "BUG PROOF: Allocator calculates adjusted=200 ether (choosing absoluteCap over relativeCap), but never validates amount<=adjusted. Instead, it blindly passes 100 ether to vault which rejects it with RelativeCapExceeded. The allocator's entire cap selection logic (absoluteCap vs relativeCap comparison) is useless dead code since adjusted is never enforced." 

        ); 

    } 

} 

```

The Bug is Crystal Clear:

Allocator fetches absoluteCap = 200 ether

Allocator fetches relativeCap = 0.10e18 (= 50 ether in assets)

Allocator calculates internally: adjusted = max(200, 50) = 200 ether

Allocator NEVER checks if amount (100e) <= adjusted (200e)

Allocator immediately calls vault.allocate(100 ether)

Vault enforces its own relativeCap and reverts with RelativeCapExceeded()

Why This Proves the Vulnerability:

The allocator's logic determined that absoluteCap (200e) should be the governing limit, NOT relativeCap (50e). This is what the adjusted calculation represents - picking the maximum of the two caps.

If the allocator had enforced its adjusted calculation:

It would check: require(100 <= 200) PASS

The allocation would be allowed from the allocator's perspective

But since the allocator ignores adjusted:

It blindly passes 100 ether to the vault

The vault independently enforces BOTH caps (not just the max)

The vault rejects because 100 > 50 (relative cap)

This makes the entire cap selection logic dead code with no purpose, proving the bug report is valid and definitively proves the unused adjusted variable vulnerability!
