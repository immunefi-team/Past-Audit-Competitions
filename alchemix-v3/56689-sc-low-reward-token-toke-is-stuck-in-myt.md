# 56689 sc low reward token toke is stuck in myt

**Submitted on Oct 19th 2025 at 13:57:24 UTC by @ox9527 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56689
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

The reward token for TokeAutoEthStrategy is TOKE, which is transferred to MYT after invoking MYTStrategy.sol::claimRewards(). However, the MYT Morpho V2 Vault is designed to handle only one asset type (WETH). As a result, the TOKE tokens cannot be processed and become stuck in the MYT contract.

## Vulnerability Details

TokeAutoEth.sol::\_claimRewards():

```solidity
    function _claimRewards() internal override returns (uint256 rewardsClaimed) {
        rewardsClaimed = rewarder.earned(address(this));
        rewarder.getReward(address(this), address(MYT), false); <@
    }
```

From rewarder contract : <https://etherscan.io/address/0x60882D6f70857606Cdd37729ccCe882015d1755E#readContract> We can find the rewardToken TOKE : <https://etherscan.io/address/0x2e9d63788249371f1DFC918a52f8d799F4a38C94>

## Impact Details

TOKE token become stuck in MYT

## References

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import "forge-std/console2.sol";
import {IMainRewarder, IAutopilotRouter} from "../strategies/interfaces/ITokemac.sol";
import {IERC4626} from "../../lib/openzeppelin-contracts/contracts/interfaces/IERC4626.sol";
interface IERC4626Like is IERC4626 {
    function balanceOfActual(address account) external view returns (uint256);
}
interface IERC20 {
    function balanceOf(address) external view returns (uint256);
    function approve(address, uint256) external returns (bool);
}

interface IAavePool {
    function supply(address asset, uint256 amount, address onBehalfOf, uint16 referralCode) external;
    function withdraw(address asset, uint256 amount, address to) external returns (uint256);
}

interface IAaveAToken {
    function balanceOf(address) external view returns (uint256);
}

contract OnchainTest is Test {
    function setUp() public {
        //fork arbitrum aave usdc.
        //ARBITRUM_MAINNET_RPC_URL = "https://arb-mainnet.g.alchemy.com/v2/ep9fbX4C-PZXTbM_E9NsfLMT9SVD9Ivb";
        //start block = 390774695
        //end block = 390794695
        vm.createSelectFork("https://eth-mainnet.g.alchemy.com/v2/ep9fbX4C-PZXTbM_E9NsfLMT9SVD9Ivb", 23600616);
    }

    function test_POC_5 () public {
        address router = 0x39ff6d21204B919441d17bef61D19181870835A2;
        address weth = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
        address alice = address(0x1001);
        address autoETH = 0x0A2b94F6871c1D7A32Fe58E1ab5e6deA2f114E56;
        address rewarder = 0x60882D6f70857606Cdd37729ccCe882015d1755E;
        address TOKE = 0x2e9d63788249371f1DFC918a52f8d799F4a38C94; // <@ TOKE address
        //transfer 1e18 ETH to alice.
        deal(weth,alice, 1e18);
        assertEq(IERC20(weth).balanceOf(alice), 1e18);

        vm.startPrank(alice);
        IERC20(weth).approve(router, type(uint256).max);
        uint256 shares = IAutopilotRouter(router).depositMax(IERC4626Like(autoETH), address(alice), 0); // minOut.

        IERC20(autoETH).approve(rewarder, shares);

        //stake.
        IMainRewarder(rewarder).stake(address(alice), shares);

        vm.roll(23610616);
        vm.warp(block.timestamp + 30 days);

        console2.log("earned:", IMainRewarder(rewarder).earned(address(alice)));

        console2.log("init:",IERC20(TOKE).balanceOf(alice));
        IMainRewarder(rewarder).getReward(alice, alice, false);
        console2.log("after claim reward:",IERC20(TOKE).balanceOf(alice));
    }

```

Out:

```shell
[PASS] test_POC_5() (gas: 597992)
Logs:
  earned: 361730934590483389
  init: 0
  after claim reward: 361730934590483389

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 605.71ms (5.39ms CPU time)

Ran 1 test suite in 610.14ms (605.71ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
