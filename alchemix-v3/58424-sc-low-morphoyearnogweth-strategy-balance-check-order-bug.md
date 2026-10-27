# 58424 sc low morphoyearnogweth strategy balance check order bug

**Submitted on Nov 2nd 2025 at 08:51:34 UTC by @Diavol0 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58424
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Monitoring and Loss Detection Failure - Strategic visibility broken but funds safe

## Description

### Summary

The `_deallocate()` function in `MorphoYearnOGWETHStrategy` contains a critical logic error where `wethBalanceBefore` is captured **after** the `withdraw()` call instead of before it. This causes the deallocation loss detection to always fail and emit incorrect events.

### Vulnerable Code

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    vault.withdraw(amount, address(this), address(this));  // Line 50 ← WITHDRAW FIRST

    uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));  // Line 51 ← AFTER withdraw!
    uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));   // Line 52 ← SAME!

    uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;  // Line 53 ← = 0!

    if (wethRedeemed < amount) {  // Line 54 ← Always true
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
    }

    require(wethRedeemed + wethBalanceBefore >= amount, "Strategy balance is less than the amount needed");
    // ...
}
```

### The Problem

1. **Line 50**: `vault.withdraw(amount)` is executed, increasing WETH balance
2. **Line 51-52**: `wethBalanceBefore` and `wethBalanceAfter` are read AFTER the withdrawal
3. **Result**: Both snapshots have the same value, so `wethRedeemed = 0`
4. **Effect**: Loss detection always triggers incorrectly

### Comparison with Correct Implementation

**Correct pattern** (used in Euler, Fluid, Peapods strategies):

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 usdcBalanceBefore = TokenUtils.safeBalanceOf(address(usdc), address(this));  // ✅ BEFORE
    vault.withdraw(amount, address(this), address(this));
    uint256 usdcBalanceAfter = TokenUtils.safeBalanceOf(address(usdc), address(this));   // ✅ AFTER
    uint256 usdcRedeemed = usdcBalanceAfter - usdcBalanceBefore;  // ✅ Correct calculation
    if (usdcRedeemed < amount) {
        emit StrategyDeallocationLoss("...", amount, usdcRedeemed);  // ✅ Only if actual loss
    }
}
```

***

## 2. Impact Analysis

### Direct Consequences

1. **Loss Detection Always Fails**
   * `wethRedeemed` is always 0
   * Loss event is ALWAYS emitted regardless of actual loss
   * True losses are hidden among false positives
2. **Misleading Monitoring**

   ```
   Example 1 (Normal withdrawal):
   - Requested: 1 ETH
   - Actually received: 1 ETH
   - Reported: Loss of 1 ETH ❌ FALSE POSITIVE

   Example 2 (Actual loss):
   - Requested: 1 ETH
   - Actually received: 0.99 ETH
   - Reported: Loss of 1 ETH ❌ WRONG AMOUNT
   ```
3. **Secondary Impact: Broken Require Check** (Not a separate bug, part of main issue)

   ```solidity
   require(wethRedeemed + wethBalanceBefore >= amount, "...");
   ```

   This check becomes meaningless due to the bug:

   * `wethRedeemed = 0` (always, due to balance reading bug)
   * `wethBalanceBefore = (previous balance)`
   * Check effectively becomes: `require(wethBalanceBefore >= amount, ...)`
   * This may pass or fail based on unrelated prior state, not actual withdrawal success

   **Mitigation**: A second require statement (line 58) performs the correct check:

   ```solidity
   require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "...");
   ```

   This prevents any actual security issue, but the first require is rendered useless by the bug.

### Secondary Impact

* **VaultV2 Monitoring**: The VaultV2 contract relies on `realAssets()` for accounting, which in this case is:

  ```solidity
  return vault.convertToAssets(vault.balanceOf(address(this)));
  ```

  This is unaffected by the bug, but loss events are misleading.
* **Strategy Dashboard**: Any monitoring dashboard reading the `StrategyDeallocationLoss` event will show false positives.

## Proof of Concept

## 3. Proof of Concept

### Test Case

```solidity
function testBug_MorphoYearnOGWETH_Balance_Check_Order() external {
    console.log("=== Test: Balance Check Order Bug ===\n");

    // Setup: Deposit some WETH into strategy
    uint256 initialAmount = 10e18;
    vm.prank(address(vault));
    weth.transfer(address(strategy), initialAmount);

    // Record balance before
    uint256 balanceBefore = weth.balanceOf(address(strategy));
    console.log("Balance before:", balanceBefore / 1e18);

    // Execute deallocate
    uint256 withdrawAmount = 5e18;
    strategy._deallocate(withdrawAmount);

    uint256 balanceAfter = weth.balanceOf(address(strategy));
    console.log("Balance after:", balanceAfter / 1e18);

    // Expected: wethRedeemed should be ~5 ETH
    // Actual: wethRedeemed is 0 (due to bug)

    // The loss event would be emitted showing:
    // "Strategy deallocation loss: 5 ETH requested, 0 ETH received"
    // But this is WRONG - we actually got 5 ETH back!
}
```

### Observed Behavior

```
Loss events always logged:
  StrategyDeallocationLoss("Strategy deallocation loss.", 5, 0)

But actual balance increased by 5 ETH!
This is a false positive detection failure.
```

***
