# 56751 sc medium stargateethpoolstrategy deallocate function redeem less weth than expected

**Submitted on Oct 20th 2025 at 11:44:00 UTC by @ox9527 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56751
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/StargateEthPoolStrategy.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

StargateEthPoolStrategy deallocate function redeem weth using native token , however the amount of weth is less than expected , which can lead to `Strategy balance is less than the amount needed` revert.

## Vulnerability Details

From the StargateEthPoolStrategy.sol::\_deallocate()

```solidity
        lp.approve(address(pool), lpNeeded);
        uint256 ethBalanceBefore = address(this).balance;
        pool.redeem(lpNeeded, address(this));
        uint256 ethBalanceAfter = address(this).balance;
        uint256 ethRedeemed = ethBalanceAfter - ethBalanceBefore;
        if (ethRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss which includes rounding loss.", amount, ethRedeemed);
        }
        if (ethRedeemed + ethBalanceBefore >= amount) { //@audit ?
            weth.deposit{value: ethRedeemed}();
        }
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
```

The value `ethRedeemed` is used to redeem WETH. However, the `pool.redeem()` function returns a smaller amount of native tokens than the input value due to rounding down.

## Impact Details

1.the \_deallocate function revert lead to assets become stuck in the contract

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


interface IStargatePool {
    function deposit(address receiver, uint256 amountLD) external payable returns (uint256 amountLDOut);
    function redeem(uint256 lpAmount, address receiver) external returns (uint256 amountLDOut);
    function redeemable(address owner) external view returns (uint256 amountLD); // underlying denom
    function lpToken() external view returns (address);
    function tvl() external view returns (uint256);
}

interface IWETH {
    function deposit() external payable;
    function withdraw(uint256) external;
}

interface IERC20Minimal {
    function totalSupply() external view returns (uint256);
    function balanceOf(address) external view returns (uint256);
    function approve(address, uint256) external returns (bool);
}


contract OnchainTest is Test {
    function setUp() public {
        //fork arbitrum aave usdc.
        //ARBITRUM_MAINNET_RPC_URL = "https://arb-mainnet.g.alchemy.com/v2/ep9fbX4C-PZXTbM_E9NsfLMT9SVD9Ivb";
        //start block = 390774695
        //end block = 390794695
        vm.createSelectFork("https://opt-mainnet.g.alchemy.com/v2/ep9fbX4C-PZXTbM_E9NsfLMT9SVD9Ivb", 142673254);
        // vm.createSelectFork("https://arb-mainnet.g.alchemy.com/v2/ep9fbX4C-PZXTbM_E9NsfLMT9SVD9Ivb", 390794695);
        // vm.createSelectFork("https://eth-mainnet.g.alchemy.com/v2/ep9fbX4C-PZXTbM_E9NsfLMT9SVD9Ivb", 23600616);
    }


    receive() external payable {}
    function test_POC_10() public {
        address weth = 0x4200000000000000000000000000000000000006;
        IStargatePool pool = IStargatePool(0xe8CDF27AcD73a434D661C84887215F7598e7d0d3);
        IERC20Minimal lp = IERC20Minimal(IStargatePool(pool).lpToken());
        address stargatePool = address(0x1001);

        uint256 allocateAmount = 10888888888888888888;
        deal(weth,stargatePool,allocateAmount);
        
        vm.startPrank(stargatePool);

        //allocate start.
        IWETH(weth).withdraw(allocateAmount);
        uint256 depositAmount = (allocateAmount / 1e12) * 1e12;
        pool.deposit{value: depositAmount}(stargatePool, depositAmount);
        //allocate end.

        uint256 deallocateAmount = 2222222222123456789;

        //deallocate start.
        lp.approve(address(pool), deallocateAmount);

        uint256 ethBalanceBefore = stargatePool.balance;
        pool.redeem(deallocateAmount, stargatePool);
        uint256 ethBalanceAfter = stargatePool.balance;
        uint256 ethRedeemed = ethBalanceAfter - ethBalanceBefore;

        assert(IERC20(weth).balanceOf(stargatePool) == 0 );
        assert(ethRedeemed + ethBalanceBefore > deallocateAmount);

        IWETH(weth).deposit{value: ethRedeemed}();
        //deallocate end.

        console2.log("deallocateAmount:",deallocateAmount);
        console2.log("weth redeemed:",IERC20(weth).balanceOf(stargatePool));

        assert(IERC20(weth).balanceOf(stargatePool) < deallocateAmount);
    }
```

out:

```shell
[PASS] test_POC_10() (gas: 380096)
Logs:
  deallocateAmount: 2222222222123456789
  weth redeemed: 2222222000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.07s (2.92ms CPU time)

Ran 1 test suite in 1.07s (1.07s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
