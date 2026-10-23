# 56806 sc insight broken withdrawal logic in aavev3arbwethstrategy permanently locks user funds

**Submitted on Oct 20th 2025 at 20:47:17 UTC by @haidoka017 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56806
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/arbitrum/AaveV3ARBWETHStrategy.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

While i am doing investigating further, I discovered also this same vulnerability exists in AaveV3ARBWETHStrategy like . The WETH strategy has identical flawed logic that prevents withdrawals due to Aave V3's aToken rounding behavior.

## Vulnerability Details

After submitting my report on the USDC strategy, I went back to check if any other Aave strategies in the codebase might have similar problems. Turns out AaveV3ARBWETHStrategy has the exact same issue - the withdrawal logic doesn't properly handle how Aave V3 mints and redeems aTokens. Let me explain what I've been seeing in my tests. When I deposit WETH through the strategy, Aave's supply() function doesn't just give you back the same amount in aTokens. It calculates based on something called the liquidity index, which is how Aave tracks interest accrual. Every time I ran tests on a forked Arbitrum mainnet - and I mean every single time - depositing 100 WETH would result in receiving slightly less than 100 aWETH. Usually around 99.999 WETH worth. Here's the problematic code in AaveV3ARBWETHStrategy.sol:

```
function _deallocate(uint256 amount) internal override returns (uint256) {
        uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
        // withdraw exact underlying amount back to this adapter
        pool.withdraw(address(weth), amount, address(this));
        uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
        uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
        if (wethRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
        }
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
        TokenUtils.safeApprove(address(weth), msg.sender, amount);
        return amount;
    }
```

This is functionally identical to what I saw in the USDC strategy. The logic seems contradictory to me - the code detects when wethRedeemed < amount and even emits an event about it, but then immediately requires the balance to be >= amount. If we just detected that less was redeemed, how is that balance check supposed to pass? Now, I know there's a test case called test\_strategy\_deallocate\_reverts\_due\_to\_slippage that expects reverts to happen. But I don't think that test is documenting this as acceptable behavior - it's just verifying that the revert occurs. The fact that there's a StrategyDeallocationLoss event suggests someone on the team thought about handling partial withdrawals, but the implementation never actually does that. What actually happens when I run my PoC:

I deposit 100 WETH through the strategy Aave mints aWETH based on the current liquidity index Due to Aave's rounding, I get approximately 99.999 WETH worth of aWETH back When I try to withdraw, the strategy attempts to pull the full 100 WETH But only \~99.999 aWETH are available in the contract Aave's pool reverts with error 0x47bc4b2c before we even reach the strategy's require statement User funds are stuck

## Impact Details

The WETH strategy has similar caps to USDC (configurable up to significant amounts) WETH is a high-value asset - even small amounts represent substantial USD value No recovery mechanism exists without a protocol upgrade The StrategyDeallocationLoss event suggests the developers anticipated partial withdrawals, but the implementation doesn't handle them

Additional considerations for WETH: WETH deposits tend to be larger in absolute value than USDC (since 1 WETH ≈ thousands of dollars). This means:

Even a 0.001 WETH rounding loss per transaction is \~$2-3 at current prices Larger deposits will see proportionally larger rounding discrepancies The cumulative effect across multiple users could lock significant value

## References

File: AaveV3ARBWETHStrategy.sol

Function: \_deallocate()

GitHub: <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/arbitrum/AaveV3ARBWETHStrategy.sol>

## Proof of Concept

## Proof of Concept

```
function testExploit_AaveV3ARBWETHStrategyvulnerability() public { 
    
   
    uint256 depositAmount = 100e18; 
    vm.startPrank(vault);
    deal(WETH, address(strategy), depositAmount);
    bytes memory prevAllocation = abi.encode(0);
    IMYTStrategy(strategy).allocate(prevAllocation, depositAmount, "", address(vault));
    uint256 realAssets = IMYTStrategy(strategy).realAssets();
    assertGt(realAssets, 0, "Should have assets in Aave");
    uint256 withdrawAmount = depositAmount;  // User thinks they deposited 100K
    bytes memory prevAllocation2 = abi.encode(depositAmount);
    vm.expectRevert();  
    IMYTStrategy(strategy).deallocate(prevAllocation2, withdrawAmount, "", address(vault));
    vm.stopPrank();
    }
```

and launch :

forge test --match-test testExploit\_AaveV3ARBWETHStrategyvulnerability -vvvv
