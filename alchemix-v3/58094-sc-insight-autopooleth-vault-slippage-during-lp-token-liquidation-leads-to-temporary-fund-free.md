# 58094 sc insight autopooleth vault slippage during lp token liquidation leads to temporary fund freezing

**Submitted on Oct 30th 2025 at 15:44:18 UTC by @dobrevaleri for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58094
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

The `TokeAutoEthStrategy::_deallocate()` function calls `AutopoolETH::redeem()` which may fail or return insufficient funds due to slippage occurring during LP token liquidation. This can result in temporary freezing of funds until the next rebalancing period.

## Vulnerability Details

The `TokeAutoEthStrategy` deposits WETH into the Tokemac AutopoolETH vault and stakes the received shares in a rewarder contract. During deallocation, the strategy calls `autoEth.redeem()` to withdraw the underlying WETH:

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 sharesNeeded = autoEth.convertToShares(amount);
    // ... withdraw from rewarder ...
    
    autoEth.redeem(sharesNeeded, address(this), address(this));
    
    uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
    if (wethRedeemed < amount) {
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
    }
    require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
    // ...
}
```

When users deposit into AutopoolETH, funds are allocated to different liquidity provider (LP) positions across various pools. The protocol maintains an idle amount for small redemptions, but larger redemptions require selling LP tokens to create the necessary liquidity.

During the LP token selling process within `AutopoolETH::redeem()`, slippage can occur due to:

1. Market conditions affecting LP token prices
2. Insufficient liquidity in the underlying pools
3. Large withdrawal amounts relative to idle reserves

According to the Tokemac documentation, if redemption cannot be fulfilled immediately due to insufficient idle funds, the funds may not be freed until the next rebalancing period. This can trap funds temporarily.

## Impact Details

If redemption cannot be processed due to insufficient idle liquidity, strategy's funds remain locked until AutopoolETH's next rebalancing cycle.

## References

<https://docs.auto.finance/developer-docs/contracts-overview/autopool-eth-contracts-overview/autopool-contracts-and-systems/autopools#estimating-withdraw-value>

<https://github.com/Tokemak/v2-core-pub/blob/de163d5a1edf99281d7d000783b4dc8ade03591e/src/vault/AutopoolETH.sol#L357-L390>

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "forge-std/Test.sol";
import {TokeAutoEthStrategy} from "../strategies/mainnet/TokeAutoEth.sol";
import {IMYTStrategy} from "../interfaces/IMYTStrategy.sol";
import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";

interface IMainRewarder {
    function balanceOf(address account) external view returns (uint256);
}

contract PoC_TokeAutoEthRedeemSlippage is Test {
    address public constant TOKE_AUTO_ETH_VAULT = 0x0A2b94F6871c1D7A32Fe58E1ab5e6deA2f114E56;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant MAINNET_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;
    address public constant AUTOPILOT_ROUTER = 0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21;
    address public constant REWARDER = 0x60882D6f70857606Cdd37729ccCe882015d1755E;
    address public constant ORACLE = 0x61F8BE7FD721e80C0249829eaE6f0DAf21bc2CaC;

    TokeAutoEthStrategy public strategy;
    address public mockVault = address(0xBEEF);
    uint256 public mainnetFork;

    function setUp() public {
        string memory rpc;
        try vm.envString("MAINNET_RPC_URL") returns (string memory envRpc) {
            rpc = envRpc;
        } catch {
            rpc = "https://eth.llamarpc.com";
        }
        mainnetFork = vm.createFork(rpc, 23_677_700);
        vm.selectFork(mainnetFork);

        IMYTStrategy.StrategyParams memory params = IMYTStrategy.StrategyParams({
            owner: address(this),
            name: "TokeAutoEth",
            protocol: "TokeAutoEth",
            riskClass: IMYTStrategy.RiskClass.MEDIUM,
            cap: 100_000e18,
            globalCap: 1e18,
            estimatedYield: 100e18,
            additionalIncentives: false,
            slippageBPS: 100 // 1% slippage tolerance
        });

        strategy = new TokeAutoEthStrategy(mockVault, params, TOKE_AUTO_ETH_VAULT, AUTOPILOT_ROUTER, REWARDER, WETH, ORACLE, MAINNET_PERMIT2);

        vm.etch(mockVault, bytes("mock"));
    }

    function test_PoC_RedeemSlippageTrapsFlunds() public {
        uint256 allocationAmount = 10 ether;

        deal(WETH, address(strategy), allocationAmount);
        vm.prank(mockVault);
        strategy.allocate(abi.encode(0), allocationAmount, "", mockVault);

        uint256 sharesStaked = IMainRewarder(REWARDER).balanceOf(address(strategy));
        uint256 realAssetsBefore = strategy.realAssets();

        assertGt(sharesStaked, 0, "Allocation failed");
        assertGt(realAssetsBefore, 0, "No assets in strategy");

        // It will currently revert due to unhandled slippage
        vm.prank(mockVault);
        vm.expectRevert(bytes("Strategy balance is less than the amount needed"));
        strategy.deallocate(abi.encode(allocationAmount), realAssetsBefore, "", mockVault);

        // Verify funds are trapped
        uint256 realAssetsAfter = strategy.realAssets();
        assertGt(realAssetsAfter, 0, "Funds should be trapped");
        assertEq(realAssetsAfter, realAssetsBefore, "Assets unchanged - trapped in strategy");
    }
}
```
