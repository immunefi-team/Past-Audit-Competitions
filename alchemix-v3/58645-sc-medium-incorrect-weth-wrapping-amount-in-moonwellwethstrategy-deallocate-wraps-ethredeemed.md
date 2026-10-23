# 58645 sc medium incorrect weth wrapping amount in moonwellwethstrategy deallocate wraps ethredeemed instead of amount&#x20;

**Submitted on Nov 3rd 2025 at 19:18:10 UTC by @ZenHunter for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58645
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/MoonwellWETHStrategy.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The `_deallocate()` function in `MoonwellWETHStrategy.sol` contains incorrect logic for wrapping ETH to WETH. While the condition `if (ethRedeemed + ethBalanceBefore >= amount)` correctly checks if sufficient total ETH exists, the wrapping operation on line 65 incorrectly wraps `ethRedeemed` instead of `amount`. This creates a critical scenario: **when `ethRedeemed < amount` (losses occur) but sufficient total ETH exists** (`ethBalanceAfter >= amount`), the function wraps only `ethRedeemed` ETH, resulting in only `ethRedeemed` WETH. The subsequent require check `require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount)` then **fails and reverts the transaction**, even though sufficient ETH exists in the contract to wrap the required `amount`. This bug causes legitimate deallocations to fail unnecessarily when losses occur but pre-existing ETH compensates.

### Vulnerability Details

The `MoonwellWETHStrategy._deallocate()` function redeems WETH from the Moonwell protocol, which returns native ETH instead of WETH. The function must then wrap this ETH to WETH before approving it for withdrawal. However, the wrapping logic contains a critical flaw.

