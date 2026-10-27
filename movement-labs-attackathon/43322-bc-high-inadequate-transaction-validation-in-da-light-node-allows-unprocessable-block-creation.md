# #43322 \[BC-High] inadequate transaction validation in da light node allows unprocessable block creation

## #43322 \[BC-High] Inadequate Transaction Validation in DA Light Node Allows Unprocessable Block Creation

**Submitted on Apr 4th 2025 at 15:50:51 UTC by @Blockian for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43322
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/protocol/light-node>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

### Description

## Movement Bug Report

### Inadequate Transaction Validation in DA Light Node Allows Unprocessable Block Creation

#### Summary

The DA Light Node does not properly validate incoming transactions, which allows a malicious node to inject malformed data. If such a corrupt transaction is included in a block, it causes block execution to fail, thereby halting the processing of all other valid transactions in the same block.

### Root Cause Analysis

When a node processes a block, it calls the `execute_block` function:

```rs
	async fn execute_block(
		&mut self,
		block: Block,
		block_timestamp: u64,
	) -> anyhow::Result<BlockCommitment> {
		// ... (omitted non-relevant code)

		for transaction in block.transactions() {
			let signed_transaction: SignedTransaction = bcs::from_bytes(transaction.data())?;
        
        // ... (omitted non-relevant code)

        }
    }
```

If `transaction.data()` is not a valid `SignedTransaction`, the `bcs::from_bytes` call will return an error, which halts block execution.

Although blocks are sourced from the DA Light Node (assumed to be honest and under Movement's control), there is a validation gap during `batch_write`, which publishes transactions to the DA:

```rs
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
				// ... (omitted non-relevant code)
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

In the absence of a `prevalidator`, transactions are published without verifying that `transaction.data` is a valid `SignedTransaction`. The only check performed ensures it's a valid `Vec<u8>`, which is insufficient to guarantee executable block integrity.

### Impact

An attacker can exploit this validation gap by submitting a malformed transaction payload to the DA Light Node.

When an honest node subsequently pulls and attempts to execute a block containing this transaction, it will fail during deserialization, resulting in:

* Complete failure to process the block.
* Potential denial-of-service (DoS) vectors if such malformed blocks are propagated repeatedly.
* Loss of liveness in the network due to execution halts on otherwise valid blocks.

### Proposed Fixes

Introduce stricter validation logic in `batch_write` to ensure that all submitted transactions conform to the `SignedTransaction` format before being published.

### Proof of Concept

### Proof of Concept

1. Start the DA Light Node.
2. Create a malformed transaction with the following arbitrary values:
   * `data` = random `Vec<u8>`
   * `application_priority` = random `u64`
   * `sequence_number` = random `u64`
   * `id` = random `[u8; 32]`
3. Send a gRPC request to the Light Node's `write_batch` endpoint with the malformed transaction.

```rs
// Example: Randomly constructed Transaction (pseudo-code)
let corrupt_transaction = Transaction {
	data: vec![1, 255, 42, 0, 0, 100, 200],
	application_priority: 1,
	sequence_number: 1,
	id: [0u8; 32],
};
```
