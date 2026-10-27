# #43288 \[BC-Critical] Attackers could force Nodes to process TraAttackers could force Nodes to process Transactions in wrong order, by attacking moveRocks/sequencing implementation

**Submitted on Apr 4th 2025 at 10:38:09 UTC by @Berserk for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43288
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/sequencing/memseq/sequencer>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

## Description

## Brief/Intro

There is a vulnerability in the sequencer implementation that allows attackers to force nodes to process transactions from the same account in a wrong sequence number order.\
(e.g., 3 transactions from Account A: seqnr 1, seqnr 2, seqnr 3. Attackers will force nodes to process tx with seq 3 first, seq 2 second, and seqnr 1 last, leading to all transactions failing (other than seqnr 1 of course))\
The mempool allows 32 transactions per account (by abusing this, attackers could force nodes to process at least 31 invalid transactions per account at a time at no cost)

## Vulnerability Details

### Root cause brief

Ordering of transactions in MoveRocks/RocksDB prioritizes the gas\_unit\_price first and doesn't **differentiate between accounts**. So if we have 2 transactions in rocks db from the **same account**:

* Transaction 1: seqnr 1 / gas unit price 100
* Transaction 2: seqnr 2 / gas unit price 150

When the sequencer tries to build the block, it will put Transaction 2 first and then Transaction 1 because Transaction 2 has a bigger gas unit price.\
\=> This will mess up the order of transactions as they should be executed in correct ascending seqnr order (so in this case Transaction 2 will fail because seqnr 2 > account\_seqnr(0) + 1)

## Detailed description of implementation

The sequencer relies on a newly developed mempool implementation that relies on rocks DB. when recieving transactions from the full node through batch\_write, the sequencer will insert the batch in the database and will construct for each one of them a key to allow for ordering/prioritizing.

`Source:protocol-units/mempool/move-rocks/src/lib.rs#L22-L36`

```rust
fn construct_mempool_transaction_key(transaction: &MempoolTransaction) -> Result<String, Error> {
	// Pre-allocate a string with the required capacity
	let mut key = String::with_capacity(32 + 1 + 32 + 1 + 32 + 1 + 32);
	// Write key components. The numbers are zero-padded to 32 characters.
	key.write_fmt(format_args!(
		"{:032}:{:032}:{:032}:{}",
@>		transaction.transaction.application_priority(),
		transaction.timestamp,
		transaction.transaction.sequence_number(),
		transaction.transaction.id(),
	))
	.map_err(|_| Error::msg("Error writing mempool transaction key"))?;
	Ok(key)
}
```

As we can see the first part of the key used contains the transaction application priority, which is `U64::max - transaction.gas_unit_price`

When building a new block to be submitted to DA the sequencer will pop transactions from the RockDB database:

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
@>>			let mut transactions_to_add = self.mempool.pop_transactions(remaining as usize).await?;
			transactions.append(&mut transactions_to_add);
---
```

The order by which those transactions stored in the db wil be poped is defiend by the key constructed.

1. gas\_unit\_price (highest first)
2. addition to rocksdb timestamp (oldest first)
3. sequence nr (lowest first)
4. transaction id

Meaning if two transactions from same account if they have a differing `gas_unit_price` we will order them first based on the `gas_unit_price` and not the `sequence_nr`

## Impact Details

* Force network processing nodes to process transactions in the wrong sequence number order
* DoS network by flooding the network with up to 31 invalid transactions per account at a time at no cost

## References

* `Source:protocol-units/mempool/move-rocks/src/lib.rs#L22-L36`
* `Source:protocol-units/sequencing/memseq/sequencer/src/lib.rs#L100-L122`

## Proof of Concept

## Proof of Concept

### Attack Scenario

To demonstrate the vulenrabilty we will create a batch of 4 transactions and submit them through the rpc.\
The batch contains the following transactions (valid transfer transactions):

* Tx1: sender: Account A, seqnr 0, gas\_unit\_price 100
* Tx2: sender: Account A, seqnr 1, gas\_unit\_price 110
* Tx3: sender: Account A, seqnr 2, gas\_unit\_price 120
* Tx4: sender: Account A, seqnr 3, gas\_unit\_price 130

All of the transactions provided are valid transactions and the order is correct when submitting it through the rpc.

However after exectuting the 4 transactions we will notice from the fullnode logs that the order in which the transactions are executed

1. Tx4, Result: Discard(SEQUENCE\_NUMBER\_TOO\_NEW)
2. Tx3, Result: Discard(SEQUENCE\_NUMBER\_TOO\_NEW)
3. Tx2, Result: Discard(SEQUENCE\_NUMBER\_TOO\_NEW)
4. Tx1, Result: Keep(Success)

### Coded Poc

* main.rs: <https://gist.github.com/aliX40/92e94f9713d17ab22d5801958003f7a2>
* cargo.toml: <https://gist.github.com/aliX40/1257e284afc4b01f5f5430d5e37d0a9b>
* full node log: <https://gist.github.com/aliX40/72da56e976a6aeabbe3d42860c43d0f0>

By executing the main.rs script the following result was printed:

```log
     Running `/root/attack/attackathon-movement/target/debug/transaction-tester`
2025-04-04T10:26:44.814504Z  INFO transaction_tester: Initializing transaction test
2025-04-04T10:26:44.871918Z  INFO transaction_tester: Account address: 0x879daa9a9dda13c4722df4930114341c2918ffd44ee2764210bd2118eb1d981d
2025-04-04T10:26:44.872018Z  INFO transaction_tester: Auth key: 879daa9a9dda13c4722df4930114341c2918ffd44ee2764210bd2118eb1d981d
2025-04-04T10:26:44.872086Z  INFO transaction_tester: Public key: 80fc94ab796101eef9d3b6d46b7aa2da9c7ffdea0655b0c48c901d765f01bb12
2025-04-04T10:26:55.390127Z  INFO transaction_tester: Account address: 0xf12e7acff612f85187b7c8444de00ea716a11b6f4c273b31bec29e335e558af1
2025-04-04T10:26:55.390213Z  INFO transaction_tester: Auth key: f12e7acff612f85187b7c8444de00ea716a11b6f4c273b31bec29e335e558af1
2025-04-04T10:26:55.390241Z  INFO transaction_tester: Public key: fbd4b4439c2db86073f8d20994d5cbbe3e0db78bdc98adde02cb1b62fdd9fc71
Balance Before: 10000700
2025-04-04T10:27:24.304391Z  INFO transaction_tester: Creating test transactions
2025-04-04T10:27:24.942662Z  INFO transaction_tester: Batch of 4 transactions submitted successfully
Response: Response { inner: TransactionsBatchSubmissionResult { transaction_failures: [] }, state: State { chain_id: 27, epoch: 70082, version: 1014122, timestamp_usecs: 1743762417361210, oldest_ledger_version: 0, oldest_block_height: 0, block_height: 70082, cursor: None } }
Balance After: 9999200
2025-04-04T10:27:39.972507Z  INFO transaction_tester: Transaction test completed successfully
```

To check the execution result of the 4 submitted transactions we need to grep for `compute_status_for_input_txns` in the full node logs:

```log
compute_status_for_input_txns: [Keep(Success), // system transaction
Discard(SEQUENCE_NUMBER_TOO_NEW), // tx4 dropped
Discard(SEQUENCE_NUMBER_TOO_NEW), // tx3 dropped
Discard(SEQUENCE_NUMBER_TOO_NEW), // tx2 dropped
Keep(Success)] // tx1 accepted
```

To check the full logs of the full node, please see <https://gist.github.com/aliX40/72da56e976a6aeabbe3d42860c43d0f0>
