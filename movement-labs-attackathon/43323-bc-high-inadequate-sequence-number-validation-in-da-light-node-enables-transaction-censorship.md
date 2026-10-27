# #43323 \[BC-High] inadequate sequence number validation in da light node enables transaction censorship

## #43323 \[BC-High] Inadequate Sequence Number Validation in DA Light Node Enables Transaction Censorship

**Submitted on Apr 4th 2025 at 15:54:33 UTC by @Blockian for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43323
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/protocol/light-node>
* **Impacts:**
  * Modification of transaction fees outside of design parameters
  * Griefing
  * Tx Censorship

### Description

## Movement Bug Report

### Inadequate Sequence Number Validation in DA Light Node Enables Transaction Censorship

#### Summary

The DA Light Node does not verify that the sequence number declared in the outer `Transaction` object matches the one in the inner, signed `SignedTransaction`. This inconsistency allows a malicious node to submit a valid transaction payload with a **forged sequence number**, leading to censorship and disruption of correct transaction execution.

### Root Cause Analysis

During the `batch_write` process, transaction validation occurs through the following flow:

```rs
#[tonic::async_trait]
	#[tonic::async_trait]
impl PrevalidatorOperations<Transaction, Transaction> for Validator {
	/// Verifies a Transaction as a Valid Transaction
	async fn prevalidate(
		&self,
		transaction: Transaction,
	) -> Result<Prevalidated<Transaction>, Error> {
		let application_priority = transaction.application_priority();
		let sequence_number = transaction.sequence_number();

		let aptos_transaction = AptosTransactionValidator.prevalidate(transaction).await?;
		let aptos_transaction = self
			.whitelist_validator
			.prevalidate(aptos_transaction.into_inner())
			.await?
			.into_inner();

		Ok(Prevalidated(Transaction::new(
			bcs::to_bytes(&aptos_transaction).map_err(|e| {
				Error::Internal(format!("Failed to serialize AptosTransaction: {}", e))
			})?,
			application_priority,
			sequence_number,
		)))
	}
}
```

And `AptosTransactionValidator` performs basic deserialization and signature validation:

```rs
#[tonic::async_trait]
impl PrevalidatorOperations<Transaction, AptosTransaction> for Validator {
	/// Verifies a Transaction as a Valid AptosTransaction
	async fn prevalidate(
		&self,
		transaction: Transaction,
	) -> Result<Prevalidated<AptosTransaction>, Error> {
		// Only allow properly signed user transactions that can be deserialized from the transaction.data()
		let aptos_transaction: AptosTransaction =
			bcs::from_bytes(&transaction.data()).map_err(|e| {
				Error::Validation(format!("Failed to deserialize AptosTransaction: {}", e))
			})?;

		aptos_transaction
			.verify_signature()
			.map_err(|e| Error::Validation(format!("Failed to prevalidate signature: {}", e)))?;

		Ok(Prevalidated::new(aptos_transaction))
	}
}
```

However, **no check is performed to ensure that** `transaction.sequence_number` == `aptos_transaction.raw_txn.sequence_number`. As a result, a malicious node can replace the sequence number in the outer `Transaction` wrapper while keeping the signed inner transaction valid.

### Impact

This vulnerability enables the following attacks:

* **Targeted Censorship**: An attacker can intercept or clone a valid transaction and submit it to the DA with a modified (invalid) sequence number. While the transaction’s signature remains valid, the mismatch causes the execution engine to skip or reject it.
* **Loss of Liveness**: Users may be unable to progress their accounts due to sequence number mismatches on their own transactions.
* **Silent Failure**: Because the transaction is valid at a signature level, it enters the system undetected but fails execution silently, creating confusing behavior for users and developers.
* **Exploitation by Malicious Actors**: Attackers can consistently front-run or delay target transactions simply by replicating them with a sequence number offset.

### Proposed Fixes

**Enforce Sequence Number Consistency**\
During `prevalidate`, explicitly check that the outer `Transaction.sequence_number` matches the inner `raw_txn.sequence_number` from the deserialized `AptosTransaction`. Reject mismatched transactions.

### Proof of Concept

### Proof of Concept (PoC)

1. Start the DA Light Node.
2. Identify a legitimate transaction you wish to censor from a cooperating or observed user.
3. Create a modified version of that transaction by keeping the `data` field the same but changing the `sequence_number` field in the outer `Transaction` wrapper.
4. Submit the altered transaction using the `write_batch` gRPC endpoint:

```rs
// Pseudo-code
let censored_tx = Transaction {
	data: original_tx.data.clone(), // Same signed inner payload
	application_priority: original_tx.application_priority,
	sequence_number: original_tx.sequence_number + 1000, // Invalid value to break execution
	id: original_tx.id,
};
```
