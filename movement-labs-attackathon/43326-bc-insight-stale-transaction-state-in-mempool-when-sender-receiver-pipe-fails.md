# #43326 \[BC-Insight] stale transaction state in mempool when sender receiver pipe fails

## #43326 \[BC-Insight] Stale Transaction State in Mempool When Sender/Receiver Pipe Fails

**Submitted on Apr 4th 2025 at 15:59:54 UTC by @Blockian for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43326
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/networks/movement/movement-full-node>
* **Impacts:**

### Description

## Movement Bug Report

### Stale Transaction State in Mempool When Sender/Receiver Pipe Fails

#### Summary

When a transaction is submitted to a node, it is first added to the `core_mempool` and then forwarded to the `transaction_ingress` via an `mpsc::Sender`. However, if the channel send operation fails (e.g., due to a broken pipe, etc...), the transaction remains in the mempool **without any follow-up processing or cleanup**. This results in a stale transaction that cannot be re-submitted or processed properly until the GC occurs.

### Root Cause Analysis

In the `submit_transaction` method:

```rs
async fn submit_transaction(
		&mut self,
		transaction: SignedTransaction,
	) -> Result<SubmissionStatus, Error> {
		// ... (omitted non-relevant code)
		let status = self.core_mempool.add_txn(
			transaction.clone(),
			0,
			sequence_number,
			TimelineState::NonQualified,
			true,
		);

		match status.code {
			MempoolStatusCode::Accepted => {
				// ... (omitted non-relevant code)
				self.transaction_sender
					.send((application_priority, transaction))
					.await
					.map_err(|e| anyhow::anyhow!("Error sending transaction: {:?}", e))?;
				// ...
		}

		// report status
		Ok((status, None))
	}
```

The key issue lies in the fact that the transaction is added to the `core_mempool` before attempting to forward it via the `transaction_sender` and there is no cleanup in case of failure.

If `send` fails, the transaction remains stuck in the mempool. Because the node considers it "already accepted," resubmission of the same transaction wont work.

### Impact

* **Stale Transactions**: Transactions remain stuck in the mempool but are never propagated to the DA or processed by downstream systems.
* **Submission Blockage**: The user cannot re-submit the same transaction (same nonce/sequence number) due to mempool duplication checks.
* **Silent Failures**: The failure mode is silent from the perspective of the user or client — the transaction appears to have been accepted but is effectively discarded.

### Proposed Fixes

**Rollback on Failure**\
If `transaction_sender.send(...)` fails, remove the transaction from the `core_mempool` to ensure the user can retry.

### Proof of Concept

### Proof of Concept (PoC)

1. Modify the `submit_transaction` function to simulate a failure in `transaction_sender.send` (e.g., by manually dropping the receiver or inserting a forced error).
2. Submit a valid transaction.
3. Observe that the transaction is stored in the `core_mempool` but never forwarded to the data availability (DA) layer or any execution pipeline.
4. Attempt to re-submit the transaction with the same sender and sequence number — it will be rejected as a duplicate, despite never being processed.
