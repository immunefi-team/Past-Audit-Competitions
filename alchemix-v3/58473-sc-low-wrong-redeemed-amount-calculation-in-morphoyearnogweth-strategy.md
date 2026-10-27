# 58473 sc low wrong redeemed amount calculation in morphoyearnogweth strategy

**Submitted on Nov 2nd 2025 at 14:55:36 UTC by @Davuka for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58473
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

When deallocation occurs in morphoYearnOGWETH strategy calculation of the redeemed weth is incorrectly calculated.

## Vulnerability Details

From the code below, it is observed that the `wethBalanceBefore` and `wethBalanceAfter` are called after the withdrawal of the amount from the Morpho vault. After the withdrawal, the `wethRedeemed` is calculated by subtracting `wethBalanceAfter` from `wethBalanceBefore`. But since both variables were called after collateral withdrawal, the `wethRedeemed` amount will always be zero.

```solidity
  function _deallocate(uint256 amount) internal override returns (uint256) {
        console.log("Hey dave you got to this point"); 
        vault.withdraw(amount, address(this), address(this));
        uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this)); //@audit >> wethBefore should come before vaultWithdraw
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

## Impact Details

Because the `wethRedeemed` amount will always be zero, which would be less than the withdrawn amount from the vault. This means the if condition will always evaluate to true, and the emitted event for designated conditions gets triggered every time. This emit will be misleading for off-chain monitoring system.

## References

//

## Proof of Concept

## Proof of Concept

Add this to src/test/strategies/MorphoYearnOGWETHStrategy.t.sol import {IERC20} from "openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";

```solidity
   event StrategyDeallocationLoss(string message, uint256 amountRequested, uint256 actualAmountSent);

        function test_wrong_reedem_calculation() public {
        uint256 amountToAllocate = 10e18;
        console.log("Strategy balance before allocation: ", IERC20(WETH).balanceOf(address(strategy)));

        vm.startPrank(vault);
        deal(WETH, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets > 0, "Initial real assets is 0");

        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
         
        vm.expectEmit();
        emit StrategyDeallocationLoss("Strategy deallocation loss.", initialRealAssets, 10);

        deal(WETH, strategy, 10e18);
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, initialRealAssets, "", address(vault));
        vm.stopPrank();

      
    }
```
