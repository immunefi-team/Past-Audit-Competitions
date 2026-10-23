# #41686 \[BC-High] The passthrough DA light node streams transactions instead of blocks which means that the block cannot be deserialized

**Submitted on Mar 17th 2025 at 14:52:18 UTC by @KlosMitSoss for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41686
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

The passthrough DA light node does not stream blocks, and therefore full node execution fails when it tries to deserialize blobs as blocks.

## Vulnerability Details

Whenever a full node tries to execute a block, the block is deserialized from the `block_bytes`.

```rust
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
			// To allow for DA migrations we accept both sequenced and passed through blobs
			blob_response::BlobType::SequencedBlobBlock(blob) => {
				(blob.data, blob.timestamp, blob.blob_id, blob.height)
			}
			// To allow for DA migrations we accept both sequenced and passed through blobs
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

		// check if the block has already been executed
		if self.da_db.has_executed_block(block_id.clone()).await? {
			info!("Block already executed: {:#?}. It will be skipped", block_id);
			return Ok(());
		}

		// the da height must be greater than 1
		if da_height < 2 {
			anyhow::bail!("Invalid DA height: {:?}", da_height);
		}

>>		let block: Block = bcs::from_bytes(&block_bytes[..])?;
        ... ...
	}
```

These bytes are obtained from the DA light node blobs that have been streamed by the DA light node. However, the passthrough DA light node does not serialize blocks but single transactions which means that any blob that is sent by the passthrough cannot be deserialized.

The passthrough DA light node processes blobs without doing anything with them so the blobs are just transactions.

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/networks/movement/movement-full-node/src/node/tasks/transaction\\_ingress.rs#L104-L111>

In contrast, the sequencer DA light node builds blocks from the transactions that it receives which means that the blobs can be deserialized as blocks.

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/light-node/src/sequencer.rs#L165-L173>

The README (<https://github.com/immunefi-team/attackathon-movement/blob/main/protocol-units/da/movement/protocol/light-node/README.md>) of the DA light node explicitly states that both the passthrough (blocky) and sequencer mode can be used.

## Impact Details

The passthrough DA light node does not serialize blocks which means that the bytes cannot be deserialized as a block. As a result, the block cannot be executed which prevents the execution of any other block as well since Celestia blobs must be processed in order.

## References

Code references are provided throughout the report.

## Proof of Concept

## Proof of Concept

1. A request is sent to the passthrough DA light node which only contains transactions (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/networks/movement/movement-full-node/src/node/tasks/transaction\\_ingress.rs#L104-L111>). The passthrough DA light node only submits these transactions without building blocks (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/light-node/src/passthrough.rs#L235-L244>).
2. Then the blobs are sent to Celestia and streamed back to the full node in `execute_settle::run()` (<https://github.com/movementlabsxyz/movement/blob/ec71271bbd022e89a1e3e917629b83442ac2e9d4/networks/movement/movement-full-node/src/node/tasks/execute\\_settle.rs#L70-L98>).
3. This function calls `process_block_from_da()` (<https://github.com/movementlabsxyz/movement/blob/ec71271bbd022e89a1e3e917629b83442ac2e9d4/networks/movement/movement-full-node/src/node/tasks/execute\\_settle.rs#L100-L188>). Inside of this function, `bcs::from_bytes()` (<https://github.com/movementlabsxyz/movement/blob/ec71271bbd022e89a1e3e917629b83442ac2e9d4/networks/movement/movement-full-node/src/node/tasks/execute\\_settle.rs#L145>) is called to deserialize the bytes as a block. However, this is not possible as the passthrough does not serialize blocks. As a result, the function returns an error.
