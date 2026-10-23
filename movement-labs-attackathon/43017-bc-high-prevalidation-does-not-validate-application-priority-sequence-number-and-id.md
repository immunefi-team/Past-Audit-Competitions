# #43017 \[BC-High] Prevalidation does not validate application priority, sequence number and ID

**Submitted on Mar 31st 2025 at 17:28:09 UTC by @KlosMitSoss for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43017
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

## Brief/Intro

When the DA light node validates the transaction data, it does not validate the application priority nor sequence number. As a result, it is not ensured that these values fit the Aptos transaction. For example, a transaction directly sent to the DA light node could still have an application priority of 0 (highest priority) even though this priority is not justified by the gas price specified in the transaction data.

## Vulnerability Details

The sequencer DA light node verifies that a transaction is valid.

```rust
	async fn batch_write(
		&self,
		request: tonic::Request<grpc::BatchWriteRequest>,
	) -> std::result::Result<tonic::Response<grpc::BatchWriteResponse>, tonic::Status> {
		let blobs_for_submission = request.into_inner().blobs;

		// make transactions from the blobs
		let mut transactions = Vec::new();
		for blob in blobs_for_submission {
			let transaction: Transaction = serde_json::from_slice(&blob.data)
				.map_err(|e| tonic::Status::internal(e.to_string()))?;

			match &self.prevalidator {
				Some(prevalidator) => {
					// match the prevalidated status, if validation error discard if internal error raise internal error
>>					match prevalidator.prevalidate(transaction).await {
						Ok(prevalidated) => {
							transactions.push(prevalidated.into_inner());
						}
                        ... ...
	}
```

However, this validation is only done for the `transaction.data` and not for the application priority, sequence number, and ID.

```rust
	async fn prevalidate(
		&self,
		transaction: Transaction,
	) -> Result<Prevalidated<AptosTransaction>, Error> {
		// Only allow properly signed user transactions that can be deserialized from the transaction.data()
		let aptos_transaction: AptosTransaction =
>>			bcs::from_bytes(&transaction.data()).map_err(|e| {
				Error::Validation(format!("Failed to deserialize AptosTransaction: {}", e))
			})?;

		aptos_transaction
			.verify_signature()
			.map_err(|e| Error::Validation(format!("Failed to prevalidate signature: {}", e)))?;

		Ok(Prevalidated::new(aptos_transaction))
	}
```

As a result, a transaction could be sent directly to the DA light node with an application priority that does not fit the gas price specified in the transaction data. For example, the price to be paid per gas unit could be really low but the application priority could still be 0 (highest priority).

The fact that there is some prevalidation for the transaction data implies that it is possible to send transactions with aribtrary data. Hence, the prevalidation is needed. However, then the application priority could also be some arbitrary data. But there is no validation for this.

Furthermore, by using a fake transaction ID that matches the transaction ID of a transaction that an attacker wants to DoS, the mempool can be filled with the fake transaction such that the legitimate transaction is skipped.

```rust
	async fn add_mempool_transactions(
		&self,
		transactions: Vec<MempoolTransaction>,
	) -> Result<(), anyhow::Error> {
		let db = self.db.clone();
		tokio::task::spawn_blocking(move || {
			let mempool_transactions_cf_handle = db
				.cf_handle(cf::MEMPOOL_TRANSACTIONS)
				.ok_or_else(|| Error::msg("CF handle not found"))?;
			let transaction_lookups_cf_handle = db
				.cf_handle(cf::TRANSACTION_LOOKUPS)
				.ok_or_else(|| Error::msg("CF handle not found"))?;

			// Add the transactions and update the lookup table atomically
			// in a single write batch.
			// https://github.com/movementlabsxyz/movement/issues/322

			let mut batch = WriteBatch::default();

			for transaction in transactions {
>>				if Self::internal_has_mempool_transaction(&db, transaction.transaction.id())? {
>>					continue;
				}
                ... ...
	}
```

## Impact Details

Transactions could be sent with a lower application priority than what it should have based on the price to be paid per gas unit. Furthermore, transactions with a known transaction ID can be DoSed.

## References

Code references are provided throughout the report.

## Proof of Concept

## Proof of Concept

1. A transaction with valid data is directly sent to the sequencer DA light node. This transaction is then prevalidated (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/mempool/move-rocks/src/lib.rs#L139-L141>).
2. It is validated that the transaction data can be deserialized (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/prevalidator/src/aptos/transaction.rs#L15-L18>). However, it is not ensured that the application priority fits the price to be paid per gas unit. Sequence number and transaction ID are also not validated.
3. In the case of a fake application priority with 0 value, this transaction is now the first to get processed.
