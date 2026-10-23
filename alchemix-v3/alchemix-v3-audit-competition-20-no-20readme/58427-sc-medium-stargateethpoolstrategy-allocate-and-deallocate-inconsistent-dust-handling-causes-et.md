# 58427 sc medium stargateethpoolstrategy allocate and deallocate inconsistent dust handling causes eth to be permanently locked in strategy contract

**Submitted on Nov 2nd 2025 at 09:09:43 UTC by @joicygiore for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58427
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/StargateEthPoolStrategy.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The `StargateEthPoolStrategy::_allocate()` and `_deallocate()` functions exhibit inconsistent logic in handling ETH precision rounding (dust), leading to unwrapped ETH being permanently locked in the strategy contract. This issue arises because the dust (ETH less than 1e12) is not appropriately handled, Over time, the dust will continue to expand, resulting in financial losses.

## Vulnerability Details

In the `StargateEthPoolStrategy::_allocate()` function, when a deposit leaves dust (less than 1e12), the contract emits an event instead of recovering or handling the remaining amount:

```js
    function _allocate(uint256 amount) internal override returns (uint256) {
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "not enough WETH");
        // unwrap to native ETH for Pool Native
        weth.withdraw(amount);
        uint256 amountToDeposit = (amount / 1e12) * 1e12;
        uint256 dust = amount - amountToDeposit;
        if (dust > 0) {
            emit StrategyAllocationLoss("Strategy allocation loss due to rounding.", amount, amountToDeposit);
        }
        pool.deposit{value: amountToDeposit}(address(this), amountToDeposit);
        return amount;
    }
```

However, in the `StargateEthPoolStrategy::_deallocate()` function, the logic uses `ethRedeemed + ethBalanceBefore >= amount` to check if the contract has enough ETH, but only uses `ethRedeemed` as the parameter for wrapping with `weth`. The `dust` portion is left unwrapped and becomes permanently locked in the contract:

```js

    function _deallocate(uint256 amount) internal override returns (uint256) {
        // Compute LP needed ∝ TVL to withdraw `amount` underlying
        // For Stargate, LP tokens are 1:1 with underlying
        // So we can just redeem the amount directly
        uint256 lpBalance = lp.balanceOf(address(this));
        uint256 lpNeeded = amount; // 1:1 ratio


        // Cap at available LP balance
        if (lpNeeded > lpBalance) {
            lpNeeded = lpBalance;
        }


        // Redeem LP to native ETH, then wrap back to WETH
        lp.approve(address(pool), lpNeeded);
@>        uint256 ethBalanceBefore = address(this).balance;
        pool.redeem(lpNeeded, address(this));
        uint256 ethBalanceAfter = address(this).balance;
        uint256 ethRedeemed = ethBalanceAfter - ethBalanceBefore;
        if (ethRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss which includes rounding loss.", amount, ethRedeemed);
        }
@>        if (ethRedeemed + ethBalanceBefore >= amount) {
            weth.deposit{value: ethRedeemed}();
        }
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
        TokenUtils.safeApprove(address(weth), msg.sender, amount);
        return amount;
    }
```

## Impact Details

When multiple allocate/deallocate cycles occur, the unwrapped ETH dust accumulates in the contract, leading to funds being locked permanently. This issue can result in inaccurate accounting and operational inefficiencies, as the contract’s available balance will always be slightly lower than required for withdrawals.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/optimism/StargateEthPoolStrategy.sol#L42-L54>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/optimism/StargateEthPoolStrategy.sol#L55-L82>

## Proof of Concept

## Proof of Concept

Add the following test to `src/test/strategies/StargateEthPoolStrategy.t.sol` and run it:

```js
    function test_strategy_dust(uint256 amountToAllocate) public {
        amountToAllocate = bound(amountToAllocate, 1e18, testConfig.vaultInitialDeposit);
        // Ensure there's dust by adding a non-zero remainder
        // This makes the amount not fully divisible by 1e12
        uint256 dust = amountToAllocate % 1e12;
        if (dust == 0) {
            // If by chance it's divisible, add some dust
            amountToAllocate += 1; // or any value < 1e12
        }
        uint256 amountToDeallocate = amountToAllocate - (amountToAllocate % 1e12);
        uint256 firstDustAmount;
        uint256 secondDustAmount;
        vm.startPrank(vault);
        for(uint256 i; i < 2; ++i) {
            deal(WETH, strategy, amountToAllocate);
            bytes memory prevAllocationAmount = abi.encode(0);
            IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
            uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
            require(initialRealAssets > 0, "Initial real assets is 0");
            bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
            IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
            if (i == 0) {
                firstDustAmount = address(strategy).balance;
            } else {
                secondDustAmount = address(strategy).balance;
            }  
        }
        vm.stopPrank();  
        assertGt(secondDustAmount, firstDustAmount); // ❌
    }
```
