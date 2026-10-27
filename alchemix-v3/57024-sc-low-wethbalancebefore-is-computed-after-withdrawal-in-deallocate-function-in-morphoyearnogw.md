# 57024 sc low wethbalancebefore is computed after withdrawal in deallocate function in morphoyearnogwethstrategy contract leading to systematic strategydeallocationloss event emission&#x20;

**Submitted on Oct 22nd 2025 at 18:59:02 UTC by @Tadev for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57024
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The MorphoYearnOGWETHStrategy contract implements the `_deallocate` function as follows:

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

Contrary to all other strategies in scope, this one computes `wethBalanceBefore` after calling `vault.withdraw` . This means `wethBalanceBefore == wethBalanceAfter` and therefore `wethRedeemed == 0`. Hence, the line `emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);` is always executed, emitting the event of deallocation loss during each deallocation.

## Vulnerability Details

The vulnerability lies in the incorrect logic in `_deallocate`. The snippet:

```
        vault.withdraw(amount, address(this), address(this));
        uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
        uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
```

should be :

```
        uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
        vault.withdraw(amount, address(this), address(this));
        uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
```

## Impact Details

The impact of this issue is low as it is related to incorrect logic in the contract, leading to wrong emission of events.

## Proof of Concept

## Proof of Concept

Please copy paste the following test in MorphoYearnOGWETHStrategyTest.t.sol file:

```
    function testStrategyDeallocationLoss(uint256 amountToAllocate, uint256 amountToDeallocate) public {
        amountToAllocate = bound(amountToAllocate, 1e18, testConfig.vaultInitialDeposit);
        amountToDeallocate = amountToAllocate;

        vm.startPrank(vault);
        deal(WETH, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));

        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets > 0, "Initial real assets is 0");
        
        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
        vm.warp(block.timestamp + 1 days);
        
        vm.expectEmit(strategy);
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amountToAllocate, 0);
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
        vm.stopPrank();
    }
```

This test:

* deposits `amountToAllocate` in the vault
* withdraws `amountToAllocate` 1 day later

Deallocation is successful, but the `StrategyDeallocationLoss` event is emitted with a value of 0 for the actual reedemed amount, which is incorrect.
