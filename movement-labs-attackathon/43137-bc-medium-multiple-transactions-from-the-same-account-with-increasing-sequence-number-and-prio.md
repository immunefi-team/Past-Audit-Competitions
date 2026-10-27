# #43137 \[BC-Medium] Multiple Transactions from the same account with increasing sequence number and priorities will be sorted incorrectly in the block causing some to fail

**Submitted on Apr 2nd 2025 at 16:45:06 UTC by @niroh for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43137
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/sequencing/memseq/sequencer>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

## Description

## Brief/Intro

Multiple Transactions from the same account with increasing sequence number and priorities will be sorted in the wrong order in terms of seq num, causing some to fail during block execution

## Vulnerability Details

Transaction sorting within a block is done by the memseq when building a block. The final sorting is does when in build\_next\_block by entering the txs to a BTreeSet as can be seen here:

```rust
async fn build_next_block(
    &self,
    metadata: block::BlockMetadata,
    transactions: Vec<Transaction>,
) -> Result<Block, anyhow::Error> {
    let mut parent_block = self.parent_block.write().await;
    let new_block = Block::new(metadata, *parent_block, BTreeSet::from_iter(transactions));
    *parent_block = new_block.id();
    Ok(new_block)
}
```

BTreeSet sorts the transactions based on their Ord implementation, which sorts them by priority first, then by timestamp and then by sequence number (within transaction.cmp):

```rust
impl Ord for MempoolTransaction {
	fn cmp(&self, other: &Self) -> Ordering {
		// First, compare the application priority
		// Note: this also happens again in the inner transaction comparison, but the priority should come first both in the [MempoolTransaction] and in the [Transaction] by itself.
		match self
			.transaction
			.application_priority()
			.cmp(&other.transaction.application_priority())
		{
			Ordering::Equal => {}
			non_equal => return non_equal,
		}

		// Then, compare by timestamps
		match self.timestamp.cmp(&other.timestamp) {
			Ordering::Equal => {}
			non_equal => return non_equal,
		}

		// If timestamps are equal, then compare by transaction on the whole
		self.transaction.cmp(&other.transaction)
	}
}

```

This means that if multiple transactions from the same account are in the block where the ones with lower sequence number have higher priority, they will be sorted against their sequence order, causing some of them to fail execution when the block is picked up by the full node for execution.

## Impact Details

Transactions that should have succeeded had they been executed in the correct order, will fail.

## Proof of Concept

## Proof of Concept

1. Account A with sequence number 100 sends two transactions: Tx1 - sequence number 100 gas unit price: 100, Tx2 - sequence number 101 gas unit price: 200.
2. The two transactions end up getting fetched by the sequencer from rocket db for block building using mempool.pop\_transactions.
3. The sequencer sends the transactions to build\_next block, adding them as a BTreeSet which sorts them based on their Ord implementation (priority first, seqnum after)
4. When the block is picked up by the full node for execution, the higher priority higher seqnum transaction will be first in order, causing it to fail execution.
