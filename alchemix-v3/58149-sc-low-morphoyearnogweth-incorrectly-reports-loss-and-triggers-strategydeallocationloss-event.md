# 58149 sc low morphoyearnogweth incorrectly reports loss and triggers strategydeallocationloss event

**Submitted on Oct 30th 2025 at 23:30:44 UTC by @magtentic for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58149
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`MorphoYearnOGWETH` will always emit the `StrategyDeallocationLoss` event as the calculations done will always lead to 0.

## Vulnerability Details

Currently in `_deallocate`, the contract checks a `wethBalanceBefore` and `wethBalanceAfter`, the problem with this is that the contract does this sequentially rather than before calling `vault.withdraw`.\
What this effectively does is that it will have the same value resulting in `wethRedeemed` always been 0.

```
    function _deallocate(uint256 amount) internal override returns (uint256) {
        vault.withdraw(amount, address(this), address(this));
        uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
        uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
        uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
        if (wethRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
        }
...
```

When this occurs, this triggers the event `StrategyDeallocationLoss`. Further down, there is then also a duplication of the check which would effectively be the same calculated values.

```
...
        require(wethRedeemed + wethBalanceBefore >= amount, "Strategy balance is less than the amount needed");
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
...

### Summary 
- The contract calls `vault.withdraw` first, then measures `wethBalanceBefore` and `wethBalanceAfter`.   
- Since both balances are measured after the withdrawal, `wethRedeemed` is always 0.  
- This triggers the `StrategyDeallocationLoss` event erroneously.

```

## Impact Details

* No user funds are lost, and the function executes correctly. However, it emits misleading `StrategyDeallocationLoss` events, which may confuse monitoring systems or off-chain analytics.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>

## Proof of Concept

## Proof of Concept

The following logs have been added to the output to show this.\
The logs confirm that the `wethRedeemed` value is 0 as per provided image.

```
...
+import "forge-std/console.sol";
...
    function _deallocate(uint256 amount) internal override returns (uint256) {
        vault.withdraw(amount, address(this), address(this));
        uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
        uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
+        console.log("Balance before : ", wethBalanceBefore);
+        console.log("Balance after : ", wethBalanceAfter);
        uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
        if (wethRedeemed < amount) {
+            console.log("Emit for 'Strategy deallocation loss.' : ", amount, wethRedeemed);
            emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
        }
        require(wethRedeemed + wethBalanceBefore >= amount, "Strategy balance is less than the amount needed");
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
+        console.log("(wethRedeemed + wethBalanceBefore >= amount) values : ", wethRedeemed, wethBalanceBefore, amount);
+        console.log("(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount) values : ", TokenUtils.safeBalanceOf(address(weth), address(this)), amount);
        TokenUtils.safeApprove(address(weth), msg.sender, amount);
        return amount;
    }
```

Proof Of Concept to trigger the above logs.

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import "../libraries/BaseStrategyTest.sol";
import {MorphoYearnOGWETHStrategy} from "../../strategies/mainnet/MorphoYearnOGWETH.sol";

contract MockMorphoYearnOGWETHStrategy is MorphoYearnOGWETHStrategy {
    constructor(address _myt, StrategyParams memory _params, address _vault, address _weth, address _permit2Address)
        MorphoYearnOGWETHStrategy(_myt, _params, _vault, _weth, _permit2Address)
    {}
}

contract MorphoYearnOGWETHStrategyTest is BaseStrategyTest {
    address public constant MORPHO_YEARN_OG_VAULT = 0xE89371eAaAC6D46d4C3ED23453241987916224FC;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant MAINNET_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;

    function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) {
        return IMYTStrategy.StrategyParams({
            owner: address(1),
            name: "MorphoYearnOGETH",
            protocol: "MorphoYearnOGETH",
            riskClass: IMYTStrategy.RiskClass.LOW,
            cap: 10_000e18,
            globalCap: 1e18,
            estimatedYield: 100e18,
            additionalIncentives: false,
            slippageBPS: 1
        });
    }

    function getTestConfig() internal pure override returns (TestConfig memory) {
        return TestConfig({vaultAsset: WETH, vaultInitialDeposit: 1000e18, absoluteCap: 10_000e18, relativeCap: 1e18, decimals: 18});
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 23_298_447;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("MAINNET_RPC_URL");
    }

    function createStrategy(address vault, IMYTStrategy.StrategyParams memory params) internal override returns (address) {
        return address(new MockMorphoYearnOGWETHStrategy(vault, params, MORPHO_YEARN_OG_VAULT, WETH, MAINNET_PERMIT2));
    }

    function testStrategyDeallocateIncorrectFunction(uint256 amountToAllocate, uint256 amountToDeallocate) public {
        amountToAllocate = bound(amountToAllocate, 1e18, testConfig.vaultInitialDeposit);
        amountToDeallocate = amountToAllocate / 2;
        vm.startPrank(vault);
        deal(WETH, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets > 0, "Initial real assets is 0");
        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
        vm.stopPrank();
    }
}
```
