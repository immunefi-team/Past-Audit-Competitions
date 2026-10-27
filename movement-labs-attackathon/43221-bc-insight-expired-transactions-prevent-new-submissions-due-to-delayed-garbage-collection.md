# #43221 \[BC-Insight] Expired transactions prevent new submissions due to delayed garbage collection

**Submitted on Apr 3rd 2025 at 19:46:19 UTC by @KlosMitSoss for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43221
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**

## Description

## Brief/Intro

A transaction cannot be submitted when the mempool is full. This remains true even if there are expired transactions that could be garbage collected since the transaction is initially submitted before the garbage collection process occurs.

## Vulnerability Details

The garbage collection functions are called only after `submit_transaction()` is executed. Consequently, even when the mempool is full and contains expired transactions, the new transaction cannot be submitted.

As a result, the transaction must be resubmitted, despite the presence of expired transactions in the mempool. To address this issue, the garbage collection functions should be invoked at the start of the `receive_transaction_tick()` function.

## Impact Details

A transaction cannot be submitted even when there are expired transactions in the mempool. As a result, the transaction needs to be submitted again.

## References

Code references are provided throughout the report

## Proof of Concept

## Proof of Concept

1. The mempool is full but there is at least one transaction that could be garbage collected.
2. The transaction cannot be submitted since the mempool is full and `gc()` is only called after the call to `submit_transaction()` (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L140-L161>).
3. The function needs to be called again to submit the transaction.
