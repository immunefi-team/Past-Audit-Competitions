# #43307 \[BC-High] Not verifying the signatures upon execution leads to direct loss of funds

**Submitted on Apr 4th 2025 at 13:05:49 UTC by @HollaDieWaldfee for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43307
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/networks/movement/movement-full-node>
* **Impacts:**
  * Direct loss of funds

## Description

## Brief/Intro

Upon execution of a block, every transaction is treated as valid since there is no verification that the signature is valid. Hence, an attacker could send any transaction directly to the DA light node with an invalid signature.

## Vulnerability Details

When submitting a transaction, it is verified that the signature is valid. Therefore, every transaction is treated as a valid transaction with a verified signature after it is submitted. However, an attacker could send transactions directly to the DA light node with an invalid signature. This transaction will then be treated like a valid transaction with a verified signature as can be seen in the code snippet below.

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
			let signed_transaction: SignedTransaction = bcs::from_bytes(transaction.data())?;

			// check if the transaction has already been executed to prevent replays
			if self
				.executor
				.has_executed_transaction_opt(signed_transaction.committed_hash())?
			{
				continue;
			}

@>			let signature_verified_transaction = SignatureVerifiedTransaction::Valid(
				Transaction::UserTransaction(signed_transaction),
			);
			block_transactions.push(signature_verified_transaction);
		}
                // ..
	}
```

On the other hand, Aptos Core verifies the signatures upon execution (reference (3)):

```rust
    async fn prepare_block(
        execute_block_tx: mpsc::UnboundedSender<ExecuteBlockCommand>,
        command: PrepareBlockCommand,
    ) {
            // ..
            let sig_verified_txns: Vec<SignatureVerifiedTransaction> =
                SIG_VERIFY_POOL.install(|| {
                    let num_txns = txns_to_execute.len();
                    txns_to_execute
                        .into_par_iter()
                        .with_min_len(optimal_min_len(num_txns, 32))
                        .map(|t| t.into())
                        .collect::<Vec<_>>()
                });
            // ..
    }
```

Furthermore, Aptos Core only executes valid transactions and discards invalid transactions (reference (4)):

```rust
    pub fn execute_single_transaction(
        &self,
        txn: &SignatureVerifiedTransaction,
        resolver: &impl AptosMoveResolver,
        log_context: &AdapterLogSchema,
    ) -> Result<(VMStatus, VMOutput), VMStatus> {
        assert!(!self.is_simulation, "VM has to be created for execution");

        if let SignatureVerifiedTransaction::Invalid(_) = txn {
            let vm_status = VMStatus::error(StatusCode::INVALID_SIGNATURE, None);
            let discarded_output = discarded_output(vm_status.status_code());
            return Ok((vm_status, discarded_output));
        }
        // ..
    }
```

To fix this issue, there must be verification in execute\_settle.rs#execute\_block() that the signature is valid before executing a block. The verification could look like reference (3 & 4) that is made inside of Aptos Core. The way in which Movement integrates with the Aptos Core block execution is invalid since it lacks the crucial signature validation.

## Impact Details

Since an attacker can send any transaction directly to the DA light node and that transaction is going to be treated like a valid transaction without signature validation, this causes a direct loss of funds.

## References

(1): <https://github.com/movementlabsxyz/movement/blob/ec71271bbd022e89a1e3e917629b83442ac2e9d4/protocol-units/da/movement/protocol/light-node/src/passthrough.rs#L230-L248>

(2): <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/networks/movement/movement-full-node/src/node/tasks/execute\\_settle.rs#L246-L248>

(3): <https://github.com/immunefi-team/attackathon-movement-aptos-core/blob/627b4f9e0b63c33746fa5dae6cd672cbee3d8631/consensus/src/execution\\_pipeline.rs#L129-L136>

(4): <https://github.com/immunefi-team/attackathon-movement-aptos-core/blob/627b4f9e0b63c33746fa5dae6cd672cbee3d8631/aptos-move/aptos-vm/src/aptos\\_vm.rs#L2304-L2308>

## Proof of Concept

## Proof of Concept

1. An attacker sends a transaction directly to the DA light node by calling `passthrough::batch_write()` (reference (1)).
2. This transaction is going to be treated as valid since there is no verification for the signature upon execution (reference (2)). The transaction is treated as a `SignatureVerifiedTransaction::Valid` type without actually verifying the signature.
