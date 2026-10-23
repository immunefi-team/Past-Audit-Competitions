# 58672 sc low incorrect balance check sequence&#x20;

**Submitted on Nov 4th 2025 at 00:20:21 UTC by @Shivansh for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58672
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The \_deallocate function contains a critical logic error where balanceBefore is calculated AFTER the vault.withdraw() call instead of before. This means balanceBefore and balanceAfter will always be identical values, causing wethRedeemed to always equal 0. Since the function checks if (wethRedeemed < amount) and reverts when true, this function will ALWAYS revert, making it impossible to withdraw funds from the strategy. This is a permanent denial of service that locks all funds in the vault.

## Vulnerability Details

The \_deallocate function contains a critical logic error where balanceBefore is calculated AFTER the vault.withdraw() call instead of before. This means balanceBefore and balanceAfter will always be identical values, causing wethRedeemed to always equal 0. Since the function checks if (wethRedeemed < amount) and reverts when true, this function will ALWAYS revert, making it impossible to withdraw funds from the strategy. This is a permanent denial of service that locks all funds in the vault.

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
        vault.withdraw(amount, address(this), address(this));
        uint256 balanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
        uint256 balanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
        uint256 wethRedeemed = balanceAfter - balanceBefore;
        
        if (wethRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
            revert("StrategyDeallocationLoss");
        }
        
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
        TokenUtils.safeApprove(address(weth), msg.sender, amount);
        return amount;
    }
```

## Impact Details

All funds deposited into the strategy become permanently locked. Users cannot withdraw their WETH from the vault, resulting in complete loss of funds. The strategy becomes unusable after any allocation since deallocations will always fail.

## References

Link : - <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/mainnet/MorphoYearnOGWETH.sol#L51>

## Proof of Concept

## Proof of Concept

For running the test , Go in a file in src/test/strategies/MorphoYearnOGWETHStrategy.t.sol , delete the old code , then copy this given code and paste it there .

I used the setup for testing which is already in the file, where function is tested.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import "../libraries/BaseStrategyTest.sol";
import {MorphoYearnOGWETHStrategy} from "../../strategies/mainnet/MorphoYearnOGWETH.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {console} from "forge-std/console.sol";

event StrategyDeallocationLoss(string message, uint256 amountRequested, uint256 actualAmountSent);

contract MockMorphoYearnOGWETHStrategy is MorphoYearnOGWETHStrategy {
    constructor(address _myt, StrategyParams memory _params, address _vault, address _weth, address _permit2Address)
        MorphoYearnOGWETHStrategy(_myt, _params, _vault, _weth, _permit2Address)
    {}
}

contract MorphoYearnOGWETHStrategyTest is BaseStrategyTest {
    address public constant MORPHO_YEARN_OG_VAULT = 0xE89371eAaAC6D46d4C3ED23453241987916224FC;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant MAINNET_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;


    function test_balance_check_after_withdrawal() public {
        // Initial setup - we'll use 10 WETH
        uint256 amountToAllocate = 10 ether;
        vm.startPrank(vault);
        deal(WETH, strategy, amountToAllocate);
        
        // Step 1: Allocate WETH to Morpho Vault
        bytes memory allocateData = abi.encode(0);
        IMYTStrategy(strategy).allocate(allocateData, amountToAllocate, "", address(vault));

        // Step 2: Calculate actual balance after allocation
        uint256 allocatedShares = IERC20(MORPHO_YEARN_OG_VAULT).balanceOf(strategy);
        
        // Step 3: Try to deallocate half the amount (5 ETH)
        uint256 amountToDeallocate = amountToAllocate / 2; // 5 ETH
        bytes memory deallocateData = abi.encode(amountToDeallocate);
        
        // Set up event listener for StrategyDeallocationLoss
        vm.expectEmit(true, true, true, true);
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amountToDeallocate, 0);
        
        // This should revert with StrategyDeallocationLoss
        vm.expectRevert("StrategyDeallocationLoss");
        IMYTStrategy(strategy).deallocate(deallocateData, amountToDeallocate, "", address(vault));
        
        // Log the deallocation attempt details
        console.log("=== Strategy Deallocation Loss Details ===");
        console.log("Amount requested to deallocate:", amountToDeallocate);
        console.log("Actual amount redeemed: 0");
        console.log("=======================================");
        
        vm.stopPrank();
    }

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

}
```

after this run a command

## Running Tests

```bash
# Install dependencies
forge install

# Run specific balance check test
forge test --match-test test_balance_check_after_withdrawal

this test uses external contract to run, if have an issue running then get API from `alchemy` and try running with `RPC_URL`.
```
