# #43324 \[BC-High] insufficient validation in da light node allows malicious override of application priority&#x20;

## #43324 \[BC-High] Insufficient Validation in DA Light Node Allows Malicious Override of \`application\_priority\`

**Submitted on Apr 4th 2025 at 15:57:44 UTC by @Blockian for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43324
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/protocol/light-node>
* **Impacts:**
  * Modification of transaction fees outside of design parameters
  * Griefing
  * Tx Censorship

### Description

## Movement Bug Report

### Insufficient Validation in DA Light Node Allows Malicious Override of `application_priority`

#### Summary

The DA Light Node does not validate the `application_priority` field on incoming transactions. Because this field is not part of the signed payload, a malicious node can arbitrarily alter the priority of a transaction, allowing for unfair ordering or censorship of other transactions.

***

### Root Cause Analysis

During the `batch_write` process, the `prevalidator` validates incoming transactions through the following logic:

```rs
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

However, the `application_priority` is **not part of the signed transaction payload**. This means:

* The priority value is not cryptographically tied to the transaction content.
* The value can be modified by any party submitting the transaction (e.g., a malicious node).
* There is no verification step to ensure the priority value matches the calculated original intent.

### Impact

The `application_priority` field plays a role in transaction ordering. Allowing it to be arbitrarily modified introduces several vulnerabilities:

* **Priority Inflation**: A malicious actor can artificially boost the priority of their own transactions, gaining unfair access to limited execution bandwidth.
* **Transaction Starvation**: Legitimate transactions with genuine lower priorities may be delayed or dropped due to queue saturation.
* **Targeted Censorship**: An attacker can reduce the priority of a victim’s transaction to push it behind attacker-controlled transactions.
* **Loss of Fairness and Predictability**: The system’s ability to enforce deterministic or economic ordering (e.g., fee-based) is undermined if priority is mutable.

### Proposed Fixes

**Derive Priority On the Da Light Node**\
Eliminate the client-provided `application_priority` entirely and compute it deterministically on the Light Node.

### Proof of Concept

### Proof of Concept (PoC)

1. Run the DA Light Node.
2. Identify a legitimate transaction to target.
3. Copy the transaction’s `data`, `sequence_number`, and `id`, but modify the `application_priority` to a higher or lower value.
4. Send a gRPC request to the Light Node’s `write_batch` endpoint with the modified transaction.

```rs
// Pseudo-code
let manipulated_tx = Transaction {
	data: original_tx.data.clone(),
	application_priority: 9999, // Arbitrarily priority
	sequence_number: original_tx.sequence_number,
	id: original_tx.id,
};
```
