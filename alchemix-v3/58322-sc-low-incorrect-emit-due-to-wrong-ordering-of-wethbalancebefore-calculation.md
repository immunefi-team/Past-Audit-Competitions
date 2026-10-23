# 58322 sc low incorrect emit due to wrong ordering of wethbalancebefore calculation

**Submitted on Nov 1st 2025 at 09:08:33 UTC by @SOPROBRO for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58322
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

In `MorphoYearnOGWETH::_deallocate`, the function incorrectly calculates `wethBalanceBefore` after performing the `vault.withdraw` call. This causes `wethRedeemed` to always evaluate to `0`, resulting in an incorrect emission of a `StrategyDeallocationLoss` event even when there is no actual loss.

## Vulnerability Details

```solidity
vault.withdraw(amount, address(this), address(this));
// @audit `wethBalanceBefore` is calculated after withdrawal
uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
if (wethRedeemed < amount) {
	emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
}
```

The issue is that `wethBalanceBefore` should represent the contract’s WETH balance **before** the withdrawal is executed.\
However, since it is calculated **after** the `vault.withdraw` call, both `wethBalanceBefore` and `wethBalanceAfter` are the same, leading to:

```solidity
wethRedeemed = wethBalanceAfter - wethBalanceBefore = 0
```

This triggers a false `StrategyDeallocationLoss` event regardless of whether the strategy incurred a real loss.

## Impact Details

The `StrategyDeallocationLoss` event will always be emitted, even when there are no losses, which may result in false alerts

## References

(Code Location) <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/mainnet/MorphoYearnOGWETH.sol#L50-L56>

## Proof of Concept

## Proof Of Concept

Add the following test to `MorphoYearnOGWETHStrategy.t.sol` and run in the console `forge test --mt test_strat_always_emits_strat_loss -vv`, and see that the following event will be emitted `emit StrategyDeallocationLoss("Strategy deallocation loss.", 5e18, 0);`

```solidity
event StrategyDeallocationLoss(string message, uint256 amountRequested, uint256 actualAmountSent);

function test_strat_always_emits_strat_loss() public {
	uint256 amountToAllocate = 10e18;
	uint256 amountToDeallocate = amountToAllocate / 2;
	vm.startPrank(vault);
	deal(testConfig.vaultAsset, strategy, amountToAllocate);
	bytes memory prevAllocationAmount = abi.encode(0);
	IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
	bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
	vm.expectEmit();
	emit StrategyDeallocationLoss("Strategy deallocation loss.", 5e18, 0);
	IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
}
```
