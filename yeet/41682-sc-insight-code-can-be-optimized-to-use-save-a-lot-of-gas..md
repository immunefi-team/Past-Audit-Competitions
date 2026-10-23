# #41682 \[SC-Insight] Code can be optimized to use save a lot of gas.

**Submitted on Mar 17th 2025 at 14:39:34 UTC by @k1k1 for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41682
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Reward.sol>
* **Impacts:**

## Description

## Brief/Intro

The code uses `YeetGameSettings` contract for settings but the way the `YeetGameSettings` is defined is unoptimized , as the game settings address doesn't get re-initialized it must have been declared as `immutable` , as it's not optimized it consumes extra gas as a state variable would.

## Vulnerability Details

look at [line](https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/Yeet.sol#L117)

```solidity
YeetGameSettings public gameSettings;
```

and here [also](https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/Yeet.sol#L534-L545)

```solidiity
    function copySettings() internal {
        YEET_TIME_SECONDS = gameSettings.YEET_TIME_SECONDS();
        POT_DIVISION = gameSettings.POT_DIVISION();
        TAX_PER_YEET = gameSettings.TAX_PER_YEET();
        TAX_TO_STAKERS = gameSettings.TAX_TO_STAKERS();
        TAX_TO_PUBLIC_GOODS = gameSettings.TAX_TO_PUBLIC_GOODS();
        TAX_TO_TREASURY = gameSettings.TAX_TO_TREASURY();
        YEETBACK_PERCENTAGE = gameSettings.YEETBACK_PERCENTAGE();
        COOLDOWN_TIME = gameSettings.COOLDOWN_TIME();
        BOOSTRAP_PHASE_DURATION = gameSettings.BOOSTRAP_PHASE_DURATION();
        MINIMUM_YEET_POINT = gameSettings.MINIMUM_YEET_POINT();
    }
```

now when ever `restart` function gets called this `copySettings` function gets triggered , which causes around `25841` more gas consumption for the transaction.

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

import {YeetGameSettings} from "../../src/YeetGameSettings.sol";

  

contract copy_settings is Test {

YeetGameSettings _game_settings;

A a;

B b;

  

uint256 gas_consumed_1;

uint256 gas_consumed_2;

uint256 net_gas_consumed;

  

function setUp() public {

_game_settings = new YeetGameSettings();

a = new A(_game_settings);

b = new B(_game_settings);

}

  

function _getting_non_immutable_value() public {

gas_consumed_1 = a._copy_settings();

  

console.log(

"Gas consumed for non-immutable variable: ",

gas_consumed_1

);

}

  

function _getting_immutable_value() public {

gas_consumed_2 = b._copy_settings();

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

YeetGameSettings public gameSettings;

  

// Settings from YeetGameSettings

/// @notice see `YeetGameSettings` for more information

uint256 public YEET_TIME_SECONDS; // The time between yeets

uint256 public POT_DIVISION; // 1/xth of the pot eg, x%, 1/200 = 0.5%

uint256 public TAX_PER_YEET; // The tax per yeet in %

uint256 public TAX_TO_STAKERS; // The tax to stakers in %

uint256 public TAX_TO_PUBLIC_GOODS; // The tax to public goods in %

uint256 public TAX_TO_TREASURY; // The tax to treasury in %

uint256 public YEETBACK_PERCENTAGE; // The percentage of the pot that goes to yeetback

uint256 public COOLDOWN_TIME; // The time between rounds

uint256 public BOOSTRAP_PHASE_DURATION; // The time between rounds

uint256 public MINIMUM_YEET_POINT; // The minimum amount needed to yeet

  

constructor(YeetGameSettings _game_settings) {

gameSettings = _game_settings;

}

  

function copySettings() internal {

YEET_TIME_SECONDS = gameSettings.YEET_TIME_SECONDS();

POT_DIVISION = gameSettings.POT_DIVISION();

TAX_PER_YEET = gameSettings.TAX_PER_YEET();

TAX_TO_STAKERS = gameSettings.TAX_TO_STAKERS();

TAX_TO_PUBLIC_GOODS = gameSettings.TAX_TO_PUBLIC_GOODS();

TAX_TO_TREASURY = gameSettings.TAX_TO_TREASURY();

YEETBACK_PERCENTAGE = gameSettings.YEETBACK_PERCENTAGE();

COOLDOWN_TIME = gameSettings.COOLDOWN_TIME();

BOOSTRAP_PHASE_DURATION = gameSettings.BOOSTRAP_PHASE_DURATION();

MINIMUM_YEET_POINT = gameSettings.MINIMUM_YEET_POINT();

}

  

function _copy_settings() public returns (uint256 gas_consumed) {

uint256 gas_in;

uint256 gas_out;

gas_in = gasleft();

copySettings();

gas_out = gasleft();

gas_consumed = gas_in - gas_out;

}

}

  

contract B {

YeetGameSettings public immutable gameSettings;

  

// Settings from YeetGameSettings

/// @notice see `YeetGameSettings` for more information

uint256 public YEET_TIME_SECONDS; // The time between yeets

uint256 public POT_DIVISION; // 1/xth of the pot eg, x%, 1/200 = 0.5%

uint256 public TAX_PER_YEET; // The tax per yeet in %

uint256 public TAX_TO_STAKERS; // The tax to stakers in %

uint256 public TAX_TO_PUBLIC_GOODS; // The tax to public goods in %

uint256 public TAX_TO_TREASURY; // The tax to treasury in %

uint256 public YEETBACK_PERCENTAGE; // The percentage of the pot that goes to yeetback

uint256 public COOLDOWN_TIME; // The time between rounds

uint256 public BOOSTRAP_PHASE_DURATION; // The time between rounds

uint256 public MINIMUM_YEET_POINT; // The minimum amount needed to yeet

  

constructor(YeetGameSettings _game_settings) {

gameSettings = _game_settings;

}

  

function copySettings() internal {

YEET_TIME_SECONDS = gameSettings.YEET_TIME_SECONDS();

POT_DIVISION = gameSettings.POT_DIVISION();

TAX_PER_YEET = gameSettings.TAX_PER_YEET();

TAX_TO_STAKERS = gameSettings.TAX_TO_STAKERS();

TAX_TO_PUBLIC_GOODS = gameSettings.TAX_TO_PUBLIC_GOODS();

TAX_TO_TREASURY = gameSettings.TAX_TO_TREASURY();

YEETBACK_PERCENTAGE = gameSettings.YEETBACK_PERCENTAGE();

COOLDOWN_TIME = gameSettings.COOLDOWN_TIME();

BOOSTRAP_PHASE_DURATION = gameSettings.BOOSTRAP_PHASE_DURATION();

MINIMUM_YEET_POINT = gameSettings.MINIMUM_YEET_POINT();

}

  

function _copy_settings() public returns (uint256 gas_consumed) {

uint256 gas_in;

uint256 gas_out;

gas_in = gasleft();

copySettings();

gas_out = gasleft();

gas_consumed = gas_in - gas_out;

}

}
```
