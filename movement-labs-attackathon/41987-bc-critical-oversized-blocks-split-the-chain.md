# 41987 bc critical oversized blocks split the chain

## #41987 \[BC-Critical] Oversized blocks split the chain

**Submitted on Mar 19th 2025 at 19:46:55 UTC by @jovi for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41987
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/sequencing/memseq/sequencer>
* **Impacts:**
  * Unintended permanent chain split requiring hard fork (network partition requiring hard fork)
  * Unintended chain split (network partition)

### Description

## Summary

An attacker can craft blocks that exceeds Celestia’s \~2 MiB blob limit. Because the sequencer does not split or retry large blocks, such a block is **rejected by Celestia**, yet the sequencer proceeds as if it were valid. This situation creates a one‐sided fork: the Movement’s local chain diverges from the network’s recognized chain on Celestia DA layer.

***

## Vulnerability Description

1. **Block Building**\
   The sequencer packages transactions in a block. Large user transactions can inflate the block’s serialized size beyond Celestia’s limit (roughly 2 MiB). I've included an oversized block getting built at the POC.
2. **DA Submission**\
   After building a block, the aggregator calls:

   ```rust
   async fn submit_blocks(&self, blocks: Vec<Block>) -> Result<(), anyhow::Error> {
       for block in blocks {
           let data: InnerSignedBlobV1Data<C> = block.try_into()?;
           let blob = data.try_to_sign(&self.pass_through.signer).await?;
           self.pass_through.da.submit_blob(blob.into()).await?;
       }
       Ok(())
   }
   ```

   Here, `block` is serialized (BCS) and made even larger by adding signature data.
3. **Submit blob**\
   Submit blob will create the Celestia Blob then submit it:

```rust
{
	fn submit_blob(
		&self,
		data: DaBlob<C>,
	) -> Pin<Box<dyn Future<Output = Result<(), DaError>> + Send + '_>> {
		Box::pin(async move {
			debug!("submitting blob to celestia {:?}", data);

			// create the blob
			let celestia_blob = self
				.create_new_celestia_blob(data)
				.map_err(|e| DaError::Internal(format!("failed to create celestia blob :{e}")))?;

			debug!("created celestia blob {:?}", celestia_blob);

			// submit the blob to the celestia node
			self.submit_celestia_blob(celestia_blob)
				.await
				.map_err(|e| DaError::Internal(format!("failed to submit celestia blob :{e}")))?;

			Ok(())
		})
	}
```

4. **Non-blocking error**\
   When Celestia’s `blob_submit` sees a blob larger than the max allowed (\~2 MiB), it **rejects** the transaction. The node, however, does **not** re-queue or split that block; instead, it continues believing it has advanced to “Block #N.” Other nodes see no record of that block on Celestia.

```rust
pub async fn submit_celestia_blob(&self, blob: CelestiaBlob) -> Result<u64, anyhow::Error> {
		let config = TxConfig::default();
		// config.with_gas(2);
@>		let height = self.default_client.blob_submit(&[blob], config).await.map_err(|e| {
			error!(error = %e, "failed to submit the blob");
			anyhow::anyhow!("Failed submitting the blob: {}", e)
		})?;

		Ok(height)
	}
```

