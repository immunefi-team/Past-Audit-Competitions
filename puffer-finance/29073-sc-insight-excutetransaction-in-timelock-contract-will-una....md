# 29073 - \[SC - Insight] excuteTransaction in timelock contract will una...

Submitted on Mar 6th 2024 at 15:16:49 UTC by @Kenzo for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #29073

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x3C28B7c7Ba1A1f55c9Ce66b263B33B204f2126eA#code>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Vulnerability Details

The function `timelock::executeTransaction` Executes a transaction after the delay period for Operations Multisig and Community multisig can execute transactions without any delay. Expected behavior is that the function should revert if the call to the target contract fails. If the target contract with no contract code added or yet to be deployed is set as input of `timelock::executeTransaction`, the function won't revert which is not ideal behavior of the function. Similar issue was found with OZ: <https://github.com/OpenZeppelin/openzeppelin-contracts/issues/3874>

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

## Impact Details

1. Transactions with contracts that are yet to deploy like new tokens whose address can be predetermined at risk while interacting with this contract.
2. `timelock::executeTransaction` will execute invalid transaction successfully instead of failing on non-existent contract.
3. Funds can be lost in some scenario.

## Risk Breakdown

Low Severity as the function is being controlled by the `OPERATIONS_MULTISIG` and `COMMUNITY_MULTISIG`.

## Recommendation

Consider adding contract existence check on `timelock.executeTransaction`.

POC is attached.

## Proof of concept

The POC can be run by placing the following code in `PufferTest.integration.t.sol` and running this command: `forge test --mt test_call_to_unknown_contract -vv`

```solidity
function test_call_to_unknown_contract() public {
        vm.startPrank(timelock.OPERATIONS_MULTISIG());
        address target = makeAddr("UnknownContract");

        bytes memory callData = abi.encodeCall(Timelock.setDelay, (15 days));

        assertTrue(timelock.delay() != 15 days, "initial delay");

        uint256 operationId = 1234;

        bytes32 txHash = timelock.queueTransaction(target, callData, operationId);
        
        console2.log("Transaction added to the queue:");
        console2.logBytes32(txHash);

        uint256 lockedUntil = block.timestamp + timelock.delay();
        

        vm.warp(lockedUntil + 1);
        
        try timelock.executeTransaction(target, callData, operationId) {
             console.log("Transaction successful despite non existent contract.");
        }
        catch {
             console.log("Transaction unsuccessful.");
        }
    }
```

Output:

```
[PASS] test_call_to_unknown_contract() (gas: 41202)
Logs:
  Transaction added to the queue:
  0x49df1e329caafa22eef3d051ceb862da04b4bf32c005264307826245abfea259
  Transaction successful despite non existent contract.

Test result: ok. 1 passed; 0 failed; finished in 8.45ms
```
