# 29033 - \[SC - High] Queued data will be lost if Tx is unsuccessful ...

Submitted on Mar 5th 2024 at 06:07:31 UTC by @Obin for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #29033

Report type: Smart Contract

Report severity: High

Target: <https://etherscan.io/address/0x3C28B7c7Ba1A1f55c9Ce66b263B33B204f2126eA#code>

Impacts:

* Queued data will be lost if Tx is unsuccessful or unaffected as intended

## Description

## Brief/Intro

Timelock::executeTransaction() implementation is too optimistic given that it doesnt handle situations of:

1. Unsuccessful Call
2. Successful call but unaffected operations / returnValue (false) In these scenarios the queued Tx data is lost and may require requeue and undesirable delay.

## Vulnerability Details

Depth 1: In a situation where Timelock::executeTransaction() is unsuccessful, the transaction does not revert. instead it overwrites the queued tx. This is obviously an undesirable bug in implementation as OPERATOR at best will need to re-queue and be delayed.

NOTE: If this is the desired implementation, then contract is wrong to not emit call status as an argument as the emission makes the public wrongly assume it was a successful Tx. I doubt it is though.

First, its should be noted that the depth in impact of this bug does not end in appending `require(success, "Timelock: TX failed")` as data can still be lost even at that. Hence I'll explain the 2 depths and of course deeper Impact illustrated in POC.

Depth 2: From a hacker perspective, the bool return value in a call is just a dummy and does not guarantee the desired effect on target contract. Lets not be fooled by that. The problem here is akin to what Openzeppelin tries to solve by implementing SafeERC20 which provides a logic that unites various ERC20 of arbitrary addresses and return signatures. A Tx or call can be successful (return the dummy true value) yet intended operation remains undesirable / unaffected at target contract. In this situation, will return `bool success == true;` and queued data `queue[txHash]` will still be lost.

This scenario is actually very realistic and possible given that these calls are to arbitrary addresses whose implementations and return signatures are of arbitrary patterns.

## Impact Details

Contract will lose queued data permanently. Forcing operator to requeue and delay before transacting. Given that this is of high chance occurrence, its a High severity. Note that the re-queued Tx will still be susceptible.

## References

Add any relevant links to documentation or code

## Proof of Concept

