# 29067 - \[SC - Low] Puffer Finance Missing Verification of Externa...

Submitted on Mar 6th 2024 at 12:04:59 UTC by @Norah for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #29067

Report type: Smart Contract

Report severity: Low

Target: <https://etherscan.io/address/0x3C28B7c7Ba1A1f55c9Ce66b263B33B204f2126eA#code>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`Timelock.sol` has `executeTransaction()` function which takes the target address, calldata and operation ID as arguments, and it calls the function specified in calldata along with the argument values on the target address.

## Vulnerability Details

First of all, there is no input validation on the target address as well as the call data being supplied in the argument, which is fine. But there is no check on whether the external call succeeded or not.

```
       .
       .
       .

        if (block.timestamp < lockedUntil) {
            revert Locked(txHash, lockedUntil);
        }

        queue[txHash] = 0;
        (success, returnData) = _executeTransaction(target, callData);

        emit TransactionExecuted(txHash, target, callData, operationId);

        return (success, returnData);

        .
        .
        .
      
       function _executeTransaction(address target, bytes calldata callData) internal returns (bool, bytes memory) {
        return target.call(callData);
    }

```

While, current implementation does catch boolean return by external call indication success or failure of the same.

But, there is no check placed using that boolean to verify the transaction succeeded or not.

## Impact Details

In cases, where wrong inputs (like non-existent address or incorrect signatures) are provided to `executeTransaction` i, the external call will silently fail.

## Recommendation

```
        .
        .
        .

        if (block.timestamp < lockedUntil) {
            revert Locked(txHash, lockedUntil);
        }

        queue[txHash] = 0;
        (success, returnData) = _executeTransaction(target, callData);

+++     require(success, "Transaction failed");

        emit TransactionExecuted(txHash, target, callData, operationId);

        return (success, returnData);

      }

```

## References

Add any relevant links to documentation or code

## Proof of Concept

* Add below tests in the timelock.t.sol file in the current test suite :
* <https://github.com/PufferFinance/pufETH/blob/main/test/unit/Timelock.t.sol>

```
   function testCalltoNonExistentAddress() public {

        vm.startPrank(timelock.OPERATIONS_MULTISIG());

        bytes memory callData = abi.encodeCall(Timelock.setDelay, (15 days));

        uint256 operationId = 1234;

        uint snap = vm.snapshot();

        //Non-existent is provided instead of timelock address
        address nonExistentAddress = address(0xDead);
        bytes32 codehash;
        assembly { 
            codehash := extcodehash(nonExistentAddress) 
        }
        assert(codehash == 0x0);
        bytes32 txHash = timelock.queueTransaction(nonExistentAddress, callData, operationId);

        uint256 lockedUntil = block.timestamp + timelock.delay();

        vm.warp(lockedUntil + 1);

        //This should have reverted.
        timelock.executeTransaction(nonExistentAddress, callData, operationId);

        vm.revertTo(snap);

    }

    function testNonExistentMethod() public {

        vm.startPrank(timelock.OPERATIONS_MULTISIG());

        //signature of non existent method
        bytes memory callData = bytes('0xaaaaaaaa');

        uint256 operationId = 1234;

        bytes32 txHash = timelock.queueTransaction(address(timelock), callData, operationId);

        uint256 lockedUntil = block.timestamp + timelock.delay();

        vm.warp(lockedUntil + 1);

        //This will succeed without reverting
        timelock.executeTransaction(address(timelock), callData, operationId);

    }

```
