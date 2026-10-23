# 26527 - \[SC - Insight] Possible emission of wrong data in cancelTransa...

Submitted on Dec 4th 2023 at 14:22:36 UTC by @Obin for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #26527

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0xf2991507952d9594e71a44a54fb19f3109d213a5#code>

Impacts:

* Contract can wrongly emit cancellation of not-previously-queued Tx

## Description

## Bug Description

The `cancelTransaction()` only sets the hash of the inputs to false. this implementation is sub-optimal and buggy as contract doesn't check if inputed data is correct.

## Impact

In situation of input error, this will lead to:

1. Not actually deleting Tx as intended
2. Wrongly publishing / emitting a false data to have been cancelled.

## Recommendation

```diff
function cancelTransaction(address target, uint value, string memory signature, bytes memory data, uint eta) public {
        require(msg.sender == admin, "Timelock::cancelTransaction: Call must come from admin.");

        bytes32 txHash = keccak256(abi.encode(target, value, signature, data, eta));
+       require(queuedTransactions[txHash], "Timelock::cancelTransaction:Incorrect input or not queued");
        queuedTransactions[txHash] = false;

        emit CancelTransaction(txHash, target, value, signature, data, eta);
    }

```

## Reference
