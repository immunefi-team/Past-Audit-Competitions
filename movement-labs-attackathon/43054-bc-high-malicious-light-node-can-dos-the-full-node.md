# #43054 \[BC-High] malicious light node can dos the full node

## #43054 \[BC-High] Malicious Light Node can DoS the Full Node

**Submitted on Apr 1st 2025 at 08:23:50 UTC by @Blockian for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43054
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/networks/movement/movement-full-node>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

### Description

## Movement Bug Report

### Malicious Light Node can DoS the Full Node

#### Summary

The `StreamBlocks::execute()` function in `src/da/stream_blocks/mod.rs` initiates a block stream from a light node using `stream_read_from_height`. However, it does not impose any limit on how many blocks can be returned or processed. A malicious or faulty light node can exploit this to flood the client with blocks, causing a Denial of Service (DoS).

### Root Cause Analysis

Examining the `StreamBlocks::execute()` function:

```rust
	pub async fn execute(&self) -> Result<(), anyhow::Error> {
    // ... not relevant for this issue
		let mut blocks_from_da = client
			.stream_read_from_height(StreamReadFromHeightRequest { height: self.from_height })
			.await
			.context("Failed to stream blocks from DA")?;

		// ... not relevant for this issue

		while let Some(block_res) = blocks_from_da.next().await {
			// ... not relevant for this issue
    }
	}
```

When processing the blocks received from the light node client, there is no stop mechanism besides `blocks_from_da.next()` being empty.

Thus, a malicious light node can send a huge amount of data, for example a huge amount of `blob_response::BlobType::Heartbeat` and cause the loop to run indefinitely.

### Impact

* A malicious light node can continuously stream an excessive or infinite number of blocks.
* The node processing this stream can get stuck in the loop indefinitely.
* This can compromise the availability and liveness of the chain.

### Proposed Fixes

* Add an upper bound for the number of blocks to stream (e.g., `max_blocks`).
* Break the loop if a certain block height is reached or if the block stream exceeds a safe limit.

### Proof of Concept

## Proof of Concept (PoC)

1. Connect the client to a light node controlled by you.
2. When `stream_read_from_height` is requested from the light node, respond with a huge number of blocks or continuously stream blocks without bound.
3. Observe that the loop taking a huge time to finish or never exits and system resource usage increases over time.
