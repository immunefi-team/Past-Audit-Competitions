# 28789 - \[SC - Low] Return value of call is not checked causing fai...

Submitted on Feb 27th 2024 at 11:05:49 UTC by @MrPotatoMagic for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28789

Report type: Smart Contract

Report severity: Low

Target: <https://etherscan.io/address/0x3C28B7c7Ba1A1f55c9Ce66b263B33B204f2126eA#code>

Impacts:

* Failed transactions continue execution without reverting

## Description

## Brief/Intro

The `executeTransaction()` function allows operators (with a delay) to execute calldata on a target contract.

The issue is that if the operation multisig executes some calldata on a target contract and the external call fails, the return value of `success` is not checked to be true or false.

## Impact Details

Since the return value of `success` is not checked, the operation multisig would need to wait for another `delay` amount of time before being able to execute the transaction again. Due to this, normal and/or important operations are delayed further (by atleast `MINIMUM_DELAY` amount of time).

**Some points to note:**

1. The failure of the external call could go unnoticed by the operators since `executeTransaction()` went through successfully.
2. The operations multisig would need to reach quorum again before being able to queue up the transaction again.
3. On request of the operations multisig, the community multisig may or may not take over this task since it would require them to reach the expected quorum among themselves in the first place.
4. If calling the transaction calldata is restricted to only the operations multisig, the community multisig would be of no help.
5. Overall, there would still be a significant delay caused in the execution of the transaction, which may or may not hurt the protocol or users depending on the reason behind the call.

## Vulnerability Details

Here is the whole process:

1. First, let's take a look at the `MINIMUM_DELAY` value in the Timelock.sol contract:

```solidity
File: Timelock.sol
101:     uint256 public constant MINIMUM_DELAY = 7 days; 
```

2. Operation multisig queues up a transaction to call `callData` on `target` contract using function `queueTransaction()`:

```solidity
File: Timelock.sol
144:     function queueTransaction(
145:         address target,
146:         bytes memory callData,
147:         uint256 operationId
148:     ) public returns (bytes32) {
149:         if (msg.sender != OPERATIONS_MULTISIG) {
150:             revert Unauthorized();
151:         }
152:       
153:         bytes32 txHash = keccak256(abi.encode(target, callData, operationId));
154:         uint256 lockedUntil = block.timestamp + delay;
155:         if (queue[txHash] != 0) {
156:             revert InvalidTransaction(txHash);
157:         }
158:         queue[txHash] = lockedUntil;
159:         // solhint-disable-next-line func-named-parameters
160:         emit TransactionQueued(
161:             txHash,
162:             target,
163:             callData,
164:             operationId,
165:             lockedUntil
166:         );
167: 
168:         return txHash;
169:     }
```

3. Once the delay has passed, the operations multisig calls function `executeTransaction()`:

* **Note: Line 255 below clears the `queue[txHash]` value before calling internal function \_executeTransaction() on the next line.**

```solidity
File: Timelock.sol
226:     function executeTransaction(
227:         address target,
228:         bytes calldata callData,
229:         uint256 operationId
230:     ) external returns (bool success, bytes memory returnData) {
231:         // Community Multisig can do things without any delay
233:         
234:         if (msg.sender == COMMUNITY_MULTISIG) {
235:             return _executeTransaction(target, callData);
236:         }
237: 
238:         // Operations multisig needs to queue it and then execute after a delay
239:         if (msg.sender != OPERATIONS_MULTISIG) {
240:             revert Unauthorized();
241:         }
242: 
243:         bytes32 txHash = keccak256(abi.encode(target, callData, operationId));
244:         uint256 lockedUntil = queue[txHash];
245: 
246:         // slither-disable-next-line incorrect-equality
247:         if (lockedUntil == 0) {
248:             revert InvalidTransaction(txHash);
249:         }
250: 
251:         if (block.timestamp < lockedUntil) {
252:             revert Locked(txHash, lockedUntil);
253:         }
254: 
255:         queue[txHash] = 0;
256:         (success, returnData) = _executeTransaction(target, callData);
257:        
258: 
259:         emit TransactionExecuted(txHash, target, callData, operationId);
260: 
261:         return (success, returnData);
262:     }
```

4. In function \_executeTransation(), the `callData` is called on the `target` contract, which provides the return values to the executeTransaction() function. These values are stored in `success` and `returnData` on Line 256 above.

```solidity
File: Timelock.sol
303:     function _executeTransaction(
304:         address target,
305:         bytes calldata callData
306:     ) internal returns (bool, bytes memory) {
307:         // slither-disable-next-line arbitrary-send-eth
308:         return target.call(callData);
309:     }
```

5. Now if the external call had succeeded, the event `TransactionExecuted` is emitted and the execution ends. But if the external call failed, there is no check put in place after Line 256 to ensure the whole execution context reverts. Due to this, the event `TransactionExecuted` is still emitted though the external call did not succeed.

