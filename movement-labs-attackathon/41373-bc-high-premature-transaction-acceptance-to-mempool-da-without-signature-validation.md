# #41373 \[BC-High] Premature transaction acceptance to mempool/DA without signature validation

**Submitted on Mar 14th 2025 at 13:02:46 UTC by @Cartel for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41373
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/protocol/prevalidator>
* **Impacts:**
  * Direct loss of funds
  * Causing network processing nodes to process transactions from the mempool beyond set parameters
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## Brief/Intro

When the system is configured without whitelisted accounts, it completely bypasses transaction signature validation. As a result, an attacker can submit transactions with invalid or missing signatures, which will still be accepted into the mempool and subsequently included in the DA. This vulnerability allows attackers to flood the mempool and DA layer with invalid transactions that appear to come from any address, wasting network resources and causing sequencers to pay gas fees for publishing useless data.

## Vulnerability Details

A user submits a transaction to the Movement Network. After a two level validation, the transaction is placed into the mempool, awaiting inclusion.\
Then, the sequencer extracts a batch of transactions from the mempool, arranges them in order, and publishes the batch’s transaction data to the DA service (either L1 or an alternative DA). Afterward, the executor processes these transactions.

Before transactions are added to the mempool, the sequencer performs a two-level prevalidation process:

* Sender Validation – Verifies if the transaction sender is included in a predefined whitelist.
* Signature Validation – Ensures the transaction’s signature is valid by invoking a function from the Aptos codebase.

```rust
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

Whitelisting senders is an arbitrary choice, and it is possible to configure the system to accept transactions from any user without specifying a whitelist. However, in the sequencer implementation, if no whitelist is configured, both validation steps are entirely skipped. This means that when sender whitelisting is not enforced, transaction signature validation is also bypassed. As a result, users can submit transactions with invalid signatures, which will be accepted into the mempool and subsequently published to the DA:

```rust
	/// Batch write blobs.
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
					match prevalidator.prevalidate(transaction).await {
						Ok(prevalidated) => {
							transactions.push(prevalidated.into_inner());
						}
						Err(e) => {
							match e {
								movement_da_light_node_prevalidator::Error::Validation(_) => {
									// discard the transaction
									info!(
										"discarding transaction due to prevalidation error {:?}",
										e
									);
								}
								movement_da_light_node_prevalidator::Error::Internal(e) => {
									return Err(tonic::Status::internal(e.to_string()));
								}
							}
						}
					}
				}
				None => transactions.push(transaction),
			}
		}

		// publish the transactions
		let memseq = self.memseq.clone();
		memseq
			.publish_many(transactions)
			.await
			.map_err(|e| tonic::Status::internal(e.to_string()))?;

		Ok(tonic::Response::new(grpc::BatchWriteResponse { blobs: vec![] }))
	}
```

## Impact Details

Attackers can submit transactions with invalid signatures, and these transactions will be accepted into the mempool and later published to the DA. In the current implementation, the sequencer is responsible for covering the gas fees associated with publishing transactions to the DA. As a result, injecting these junk transactions into the DA imposes a financial loss on the sequencer due to unnecessary gas expenditures.

Furthermore, when this vulnerability is exploited in conjunction with another bug (Report ID 41063), an attacker can submit an excessively large blob request to the light node. This request is eventually published to the DA, significantly increasing the gas fees paid by the sequencer and amplifying the financial impact of the attack.

## Proof of Concept

## Proof of Concept

Detailed explanations inside Vulnerability Details.
