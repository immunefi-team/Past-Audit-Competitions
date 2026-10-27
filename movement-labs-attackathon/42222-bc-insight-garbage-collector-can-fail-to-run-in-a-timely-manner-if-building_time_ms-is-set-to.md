# #42222 \[BC-Insight] Garbage Collector can fail to run in a timely manner if building\_time\_ms is set to a low value

**Submitted on Mar 21st 2025 at 19:56:48 UTC by @Cryptor for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42222
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/sequencing/memseq/sequencer>
* **Impacts:**

## Description

## Brief/Intro

Garbage Collector can fail to run in a timely manner if building\_time\_ms is set to a low value

## Vulnerability Details

In the following code, the function gc is essentially a garbage collector function

```
async fn gc(&self) -> Result<(), anyhow::Error> {
  
    
    let gc_interval = self.building_time_ms * 2 / 1000 + 1;
		let timestamp_threshold = SystemTime::now()
			.duration_since(UNIX_EPOCH)
			.unwrap()
			.as_secs()
			.saturating_sub(gc_interval);
		let gc_count = self.mempool.gc_mempool_transactions(timestamp_threshold).await?;
		if gc_count != 0 {
			info!("pruned {gc_count} transactions");
		} else {
			debug!("no transactions to prune")
		}
    
    tokio::time::sleep(Duration::from_secs(gc_interval)).await;
    Ok(())
}
```

After each gc run, the task goes to sleep for the duration of the gc\_interval. The problem is that gc\_interval essentially adds 1 second to the building\_time\_ms

```
let gc_interval = self.building_time_ms * 2 / 1000 + 1;
```

This mean that if the building\_time\_ms is set to a value low enough, ie. any value below 500, then the garbage collector will be unable to discard old transactions , potentially adding bloat to the mempool

## Impact Details

Adding a second to the gc\_interval can render the gc to be ineffective when the building time is set to a very low value.

## References

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/sequencing/memseq/sequencer/src/lib.rs#L133-L149>

## Proof of Concept

## Proof of Concept

For example, when building\_time\_ms is 400, the GC interval becomes 1000ms. This creates a window where transactions that should be garbage collected remain in the mempool longer than intended. Over time, this can lead to memory bloat and potential denial of service conditions as the GC runs less frequently than designed. Additionally, any building time less than 500ms will result in the same 1-second GC interval due to the integer division, making the GC unreliable for small building time values.
