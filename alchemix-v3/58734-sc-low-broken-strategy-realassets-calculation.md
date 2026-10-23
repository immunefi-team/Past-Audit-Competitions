# 58734 sc low broken strategy realassets calculation

**Submitted on Nov 4th 2025 at 10:19:35 UTC by @Cyborg for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58734
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Broken strategy realAssets calculation leading to wrong report of current deposit.

## Vulnerability Details

RE: Morpho's documentation ( <https://docs.morpho.org/learn/concepts/vault-v2/#1-the-adapter-model-a-universal-gateway-to-yield> ) - Function `realAssets()` returns the current value of all investments that the adapter manages. This allows the vault to automatically calculate total assets by aggregating across all adapters.

But that's not the case in strategies <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol> and <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoUSDStrategy.sol>. Inside of them the `realAssets()` method doesn't return the accurate value of the current investment thus passing the `realAssets()` output to the `deallocate` leads to revert.

Toke Auto Eth Vault protocol have different formulas to calculate allocations and currently the 2 strategies are not considering this in the calculation of the `realAssets()` - <https://github.com/Tokemak/v2-core-pub/blob/main/src/vault/libs/Autopool4626.sol#L54>

## Impact Details

Two of the in-scope strategies returning wrong amounts of the current allocation.

## Recommendation

Consider updating the `realAssets` method of both *TokeAutoEth.sol* and *TokeAutoUSDStrategy.sol* strategies:

```
function realAssets() external view returns (uint256) {
        uint256 shares = rewarder.balanceOf(address(this));
        uint256 assets = autoEth.convertToAssets(
            shares, 
            autoEth.totalAssets(IERC4626Like.TotalAssetPurpose.Withdraw), 
            autoEth.totalSupply(), 
            IERC4626Like.Rounding.Down
        );
        return assets;
    }

interface IERC4626Like is IERC4626 {
    function balanceOfActual(address account) external view returns (uint256);

    function convertToAssets(
        uint256 shares,
        uint256 totalAssetsForPurpose,
        uint256 supply,
        Rounding rounding
    ) external view returns (uint256 assets);

    function totalAssets(TotalAssetPurpose purpose) external view returns (uint256);

    enum Rounding {
        Down, // Toward negative infinity
        Up, // Toward infinity
        Zero // Toward zero
    }

    enum TotalAssetPurpose {
        Global,
        Deposit,
        Withdraw
    }
}
```

## Proof of Concept

## Proof of Concept

Create test file `src/test/strategies/TokeAutoETHStrategy.deallocate.t.sol` and run it with command `forge test src/test/strategies/TokeAutoETHStrategy.deallocate.t.sol -vv`:

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;
// Adjust these imports to your layout

import "forge-std/Test.sol";
import {TokeAutoEthStrategy} from "src/strategies/mainnet/TokeAutoEth.sol";
import {BaseStrategyTest} from "../libraries/BaseStrategyTest.sol";
import {IMYTStrategy} from "../../interfaces/IMYTStrategy.sol";
import {IMainRewarder, IAutopilotRouter} from "src/strategies/interfaces/ITokemac.sol";
import {IERC4626} from "lib/openzeppelin-contracts/contracts/interfaces/IERC4626.sol";

interface ITokeAutoEthStrategy {
    function rewarder() external view returns (address);
    function claimRewards() external returns (uint256);
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

contract TokeAutoETHStrategyTest is BaseStrategyTest {
    address public constant TOKE_AUTO_ETH_VAULT = 0x0A2b94F6871c1D7A32Fe58E1ab5e6deA2f114E56;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant MAINNET_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;
    address public constant AUTOPILOT_ROUTER = 0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21;
    address public constant REWARDER = 0x60882D6f70857606Cdd37729ccCe882015d1755E;
    address public constant ORACLE = 0x61F8BE7FD721e80C0249829eaE6f0DAf21bc2CaC;
    address public staker = address(0x123);

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
        return 22_089_302;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("MAINNET_RPC_URL");
    }

    function test_deallocate_fail() public {
        vm.startPrank(vault);
        uint256 amountToAllocate = 12345 * 10 ** 18;
        deal(testConfig.vaultAsset, strategy, amountToAllocate);

        require(IMYTStrategy(strategy).realAssets() == 0, "ERROR: realAsset isn't 0");

        /// staking to the REWARDER contract through the TokeAutoEthStrategy's allocate method
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(0));
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets > 0, "Initial real assets is 0");

        /// strategy deallocate doesn't work with passing the realAssets output as realAssets is wrong
        IMYTStrategy(strategy).deallocate(abi.encode(amountToAllocate), IMYTStrategy(strategy).realAssets(), "", address(vault));
    }

    function test_deallocate_success() public {
        vm.startPrank(vault);
        uint256 amountToAllocate = 12345 * 10 ** 18;
        deal(testConfig.vaultAsset, strategy, amountToAllocate);

        require(IMYTStrategy(strategy).realAssets() == 0, "ERROR: realAsset isn't 0");

        /// staking to the REWARDER contract through the TokeAutoEthStrategy's allocate method
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(0));
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets > 0, "Initial real assets is 0");

        /// strategy deallocate worked successfully, because realAssetsFixed has the correct value of the allocation
        IMYTStrategy(strategy).deallocate(abi.encode(amountToAllocate), realAssetsFixed(), "", address(vault));

        require(IMYTStrategy(strategy).realAssets() == 0, "ERROR: realAsset isn't 0");
        require(realAssetsFixed() == 0, "ERROR: realAssetsFixed isn't 0");
    }

    function realAssetsFixed() internal view returns (uint256) {
        uint256 shares = IMainRewarder(ITokeAutoEthStrategy(strategy).rewarder()).balanceOf(address(strategy));
        uint256 assets = IERC4626Like(TOKE_AUTO_ETH_VAULT).convertToAssets(
            shares, 
            IERC4626Like(TOKE_AUTO_ETH_VAULT).totalAssets(IERC4626Like.TotalAssetPurpose.Withdraw), 
            IERC4626Like(TOKE_AUTO_ETH_VAULT).totalSupply(), 
            IERC4626Like.Rounding.Down
        );
        return assets;
    }
}

interface IERC4626Like is IERC4626 {
    function convertToAssets(
        uint256 shares,
        uint256 totalAssetsForPurpose,
        uint256 supply,
        Rounding rounding
    ) external view returns (uint256 assets);

    function totalAssets(TotalAssetPurpose purpose) external view returns (uint256);

    enum Rounding {
        Down, // Toward negative infinity
        Up, // Toward infinity
        Zero // Toward zero
    }

    enum TotalAssetPurpose {
        Global,
        Deposit,
        Withdraw
    }
}
```

Test scenario `test_deallocate_fail` shows that `realAssets` is broken and test scenario `test_deallocate_success` shows that the fixed version `realAssetsFixed` works when requesting the strategy's `deallocate` method.