```
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.8.0 <0.9.0;

import { Test } from "forge-std/Test.sol";
import { console2 } from "forge-std/console2.sol";

interface Timelock {
    event DelayChanged(uint256 oldDelay, uint256 newDelay);
    event PauserChanged(address oldPauser, address newPauser);
    event TransactionCanceled(
        bytes32 indexed txHash, address indexed target, bytes callData, uint256 indexed operationId
    );
    event TransactionExecuted(
        bytes32 indexed txHash, address indexed target, bytes callData, uint256 indexed operationId
    );
    event TransactionQueued(
        bytes32 indexed txHash, address indexed target, bytes callData, uint256 indexed operationId, uint256 lockedUntil
    );

    function ACCESS_MANAGER() external view returns (address);
    function COMMUNITY_MULTISIG() external view returns (address);
    function MINIMUM_DELAY() external view returns (uint256);
    function OPERATIONS_MULTISIG() external view returns (address);
    function cancelTransaction(address target, bytes memory callData, uint256 operationId) external;
    function delay() external view returns (uint256);
    function executeTransaction(address target, bytes memory callData, uint256 operationId)
        external
        returns (bool success, bytes memory returnData);
    function pause(address[] memory targets) external;
    function pauserMultisig() external view returns (address);
    function queue(bytes32 transactionHash) external view returns (uint256 lockedUntil);
    function queueTransaction(address target, bytes memory callData, uint256 operationId) external returns (bytes32);
    function setDelay(uint256 newDelay) external;
    function setPauser(address newPauser) external;
}

contract ClubRegistry {
    mapping (address => bool) members;

    function joinClub( address _newMember) external returns(bool joined) {
        uint timeStamp = block.timestamp;

        if (timeStamp % 3 == 0) {
            revert();
        }
        else if (timeStamp % 2 == 0) {
            // The desired effet seeked by timelock::OPERATIONS_MULTISIG
            members[_newMember] = true;
        }
        
        else {
            if (false) { //cannot be reached (for test purposes)
                members[_newMember] = true;
            }
        }
        joined = getstatus(_newMember);
    }

    function getstatus(address _querry) public view returns(bool){
        return members[_querry];
    }

    // function doo() public pure {
    //     revert();
    // }
}

contract TimelockTest is Test { 
    string internal mainnetRpc = "https://eth-mainnet.g.alchemy.com/v2/CZIEYE1tyZTnLXtG8e6wh2YboDUPJ43f";
    address public timeLockAddr = 0x3C28B7c7Ba1A1f55c9Ce66b263B33B204f2126eA;
    address OPERATIONS_MULTISIG;
    uint blockId;
    Timelock timelock;
    ClubRegistry joinClubContract; // Sample joinClubContract. OPERATIONS_MULTISIG intends to register a member.

    function setUp() public {
        blockId = vm.createSelectFork(mainnetRpc);
        timelock = Timelock(timeLockAddr);
        joinClubContract = new ClubRegistry();
        OPERATIONS_MULTISIG = timelock.OPERATIONS_MULTISIG();
    }

    function test_data_loss_on_unsuccessful_transaction() public {
        address joinee = makeAddr("joinee");
    
        //make Tx for ClubRegistry::do() (which will revert)
        address joinClubContractAddr = address(joinClubContract);
        uint operationId = 444;
        bytes memory KallData = abi.encodeCall(ClubRegistry.joinClub, (joinee));
        
        //Queue Tx
        vm.startPrank(OPERATIONS_MULTISIG);
        bytes32 TxHash = timelock.queueTransaction(joinClubContractAddr,KallData,operationId);

        uint LockedUntil = timelock.queue(TxHash);
        assertTrue(LockedUntil != 0); // Tx queued correctly.

        vm.warp(find_Revert_Time());// ensures its a failed Tx
        (bool succes,) = timelock.executeTransaction(joinClubContractAddr,KallData,operationId);
        console2.log("succes",succes, block.timestamp);
        assertTrue(succes == false); // ensures its a failed Tx

        //check loss od data
        uint newLockset = timelock.queue(TxHash);

        assertEq(newLockset, 0); // data Lost!!!

    }

    function test_data_lost_after_uneffected_operation_but_SuccesfulTx() public {
        //address Msig
        address joinee = makeAddr("joinee");

        //make Tx for queue (join / register a member into ClubRegistry)
        address joinClubContractAddr = address(joinClubContract);
        uint operationId = 444;
        bytes memory KallData = abi.encodeCall(ClubRegistry.joinClub, (joinee));
        
        //Queue Tx
        vm.startPrank(OPERATIONS_MULTISIG);
        bytes32 TxHash = timelock.queueTransaction(joinClubContractAddr,KallData,operationId);

        uint LockedUntil = timelock.queue(TxHash);
        assertTrue(LockedUntil != 0); // Tx queued correctly.

        vm.warp(find_False_Time());// ensures its a Succesful Tx but uneffected operation
        (bool succes,) = timelock.executeTransaction(joinClubContractAddr,KallData,operationId);

        //===Assertions======
        assertTrue(succes == true); // ensures its a successful Tx but undesirous operation (returned false)
        //Undesired effect
        assertEq(joinClubContract.getstatus(joinee), false);

        //Data lost
        //check loss od data
        uint newLockset = timelock.queue(TxHash);

        assertEq(newLockset, 0); // data Lost!!!


        
    }

    function find_Revert_Time()internal view returns(uint timeNow){
        timeNow = block.timestamp + timelock.delay();
        while ( timeNow % 3 != 0 ) {
            ++timeNow;
        }
    }

    function find_False_Time()internal view returns(uint timeNow){
        timeNow = block.timestamp + timelock.delay();
        while (timeNow % 2 == 0 || timeNow % 3 == 0) {
            ++timeNow;
        }
    }

}

```
