# 75096 bc medium da backlog bytes incorrectly reports the backlog

\#75096 \[BC-Medium] `da_backlog_bytes()` incorrectly reports the backlog

**Submitted on Apr 27th 2026 at 08:04:40 UTC by @shadowHunter for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75096
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.24>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

### Description

#### Brief/Intro

In Span mode, `da_backlog_bytes()` incorrectly reports the backlog as `0` while blocks are staged in `span_accumulator` awaiting channel flush.

The encoder maintains a `da_backlog_bytes` cache field that is incremented when a block is added and decremented when a block is considered "encoded." In Span mode, the decrement happens when a block enters `span_accumulator` in `step()`. However `span_accumulator` is an in-memory staging area, blocks staged there have not been written to any channel, produced any frames, or been submitted to L1 in any form. The decrement is applied at the wrong lifecycle point.

In Single mode the decrement is correct because a block moves directly into an open channel during `step()` with no intermediate staging area. The bug is specific to Span mode where the two-phase accumulate-then-flush design means `step()` and actual channel encoding are decoupled.

#### Vulnerability Details

{% stepper %}
{% step %}
Operator starts batcher with Span mode and throttling enabled per the below config.

```rust
OP_BATCHER_BATCH_TYPE=1
OP_BATCHER_THROTTLE_THRESHOLD=1000000
```

{% endstep %}

{% step %}
Sequencer produces L2 blocks. Driver calls `add_block` for each, incrementing the backlog cache by the block's user tx bytes.

```rust
// encoder.rs:378-380
let backlog_bytes = Self::block_da_backlog_bytes(&block);
self.da_backlog_bytes += backlog_bytes;
self.blocks.push_back(block);
```

{% endstep %}

{% step %}
Driver calls `step()` to encode each block. In Span mode each block is pushed into `span_accumulator` and `da_backlog_bytes` is immediately subtracted before any channel is opened or any frame is produced.

```rust
// encoder.rs:421-423
self.span_accumulator.push((single_batch, seq_num));
self.block_cursor += 1;
self.da_backlog_bytes = self.da_backlog_bytes.saturating_sub(block_da_backlog_bytes);
```

{% endstep %}

{% step %}
`span_accumulator` now holds unshipped blocks but `da_backlog_bytes` reads `0`. No channel has been opened. No frames exist. Nothing has been submitted to L1.

```rust
// encoder.rs:748
fn da_backlog_bytes(&self) -> u64 {
    self.da_backlog_bytes  // returns 0
}
```

{% endstep %}

{% step %}
Driver calls `throttle.apply()` with `0` as the backlog signal on every step loop iteration.

```rust
// driver.rs:155
self.throttle.apply(self.pipeline.da_backlog_bytes()).await;
```

{% endstep %}

{% step %}
Throttle compares `0` against `threshold_bytes` of `1_000_000`. Condition is never true. Sequencer is never told to slow down.

```rust
// throttle.rs:148
if da_backlog_bytes >= self.config.threshold_bytes {
```

{% endstep %}

{% step %}
Blocks only leave `span_accumulator` when `close_current_channel` is called on timeout or size trigger. Until then `da_backlog_bytes` stays `0` regardless of how many blocks accumulate — `span_raw_bytes` which tracks the exact byte count is never included.

```rust
// encoder.rs:196-198
if add_ok {
    self.span_accumulator.clear();
    self.span_raw_bytes = 0;  // span_raw_bytes zeroed here but da_backlog_bytes already 0
```

{% endstep %}
{% endstepper %}

#### Impact Details

The throttle compares `da_backlog_bytes()` value against `threshold_bytes` to decide whether to instruct the sequencer to slow down block production. With `da_backlog_bytes()` returning `0` while real backlog exists in `span_accumulator`, the throttle never fires during the accumulation window. The sequencer receives no slowdown signal and continues producing blocks at full speed, causing unbounded growth of `span_accumulator` in memory until the next channel flush.

#### Recommendation

Do not subtract from `da_backlog_bytes` when a block enters `span_accumulator` in `step()`. Remove this subtraction:

```rust
// encoder.rs — REMOVE these lines in the Span mode path of step()
self.da_backlog_bytes =
    self.da_backlog_bytes.saturating_sub(block_da_backlog_bytes);
```

