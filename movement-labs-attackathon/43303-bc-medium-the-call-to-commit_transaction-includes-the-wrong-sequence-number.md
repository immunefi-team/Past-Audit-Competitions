# #43303 \[BC-Medium] The call to \`commit\_transaction()\` includes the wrong sequence number

**Submitted on Apr 4th 2025 at 12:16:17 UTC by @HollaDieWaldfee for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43303
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

## Brief/Intro

When a new transaction is submitted, `commit_transaction()` is called with the account's sequence number. However, `commit_transaction()` should be called with `account_sequence_number - 1` instead such that all transactions with sequence number < `account_sequence_number` will be deleted.

## Vulnerability Details

When a transaction gets accepted, `commit_transaction()` is called to handle the transaction commit. This includes deleting all transactions with a lower sequence number. However, the call to `commit_transaction()` is done with the current account's sequence number.

```rust
    pub fn commit_transaction(&mut self, account: &AccountAddress, sequence_number: u64) {
        let current_seq_number = self.get_sequence_number(account).map_or(0, |v| *v);
        let new_seq_number = max(current_seq_number, sequence_number + 1);
        self.sequence_numbers.insert(*account, new_seq_number);
        self.clean_committed_transactions(account, new_seq_number);
        self.process_ready_transactions(account, new_seq_number);
    }
```

As a result, `clean_committed_transactions()` will be called with `account_sequence_number + 1` which means that all transactions with sequence number <= `account_sequence_number` will be deleted. But the current `account_sequence_number` is already a new sequence number which means that it should not be deleted. And in `process_ready_transactions()`, the mempool can prematurely delete the transaction with `sequence_number`.

The solution is to call the `commit_transaction()` function with `sequence_number - 1` when the sequence number is greater than 0, and to not call `commit_transaction()` at all when sequencer number is equal to 0.

## Impact Details

Commiting a new transaction deletes too many transactions.

Although Immunefi classifies the impact of "Causing network processing nodes to process transactions from the mempool beyond set parameters" as a High severity issue, and we submit the issue with High severity according to Immunefi rules, we believe that it should be Low severity.

## References

(1): <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L299>

(2): <https://github.com/immunefi-team/attackathon-movement-aptos-core/blob/627b4f9e0b63c33746fa5dae6cd672cbee3d8631/mempool/src/core\\_mempool/transaction\\_store.rs#L496-L502>

## Proof of Concept

## Proof of Concept

1. `submit_transaction()` is called which calls `commit_transaction()` with the account's sequence number when the submitted transaction gets accepted (reference (1)).
2. Inside of this function, the sequence number gets incremented by 1 and then `clean_committed_transactions()` gets called with the incremented sequence number (reference (2)). As a result, all transactions with sequence number <= `account_sequence_number` will be deleted.
