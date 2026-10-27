# 58425 sc high missing slippage protection when depositing to tokeauto strategies

**Submitted on Nov 2nd 2025 at 09:04:40 UTC by @Cyborg for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58425
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Theft of unclaimed yield
  * Permanent freezing of funds

## Description

## Brief/Intro

Missing slippage protection when depositing to TokeAuto strategies leading to funds loss of the allocacted funds.

## Vulnerability Details

The TokeAutoEth.sol and TokeAutoUSDStrategy.sol strategies are allocating funds to the AutopilotRouter and at this step the router returns amount of shares to be staked to the Rewarder contract, but the problem is that the deposit action to the AutopilotRouter has hardcoded 0 slippage value. The router deposit method is `function depositMax(IERC4626 vault, address to, uint256 minSharesOut)` and the `minSharesOut` is passed as 0 value:

* <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol#L59>
* <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoUSDStrategy.sol#L45>

This is a huge risk for the strategy to receive lesser amount of shares than the actual expected. Reasons could be many - manipulations of the totalSupply or a price oracle, sandwich attacks, protocol insolvency, etc. The strategy should protect the allocated funds at any cost and it should revert in the case of the router returning lesser shares than expected.

## Impact Details

Allocated funds are at manipulation risk, because both strategies TokeAutoEth.sol and TokeAutoUSDStrategy.sol are not safe due to missing slippage protection. Impacted strategies:

* <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoUSDStrategy.sol>
* <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>

## Recommendation

Consider adding slippage protection for the both impacted strategies. The slippage should be defined off-chain and then passed to the smart contracts, because on-chain generated slippage is not fully secure.

## Proof of Concept

## Proof of Concept

Create test file `src/test/strategies/TokeAutoETHStrategy.MissingSlippageProtection.t.sol` and run the following command `forge test src/test/strategies/TokeAutoETHStrategy.MissingSlippageProtection.t.sol -vv`:

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;
// Adjust these imports to your layout

import "forge-std/Test.sol";
import {TokeAutoEthStrategy} from "src/strategies/mainnet/TokeAutoEth.sol";
import {BaseStrategyTest} from "../libraries/BaseStrategyTest.sol";
import {IMYTStrategy} from "../../interfaces/IMYTStrategy.sol";
import {SafeERC20} from "../../libraries/SafeERC20.sol";

interface ITokeAutoVault {
    function deposit(uint256 assets, address receiver) external returns (uint256 shares);

    function convertToAssets(uint256 shares) external view returns (uint256 assets);
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
    address public whaleDepositor = address(0x1234);

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

    // Add any strategy-specific tests here
    function test_strategy_deallocate_reverts_due_to_slippage() public {
        vm.startPrank(whaleDepositor);
        deal(whaleDepositor, 1000 ether);
        uint256 whaleDepositorAmount = 100000000000 * 10 ** 18;
        deal(testConfig.vaultAsset, whaleDepositor, whaleDepositorAmount);
        SafeERC20.safeApprove(testConfig.vaultAsset, TOKE_AUTO_ETH_VAULT, type(uint256).max);
        uint256 whaleShares = ITokeAutoVault(TOKE_AUTO_ETH_VAULT).deposit(whaleDepositorAmount, whaleDepositor);

        bytes32 initialTotalSupply = vm.load(TOKE_AUTO_ETH_VAULT, bytes32(uint256(4)));
        console.log(uint256(initialTotalSupply), 'initialTotalSupply');

        /// creating a test scenario where the Toke Auto Eth Vault's totalSupply has been manipulated thus returning way less shares for deposited assets
        _manipulateTokeAuthEthVaultTotalSupply(uint256(1 * 10 ** 18));
        console.log(uint256(vm.load(TOKE_AUTO_ETH_VAULT, bytes32(uint256(4)))), 'current manipulated totalSupply');

        vm.startPrank(vault);
        uint256 amountToAllocateThroughStrategy = 100000 * 10 ** 18;
        deal(testConfig.vaultAsset, strategy, amountToAllocateThroughStrategy);

        require(IMYTStrategy(strategy).realAssets() == 0);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocateThroughStrategy, "", address(0));
        require(IMYTStrategy(strategy).realAssets() > 0, "ERROR: allocate method failed");

         /// return back the totalSupply to it's initial value
        _manipulateTokeAuthEthVaultTotalSupply(uint256(initialTotalSupply));
        console.log(uint256(vm.load(TOKE_AUTO_ETH_VAULT, bytes32(uint256(4)))), 'current totalSupply back to its initial value');

        console.log(IMYTStrategy(strategy).realAssets(), 'IMYTStrategy(strategy).realAssets()');

        /// the following two requires validate that the manipulation was done successfully, because the strategy deposited to the TOKE_AUTO_ETH_VAULT with 0 slippage as protection
        require(IMYTStrategy(strategy).realAssets() < amountToAllocateThroughStrategy, "ERROR: strategy didn't lose value");
        require(ITokeAutoVault(TOKE_AUTO_ETH_VAULT).convertToAssets(whaleShares) > whaleDepositorAmount, "ERROR: whale was not able to gain value");
    }

    function _manipulateTokeAuthEthVaultTotalSupply(uint256 value) internal {
        vm.store(TOKE_AUTO_ETH_VAULT, bytes32(uint256(4)), bytes32(uint256(value)));
    }
}
```

The PoC proves that after successful manipulation the strategy allocated funds are being stolen.
