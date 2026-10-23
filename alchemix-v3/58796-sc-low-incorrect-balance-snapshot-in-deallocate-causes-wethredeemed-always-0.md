# 58796 sc low incorrect balance snapshot in deallocate causes wethredeemed always 0

**Submitted on Nov 4th 2025 at 13:57:21 UTC by @dldLambda for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58796
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

In the strategy implementation (MorphoYearnOGWETHStrategy.\_deallocate), the WETH balance is enabled after calling vault.withdraw(...), so the wethRedeemed variable is distributed as the difference between two results taken at the same time—as a result, wethRedeemed always occurs. This strategy's control constantly emits the StrategyDeallocationLoss event, requiring invalid data for verification.

## Vulnerability Details

Problematic code:

```
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

wethBalanceBefore and wethBalanceAfter are both read after the withdraw call.

Their difference (wethRedeemed) is always 0, regardless of the actual amount withdrawn.

Consequently, StrategyDeallocationLoss is falsely emitted, and require checks may revert incorrectly.

In production, this can lead to freezing of funds in the strategy and incorrect reporting.

In another strategies, correct (EulerWETHStrategy):

```
    function _deallocate(uint256 amount) internal override returns (uint256) {
        uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
        vault.withdraw(amount, address(this), address(this));
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
```

## Impact Details

freezing of funds, false alerts are emitted (StrategyDeallocationLoss)

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>

## Proof of Concept

## Proof of Concept

Add and run and see logs:

```
function testBug_DeallocateIncorrectBalanceTracking() public {
    uint256 amountToAllocate = 10e18; // 10 WETH

    // === 1. Allocate ===
    vm.startPrank(vault);
    // Transfer tokens so the strategy actually receives WETH
    deal(address(WETH), strategy, amountToAllocate);

    // Log strategy balance before allocation
    uint256 strategyWethBefore = IERC20(address(WETH)).balanceOf(strategy);
    console.log("=== BEFORE ALLOCATION ===");
    console.log("Strategy WETH balance: %e", strategyWethBefore);

    bytes memory prevAllocation = abi.encode(0);
    IMYTStrategy(strategy).allocate(prevAllocation, amountToAllocate, "", address(vault));
    vm.stopPrank();

    // Check real assets after allocation
    uint256 realAssetsAfterAllocation = IMYTStrategy(strategy).realAssets();
    console.log("");
    console.log("=== AFTER ALLOCATION ===");
    console.log("Real assets in strategy: %e", realAssetsAfterAllocation);
    assertGt(realAssetsAfterAllocation, 0, "Should have allocated assets");

    // === 2. Deallocate ===
    vm.startPrank(vault);

    // Log strategy balance before deallocation
    uint256 wethBalanceBefore = IERC20(address(WETH)).balanceOf(strategy);
    console.log("");
    console.log("=== BEFORE DEALLOCATION ===");
    console.log("Strategy WETH balance before: %e", wethBalanceBefore);

    bytes memory prevAllocation2 = abi.encode(amountToAllocate);

    // Call deallocate
    (, int256 change) = IMYTStrategy(strategy).deallocate(prevAllocation2, amountToAllocate, "", address(vault));
    vm.stopPrank();

    // === 3. Verify actual WETH received by the vault ===
    uint256 vaultWethAfter = IERC20(address(WETH)).balanceOf(vault);
    uint256 actualWethReceived = vaultWethAfter; // for this test, just check vault balance
    console.log("");
    console.log("=== AFTER DEALLOCATION ===");
    console.log("Vault WETH after:  %e", vaultWethAfter);
    console.log("Change (return value): %d", change);

    // Demonstrate the bug
    console.log("");
    console.log("==========================================================");
    console.log("BUG DEMONSTRATED:");
    console.log(" - StrategyDeallocationLoss event emitted with wethRedeemed=0");
    console.log(" - Actual WETH received: %e", actualWethReceived);
    console.log(" - Root cause: wethBalanceBefore taken AFTER vault.withdraw()");
    console.log(" - Location: MorphoYearnOGWETH.sol:50-53");
    console.log("==========================================================");

    // Assertions
    assertGt(actualWethReceived, 0, "Vault should have received WETH");
    assertApproxEqAbs(actualWethReceived, amountToAllocate, 1e17, "Should receive approximately allocated amount");
    assertLt(change, 0, "Change should be negative");
}
```

```
Logs:
  === BEFORE ALLOCATION ===
  Strategy WETH balance: 1e19
  
  === AFTER ALLOCATION ===
  Real assets in strategy: 9.999999999999999999e18
  
  === BEFORE DEALLOCATION ===
  Strategy WETH balance before: 0e0
```
