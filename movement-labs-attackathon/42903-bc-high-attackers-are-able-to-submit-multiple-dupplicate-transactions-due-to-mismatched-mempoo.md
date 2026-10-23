# #42903 \[BC-High] Attackers are able to submit multiple dupplicate transactions due to mismatched Mempool Implementation

**Submitted on Mar 28th 2025 at 17:54:17 UTC by @Berserk for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42903
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

## Brief/Intro

A vulnerability exists in Movement Protocol's transaction handling where attackers can DOS the network by submitting duplicate transactions with increased/incremented gas prices, exploiting a mismatched mempool implementation inherited from Aptos.

(the mempool in aptos, accepts dupplicate transaction, if the second transaction increases the unit gas price and will use it instead of the first one. In movment, both transactions will be added to the block and accepted.)

## Vulnerability Details

The vulnerability stems from Movement using Aptos's mempool implementation despite having a different transaction lifecycle:

1. **Transaction Flow in Movement:**
   * Transaction received via RPC
   * Initial validation (signature and format)
   * Processed by transaction\_pipe.rs
   * If valid, directly passed to `transaction_ingress` (sent via channel`transaction_sender`), and then directly be added to next block via `transaction_ingress`
2. **Key Implementation Issue:**

`protocol-units/execution/maptos/opt-executor/src/background/transaction_pipe.rs`

```rust
async fn submit_transaction(&mut self, transaction: SignedTransaction) {
    // ... existing code ...
@>>    let status = self.core_mempool.add_txn(
        transaction.clone(),
        0,
        sequence_number,
        TimelineState::NonQualified,
        true,
    );
    
    if status.code == MempoolStatusCode::Accepted {
        // Transaction directly sent to transaction_ingress
@>>        self.transaction_sender.send((application_priority, transaction)).await?;
    }
    // ... existing code ...
}
```

As we can see in the `submit_transaction()` function, the final step in the checks would be to try to add the transaction to the mempool. and if it get accepted in the mempool we will directly send it using `transaction_sender` channel to the `transaction_ingress` to be posted in the next blob/block.

The vulnerability however occurs because we use the exact mempool implementation. in `add_txn()` we will do some checks and then will try to insert the given transaction in the transaction\_store (`let status = self.transactions.insert(txn_info)`).

3. **Problematic Behavior:**\
   The vulnerability occurs because Aptos's mempool allows duplicate transactions if they increase gas price:

`aptos-core/mempool/src/core_mempool/transaction_store.rs#L186-L200`

```rust
    /// Insert transaction into TransactionStore. Performs validation checks and updates indexes.
    pub(crate) fn insert(&mut self, txn: MempoolTransaction) -> MempoolStatus {
        let address = txn.get_sender();
        let txn_seq_num = txn.sequence_info.transaction_sequence_number;
        let acc_seq_num = txn.sequence_info.account_sequence_number;

@>>        // If the transaction is already in Mempool, we only allow the user to
@>>        // increase the gas unit price to speed up a transaction, but not the max gas.
        //
        // Transactions with all the same inputs (but possibly signed differently) are idempotent
        // since the raw transaction is the same
```

In Movement's implementation, by the time a duplicate transaction with higher gas price is accepted, the original transaction has already been sent to transaction\_ingress. This leads to both transactions being processed (added to the block and then executed), unlike Aptos where only one would be executed.

> N.B by creating a batch of transaction with incrementing gas\_unit\_price, we will be able to bypass the validity check and submit multiple duplicates transactions to be processes by the working nodes

## Impact

* Attackers can flood the network with duplicate transactions (only pay gas for 1 transaction) -> DoS
* Each duplicate will be processed during execution stage by the network processing node -> Causing network processing nodes to process transactions from the mempool beyond set parameters

## Mitigation

As a dirty fix, we simply recommend that the function insert() in tansaction\_store.rs from aptos\_core be adjusted to also reject dupplicated transactions that increase the unit gas price.

## References

* protocol-units/execution/maptos/opt-executor/src/background/transaction\_pipe.rs
* aptos-core/mempool/src/core\_mempool/transaction\_store.rs

## Proof of Concept

## Proof of Concept

### Attack Method:

