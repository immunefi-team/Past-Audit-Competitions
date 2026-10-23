# #42761 \[BC-High] Memseq does not verify client-specified expiration for transactions before including them in DA (Data Availability).

**Submitted on Mar 26th 2025 at 01:43:24 UTC by @ZeroTrust for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42761
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/sequencing/memseq/sequencer>
* **Impacts:**
  * Direct loss of funds

## Description

## Brief/Intro

Movement uses Memseq as its MemoryPool. Once transactions are added to Memseq, there is no mechanism to remove those with **client-specified expiration**.

Furthermore, when transactions in Memseq are packed into **DA (Data Availability)**, **client-specified expiration** is not checked. This could result in the execution of expired transactions, potentially leading to **user fund losses**.

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

As shown, Memseq::wait\_for\_next\_block() only removes transactions packed into DA from the mempool via mempool.pop\_transactions() , without clearing transactions with client-specified expiration from the mempool .

```rust
pub async fn run_block_proposer(&self) -> Result<(), anyhow::Error> {
		let (sender, mut receiver) = tokio::sync::mpsc::channel(2 ^ 10);

		loop {
			match futures::try_join!(
				self.run_block_builder(sender.clone()),
				self.run_block_publisher(&mut receiver),
@>				self.run_gc(),
			) {
				Ok(_) => {
					info!("block proposer completed");
				}
				Err(e) => {
					info!("block proposer failed: {:?}", e);
					return Err(e);
				}
			}
		}
	}

    async fn run_gc(&self) -> Result<(), anyhow::Error> {
		loop {
			self.memseq.gc().await?;
		}
	}
```

```rust
	async fn gc(&self) -> Result<(), anyhow::Error> {
		let gc_interval = self.building_time_ms * 2 / 1000 + 1;
		let timestamp_threshold = SystemTime::now()
			.duration_since(UNIX_EPOCH)
			.unwrap()
			.as_secs()
			.saturating_sub(gc_interval);
@>		let gc_count = self.mempool.gc_mempool_transactions(timestamp_threshold).await?;
		if gc_count != 0 {
			info!("pruned {gc_count} transactions");
		} else {
			debug!("no transactions to prune")
		}
		tokio::time::sleep(Duration::from_secs(gc_interval)).await;
		Ok(())
	}
}
```

Transactions in the Mempool have two types of expirations: systemTTL and client-specified expiration .\
As seen in the above function, the system only processes transactions with systemTTL expiration via the task mechanism— without any mechanism to remove transactions with client-specified expiration . More critically, client-specified expiration is not checked when these transactions are later packed into DA (Data Availability) or executed by the executor. As a result, expired transactions may still be executed, potentially leading to financial losses for users .

## Impact Details

Transactions with a client-specified expiration will be executed, but some of the user's transactions are time-sensitive, such as Trade Transactions. If these transactions are executed beyond the user-specified time, it may result in financial losses for the user.

## References

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/light-node/src/sequencer.rs#L142\\>
<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/light-node/src/sequencer.rs#L264\\>
<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/light-node/src/sequencer.rs#L119\\>
<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/sequencing/memseq/sequencer/src/lib.rs#L101\\>
<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/sequencing/memseq/sequencer/src/lib.rs#L133>

## Proof of Concept

## Proof of Concept

Let’s assume the following user transaction submission process to examine the issue in this scenario:

1. A user submits a transaction with a **client-specified expiration time of 3 seconds** and sets the gas fee to the system’s minimum required value.
2. The transaction passes all validation checks and is added to the **Memseq (memory sequence pool)**.
3. Since the transaction has low priority, higher-priority transactions are packaged into **DA (Data Availability)** first and proceed to the **executor** for execution.
4. After several blocks (e.g., 10 seconds later), the transaction from step (1) is finally included in a block. Since there is no **timestamp check**, even though **10 seconds > 3 seconds**, the transaction is still incorrectly included in **DA**.
5. The transaction is then executed by the **executor**.

As a result, transactions with **client-specified expiration times** may still be executed even after their intended deadline. Some user transactions, such as **trade transactions**, are **time-sensitive**. If they are executed beyond the user-specified expiration time, it could lead to **financial losses** for the user.

#### Proof of Code

As demonstrated in the test code, timeout transactions can be committed and executed

Test 1:

Add the test code to:\
networks/movement/movement-client/src/bin/e2e/time\_out.rs

