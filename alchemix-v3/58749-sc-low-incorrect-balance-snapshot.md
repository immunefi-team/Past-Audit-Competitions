# 58749 sc low incorrect balance snapshot

**Submitted on Nov 4th 2025 at 11:40:51 UTC by @Anirruth for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58749
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

MorphoYearnOGWETH.\_deallocate measures the strategy’s WETH balance “before” the withdraw after the withdraw has already happened. This makes the computed redeemed amount always zero and emits a misleading “loss” event.

## Vulnerability Details

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

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol#L49-L61>

Because wethBalanceBefore is read after vault.withdraw, wethRedeemed computes to 0 even when funds arrive. This triggers a spurious loss event and renders the first require equivalent to the second (both reduce to “balanceAfter ≥ amount”).

Correct approach: snapshot wethBalanceBefore before vault.withdraw, then compute wethRedeemed = wethBalanceAfter - wethBalanceBefore.

## Impact Details

Wrongly emit a loss event.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol#L49-L61>

## Proof of Concept

## Proof of Concept

Paste the following test in MorphoYearnOGWETHStrategy.t.sol.

```
function test_deallocate_emits_zero_redeemed_event_but_balance_increases() public {
        // Arrange: allocate some WETH to the strategy so it holds a vault position
        uint256 amountToAllocate = 10e18;
        vm.startPrank(vault);
        deal(WETH, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets > 0, "initial realAssets == 0");

        // We plan to deallocate half (adjusted for slippage per strategy preview)
        uint256 target = amountToAllocate / 2;
        uint256 amountToDeallocate = IMYTStrategy(strategy).previewAdjustedWithdraw(target);

        // Record strategy WETH balance before deallocation
        uint256 balBefore = TokenUtils.safeBalanceOf(WETH, strategy);

        // Expect the flawed event with actualAmountSent == 0 due to wrong before/after ordering
        vm.expectEmit(true, true, true, true);
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amountToDeallocate, 0);

        // Act: deallocate
        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));

        // Assert: strategy's WETH balance increased (withdraw succeeded),
        // yet the event reported 0 redeemed due to the measurement bug
        uint256 balAfter = TokenUtils.safeBalanceOf(WETH, strategy);
        assertGt(balAfter, balBefore, "strategy balance did not increase after withdraw");
        vm.stopPrank();
    }
```
