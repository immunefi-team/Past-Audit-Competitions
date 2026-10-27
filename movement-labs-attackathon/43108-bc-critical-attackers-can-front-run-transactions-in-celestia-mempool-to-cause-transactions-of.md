# #43108 \[BC-Critical] attackers can front run transactions in celestia mempool to cause transactions of many users revert unexpectedly&#x20;

## #43108 \[BC-Critical] Attackers can front-run transactions in celestia mempool to cause transactions of many users revert unexpectedly

**Submitted on Apr 2nd 2025 at 04:39:16 UTC by @perseverance for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43108
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/>
* **Impacts:**
  * Direct loss of funds
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

### Description

## Short summary

Attackers can front-run transactions in Celestia mempool to manipulate the blob height to manipulate order of execution of transactions. This will cause transactions of many users of Movement revert unexpectedly in some cases.

### Background Information

#### Lifecycle of a transaction

The transactions passed validation from the mempool ==> included in the mempool ==> submit (or batch\_write) to DA Lightnode

Then the transactions are included in blocks and sent from DA Lightnode to Celestia.\
Code is here: <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/sequencing/memseq/sequencer/src/lib.rs#L101-L131>

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
			let mut transactions_to_add = self.mempool.pop_transactions(remaining as usize).await?;
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

So this function `wait_for_next_block` batch the transactions to build the block the DA Lightnode batch maximum `block_size` transactions to a block and build `blob` to send to Celestia.

The `block_size` can be 2048 transactions as default value.

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/sequencing/memseq/util/src/lib.rs#L22>

```rust
/// The configuration for the MemSeq sequencer
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
pub struct Config {
	/// The chain id of the sequencer
	#[serde(default = "Config::default_sequencer_chain_id")]
	pub sequencer_chain_id: Option<String>,

	/// The path to the sequencer database
	#[serde(default = "Config::default_sequencer_database_path")]
	pub sequencer_database_path: Option<String>,

	/// The memseq build time for the block
	#[serde(default = "default_memseq_build_time")]
	pub memseq_build_time: u64,

	/// The memseq max block size
	#[serde(default = "default_memseq_max_block_size")]
	pub memseq_max_block_size: u32,
}

env_default!(default_memseq_build_time, "MEMSEQ_BUILD_TIME", u64, 1000); // @audit default memseq build time  = 1000 ms = 1 second 

env_default!(default_memseq_max_block_size, "MEMSEQ_MAX_BLOCK_SIZE", u32, 2048); // @audit default max block size is 2048 

impl Default for Config {
	fn default() -> Self {
		Config {
			sequencer_chain_id: Config::default_sequencer_chain_id(),
			sequencer_database_path: Config::default_sequencer_database_path(),
			memseq_build_time: default_memseq_build_time(),
			memseq_max_block_size: default_memseq_max_block_size(),
		}
	}
}

```

**The blob will be signed and pay for gas by DA Lightnode and send to Celestia.**

So the DA Lightnode now can work in 2 modes: blocky or sequencer mode:

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/light-node/README.md#L1-L6>

If the DA Lightnode works in `Blocky` mode, then block of transactions received from mempool are signed to create Blob data and send to Celestia.

If DA Lightnode works in `sequencer` mode, then transactions are written to Mempool RockDB and then create blocks and then send to Celestia.

The Blob Data sent to Celestia will have some data:

1. Celestia Namespace
2. Blob.Data
3. Blob.timestamp
4. Blob Sigature
5. Blob.id (computed from blob.data and timestamp)

DA Lightnode will call Celestia RPC Client to `blob_submit` blob to Celestia.

Please note that this will call the Celestia Rust RPC Client (lumina) to submit RPC to Celestia Light Node. Then "Celestia Light node" will sign for the transactions and pay from the "DA Lightnode" account to broadcast to Celestia Blockchain.

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/providers/celestia/src/da/mod.rs#L36-L52>

```rust
/// Creates a new signed blob instance with the provided DaBlob data.
	pub fn create_new_celestia_blob(&self, data: DaBlob<C>) -> Result<CelestiaBlob, anyhow::Error> {
		// create the celestia blob
		CelestiaDaBlob(data.into(), self.celestia_namespace.clone()).try_into()
	}

	/// Submits a CelestiaBlob to the Celestia node.
	pub async fn submit_celestia_blob(&self, blob: CelestiaBlob) -> Result<u64, anyhow::Error> {
		let config = TxConfig::default();
		// config.with_gas(2);
		let height = self.default_client.blob_submit(&[blob], config).await.map_err(|e| { // @audit Celestia RPC Client
			error!(error = %e, "failed to submit the blob");
			anyhow::anyhow!("Failed submitting the blob: {}", e)
		})?;

		Ok(height)
	}
```

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/Cargo.toml#L231>

```toml
celestia-rpc = { git = "https://github.com/eigerco/lumina", rev = "c6e5b7f5e3a3040bce4262fe5fba5c21a2637b5" }   #{ version = "0.7.0" }
```

