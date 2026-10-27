# #41688 \[SC-Insight] Code can be optimized to to save a significant amount of gas.

**Submitted on Mar 17th 2025 at 15:22:03 UTC by @p3nc1l for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41688
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Reward.sol>
* **Impacts:**

## Description

## Brief/Intro

The `Reward` contract uses `IERC20` type to distribute `ERC20` token as rewards , but the way `IERC20` is defined is unoptimized , as the `IERC20` type doesn't get get re-initialized, it must have been declared as `immutable`.

## Vulnerability Details

look at [line](https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/Reward.sol#L49)

```solidity
    IERC20 public token;
```

now when ever `claim()` function [from](https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/Reward.sol#L132-L140) gets called this `token` variable triggered twice , which causes around `2*2100=4200` more gas consumption for the transaction.

```solidity
    function claim() external {
        uint256 amountEarned = getClaimableAmount(msg.sender);
        require(amountEarned != 0, "Nothing to claim");
        require(token.balanceOf(address(this)) >= amountEarned, "Not enough tokens in contract");


        lastClaimedForEpoch[msg.sender] = currentEpoch - 1; // This should be the fix.
        token.transfer(msg.sender, amountEarned);
        emit Rewarded(msg.sender, amountEarned, block.timestamp);
    }
```

## Impact Details

The caller has to pay more gas for the same operation which could be done using less gas.

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.28;
import {Test, console} from "forge-std/Test.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract get_token_address is Test {
    A a;
    B b;
    IERC20 token_a = IERC20(makeAddr("Token_a"));
    IERC20 token_b = IERC20(makeAddr("Token_b"));

    uint256 gas_consumed_1;
    uint256 gas_consumed_2;
    uint256 net_gas_consumed;

    function setUp() public {
        a = new A(token_a);
        b = new B(token_b);
    }

    function _getting_non_immutable_value() public {
        (, gas_consumed_1) = a.get_token();

        console.log(
            "Gas consumed for non-immutable variable: ",
            gas_consumed_1
        );
    }

    function _getting_immutable_value() public {
        (, gas_consumed_2) = b.get_token();
        console.log("Gas consumed for immutable variable: ", gas_consumed_2);
    }

    function test_net_gas_consumed() public {
        _getting_non_immutable_value();
        _getting_immutable_value();
        net_gas_consumed = gas_consumed_1 - gas_consumed_2;
        console.log("Net gas consumed: %s", net_gas_consumed);
    }
}

contract A {
    IERC20 public _token;
    uint256 gas_in;
    uint256 gas_out;
    uint256 gas_consumed;

    constructor(IERC20 token) {
        _token = token;
    }

    function get_token() public returns (IERC20 tkn, uint256 gas_consm) {
        gas_in = gasleft();
        tkn = _token;
        gas_out = gasleft();
        gas_consm = gas_in - gas_out;
    }
}

contract B {
    IERC20 public immutable _token;
    uint256 gas_in;
    uint256 gas_out;
    uint256 gas_consumed;

    constructor(IERC20 token) {
        _token = token;
    }

    function get_token() public returns (IERC20 tkn, uint256 gas_consm) {
        gas_in = gasleft();
        tkn = _token;
        gas_out = gasleft();
        gas_consm = gas_in - gas_out;
    }
}


```
