# #42143 \[BC-Critical] Decompressing a maliciously crafted blob leads to shutting down all Movement DA Light Nodes in a Movement based network which using a centralized Sequencer.

**Submitted on Mar 21st 2025 at 09:13:30 UTC by @a090325 for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42143
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

## Description

## Brief/Intro

Movement DA Light Node serves as a broker between DA Layer to Sequencer and Execution Layers. In order to save storage size, batch of transactions are compressed using zstd by DA Light Node (running alongside a Sequencer Node) before submitting to DA Layer using Celestia Node API. The DA Light Node running alongside an Execution Node will the the reverse: fetching from DA layer, then decompressing and decoding blobs.

Zstd library can compress 100GB data into 3MB compressed file (>30k compression ratio). Malicious/compromised Sequencer can submit samll compressed data which can be decompressed into hundreds of GB in one shot into memory which will trigger Linux OOM killer on recommended machine for running Movement Node (128 GB RAM according to white paper). Since the maximum size for a response from DA layer to a request from execution layer is 33 MB (explained below), even Movement Node with 512GB RAM can be shut down using larger payload.

Since the malicious blob is submitted to DA layer, a incident resolution process must involve identifying and removing malicious blob. It leads to temporary network shutdown --> delaying block production in Execution Layer.

## Vulnerability Details

DA Light Node fetches blobs from DA Layer using \[this function] (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/providers/celestia/src/da/mod.rs#L82>)

Under the hood Movement DA Light Node uses \[celestia-rpc crate] (<https://crates.io/crates/celestia-rpc>) to send request to Celestia DA node. The rpc client has a response size limit define by: <https://github.com/eigerco/lumina/blob/c79b48a953413aee36a2696a0df663975aa9a79a/rpc/src/client.rs#L39>

```
 const MAX_EDS_SIZE_BYTES: usize = appconsts::v3::SQUARE_SIZE_UPPER_BOUND
        * appconsts::v3::SQUARE_SIZE_UPPER_BOUND
        * 4
        * SHARE_SIZE;
const MAX_RESPONSE_SIZE: usize = MAX_EDS_SIZE_BYTES + 1024 * 1024; 
```

We can check SQUARE\_SIZE\_UPPER\_BOUND = 128 and SHARE\_SIZE = 512 in this file (<https://github.com/eigerco/lumina/blob/main/types/src/consts.rs#L59> and #203)

In short, MAX\_RESPONSE\_SIZE = 33 MB which is not an issue. But then the blobs' data is decoded by\
\[below function] (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/providers/celestia/src/blob/ir.rs#L8-20>) without blob data's size validation.

```
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

**The core issue is in below line:**

```
let decompressed =
		zstd::decode_all(blob.data.as_slice()).context("failed to decompress blob")?;
```

If a 3MB maliciously crafted blob (which is much smaller than MAX\_RESPONSE\_SIZE = 33MB) is passed to zstd::decode\_all, it would be decompressed into a 100GB Vector of bytes. The malicious blob is created by decompressing a 100GB file with repeated 0s into 3MB compressed file (compression ratio >30000:1 thanks to Run Length Encoding algorithm).

**Potential fix: Instead of decoding blob data in one shot, we should use stream decoder zstd::stream::read and limit decompressed data size**.

## Impact Details

Malicious/compromised centralized sequencer can submit maliciously crafted blob to Movement DA Light Node which causing excessive memory consumption (hundreds of GB) while the blob is decoded. Since Movement DA Light Node always runs as sidecar alongside other components, an unbounded memory consumption will trigger Linux OOM killer which could kill Movement DA Light Node itself and potentially other high memory processes.

If this bug is exploited by malicious or compromised sequencer, Execution & Rollup Layer will lose capability to process transactions. A remediation would likely involve malicious blob removal in DA layer.

Malicious/compromised Celestia DA RPC provider could attack DA Light Node in the same way but the impact is much less severe.

## References

As per project's white paper, it wouldn't be an issue if we use a Decentralized Sequencer Network with PoS consensus instead of a centralized sequencer.

## Link to Proof of Concept

<https://gist.github.com/anh2025/4cb5a3f8df8ad45fb6a90bf673096e44>

## Proof of Concept

## Proof of Concept

Caveat: Although run-able POC is not required, I had tried to create one but I couldn't. Here's a step by step description on how the bug should've been reproduced successfully.

1. Run a local devnet with a full node connecting to a local Celestia node following [MovementLabs' guide](https://github.com/movementlabsxyz/movement?tab=readme-ov-file#running-natively-nix-required). Installing all required packages then run:

`nix develop` and

`just movement-full-node native build.setup.celestia-local.eth-local`

2. Generate a maliciously crafted blob using secret gist below and submit it to Celestia local node using this API:`https://node-rpc-docs.celestia.org/?version=v0.20.4#blob.Submit`\
   Note: your machine must have over 100GB free disk space since the test.bin file size is 100GB. In the end test.bin will be compressed to a 3MB zstd file. Then test.bin.zstd could be read by curl as data field and submitted to Celestia node as a blob.
3. Request malicious blob: DA light node has a gRPC API that Execution layer can call. DA light node will base on request (from Execution layer) to request blobs from DA layer (Celestia node). Internally, DA light node using below Celestia API:`https://node-rpc-docs.celestia.org/?version=v0.20.4#blob.GetAll`
4. DA light node will be OOM-killed while decompressing maliciously crafted blob using zstd. We can modify attached secret gist to increase payload size if target machine has higher RAM capacity.
