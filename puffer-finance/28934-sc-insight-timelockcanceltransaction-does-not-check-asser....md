# 28934 - \[SC - Insight] TimelockcancelTransaction does not check asser...

Submitted on Mar 2nd 2024 at 05:15:26 UTC by @Obin for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28934

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x3C28B7c7Ba1A1f55c9Ce66b263B33B204f2126eA#code>

Impacts:

* Contract accepts unqueued Tx as input leading to multiple unexpected behavior & Impacts

## Description

## Brief/Intro

The cancelTransaction() in Timelock .sol does not check that input is a valid queued Tx. This leads to multiple undesired scenarios / impacts. They include:

## Impact Details

In a situation of input error, contract will:

1. Not actually delete Tx as intended.
2. Contract can wrongly emit / publish wrong data to have been cancelled. Hence deceiving the public.
3. A reversed Decision (cancelled TX) may still go through undesirously.

## POC

See below.

## Mitigation

```diff
function cancelTransaction(address target, bytes memory callData, uint256 operationId) public {
        // Community multisig can call this by via executeTransaction
        if (msg.sender != OPERATIONS_MULTISIG && msg.sender != address(this)) {
            revert Unauthorized();
        }

        bytes32 txHash = keccak256(abi.encode(target, callData, operationId));
+       uint cacheUint = queue[txHash];
+       require (cacheUint > 0, "Timelock: unqueued Tx");
        queue[txHash] = 0;

        emit TransactionCanceled(txHash, target, callData, operationId);
    }
```

## Proof of Concept

```javascript
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

contract Bomb {
    bool bombed;

    function detonateBomb() external {
        bombed = true;
    }
    function getBombed() external view returns(bool){
        return bombed;
    }
}

contract TimelockTest is Test { 
    string internal mainnetRpc = "XYZ";
    address public timeLockAddr = 0x3C28B7c7Ba1A1f55c9Ce66b263B33B204f2126eA;
    uint blockId;
    Timelock timelock;
    Bomb bombContract; // Sample Bomb contract intended to be detonated by OPERATIONS_MULTISIG via queue and execute

    function setUp() public {
        blockId = vm.createSelectFork(mainnetRpc);
        timelock = Timelock(timeLockAddr);
        bombContract = new Bomb();
    }

    function testFork() public {
        //assert at initiation Bomb not detonated
        bool blastedA = bombContract.getBombed();
        assertEq(blastedA, false);
        
        //address Msig
        address OPERATIONS_MULTISIG = timelock.OPERATIONS_MULTISIG();

        //make Tx for queue (intends to call detonateBomb())
        address bombContractAddr = address(bombContract);
        uint operationId = 444;
        bytes memory KallData = abi.encodeCall(Bomb.detonateBomb, ());
        
        //Queue Tx
        vm.startPrank(OPERATIONS_MULTISIG);
        bytes32 CorrectTxHash = timelock.queueTransaction(bombContractAddr,KallData,operationId);

        //Cancels Tx wrongly (error prone, say input similar operationId)
        uint errorInputedOperationId = 4444; // different from 444
        
        bytes32 wrongHash = keccak256(abi.encode(bombContractAddr, KallData, errorInputedOperationId));
        //Assert that wrongHash isn't the correct Txhash
        assertTrue(wrongHash != CorrectTxHash );

        //====Bug=Impact===Contract actually emits cacellation of a false TxHash

        vm.expectEmit();
        emit Timelock.TransactionCanceled(wrongHash,bombContractAddr,KallData,errorInputedOperationId);
        timelock.cancelTransaction(bombContractAddr,KallData,/*The error*/errorInputedOperationId);

        //====Bug=Impact===Intended cancelled Tx still valid. Assert TxHash is still queued && Valid

        uint LockedUntil = timelock.queue(CorrectTxHash);
        assertTrue(LockedUntil != 0);

        //====Bug=Impact===Cancelled Tx / DEcision can still go throuh and detonate bomb 
        
        vm.warp(block.timestamp + timelock.delay());
        (bool succes,) = timelock.executeTransaction(bombContractAddr,KallData,operationId);
        bool blastedB = bombContract.getBombed();
        assertEq(blastedB, true);
        assertTrue(succes);
        
    }

}

```
