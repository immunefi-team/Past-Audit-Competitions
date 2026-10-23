# #41864 \[BC-Medium] When Memseq selects a transaction from a particular user to include in a block, it does not remove transactions from Memseq that have a sequence\_number less than or equal to the t...

**Submitted on Mar 18th 2025 at 23:16:02 UTC by @ZeroTrust for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41864
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/sequencing/memseq/sequencer>
* **Impacts:**
  * Modification of transaction fees outside of design parameters

## Description

## Brief/Intro

Movement opts to utilize MemSeq as its MemoryPool. However, when MemSeq selects a transaction (tx) from a specific user to be included in a block, it fails to remove transactions from MemSeq that have a sequence\_number less than or equal to the tx.sequence\_number. This oversight can result in unsuccessful transactions being packed into the block, leading to gas losses for the user and wastage of node resources.

## Vulnerability Details

`Memseq::wait_for_next_block()` is utilized to fetch transactions from the mempool for inclusion into a block.

```rust
/// Waits for the next block to be built, either when the block size is reached or the building time expires.
	async fn wait_for_next_block(&self) -> Result<Option<Block>, anyhow::Error> {
		let mut transactions = Vec::with_capacity(self.block_size as usize);

		let now = Instant::now();

		loop {
			let current_block_size = transactions.len() as u32;
			if current_block_size >= self.block_size {
				break;
			}

			let remaining = self.block_size - current_block_size;
@>			let mut transactions_to_add = self.mempool.pop_transactions(remaining as usize).await?;
			transactions.append(&mut transactions_to_add);

			// sleep to yield to other tasks and wait for more transactions
			tokio::task::yield_now().await;

			if now.elapsed().as_millis() as u64 > self.building_time_ms {
				break;
			}
		}

		if transactions.is_empty() {
			Ok(None)
		} else {
			let new_block =
				self.build_next_block(block::BlockMetadata::default(), transactions).await?;
			Ok(Some(new_block))
		}
	}
```

It is evident that the function `Memseq::wait_for_next_block()` only removes transactions from the mempool by using `pop_transactions`, but it does not remove transactions from the same user with smaller `sequence_number` values. As a result, transactions with smaller `sequence_number` can still be included in the block and stored in the DA (Decentralized Archive).

The root cause of this issue lies in the use of `Memseq` as a replacement for `CoreMempool` as the primary `MemoryPool`. Although the `submit_transaction` function calls `core_mempool.commit_transaction()` to remove transactions with smaller `sequence_number` for the same user in the `CoreMempool`, as shown in the code below, this mechanism is not replicated in `Memseq`. This inconsistency leads to the described problem.

```rust
async fn submit_transaction(
		&mut self,
		transaction: SignedTransaction,
	) -> Result<SubmissionStatus, Error> {
		//skip

		// Add the txn for future validation
		debug!("Adding transaction to mempool: {:?} {:?}", transaction, sequence_number);
		let status = self.core_mempool.add_txn(
			transaction.clone(),
			0,
			sequence_number,
			TimelineState::NonQualified,
			true,
		);

		match status.code {
			MempoolStatusCode::Accepted => {
				let now = chrono::Utc::now().timestamp_millis() as u64;
				debug!("Transaction accepted: {:?}", transaction);
				let sender = transaction.sender();
				let transaction_sequence_number = transaction.sequence_number();
				self.transaction_sender
					.send((application_priority, transaction))
					.await
					.map_err(|e| anyhow::anyhow!("Error sending transaction: {:?}", e))?;
				// increment transactions in flight
				{
					let mut transactions_in_flight = self.transactions_in_flight.write().unwrap();
					transactions_in_flight.increment(now, 1);
				}
@>				self.core_mempool.commit_transaction(&sender, sequence_number);

				// update the used sequence number pool
				info!(
					"Setting used sequence number for {:?} to {:?}",
					sender, transaction_sequence_number
				);
				self.used_sequence_number_pool.set_sequence_number(
					&sender,
					transaction_sequence_number,
					now,
				);
			}
			_ => {
				warn!("Transaction not accepted: {:?}", status);
			}
		}

		// report status
		Ok((status, None))
	}
}
```

However, the transactions that are included in the block do not originate from the `CoreMempool`, which leads to the aforementioned issue.\
Below is a specific example illustrating how the problem occurs:

1. **Bob submits a transaction (tx1) with `sequence_number` 1.**
2. **While tx1 is still in the mempool (i.e., `Memseq`), Bob submits another transaction (tx2) with `sequence_number` 2, but with a higher `gas_price`.**
3. **Both transactions pass all validation checks and are added to `Memseq`. However, due to its higher `gas_price`, tx2 is prioritized and included in the block blob first, followed by tx1.**
4. **During the subsequent Move VM execution, tx2 is executed first. As a result, tx1 fails during execution, causing gas losses for the user and wastage of node resources.**

## Impact Details

Since a transaction with a larger sequence\_number from a specific user has already been executed, any subsequent transaction from the same user with a smaller sequence\_number included in the block cannot possibly succeed. However, because it is still included in the DA (Decentralized Archive) block, it will still be executed in the Move VM, inevitably resulting in failure. This redundant execution leads to unnecessary gas consumption for the user and wastes valuable node resources on processing transactions that are guaranteed to fail.

## References

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/sequencing/memseq/sequencer/src/lib.rs#L101>

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L220>

## Proof of Concept

## Proof of Concept

Below is a specific example illustrating how the problem occurs:

1. **Bob submits a transaction (tx1) with `sequence_number` 1.**
2. **While tx1 is still in the mempool (i.e., `Memseq`), Bob submits another transaction (tx2) with `sequence_number` 2, but with a higher `gas_price`.**
3. **Both transactions pass all validation checks and are added to `Memseq`. However, due to its higher `gas_price`, tx2 is prioritized and included in the block blob first, followed by tx1.**
4. **During the subsequent Move VM execution, tx2 is executed first. As a result, tx1 fails during execution, causing gas losses for the user and wastage of node resources.**
