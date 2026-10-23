# 29006 - \[SC - Medium] Lack of Success check of the Timelock executeT...

Submitted on Mar 4th 2024 at 16:26:05 UTC by @HX000 for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #29006

Report type: Smart Contract

Report severity: Medium

Target: <https://etherscan.io/address/0x3C28B7c7Ba1A1f55c9Ce66b263B33B204f2126eA#code>

Impacts:

* It will impact the functionality of the protocol and reduces the efficiency of an important role
* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

In Timelock.sol the function executeTransaction() internally calls \_executeTransaction() and directly emites the event TransactionExecuted() without checking if the execution succeeded or not.

## Vulnerability Details

```solidity
   function executeTransaction(address target, bytes calldata callData, uint256 operationId)
        external
        returns (bool success, bytes memory returnData)
    {
        // Community Multisig can do things without any delay
        if (msg.sender == COMMUNITY_MULTISIG) {
            return _executeTransaction(target, callData);
        }

        // Operations multisig needs to queue it and then execute after a delay
        if (msg.sender != OPERATIONS_MULTISIG) {
            revert Unauthorized();
        }

        bytes32 txHash = keccak256(abi.encode(target, callData, operationId));
        uint256 lockedUntil = queue[txHash];

        // slither-disable-next-line incorrect-equality
        if (lockedUntil == 0) {
            revert InvalidTransaction(txHash);
        }

        if (block.timestamp < lockedUntil) {
            revert Locked(txHash, lockedUntil);
        }

        queue[txHash] = 0;
        (success, returnData) = _executeTransaction(target, callData);

        emit TransactionExecuted(txHash, target, callData, operationId);

        return (success, returnData);
    }
```

```solidity
 function _executeTransaction(address target, bytes calldata callData) internal returns (bool, bytes memory) {
        // slither-disable-next-line arbitrary-send-eth
        return target.call(callData);
    }
```

in the code executeTransaction() first checks to make sure the tx is valid then - to avoid any reentrancy risk - resets its spot in the queue which is the same effect for the cancelTransaction() function, And finally TransactionExecuted() event is emitted. The problem is, if the transaction failed for any reason(e.g., due to an invalid target contract address, out-of-gas in the called contract, or a revert in the called contract) , the tx will not revert. it will only return "false" which is never handled AND the side-effects still happening i.e queue\[txHash] = 0; and emit TransactionExecuted(txHash, target, callData, operationId);

## Impact Details

This bug will result in 2 impacts: 1- Distrub functionality of the operations\_multisig who can not re-execute the failed/ canceled transaction unless re-queue it and stand the delay time again. 2- The wrong-emitted event can lead to bad consequences for any dapp/smart contract depending on those events.

## Proof of Concept