**Vulnerable Code (lines 54-69):**

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 ethBalanceBefore = address(this).balance;  // Track before redemption
    // Pull exact amount of underlying WETH out
    mWETH.redeemUnderlying(amount);  // Returns ETH, not WETH
    // wrap any ETH received (Moonwell redeems to ETH for WETH markets)
    uint256 ethBalanceAfter = address(this).balance;
    uint256 ethRedeemed = ethBalanceAfter - ethBalanceBefore;  // Calculate redeemed ETH
    
    if (ethRedeemed < amount) {
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, ethRedeemed);
    }
    
    if (ethRedeemed + ethBalanceBefore >= amount) {
        weth.deposit{value: ethRedeemed}();  // Wrap only if condition passes
    }
    
    require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
    TokenUtils.safeApprove(address(weth), msg.sender, amount);
    return amount;
}
```

**The Problem:**

The condition on line 64 `if (ethRedeemed + ethBalanceBefore >= amount)` is actually **correct** - it checks if the total ETH balance after redemption (`ethBalanceAfter = ethBalanceBefore + ethRedeemed`) is sufficient to meet the required `amount`. However, the wrapping operation on line 65 is **incorrect**:

```solidity
if (ethRedeemed + ethBalanceBefore >= amount) {
    weth.deposit{value: ethRedeemed}();  // WRONG: Should wrap `amount`, not `ethRedeemed`
}
```

**Why This Is Wrong:**

1. The goal of `_deallocate` is to return `amount` of WETH to the caller
2. The condition has already verified that `ethBalanceAfter >= amount`, meaning sufficient ETH exists
3. The function should wrap `amount` ETH to get `amount` WETH, not just `ethRedeemed`
4. **Critical Issue:** When `ethRedeemed < amount` (losses occur) but `ethBalanceAfter >= amount` (sufficient total ETH exists), wrapping only `ethRedeemed` ETH results in only `ethRedeemed` WETH, causing the require check to fail even though sufficient ETH exists to complete the deallocation

**Comparison with MoonwellUSDCStrategy:**

* `MoonwellUSDCStrategy._deallocate()` (lines 56-68) correctly handles deallocation for USDC, which does not require wrapping. It tracks balance before and after `mUSDC.redeemUnderlying(amount)`, emits loss events when `usdcRedeemed < amount`, and then checks if `TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount`. **In the case where `usdcRedeemed < amount` but `TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount`, the function correctly proceeds and returns `amount`**, as the require check passes. This demonstrates the correct pattern: when sufficient total balance exists (from redemption plus any pre-existing balance), the function should proceed with the full `amount`, not just the redeemed portion.
* In contrast, `MoonwellWETHStrategy._deallocate()` must wrap ETH to WETH after redemption, and incorrectly wraps `ethRedeemed` instead of `amount` when sufficient total ETH exists, even though it correctly checks `ethRedeemed + ethBalanceBefore >= amount`.

**Same Issue in Other Strategy:**

* `StargateEthPoolStrategy._deallocate()` (lines 55-82) has the same bug. On line 77, it wraps `ethRedeemed` instead of `amount` when `ethRedeemed + ethBalanceBefore >= amount` (line 76). The function should wrap `amount` ETH to WETH when sufficient total ETH exists, just like `MoonwellWETHStrategy` should.

**Example of the Bug:**

Scenario (Pre-existing ETH compensates, but function still reverts):

* `ethBalanceBefore = 50` (pre-existing ETH)
* Request: `amount = 500`
* `mWETH.redeemUnderlying(500)` returns 480 ETH (20 loss)
* `ethBalanceAfter = 530`, `ethRedeemed = 480`
* Loss detected: `480 < 500` - Correctly emits `StrategyDeallocationLoss`
* Condition check: `480 + 50 >= 500` = `530 >= 500` - **PASSES** (sufficient total ETH exists)
* Wrap: `weth.deposit{value: 480}()` - Only 480 WETH created - **SHOULD WRAP 500**
* Require check: `weth.balanceOf() >= 500` = 480 >= 500 - **REVERTS**
* **Result: Transaction reverts even though 530 ETH exists (enough to wrap 500 WETH)**

### Impact Details

When `ethRedeemed < amount` (losses occur) but `ethBalanceAfter >= amount` (sufficient total ETH exists), the function wraps only `ethRedeemed` ETH instead of `amount` ETH, causing the require check to fail and revert the transaction unnecessarily. Sufficient ETH exists to fulfill the deallocation, but the bug prevents it from being wrapped and returned. This bug can be triggered when protocol fees, slippage, or rounding errors cause redemption to return less ETH than requested, but the strategy contract has pre-existing ETH that compensates for the loss. The impact is limited to edge cases where losses occur during redemption but sufficient total ETH exists in the contract, making this scenario uncommon in practice.

### References

**Affected Code:**

* <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/MoonwellWETHStrategy.sol>
* <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/StargateEthPoolStrategy.sol>

### Recommendation

**Fix: Wrap `amount` ETH when condition passes:**

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 ethBalanceBefore = address(this).balance;
    mWETH.redeemUnderlying(amount);
    uint256 ethBalanceAfter = address(this).balance;
    uint256 ethRedeemed = ethBalanceAfter - ethBalanceBefore;
    
    if (ethRedeemed < amount) {
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, ethRedeemed);
    }
    
    // Wrap `amount` ETH to WETH when sufficient total ETH exists
    if (ethRedeemed + ethBalanceBefore >= amount) {
        weth.deposit{value: amount}();  // Wrap `amount`, not `ethRedeemed`
    }
    
    require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
    TokenUtils.safeApprove(address(weth), msg.sender, amount);
    return amount;
}
```

## Proof of Concept

### Proof of Concept

**Test Implementation:**

The bug is demonstrated with a Foundry test that creates a mock Moonwell contract to simulate the loss scenario. The test is located at `src/test/strategies/MoonwellWETHStrategy.t.sol`.

**MockMoonwell Contract:**

