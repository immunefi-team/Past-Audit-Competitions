# #43110 \[BC-Critical] Validator can DoS the DA Layer by requesting a big range of blobs

**Submitted on Apr 2nd 2025 at 06:55:27 UTC by @br0nz3p1ck4x3 for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43110
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/protocol/light-node>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## Description

Inside `passthrough::stream_read_from_height()`, the validator makes a request to the DA layer to stream blobs to the validator. This is the relevant line of code:

```rust
			let mut blob_stream = da.stream_da_blobs_from_height(height).await.map_err(|e| tonic::Status::internal(e.to_string()))?;
```

The issue here is that the validator can specify `height` as `1`, which will lead to the DA layer wasting their resources doing lookups in their local DB from `height = 1` until `current_block`. Moreover, after these lookups, these have to be returned to the Validator whom made this request. As such, this simple call will DoS the DA Layer.

## Impact

This simple call will permanently waste the resources of the DA layer. Without access to the DA layer, the Validator will not be able to progress and thus, the network will be stalled.

## Recommended Mitigation Steps

On the DA Layer level, constrain the range of blob height that can be streamed.

## Proof of Concept

## Proof of Concept

The PoC is very straight-forward. We will start by defining a few variables.\
Let:

* `current_block = 184239`
* `height = 1`

Step one: The malicious validator makes a call to `stream_read_from_height()`, using the `height` variable as defined above.

Step two: The honest DA layer operator will try to lookup and return the blobs between `height <-> current_block`, which spans over **184238** blocks.

Step three: The hoenst DA layer will not be able to process this and thus, the network will stall.