#### From Celestia to Validator Executor

The DA Lightnode subscribe to Celestia and read Blob data from Celestia. After some verification, blob will be executed.

This blob is read in StreamReadFromHeightResponse mechanism.

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/networks/movement/movement-full-node/src/node/tasks/execute\\_settle.rs#L100-L154>

```rust
async fn process_block_from_da(
		&mut self,
		response: StreamReadFromHeightResponse, // @audit notice this read from StreamReadFromHeightResponse 
	) -> anyhow::Result<()> {
		// SNIP

		// get the transactions
		let transactions_count = block.transactions().len();
		let span = info_span!(target: "movement_timing", "execute_block", id = ?block_id);
		let commitment =
			self.execute_block_with_retries(block, block_timestamp).instrument(span).await?;

		// decrement the number of transactions in flight on the executor
		self.executor.decrement_transactions_in_flight(transactions_count as u64);

        // SNIP
```

For more info, please read <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/light-node/src/passthrough.rs#L126>

### Vulnerability Details and Attack Scenario

So notice that transactions from users are included in mempool and then are included in different blobs. Blobs are submited to Celestia Mempool. When receive blobs from Celestia, there is no mechanism to verify the order of blobs.

Celestia makes use of a standard gas-priced prioritized mempool. By default, transactions with gas prices higher than that of other transactions in the mempool will be prioritized by validators. (reference: Fee market and mempool <https://docs.celestia.org/how-to-guides/submit-data#fee-market-and-mempool>)

Suppose that Alice submit 2 transactions

Transaction\_1: timestamp\_1 , sequence number = 1 => Included in Blob\_1

Transaction\_2: timestampt\_2 > timestamp\_1 , sequencer number = 2 => Included in Blob\_2

Now in normal condition:

Blob\_1 are submited to Celestia first, then executed first. Transaction\_1 will pass\
Blob\_2 are submitted after Blob\_1, then executed later. Transaction\_2 will pass.

It is normal and expected.

#### Attack Scenario

Suppose that Movement configure block build time is quick, for example 1 seconds with 2048 transactions in 1 block

Both Blob\_1 and Blob\_2 are in the Celestia mempool waiting for including the next block

This is likely happened more when the Celestia network is in high trafic with a lot of transactions in the mempool waiting for including the next block.\
When the Movement network is busy. There are a lot of transactions in the mempool of Movement.

**So Blob\_1 and Blob\_2 are in Celestia Mempool waiting for being included in Celestia Block.**

An attacker can execute front-run attack and submit Blob\_2 with higher gas price.

So Blob\_2 will be included first and then executed first.\
Blob\_1 will be executed later.

Since transaction\_2 of Alice will be executed first, it will fail, because of wrong sequence number.

**Since 1 blob can contain 2048 transactions so there is high possibility that changing blob orders can cause transactions of many users to revert.**

#### Cost of attack analysis

Cost of submitting blobs to Celestia

So take some example transactions in celestia, <https://celenium.io/tx/9b00e984362bfd0db1b67906608829cf3d9d6a6994e36178bc2de2a9781f9044?tab=messages>

The cost of gas on Celestia is 0.034763 for 1019 Bytes\
\=> TIA Price = 3.66 ( <https://coinmarketcap.com/vi/currencies/celestia/)\\>
\=> Cost for 1 transaction: 0.05 \* 3.65 = 0.183 USD\
The maximum size of a blob is 2 MB.

So the data probably can be bigger and cost more gas, but let's take 0.2 USD as sample for this.

So for this one attack, the attacker need to spend 0.2 USD.

### Severity Assessment

Bug Severity: High

Impact:

* Lost of user fund: If the Movement enable charge gas fee for failed transactions, then users can lost money for failed transactions
* Causing users transactions revert unexpectedly
* Causing network processing nodes to process transactions from the mempool beyond set parameters

**Likelihood:**

* High as No special privileges required
* Can be executed by any attacker with very cheap cost of attack. The cost of an attack is just 0.2 USD

## Recommendation

There should be some mechanism to control the blob sent to Celestia and receive to be in right order. Should not rely only on Celestia block height.

### Proof of Concept

## Proof of concept

The following attack scenario demonstrate the bug.

Step 0: An attacker monitor the precondition for the attack.

When the Celestia network is in high trafic with a lot of transactions.\
When the Movement network is busy. There are a lot of transactions in the mempool.

Attacker monitors blob submission from Movemnet DA Lightnode to Celestia Namespace.

Suppose there are 2 blobs in Celestia Mempool: Blob\_1, Blob\_2

Step 1: Attacker take Blob\_2 and submit it with higher gas to make it included first .

On Movement Execution node. Blob\_2 will be executed before Blob\_1. So transactions of many users can be reverted due to wrong sequence number

The below attached sequence diagram help to visualize the attack:
