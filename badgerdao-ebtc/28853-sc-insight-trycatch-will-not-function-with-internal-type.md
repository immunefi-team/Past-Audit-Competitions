# 28853 - \[SC - Insight] Trycatch will not function with internal type

Submitted on Feb 28th 2024 at 22:08:47 UTC by @thomastech for [Boost | eBTC](https://immunefi.com/bounty/ebtc-boost/)

Report ID: #28853

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/ebtc-protocol/ebtc/blob/release-0.7/packages/contracts/contracts/PriceFeed.sol>

Impacts:

* Direct theft of 2 stETH worth of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Solidity v0.8.24 "Solidity also supports exception handling in the form of try/catch-statements, but only for external function calls and contract creation calls. Errors can be created using the revert statement." PriceFeed.sol uses internal with three function calls that are used to update ChainLink Oracle values.

## Vulnerability Details

## Impact Details

An issue with Solidity Control Structure try/catch using internal function type in:

1. PriceFeed.sol function \_getPrevChainlinkResponse lines 688 - 770
2. PriceFeed.sol function \_getCurrentChainlinkResponse lines 607 - 682
3. PriceFeed.sol function \_getCurrentFallbackResponse lines 583 - 603

These functions will not update multiple ChainLink variables.

## References

Add any relevant links to documentation or code:

<https://github.com/ebtc-protocol/ebtc/blob/main/packages/contracts/contracts/PriceFeed.sol>

## Proof of concept

Given my limited skills with Solidity/Foundry I have been unable to successfully test these functions in the time allowed but have deployed the code below on Remix and changed the function from public, external, and internal to confirm try/catch fails when internal is used.

//SPDX-License-Identifier: MIT pragma solidity 0.8.24;

contract WillThrow { function aFunction() public pure { require(true, "Try succeed message require is false");

```
}
```

}

contract ErrorHandling { event ErrorLogging(string reason); event TryError(string message); function catchError() public { WillThrow will = new WillThrow(); try will.aFunction() { //here we could do something if it works emit TryError('Try Works require is true'); } catch Error(string memory reason) { emit ErrorLogging(reason); } } }
