# #43136 \[BC-High] Multiple transactions sent by the same account in the same block timeframe can get stuck in the TranactionPipe core\_mempool

**Submitted on Apr 2nd 2025 at 16:27:25 UTC by @niroh for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43136
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

## Description

## Brief/Intro

Multiple transactions sent by the same account in the same block timeframe can get stuck in the TranactionPipe core\_mempool because of how account sequence numbers are handled in the core\_mempool

## Vulnerability Details

The core of the issue is in how aptos-core::core\_mempool handles account sequence numbers, combined with how Movement::TransactionPipe works with the core\_mempool.

Some background:\
A. TransactionPipe adds submitted transactions to core\_mempool with an account sequence number taken from the db\_reader. This sets the account sequence number in an internal map in core\_mempool, that it used for validating transaction sequence numbers when a batch it retrieved from the mempool (TransactionStore::sequence\_numbers). as can be seen in this code:

```rust
let db_seq_num = get_account_sequence_number(&state_view, transaction.sender())?;
info!(
    tx_sender = %transaction.sender(),
    db_seq_num = %db_seq_num,
    tx_seq_num = %transaction.sequence_number(),
);
let tx_hash = transaction.committed_hash();
let status = self.core_mempool.add_txn(
    transaction,
    ranking_score,
    db_seq_num, 
    TimelineState::NonQualified,
    true,
);
```

B. TransactionPipe retrieves batches of tranactions from core\_mempool in windows of MEMPOOL\_INTERVAL (240ms), and sends them to the Ingress Task which aggregates them and sends them to the da node every 2 seconds\
C. Batches are retrieved from core\_mempool by calling core\_mempool::get\_batch\_with\_ranking\_score. This function logic only allows transactions from a specific account to be included, if they start with the current account sequence number (and possibly upwards). This logic uses the sequence\_numbers map mentioned in A.\
D. the sequence\_numbers map is updated with the db value when a new transaction is added to the mempool.

The issue:\
Because TransactionPipe retrieves multiple batches from core\_mempool within the same block building time, transactions (with consecutive sequence numbers) arriving from the same account during different batches will reset the account mapped sequence number, which will cause them to stay stuck in the core\_mempool.

The POC scenario below will explain further.

## Impact Details

Transactions sent to the mempool are expected to be executed in order. In this scenario some of the transactions will remain stuck and never execute.

## Proof of Concept

## Proof of Concept

### vulnerability scenario

1. Account A's current sequence number is 10. It sends 3 transactions: Tx1 (seqnum 10), Tx2 (seqnum 11), and Tx3 (seqnum 12). All are sent when the head block in 100 and block 101 is being built.
2. Tx1 arrives first into the mempool, mapping account A to seqnum 10 in the core\_mempool.
3. TransactionPipe::tick\_mempool\_sender in invoked and calls get\_batch\_with\_ranking\_score which extracts Tx1 from the pool. It also calls core\_mempool.commit\_transaction which increments Account A seqnum to 11 in the pool
4. Tx2 arrives next and enters the mempool, when it is added to core\_mempool, it resets the seqnum mapped to account A in the pool to 10 (since the head block is still 100).
5. When the next TransactionPipe::tick\_mempool\_sender is invoked, Tx2 is not retrieved, because core\_mempool sees account A seqnum as 10, so tx 11 is skipped.
6. The same happens when Tx3 arrives
7. Both Tx2 and Tx3 are now stuck in the mempool and will remain stuck after Tx1 is executed and there in a new block, because there is nothing that updates the core\_mempool inner seqnum map for account A
8. The core\_mempool internal mapping of seqnum for account A will only be updated when Account A will send another transaction, however by then Tx2 and Tx3 are likely to be expired.
