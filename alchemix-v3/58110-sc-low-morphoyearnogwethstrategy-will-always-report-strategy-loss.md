# 58110 sc low morphoyearnogwethstrategy will always report strategy loss

**Submitted on Oct 30th 2025 at 18:02:54 UTC by @Django for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58110
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Protocol insolvency
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The strategy contracts within the AlchemixV3 system are designed to report when a strategy has incurred a loss to indicate the change to the redemption rate due to the bad debt. However, due to incorrect order of token balance checks, the event `StrategyDeallocationLoss` will be emitted every time the funds are deallocated from the strategy.

## Vulnerability Details

See the `_deallocate()` function in the `MorphoYearnOGWETHStrategy.sol` contract:

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
        vault.withdraw(amount, address(this), address(this));
        uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
        uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
        uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
        if (wethRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
        }
```

The `wethBalanceBefore` is calculated the line before `wethBalanceAfter` is calculated. The `vault.withdraw()` call should be in the middle of these two calls to accurately calculate the balance difference.

## Impact Details

The strategy will report a loss on every deallocation, leading to confusion around the protocol incurring bad debt and the resulting redemption rate.

## Proof of Concept

## Proof of Concept

***forge test --mt test\_strategy\_always\_emits\_deallocation\_loss -vv***

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import "../libraries/BaseStrategyTest.sol";
import {MorphoYearnOGWETHStrategy} from "../../strategies/mainnet/MorphoYearnOGWETH.sol";
import {MYTStrategy} from "../../MYTStrategy.sol";

contract MockMorphoYearnOGWETHStrategy is MorphoYearnOGWETHStrategy {
    constructor(address _myt, StrategyParams memory _params, address _vault, address _weth, address _permit2Address)
        MorphoYearnOGWETHStrategy(_myt, _params, _vault, _weth, _permit2Address)
    {}
}

contract MorphoYearnOGWETHStrategyTest is BaseStrategyTest {
    address public constant MORPHO_YEARN_OG_VAULT = 0xE89371eAaAC6D46d4C3ED23453241987916224FC;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant MAINNET_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;

    event StrategyDeallocationLoss(string message, uint256 amountRequested, uint256 actualAmountSent);

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

    function test_strategy_deallocate_reverts_due_to_slippage(uint256 amountToAllocate, uint256 amountToDeallocate) public {
        amountToAllocate = bound(amountToAllocate, 1e18, testConfig.vaultInitialDeposit);
        amountToDeallocate = amountToAllocate;
        vm.startPrank(vault);
        deal(WETH, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets > 0, "Initial real assets is 0");
        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
        vm.expectRevert();
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
        vm.stopPrank();
    }

    function test_strategy_always_emits_deallocation_loss(uint256 amountToAllocate, uint256 amountToDeallocate) public {
        amountToAllocate = bound(amountToAllocate, 1e18, testConfig.vaultInitialDeposit);
        amountToDeallocate = amountToAllocate / 2;
        vm.startPrank(vault);
        deal(WETH, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
        vm.expectEmit(true, true, true, true);
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amountToDeallocate, 0);
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
        vm.stopPrank();
    }

    /*     

    function test_allocated_position_generated_yield() public {
            vm.startPrank(address(vault));
            uint256 amount = 100 ether;
            deal(WETH, address(mytStrategy), amount);
            bytes memory prevAllocationAmount = abi.encode(0);
            mytStrategy.allocate(prevAllocationAmount, amount, "", address(vault));
            uint256 initialRealAssets = mytStrategy.realAssets();
            emit MorphoYearnOGWETHStrategyTestLog("initialRealAssets", initialRealAssets);
            assertApproxEqAbs(initialRealAssets, amount, 1e18);
            vm.warp(block.timestamp + 180 days);
            uint256 realAssets = mytStrategy.realAssets();
            emit MorphoYearnOGWETHStrategyTestLog("realAssets", realAssets);
            assertGt(realAssets, initialRealAssets);
            vm.stopPrank();
        }
    */
}
```
