# #42298 \[BC-Critical] Blocks from Celestia are not executed in order which breaks sequencer logic and application priorities

**Submitted on Mar 22nd 2025 at 15:02:52 UTC by @KlosMitSoss for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42298
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

## Brief/Intro

When blocks are streamed from Celestia before execution, there is no guarantee that they are in the correct order based on the priority of their transactions. This means that blocks could be executed in a different order than they were originally sequenced before being sent to Celestia.

## Vulnerability Details

When the next block is built, transactions are popped from the mempool to be included in the block. Transactions with the highest priority (lowest value) are popped first. After that, the block is assigned a parent block, which determines the order of the blocks.

```
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

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/sequencing/memseq/sequencer/src/lib.rs#L58-L68>

This means that blocks will be sequenced in a specific order, with those containing the highest-priority transactions appearing first. However, there is no guarantee that the blocks streamed from Celestia before execution will maintain this order. As a result, lower-priority transactions may be executed before higher-priority ones.

Simply sending transactions to Celestia in a certain order does not ensure that they are commited to Celestia in that same order. In fact, Celestia does not provide any ordering guarantees and the ordering of the blocks must be ensured when they are fetched from Celestia. That means a block that is streamed from Celestia must only be executed when all parent blocks have been executed.

## Impact Details

Blocks could be executed in a different order than they were originally sequenced. As a result, lower-priority transactions could be executed before higher-priority ones. This means transactions willing to pay a high fee could be delayed by lower-fee transactions.

This delay affects time-sensitive operations and creates uncertainty in the execution order, which can lead to issues when transaction outcomes depend on precise sequencing. The fact that blocks should have an order is obvious given that blocks are connected via their parent field.

The finding is submitted with impact "Causing network processing nodes to process transactions from the mempool beyond set parameters" since the application priorities are assigned by the mempool but full nodes do not execute the transactions according to their application priorities.

## References

Code references are provided throughout the report.

## Proof of Concept

## Proof of Concept

1. A new block is built when `wait_for_next_block()`(<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/sequencing/memseq/sequencer/src/lib.rs#L101-L131>) is called. This function then calls `pop_transactions()` (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/mempool/util/src/lib.rs#L138-L149>).
2. `pop_transactions()` calls `pop_mempool_transaction()` (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/mempool/move-rocks/src/lib.rs#L254-L285>) at the end of its call flow. This function pops the transactions with the highest priority (lowest application\_priority value) first.
3. These transactions are then used to build the next block by calling `build_next_block()` (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/sequencing/memseq/sequencer/src/lib.rs#L58-L67>). This function assigns a `parent_block` to each block which establishes a specific order.
4. These blocks are then submitted to Celestia. Before the different blocks are executed, they are streamed from Celestia by calling `stream_read_from_height()` (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/networks/movement/movement-full-node/src/node/tasks/execute\\_settle.rs#L73-L80>). However, it is not ensured that the received blocks are in the correct order that was established before.
