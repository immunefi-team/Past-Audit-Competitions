# #41243 \[BC-Insight] The mempool garbage collector doesn't fully execute garbage collection on each iteration

**Submitted on Mar 12th 2025 at 21:32:25 UTC by @jovi for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41243
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/mempool/move-rocks>
* **Impacts:**

## Description

## Summary

A logic flaw in `gc_mempool_transactions` causes it to remove **only one** expired transaction each time it’s invoked. Although multiple invocations (or a loop) will eventually clear all stale entries, it leads to excessive I/O overhead, unnecessary looping, and potential database bloat.

## Vulnerability Details

### Location

* **File**: `protocol-units/mempool/move-rocks/src/lib.rs`
* **Function**: `gc_mempool_transactions`
* **Caller**: `gc()` (in `sequencing/memseq/sequencer/src/lib.rs`)

### Description

* The `gc_mempool_transactions` function stops after removing the *first* qualifying transaction. It never continues iterating over all stale transactions within the same invocation.
* The `gc()` caller function (see snippet below) implies it expects multiple transactions to be pruned in one pass—logging the total removed each cycle — yet, due to the single-item removal logic, each call to `gc_mempool_transactions` can only return `gc_count == 1` when there are still many stale entries.

#### `gc()` Caller Snippet

```rust
async fn gc(&self) -> Result<(), anyhow::Error> {
    let gc_interval = self.building_time_ms * 2 / 1000 + 1;
    let timestamp_threshold = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap()
        .as_secs()
        .saturating_sub(gc_interval);

    // Attempt to prune transactions older than 'timestamp_threshold'
    let gc_count = self.mempool.gc_mempool_transactions(timestamp_threshold).await?;

    if gc_count != 0 {
        // The log implies multiple pruned transactions could be reported
        info!("pruned {gc_count} transactions");
    } else {
        debug!("no transactions to prune");
    }

    // Sleep until the next GC cycle
    tokio::time::sleep(Duration::from_secs(gc_interval)).await;
    Ok(())
}
```

Because `gc_mempool_transactions` removes only **one** transaction per call, the above function either logs *1 transaction removed* or *none*—even though the developer apparently expected that a single call might remove many stale entries.

### Impact

1. **Resource Inefficiency**: Potentially thousands of extra calls (and disk writes) if the mempool has many expired transactions.
2. **Performance Degradation**: Repeatedly scanning the mempool in separate passes prolongs garbage collection and can bloat disk usage.
3. **Operational Confusion**: Log messages (e.g. “pruned 1 transaction” repeatedly) may not reflect the intended behavior of removing all expired transactions in one pass.

***

## Recommended Fix

In `gc_mempool_transactions`, change the single `if let Some(...)` block to a loop:

```rust
while let Some(res) = iter.next() {
    let (key, value) = res?;
    let transaction: MempoolTransaction = bcs::from_bytes(&value)?;
    batch.delete_cf(&cf_handle, &key);
    batch.delete_cf(&lookups_cf_handle, transaction.transaction.id().to_vec());
    transaction_count += 1;
}
```

This ensures **all** transactions below the threshold are pruned in one invocation, aligning with the `gc()` function’s expectation that multiple transactions might be removed together.

## Proof of Concept

Override the `test_rocksdb_gc` test at RocksdbMempool implementation in `protocol-units/mempool/move-rocks/src/lib.rs`:

```rust
async fn test_rocksdb_gc() -> Result<(), Error> {
    let temp_dir = tempdir()?;
    let path = temp_dir.path().to_str().unwrap();
    let mempool = RocksdbMempool::try_new(path)?;

    let transaction1 = MempoolTransaction::at_time(Transaction::new(vec![1], 0, 0), 2);
    let transaction1_id = transaction1.transaction.id();
    mempool.add_mempool_transaction(transaction1).await?;
    assert!(mempool.has_mempool_transaction(transaction1_id).await?);

    sleep(Duration::from_secs(1)).await;

    let transaction2 = MempoolTransaction::at_time(Transaction::new(vec![2], 0, 0), 3);
    let transaction2_id = transaction2.transaction.id();
    let transaction2_ts = transaction2.timestamp;
    mempool.add_mempool_transaction(transaction2).await?;

    // Wait again so transaction1 is definitely < threshold
    sleep(Duration::from_secs(1)).await;

    // GC call #1
    mempool.gc_mempool_transactions(transaction2_ts + 1).await?;
    assert!(!mempool.has_mempool_transaction(transaction1_id).await?);
    assert!(mempool.has_mempool_transaction(transaction2_id).await?);

    // GC call #2
    mempool.gc_mempool_transactions(transaction2_ts + 1).await?;
    assert!(!mempool.has_mempool_transaction(transaction2_id).await?);

    Ok(())
}
```

This confirms that **one** transaction is pruned each time—contrary to the usual expectation of removing *all* stale transactions in a single pass.

***
