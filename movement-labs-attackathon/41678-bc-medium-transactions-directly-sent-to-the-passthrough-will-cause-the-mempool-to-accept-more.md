# #41678 \[BC-Medium] Transactions directly sent to the passthrough will cause the mempool to accept more transactions than the \`inflight\_limit\`

**Submitted on Mar 17th 2025 at 14:22:44 UTC by @KlosMitSoss for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41678
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

## Brief/Intro

Whenever a block is executed, the number of transactions in flight is decremented by the number of transactions in the block. This way, it is ensured that there are not more than the maximum number of transactions allowed (`inflight_limit`) in the mempool. However, this number can be less than it should be when transactions are directly sent to the passthrough DA light node.

## Vulnerability Details

The number of transactions in flight is incremented every time a new transaction is submitted by calling `transaction_pipe::submit_transaction()`.

```rust
	async fn submit_transaction(
		&mut self,
		transaction: SignedTransaction,
	) -> Result<SubmissionStatus, Error> {
		// Check whether the account is whitelisted
		if !self.is_whitelisted(&transaction.sender())? {
			return Ok((MempoolStatus::new(MempoolStatusCode::TooManyTransactions), None));
		}
                ... ..
				// increment transactions in flight
				{
					let mut transactions_in_flight = self.transactions_in_flight.write().unwrap();
>>					transactions_in_flight.increment(now, 1);
				}
                ... ...
	}
```

<https://github.com/movementlabsxyz/movement/blob/ec71271bbd022e89a1e3e917629b83442ac2e9d4/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L339-L342>

It is ensured that there can never be more than `inflight_limit` transactions in the mempool. Otherwise, the mempool status returns `MempoolIsFull` and new transactions cannot be submitted.

The number of transactions in flight is then decremented when a block is executed.

```rust
	async fn process_block_from_da(
		&mut self,
		response: StreamReadFromHeightResponse,
	) -> anyhow::Result<()> {
        ... ...
		let block: Block = bcs::from_bytes(&block_bytes[..])?;

		// get the transactions
		let transactions_count = block.transactions().len();
		let span = info_span!(target: "movement_timing", "execute_block", block_id = %block.id());
		let commitment =
			self.execute_block_with_retries(block, block_timestamp).instrument(span).await?;

		// decrement the number of transactions in flight on the executor
>>		self.executor.decrement_transactions_in_flight(transactions_count as u64);
        ... ...
	}
```

<https://github.com/movementlabsxyz/movement/blob/ec71271bbd022e89a1e3e917629b83442ac2e9d4/networks/movement/movement-full-node/src/node/tasks/execute\\_settle.rs#L100-L187>

However, if transactions are sent directly to the passthrough DA light node API by calling `batch_write`, the number of transactions in flight will only decrement.

The attack is possible because the full node believes that every transaction has been sent through the full node itself which is not true because the DA light node exposes a public API.

## Impact Details

The number of transactions in flight will be lower than it should be and `transaction_pipe::submit_transaction` allows to submit more transactions than it should. In fact, the number of transactions in flight can always be set to zero such that the mempool accepts an arbitrary number of transactions.

Hence, transactions can be submitted to the mempool beyond set parameters.

## References

The relevant links are provided throughout the report.

## Proof of Concept

## Proof of Concept

1. The attacker calls `passthrough::batch_write` (<https://github.com/movementlabsxyz/movement/blob/ec71271bbd022e89a1e3e917629b83442ac2e9d4/protocol-units/da/movement/protocol/light-node/src/passthrough.rs#L230-L248>) to send transactions directly to the DA light node.
2. `execute_settle::run()` (<https://github.com/movementlabsxyz/movement/blob/ec71271bbd022e89a1e3e917629b83442ac2e9d4/networks/movement/movement-full-node/src/node/tasks/execute\\_settle.rs#L70-L98>) is called which executes the blob.
3. Inside of this function, `process_block_from_da()` is called which decrements the number of transactions in flight (<https://github.com/movementlabsxyz/movement/blob/ec71271bbd022e89a1e3e917629b83442ac2e9d4/networks/movement/movement-full-node/src/node/tasks/execute\\_settle.rs#L154>).
4. However, the number of transactions in flight has never been incremented by that amount. Therefore, the transactions in flight is lower than it should be which makes it possible to submit more transactions than should be allowed.