```solidity
256:         (success, returnData) = _executeTransaction(target, callData);
257:        
258: 
259:         emit TransactionExecuted(txHash, target, callData, operationId);
```

6. Since the `queue[txHash]` was cleared previously, the operations multisig would be required to queue the transaction again and wait for another `delay` amount of time before being able to execute it.

The responsibility should be upto the contract to handle and ensure that failed transactions revert to avoid this issue of further delays in transaction execution.

## Mitigation

The operations multisig would not need to queue the transaction again if the value of success is checked for the failed call and the whole execution context is reverted.

Solution: Check the return value for both calls made by operation multisig and community multisig as shown below:

For operations multisig:

```solidity
File: Timelock.sol
256:         (success, returnData) = _executeTransaction(target, callData); 
257:         if (!success) revert(); // Revert with some reason as needed
258:         emit TransactionExecuted(txHash, target, callData, operationId);
```

For community multisig:

```solidity
File: Timelock.sol
234:         if (msg.sender == COMMUNITY_MULTISIG) {
235:             (success, returnData) = _executeTransaction(target, callData); 
236:             if (!success) revert(); // Revert with some reason as needed
238:             return (success, returnData);
238:         }
```

## Proof of Concept

Some points to understand the POC:

* Add the POC to the Timelock.t.sol file
* Run the test using `forge test --fork-url <ETH_MAINNET_RPC_URL> --match-test testReturnValueNotCheckedIssue -vvvvv`
* The use of `-vvvvv` displays the traces, which shows how the call goes through though we encountered an `InvalidDelay` error.
* The POC also demonstrates how `queue[txHash]` is deleted for a failed external call, due to which `executeTransaction()` cannot be called again, forcing the operations multisig to queue up the transaction again.
* The traces/logs have been added below the POC for proof as well as through a screenshot.
* Note, although in the POC I've demonstrated the problem through a simple transaction (i.e failing call to setDelay() - which succeeds), the issue exists for any reverting external call.

```solidity
    function testReturnValueNotCheckedIssue() public {
        vm.startPrank(timelock.OPERATIONS_MULTISIG());

        bytes memory callData = abi.encodeCall(Timelock.setDelay, (4 days));

        assertTrue(timelock.delay() != 4 days, "initial delay");

        uint256 operationId = 1234;

        bytes32 txHash = timelock.queueTransaction(address(timelock), callData, operationId);

        uint256 lockedUntil = block.timestamp + timelock.delay();

        vm.warp(lockedUntil + 1);

        // vm.expectRevert(); 
        timelock.executeTransaction(address(timelock), callData, operationId);

        // Confirms that queue[txHash] was cleared for a failed operation
        assertEq(timelock.queue(txHash), 0);

        // Confirms that executeTransaction() cannot be called again since queue[txHash] was deleted. Thus, the team would require to queue the transaction again
        vm.expectRevert(); 
        timelock.executeTransaction(address(timelock), callData, operationId);
    }
```

### Traces

* We can observe the event emission of `TransactionExecuted()` occurring though `success` is false.
* The POC also asserts that `queue[txHash]` is cleared i.e. 0
* It also demonstrates that `executeTransaction` cannot be called again.

```solidity
    ├─ [4782] Timelock::executeTransaction(Timelock: [0x9696768d5e2B611BD89181D54AeB3259Bab9616F], 0xe177246e0000000000000000000000000000000000000000000000000000000000054600, 1234)
    │   ├─ [443] Timelock::setDelay(345600 [3.456e5])
    │   │   └─ ← InvalidDelay(345600 [3.456e5])
    │   ├─ emit TransactionExecuted(txHash: 0xa0e566c0a7bef9d8a58d3a86b5cefe6a7318c000d2cacc014510f85dba20b8c0, target: Timelock: [0x9696768d5e2B611BD89181D54AeB3259Bab9616F], callData: 0xe177246e0000000000000000000000000000000000000000000000000000000000054600, operationId: 1234)
    │   └─ ← false, 0x4c89d5980000000000000000000000000000000000000000000000000000000000054600
    ├─ [502] Timelock::queue(0xa0e566c0a7bef9d8a58d3a86b5cefe6a7318c000d2cacc014510f85dba20b8c0) [staticcall]
    │   └─ ← 0
    ├─ [0] VM::expectRevert(custom error f4844814:)
    │   └─ ← ()
    ├─ [1332] Timelock::executeTransaction(Timelock: [0x9696768d5e2B611BD89181D54AeB3259Bab9616F], 0xe177246e0000000000000000000000000000000000000000000000000000000000054600, 1234)
    │   └─ ← InvalidTransaction(0xa0e566c0a7bef9d8a58d3a86b5cefe6a7318c000d2cacc014510f85dba20b8c0)
    └─ ← ()
```
