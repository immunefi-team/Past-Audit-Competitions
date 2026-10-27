# #41334 \[BC-Critical] Attacker can publish a blob that cannot be deserialized and shut down the movement chain

**Submitted on Mar 13th 2025 at 22:02:39 UTC by @KlosMitSoss for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41334
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

By publishing Celestia blobs with arbitrary data, an attacker can make the deserialization step into a `DaBlob` struct fail. As a consequence, full nodes are not able to retrieve any blocks from DA light nodes and the network is permanently shut down, and unable to process transactions, until DA light nodes are updated.

## Vulnerability Details

Anyone can publish Celestia blobs for any namespace. Therefore, anyone can publish a blob with arbitrary data that cannot be deserialized. However, once a DA light node tries to get the blobs for a namespace at a given height, the deserialization does not continue with the next blob but propagates an error instead. The step-by-step explanation is provided in the Proof of Concept section.

The root cause is in bcs::from\_bytes(), where the attacker controlled `vec<u8>` is deserialized into a `DaBlob`. If deserialization is not possible, an error is returned.

```rust
pub fn into_da_blob<C>(blob: CelestiaBlob) -> Result<DaBlob<C>, anyhow::Error>
where
	C: Curve + for<'de> Deserialize<'de>,
{
	// decompress blob.data with zstd
	let decompressed =
		zstd::decode_all(blob.data.as_slice()).context("failed to decompress blob")?;

	// deserialize the decompressed data with bcs
	let blob = bcs::from_bytes(decompressed.as_slice()).context("failed to deserialize blob")?;

	Ok(blob)
}
```

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/providers/celestia/src/blob/ir.rs#L17>

The mitigation is to continue with deserialization for the next blob instead of returning an error.

## Impact Details

If the deserialization of only one blob for a given height fails, the execution for the blocks at that height will not be possible as the deserialization will continue to fail. It is important to note that the execution always waits for the blocks at the current height to be processed until the blocks at the next height can be processed. Therefore, this issue causes blocks at the current and all the next heights to be non-executable. The network is not able to confirm new transactions and is shut down.

## References

The fact that anyone can publish blobs for any namespace has been exploited before:

<https://forum.celestia.org/t/woods-attack-on-celestia/59>

The attack is different because it requires overwhelming the network with malicious and in our attack, one malicious blob is sufficient. But again, the root cause is the same: blobs can be published for any namespace.

## Proof of Concept

## Proof of Concept

The attack consists of the following steps:

1. `execute_settle::run()` (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/networks/movement/movement-full-node/src/node/tasks/execute\\_settle.rs#L73-L80>) is called.
2. This calls `da_light_node_client.stream_read_from_height()` (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/client/src/lib.rs#L49-L66>).
3. Inside of this function, `client.client_mut().stream_read_from_height()` (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/light-node/src/sequencer.rs#L328-L333>) is called, which calls `pass_through.stream_read_from_height()` (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/light-node/src/passthrough.rs#L131-L189>).
4. Inside of this function, `da.stream_da_blobs_from_height()` (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/da/src/lib.rs#L100-L149>) is called for the requested height. This calls `stream_da_blobs_between_heights()` (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/da/src/lib.rs#L80-L98>), `get_da_blobs_at_height_for_stream()` (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/da/src/lib.rs#L69-L74>) and `get_da_blobs_at_height()` (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/providers/digest-store/src/da/mod.rs#L87-L109>) of the digest store.
5. This function then calls `get_da_blobs_at_height()` (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/providers/celestia/src/da/mod.rs#L82-L116>) of Celestia which gets the blobs for the requested height.
6. Each `CelestiaBlob` is then converted into a `DaBlob` by calling `into_da_blob()` (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/providers/celestia/src/da/mod.rs#L103>).
7. Inside of this function, the decompressed data gets deserialized with bcs (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/providers/celestia/src/blob/ir.rs#L8-L20>). If this call fails due to some arbitrary data, the error is propagated back to the `execute_settle::run()` function which then returns an error (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/networks/movement/movement-full-node/src/node/tasks/execute\\_settle.rs#L76-L80>).
