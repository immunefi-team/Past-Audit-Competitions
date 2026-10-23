# 57665 sc low incorrect balance measurement in deallocate function of morphoyearnogwethstrategy&#x20;

**Submitted on Oct 28th 2025 at 00:16:43 UTC by @Razkky for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57665
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary

A critical bug exists in the `_deallocate` function of the `MorphoYearnOGWETHStrategy` contract. The function incorrectly measures the WETH before and after `balance` after making withdrawal, resulting in `wethRedeemed` always being zero since the `wethBalanceBefore` and `wethBalanceAfter` are thesame. This causes the `StrategyDeallocationLoss` event to be emitted every time, regardless of actual loss, and breaks the intended event-based monitoring for deallocation shortfall.

### Relevant code

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    vault.withdraw(amount, address(this), address(this));
    uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
    uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
    uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
    if (wethRedeemed < amount) {
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
    }
    require(wethRedeemed + wethBalanceBefore >= amount, "Strategy balance is less than the amount needed");
    require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
    TokenUtils.safeApprove(address(weth), msg.sender, amount);
    return amount;
}
```

## Issue Explanation

* The `MorphoYearnOGWETHStrategy::_deallocate` function calls `vault.withdraw` and then reads both `wethBalanceBefore` and `wethBalanceAfter`, so both reflect the post-withdrawal balance.
* As a result, `wethRedeemed` is always zero (`wethBalanceAfter` - `wethBalanceBefore`).
* The event `StrategyDeallocationLoss` is emitted every time, with `wethRedeemed` as zero, even if the withdrawal was successful and no loss occurred.
* The require statement is logically correct, but because `wethBalanceBefore` is measured after withdrawal, it does not serve its intended purpose in this context
* If the vault returns less than the requested amount, but the post-withdrawal balance is high enough (e.g., due to a prior deposit or unrelated transfer), the function will not revert, and the strategy will silently deallocate less than intended.

## Impact

* False Event Emission: The `StrategyDeallocationLoss` event is always emitted, making it impossible to distinguish real losses from normal operations.
* Silent Under-Deallocation: The function may not revert even if less than the requested amount is actually withdrawn, as long as the post-withdrawal balance is sufficient. This can lead to accounting errors and potential loss of funds for users relying on accurate deallocation.
* Monitoring and Alerting Failure: Automated systems or users monitoring for real deallocation losses will receive false positives, reducing trust in the event system.

## Likelihood

**High.**

This bug will occur every time `_deallocate` on `MorphoYearnOGWETHStrategy` is called, regardless of the actual outcome of the withdrawal. The event will always be emitted, and the function may silently under-deallocate if the post-withdrawal balance is sufficient. This makes the issue highly likely to impact all users and monitoring systems relying on the event or on correct deallocation behavior.

## Proof of Concept

## Proof of Concept

A test that allocates and then deallocates the same amount will always emit the `StrategyDeallocationLoss` event with the wethRedeemed as `0`, even when no loss occurs. This demonstrates the bug:

The below test case should be aded to `MorphoYearnOGWETHStrategyTest` Run the command to execute the test after setting up your env variable `forge test --mt test_buggy_deallocate_always_emits_loss_event`

```solidity
    function test_buggy_deallocate_always_emits_loss_event(uint256 amountToAllocate) public {
        amountToAllocate = bound(amountToAllocate, 1e18, testConfig.vaultInitialDeposit);
        uint256 amountToDeallocate = amountToAllocate - 0.5e18; // Deallocate slightly less to avoid revert
        vm.startPrank(vault);
        deal(WETH, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        // Allocate full amount
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets > 0, "Initial real assets is 0");
        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
        // The bug: event is always emitted with wethRedeemed as 0 , despite wethRedeemed should equal actual redeemed amount
        vm.expectEmit(); // Expect event emission (should not happen in correct logic)
        emit MYTStrategy.StrategyDeallocationLoss("Strategy deallocation loss.", amountToDeallocate, 0); // Always emitted due to bug
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault)); // Triggers buggy event
        vm.stopPrank();
    }
```
