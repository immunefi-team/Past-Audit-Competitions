# #41012 \[BC-Critical] Unintended Chain Split in Movement Full Node

**Submitted on Mar 9th 2025 at 12:55:18 UTC by @yemresaritoprak for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41012
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/networks/movement/movement-full-node>
* **Impacts:**
  * Unintended chain split (network partition)

## Description

## Summary

Movement Full Node can accept multiple blocks at the same height (e.g., `height = H`) without any fork-choice or chain selection logic. This makes the node vulnerable to network partitions or byzantine validators producing conflicting blocks, causing permanent chain splits. Divergent ledger states and potential double-spend scenarios can arise, severely impacting trust and consistency.

## Vulnerability Description

### Core Issue

**Block Identification**\
Movement Full Node only checks if a specific `block_id` has been executed before. It does not check if the height is already occupied by a previously processed block.

**Lack of Chain/Fork Choice**\
In normal blockchain logic, if a second block claims the same height, the node either attempts a reorg or rejects the later block. Movement Full Node fails to do either, leading to two (or more) blocks at the same height.

**Permanent Fork**\
Because no rollback or reorg mechanism exists, once two blocks at height `H` are accepted, the ledger is irreversibly split. Different nodes may record conflicting states, enabling double spends or inconsistent final states across the network.

## Affected Code Snippet

Below is a snippet from `execute_settle.rs` that demonstrates where the node fails to detect the second block at the same height. Comments highlight the lines causing the vulnerability.

```rust
// networks/movement/movement-full-node/src/node/tasks/execute_settle.rs

async fn process_block_from_da(
	&mut self,
	response: StreamReadFromHeightResponse,
) -> anyhow::Result<()> {
	// get the block
	let (block_bytes, block_timestamp, block_id, da_height) = match response
		.blob
		.ok_or(anyhow::anyhow!("No blob in response"))?
		.blob_type
		.ok_or(anyhow::anyhow!("No blob type in response"))?
	{
		blob_response::BlobType::SequencedBlobBlock(blob) => {
			(blob.data, blob.timestamp, blob.blob_id, blob.height)
		}
		blob_response::BlobType::PassedThroughBlob(blob) => {
			(blob.data, blob.timestamp, blob.blob_id, blob.height)
		}
		blob_response::BlobType::Heartbeat(_) => {
			tracing::info!("Receive DA heartbeat");
			// Do nothing.
			return Ok(());
		}
		_ => anyhow::bail!("Invalid blob type"),
	};

	info!(
		block_id = %hex::encode(block_id.clone()),
		da_height = da_height,
		time = block_timestamp,
		"Processing block from DA"
	);

	// (1) The code only checks if this exact block_id was executed before:
	if self.da_db.has_executed_block(block_id.clone()).await? {
		info!("Block already executed: {:#?}. It will be skipped", block_id);
		return Ok(());
	}
	// VULNERABILITY:
	// *No* line checks if `da_height` was already processed.
	// So a second block at the same da_height
	// but different block_id also passes.

	if da_height < 2 {
		anyhow::bail!("Invalid DA height: {:?}", da_height);
	}

	let block: Block = bcs::from_bytes(&block_bytes[..])?;

	// ... Execution is attempted ...
	let span = info_span!(target: "movement_timing", "execute_block", id = ?block_id);
	let commitment =
		self.execute_block_with_retries(block, block_timestamp).instrument(span).await?;

	// The node marks (da_height - 1) as synced, not preventing more blocks at da_height
	self.da_db.set_synced_height(da_height - 1).await?;

	// (2) The block_id is added as 'executed', ignoring that the same height
	// might see another block_id
	self.da_db.add_executed_block(block_id.clone()).await?;

	info!(block_id = ?block_id, "Skipping settlement or proceeding...");

	Ok(())
}
```

* `has_executed_block(block_id.clone())` checks only the `block_id`.
* No section ensures “height = da\_height” is not already used. A second block at the same height proceeds as if new.

### Impact

**Double-Spend & Conflicting Transactions**\
If `BlockA` and `BlockB` each contain transactions spending the same assets, Movement Full Node commits both, effectively doubling the spend.

**Divergent Network States**\
Some Movement Full Nodes might only see `BlockA` or handle them in different orders. Over time, the network’s ledger can no longer converge on a single canonical chain.

**High Severity**\
The inability to revert or choose a single block at each height critically undermines trust in the ledger’s finality.

### Recommendations

**(a) Fork-Choice or Reorg Mechanism**\
Maintain a chain index: if a new block arrives at an already-committed height, decide whether to reorg or reject. A typical BFT-based approach ensures only one block is canonical for each height.

**(b) Height-Based Blocking**\
A simpler fix: once a block at `da_height = H` is accepted, store `height -> block_id`, and refuse any subsequent block claiming the same height.

**(c) Thorough Partition Tests**\
Expand test coverage for multi-validator ephemeral forks. Confirm Movement Full Node forcibly picks or discards one block per height.

### Conclusion

Due to an exclusive check on `block_id` and no enforcement of “one block per height,” Movement Full Node inadvertently allows multiple blocks at the same height. A simple network partition or byzantine validator can produce two valid blocks at height `H`, and the node commits both irreversibly. This threatens network consistency and finality, making the vulnerability **critical severity**.

**Thank you** for reviewing this report. Please reach out if additional details or testing are required.

## Proof of Concept

## Proof of Concept (PoC)

This PoC requires no code modifications; a multi-validator environment plus a short network partition is enough:

**Multi-Validator Celestia**\
Configure two Celestia validators (e.g., `movement-celestia-appd` and `movement-celestia-appd2`) within `docker-compose.multi-local.yml`, each referencing a distinct home directory. Both share the same chain ID so they can produce blocks at the same height.

**Movement Full Node**\
Start Movement Full Node via `just movement-full-node docker-compose multi-local`, ensuring it connects to these two DA validators.

**Short Network Partition**\
Briefly isolate the second validator, for example:

```bash
docker network disconnect <network> movement-celestia-appd2
sleep 10
docker network connect <network> movement-celestia-appd2
```

Each validator might now produce `BlockA` and `BlockB` at the same height `H`.

**Observe Movement Full Node**

After reconnection, Movement Full Node logs typically show:

```
INFO "Processing block from DA" block_id=<A> da_height=H
INFO "Executed block: <A>"
INFO "Processing block from DA" block_id=<B> da_height=H
INFO "Executed block: <B>"
```

Both blocks at height `H` are processed. No reorg or “this height is taken” message.

**Permanent Fork**

The ledger now has two blocks at height = H, leading to indefinite chain splitting.
