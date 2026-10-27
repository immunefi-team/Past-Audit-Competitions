# #43168 \[BC-Insight] Under normal usage of the blockchain, transactions will not be persisted

**Submitted on Apr 3rd 2025 at 07:02:33 UTC by @br0nz3p1ck4x3 for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43168
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/networks/movement/movement-full-node>
* **Impacts:**
  * Direct loss of funds
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

## Description

## Description

Inside `transaction_ingres::spawn_write_next_transaction_batch()`, we find the following code:

```rust
if transactions.len() > 0 {
			info!(
				target: "movement_timing",
				batch_id = %batch_id,
				transaction_count = transactions.len(),
				"built_batch_write"
			);
			let batch_write = BatchWriteRequest { blobs: transactions };
			let mut buf = Vec::new();
			batch_write.encode_raw(&mut buf);
			info!("batch_write size: {}", buf.len());
			// spawn the actual batch write request in the background
			let mut da_light_node_client = self.da_light_node_client.clone();
			tokio::spawn(async move {
				match da_light_node_client.batch_write(batch_write.clone()).await {
					Ok(_) => {
						info!(
							target: "movement_timing",
							batch_id = %batch_id,
							"batch_write_success"
						);
						return;
					}
					Err(e) => {
						warn!("failed to write batch to DA: {:?} {:?}", e, batch_id);
					}
				}
			});
		}

		Ok(Continue(()))
	}
```

This function writes to the `DA light node client`. However, there is a bug in this functionality that will result in the so-called `ghost transactions`, transaction that were made by the end-user but ended up not being stored in any DB.

The issue lays here:

```rust
tokio::spawn(async move {
				match da_light_node_client.batch_write(batch_write.clone()).await {
					Ok(_) => {
						info!(
							target: "movement_timing",
							batch_id = %batch_id,
							"batch_write_success"
						);
						return;
					}
					Err(e) => {
						warn!("failed to write batch to DA: {:?} {:?}", e, batch_id);
					}
				}
			});
		}

		Ok(Continue(()))
```

If the writing fails and they return an `Err`, the only thing that will happen is that it will warn the operator with a warning message. After the warning message, `Ok(Continue(()))` will follow.

## Impact

An honest user will make a transaction which will be successfully executed, however, this transaction will not be persisted to storage. The impact is diverse, it will lead to undefined behavior, could lead to loss of funds depending on the timing of this happening, availability of the chain.

## Recommended Mitigation Steps

If persistence fails, either wait and retry or gracefully exit.

## Proof of Concept

## Proof of Concept

This proof of concept is very straight-forward, it does not even need a malicious actor. This issue will happen during normal usage of the blockchain, which makes the impact even more severe.

Step one: Alice, Bob and Charlie are having fun on Movement Network. They are preparing themselves for a hyped NFT drop on `ClosedSea`

Step two: Alice, Bob and Charlie both submit transactions with an immense amount of gas. On their end, this transaction was successful.

Step three: However, due to `batch_write()` returning an `Err`, nothing was persisted to the storage. There is no reference of Alice, Bob and Charlie ever making this transaction, which can lead to the impacts defined above.
