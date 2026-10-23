# 28630 - \[SC - Insight] Improper Validation for Partial Filling of INCH...

Submitted on Feb 22nd 2024 at 18:40:15 UTC by @offside0011 for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28630

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x7276925e42f9c4054afa2fad80fa79520c453d6a>

Impacts:

* Permanent freezing of funds

## Description

This is a re-submission of #28620 <https://bugs.immunefi.com/dashboard/submission/28620>

## Brief/Intro

In PufferDepositor.sol, users have the capability to swap tokens to stETH through the 1inch protocol and deposit them into Puffer using the swapAndDeposit1Inch(...) function. However, since 1inch supports partial filling, any tokens that remain unswapped will be locked within the PufferDepositor contract.

## Vulnerability Details

1. The user utilizes the swapAndDeposit1Inch(...) function with partial filling enabled through 1inch, swapping token X for stETH and depositing it.
2. As a result of fluctuations in token prices or potential front-running by attackers during this swap, only a portion of token X is exchanged for stETH, with the remainder being returned to the PufferDepositor.
3. The returned token X remains locked within the PufferDepositor indefinitely.

## Impact Details

The user will lose a portion of their funds.

## References

```
function swapAndDeposit1Inch(address tokenIn, uint256 amountIn, bytes calldata callData)
        public
        payable
        virtual
        restricted
        returns (uint256 pufETHAmount)
    {
        if (tokenIn != _NATIVE_ETH) {
            SafeERC20.safeTransferFrom(IERC20(tokenIn), msg.sender, address(this), amountIn);
            SafeERC20.safeIncreaseAllowance(IERC20(tokenIn), address(_1INCH_ROUTER), amountIn);
        }

        // PUFFER_VAULT.deposit will revert if we get no stETH from this contract
        (bool success, bytes memory returnData) = _1INCH_ROUTER.call{ value: msg.value }(callData);
        if (!success) {
            revert SwapFailed(address(tokenIn), amountIn);
        }

        uint256 amountOut = abi.decode(returnData, (uint256));

        if (amountOut == 0) {
            revert SwapFailed(address(tokenIn), amountIn);
        }

        return PUFFER_VAULT.deposit(amountOut, msg.sender);
    }
```

## Proof of Concept

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.23;

import {Test, console2} from "forge-std/Test.sol";

interface IPufferDepositor {
    function swapAndDeposit1Inch(address tokenIn, uint256 amountIn, bytes calldata callData)
        external
        payable
        returns (uint256 pufETHAmount);
}

interface IRouter {
    function arbitraryStaticCall(address target, bytes calldata data) external view returns (uint256);
}

contract Mock1Inch {
    function aa(uint256 x) public payable returns(uint256) {
        //IERC20(0xae7ab96520DE3A18E5e111B5EaAb095312D7fE84).transfer(msg.sender, x / 2);
        return x / 2;
    }
}

interface Icurve {
    function exchange(int128 i, int128 j, uint256 dx, uint256 min_dy) external payable returns (uint256);
}

interface IERC20 {
    function balanceOf(address) external returns (uint256);

    function approve(address, uint256) external;

    function transfer(address, uint256) external;
}

contract ReplayVMTest is Test {

    function setUp() public {
        vm.createSelectFork("http://192.168.50.118:8645", 19282966 - 1);
    }

    function test_debug_only2() public {

        Mock1Inch mock_1inch = new Mock1Inch();
        address _ST_ETH = 0xae7ab96520DE3A18E5e111B5EaAb095312D7fE84;
        address depositor = 0x4aA799C5dfc01ee7d790e3bf1a7C2257CE1DcefF;
        address weth = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
        // address depositor = 0x7276925e42F9c4054afA2fad80fA79520C453D6A;

        // prepare
        vm.deal(address(this), 100 ether);
        Icurve(address(0xDC24316b9AE028F1497c275EB9192a3Ea0f67022)).exchange{value: 20 ether}(0, 1, 20 ether, 0);
        IERC20(_ST_ETH).transfer(address(mock_1inch), 11 ether);
        deal(weth, address(this), 100 ether);

        console2.log("weth balance of depositor %s", IERC20(weth).balanceOf(depositor));

        // user action
        bytes memory call_aa = abi.encodeWithSelector(Mock1Inch.aa.selector, 40);
        bytes memory call_router = abi.encodeWithSelector(IRouter.arbitraryStaticCall.selector, address(mock_1inch), call_aa);
        IERC20(weth).approve(address(depositor), 100 ether);
        IPufferDepositor(depositor).swapAndDeposit1Inch(weth, 1 ether, call_router);

        console2.log("weth balance of depositor %s", IERC20(weth).balanceOf(depositor));
    }
}
```
