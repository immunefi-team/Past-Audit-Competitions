# 57079 sc low h 1 morphoyearnogweth strategy incorrect balance measurement order in deallocate causes dos on withdrawals with any loss

**Submitted on Oct 23rd 2025 at 09:40:00 UTC by @Aizen09 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57079
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 1 hour

## Description

## Summary

The `MorphoYearnOGWETHStrategy._deallocate()` function measures balance AFTER withdrawal instead of BEFORE, causing `wethRedeemed` to always be 0 and making the function revert on any withdrawal loss.

## Vulnerability Details

**Location:** `src/strategies/mainnet/MorphoYearnOGWETH.sol` lines 49-56

**Buggy Code:**

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    vault.withdraw(amount, address(this), address(this));  // Line 49: Withdrawal happens HERE
    uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));  // Line 50: Measured AFTER withdrawal
    uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));   // Line 51: Same value!
    uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;  // Line 52: Always 0
    if (wethRedeemed < amount) {
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
    }
    require(wethRedeemed + wethBalanceBefore >= amount, "Strategy balance is less than the amount needed");  // Line 56: Will fail
    ...
}
```

## Root Cause

Lines 50-51 both measure balance AFTER the withdrawal (line 49), making them equal. Therefore `wethRedeemed = 0` always.

## Impact

**Severity: HIGH** - Denial of Service

1. Any deallocate with slippage/loss will revert
2. Cannot withdraw from strategy when market conditions cause losses
3. Funds stuck in strategy
4. Protocol cannot rebalance allocations

## Proof of Concept

**How to run the POC:**

```bash
forge test --match-contract MorphoYearnBugTest -vvv
```

**Scenario:** Deallocate 100 WETH with 2 WETH loss

**Execution Flow:**

```
Initial: Strategy has 0 WETH
Line 49: vault.withdraw(100, this, this)
  → Vault sends 98 WETH (2 WETH loss)
  → Strategy balance = 98 WETH

Line 50: wethBalanceBefore = safeBalanceOf() = 98 WETH
Line 51: wethBalanceAfter = safeBalanceOf() = 98 WETH  ← Same!
Line 52: wethRedeemed = 98 - 98 = 0  ← Wrong!
Line 56: require(0 + 98 >= 100)  ← Fails! (98 < 100)

Result: Transaction REVERTS
```

**Expected:** `wethRedeemed` should be 98 (actual received amount) **Actual:** `wethRedeemed` is 0, causing revert

## Code Comparison

**Correct Implementation** (TokeAutoEth):

```solidity
uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));  // BEFORE
autoEth.redeem(sharesNeeded, address(this), address(this));  // Withdrawal
uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));   // AFTER
uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;  // Correct: 98 - 0 = 98
```

**Buggy Implementation** (MorphoYearn):

```solidity
vault.withdraw(amount, address(this), address(this));  // Withdrawal FIRST
uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));  // AFTER
uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));   // AFTER
uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;  // Wrong: 98 - 98 = 0
```

## Recommendation

Move the `wethBalanceBefore` measurement BEFORE the withdrawal:

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));  // BEFORE
    vault.withdraw(amount, address(this), address(this));
    uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));   // AFTER
    uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
    if (wethRedeemed < amount) {
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
    }
    require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
    TokenUtils.safeApprove(address(weth), msg.sender, amount);
    return amount;
}
```

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "forge-std/Test.sol";

// Mock contracts
contract MockWETH {
    mapping(address => uint256) public balanceOf;
    
    function deposit() external payable {
        balanceOf[msg.sender] += msg.value;
    }
    
    function withdraw(uint256 amount) external {
        balanceOf[msg.sender] -= amount;
        payable(msg.sender).transfer(amount);
    }
    
    function transfer(address to, uint256 amount) external returns (bool) {
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        return true;
    }
    
    function approve(address, uint256) external returns (bool) {
        return true;
    }
}

contract MockVault {
    MockWETH public weth;
    uint256 public lossAmount;
    
    constructor(address _weth) {
        weth = MockWETH(_weth);
    }
    
    function setLoss(uint256 _loss) external {
        lossAmount = _loss;
    }
    
    function withdraw(uint256 assets, address receiver, address) external returns (uint256) {
        uint256 actualAmount = assets - lossAmount;
        weth.transfer(receiver, actualAmount);
        return assets;
    }
}

contract BuggyMorphoStrategy {
    MockWETH public weth;
    MockVault public vault;
    
    event StrategyDeallocationLoss(string message, uint256 requested, uint256 received);
    
    constructor(address _weth, address _vault) {
        weth = MockWETH(_weth);
        vault = MockVault(_vault);
    }
    
    function deallocate(uint256 amount) external returns (uint256) {
        vault.withdraw(amount, address(this), address(this));
        uint256 wethBalanceBefore = weth.balanceOf(address(this));
        uint256 wethBalanceAfter = weth.balanceOf(address(this));
        uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
        
        if (wethRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
        }
        
        require(wethRedeemed + wethBalanceBefore >= amount, "Strategy balance is less than the amount needed");
        return amount;
    }
}

contract MorphoYearnBugTest is Test {
    MockWETH public weth;
    MockVault public vault;
    BuggyMorphoStrategy public buggyStrategy;
    
    event StrategyDeallocationLoss(string message, uint256 requested, uint256 received);
    
    function setUp() public {
        weth = new MockWETH();
        vault = new MockVault(address(weth));
        buggyStrategy = new BuggyMorphoStrategy(address(weth), address(vault));
        
        vm.deal(address(this), 1000 ether);
        weth.deposit{value: 1000 ether}();
        weth.transfer(address(vault), 1000 ether);
    }
    
    function testBuggyStrategyRevertsOnLoss() public {
        vault.setLoss(2 ether);
        
        console.log("=== Testing BUGGY Strategy ===");
        console.log("Requesting: 100 WETH");
        console.log("Vault will return: 98 WETH (2 WETH loss)");
        
        vm.expectRevert("Strategy balance is less than the amount needed");
        buggyStrategy.deallocate(100 ether);
        
        console.log("Result: REVERTED due to bug");
    }
    
    function testBugProofWethRedeemedIsZero() public {
        vault.setLoss(2 ether);
        
        vm.expectEmit(true, true, true, true);
        emit StrategyDeallocationLoss("Strategy deallocation loss.", 100 ether, 0);
        
        vm.expectRevert();
        buggyStrategy.deallocate(100 ether);
    }
}
```
