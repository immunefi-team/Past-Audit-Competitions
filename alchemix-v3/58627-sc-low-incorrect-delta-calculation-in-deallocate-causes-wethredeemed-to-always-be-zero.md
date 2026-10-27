# 58627 sc low incorrect delta calculation in deallocate causes wethredeemed to always be zero&#x20;

**Submitted on Nov 3rd 2025 at 17:35:52 UTC by @rand for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58627
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

When a deallocation from the strategy occurs, a logic error results in the variable `wethRedeemed` always being zero. This happens because `wethBalanceBefore`, which is used in the calculation of `wethRedeemed`, is set after a withdraw from the strategy's vault occurs rather than before. As a result, `wethRedeemed` is always zero and this breaks a sanity check in a `require()` statement as well as resulting in the event `StrategyDeallocationLoss` to always emit upon deallocation, regardless of if there was any actual loss or not.

## Vulnerability Details

In the function `_deallocate`, variable `wethRedeemed` is calculated by subtracting `wethBalanceBefore` from `wethBalanceAfter`. Both of these variables get the balance of WETH held inside the strategy and are meant to measure how much WETH was received from a vault withdrawal that occurs while deallocating. The issue is that the withdrawal happens before `wethBalanceBefore` is recorded. This means that when `wethBalanceBefore` and `wethBalanceAfter` are measured, they return the same value, and since `wethRedeemed` is calculated by subtracting them by each other, it will always equal zero.

The problematic code can be seen between L50 and L53 of the strategy's contract, and appears as follows:

```solidity
vault.withdraw(amount, address(this), address(this));
uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
```

A simple remediation would be to swap the places of L50 with L51, as demonstrated below, so that the value of `wethBalanceBefore`, and, by extension, `wethRedeemed`, is accurate.

```solidity
uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
vault.withdraw(amount, address(this), address(this));
uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
```

## Impact Details

Inside the smart contract, the value of `wethRedeemed` impacts how the event `StrategyDeallocationLoss` is emitted and it breaks a sanity check which ensures that the quantity of WETH being deallocated from the strategy is not greater than what the strategy has. A more in-depth explanation of what each of these impacts entail is below.

Because the value of `wethRedeemed` is always zero, the event `StrategyDeallocationLoss` will always emit. If any off-chain systems monitor this event, the data they gather will be incorrect which could lead to issues that range from minor to severe, depending on the functionality of the system in question. For example, a system that tracks strategy performance which uses this event as a data source would show endless deallocation losses, making the strategy's performance appear to be horrifically poor, potentially discouraging investors who would otherwise be interested in the strategy and misinforming risk systems on the health of the strategy.

The aforementioned sanity check reverts if the amount of WETH that was received by the strategy when withdrawing from the vault plus the strategy's prior holdings is not enough to cover the amount being deallocated. In its current state, the require statement is completely meaningless as it cannot confirm if the amount the vault transferred (`wethRedeemed`) + prior strategy holdings (`wethBalanceBefore`) is sufficient for covering the deallocation because `wethRedeemed` is zero and `wethBalanceBefore` holds the same value as `wethBalanceAfter`.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>

## Proof of Concept

## Proof of Concept

PoC, drop into `MorphoYearnOGWETHStrategy.t.sol` in the immunefi\_audit test suite. Computes, logs, and asserts both the contract's original delta `currentWethRedeemed` (expected == 0) and a correct delta `properWethRedeemed` (expected >0). Run with `forge test src/test/strategies/MorphoYearnOGWETHStrategy.t.sol --mt test_wethRedeemed -vvvv`. Attached image is a screenshot of the logs emitted on running.

```solidity
    function test_wethRedeemed_on_deallocating(uint256 amountToAllocate, uint256 amountToDeallocate) public {
        amountToAllocate = bound(amountToAllocate, 1e18, testConfig.vaultInitialDeposit);
        amountToDeallocate = amountToAllocate - 1e17;
        deal(WETH, strategy, amountToAllocate + 1e20);
        vm.startPrank(vault);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);

        // Get wethBalanceBefore prior to calling deallocate. This will be the correct value.
        uint256 properWethBalanceBefore = TokenUtils.safeBalanceOf(address(WETH), address(strategy));

        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));

        // Local computation of wethRedeemed. Logic mirrored from MorphoYearnOGWETHStrategy.sol. 
        uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(WETH), address(strategy));
        uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(WETH), address(strategy));
        uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore; // always 0; wethBalanceAfter and wethBalanceBefore have identical values.

        // Local computation of wethRedeemd but with the correct wethBalanceBefore value. 
        uint256 properWethRedeemed = wethBalanceAfter - properWethBalanceBefore;

        emit log_named_uint("properWethBalanceBefore", properWethBalanceBefore);
        emit log_named_uint("properWethBalanceAfter", wethBalanceAfter); 
        emit log_named_uint("properWethRedeemed", properWethRedeemed);

        emit log_named_uint("currentWethBalanceBefore", wethBalanceBefore);
        emit log_named_uint("currentWethBalanceAfter", wethBalanceAfter);
        emit log_named_uint("currentWethRedeemed", wethRedeemed);
        
        //properWethRedeemed's implementation is correct, wethRedeemed uses the original incorrect implementation.
        assertNotEq(properWethRedeemed, 0);
        assertEq(wethRedeemed, 0);
    
        vm.stopPrank();
    }
```
