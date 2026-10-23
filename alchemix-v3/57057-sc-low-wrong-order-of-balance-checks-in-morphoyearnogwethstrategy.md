# 57057 sc low wrong order of balance checks in morphoyearnogwethstrategy

**Submitted on Oct 23rd 2025 at 05:55:54 UTC by @Pro\_King for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57057
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The MorphoYearnOGWETHStrategy contract contains a logic error in the `_deallocate` function where before and after balance checks are performed after withdrawal, causing every withdrawal to falsely report losses and incorrect financial reporting in production.

## Vulnerability Details

The vulnerability exists in the `_deallocate` function of the MorphoYearnOGWETHStrategy contract, where the balance checks are performed in the incorrect order:

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    vault.withdraw(amount, address(this), address(this));  // Withdrawal happens FIRST
--->    uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));  // "Before" balance recorded AFTER withdrawal
--->    uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));   // "After" balance recorded AFTER withdrawal
    uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;  // Always equals 0
    if (wethRedeemed < amount) {  // Always true (0 < amount)
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);  // Always emitted
    }
    // ... rest of function
}
```

The issue is that the "before" balance is recorded AFTER the withdrawal has already occurred, making both balance checks identical. This results in `wethRedeemed` always being 0, which is always less than the requested amount, causing the function to always emit a false loss event.

The correct implementation should record the "before" balance first, then perform the withdrawal, and finally record the "after" balance:

## Impact Details

This vulnerability has several impacts:

1. **Alert System Failure**: Every withdrawal triggers a false loss event, flooding monitoring systems with false positives. This renders loss detection systems useless, as real losses become indistinguishable from normal operations.
2. **Undetected Theft**: If a real loss occurs, it will be buried among numerous false alerts, allowing attackers to steal funds without triggering unique alerts.
3. **Risk Management Breakdown**: The strategy appears to be constantly losing money, making accurate risk assessment impossible. This could lead to incorrect allocation decisions, premature strategy replacement, or failure to replace a genuinely underperforming strategy.
4. **Financial Misreporting**: The false loss events could lead to incorrect financial reporting, potentially affecting user confidence and protocol valuation. The system would report losses of 100% on every withdrawal (showing 0 WETH received for X WETH requested).

## References

* MorphoYearnOGWETHStrategy.sol:[deallocate function](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/mainnet/MorphoYearnOGWETH.sol#L49)

## Proof of Concept

## Proof of Concept

Place this below test at v3-poc/src/test/strategies/MorphoYearnOGWETHStrategy.t.sol file and run this command `forge test --mt "test_bug_false_loss_event_always_emitted" -vv`

```solidity
function test_bug_false_loss_event_always_emitted() public {
        uint256 amountToAllocate = 10e18; 
        uint256 amountToDeallocate = 5e18; 
        
        vm.startPrank(vault);
        
        deal(WETH, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets > 0, "Initial real assets is 0");
        
        vm.expectEmit(true, true, true, true);
        emit MYTStrategy.StrategyDeallocationLoss("Strategy deallocation loss.", amountToDeallocate, 0);
        
        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));

        vm.stopPrank();
    }

```

**Output of test:**

```javascript
Ran 1 test for src/test/strategies/MorphoYearnOGWETHStrategy.t.sol:MorphoYearnOGWETHStrategyTest
[PASS] test_bug_false_loss_event_always_emitted() (gas: 1172420)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 39.07s (2.37s CPU time)
```
