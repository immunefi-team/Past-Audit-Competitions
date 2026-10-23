# 58702 sc high no slippage provided in auto strategy implementation will open room for mev attacks

**Submitted on Nov 4th 2025 at 06:29:43 UTC by @aman for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58702
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

While depositing into the **Auto Pool**, we use the router implementation from **Auto Finance**, which calls the `depositMax` function.\
This function accepts `minSharesOut` as a parameter; however, in our implementation, we pass `0` as the `minSharesOut` value.

This can lead to the **vault receiving fewer shares than expected** if the transaction gets delayed for a few blocks, as demonstrated in the **PoC**.\
Additionally, the `rebalancing` and `updateDebtReporting` calls can further affect the outcome depending on the **current state of the pool**.

## Vulnerability Details

This issue can occur in `TokeAutoEth` and `TokeAutoUSDStrategy`. But I will focus on `TokeAutoEth` in this reports. The vault will calls `_allocate` function with the given `amount`. The `router::depositMax` will deposit the assets in auto pool and return the shares to strategy contract.

```solidity
/v3-poc/src/strategies/mainnet/TokeAutoEth.sol:57
57:     function _allocate(uint256 amount) internal override returns (uint256) {
58:         require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than amount");
59:         TokenUtils.safeApprove(address(weth), address(router), amount);
60:         uint256 shares = router.depositMax(autoEth, address(this), 0); // @audit : no slippage protection here
61:         TokenUtils.safeApprove(address(autoEth), address(rewarder), shares);
62:         rewarder.stake(address(this), shares);
63:         return amount;
64:     }
```

The `router::depositMax`function takes 3 paramater the pool address , the receier address and the `minSharesOut`.

```solidity
    function depositMax(
        IAutopool vault,
        address to,
        uint256 minSharesOut
    ) public payable override returns (uint256 sharesOut) {
        IERC20 asset = IERC20(vault.asset());
        uint256 assetBalance = asset.balanceOf(msg.sender);
        uint256 maxDeposit = vault.maxDeposit(to);
        uint256 amount = maxDeposit < assetBalance ? maxDeposit : assetBalance;
        pullToken(asset, amount, address(this));

        approve(IERC20(vault.asset()), address(vault), amount);
        return deposit(vault, to, amount, minSharesOut);
    }
```

[AutopilotRouter::depositMax](https://vscode.blockscan.com/ethereum/0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE2).

From AutoPool Docs : Depending on the conditions of the Autopool, the overall market, and the timing of the debt reporting process slippage may be encountered on both entering and exiting the Autopool. It is very important to always check the shares received on entering, and the assets received on exiting, are greater than an expected amount. [4626-compliance#slippage](https://docs.auto.finance/developer-docs/integrating/4626-compliance#slippage).

## Impact Details

Due to no slippage the vault will receive less shares than expected.

## References

[TokeAutoEth](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/strategies/mainnet/TokeAutoEth.sol) [TokeAutoUSDStrategy](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/strategies/mainnet/TokeAutoUSDStrategy.sol)

## Proof of Concept

## Proof of Concept

Add the following file to `test/strategies` dir with the name `POC.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;
// Adjust these imports to your layout

import {TokeAutoEthStrategy} from "src/strategies/mainnet/TokeAutoEth.sol";
import {BaseStrategyTest} from "../libraries/BaseStrategyTest.sol";
import {IMYTStrategy} from "../../interfaces/IMYTStrategy.sol";
import {MYTStrategy} from "../../MYTStrategy.sol";
import {AlchemistAllocator} from "../../AlchemistAllocator.sol";
import {MockAlchemistAllocator} from "../mocks/MockAlchemistAllocator.sol";
import {IERC4626Like} from "src/strategies/mainnet/TokeAutoEth.sol";
import {IVaultV2} from "../../../lib/vault-v2/src/interfaces/IVaultV2.sol";
import {console} from "forge-std/console.sol";

interface IERC20 {
    function balanceOf(address account) external view returns (uint256);
}



contract MockTokeAutoEthStrategy is TokeAutoEthStrategy {
    constructor(
        address _myt,
        StrategyParams memory _params,
        address _autoEth,
        address _router,
        address _rewarder,
        address _weth,
        address _oracle,
        address _permit2Address
    ) TokeAutoEthStrategy(_myt, _params, _autoEth, _router, _rewarder, _weth, _oracle, _permit2Address) {}
}



contract POCTest is BaseStrategyTest {
    address public constant TOKE_AUTO_ETH_VAULT = 0x0A2b94F6871c1D7A32Fe58E1ab5e6deA2f114E56;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant MAINNET_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;
    address public constant AUTOPILOT_ROUTER = 0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21;
    address public constant REWARDER = 0x60882D6f70857606Cdd37729ccCe882015d1755E;
    address public constant ORACLE = 0x61F8BE7FD721e80C0249829eaE6f0DAf21bc2CaC;

    function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) {
        return IMYTStrategy.StrategyParams({
            owner: address(1),
            name: "TokeAutoEth",
            protocol: "TokeAutoEth",
            riskClass: IMYTStrategy.RiskClass.MEDIUM,
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

    function createStrategy(address vault, IMYTStrategy.StrategyParams memory params) internal override returns (address) {
        return address(new MockTokeAutoEthStrategy(vault, params, TOKE_AUTO_ETH_VAULT, AUTOPILOT_ROUTER, REWARDER, WETH, ORACLE, MAINNET_PERMIT2));
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 22024000;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("MAINNET_RPC_URL");
    }
  

    function test_strategy_auto_slippage() public {
        uint256 amountToAllocate = 40e18;
        uint256 expectedShares = IERC4626Like(TOKE_AUTO_ETH_VAULT).previewDeposit(amountToAllocate); // 1. the vault initiate the transaction to deposit assets in AutoPool
        console.log("Expected share the vaultShares would received " , expectedShares);
        vm.rollFork(block.number + (2)); // Assume the transaction has been in mempool for these 2 blocks
        uint256 receivedShares = IERC4626Like(TOKE_AUTO_ETH_VAULT).previewDeposit(amountToAllocate); // 2 . At this step suppose the transaction got mined 
        console.log("The share the vaultShares received " , receivedShares);
        assertLt(receivedShares, expectedShares , "No slippage happened");
        vm.startPrank(vault);
        deal(testConfig.vaultAsset, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets > 0, "Initial real assets is 0");
        uint256 vaultShares = IERC20(REWARDER).balanceOf(strategy);
        console.log("The share the vaultShares received " , vaultShares);
        assertEq(vaultShares , receivedShares , "shares not equal");
    }
    

}
```

And Run With command : `forge test --match-test test_strategy_auto_slippage -vvv` .
