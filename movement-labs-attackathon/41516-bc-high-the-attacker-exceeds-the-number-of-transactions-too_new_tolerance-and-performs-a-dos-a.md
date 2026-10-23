# #41516 \[BC-High] The attacker exceeds the number of transactions TOO\_NEW\_TOLERANCE and performs a DoS attack.

**Submitted on Mar 16th 2025 at 05:52:12 UTC by @zhaojie for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41516
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

## Brief/Intro

When used\_sequence\_number is 0, an attacker can commit more transactions than TOO\_NEW\_TOLERANCE, resulting in a DoS attack.

## Vulnerability Details

When the `sequence_number` of the transaction submitted by the user is 0,

The `has_invalid_sequence_number` function does not set `used_sequence_number` +1, which will cause `used_sequence_number` to remain 0 until the transaction is executed:

```rust
	fn has_invalid_sequence_number(
		&self,
		transaction: &SignedTransaction,
	) -> Result<SequenceNumberValidity, Error> {
		// check against the used sequence number pool
->		let used_sequence_number = self
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

		let min_sequence_number = (min_used_sequence_number).max(committed_sequence_number);

		let max_sequence_number = committed_sequence_number + TOO_NEW_TOLERANCE;
        ......
    }

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
		let application_priority = u64::MAX - tx_result.score(); //@audit 攻击者传入错误的 tx_result.score 会导致循环退出?
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

		// has_invalid_sequence_number -> committed_sequence_number
		// sequence_number = committed_sequence_number
->		let sequence_number = match self.has_invalid_sequence_number(&transaction)? {
			SequenceNumberValidity::Valid(sequence_number) => sequence_number,
			SequenceNumberValidity::Invalid(status) => {
				return Ok(status);
			}
		};

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
->				let transaction_sequence_number = transaction.sequence_number();
				self.transaction_sender
					.send((application_priority, transaction))
					.await
					.map_err(|e| anyhow::anyhow!("Error sending transaction: {:?}", e))?;
				// increment transactions in flight
				{
					let mut transactions_in_flight = self.transactions_in_flight.write().unwrap();
					transactions_in_flight.increment(now, 1);
				}
				self.core_mempool.commit_transaction(&sender, sequence_number);

				// update the used sequence number pool
				info!(
					"Setting used sequence number for {:?} to {:?}",
					sender, transaction_sequence_number
				);
->				self.used_sequence_number_pool.set_sequence_number(
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

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L188>

Therefore, until `committed_sequence_number` is not changed before the transaction is executed, the user can commit the transaction repeatedly with 0 `sequence_number`, and `has_invalid_sequence_number` will be validated.

`core_mempool.add_txn` allows adding transactions with the same `sequence_number`, the attacker only needs to modify gas\_price:

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
            if let Some(current_version) = txns.get_mut(&txn_seq_num) {
->                if current_version.txn.payload() != txn.txn.payload() {
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
                        self.index_remove(&txn);
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

        if self.check_is_full_after_eviction(&txn, acc_seq_num) {
            return MempoolStatus::new(MempoolStatusCode::MempoolIsFull).with_message(format!(
                "Mempool is full. Mempool size: {}, Capacity: {}",
                self.system_ttl_index.size(),
                self.capacity,
            ));
        }

        self.clean_committed_transactions(&address, acc_seq_num);

        self.transactions.entry(address).or_default();

        if let Some(txns) = self.transactions.get_mut(&address) {
            // capacity check
            if txns.len() >= self.capacity_per_user {
                return MempoolStatus::new(MempoolStatusCode::TooManyTransactions).with_message(
                    format!(
                        "Mempool over capacity for account. Number of transactions from account: {} Capacity per account: {}",
                        txns.len(),
                        self.capacity_per_user,
                    ),
                );
            }

            // insert into storage and other indexes
            self.system_ttl_index.insert(&txn);
            self.expiration_time_index.insert(&txn);
            self.hash_index
                .insert(txn.get_committed_hash(), (txn.get_sender(), txn_seq_num));
            self.sequence_numbers.insert(txn.get_sender(), acc_seq_num);
            self.size_bytes += txn.get_estimated_bytes();
            txns.insert(txn_seq_num, txn);
            self.track_indices();
        }
        self.process_ready_transactions(&address, acc_seq_num);
        MempoolStatus::new(MempoolStatusCode::Accepted)
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

Thus, an attacker can submit a large number of invalid transactions, exceeding the number limit of TOO\_NEW\_TOLERANCE.

And the cost to the attacker is very low, only a small amount of gas needs to pass the verification of the transaction.

## Impact Details

Anyone can conduct a DoS attack on the network.

## References

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L188>

## Proof of Concept

## Proof of Concept

1. The attacker deposits a small amount of gas into the new account and verifies the transaction.
2. The new account sequence\_number is 0.
3. The attacker increases the gas price by one unit and uses the same sequence\_number(=0) to generate transaction data in batches.
4. The attacker submits more transactions than the TOO\_NEW\_TOLERANCE limit.