Instead subtract in `close_current_channel()` at the same point `span_raw_bytes` is zeroed on successful flush:

```rust
// encoder.rs — ADD in close_current_channel() on the add_ok path
if add_ok {
    self.span_accumulator.clear();
    self.da_backlog_bytes =
        self.da_backlog_bytes.saturating_sub(self.span_raw_bytes as u64);
    self.span_raw_bytes = 0;
    self.span_opened_at_l1 = None;
}
```

### Proof of Concept

Add in `crates/batcher/encoder/src/encoder.rs`, inside the `#[cfg(test)] mod tests { }` block, right after the existing `test_da_backlog_excludes_deposits` test (around line 852).

```rust
/// POC: da_backlog_bytes drops to zero while blocks are still in span_accumulator,
/// causing the throttle to never fire despite a real backlog.
///
/// In Span mode, step() moves blocks into span_accumulator and subtracts their
/// bytes from da_backlog_bytes. But span_accumulator has NOT been flushed to a
/// channel yet — the bytes are still unshipped backlog. da_backlog_bytes returns
/// 0 (or severe undercount) while blocks are sitting unsubmitted in the accumulator.
///
/// The throttle in driver.rs calls da_backlog_bytes() as its sole backpressure
/// signal. With this bug, the throttle is blind to span_accumulator contents.
#[test]
fn poc_span_mode_da_backlog_drops_to_zero_before_flush() {
    let rollup_config = Arc::new(RollupConfig::default());
    let config = EncoderConfig {
        batch_type: BatchType::Span,
        ..EncoderConfig::default()
    };
    let mut encoder = BatchEncoder::new(rollup_config, config);

    // Add 3 blocks with user txs — these contribute real backlog bytes.
    let b1 = make_block_with_user_tx(B256::ZERO);
    let b1_hash = b1.header.hash_slow();
    let b2 = make_block_with_user_tx(b1_hash);
    let b2_hash = b2.header.hash_slow();
    let b3 = make_block_with_user_tx(b2_hash);

    encoder.add_block(b1).unwrap();
    encoder.add_block(b2).unwrap();
    encoder.add_block(b3).unwrap();

    // Confirm backlog is non-zero after adding blocks — bytes are unshipped.
    let backlog_before_step = encoder.da_backlog_bytes();
    assert!(
        backlog_before_step > 0,
        "backlog must be non-zero after adding blocks with user txs"
    );

    // Step through all blocks. In Span mode this moves each block into
    // span_accumulator. No channel has been opened, no frames produced,
    // nothing has been submitted to L1 yet.
    encoder.step().unwrap(); // block 1 → span_accumulator
    encoder.step().unwrap(); // block 2 → span_accumulator
    encoder.step().unwrap(); // block 3 → span_accumulator

    // At this point: span_accumulator holds 3 blocks. No channel open.
    // No frames produced. Nothing submitted. Backlog should still be non-zero.
    let backlog_after_step = encoder.da_backlog_bytes();

    // BUG: this assertion FAILS — da_backlog_bytes returns 0 because the
    // perf-caching refactor subtracts bytes when blocks enter span_accumulator
    // instead of when they are flushed to a channel.
    assert!(
        backlog_after_step > 0,
        "BUG: da_backlog_bytes is {} after stepping 3 blocks into span_accumulator \
         but before any channel flush — throttle is blind to {} bytes of real backlog",
        backlog_after_step,
        backlog_before_step,
    );
}
```

Output

`da_backlog_bytes()` returns `0` while `225` bytes of real unshipped backlog sit in `span_accumulator`. The throttle receives `0`, never fires, sequencer is never told to slow down.

```
root@LAPTOP-PGMUQ2ED:~/base# RUSTFLAGS="-C linker=gcc" cargo test -p base-batcher-encoder poc_span_mode_da_backlog_drops_to_zero_before_flush -- --nocapture
running 1 test

thread 'encoder::tests::poc_span_mode_da_backlog_drops_to_zero_before_flush' (61046) panicked at crates/batcher/encoder/src/encoder.rs:905:5:
BUG: da_backlog_bytes is 0 after stepping 3 blocks into span_accumulator but before any channel flush — throttle is blind to 225 bytes of real backlog

test encoder::tests::poc_span_mode_da_backlog_drops_to_zero_before_flush ... FAILED
```
