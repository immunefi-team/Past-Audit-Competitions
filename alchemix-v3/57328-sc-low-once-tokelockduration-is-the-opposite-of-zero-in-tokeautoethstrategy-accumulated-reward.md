# 57328 sc low once tokelockduration is the opposite of zero in tokeautoethstrategy accumulated rewards in acctoke can be stuck

**Submitted on Oct 25th 2025 at 09:36:13 UTC by @pashap9990 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57328
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

### Finding Description and Impact

Anyone can call `MYTStrategy::claimRewards` to get rewards from the `TokeAutoETH` strategy. Nonetheless, rewards may remain indefinitely inaccessible as Toke's rewarder stakes them in accToke when `tokeLockDuration` exceeds zero, leading to a perpetual yield lock due to the absence of a mechanism to claim and unstake the rewards from `accToke`.

<https://vscode.blockscan.com/ethereum/0x60882D6f70857606Cdd37729ccCe882015d1755E>

```solidity
    function _getReward(address account, address recipient) internal {
        Errors.verifyNotZero(account, "account");
        Errors.verifyNotZero(recipient, "recipient");

        uint256 reward = earned(account);
        (IAccToke accToke, address tokeAddress) = (systemRegistry.accToke(), address(systemRegistry.toke()));

        // slither-disable-next-line incorrect-equality
        if (reward == 0) return;

        // if NOT toke, or staking is turned off (by duration = 0), just send reward back
        if (rewardToken != tokeAddress || tokeLockDuration == 0) {
            rewards[account] = 0;
            emit RewardPaid(account, recipient, reward);

            IERC20(rewardToken).safeTransfer(recipient, reward);
        } else if (accToke.isStakeableAmount(reward)) {
            rewards[account] = 0;
            emit RewardPaid(account, recipient, reward);
            // authorize accToke to get our reward Toke
            LibAdapter._approve(IERC20(tokeAddress), address(accToke), reward);

            // stake Toke
     @>>>       accToke.stake(reward, tokeLockDuration, recipient);
        }
    }
```

### Textual PoC

1- WETHs are staked in `TokeAutoETH`. 2- New rewards are notified in the rewarder. 3- `tokeLockDuration` is updated by an operator. 4-The `MYTStrategy::claimRewards` function can be invoked by any individual, as it is permissionless. Reward tokens are staked in `accToke`, as `tokeLockDuration` is not equal to zero. 5- The function `AccToke::addWETHRewards` is invoked to allocate rewards to stakers. As a result, staked rewards are irrevocable. Additionally, rewards in `accToke` are non-redeemable due to lack of any functionality in `TokeAutoEthStrategy` to unstake and claim rewards from `accToke`.

### Code Snippet

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;
// Adjust these imports to your layout

import {TokeAutoEthStrategy} from "src/strategies/mainnet/TokeAutoEth.sol";
import {BaseStrategyTest} from "../libraries/BaseStrategyTest.sol";
import {IMYTStrategy} from "../../interfaces/IMYTStrategy.sol";

interface IERC20 {
    function approve(address spender, uint256 amount) external returns (bool);
    function balanceOf(address a) external view returns (uint256);
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
contract TokeAutoETH {
    function depositMax(
        address vault,
        address to,
        uint256 minSharesOut
    ) public payable returns (uint256 sharesOut) {
        return 1000e18;
    }
}

contract Rewarder {
    uint lockDuration = 30 days;
    AccToke public acc;
    constructor() {
        acc = new AccToke();
    }
    function stake(address account, uint256 amount) public {

    }

    function balanceOf(address account) public returns(uint){
        return 100e18;
    }

    function getReward(address account, address recipient, bool claimExtras) public {
        if(lockDuration > 0 )
            acc.stake();
    }

    function earned(address strategy) public returns(uint){
        return 100e18;
    }
}

contract AccToke{
    uint reward;
    function stake() public {

    }

    function addWETHRewards(uint256 amount) external {
        reward = amount;
    }
    function collectRewards() external returns (uint256) {
        return reward;
    }
}


contract TokeAutoETHStrategyTest is BaseStrategyTest {
    address public constant TOKE_AUTO_ETH_VAULT = 0x0A2b94F6871c1D7A32Fe58E1ab5e6deA2f114E56;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant MAINNET_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;
    TokeAutoETH public router;
    Rewarder public rewarder;
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
        router = new TokeAutoETH();
        rewarder = new Rewarder();

        return address(new MockTokeAutoEthStrategy(vault, params, TOKE_AUTO_ETH_VAULT, address(router), address(rewarder), WETH, ORACLE, MAINNET_PERMIT2));
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 22_089_302;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("MAINNET_RPC_URL");
    }

    // Add any strategy-specific tests here


    function testRewardsCanBeLockedInTokeAutoStrategy() public {
        uint amountToAllocate = 100e18;
        vm.startPrank(vault);
        deal(testConfig.vaultAsset, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        IMYTStrategy(strategy).claimRewards();
        AccToke(rewarder.acc()).addWETHRewards(10e18);//suppose 10e18 weth will be transferred as a ereward to accToke

        assertEq(AccToke(rewarder.acc()).collectRewards(), 10e18);


    }
}

```
