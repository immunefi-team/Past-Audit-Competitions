# 28688 - \[SC - Insight] Unhandled Failure of \_executeTransaction Call i...

Submitted on Feb 24th 2024 at 00:15:20 UTC by @ihtishamsudo for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28688

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x3C28B7c7Ba1A1f55c9Ce66b263B33B204f2126eA#code>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `executeTransaction` function in `Timelock.sol` does not handle the case where `_executeTransaction` call fails because bool only does not revert but returns true or false

## Vulnerability Details

When a transaction is executed via `executeTransaction` the `_executeTransaction` is called. This `_executeTransaction` function is a `low-level` call that forward call with payload to another contract or address. The problem is that if this call fails if doesn't throw any exception, but instead it returns false as a success value.

```solidity
function _executeTransaction(address target, bytes calldata callData) internal returns (bool, bytes memory) {
    // slither-disable-next-line arbitrary-send-eth
    return target.call(callData);
}
```

However, calling function `executeTransaction` doesn't handle the failure case. the TransactionExecuted event is emitted regardless of the success of the `_executeTransaction` call.

## Impact Details

If the \_executeTransaction call fails, it's likely that something went wrong with the transaction. However, the executeTransaction function would still emit the TransactionExecuted event and return false and an empty bytes array. This could lead to a false sense of successful transaction execution. In a worst-case scenario, this could result in loss of funds or incorrect state changes if actions are taken based on the incorrect assumption that the transaction was successful.

## Mitigation

To handle the failure case, you could add a condition to check the success value and revert the transaction if the call failed

```diff
function executeTransaction(address target, bytes calldata callData, uint256 operationId)
        external
        returns (bool success, bytes memory returnData)
    {
       Code....

        queue[txHash] = 0;
        (success, returnData) = _executeTransaction(target, callData); 
+      if (!success) {
+    revert ("Transaction Failed");
+   }        
        emit TransactionExecuted(txHash, target, callData, operationId);

        return (success, returnData);
    }
```

## References

[Timelock.executeTransaction](https://etherscan.io/address/0x3C28B7c7Ba1A1f55c9Ce66b263B33B204f2126eA?utm_source=immunefi#code#F1#L218)

<https://docs.soliditylang.org/en/v0.8.4/control-structures.html#error-handling-assert-require-revert-and-exceptions>

## Proof of Concept

N/A