[Celestia documentation stating it will reject blobs over 2MiB](https://docs.celestia.org/how-to-guides/submit-data#maximum-blob-size) :`The maximum total blob size in a transaction is just under **2 MiB (1,973,786 bytes)**, based on a 64x64 share grid (4096 shares).`\
Also `It is advisable to submit transactions where the total blob size is significantly smaller than 1.8 MiB (e.g. 500 KiB) in order for your transaction to get included in a block quickly. If a tx contains blobs approaching 1.8 MiB then there will be no room for any other transactions. This means that your transaction will only be included in a block if it has a higher gas price than every other transaction in the mempool.`

5. **Result: Local Fork**\
   If the sequencer node builds further blocks on top of the rejected block, it diverges from the chain the rest of the network acknowledges (the chain posted to Celestia DA). That is effectively an **unintended local chain split (network partition).**

***

## Attack Scenario

* **Attacker Goal:** Force aggregator to produce an invalid blob from Celestia’s standpoint, so the DA layer/bridging logic see a discrepancy (the aggregator’s chain vs. the canonical chain).
* **Approach:**
  1. Flood the sequencer’s mempool with large‐payload transactions, paying enough fees to ensure they are included. Considering we are utilizing approximately 1500 bytes
  2. The sequencer eventually builds a block with a **serialized size > 2 MiB**.
  3. Celestia rejects the block's blob as it is oversized. The node does not re-try or roll back, so it continues on an isolated fork. The reason for that is: `tick_build_blocks` will process the block and submit it to a receiver channel that will build the blobs. Once the sender channel is done with the block, it considers the block as processed and waits for the next block from memseq.
* **Cost**: The attacker’s cost depends on gas fees + setting up multiple proxy connections to make transaction submissions without reaching rate limiting + setting up enough accounts to avoid the max number of transactions per address at the mempool.
* If we consider solely the gas costs for one attack to generate a fork we have: 1500 bytes per transaction.
* It would be broken down into:
  * (MIN\_TRANSACTION\_GAS\_UNITS + INTRINSIC\_GAS\_PER\_BYTE \* excess) \* minimum gas unit price
  * We consider minimum gas unit price as the proper value because other transactions that pay more than this value will serve to make the block even bigger.
* Excess bytes is: 1500 - 600 (large\_transaction\_cutoff) bytes per transaction.
* min\_transaction\_gas\_units: 2\_760\_000
* intrinsic\_gas\_per\_byte is the price per excess byte: 1\_158
* GAS\_UNIT\_PRICE is set to 100.\
  Each attack transaction would be worth 380220000 units of Move. Considering the token has 8 decimals, 3,8 Move would be charged. At the current price of 0.45 USD per Move token, 1,71 \* 2000 transactions would mean a chain fork would cost 3400 USD per attack.

***

## Impact

**Cross-Layer Impact**

* **Bridge Failures**: Bridges relying on Celestia for Movement chain data would **halt withdrawals** if the DA layer lacks proof of the forked blocks, freezing cross-chain assets.
* **Rollup Disruption**: Rollups depending on Movement’s DA would generate **invalid state transitions**, forcing costly recalculations.
* **Validator Discord**: Honest validators following Celestia’s canonical chain would **reject the sequencer’s fork**, creating consensus instability.

**Natural Triggers (Non-Malicious)**

* **Organic Traffic Surges**: A sudden influx of large transactions (e.g., NFT mints, DeFi activity) could push blocks over the limit **without attacker intent**, causing accidental forks.
* **Upgrade Artifacts**: Post-upgrade, new transaction types with larger payloads might inadvertently violate size constraints.

***

## Recommended Mitigation

1. **Pre‐Check Block Size**\
   Enforce a **dynamic block size limit** accounting for:
   * BCS serialization overhead.
   * Signature/public key additions during blob creation.
   * Celestia’s *current* max blob size (allowing for on-chain parameter updates).
   * **Throughput Preservation**:
     * The sequencer can **immediately build subsequent blocks** even if one is split/rejected.
     * Since each Celestia blob corresponds to one block, splitting an oversized block into multiple smaller ones **doesn’t delay processing**—transactions are repackaged into new blocks instantly.
2. **Roll Back or Split**\
   If submission fails, revert to the last valid block or split the transactions into multiple smaller blocks. Make sure the transactions are not lost.
3. **Monitor Blob Submit Return Values**\
   On failure, do not finalize that block in the aggregator’s local chain.

### Proof of Concept

***

## Proof of concept

In order to showcase how an oversized block is built, please paste the following test at the `protocol-units/sequencing/memseq/sequencer/src/lib.rs` file:

```rust
	#[tokio::test]
async fn test_block_build_over_2_mb() -> Result<(), anyhow::Error> {
    use std::time::Instant;
    use toml;
    use serde_json;
    use bcs; // Make sure bcs is in Cargo.toml, e.g., `bcs = "0.1"`

    // Create a temporary directory for the RocksDB instance.
    let dir = tempfile::tempdir()?;
    let path = dir.path().to_path_buf();

    // Create a Memseq instance that can hold 2,000 transactions per block,
    // with a very short building time for demonstration.
    let memseq = Memseq::try_move_rocks(path, 2000, 10)?
        .with_block_size(2000)
        .with_building_time_ms(10);

    // For demonstration, build one block.
    for block_index in 0..1 {
        // Create a batch of 2,000 transactions.
        let txs: Vec<Transaction> = (0..2000)
            .map(|i| {
                let global_index: i32 = block_index * 2000 + i;
                // Convert the i32 to 4 bytes (little endian) and repeat that slice 375 times
                // to yield exactly 1500 bytes.
                let data = global_index.to_le_bytes()[..].repeat(375);
                Transaction::new(data, 0, 0)
            })
            .collect();

        // Publish all transactions in a single batch.
        memseq.publish_many(txs).await?;
        let start_time = Instant::now();

        // Wait for the block to be built.
        let maybe_block = memseq.wait_for_next_block().await?;
        let elapsed = start_time.elapsed();

        assert!(
            maybe_block.is_some(),
            "Expected block #{} to be created",
            block_index + 1
        );

        let block = maybe_block.unwrap();

        println!(
            "Time to build block #{}: {} ms ({} transactions)",
            block_index + 1,
            elapsed.as_millis(),
            block.transactions().len()
        );

        // --- Verifying the block size ---

        // 1. BCS Serialization
        let bcs_serialized = bcs::to_bytes(&block)?;
        println!(
            "Serialized block size (BCS) for block #{}: {} bytes",
            block_index + 1,
            bcs_serialized.len()
        );
        assert!(
            bcs_serialized.len() > 2 * 1024 * 1024,
            "BCS-serialized block should exceed 2 MB"
        );
    }

    Ok(())
}

```

Make sure to import the used dependencies at the local Cargo.toml:

```toml
[dependencies]
...
bcs = { workspace = true }
serde_json = { workspace = true }
toml = { workspace = true }
...
```

Run the tests locally with the following command:

```shell
cargo 'test' '--package' 'memseq' '--lib' '--' 'test::test_block_build_over_2_mb' '--exact' '--show-output'
```

The output should look like this:

```shell
running 1 test
test test::test_block_build_over_2_mb ... ok

successes:

---- test::test_block_build_over_2_mb stdout ----
Time to build block #1: 128 ms (2000 transactions)
Serialized block size (BCS) for block #1: 3100067 bytes
```

The test’s BCS-serialized block (3.1 MiB) **does not include signatures**. Final Celestia blobs would be even larger (due to `InnerSignedBlobV1` additions), guaranteeing rejection.\
This proves the sequencer’s local chain progresses with invalid blocks, while Celestia ignores them.
