# 58410 sc low tokemak strategy deallocation causes toke token lockup

**Submitted on Nov 2nd 2025 at 02:11:44 UTC by @Another for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58410
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield
  * Permanent freezing of funds

## Description

## Brief/Intro

The `deallocate` functions in both `TokeAutoEthStrategy` and `TokeAutoUSDStrategy` automatically claim rewards when withdrawing from the Tokemak rewarder. However, the claimed TOKE tokens are either staked for the strategy contract or transferred directly to it, but the vault system has no functionality to handle these TOKE tokens either via withdrawals or staked token claiming. This results in rewards being permanently locked in the strategy contracts.

## Vulnerability Details

While TokeAutoEthStrategy.sol has a `claimRewards` function which claims rewards to the MYT, the `deallocate` function also claims rewards but claims for the strategy contract instead. The `deallocate` function calls `rewarder.withdraw(..., true)` with the `claim` parameter set to `true`, which automatically claims any accumulated TOKE rewards.

According to Tokemak's `AbstractRewarder` contract, when rewards are claimed:

* If the reward token is TOKE and `tokeLockDuration > 0`, the [TOKE is automatically staked in the `accToke`](https://github.com/Tokemak/v2-core-pub/blob/de163d5a1edf99281d7d000783b4dc8ade03591e/src/rewarders/AbstractRewarder.sol#L329) contract for the strategy address
* If staking is disabled (`tokeLockDuration == 0`), [the TOKE is transferred directly to the strategy contract.](https://github.com/Tokemak/v2-core-pub/blob/de163d5a1edf99281d7d000783b4dc8ade03591e/src/rewarders/AbstractRewarder.sol#L321)
* And from Etherscan, we can see tha the [reward token is TOKE.](https://etherscan.io/address/0x60882d6f70857606cdd37729ccce882015d1755e#readContract#F23)

The strategy system has no functionality to:

* Withdraw staked TOKE from `accToke` (incase it is staked)
* Withraw or utilize TOKE tokens held by the strategy (incase it is not staked).

In `TokeAutoEthStrategy.sol`:

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 sharesNeeded = autoEth.convertToShares(amount);
    uint256 actualSharesHeld = rewarder.balanceOf(address(this));
    uint256 shareDiff = actualSharesHeld - sharesNeeded;
    if (shareDiff <= 1e18) {
        sharesNeeded = actualSharesHeld;
    }
    // withdraw shares, claim any rewards
    rewarder.withdraw(address(this), sharesNeeded, true); //<==
    // ... rest of function
}
```

It's even worse in TokeAutoUSDStrategy.sol as it does not have a `claimRewards` function, so all rewards go to the strategy contract. In `TokeAutoUSDStrategy.sol`:

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 sharesNeeded = autoUSD.convertToShares(amount);
    uint256 actualSharesHeld = rewarder.balanceOf(address(this));
    uint256 shareDiff = actualSharesHeld - sharesNeeded;
    if (shareDiff <= 1e18) {
        sharesNeeded = actualSharesHeld;
    }
    // withdraw shares, claim any rewards
    rewarder.withdraw(address(this), sharesNeeded, true); //<==
    // ... rest of function
}
```

## Impact Details

Impact is high as the claimed TOKE rewards become permanently inaccessible because:

* The strategy contracts don't have functions to handle TOKE tokens
* The vault cannot interact with the `accToke` staking contract

## References

<https://github.com/Tokemak/v2-core-pub/blob/de163d5a1edf99281d7d000783b4dc8ade03591e/src/rewarders/AbstractRewarder.sol#L329>

<https://github.com/Tokemak/v2-core-pub/blob/de163d5a1edf99281d7d000783b4dc8ade03591e/src/rewarders/AbstractRewarder.sol#L321>

<https://etherscan.io/address/0x60882d6f70857606cdd37729ccce882015d1755e#readContract#F23>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/mainnet/TokeAutoUSDStrategy.sol#L62>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/mainnet/TokeAutoEth.sol#L76>

## Proof of Concept

## Proof of Concept

We create the test file below to simulate the scenario where TOKE rewards are claimed and sent to the strategy contract during deallocation locking them in the strategy permanently. In reality, the TOKE rewards would be staked in the `accToke` contract, and the strategy contract still does not have access to them.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

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

contract TokeAutoETHStrategyTest is BaseStrategyTest {
    address public constant TOKE_AUTO_ETH_VAULT = 0x0A2b94F6871c1D7A32Fe58E1ab5e6deA2f114E56;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant MAINNET_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;
    address public constant AUTOPILOT_ROUTER = 0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21;
    address public constant REWARDER = 0x60882D6f70857606Cdd37729ccCe882015d1755E;
    address public constant ORACLE = 0x61F8BE7FD721e80C0249829eaE6f0DAf21bc2CaC;

    address public constant TOKE_TOKEN = 0x2e9d63788249371f1DFC918a52f8d799F4a38C94;
IERC20 public tokeToken;
    uint256 public initialTokeBalance;


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
        tokeToken = IERC20(TOKE_TOKEN);
        return address(new MockTokeAutoEthStrategy(vault, params, TOKE_AUTO_ETH_VAULT, AUTOPILOT_ROUTER, REWARDER, WETH, ORACLE, MAINNET_PERMIT2));
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 22_089_302;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("MAINNET_RPC_URL");
    }

    function test_toke_rewards_become_locked_during_deallocation() public {
        uint256 amountToAllocate = 1e18; // 1 WETH
        
        // Record initial TOKE balance of strategy
        initialTokeBalance = tokeToken.balanceOf(strategy);
        emit log_named_uint("Initial TOKE balance of strategy", initialTokeBalance);

        // Allocate funds to strategy
        vm.startPrank(vault);
        deal(testConfig.vaultAsset, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        
        // Verify allocation worked
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        assertTrue(initialRealAssets > 0, "Strategy should have real assets after allocation");

        // Simulate TOKE reward accumulation by moving blocks forward
        // In a real scenario, rewards would accumulate over time
        vm.roll(block.number + 100000); // Move forward 100k blocks
        

        // Now deallocate - this will claim rewards for the strategy and cause TOKE lockup
        uint256 amountToDeallocate = amountToAllocate/2;
        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
        
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
        
        // Check TOKE balance after deallocation showing that TOKE rewards now exist in the strategy
        // In reality, the TOKE might be staked in accToke and not even in the strategy's balance

        uint256 finalTokeBalance = tokeToken.balanceOf(strategy);
        emit log_named_uint("Final TOKE balance of strategy", finalTokeBalance);

        vm.stopPrank();
    }
}

```
