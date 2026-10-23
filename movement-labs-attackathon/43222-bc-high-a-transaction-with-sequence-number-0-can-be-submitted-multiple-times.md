# #43222 \[BC-High] A transaction with sequence number 0 can be submitted multiple times

**Submitted on Apr 3rd 2025 at 19:48:29 UTC by @KlosMitSoss for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43222
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

## Brief/Intro

When checking if a transaction's sequence number has been used before using the `has_invalid_sequence_number()` function, the case when the sequence number is 0 is not properly handled.\
As a result, a transaction with sequence number 0 can be submitted multiple times.

## Vulnerability Details

When the previously used sequence number for the transaction sender is retrieved from the `used_sequence_number_pool`, it defaults to 0 if no sequence number is found.

```rust
	fn has_invalid_sequence_number(
		&self,
		transaction: &SignedTransaction,
	) -> Result<SequenceNumberValidity, Error> {
		// check against the used sequence number pool
		let used_sequence_number = self
			.used_sequence_number_pool
			.get_sequence_number(&transaction.sender())
>>			.unwrap_or(0);
        ... ...
	}
```

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L164-L218>

Later in the function, if the last used sequence number was 0 (or none was found), the minimum allowed sequence number also becomes 0.

```rust
	fn has_invalid_sequence_number(
		&self,
		transaction: &SignedTransaction,
	) -> Result<SequenceNumberValidity, Error> {
        ... ...
>>		let min_used_sequence_number =
			if used_sequence_number > 0 { used_sequence_number + 1 } else { 0 };
        ... ...
	}
```

This means that transactions with sequence number 0 can be submitted multiple times to the mempool, bypassing the normal sequence number check.

To mitigate this, a distinction should be made between the absence of a used sequence number and the used sequence number being 0. If no used sequence number exists, 0 should be allowed. However, if 0 is the last used sequence number, it should not be allowed (only 1,2 and so on should be allowed).

## Impact Details

A transaction with the sequence number being 0 could be submitted multiple times.

While technically the impact matches "Causing network processing nodes to process transactions from the mempool beyond set parameters" which is a "High" impact, we belive the issue should be "Low".

## References

Code references are provided throughout the report.

## Proof of Concept

## Proof of Concept

1. The `used_sequence_number_pool` is currently empty and a transaction with sequence number 0 is submitted.
2. The transaction with sequence number 0 can be submitted again.
