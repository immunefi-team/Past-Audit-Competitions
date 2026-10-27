# #38257 \[SC-Insight] Freezing of msg.value passed in Bridge.deposit() if adapter is address zero

**Submitted on Dec 29th 2024 at 09:32:35 UTC by @Shahen for** [**Audit Comp | Lombard**](https://immunefi.com/audit-competition/audit-comp-lombard)

* **Report ID:** #38257
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/lombard-finance/evm-smart-contracts/blob/main/contracts/bridge/Bridge.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 30 days

## Description

## Brief/Intro

When interacted with the contract by calling `Bridge.deposit()` with a certain `msg.value`, usually what happens is any excess eth > fee amount will be appended to `refunds[fromAddress]` by the adapter therefore the msg.sender can withdraw the refund later, But if adapter is == address(0) any passed eth through the `Bridge.deposit()` function will be stuck in the Bridge contract without a refund.

Please refer to the below coded foundry poc, Run `forge test -vvv` , Install the foundry plugin for hardhat.

(Note - Also comment out `_disableInitializers()` inside the constructors of both LBTC and Bridge contracts to run this test.)

## Vulnerability Details

Same as above Brief/Intro

## Impact Details

Freezing of msg.value passed in Bridge.deposit() if adapter is address zero

## References

<https://github.com/lombard-finance/evm-smart-contracts/blob/a818ea0489178ccd00019edab24637c38501af7b/contracts/bridge/Bridge.sol#L147>

## Proof of Concept

## Proof of Concept

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import "../lib/forge-std/src/Test.sol";
import {LBTC} from "../contracts/LBTC/LBTC.sol";
import {Bridge} from "../contracts/bridge/Bridge.sol";
import {ConsortiumMock} from "../contracts/mock/ConsortiumMock.sol";
import {ILBTC} from "../contracts/Bridge/IBridge.sol";
import {IAdapter} from "../contracts/bridge/adapters/IAdapter.sol";
import {RateLimits} from "../contracts/libs/RateLimits.sol";




contract lombardtest is Test {

    LBTC public _LBTC;
    Bridge public _bridge;
    ConsortiumMock public _ConsortiumMock;
    
    address bob = address(0x7);
    address alex = address(0x8);

    function setUp() public {

       
       _LBTC = new LBTC(); // comment out `_disableInitializers()` inside the constructor please..
       _bridge = new Bridge(); // comment out `_disableInitializers()` inside the constructor please..
       _ConsortiumMock = new ConsortiumMock();

       _LBTC.initialize(address(_ConsortiumMock),uint64(0),address(0x3),address(this));
       _bridge.initialize(ILBTC(address(_LBTC)),address(0x1),address(this));
       _bridge.addDestination(bytes32(uint256(1)),bytes32(uint256(uint160(address(0x4)))),uint16(1000),uint64(0),IAdapter(address(0)),true);
       RateLimits.Config[] memory _config = new RateLimits.Config[](1);
       _config[0] = RateLimits.Config({chainId: bytes32(uint256(1)),limit: uint256(5e8) ,window: uint256(1000000000)});
       _bridge.setRateLimits(_config,_config);
       _LBTC.addMinter(address(_bridge));
       deal(bob,0.01 ether);
       assertEq(bob.balance,0.01 ether);
       deal(address(_LBTC),bob,2e8);
       assertEq(_LBTC.balanceOf(bob),2e8);       
    }

    function test_msgValue_lost() public {

     //Test case proving that if config.adapter == address(0), Any msg.value passed is not returned back to the caller but stuck in contract itself.   
       vm.startPrank(bob);
       _LBTC.approve(address(_bridge), 2e8); // just using a higher value,ignore it
       _bridge.deposit{value: 0.01 ether}(bytes32(uint256(1)), bytes32(uint256(uint160(alex))), 1e8);
       assertEq(address(_bridge).balance,0.01 ether); // Asserting that the bridge contract balance is of the passed ether value.
       vm.stopPrank();
       


       


    }

}

```
