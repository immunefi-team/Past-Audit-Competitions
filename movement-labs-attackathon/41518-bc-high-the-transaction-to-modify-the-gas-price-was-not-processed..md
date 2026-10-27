# #41518 \[BC-High] The transaction to modify the gas price was not processed.

**Submitted on Mar 16th 2025 at 06:45:34 UTC by @zhaojie for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41518
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**
  * Direct loss of funds

## Description

The transaction to modify the gas price was not processed.

## Brief/Intro

When the user modifies the gas price, the transaction may be executed repeatedly, or the modification may fail.

## Vulnerability Details

In the aptos sdk, when adding a transaction to mempool, if the transaction already exists and the `sequence_number` is the same, the transaction modified `gas_price` will be processed: delete existing transactions in mempool and add new ones:

core\_mempool.add\_txn -> self.transactions.insert(txn\_info)

```rust
  /// Insert transaction into TransactionStore. Performs validation checks and updates indexes.
    pub(crate) fn insert(&mut self, txn: MempoolTransaction) -> MempoolStatus {
        let address = txn.get_sender();
        let txn_seq_num = txn.sequence_info.transaction_sequence_number;
        let acc_seq_num = txn.sequence_info.account_sequence_number;

        // If the transaction is already in Mempool, we only allow the user to
        // increase the gas unit price to speed up a transaction, but not the max gas.
        //
        // Transactions with all the same inputs (but possibly signed differently) are idempotent
        // since the raw transaction is the same
        if let Some(txns) = self.transactions.get_mut(&address) {
->          if let Some(current_version) = txns.get_mut(&txn_seq_num) {
                if current_version.txn.payload() != txn.txn.payload() {
                    return MempoolStatus::new(MempoolStatusCode::InvalidUpdate).with_message(
                        "Transaction already in mempool with a different payload".to_string(),
                    );
                } else if current_version.txn.expiration_timestamp_secs()
                    != txn.txn.expiration_timestamp_secs()
                {
                    return MempoolStatus::new(MempoolStatusCode::InvalidUpdate).with_message(
                        "Transaction already in mempool with a different expiration timestamp"
                            .to_string(),
                    );
                } else if current_version.txn.max_gas_amount() != txn.txn.max_gas_amount() {
                    return MempoolStatus::new(MempoolStatusCode::InvalidUpdate).with_message(
                        "Transaction already in mempool with a different max gas amount"
                            .to_string(),
                    );
->                } else if current_version.get_gas_price() < txn.get_gas_price() {
                    // Update txn if gas unit price is a larger value than before
                    if let Some(txn) = txns.remove(&txn_seq_num) {
->                      self.index_remove(&txn);
                    };
                    counters::CORE_MEMPOOL_GAS_UPGRADED_TXNS.inc();
                } else if current_version.get_gas_price() > txn.get_gas_price() {
                    return MempoolStatus::new(MempoolStatusCode::InvalidUpdate).with_message(
                        "Transaction already in mempool with a higher gas price".to_string(),
                    );
                } else {
                    // If the transaction is the same, it's an idempotent call
                    // Updating signers is not supported, the previous submission must fail
                    counters::CORE_MEMPOOL_IDEMPOTENT_TXNS.inc();
                    return MempoolStatus::new(MempoolStatusCode::Accepted);
                }
            }
        }
......

  pub fn add_txn(
        &mut self,
        txn: SignedTransaction,
        ranking_score: u64,
        db_sequence_number: u64,
        timeline_state: TimelineState,
        client_submitted: bool,
    ) -> MempoolStatus {
        ......
->      let status = self.transactions.insert(txn_info);
        counters::core_mempool_txn_ranking_score(
            counters::INSERT_LABEL,
            status.code.to_string().as_str(),
            self.transactions.get_bucket(ranking_score),
            ranking_score,
        );
        status
    }

```

The problem is that Movement Network does not do this.

If sequence\_number = 0, duplicate transactions are added to the DA, causing the transaction to be executed repeatedly.\
If sequence\_number > 0, `has_invalid_sequence_number` fails. As a result, the gas price cannot be modified.

