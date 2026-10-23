# #41466 \[BC-Medium] Incorrect sequence number tracking in mempool commit

**Submitted on Mar 15th 2025 at 16:16:48 UTC by @Rhaydden for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41466
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

## Brief/Intro

`commit_transaction` in `TransactionPipe`impl wronglly uses the on-chain sequence number instead of the transaction's actual sequence number when committing transactions to the mempool. This allows invalid transaction ordering, enabling transactions with lower sequence numbers to be accepted after higher ones. In production, this could lead to double-spends, failed txns, and consensus failures due to invalid transaction sequences.

## Vulnerability Details

The `core_mempool.commit_transaction()` call in `submit_transaction` uses `sequence_number` (on-chain value) instead of the transaction's `transaction_sequence_number`. This breakss the mempool's sequence tracking contract.

<https://github.com/immunefi-team/attackathon-movement//blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L299>

```rust
self.core_mempool.commit_transaction(&sender, sequence_number); // Using on-chain sequence number
```

The variable `sequence_number` is obtained from the validation process through:

```rust
let sequence_number = match self.has_invalid_sequence_number(&transaction)? {
    SequenceNumberValidity::Valid(sequence_number) => sequence_number,
    SequenceNumberValidity::Invalid(status) => {
        return Ok(status);
    }
};
```

In `has_invalid_sequence_number`, the sequence number is retrieved using the current on-chain state rather than the transaction's actual sequence number.

Looking at how `commit_transaction` is implemented in the Aptos `TransactionStore` contract:

<https://github.com/aptos-labs/aptos-core/blob/308d59ec2e7d9c3937c8b6b4fca6dd7e97fd3196/mempool/src/core\\_mempool/transaction\\_store.rs#L584C4-L590C6>

```rust
 pub fn commit_transaction(&mut self, account: &AccountAddress, sequence_number: u64) {
        let current_seq_number = self.get_sequence_number(account).map_or(0, |v| *v);
        let new_seq_number = max(current_seq_number, sequence_number + 1);
        self.sequence_numbers.insert(*account, new_seq_number);
        self.clean_committed_transactions(account, new_seq_number);
        self.process_ready_transactions(account, new_seq_number);
    }
```

This function:

* Gets the current tracked sequence number for the account
* Sets the new sequence number to be the maximum of (current\_seq\_number, sequence\_number + 1)
* Updates the account's sequence number in the mempool
* Processes potentially ready transactions for the account

Problem is when a txn with a higher sequence number (e.g., 5) than the current on-chain state sequence number (e.g., 3) is submitted. The mempool incorrectly updates its tracking to use the on-chain sequence number + 1 (4) instead of the transaction's sequence number + 1 (6). This causes subsequent transactions from the same account with sequence numbers between these values (e.g., 4, 5) to be incorrectly rejected as "too old" or "invalid sequence number"

Aptos maintains strict sequence number ordering by tracking the highest sequence number seen. Aptos uses `max()` to ensure monotonic increases.

Ourown impl breaks these guarantees by using on-chain sequence numbers instead of transaction sequence numbers. Its not maintaining proper tracking of highest seen sequence numbers thus, allowing out-of-order transaction acceptance.

`sequence_number` in `transaction_pipe.rs` is the on-chain value (from `has_invalid_sequence_number()`), not the transaction's actual sequence number. This causes the mempool to track an incorrect "next expected sequence number", allowing older transactions to bypass sequence checks after newer ones are added.

## Impact Details

This falls under "Causing network processing nodes to process transactions from the mempool beyond set parameters" with the following specific impacts:

* Mempool can accept transactions out of sequence (e.g., TX4 after TX5). breaks fundamental txn ordering guarantees.
* Nodes process transactions in incorrect sequence

Most especiallly, this enables a potential transaction reordering attack case.

## References

Aptos mempool implementation: `transaction_store.rs` ---> <https://github.com/aptos-labs/aptos-core/blob/308d59ec2e7d9c3937c8b6b4fca6dd7e97fd3196/mempool/src/core\\_mempool/transaction\\_store.rs#L584C4-L590C6>

<https://github.com/immunefi-team/attackathon-movement//blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L299>

## Proof of Concept

## Proof of Concept

1. Initial State:

* On-chain sequence for Alice: 3
* Mempool stored sequence: 3

2. Submit TX5 (seq=5):

* Passes `has_invalid_sequence_number` check (5 ≤ 3 + 32).
* `commit_transaction(3)` updates mempool to seq=4 (3+1).

3. Submit TX4 (seq=4):

Check: 4 ≥ 4 (current mempool seq) → Accepted.

4. Result:\
   Mempool contains TX4 and TX5 → Invalid ordering → Consensus rejects block.

## Fix

Use the transaction's actual sequence number (`transaction_sequence_number`) instead of the validated sequence number from the on-chain state (`sequence_number`) when committing the transaction to the mempool.

```diff
- self.core_mempool.commit_transaction(&sender, sequence_number);
+ self.core_mempool.commit_transaction(&sender, transaction_sequence_number);
```
