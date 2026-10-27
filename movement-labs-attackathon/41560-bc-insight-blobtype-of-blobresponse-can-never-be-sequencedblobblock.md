# #41560 \[BC-Insight] BlobType of BlobResponse can never be SequencedBlobBlock

**Submitted on Mar 16th 2025 at 14:58:42 UTC by @KlosMitSoss for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41560
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/>
* **Impacts:**

## Description

## Brief/Intro

When streaming blobs from a specified height, a DaBlob is converted into a BlobResponse with the BlobType being either sequenced or passed through. However, the BlobType of the BlobResponse can never be `SequencedBlobBlock` as `to_blob_passed_through_read_response()` is always called.

## Vulnerability Details

```rust
	async fn stream_read_from_height(
		&self,
		request: tonic::Request<StreamReadFromHeightRequest>,
	) -> std::result::Result<tonic::Response<Self::StreamReadFromHeightStream>, tonic::Status> {
            ... ...
			loop {
				let response_content = tokio::select! {
					// Yield from the data stream
					block_opt = blob_stream.next() => {
						match block_opt {
							Some(Ok((height, da_blob))) => {
								match verifier.verify(da_blob, height.as_u64()).await.map_err(|e| tonic::Status::internal(e.to_string())).and_then(|verifed_blob| {
>>							verifed_blob.into_inner().to_blob_passed_through_read_response(height.as_u64()).map_err(|e| tonic::Status::internal(e.to_string()))
								}) {
            ... ...
```

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/light-node/src/passthrough.rs#L155>

As can be seen in the provided code snippet, only `to_blob_passed_through_read_response()` is called. On the other hand, `to_blob_sequenced_read_response()` is never called which means that the BlobType can never be `SequencedBlobBlock` even when the DA light node is used in sequencer mode.

The DA light node explicitly differentiates between two types of API responses depending on whether passthrough mode or sequencer mode is used. This means the sequencer mode should return a BlobResponse with the correct BlobType.

## Impact Details

The BlobType of the BlobResponse can never be `SequencedBlobBlock`. However, this does not have any big impact as both types are treated the same way in the execution logic (<https://github.com/movementlabsxyz/movement/blob/ec71271bbd022e89a1e3e917629b83442ac2e9d4/networks/movement/movement-full-node/src/node/tasks/execute\\_settle.rs#L111-L118>).

## Proof of Concept

## Proof of Concept

1. `execute_sette::run()` (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/networks/movement/movement-full-node/src/node/tasks/execute\\_settle.rs#L70-L98>) is called.
2. Inside of this function, `da_light_node_client.stream_read_from_height()` (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/client/src/lib.rs#L49-L66>) is called which then calls `client.client_mut().stream_read_from_height()` (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/light-node/src/sequencer.rs#L328-L333>).
3. This function calls `stream_read_from_height()` (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/light-node/src/passthrough.rs#L131-L189>) of the passthrough.
4. Here, blobs from a specified height are streamed and verified. While doing so, `to_blob_passed_through_read_response()` (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/util/src/blob/ir/blob.rs#L138-L144>) is called.
5. This function converts a DaBlob into a BlobResponse with the blob passed through (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/util/src/blob/ir/blob.rs#L129>). Specifically, this means that the BlobType of the BlobResponse is going to be `PassedThroughBlob`. However, `to_blob_sequenced_read_response()` (<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/util/src/blob/ir/blob.rs#L147-L153>) is never called which means that the BlobType of the BlobResponse can never be `SequencedBlobBlock`.
