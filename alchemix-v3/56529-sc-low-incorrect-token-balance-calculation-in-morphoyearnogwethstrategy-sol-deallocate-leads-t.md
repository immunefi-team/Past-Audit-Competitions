# 56529 sc low incorrect token balance calculation in morphoyearnogwethstrategy sol deallocate leads to wrong event emitted every time

**Submitted on Oct 17th 2025 at 09:44:18 UTC by @Vanshika for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56529
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`MorphoYearnOGWETHStrategy.sol::_deallocate()` always calculates `wethRedeemed` as 0. Regardless of the result of the withdrawal, it will always emit the `StrategyDeallocationLoss` event. This might lead to off-chain errors, although it has no impact on-chain.

## Vulnerability Details

`MorphoYearnOGWETHStrategy.sol::_deallocate()` has the following code:

```
    function _deallocate(uint256 amount) internal override returns (uint256) {
        vault.withdraw(amount, address(this), address(this));
@>  uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
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

`wethBalanceBefore` and `wethBalanceAfter` are both calculated after the withdrawal call to the vault. They will return the same value, and so the `wethRedeemed` will always be 0. The `wethBalanceBefore` in this function should be calculated before the external call to the vault. Other strategies do this correctly.

## Impact Details

Incorrect event emitted for off-chain indexers or front-ends. No on-chain impact since the function uses the input `amount` for approval and return value and not the incorrectly calculated `wethRedeemed`.

## Proof of Concept

## Proof of Concept

Copy the given test in test/strategies/MorphoYearnOGWETHStrategy.t.sol, and run with any input values using the following command: `forge test --mt test_deallocate_POC -vvvv`

```
    function test_deallocate_POC() public {
        // using test inherited from BaseStrategyTest
        test_strategy_deallocate(1e18, 1e17); // any input will produce the same result
    }
```

In the call trace about halfway through, you will always see the following event even if there is no loss:

```
emit StrategyDeallocationLoss(message: "Strategy deallocation loss.", amountRequested: 999900000000000000 [9.999e17], actualAmountSent: 0)
```
