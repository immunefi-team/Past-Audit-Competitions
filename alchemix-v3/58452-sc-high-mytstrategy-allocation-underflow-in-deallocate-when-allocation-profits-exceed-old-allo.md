# 58452 sc high mytstrategy allocation underflow in deallocate when allocation profits exceed old allocation

**Submitted on Nov 2nd 2025 at 13:31:20 UTC by @Brainiac5 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58452
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* **Impacts:**
  * Permanent freezing of unclaimed royalties
  * Permanent freezing of unclaimed yield

## Description

## Summary

The `MYTStrategy` contract reads stale allocation accounting and thus cannot handle strategies that return profits. When a strategy has earned yield and returns more assets than initially allocated, the subtraction `newAllocation = oldAllocation - amountDeallocated` underflows, causing transaction reverts and locking funds.

## Vulnerable Code

### AlchemistAllocator Reads Allocation from Vault

**File**: `src/AlchemistAllocator.sol`\
**Lines**: 55-67

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
    // Reads allocation from vault and passes to strategy
    bytes memory oldAllocation = abi.encode(vault.allocation(id));
    vault.deallocate(adapter, oldAllocation, amount);
}
```

### MYTStrategy Receives Allocation and Underflows

**File**: `src/MYTStrategy.sol`\
**Lines**: 119-133

```solidity
function deallocate(bytes memory data, uint256 assets, bytes4 selector, address sender)
    external
    onlyVault
    returns (bytes32[] memory strategyIds, int256 change)
{
    if (killSwitch) {
        return (ids(), int256(0));
    }
    require(assets > 0, "Zero amount");
    
    // Receives oldAllocation from AlchemistAllocator
    uint256 oldAllocation = abi.decode(data, (uint256));
    uint256 amountDeallocated = _deallocate(assets);
    
    // @audit BUG: When amountDeallocated > oldAllocation (due to profits)
    // this subtraction underflows!
    uint256 newAllocation = oldAllocation - amountDeallocated;
    
    emit Deallocate(amountDeallocated, address(this));
    return (ids(), int256(newAllocation) - int256(oldAllocation));
}
```

## Vulnerability Details

### The Profit Scenario

1. **Initial State**: Strategy allocated `100e6` USDC
2. **Time Passes**: Strategy earns `10e6` USDC in yield (10% profit)
3. **Total Assets**: Strategy now holds `110e6` USDC worth of shares
4. **Deallocate Call**: Vault requests withdrawal of all funds (`100e6`)
5. **Strategy Returns**: `_deallocate()` withdraws everything and returns `110e6`
6. **Calculation**: `newAllocation = 100e6 - 110e6` = **UNDERFLOW**

### Root Cause Analysis

The allocation tracking assumes:

```solidity
newAllocation = oldAllocation - amountDeallocated
```

This works when `amountDeallocated <= oldAllocation`, but **breaks with profits**:

* Old allocation: `100e6`
* Amount deallocated (with profit): `110e6`
* Subtraction: `100e6 - 110e6` attempts to store `-10e6` in `uint256`
* Result: **Arithmetic underflow** (Solidity 0.8.28 reverts on underflow)

### Why Strategies Return More Than Allocated

Yield-generating strategies naturally accumulate value:

```solidity
// Example: Euler USDC Strategy
function _deallocate(uint256 amount) internal override returns (uint256) {
    // Withdraw from yield vault
    vault.withdraw(amount, address(this), address(this));
    
    // May receive more than 'amount' if the vault has appreciated
    uint256 actualReceived = IERC20(usdc).balanceOf(address(this));
    
    // Returns actual amount, which can exceed requested amount
    return actualReceived;  // Can be > amount!
}
```

## Impact

* **Funds Locked**: Cannot deallocate from profitable strategies
* **Protocol DOS**: Deallocations revert, preventing vault rebalancing
* **Yield Trap**: Profits make positions un-withdrawable
* **Liquidity Crisis**: Users cannot exit positions that have earned yield
* **Strategy Stuck**: Once profitable, strategy becomes permanently locked

## Proof of Concept

## Proof of Concept

**Create Test File**: `test/VaultAllocationUnderflow.t.sol` **Add below code to the test file**

**Run with**:

```bash
forge test --match-path test/VaultAllocationUnderflow.t.sol -vv
```

## Code snippet

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "forge-std/Test.sol";
import "forge-std/console.sol";
import {VaultV2} from "../lib/vault-v2/src/VaultV2.sol";
import {AlchemistAllocator} from "../src/AlchemistAllocator.sol";
import {IMYTStrategy} from "../src/interfaces/IMYTStrategy.sol";
import {MYTStrategy} from "../src/MYTStrategy.sol";

contract MockERC20 {
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    uint8 public constant decimals = 6;
    
    function mint(address to, uint256 amount) external {
        balanceOf[to] += amount;
    }
    
    function transfer(address to, uint256 amount) external returns (bool) {
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        return true;
    }
    
    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        allowance[from][msg.sender] -= amount;
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        return true;
    }
    
    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        return true;
    }
}

/**
 * @notice Mock strategy that simulates profit accrual
 * @dev When deallocating 100e6 from a strategy with 10% profit:
 *      - oldAllocation = 100e6
 *      - amountDeallocated = 110e6 (includes 10e6 profit)
 *      - newAllocation = oldAllocation - amountDeallocated = 100e6 - 110e6 = UNDERFLOW
 */
/// @notice Mock strategy that inherits MYTStrategy and only overrides _allocate/_deallocate
contract MytProfitStrategyMock is MYTStrategy {
    uint256 public profitBps = 1000; // 10% profit

    constructor(
        address myt,
        IMYTStrategy.StrategyParams memory sp,
        address permit2,
        address receipt
    ) MYTStrategy(myt, sp, permit2, receipt) {}

    // Return exactly the requested amount (no slippage on allocate)
    function _allocate(uint256 amount) internal override returns (uint256) {
        console.log("=== ALLOCATE ===");
        console.log("  Assets allocated:", amount);
        return amount;
    }

    // Return more than requested (simulate realized profit on exit)
    // This will make MYTStrategy.deallocate() compute
    // newAllocation = oldAllocation - amountDeallocated
    // which underflows when amountDeallocated > oldAllocation
    function _deallocate(uint256 amount) internal override returns (uint256) {
        uint256 withProfit = amount + (amount * profitBps / 10_000);
        console.log("\n=== DEALLOCATE (WITH PROFIT) ===");
        console.log("  Requested assets:", amount);
        console.log("  Assets with profit:", withProfit);
        return withProfit;
    }
}

/**
 * @title VaultAllocationUnderflowTest
 * @notice Demonstrates that VaultV2 allocation tracking underflows when strategies return profits
 */
contract VaultAllocationUnderflowTest is Test {
    MockERC20 public asset;
    VaultV2 public vault;
    AlchemistAllocator public allocator;
    MytProfitStrategyMock public strategy;
    MockERC20 public receiptToken;
    
    address public owner = address(0x1);
    address public curator = address(0x2);
    address public admin = address(0x3);
    
    function setUp() public {
        asset = new MockERC20();
        
        // Deploy VaultV2
        vault = new VaultV2(owner, address(asset));
        
        // Deploy AlchemistAllocator
        vm.prank(admin);
        allocator = new AlchemistAllocator(address(vault), admin, admin);
        
        // Deploy a mock receipt token for MYTStrategy's constructor approval
        receiptToken = new MockERC20();

        // Prepare MYTStrategy params
        IMYTStrategy.StrategyParams memory sp = IMYTStrategy.StrategyParams({
            owner: admin,
            name: "Mock Profit Strategy",
            protocol: "MOCK_PROFIT_STRATEGY",
            riskClass: IMYTStrategy.RiskClass.MEDIUM,
            cap: type(uint256).max,
            globalCap: type(uint256).max,
            estimatedYield: 0,
            additionalIncentives: false,
            slippageBPS: 0
        });

        // Deploy strategy inheriting MYTStrategy. `myt` is the vault address.
        strategy = new MytProfitStrategyMock(address(vault), sp, address(0xBEEF), address(receiptToken));
        
        // Setup vault
        vm.startPrank(owner);
        vault.setCurator(curator);
        vm.stopPrank();
        
        // Configure vault with curator
        vm.startPrank(curator);
        
        // Add adapter
        bytes memory addAdapterData = abi.encodeWithSelector(VaultV2.addAdapter.selector, address(strategy));
        vault.submit(addAdapterData);
        vm.warp(block.timestamp + 1);
        vault.addAdapter(address(strategy));
        
        // Set caps - use the raw protocol string that generates the strategyId
    bytes memory strategyIdData = abi.encode("MOCK_PROFIT_STRATEGY");
        
        bytes memory increaseAbsoluteCapData = abi.encodeWithSelector(
            VaultV2.increaseAbsoluteCap.selector, 
            strategyIdData, 
            1000e6
        );
        vault.submit(increaseAbsoluteCapData);
        vm.warp(block.timestamp + 1);
        vault.increaseAbsoluteCap(strategyIdData, 1000e6);
        
        bytes memory increaseRelativeCapData = abi.encodeWithSelector(
            VaultV2.increaseRelativeCap.selector,
            strategyIdData,
            1e18
        );
        vault.submit(increaseRelativeCapData);
        vm.warp(block.timestamp + 1);
        vault.increaseRelativeCap(strategyIdData, 1e18);
        
        // Set allocator
        bytes memory setAllocatorData = abi.encodeWithSelector(VaultV2.setIsAllocator.selector, address(allocator), true);
        vault.submit(setAllocatorData);
        vm.warp(block.timestamp + 1);
        vault.setIsAllocator(address(allocator), true);
        
        vm.stopPrank();
        
        // Give vault assets
        asset.mint(address(vault), 1000e6);
    }
    
    /**
     * @notice THE BUG: Allocation underflows when deallocating with profit
     * @dev This test will revert with arithmetic underflow
     */
    function test_AllocationUnderflowsWithProfit() public {
        uint256 allocateAmount = 100e6;
        
        console.log("\n================================================================");
        console.log("  BUG: VaultV2 Allocation Tracking Underflows With Profits");
        console.log("================================================================\n");
        
        // Step 1: Allocate 100e6 to strategy
        vm.prank(admin);
        allocator.allocate(address(strategy), allocateAmount);
        
        uint256 allocation = vault.allocation(strategy.adapterId());
        console.log("\nAllocation successful. Current allocation:", allocation);
        
        // Step 2: Simulate 10% profit in strategy
        console.log("\nTime passes, strategy earns 10% profit...");
        
        // Step 3: Try to deallocate - THIS WILL UNDERFLOW
        console.log("\nAttempting to deallocate all funds (including profit)...");
        
        vm.prank(admin);
        vm.expectRevert(stdError.arithmeticError); // Expect arithmetic underflow (Panic(0x11))
        // strategy will deallocate allocateAmount + profits
        // This is to simplify the whole process and avoid having to deallocate multiple times to show the vulnerability
        allocator.deallocate(address(strategy), allocateAmount);
        
        console.log("\nREVERTED: Arithmetic underflow in MYTStrategy.sol line 129");
        console.log("  newAllocation = oldAllocation - amountDeallocated");
        console.log("  newAllocation = 100e6 - 110e6 = UNDERFLOW");
        console.log("\n================================================================\n");
    }
}

```
