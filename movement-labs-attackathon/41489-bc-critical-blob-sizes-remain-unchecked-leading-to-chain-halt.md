# #41489 \[BC-Critical] Blob sizes remain unchecked leading to chain halt

**Submitted on Mar 15th 2025 at 20:47:46 UTC by @okmxuse for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41489
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/networks/movement/movement-full-node>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

The blob transactions are handled in the following way:

```javascript
	pub async fn execute(&self) -> Result<(), anyhow::Error> {
		// Get the config

		let mut client = MovementDaLightNodeClient::try_http2(self.light_node_url.as_str())
			.await
			.context("Failed to connect to light node")?;

		let mut blocks_from_da = client
			.stream_read_from_height(StreamReadFromHeightRequest { height: self.from_height })
			.await
			.context("Failed to stream blocks from DA")?;

		info!("streaming blocks from DA");

		while let Some(block_res) = blocks_from_da.next().await {
			let response = block_res.context("Failed to get block")?;
			let (_block_bytes, block_timestamp, block_id, da_height) = match response
				.blob
				.ok_or(anyhow::anyhow!("No blob in response"))?
				.blob_type
				.ok_or(anyhow::anyhow!("No blob type in response"))?
			{
				blob_response::BlobType::SequencedBlobBlock(blob) => {
					tracing::info!("Receive SequencedBlobBlock blob");
					(blob.data, blob.timestamp, blob.blob_id, blob.height)
				}
				blob_response::BlobType::PassedThroughBlob(blob) => {
					tracing::info!("Receive PassedThroughBlob blob");
					(blob.data, blob.timestamp, blob.blob_id, blob.height)
				}
				blob_response::BlobType::Heartbeat(_) => {
					tracing::info!("Receive heartbeat blob");
					continue;
				}
				_ => {
					anyhow::bail!("Invalid blob type in response")
				}
			};
			// pretty print (with labels) the block_id, block_timestamp, and da_height
			tracing::info!(
				"Block ID: {}, Block Timestamp: {:?}, DA Height: {}",
				hex::encode(block_id),
				// unix date string from the block timestamp which is in microseconds
				chrono::DateTime::from_timestamp_micros(block_timestamp as i64)
					.context("Failed to convert timestamp to date")?,
				da_height
			);
		}

		info!("Finished streaming blocks from DA");

		Ok(())
	}
}
```

They are retrieved and categorized into three categories. a Sequenced blob, a Passed through blob and finally an invalid blob.

The issue however is that the `blob.data` size remains unchecked and is never checked at all. This is an easy DOS factor for a malicious user by simply providing enermous blobs and therefore causing a chain halt.

### Impact

Malicious user can cause chain halt

### Recommendation

Introduce a `blob.data` check to ensure that the blob is a valid size.

## Proof of Concept

### POC

Since this is a common DOS factor we will showcase this with a POC walkthrough:

* Bob, a malicious user submits blobs
* Bob submits these blobs by creating a script and submitting multiple blobs with the `blob.data` being of an enermous amount
* The `execute` function will pick this up and try to execute it, but due to the size of the blob it will eventually cause the chain to halt