```rust
	async fn submit_transaction(
		&mut self,
		transaction: SignedTransaction,
	) -> Result<SubmissionStatus, Error> {
        ......
		// Pre-execute Tx to validate its content.
		// Re-create the validator for each Tx because it uses a frozen version of the ledger.
		let vm_validator = VMValidator::new(Arc::clone(&self.db_reader));
		let tx_result = vm_validator.validate_transaction(transaction.clone())?;
		// invert the application priority with the u64 max minus the score from aptos (which is high to low)
		let application_priority = u64::MAX - tx_result.score();
		match tx_result.status() {
			Some(_) => {
				let ms = MempoolStatus::new(MempoolStatusCode::VmError);
				debug!("Transaction not accepted: {:?}", tx_result.status());
				return Ok((ms, tx_result.status()));
			}
			None => {
				debug!("Transaction accepted by VM: {:?}", transaction);
			}
		}

->		let sequence_number = match self.has_invalid_sequence_number(&transaction)? {
			SequenceNumberValidity::Valid(sequence_number) => sequence_number,
			SequenceNumberValidity::Invalid(status) => {
				return Ok(status);
			}
		};
        .....
    }

	fn has_invalid_sequence_number(
		&self,
		transaction: &SignedTransaction,
	) -> Result<SequenceNumberValidity, Error> {
		// check against the used sequence number pool
		let used_sequence_number = self
			.used_sequence_number_pool
			.get_sequence_number(&transaction.sender())
			.unwrap_or(0);

		// validate against the state view
		let state_view = self.db_reader.latest_state_checkpoint_view().map_err(|e| {
			Error::InternalError(format!("Failed to get latest state view: {:?}", e))
		})?;

		// this checks that the sequence number is too old or too new
		let committed_sequence_number =
			vm_validator::get_account_sequence_number(&state_view, transaction.sender())?;

		debug!(
			"Used sequence number: {:?} Committed sequence number: {:?}",
			used_sequence_number, committed_sequence_number
		);
		let min_used_sequence_number =
->			if used_sequence_number > 0 { used_sequence_number + 1 } else { 0 };

->		let min_sequence_number = (min_used_sequence_number).max(committed_sequence_number);

		let max_sequence_number = committed_sequence_number + TOO_NEW_TOLERANCE;

		info!(
			"min_sequence_number: {:?} max_sequence_number: {:?} transaction_sequence_number {:?}",
			min_sequence_number,
			max_sequence_number,
			transaction.sequence_number()
		);

		if transaction.sequence_number() < min_sequence_number {
			info!("Transaction sequence number too old: {:?}", transaction.sequence_number());
			return Ok(SequenceNumberValidity::Invalid((
				MempoolStatus::new(MempoolStatusCode::InvalidSeqNumber),
				Some(DiscardedVMStatus::SEQUENCE_NUMBER_TOO_OLD),
			)));
		}

		if transaction.sequence_number() > max_sequence_number {
			info!("Transaction sequence number too new: {:?}", transaction.sequence_number());
			return Ok(SequenceNumberValidity::Invalid((
				MempoolStatus::new(MempoolStatusCode::InvalidSeqNumber),
				Some(DiscardedVMStatus::SEQUENCE_NUMBER_TOO_NEW),
			)));
		}

		Ok(SequenceNumberValidity::Valid(committed_sequence_number))
	}

```

When sequence\_number = 0, If `add_txn` returns `Accepted`, the transaction will be submitted to the DA without processing the transaction modifying gas\_price:

```rust
async fn submit_transaction(
		&mut self,
		transaction: SignedTransaction,
	) -> Result<SubmissionStatus, Error> {
        ......
->        let status = self.core_mempool.add_txn(
			transaction.clone(),
			0,
			sequence_number,
			TimelineState::NonQualified,
			true,
		);

->		match status.code {
			MempoolStatusCode::Accepted => {
				let now = chrono::Utc::now().timestamp_millis() as u64;
				debug!("Transaction accepted: {:?}", transaction);
				let sender = transaction.sender();
				let transaction_sequence_number = transaction.sequence_number();
->				self.transaction_sender
					.send((application_priority, transaction))
					.await
					.map_err(|e| anyhow::anyhow!("Error sending transaction: {:?}", e))?;
        ......
    }
```

Since the new transaction modifies gas\_price, a different `transaction.id` is generated, and sequence\_number duplicate transactions can be written to DA:

```rust
impl Transaction {
	pub fn new(data: Vec<u8>, application_priority: u64, sequence_number: u64) -> Self {
		let mut hasher = blake3::Hasher::new();
		hasher.update(&data);
		hasher.update(&sequence_number.to_le_bytes());
->		let id = Id(hasher.finalize().into());
		Self { data, sequence_number, application_priority, id }
	}

	pub fn id(&self) -> Id {
		self.id
	}
    ......
}

fn construct_mempool_transaction_key(transaction: &MempoolTransaction) -> Result<String, Error> {
	// Pre-allocate a string with the required capacity
	let mut key = String::with_capacity(32 + 1 + 32 + 1 + 32 + 1 + 32);
	// Write key components. The numbers are zero-padded to 32 characters.
	key.write_fmt(format_args!(
		"{:032}:{:032}:{:032}:{}",
		transaction.transaction.application_priority(),
		transaction.timestamp,
		transaction.transaction.sequence_number(),
->		transaction.transaction.id(),
	))
	.map_err(|_| Error::msg("Error writing mempool transaction key"))?;
	Ok(key)
}

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

			// Add the transactions and update the lookup table atomically in a single write batch.
			// https://github.com/movementlabsxyz/movement/issues/322

			let mut batch = WriteBatch::default();

			for transaction in transactions {
->				if Self::internal_has_mempool_transaction(&db, transaction.transaction.id())? {
					continue;
				}

				let serialized_transaction = bcs::to_bytes(&transaction)?;
				let key = construct_mempool_transaction_key(&transaction)?;
				batch.put_cf(&mempool_transactions_cf_handle, &key, &serialized_transaction);
				batch.put_cf(
					&transaction_lookups_cf_handle,
					transaction.transaction.id().to_vec(),
					&key,
				);
			}

			db.write(batch)?;

			Ok::<(), Error>(())
		})
		.await??;
		Ok(())
	}

```

Therefore, the user's transaction will be executed repeatedly.

## Impact Details

Transactions are executed repeatedly, resulting in the loss of user funds.

## References

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L220>

<https://github.com/aptos-labs/aptos-core/blob/308d59ec2e7d9c3937c8b6b4fca6dd7e97fd3196/mempool/src/core\\_mempool/transaction\\_store.rs#L252-L257>

## Proof of Concept

## Proof of Concept

1. Alice is a new user sequence\_number = 0.
2. Alice transferred 1000 usdc to Bob and successfully submitted the transaction.
3. Alice wants to speed up the execution of the transaction, so she changes the gas price.
4. core\_mempool deletes the old transaction and adds the new transaction and returns success.
5. However, the request is not processed in TransactionPipe.submit\_transaction, and the duplicate transaction is written to the DA.
6. The transaction is executed Alice's balance is deducted twice and Bob receives 2000 usdc.
