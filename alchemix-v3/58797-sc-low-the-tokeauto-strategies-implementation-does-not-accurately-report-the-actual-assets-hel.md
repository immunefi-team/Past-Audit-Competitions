# 58797 sc low the tokeauto strategies implementation does not accurately report the actual assets held by the strategy

**Submitted on Nov 4th 2025 at 13:57:23 UTC by @aman for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58797
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The strategy includes a `realAssets` function intended to report the current assets held, but due to incorrect implementation in the `TokeAutoEth` and `TokeAutoUSDStrategy` contracts, it consistently reports a higher value than what can actually be redeemed.

## Vulnerability Details

Let have a look how the `realAssets()` calcualte the assets held in strategy:

```solidity
/v3-poc/src/strategies/mainnet/TokeAutoEth.sol:152
152:     function realAssets() external view override returns (uint256) {
153:         uint256 shares = rewarder.balanceOf(address(this));
154:         uint256 assets = autoEth.convertToAssets(shares);
155:         return assets;
156:     }
```

At `line 153`, the code fetches the Toke shares from the rewarder contract and calls `autoEth::convertToAssets`, which returns the potential redeemable assets but not the actual assets.

In my PoC, I used the `previewRedeem` function on the `autoEth` contract to measure the difference, which was approximately `0.00117 WETH` for a 40 WETH deposit. this discrepancy grows with larger deposits.

## Impact Details

Overstating `realAssets` makes the strategy unreliable, undermines trust, and can mislead users about the actual funds available for withdrawal.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/strategies/mainnet/TokeAutoEth.sol#L146-L150>

## Proof of Concept

## Proof of Concept

**Note: This report assume that `report: 58728` recommended fix has been implemented** Add Following file to `test/strategies` Dir with the name `POC.t.sol` :

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
    function transfer(address to, uint256 value) external returns (bool);
    function approve(address spender, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
}

interface IAutopool {
    struct AssetBreakdown {
        uint256 totalIdle;
        uint256 totalDebt;
        uint256 totalDebtMin;
        uint256 totalDebtMax;
    }
    function getAssetBreakdown() external view returns (AssetBreakdown memory);
    function getDestinations() external view returns (address[] memory);
    function getDebtReportingQueue() external view returns (address[] memory);
}

// AutoEth's previewDeposit/previewRedeem are non-view in this codebase.
// Calling through a view IERC4626 interface triggers staticcall and fails.
interface IAutoEthNonView {
    function previewDeposit(uint256 assets) external returns (uint256 shares);
    function previewRedeem(uint256 shares) external returns (uint256 assets);
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

  
    // ==== Helpers ====

    function getTestConfig() internal pure override returns (TestConfig memory) {
        return TestConfig({vaultAsset: WETH, vaultInitialDeposit: 1000e18, absoluteCap: 10_000e18, relativeCap: 1e18, decimals: 18});
    }

    function createStrategy(address vault, IMYTStrategy.StrategyParams memory params) internal override returns (address) {
        return address(new MockTokeAutoEthStrategy(vault, params, TOKE_AUTO_ETH_VAULT, AUTOPILOT_ROUTER, REWARDER, WETH, ORACLE, MAINNET_PERMIT2));
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 22_089_302;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("MAINNET_RPC_URL");
    }
  


        function test_strategy_auto_slippage() public {
        uint256 amountToAllocate = 400e18;
        uint256 amountToDeallocate = IMYTStrategy(address(strategy)).previewAdjustedWithdraw(amountToAllocate);
        vm.startPrank(vault);
        deal(testConfig.vaultAsset, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        uint256 vaultShares = IERC20(REWARDER).balanceOf(strategy); // rewards holds the share of strategy in Auto pool
        uint256 previewWithdrawAmount = IAutoEthNonView(TOKE_AUTO_ETH_VAULT).previewRedeem(vaultShares); // calls the previewRedeem to check how much can we redeem in  assets
        console.log("The share the previewWithdrawAmount " , previewWithdrawAmount);
        console.log("The share the initialRealAssets  " , initialRealAssets);
        assertGt(initialRealAssets ,previewWithdrawAmount , "Initial real assets does not matched" );
         bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
         vm.stopPrank();
        assertEq(IERC20(WETH).balanceOf(strategy), previewWithdrawAmount , "Strategy Receive WETH Not Equal to previewWithdrawAmount");
        assertEq(IERC20(REWARDER).balanceOf(strategy) , 0 , "Strategy share in rewarder should be zero after full deallocation");
        assertNotEq(IERC20(WETH).balanceOf(strategy), initialRealAssets , "Strategy Receive WETH  Equal to realAssets Reported So No issue");


    }
    
}
```

run with command : `forge test --match-test test_strategy_auto_slippage -vvv --decode-internal`
