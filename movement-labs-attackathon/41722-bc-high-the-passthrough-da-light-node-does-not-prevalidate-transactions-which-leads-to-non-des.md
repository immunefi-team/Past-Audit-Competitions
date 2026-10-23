# #41722 \[BC-High] The passthrough DA light node does not prevalidate transactions which leads to non-deserializable transactions that prevent execution

**Submitted on Mar 17th 2025 at 19:39:34 UTC by @KlosMitSoss for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41722
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

The sequencer DA light node verifies a transaction as valid. However, this prevalidation is not performed by the passthrough DA light node. As a result, there is no guarantee that transactions can be deserialized during execution. Furthermore, it is not ensured that the transaction sender is whitelisted, nor is the signature verified.

## Vulnerability Details

Inside of `sequencer::batch_write`, every transaction is prevalidated which verifies a transaction as a valid transaction.

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

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/light-node/src/sequencer.rs#L397>

Inside of `prevalidator.prevalidate()`, it is validated that the `transaction.sender()` is whitelisted. Furthermore, `AptosTransactionValidator.prevalidate()` is called which validates the transaction to be a properly signed user transaction that can be deserialized from the `transaction.data()`.

```
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
```

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/prevalidator/src/aptos/transaction.rs#L10-L25>

This is done to ensure that transactions can be deserialized such that the deserialization of these transactions during the processing steps before execution does not fail.

```rust
	async fn execute_block(
		&mut self,
		block: Block,
		block_timestamp: u64,
	) -> anyhow::Result<BlockCommitment> {
		let block_id = block.id();
		let block_hash = HashValue::from_slice(block.id())?;

		// get the transactions
		let mut block_transactions = Vec::new();
		let block_metadata = self.executor.build_block_metadata(
			HashValue::sha3_256_of(block_id.as_bytes().as_slice()),
			block_timestamp,
		)?;
		let block_metadata_transaction =
			SignatureVerifiedTransaction::Valid(Transaction::BlockMetadata(block_metadata));
		block_transactions.push(block_metadata_transaction);

		for transaction in block.transactions() {
>>			let signed_transaction: SignedTransaction = bcs::from_bytes(transaction.data())?;
            ... ...
	}
```

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/networks/movement/movement-full-node/src/node/tasks/execute\\_settle.rs#L236>

However, this prevalidation is not done in the passthrough DA light node.

```
	async fn batch_write(
		&self,
		request: tonic::Request<BatchWriteRequest>,
	) -> std::result::Result<tonic::Response<BatchWriteResponse>, tonic::Status> {
		let blobs = request.into_inner().blobs;
		for data in blobs {
			let blob = InnerSignedBlobV1Data::now(data.data)
				.try_to_sign(&self.signer)
				.await
				.map_err(|e| tonic::Status::internal(format!("Failed to sign blob: {}", e)))?;
			self.da
				.submit_blob(blob.into())
				.await
				.map_err(|e| tonic::Status::internal(e.to_string()))?;
		}

		// * We are currently not returning any blobs in the response.
		Ok(tonic::Response::new(BatchWriteResponse { blobs: vec![] }))
	}
```

## Impact Details

The passthrough DA light node allows transactions that do not originate from a whitelisted sender. Furthermore, it does not ensure that transactions can be deserialized.\
As a result, transactions that cannot be deserialized prevent the execution not only of the block containing this transaction but also of all subsequent blocks, as Celestia blobs must be processed in order.

Hence, this issue also causes network processing nodes to process transactions from the mempool beyond set parameters since the DA light nodes are configured with a whitelist.

## References

References are provided throughout the report.

## Proof of Concept

## Proof of Concept

The following steps are needed for this issue to occur:

1. A request is sent to the passthrough DA light node which contains at least one transaction that cannot be deserialized (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/networks/movement/movement-full-node/src/node/tasks/transaction\\_ingress.rs#L104-L111>). The passthrough DA light node submits these transactions without prevalidating them (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/light-node/src/passthrough.rs#L235-L244>).
2. Then the blobs are sent to Celestia and streamed back to the full node in `execute_settle::run()` (<https://github.com/movementlabsxyz/movement/blob/ec71271bbd022e89a1e3e917629b83442ac2e9d4/networks/movement/movement-full-node/src/node/tasks/execute\\_settle.rs#L70-L98>).
3. This function calls `process_block_from_da()` (<https://github.com/movementlabsxyz/movement/blob/ec71271bbd022e89a1e3e917629b83442ac2e9d4/networks/movement/movement-full-node/src/node/tasks/execute\\_settle.rs#L100-L188>). This function calls `execute_block_with_retries()` (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/networks/movement/movement-full-node/src/node/tasks/execute\\_settle.rs#L197-L215>) which then calls `execute_block()` (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/networks/movement/movement-full-node/src/node/tasks/execute\\_settle.rs#L217-L263>).
4. Inside of this function, `bcs::from_bytes()` (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/networks/movement/movement-full-node/src/node/tasks/execute\\_settle.rs#L236>) is called to deserialize the transactions. However, this is not possible as the passthrough does not prevalidate the transactions to ensure that they can be deserialized. As a result, the function returns an error.
