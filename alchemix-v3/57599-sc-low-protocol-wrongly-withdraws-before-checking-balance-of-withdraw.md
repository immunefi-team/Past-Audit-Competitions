# 57599 sc low protocol wrongly withdraws before checking balance of withdraw

**Submitted on Oct 27th 2025 at 12:17:17 UTC by @securehash1 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57599
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The \_deallocate function in the MorphoYearnOGWETHStrategy contract contains a logical flaw that results in incorrect accounting during the deallocation process. It wrongly `makes withdrawal before checking balanceBefore`.

## Vulnerability Details

When deallocating funds, the function attempts to measure `the amount of WETH received from the Yearn vault by comparing the WETH balance` before and after the withdrawal. However, due to a logical error, both balance accounting (wethBalanceBefore and wethBalanceAfter) are taken after the vault.withdraw() call has completed.

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    vault.withdraw(amount, address(this), address(this));
    // BUG: Both balance checks happen AFTER the withdrawal.
    uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
    uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
    // `wethRedeemed` will always be 0 as `wethBalanceAfter` equals `wethBalanceBefore`.
    uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
    if (wethRedeemed < amount) {
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
    }
    // This check becomes `require(wethBalanceAfterWithdrawal >= amount)`
    require(wethRedeemed + wethBalanceBefore >= amount, "Strategy balance is less than the amount needed");

}
```

As a result, wethRedeemed is always calculated as 0. This incorrectly triggers the StrategyDeallocationLoss event on every deallocation (since amount will be > 0), even when no loss has occurred.

## Impact Details

Contract would fail to deliver promised return

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol?utm\\_source=immunefi>

## Proof of Concept

## Proof of Concept

Copy and paste to MorphoYearnOGWETHStrategy.t.sol

```solidity
    function test_strategy_deallocate_succeeds(uint256 amountToAllocate) public {
        amountToAllocate = bound(amountToAllocate, 1e18, testConfig.vaultInitialDeposit);
        uint256 amountToDeallocate = IMYTStrategy(strategy).previewAdjustedWithdraw(amountToAllocate);
        vm.startPrank(vault);
        deal(WETH, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets > 0, "Initial real assets is 0");
        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
        uint256 vaultBalanceBefore = IERC20(WETH).balanceOf(vault);
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", vault);
        uint256 vaultBalanceAfter = IERC20(WETH).balanceOf(vault);
        assertApproxEqAbs(vaultBalanceAfter - vaultBalanceBefore, amountToDeallocate, 1e18);
        vm.stopPrank();
    }

    function test_allocated_position_generated_yield() public {
            vm.startPrank(address(vault));
            uint256 amount = 100 ether;
            deal(WETH, address(strategy), amount);
            bytes memory prevAllocationAmount = abi.encode(0);
            IMYTStrategy(strategy).allocate(prevAllocationAmount, amount, "", address(vault));
            uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
            assertApproxEqAbs(initialRealAssets, amount, 1e18);
            vm.warp(block.timestamp + 180 days);
            uint256 realAssets = IMYTStrategy(strategy).realAssets();
            assertGt(realAssets, initialRealAssets);
            vm.stopPrank();
        }

}
```