1. Use the RPC endpoint `submit batch transactions`
2. Submit a batch of 10 identical Aptos transfer transactions
3. Each subsequent transaction increases the unit\_gas\_field by 10
4. The order of transactions (increasing gas price) is crucial for the attack
5. All the transaction in the batch will be added to the block and executed (see fullnode log)

### Coded Poc

* `fullnodelog`: <https://gist.github.com/aliX40/d5b618aa2bf01a6d83597b7f324ad6e9>
* `main.rs` Poc: <https://gist.github.com/aliX40/37a95cc64d96757ec1837b6c8cff1ad9>
* `cargo.toml` for PoC: <https://gist.github.com/aliX40/6fd53043b29b8e3c9ccf5f56a79c59c0>

This is the result of running the poc in `main.rs`

```log
     Running `/root/attack/attackathon-movement/target/debug/transaction-tester`
2025-03-28T17:30:42.276289Z  INFO transaction_tester: Initializing transaction test
2025-03-28T17:30:42.375215Z  INFO transaction_tester: Account address: 0x971fadb4e8f4fe52d0b05e69f1e2d0983e3ed0e88e187c148aa68883a6b2324a
2025-03-28T17:30:42.375373Z  INFO transaction_tester: Auth key: 971fadb4e8f4fe52d0b05e69f1e2d0983e3ed0e88e187c148aa68883a6b2324a
2025-03-28T17:30:42.377324Z  INFO transaction_tester: Public key: 8ffa3fca8ad5e029603317d066cbdf15cb356587228aa29f56d0f8f0f6800b55
2025-03-28T17:30:49.796875Z  INFO transaction_tester: Account address: 0x2702094c289de62091b94a44e5cf470e4379c864759af02fb9c19b725e4f3851
2025-03-28T17:30:49.796960Z  INFO transaction_tester: Auth key: 2702094c289de62091b94a44e5cf470e4379c864759af02fb9c19b725e4f3851
2025-03-28T17:30:49.797006Z  INFO transaction_tester: Public key: aa98df9068bfa11aadeba0cc1bde5e9a475c7a0715a3b155813e4c9eda7b3df4
Test Account: 10000700
2025-03-28T17:31:18.648647Z  INFO transaction_tester: Creating test transactions
2025-03-28T17:31:20.085503Z  INFO transaction_tester: Batch of 10 transactions submitted successfully
Response: Response { inner: TransactionsBatchSubmissionResult { transaction_failures: [] }, state: State { chain_id: 27, epoch: 66, version: 188, timestamp_usecs: 1743183052019615, oldest_ledger_version: 0, oldest_block_height: 0, block_height: 66, cursor: None } }
Test Account: 9999790
2025-03-28T17:31:35.112875Z  INFO transaction_tester: Transaction test completed successfully
```

The full-node logs confirm that all 10 transactions were included in the block and processed during execution, as shown in the `compute_status_for_input_txns:` output.

In the first batch, the initial transfer failed due to insufficient funds (though the sequence number was still incremented). All subsequent transactions were discarded for having outdated sequence numbers.

```log
compute_status_for_input_txns: 
[Keep(Success),
Keep(MoveAbort { location: 0000000000000000000000000000000000000000000000000000000000000001::transaction_validation, code: 132077, info: None })
Discard(SEQUENCE_NUMBER_TOO_OLD),
Discard(SEQUENCE_NUMBER_TOO_OLD),
Discard(SEQUENCE_NUMBER_TOO_OLD)]
```

This is the results of the second batch read form the full node logs: (all the transactions in it have an old transaction nr and thus dropped )

```log
compute_status_for_input_txns:
[Keep(Success),
Discard(SEQUENCE_NUMBER_TOO_OLD), 
Discard(SEQUENCE_NUMBER_TOO_OLD), 
Discard(SEQUENCE_NUMBER_TOO_OLD), 
Discard(SEQUENCE_NUMBER_TOO_OLD), 
Discard(SEQUENCE_NUMBER_TOO_OLD), 
Discard(SEQUENCE_NUMBER_TOO_OLD)]
```

As we can see we were able to submit 10 dupplicated through rpc bypassing mempool restrictions and forcing the full nodes to process those transactions during the execution stage.

( see full log here: <https://gist.github.com/aliX40/d5b618aa2bf01a6d83597b7f324ad6e9>)

> N.B in each block the first transaction is a system generated one and that's why we have always keep(success) at first position in the compute\_status\_for\_input\_txns array