```solidity
// Mock Moonwell contract that implements IMToken interface
contract MockMoonwell {
    uint256 public redeemAmount;
    mapping(address => uint256) public balances;
    address public immutable weth;
    
    constructor(address _weth) {
        weth = _weth;
    }
    
    function setRedeemAmount(uint256 _amount) external {
        redeemAmount = _amount;
    }
    
    function mint(uint256 amount) external returns (uint256) {
        require(IERC20(weth).transferFrom(msg.sender, address(this), amount), "Transfer failed");
        balances[msg.sender] += amount;
        return 0;
    }
    
    function redeemUnderlying(uint256) external returns (uint256) {
        payable(msg.sender).transfer(redeemAmount);
        if (balances[msg.sender] >= redeemAmount) {
            balances[msg.sender] -= redeemAmount;
        } else {
            balances[msg.sender] = 0;
        }
        return 0;
    }
    
    function balanceOfUnderlying(address owner) external view returns (uint256) {
        return balances[owner];
    }
    
    function balanceOf(address owner) external view returns (uint256) {
        return balances[owner];
    }
    
    function exchangeRateStored() external pure returns (uint256) {
        return 1e18;
    }
    
    function exchangeRateCurrent() external returns (uint256) {
        return 1e18;
    }
    
    receive() external payable {}
}
```

**Test Code:**

```solidity
function test_PoC_BUG_incorrect_weth_wrapping_amount() public {
    uint256 amountToAllocate = 500e18;
    uint256 amountToDeallocate = 500e18;
    uint256 preExistingETH = 50e18;
    uint256 ethRedeemed = 480e18;
    
    vm.startPrank(vault);
    
    MockMoonwell mockMoonwell = new MockMoonwell(WETH);
    vm.deal(address(mockMoonwell), 1000e18);
    
    MockMoonwellWETHStrategy testableStrategy = new MockMoonwellWETHStrategy(
        vault,
        getStrategyConfig(),
        address(mockMoonwell),
        WETH,
        OPTIMISM_PERMIT2
    );
    
    deal(testConfig.vaultAsset, address(testableStrategy), amountToAllocate);
    bytes memory prevAllocationAmount = abi.encode(0);
    IMYTStrategy(address(testableStrategy)).allocate(prevAllocationAmount, amountToAllocate, "", vault);
    
    vm.deal(address(testableStrategy), preExistingETH);
    mockMoonwell.setRedeemAmount(ethRedeemed);
    
    bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
    vm.expectRevert("Strategy balance is less than the amount needed");
    IMYTStrategy(address(testableStrategy)).deallocate(prevAllocationAmount2, amountToDeallocate, "", vault);
    
    vm.stopPrank();
}
```

**Test Execution Results:**

```
Ran 1 test for src/test/strategies/MoonwellWETHStrategy.t.sol:MoonwellWETHStrategyTest
[PASS] test_PoC_BUG_incorrect_weth_wrapping_amount() (gas: 2878353)
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

**Test Scenario:**

1. **Setup:**
   * Allocate 500 WETH to the strategy
   * Add 50 ETH as pre-existing ETH to the strategy contract
   * Configure mock Moonwell to return only 480 ETH when `redeemUnderlying(500)` is called (simulating a 20 ETH loss)
2. **Execution:**
   * Call `deallocate(500)` to deallocate 500 WETH
   * `_deallocate()` records `ethBalanceBefore = 50 ETH`
   * `mockMoonwell.redeemUnderlying(500)` sends 480 ETH to strategy
   * `ethBalanceAfter = 50 + 480 = 530 ETH`
   * `ethRedeemed = 530 - 50 = 480 ETH`
   * Loss detected: `480 < 500` - emits `StrategyDeallocationLoss`
   * Condition check: `480 + 50 >= 500` = `530 >= 500` - **PASSES** (sufficient total ETH exists)
   * Wrap: `weth.deposit{value: 480}()` - Only 480 WETH created (BUG: should wrap 500)
   * Require check: `weth.balanceOf() >= 500` = `480 >= 500` - **REVERTS** with "Strategy balance is less than the amount needed"
3. **Result:**
   * **Test passes with `vm.expectRevert()`** - confirming the transaction reverts even though 530 ETH exists (enough to wrap 500 WETH)
   * The bug prevents successful deallocation when losses occur but sufficient total ETH exists
