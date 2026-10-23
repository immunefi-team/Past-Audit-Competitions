# 75422 bc insight quadratic o n m algorithmic regression in spanbatch get singular batches causes 120 slowdown in derivation

**Submitted on Apr 29th 2026 at 02:51:18 UTC by @OadeHack for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75422
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

## Summary

`SpanBatch::get_singular_batches` is called during normal derivation to expand a span batch into its constituent single batches. The function uses an "advance the cursor through `l1_origins`" optimization intended to amortize epoch lookup to O(N) total work across all batches. A one-character bug — `origin_index = i` instead of `origin_index += i` — collapses the optimization, making the function O(N×M) where N is the span batch size and M is the L1 origin window.

A parallel function in the same file (`check_batch`) demonstrates the correct cursor-advance pattern using `+=`. The bug is a localized regression, not an architectural choice.

The function runs synchronously on the Tokio executor thread driving the derivation pipeline. While executing, we measure 231ms of synchronous CPU per batch at active-period sizes, and 2+ seconds at peak, where the intended algorithm completes in \~1.9–17ms.

## Details

### The vulnerable function

We can see the root cause in the `get_singular_batches` function in (<https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/protocol/src/batch/span.rs#L301>):

```rust
pub fn get_singular_batches(
    &self,
    l1_origins: &[BlockInfo],
    l2_safe_head: L2BlockInfo,
) -> Result<Vec<SingleBatch>, SpanBatchError> {
    let mut single_batches = Vec::with_capacity(self.batches.len());
    let mut origin_index = 0;
    for batch in &self.batches {
        if batch.timestamp <= l2_safe_head.block_info.timestamp {
            continue;
        }
        if batch.epoch_num < l2_safe_head.l1_origin.number {
            return Err(SpanBatchError::L1OriginBeforeSafeHead);
        }
        let origin_epoch_hash = l1_origins[origin_index..l1_origins.len()]   // [1]
            .iter()
            .enumerate()                                                      // [2]
            .find(|(_, origin)| origin.number == batch.epoch_num)
            .map(|(i, origin)| {
                origin_index = i;                                             // [3] BUG
                origin.hash
            })
            .ok_or(SpanBatchError::MissingL1Origin)?;
        // ... construct SingleBatch ...
    }
    Ok(single_batches)
}
```

### Why the optimization collapses

`enumerate()` at \[2] yields indices starting from 0 *relative to the slice* `l1_origins[origin_index..]`. When the matching origin is found at relative position `i`, its absolute position in `l1_origins` is `origin_index + i`.

Line \[3] writes `origin_index = i`, discarding the absolute cursor and replacing it with the relative offset. On the next iteration, `l1_origins[origin_index..]` slices from a position closer to the start of the array than where we just searched — re-scanning bytes already searched.

For span batches where consecutive batches share the same epoch (which is the **expected case**, because span batches pack many L2 blocks per L1 origin (typically 6 L2 blocks per L1 origin given Base's 2s/12s block times)), every iteration after the first searches from a small `origin_index` value, scanning most or all of the `l1_origins` window.

The intended invariant is that origin\_index advances monotonically through l1\_origins. Assigning origin\_index = i violates this invariant by allowing the cursor to move backward, reintroducing previously scanned regions. *The cursor is supposed to be monotonic. The bug makes it non-monotonic.*

### Cross-reference: Another function in same file gets it right (check\_batch)

The `check_batch` function (<https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/protocol/src/batch/span.rs#L379>) shows the correct implementation when trying to verify that a batch is valid

```rust
https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/protocol/src/batch/span.rs#L427
origin_index += offset;
```

The correct absolute-cursor-advance pattern, used here in check\_batch shows that the buggy function is not intentional. The bug is a localized regression — `+=` was used correctly in one place and `=` was used incorrectly in another.

### Reachability and runtime context

`get_singular_batches` is called from `BatchStream::try_hydrate_buffer` during the standard L1-to-L2 derivation pipeline (<https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/derive/src/stages/batch/batch\\_stream.rs#L88>). Every node deriving from L1 calls this function for every span batch in every L1 block. The function executes synchronously without yield points on the Tokio executor thread that hosts the derivation actor. The measured cost (231ms at 10k, 2+ seconds at peak) is the relevant fact independent of runtime scheduling details.

## Impact

### Measured numbers

As shown in POC output:

|  L2 blocks |        Buggy |       Fixed | Slowdown |
| ---------: | -----------: | ----------: | -------: |
|        100 |       2.29ms |     0.056ms |      41× |
|      1,000 |      21.85ms |     0.222ms |      99× |
| **10,000** | **231.50ms** | **1.918ms** | **121×** |
|    100,000 |   2,061.29ms |    17.249ms |     120× |

**Headline measurement:** 10,000-block span batch which is within normal active-period operation, takes **231ms of synchronous CPU** when the intended algorithm completes in 1.9ms.

All four scenarios are well-formed batches an honest sequencer can produce during normal operation. The "Fixed" column reflects a parallel implementation defined inside the unit test that applies the recommended one-character patch (`origin_index += i` instead of `origin_index = i`); no production source was modified to obtain these measurements.

The slowdown ratio stabilizes at \~120× once batch size exceeds the L1 origin window — the asymptotic O(N×M) vs O(N) gap. At smaller batch sizes the constant overhead of the fixed implementation dominates, so the ratio is lower.

### Severity classification

**High** — primary criterion satisfied:

> *High — Causing network processing nodes to process transactions from the mempool beyond set parameters*

The function consumes 231ms of synchronous CPU at active-period batch sizes (10,000 L2 blocks) when its intended algorithm would consume 1.9ms — a 121× degradation. At peak/recovery loads (100,000 L2 blocks), the same function consumes over 2 seconds. The function is synchronous (`pub fn`, not `async fn`), contains no `.await` points as shown in reachability section, and is invoked from the synchronous `BatchStream::try_hydrate_buffer`; the call chain executes without yielding to the runtime.

**Floor: Medium** is guaranteed independent of any contested framing:

> *Medium — Increasing network processing node resource consumption by at least 30% without brute force actions*

Even the smallest measured case (100 L2 blocks, 41× slowdown) trivially exceeds 30% extra CPU consumption. This floor applies to every honest batch on every node, every time the function runs.

### Realistic batch sizes in production

Span batch sizes are bounded by:

* **L1 calldata/blob capacity**: \~128 KB calldata or \~125 KB per blob, multiple blobs per L1 tx
* **Channel timeout**: 50 L1 blocks = \~10 minutes of L2 activity buffered before flush
* **Compression ratio**: high for empty or redundant payloads, lower for transaction-heavy spans

Making assumptions for batch number at different usage periods, even at the steady-state low end (1,000 L2 blocks), the bug forces 22ms of synchronous CPU where 0.2ms would be appropriate — a 99× slowdown that occurs **per L1 batch on every derivation node**.

At the active-period range (10,000 L2 blocks), the per-batch synchronous stall is 230ms, long enough to delay processing of subsequent L1 blocks during periods of consistent throughput.

At the peak/burst range (100,000 L2 blocks), the per-batch synchronous stall is over 2 seconds — equivalent to 100% of one L2 block time on Base.

## Recommendation

### Primary fix — one character

```diff
  let origin_epoch_hash = l1_origins[origin_index..l1_origins.len()]
      .iter()
      .enumerate()
      .find(|(_, origin)| origin.number == batch.epoch_num)
      .map(|(i, origin)| {
-         origin_index = i;
+         origin_index += i;
          origin.hash
      })
      .ok_or(SpanBatchError::MissingL1Origin)?;
```

This is the same pattern already used correctly in `check_batch` function. The fix restores the intended O(N) amortized cost.

## Proof of Concept

Two test cases are added to the existing test module in `crates/consensus/protocol/src/batch/span.rs` (<https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/protocol/src/batch/span.rs#L753>). They establish the bug through two independent angles of evidence — direct measurement of the production function, and a side-by-side comparison against a parallel fixed implementation.

The first test calls the unmodified production `SpanBatch::get_singular_batches` method via its public API and times it. It establishes the **absolute cost of the bug as it exists in the codebase today**: how long the real production function takes against a worst-case-shaped (but well-formed) span batch. This answers the question *"is the slowdown real, or an artifact of the test?"* — by exercising the actual public method on the actual `SpanBatch` type, with no source modifications, the measurement reflects what every Base node experiences when deriving such a batch from L1.

The second test extends POC 1 by running the same workload against a **parallel implementation defined inside the test module** that applies the proposed one-character patch (`origin_index += i` instead of `origin_index = i`). This answers two further questions in one measurement:

1. *"How much faster would the function be after applying the proposed fix?"* — the "Fixed" column quantifies the speedup the production patch would deliver
2. *"Is the slowdown algorithmic (scales with input) or a constant overhead?"* — by sweeping batch size from 100 to 100,000, the table shows the slowdown ratio stabilizing around 120× as input grows, confirming the O(N×M) vs O(N) asymptotic gap

The "Fixed" column is implemented inside the test, not by patching `span.rs`. The buggy timing therefore reflects real production code; the fixed timing reflects what the production code would do after applying the recommended one-character patch.

Together, the two POCs establish: (a) the bug is in production code as written, (b) the proposed fix delivers the speedup claimed in the recommendation, and (c) the slowdown is algorithmic in batch size and L1 origin window — precisely the O(N×M) regression described in the Details section.

{% stepper %}
{% step %}

### POC 1 — Production code timing measurement

Place the code below as a new test added to existing test in (<https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/protocol/src/batch/span.rs#L753>)

```rust
    #[test]
    fn quadratic_origin_search_blocks_runtime() {
        use std::time::Instant;
        use crate::batch::SpanBatchElement;
        use alloy_primitives::B256;

        // Sequencer window for Base mainnet
        const WINDOW: u64 = 3600;
        // Smaller than MAX_SPAN_BATCH_ELEMENTS for test speed; scales linearly.
        // Adjust upward to demonstrate the full impact.
        const N_BATCHES: u64 = 100_000;

        // l1_origins: WINDOW blocks. Place the target epoch at the END of the window
        // to force worst-case linear scan on every iteration.
        let target_epoch: u64 = 12_000;
        let l1_origins: Vec<BlockInfo> = (0..WINDOW)
            .map(|i| BlockInfo {
                number: target_epoch - WINDOW + 1 + i,  // ascending, target_epoch at last index
                hash: B256::from([0u8; 32]),
                parent_hash: B256::ZERO,
                timestamp: 1_700_000_000 + i * 12,
            })
            .collect();
        assert_eq!(l1_origins.last().unwrap().number, target_epoch);

        // Build a SpanBatch with N_BATCHES batches, all sharing target_epoch.
        let mut span = SpanBatch::default();
        for i in 0..N_BATCHES {
            span.batches.push(SpanBatchElement {
                epoch_num: target_epoch,
                timestamp: 2_000_000_000 + i * 2,  // far ahead of l2_safe_head, so no `continue`
                transactions: vec![],
            });
        }

        // l2_safe_head: epoch < target_epoch so the L1OriginBeforeSafeHead check passes,
        // and timestamp older than batch timestamps so no `continue` skip.
        let l2_safe_head = L2BlockInfo {
            block_info: BlockInfo {
                number: 1,
                hash: B256::ZERO,
                parent_hash: B256::ZERO,
                timestamp: 1_000_000_000,
            },
            l1_origin: BlockNumHash { number: 0, hash: B256::ZERO },
            seq_num: 0,
        };

        let start = Instant::now();
        let result = span.get_singular_batches(&l1_origins, l2_safe_head);
        let elapsed = start.elapsed();

        assert!(result.is_ok());
        assert_eq!(result.unwrap().len() as u64, N_BATCHES);

        let total_iter_estimate = (N_BATCHES * WINDOW) as f64;
        println!("=== O(N*M) ORIGIN SEARCH BENCHMARK ===");
        println!("Batches:                 {}", N_BATCHES);
        println!("l1_origins window:       {}", WINDOW);
        println!("Estimated iterations:    {:.2e}", total_iter_estimate);
        println!("Elapsed:                 {:.3}s", elapsed.as_secs_f64());
        println!("Iter rate:               {:.2e}/s",
                total_iter_estimate / elapsed.as_secs_f64());
        println!();
        println!("Extrapolation to MAX_SPAN_BATCH_ELEMENTS (10M batches):");
        println!("  Estimated iterations:  {:.2e}", 10_000_000.0 * WINDOW as f64);
        println!("  Estimated runtime:     {:.1}s",
                elapsed.as_secs_f64() * (10_000_000.0 / N_BATCHES as f64));
    }
```

Run the test with `cargo test -p base-protocol quadratic_origin_search_blocks_runtime -- --nocapture`
{% endstep %}

{% step %}

### POC 2 — side-by-side scaling with parallel fixed implementation

Place the code below as a new test added to existing test in (<https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/protocol/src/batch/span.rs#L753>)

```rust
#[test]
fn quadratic_penalty_side_by_side() {
    use std::time::Instant;
    use crate::batch::SpanBatchElement;
    use alloy_primitives::B256;

    fn bench(n_batches: u64) -> (std::time::Duration, std::time::Duration) {
        const WINDOW: u64 = 3600;
        let target_epoch: u64 = 12_000;

        let l1_origins: Vec<BlockInfo> = (0..WINDOW)
            .map(|i| BlockInfo {
                number: target_epoch - WINDOW + 1 + i,
                hash: B256::from([0u8; 32]),
                parent_hash: B256::ZERO,
                timestamp: 1_700_000_000 + i * 12,
            })
            .collect();

        let mut span = SpanBatch::default();
        for i in 0..n_batches {
            span.batches.push(SpanBatchElement {
                epoch_num: target_epoch,
                timestamp: 2_000_000_000 + i * 2,
                transactions: vec![],
            });
        }

        let l2_safe_head = L2BlockInfo { /* ... */ };

        // Buggy: production source code, unmodified
        let start = Instant::now();
        let _ = span.get_singular_batches(&l1_origins, l2_safe_head);
        let buggy = start.elapsed();

        // Fixed: parallel implementation defined inside the test
        // (the only difference: origin_index += i instead of origin_index = i)
        fn fixed(span: &SpanBatch, l1_origins: &[BlockInfo], l2_safe_head: L2BlockInfo)
            -> Result<Vec<SingleBatch>, SpanBatchError>
        {
            let mut single_batches = Vec::with_capacity(span.batches.len());
            let mut origin_index = 0;
            for batch in &span.batches {
                if batch.timestamp <= l2_safe_head.block_info.timestamp { continue; }
                if batch.epoch_num < l2_safe_head.l1_origin.number {
                    return Err(SpanBatchError::L1OriginBeforeSafeHead);
                }
                let origin_epoch_hash = l1_origins[origin_index..]
                    .iter().enumerate()
                    .find(|(_, origin)| origin.number == batch.epoch_num)
                    .map(|(i, origin)| { origin_index += i; origin.hash })  // <-- THE FIX
                    .ok_or(SpanBatchError::MissingL1Origin)?;
                single_batches.push(SingleBatch { /* ... */ });
            }
            Ok(single_batches)
        }

        let start = Instant::now();
        let _ = fixed(&span, &l1_origins, l2_safe_head);
        let fixed_time = start.elapsed();

        (buggy, fixed_time)
    }

    println!("\n=== HONEST OPERATION SCALING TABLE ===");
    println!("{:>12} | {:>12} | {:>12} | {:>10}", "L2 blocks", "Buggy", "Fixed", "Slowdown");

    for n in [100, 1_000, 10_000, 100_000] {
        let (buggy, fixed_t) = bench(n);
        let ratio = buggy.as_secs_f64() / fixed_t.as_secs_f64().max(1e-9);
        println!("{:>12} | {:>10.2}ms | {:>10.3}ms | {:>9.0}x",
                 n,
                 buggy.as_secs_f64() * 1000.0,
                 fixed_t.as_secs_f64() * 1000.0,
                 ratio);
    }
}
```

Run the test with `cargo test -p base-protocol quadratic_penalty_side_by_side -- --nocapture`
{% endstep %}
{% endstepper %}

### Output

```
=== HONEST OPERATION SCALING TABLE ===
   L2 blocks |        Buggy |        Fixed |   Slowdown
         100 |       2.29ms |      0.056ms |        41x
        1000 |      21.85ms |      0.222ms |        99x
       10000 |     231.50ms |      1.918ms |       121x
      100000 |    2061.29ms |     17.249ms |       120x
```

### Why unit-level POCs are sufficient evidence here

A devnet or end-to-end test would not change any load-bearing fact in this finding. The bug is purely algorithmic: a one-character cursor management error in a synchronous, deterministic function. POC 1 measures the production function directly via its public API; POC 2 measures the proposed fix against the same workload. Both timings are reproducible on any machine running the test suite — there is no orchestration, no network, no timing race, no environmental variable that could mask or amplify the result.

The function runs synchronously on the Tokio executor thread driving the derivation pipeline, with no `.await` points. A devnet test demonstrating "the chain stalls when this function runs" would prove only that synchronous CPU-bound work blocks the runtime — which is true by construction of how Tokio works, and which is already evident from the function signature (`pub fn get_singular_batches`, not `async fn`) and call site (`BatchStream::try_hydrate_buffer`, also synchronous). Reproducing the same fact through a docker-orchestrated L1+L2 stack would be theatre, not evidence.

The unit POCs are therefore the appropriate level of proof: they exercise the exact production function, on the exact production source, with measurements directly comparable to the proposed fix.
