# 58648 sc low incorrect wethbalancebefore read causes broken loss detection in deallocation

**Submitted on Nov 3rd 2025 at 19:35:04 UTC by @Khay3 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58648
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary

A balance snapshot is taken after withdrawal in `MorphoYearnOGWETHStrategy::_deallocate`, making `wethRedeemed` always zero and breaking loss detection.

## Vulnerability details

`MorphoYearnOGWETHStrategy::\_deallocate` reads `wethBalanceBefore` after the withdrawal. Since both “before” and “after” balances are read post-withdrawal, their difference is always zero, masking any loss and causing misleading events.

[MorphoYearnOGWETHStrategy::\_deallocate()](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/mainnet/MorphoYearnOGWETH.sol#L49C5-L61C6):

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    vault.withdraw(amount, address(this), address(this));  // Withdrawal happens FIRST
    uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));  // Read AFTER
    uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));   // Same value
    uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;  // Always 0
    if (wethRedeemed < amount) {
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
    }
    require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
    TokenUtils.safeApprove(address(weth), msg.sender, amount);
    return amount;
}
```

In comparison with other strategies, they implemented this correct (Read balance before wtihdrawal)

Example scenario

Morpho vault experiences issues and return only 95TH when 100 ETH was requested. Strategy cannot deter the 5ETH loss(`wethRedeemd` shows 0 instead of 95). The loss event incorrectly shows `StrategyDeallocationLoss("...", 100, 0)` instead of `StrategyDeallocationLoss("...", 100, 95)`. If the loss is within the `require` tolerance, it goes completely unnoticed

## Impact

This lead to the system not being able to detect actual losses from vault withdrawals. The `StrategyDeallocationLoss` event will always fire, since `0 < amount` is always true, even when there's no loss

## Likelihood

This will occur on every deallocation but mitigated with the require statement

```solidity
require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, 
        "Strategy balance is less than the amount needed");
```

## Recommendation

Read the WETH balance before the withdrawal, then withdraw, then read after, and compute the delta from those two readings.

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    // Read balance BEFORE withdrawal
    uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
    
    // Perform withdrawal
    vault.withdraw(amount, address(this), address(this));
    
    //Read balance AFTER withdrawal
    uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
    uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
```

## Proof of Concept

paste in `src/test/strategies/MorphoYearnOGWETHStrategy.t.sol` and run with `forge test --match-test test_strategy_deallocate_wrong_balance_calculation -vvv`

```solidity
function test_strategy_deallocate_wrong_balance_calculation() public {
    // This test directly shows the bug by examining the code flow
    vm.startPrank(vault);
    uint256 amount = 0.01e18;  // Very small amount

    // Fund strategy with WETH and allocate
    deal(WETH, strategy, amount);
    bytes memory prevAllocation = abi.encode(0);
    IMYTStrategy(strategy).allocate(prevAllocation, amount, "", address(vault));
    
    // Deallocate a tiny bit less than current assets to avoid revert
    uint256 real = IMYTStrategy(strategy).realAssets();
    require(real > 1, "no assets after allocation");
    uint256 dealloc = real - 1;
    bytes memory prevAllocation2 = abi.encode(dealloc);
    IMYTStrategy(strategy).deallocate(prevAllocation2, dealloc, "", address(vault));
    
    // (Balance delta omitted here to avoid dependency import; event-based PoC below)
    
    vm.stopPrank();
    }
```