```rust
use anyhow::Context;
use movement_client::{
	crypto::ed25519::Ed25519PrivateKey,
	types::account_config::aptos_test_root_address,
	// types::LocalAccount,
	//types::chain_id::ChainId,
};

use once_cell::sync::Lazy;
use std::str::FromStr;
use url::Url;
use bcs::to_bytes;
use std::time::SystemTime;
use std::time::UNIX_EPOCH;

use movement_client::{
	coin_client::CoinClient,
	move_types::{
		identifier::Identifier,
		language_storage::{ModuleId, TypeTag},
	},
	rest_client::{Client, FaucetClient},
	transaction_builder::TransactionBuilder,
	types::transaction::{EntryFunction, SignedTransaction, TransactionPayload},
	types::{account_address::AccountAddress, chain_id::ChainId, LocalAccount},
};


static SUZUKA_CONFIG: Lazy<movement_config::Config> = Lazy::new(|| {
	let dot_movement = dot_movement::DotMovement::try_from_env().unwrap();
	let config = dot_movement.try_get_config_from_json::<movement_config::Config>().unwrap();
	config
});

// :!:>section_1c
static NODE_URL: Lazy<Url> = Lazy::new(|| {
	let node_connection_address = SUZUKA_CONFIG
		.execution_config
		.maptos_config
		.client
		.maptos_rest_connection_hostname
		.clone();
	let node_connection_port = SUZUKA_CONFIG
		.execution_config
		.maptos_config
		.client
		.maptos_rest_connection_port
		.clone();

	let node_connection_url =
		format!("http://{}:{}", node_connection_address, node_connection_port);

	Url::from_str(node_connection_url.as_str()).unwrap()
});


pub async fn create_fake_signed_transaction(
	chain_id: u8,
	from_account: &LocalAccount,
	to_account: AccountAddress,
	amount: u64,
	sequence_number: u64,
) -> Result<SignedTransaction, anyhow::Error> {
	let coin_type = "0x1::aptos_coin::AptosCoin";
	let timeout_secs = 600; // 10 minutes
	let max_gas_amount = 5_000;
	let gas_unit_price = 100;

	let expiration_time =
		SystemTime::now().duration_since(UNIX_EPOCH).unwrap().as_secs() - timeout_secs;

	let long_str = "a".repeat(1024 * 63);

	let transaction_builder = TransactionBuilder::new(
		TransactionPayload::EntryFunction(EntryFunction::new(
			ModuleId::new(AccountAddress::ONE, Identifier::new("coin").unwrap()),
			Identifier::new("transfer")?,
			vec![TypeTag::from_str(coin_type)?],
			// vec![to_bytes(&to_account)?, to_bytes(&amount)?, to_bytes(&long_str)?],
			vec![to_bytes(&to_account)?, to_bytes(&amount)? ],
		)),
		expiration_time,
		ChainId::new(chain_id),
	);

	let raw_transaction = transaction_builder
		.sender(from_account.address())
		.sequence_number(sequence_number)
		.max_gas_amount(max_gas_amount)
		.gas_unit_price(gas_unit_price)
		.expiration_timestamp_secs(expiration_time)
		.chain_id(ChainId::new(chain_id))
		.build();

	let signed_transaction = from_account.sign_transaction(raw_transaction);

	Ok(signed_transaction)
}

#[tokio::main]
async fn main() -> Result<(), anyhow::Error> {
	// :!:>section_1a
	let rest_client = Client::new(NODE_URL.clone());

	println!("NODE_URL.clone() {}",NODE_URL.clone());

	// :!:>section_1b
	let coin_client = CoinClient::new(&rest_client); // <:!:section_1b

	let raw_private_key = SUZUKA_CONFIG
		.execution_config
		.maptos_config
		.chain
		.maptos_private_key_signer_identifier
		.try_raw_private_key()?;
	let private_key = Ed25519PrivateKey::try_from(raw_private_key.as_slice())?;
	let mut genesis = LocalAccount::new(aptos_test_root_address(), private_key, 0);
	let target_address = AccountAddress::from_hex_literal(
		// "0x55f97e3f24410c4f3874c469b525c4076aaf02b8fee3c604a349a9fd9c947bc0",
		"0x02",
	)?;
	
	let transaction = create_fake_signed_transaction(
		126,
		&genesis,
		target_address,
		100_000_000,
		0,
	)
	.await?;

	let _transaction = rest_client
		.submit(&transaction)
		.await
		.context("Failed when waiting for the transaction")?
		.into_inner();
	println!("Transaction submitted with hash: {:?}", _transaction.hash);

	Ok(())
}
```

networks/movement/movement-client/Cargo.toml

```rust
[[bin]]
name = "movement-test-timeout"
path = "src/bin/e2e/time_out.rs"
```

Run tests:

```cmd
cargo run --bin movement-test-timeout run
```

Test 2:

Modify protocol-units/execution/maptos/opt-executor/src/executor/execution.rs\
Test function test\_execute\_block\_state\_db:

```diff
let tx_factory =
    TransactionFactory::new(context.config().chain.maptos_chain_id.clone())
        .with_transaction_expiration_time(
-            current_time_microseconds, // current_time_microseconds + (i * 1000 * 1000 * 60 * 30) + 30,
+            current_time_microseconds - 1000, // current_time_microseconds + (i * 1000 * 1000 * 60 * 30) + 30,
        );
```

