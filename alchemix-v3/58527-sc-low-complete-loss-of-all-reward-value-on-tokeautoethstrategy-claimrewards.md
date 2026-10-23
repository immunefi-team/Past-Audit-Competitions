# 58527 sc low complete loss of all reward value on tokeautoethstrategy claimrewards

**Submitted on Nov 3rd 2025 at 02:43:33 UTC by @gizzy for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58527
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The `TokeAutoEthStrategy` claims rewards in TOKE\_MAC tokens and sends them directly to the vault, but the vault only expects and handles WETH (its underlying asset). This causes TOKE\_MAC reward tokens to become permanently stuck in the vault with no way to utilize or recover them, resulting in a complete loss of protocol rewards.

## Vulnerability Details

```solidity
function _claimRewards() internal override returns (uint256 rewardsClaimed) {
    rewardsClaimed = rewarder.earned(address(this));
    rewarder.getReward(address(this), address(MYT), false);
}
```

The Tokemac rewarder (`IMainRewarder`) distributes rewards in **TOKE\_MAC** tokens:

* Rewarder address: `0x60882D6f70857606Cdd37729ccCe882015d1755E`
* Reward token: `0x2e9d63788249371f1DFC918a52f8d799F4a38C94` (TOKE\_MAC)

However, the vault (MYT) is designed to handle only **WETH** as its underlying asset:

* Vault asset: `0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2` (WETH)

The `getReward()` function signature from `IMainRewarder`:

```solidity
function getReward(address account, address recipient, bool claimExtras) external;
```

Parameters used:

* `account` = `address(this)` (the strategy contract)
* `recipient` = `address(MYT)` (the vault) ⚠️ **WRONG RECIPIENT**
* `claimExtras` = `false`

This sends TOKE\_MAC tokens directly to the vault, which has no mechanism to handle them.

The Morpho Vault V2 (`MYT`) is designed to:

* Only accept and manage its underlying asset (WETH)
* Has no functionality to handle arbitrary ERC20 tokens
* Has no recovery mechanism for tokens sent by mistake

## Impact Details

All TOKE\_MAC reward tokens become permanently stuck and unusable

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/mainnet/TokeAutoEth.sol#L95C4-L98C6>

## Proof of Concept

## Proof of Concept

copy and paste in `TokeAutoETHStrategy.t.sol`

Run

```bash
forge test --match-path "src/test/strategies/TokeAutoETHStrategy.t.sol"  --evm-version cancun --mt testClaimRewards -vvvv
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;
// Adjust these imports to your layout

import {TokeAutoEthStrategy} from "src/strategies/mainnet/TokeAutoEth.sol";
import {BaseStrategyTest} from "../libraries/BaseStrategyTest.sol";
import {IMYTStrategy} from "../../interfaces/IMYTStrategy.sol";
import {console} from "forge-std/console.sol";
import {IMainRewarder} from "../../strategies/interfaces/ITokemac.sol";
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
    address public constant TOKE_MAC = 0x2e9d63788249371f1DFC918a52f8d799F4a38C94;

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

    //Shows that the reward token is TOKE_MAC in ethereum mainnet

    function testClaimRewardsSendTokeMacToVault() public {
        vm.startPrank(address(vault));
        deal(testConfig.vaultAsset, strategy, testConfig.vaultInitialDeposit);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, testConfig.vaultInitialDeposit, "", address(vault));
        vm.stopPrank();
        address rewardToken = IMainRewarder(REWARDER).rewardToken();
        console.log("Reward token:", rewardToken);
        console.log("TOKE_MAC:", TOKE_MAC);
        assertEq(rewardToken, TOKE_MAC, "Reward token is not TOKE_MAC");

        
        vm.warp(block.timestamp + 30 days);
        uint256 rewardsClaimed = IMYTStrategy(strategy).claimRewards();
        console.log("Rewards claimed:", rewardsClaimed);
    }
}
```