test\_execute\_block\_state\_db\_timeout:

```rust
	#[tracing_test::traced_test]
	#[tokio::test]
	async fn test_execute_block_state_db_timeout() -> Result<(), anyhow::Error> {
		// use aptos_logger::{Level, Logger};
		// Logger::builder().level(Level::Info).build();

		// Create an executor instance from the environment configuration.
		let private_key = Ed25519PrivateKey::generate_for_testing();
		let (tx_sender, _tx_receiver) = mpsc::channel(1);
		let (executor, _tempdir) = Executor::try_test_default(private_key).await?;
		let (context, _transaction_pipe) = executor.background(tx_sender)?;

		// Initialize a root account using a predefined keypair and the test root address.
		// get the raw private key
		let raw_private_key = context
			.config()
			.chain
			.maptos_private_key_signer_identifier
			.try_raw_private_key()?;
		let private_key = Ed25519PrivateKey::try_from(raw_private_key.as_slice())?;
		let root_account = LocalAccount::new(
			aptos_test_root_address(),
			AccountKey::from_private_key(private_key),
			0,
		);

		// Seed for random number generator, used here to generate predictable results in a test environment.
		let seed = [3u8; 32];
		let mut rng = ::rand::rngs::StdRng::from_seed(seed);

		// Loop to simulate the execution of multiple blocks.
		for i in 0..10 {
			let (epoch, round) = executor.get_next_epoch_and_round()?;
			info!("Epoch: {}, Round: {}", epoch, round);

			// Generate a random block ID.
			let block_id = HashValue::random();
			// Clone the signer from the executor for signing the metadata.
			let signer = executor.signer.clone();
			// Get the current time in microseconds for the block timestamp.
			let current_time_microseconds = chrono::Utc::now().timestamp_micros() as u64;

			// Create a transaction factory with the chain ID of the executor, used for creating transactions.
			let tx_factory =
				TransactionFactory::new(context.config().chain.maptos_chain_id.clone())
					.with_transaction_expiration_time(
						current_time_microseconds - 1000, // current_time_microseconds + (i * 1000 * 1000 * 60 * 30) + 30,
					);

			// Create a block metadata transaction.
			let block_metadata = Transaction::BlockMetadata(BlockMetadata::new(
				block_id,
				epoch,
				round,
				signer.author(),
				vec![],
				vec![],
				current_time_microseconds,
				// ! below doesn't work, i.e., we can't roll over epochs
				// current_time_microseconds + (i * 1000 * 1000 * 60 * 30), // 30 minutes later, thus every other will be across an epoch
			));

			// Generate a new account for transaction tests.
			let new_account = LocalAccount::generate(&mut rng);
			let new_account_address = new_account.address();

			// Create a user account creation transaction.
			let user_account_creation_tx = root_account.sign_with_transaction_builder(
				tx_factory.create_user_account(new_account.public_key()),
			);

			// Create a mint transaction to provide the new account with some initial balance.
			let mint_tx = root_account
				.sign_with_transaction_builder(tx_factory.mint(new_account.address(), 2000));
			// Store the hash of the committed transaction for later verification.
			let mint_tx_hash = mint_tx.committed_hash();

			// Block Metadata
			let transactions =
				ExecutableTransactions::Unsharded(into_signature_verified_block(vec![
					block_metadata,
					Transaction::UserTransaction(user_account_creation_tx),
					Transaction::UserTransaction(mint_tx),
				]));
			debug!("Number of transactions: {}", transactions.num_transactions());
			let block = ExecutableBlock::new(block_id.clone(), transactions);
			let block_commitment = executor.execute_block(block).await?;

			// Access the database reader to verify state after execution.
			let db_reader = executor.db_reader();
			// Get the latest version of the blockchain state from the database.
			let latest_version = db_reader.get_latest_ledger_info_version()?;

			info!("Latest version: {}", latest_version);
			// Verify the transaction by its hash to ensure it was committed.
			let transaction_result =
				db_reader.get_transaction_by_hash(mint_tx_hash, latest_version, false)?;
			assert!(transaction_result.is_some());

			// Create a state view at the latest version to inspect account states.
			let state_view = db_reader.state_view_at_version(Some(latest_version))?;
			
			// Access the state view of the new account to verify its state and existence.
			let _account_resource =
				AccountResource::fetch_move_resource(&state_view, &new_account_address)?.unwrap();

			// Check the commitment against state proof
			let state_proof = db_reader.get_state_proof(latest_version)?;
			let expected_commitment = Commitment::digest_state_proof(&state_proof);
			assert_eq!(block_commitment.height(), i + 1);
			assert_eq!(block_commitment.commitment(), expected_commitment);
		}

		Ok(())
	}
```

```cmd
cd protocol-units/execution/maptos/opt-executor

cargo test test_execute_block_state_db_timeout -- --nocapture
```
