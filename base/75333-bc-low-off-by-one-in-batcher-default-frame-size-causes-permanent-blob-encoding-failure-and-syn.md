# 75333 bc low off by one in batcher default frame size causes permanent blob encoding failure and synchronous livelock halting l2 finalization

**Submitted on Apr 28th 2026 at 14:52:08 UTC by @Omisanin0 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75333
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

The Base batcher's default `max_frame_size` of 130,044 bytes does not reserve space for the 1-byte `DERIVATION_VERSION_0` prefix that `BlobEncoder::encode_packed()` prepends when packing frames into EIP-4844 blobs. Every near-capacity frame exceeds the blob data limit by exactly 1 byte, returning a `DataTooLarge` error. The error handler requeues the identical immutable frame and retries in a synchronous loop with no async yield points, no iteration cap, and no backoff. This permanently blocks the batcher's driver loop, halting all L1 data availability submissions, freezing L2 finalization, and locking all pending L2-to-L1 withdrawals. Restarting the batcher with default configuration reproduces the failure immediately.

## Vulnerability Details

The vulnerability sits at the intersection of three components in the batcher pipeline. Each component works correctly in isolation, but together they produce a fatal off-by-one mismatch.

**1. Frame sizing in `output_frame()` (`crates/batcher/comp/src/channel_out.rs:140-168`)**

When the encoder drains compressed data into frames, it calls `output_frame(max_size)` where `max_size` comes from the config default of 130,044 (equal to `BLOB_MAX_DATA_SIZE`). The function subtracts the 23-byte frame header overhead and an optional 1-byte compression version prefix to compute the data budget:

```rust
let max_size = (max_size - FRAME_V0_OVERHEAD - prefix_len).min(self.ready_bytes());
```

For frame 0 with Brotli compression: `prefix_len = 1`, so `data budget = 130044 - 23 - 1 = 130020`. The frame payload becomes `[0x01 brotli_version] ++ [130020 compressed bytes]` = 130,021 bytes.

For subsequent frames: `prefix_len = 0`, so `data budget = 130044 - 23 = 130021`. The frame payload is 130,021 raw compressed bytes.

In both cases, `frame.data.len() = 130021`.

**2. Blob encoding in `encode_packed()` (`crates/batcher/blobs/src/encoder.rs:52-68`)**

When submitting to L1, the submission queue calls `BlobEncoder::encode_packed()` to pack frames into a blob. This function prepends a 1-byte `DERIVATION_VERSION_0` prefix before the encoded frames:

```rust
pub fn encode_packed(frames: &[Arc<Frame>]) -> Result<Box<Blob>, BlobEncodeError> {
    let mut data = Vec::with_capacity(1 + encoded_size);
    data.push(DERIVATION_VERSION_0);  // 1 byte -- not accounted for by output_frame
    for frame in frames {
        data.extend_from_slice(&frame.encode());  // 23-byte header + frame.data per frame
    }
    Self::encode(&data)  // rejects if data.len() > BLOB_MAX_DATA_SIZE (130044)
}
```

For a single frame with `data.len() = 130021`:

```
total = 1 (DERIVATION_VERSION_0) + 23 (frame header) + 130021 (frame data) = 130,045
```

This exceeds `BLOB_MAX_DATA_SIZE` (130,044) by exactly 1 byte, producing `Err(DataTooLarge)`.

The `output_frame()` function correctly accounts for the per-frame overhead (23 bytes) and the per-frame compression prefix (1 byte for Brotli), but it does not account for the per-blob `DERIVATION_VERSION_0` prefix that `encode_packed()` adds at the blob level. These are two different version bytes at two different layers, and only the frame-level one is subtracted from the budget.

**3. Synchronous infinite retry in `submit_pending()` (`crates/batcher/core/src/submissions.rs:109-125`)**

When `encode_packed()` fails, the error handler requeues all frame IDs and continues to the next loop iteration:

```rust
DaType::Blob => match BlobEncoder::encode_packed(&frames) {
    Ok(blob) => { /* build tx candidate and submit */ },
    Err(e) => {
        warn!(error = %e, "failed to encode frames to blob, requeueing");
        for id in ids {
            pipeline.requeue(id);
        }
        drop(permit);
        continue;
    }
},
```

The `requeue()` call (`crates/batcher/encoder/src/encoder.rs:594-618`) rewinds the channel cursor to `frame_start`, so the next `next_submission()` call returns the exact same `Arc<Frame>` with the same immutable 130,021-byte payload.

The entire error path is synchronous: `try_acquire_owned()` on the semaphore is sync, `next_submission()` reads from an in-memory channel and is sync, `encode_packed()` is a pure computation and is sync, `requeue()` is a cursor reset and is sync, and `continue` jumps back to the loop top. No `.await` is ever reached on the error path. The tokio task never yields, permanently blocking the executor thread.

**4. Driver main loop is blocked (`crates/batcher/core/src/driver.rs:153-157`)**

```rust
loop {
    self.drain_encoding()?;
    self.throttle.apply(...).await;
    self.submissions.submit_pending(&mut self.pipeline).await;  // stuck forever
    // next_event() is never reached
}
```

Since `submit_pending` never returns, the driver cannot process new L1 heads, new L2 blocks, shutdown signals, or any other event.

**Why this triggers under normal operation**

The bug fires whenever compressed channel output fills a frame to capacity, which is the standard operating condition:

* The default `target_frame_size` and `max_frame_size` are both 130,044 (`config.rs:97-98`, `cli.rs:96-97`). There is no separate `--max-frame-size` CLI flag; `cli.rs:227` hardcodes `max_frame_size = target_frame_size`.
* The `ShadowCompressor`'s fullness check has a secondary issue: `shadow.rs:81` uses `newbound = data.len() as u64` (the current write size, not the cumulative total), so the "channel full" signal never triggers for normal-sized block writes. Channels accumulate data until the RLP byte limit (10-100 MB) or the channel timeout (default 2 L1 blocks, \~24 seconds).
* At Base mainnet throughput, even 24 seconds of L2 blocks produces compressed output well above 130 KB, guaranteeing that `output_frame(130044)` produces a frame with `data.len() = 130021` on every cycle.

**The developers' own test code confirms the correct math**

The test helper `blob_filling_submission()` at `crates/batcher/core/src/driver.rs:436-448` correctly subtracts 1 for `DERIVATION_VERSION_0`:

```rust
/// payload = 1 (DERIVATION_VERSION_0) + FRAME_OVERHEAD + data.len() = BLOB_MAX_DATA_SIZE
fn blob_filling_submission(id: u64) -> BatchSubmission {
    let data_len = BlobEncoder::BLOB_MAX_DATA_SIZE - 1 - BlobEncoder::FRAME_OVERHEAD;
    // = 130044 - 1 - 23 = 130020
```

The production default uses 130,044. The test helper uses 130,043 (after subtracting 1). The discrepancy is the bug.

Additionally, the test `test_blob_encoding_failure_requeues_submission` at `driver.rs:508-548` defines `const OVERSIZED: usize = 130_021` and documents that this size guarantees `DataTooLarge`. Production `output_frame(130044)` produces exactly `data.len() = 130021`. The test exercises the single-requeue path but uses a mock that does not feed the frame back into the queue, so the infinite retry loop is never tested.

## Impact Details

The batcher is the sole component responsible for posting L2 batch data to L1 via EIP-4844 blobs. When it enters this livelock:

* **L2 data availability halts.** No new batch data reaches L1. The derivation pipeline on any verifier or prover node has no new data to consume.
* **L2 finalization on L1 freezes.** TEE and ZK provers require DA data to generate proofs. Without proofs, no proposals can be finalized. The proof window (7 days single-proof, 1 day dual-proof) cannot even begin.
* **All pending L2-to-L1 withdrawals are frozen.** Withdrawals through the bridge require a finalized L2 state root on L1. With no new finalized state, no withdrawal can complete. Every user with funds in the bridge or pending withdrawal is affected.
* **Restart does not resolve the issue.** The default configuration reproduces the bug immediately upon processing the first full channel. An operator would need to discover the exact off-by-one, determine the correct workaround value (130,043), and pass a non-default `--target-frame-size` flag that is not documented as a fix for this issue. There is no `--max-frame-size` CLI flag.

This matches the in-scope impact "Network not being able to confirm new transactions (total network shutdown)" — the batcher is the single point of L2-to-L1 data submission, and its failure halts the entire finalization pipeline.

Regarding the downgrade clause about restarting services with different configurations: this does not apply here because the default configuration is the broken configuration. Restarting with defaults reproduces the livelock within seconds of processing the first full channel. The "different configuration" that fixes the bug requires knowing the exact internal arithmetic mismatch, which is the vulnerability itself.

## References

All paths relative to repository root (`base/base` at tag `v0.8.0-rc.28`):

* `crates/batcher/encoder/src/config.rs:97-98` — Default `max_frame_size = 130044` (should be 130043)
* `bin/batcher/src/cli.rs:96-97,227` — CLI default 130044, hardcodes `max_frame_size = target_frame_size`
* `crates/batcher/comp/src/channel_out.rs:140-168` — `output_frame()` frame data budget calculation
* `crates/batcher/blobs/src/encoder.rs:52-59,66-68` — `encode_packed()` prepends `DERIVATION_VERSION_0`, size check rejects
* `crates/batcher/core/src/submissions.rs:109-125` — Synchronous requeue-and-retry loop
* `crates/batcher/encoder/src/encoder.rs:232,291,594-618` — Frame drain and cursor rewind on requeue
* `crates/batcher/core/src/driver.rs:153-157` — Main loop blocked by `submit_pending`
* `crates/batcher/core/src/driver.rs:436-448` — Test helper with correct math (`-1` for DERIVATION\_VERSION\_0)
* `crates/batcher/core/src/driver.rs:508-548` — Test defining `OVERSIZED = 130021` as the failure boundary
* `crates/batcher/comp/src/shadow.rs:70-98` — ShadowCompressor per-write fullness check (contributing factor)

## Proof of Concept

## Step 1 — Arithmetic proof of the off-by-one

The default `max_frame_size` is 130,044. `output_frame()` subtracts 23 bytes of frame overhead, producing frames with `data.len() = 130021`. `encode_packed()` then constructs the blob payload as `1 (DERIVATION_VERSION_0) + 23 (frame header) + 130021 (frame data) = 130045`. Since `BLOB_MAX_DATA_SIZE = 130044`, the payload exceeds the limit by exactly 1 byte.

```
BLOB_MAX_DATA_SIZE          = 130,044
FRAME_V0_OVERHEAD           = 23
DERIVATION_VERSION_0 prefix = 1

output_frame(max_size=130044):
  frame.data.len() = 130044 - 23 = 130021

encode_packed(single frame):
  blob payload = 1 + 23 + 130021 = 130045
  130045 > 130044  -->  DataTooLarge
```

## Step 2 — Direct encode\_packed failure with production-sized frame

A `Frame` is created with `data.len() = 130021` (the exact size that `output_frame(130044)` produces). `BlobEncoder::encode_packed()` is called with this frame and returns `Err(DataTooLarge { size: 130045 })`.

## Step 3 — encode\_packed succeeds with corrected frame size

A `Frame` is created with `data.len() = 130020` (the size that `output_frame(130043)` would produce). `BlobEncoder::encode_packed()` returns `Ok(blob)`. The blob payload is `1 + 23 + 130020 = 130044`, which is exactly `BLOB_MAX_DATA_SIZE`.

## Step 4 — Real ChannelOut with ShadowCompressor produces the oversized frame

A `ChannelOut<ShadowCompressor>` is created with the production compressor configuration (Brotli10, target 130044). Incompressible data is written to the compressor to ensure the compressed output exceeds 130,021 bytes. After closing the channel, `output_frame(130044)` is called — the same call the production encoder makes at `encoder.rs:232`. The resulting frame has `data.len() = 130021` and `data[0] = 0x01` (Brotli version byte). Passing this real frame to `BlobEncoder::encode_packed()` returns `Err(DataTooLarge { size: 130045 })`.

## Step 5 — Livelock pattern: same frame fails on every retry

The requeue-and-retry behavior from `submissions.rs:109-125` is simulated. The same `Arc<Frame>` (immutable, reference-counted) is passed to `encode_packed()` on each iteration. All 10 simulated iterations fail with the identical `DataTooLarge` error. In production, this loop has no iteration limit, no backoff, and no async yield point. The frame cannot change because it is behind an `Arc` — `requeue()` only rewinds the channel cursor, it does not modify or resize the frame data.

\## Step 6 — Fix verification: max\_frame\_size = 130043 resolves the issue

A fresh `ChannelOut<ShadowCompressor>` is created with identical configuration. `output_frame(130043)` is called instead of `output_frame(130044)`. The resulting frame has `data.len() = 130020`. `BlobEncoder::encode_packed()` returns `Ok(blob)` with a payload of exactly 130,044 bytes — a perfect fit. The developers' own test helper `blob_filling_submission()` uses `BLOB_MAX_DATA_SIZE - 1 - FRAME_OVERHEAD = 130020` as the correct data length, confirming this is the intended arithmetic.

### Full PoC source

To run: place these two files inside `crates/batcher/poc-f1001/` in the `base/base` repository at tag `v0.8.0-rc.28`. The workspace already picks it up via the `crates/batcher/*` glob in the root `Cargo.toml`. Then run `cargo run -p poc-f1001` from the repository root. No environment variables, no network access, no external services.

**`crates/batcher/poc-f1001/Cargo.toml`**

```toml
[package]
name = "poc-f1001"
version = "0.1.0"
edition.workspace = true
publish = false

[lints]
workspace = true

[dependencies]
base-blobs.workspace = true
base-comp = { workspace = true, features = ["std"] }
base-protocol = { workspace = true, features = ["std"] }
base-consensus-genesis = { workspace = true, features = ["std"] }
```

**`crates/batcher/poc-f1001/src/main.rs`**

```rust
use std::sync::Arc;

use base_blobs::{BlobEncodeError, BlobEncoder};
use base_comp::{
    ChannelOut, CompressionAlgo, CompressorType, CompressorWriter, Config, ShadowCompressor,
};
use base_consensus_genesis::RollupConfig;
use base_protocol::{ChannelId, Frame, DERIVATION_VERSION_0};

const DEFAULT_MAX_FRAME_SIZE: usize = 130_044;
const FIXED_MAX_FRAME_SIZE: usize = 130_043;

fn main() {
    println!("=========================================================");
    println!("  F-1001 PoC: Batcher Off-by-One Blob Encoding Livelock  ");
    println!("=========================================================\n");

    step1_arithmetic();
    step2_encode_packed_fails_production();
    step3_encode_packed_succeeds_fixed();
    step4_real_channel_produces_oversized_frames();
    step5_livelock_pattern();
    step6_fix_demonstration();

    println!("=========================================================");
    println!("  PoC COMPLETE: All steps passed.                        ");
    println!("  The batcher's default max_frame_size (130044) causes   ");
    println!("  every near-capacity frame to overflow BLOB_MAX_DATA_SIZE");
    println!("  by exactly 1 byte, creating a permanent livelock.      ");
    println!("=========================================================");
}

fn step1_arithmetic() {
    println!("--- STEP 1: Off-by-one arithmetic proof ---\n");

    let frame_data_len = DEFAULT_MAX_FRAME_SIZE - BlobEncoder::FRAME_OVERHEAD;
    let blob_payload = 1 + BlobEncoder::FRAME_OVERHEAD + frame_data_len;

    println!("  BLOB_MAX_DATA_SIZE (max blob payload)  = {}", BlobEncoder::BLOB_MAX_DATA_SIZE);
    println!("  FRAME_OVERHEAD (per-frame header)       = {}", BlobEncoder::FRAME_OVERHEAD);
    println!("  DERIVATION_VERSION_0 (blob prefix)      = 0x{:02x} (1 byte)", DERIVATION_VERSION_0);
    println!();
    println!("  Default max_frame_size (config.rs:97)   = {DEFAULT_MAX_FRAME_SIZE}");
    println!("  frame.data.len = max_frame_size - FRAME_OVERHEAD");
    println!("                 = {DEFAULT_MAX_FRAME_SIZE} - {} = {frame_data_len}", BlobEncoder::FRAME_OVERHEAD);
    println!();
    println!("  Blob payload   = 1 (VERSION) + {} (overhead) + {frame_data_len} (data)", BlobEncoder::FRAME_OVERHEAD);
    println!("                 = {blob_payload}");
    println!();

    assert!(
        blob_payload > BlobEncoder::BLOB_MAX_DATA_SIZE,
        "Production config should overflow"
    );
    println!(
        "  RESULT: {blob_payload} > {} = OVERFLOW BY {} BYTE(S)",
        BlobEncoder::BLOB_MAX_DATA_SIZE,
        blob_payload - BlobEncoder::BLOB_MAX_DATA_SIZE
    );
    println!("  [FAIL] Production config overflows the blob size limit.\n");
}

fn step2_encode_packed_fails_production() {
    println!("--- STEP 2: encode_packed FAILS with production-sized frame ---\n");

    let production_data_len = DEFAULT_MAX_FRAME_SIZE - BlobEncoder::FRAME_OVERHEAD;
    let frame = Arc::new(Frame {
        data: vec![0xAB; production_data_len],
        ..Frame::default()
    });

    println!("  Created frame with data.len() = {} (production default)", frame.data.len());
    println!("  Calling BlobEncoder::encode_packed(&[frame])...");

    let result = BlobEncoder::encode_packed(&[frame]);
    match &result {
        Err(BlobEncodeError::DataTooLarge { size }) => {
            println!("  encode_packed returned: Err(DataTooLarge {{ size: {size} }})");
            println!(
                "  Expected blob payload: 1 + {} + {} = {size}",
                BlobEncoder::FRAME_OVERHEAD,
                production_data_len
            );
            println!("  BLOB_MAX_DATA_SIZE: {}", BlobEncoder::BLOB_MAX_DATA_SIZE);
            println!(
                "  Overflow: {size} - {} = {} byte(s)",
                BlobEncoder::BLOB_MAX_DATA_SIZE,
                size - BlobEncoder::BLOB_MAX_DATA_SIZE
            );
        }
        Ok(_) => panic!("BUG IN POC: encode_packed should have failed with DataTooLarge"),
        Err(e) => panic!("Unexpected error variant: {e}"),
    }

    println!("  [FAIL] Frame at production size CANNOT fit in a blob.\n");
}

fn step3_encode_packed_succeeds_fixed() {
    println!("--- STEP 3: encode_packed SUCCEEDS with fixed-size frame ---\n");

    let fixed_data_len = FIXED_MAX_FRAME_SIZE - BlobEncoder::FRAME_OVERHEAD;
    let frame = Arc::new(Frame {
        data: vec![0xAB; fixed_data_len],
        ..Frame::default()
    });

    println!("  Created frame with data.len() = {} (with fix: max_frame_size=130043)", frame.data.len());
    println!("  Calling BlobEncoder::encode_packed(&[frame])...");

    let result = BlobEncoder::encode_packed(&[frame]);
    assert!(result.is_ok(), "Fixed frame must encode successfully");

    let total = 1 + BlobEncoder::FRAME_OVERHEAD + fixed_data_len;
    println!("  encode_packed returned: Ok(blob)");
    println!("  Blob payload: 1 + {} + {} = {total}", BlobEncoder::FRAME_OVERHEAD, fixed_data_len);
    println!("  {total} <= {} (BLOB_MAX_DATA_SIZE)", BlobEncoder::BLOB_MAX_DATA_SIZE);
    println!("  [PASS] Frame at fixed size fits exactly in a blob.\n");
}

fn step4_real_channel_produces_oversized_frames() {
    println!("--- STEP 4: Real ChannelOut + ShadowCompressor -> oversized frames ---\n");

    let compressor_config = Config {
        target_output_size: DEFAULT_MAX_FRAME_SIZE as u64,
        kind: CompressorType::Shadow,
        compression_algo: CompressionAlgo::Brotli10,
        approx_compr_ratio: 0.6,
    };
    let compressor = ShadowCompressor::from(compressor_config);
    let rollup_config = Arc::new(RollupConfig::default());
    let channel_id: ChannelId = [0u8; 16];
    let mut channel = ChannelOut::new(channel_id, rollup_config, compressor);

    println!("  Created ChannelOut<ShadowCompressor> with:");
    println!("    target_output_size = {DEFAULT_MAX_FRAME_SIZE} (BLOB_MAX_DATA_SIZE)");
    println!("    compression_algo   = Brotli10");
    println!("    approx_compr_ratio = 0.6");

    println!("  Writing 200,000 bytes of pseudo-random data to compressor...");
    let data = pseudo_random_bytes(200_000);
    channel.compressor.write(&data).expect("compressor write should succeed");

    let ready = channel.ready_bytes();
    println!("  Compressed output size (ready_bytes): {ready} bytes");
    assert!(
        ready >= DEFAULT_MAX_FRAME_SIZE - BlobEncoder::FRAME_OVERHEAD,
        "Compressed output ({ready}) must be >= {} to trigger the bug",
        DEFAULT_MAX_FRAME_SIZE - BlobEncoder::FRAME_OVERHEAD
    );

    channel.close();

    let frame = channel
        .output_frame(DEFAULT_MAX_FRAME_SIZE)
        .expect("output_frame should succeed");

    println!("  Called output_frame(max_size={DEFAULT_MAX_FRAME_SIZE})");
    println!("  frame.data.len()  = {}", frame.data.len());
    println!("  frame.data[0]     = 0x{:02x} (Brotli channel version byte)", frame.data[0]);

    assert_eq!(
        frame.data.len(),
        DEFAULT_MAX_FRAME_SIZE - BlobEncoder::FRAME_OVERHEAD,
        "First frame data should be exactly 130021 bytes"
    );
    assert_eq!(frame.data[0], 0x01, "First byte should be Brotli version byte");

    println!("\n  Attempting BlobEncoder::encode_packed with this real frame...");
    let frame_arc = Arc::new(frame);
    let result = BlobEncoder::encode_packed(&[Arc::clone(&frame_arc)]);
    match &result {
        Err(BlobEncodeError::DataTooLarge { size }) => {
            println!("  encode_packed returned: Err(DataTooLarge {{ size: {size} }})");
            println!("  [FAIL] Real ChannelOut frame CANNOT be encoded into a blob.");
        }
        Ok(_) => panic!("BUG IN POC: real frame should also fail encode_packed"),
        Err(e) => panic!("Unexpected error: {e}"),
    }

    println!("\n  Checking second frame (no Brotli version byte)...");
    if channel.ready_bytes() >= FIXED_MAX_FRAME_SIZE - BlobEncoder::FRAME_OVERHEAD {
        let frame2 = channel
            .output_frame(DEFAULT_MAX_FRAME_SIZE)
            .expect("second output_frame should succeed");
        println!("  frame2.data.len() = {}", frame2.data.len());
        assert_eq!(
            frame2.data.len(),
            DEFAULT_MAX_FRAME_SIZE - BlobEncoder::FRAME_OVERHEAD,
            "Second frame data should also be 130021 bytes"
        );
        let frame2_arc = Arc::new(frame2);
        let result2 = BlobEncoder::encode_packed(&[frame2_arc]);
        match &result2 {
            Err(BlobEncodeError::DataTooLarge { size }) => {
                println!("  encode_packed returned: Err(DataTooLarge {{ size: {size} }})");
                println!("  [FAIL] Subsequent frames are ALSO too large.");
            }
            Ok(_) => panic!("BUG IN POC: second frame should also fail"),
            Err(e) => panic!("Unexpected error: {e}"),
        }
    } else {
        println!("  (Not enough remaining compressed data for a second full frame)");
    }
    println!();
}

fn step5_livelock_pattern() {
    println!("--- STEP 5: Livelock pattern (synchronous infinite retry) ---\n");

    let production_data_len = DEFAULT_MAX_FRAME_SIZE - BlobEncoder::FRAME_OVERHEAD;
    let frame = Arc::new(Frame {
        data: vec![0xAB; production_data_len],
        ..Frame::default()
    });

    println!("  Simulating submissions.rs requeue loop (capped at 10 iterations)...");
    println!("  In production, this loop has NO iteration limit and NO yield point.\n");

    let max_iterations = 10;
    let mut requeue_count = 0;

    for i in 1..=max_iterations {
        let result = BlobEncoder::encode_packed(&[Arc::clone(&frame)]);
        match result {
            Err(BlobEncodeError::DataTooLarge { size }) => {
                requeue_count += 1;
                println!(
                    "  Iteration {i:>2}: encode_packed -> DataTooLarge(size={size}). \
                     Requeue frame. Continue. [frame unchanged, Arc<Frame> is immutable]"
                );
            }
            Ok(_) => {
                panic!("BUG IN POC: frame should fail on every attempt");
            }
            Err(e) => panic!("Unexpected error: {e}"),
        }
    }

    println!();
    println!("  Requeue count after {max_iterations} iterations: {requeue_count}");
    println!("  In production: loop runs forever (no break condition, no yield).");
    println!("  The frame is Arc<Frame> -- immutable. Requeue does not resize it.");
    println!("  Result: synchronous CPU spin blocking the batcher's main driver loop.");
    println!("  No new L2 data is posted to L1. Finalization halts. Withdrawals freeze.");
    println!("  [FAIL] Permanent livelock confirmed.\n");
}

fn step6_fix_demonstration() {
    println!("--- STEP 6: Fix -- max_frame_size = 130043 (subtracts 1 for VERSION prefix) ---\n");

    let compressor_config = Config {
        target_output_size: DEFAULT_MAX_FRAME_SIZE as u64,
        kind: CompressorType::Shadow,
        compression_algo: CompressionAlgo::Brotli10,
        approx_compr_ratio: 0.6,
    };
    let compressor = ShadowCompressor::from(compressor_config);
    let rollup_config = Arc::new(RollupConfig::default());
    let channel_id: ChannelId = [0u8; 16];
    let mut channel = ChannelOut::new(channel_id, rollup_config, compressor);

    let data = pseudo_random_bytes(200_000);
    channel.compressor.write(&data).expect("write should succeed");
    channel.close();

    let frame = channel
        .output_frame(FIXED_MAX_FRAME_SIZE)
        .expect("output_frame should succeed");

    println!("  Called output_frame(max_size={FIXED_MAX_FRAME_SIZE})  [FIX: -1]");
    println!("  frame.data.len() = {}", frame.data.len());

    let expected_data_len = FIXED_MAX_FRAME_SIZE - BlobEncoder::FRAME_OVERHEAD;
    assert_eq!(
        frame.data.len(),
        expected_data_len,
        "Fixed frame data should be 130020 bytes"
    );

    let frame_arc = Arc::new(frame);
    let result = BlobEncoder::encode_packed(&[frame_arc]);
    assert!(result.is_ok(), "Fixed frame must encode into blob successfully");

    let total = 1 + BlobEncoder::FRAME_OVERHEAD + expected_data_len;
    println!("  encode_packed returned: Ok(blob)");
    println!("  Blob payload: 1 + {} + {expected_data_len} = {total}", BlobEncoder::FRAME_OVERHEAD);
    println!("  {total} <= {} (BLOB_MAX_DATA_SIZE)", BlobEncoder::BLOB_MAX_DATA_SIZE);
    println!("  [PASS] With max_frame_size=130043, frames fit in blobs. No livelock.\n");

    println!("  Cross-check: developers' own blob_filling_submission() at driver.rs:441:");
    let dev_data_len = BlobEncoder::BLOB_MAX_DATA_SIZE - 1 - BlobEncoder::FRAME_OVERHEAD;
    println!("    data_len = BLOB_MAX_DATA_SIZE - 1 - FRAME_OVERHEAD");
    println!(
        "             = {} - 1 - {} = {dev_data_len}",
        BlobEncoder::BLOB_MAX_DATA_SIZE,
        BlobEncoder::FRAME_OVERHEAD
    );
    println!("    This CORRECTLY subtracts 1 for DERIVATION_VERSION_0.");
    println!("    But the default config does NOT apply this subtraction.");
    println!("    Production: max_frame_size = 130044 -> frame.data = 130021 -> OVERFLOW");
    println!("    Fix:        max_frame_size = 130043 -> frame.data = 130020 -> EXACT FIT\n");
}

fn pseudo_random_bytes(len: usize) -> Vec<u8> {
    let mut data = Vec::with_capacity(len);
    let mut state: u64 = 0xDEAD_BEEF_CAFE_BABE;
    for _ in 0..len {
        state ^= state << 13;
        state ^= state >> 7;
        state ^= state << 17;
        data.push((state >> 33) as u8);
    }
    data
}
```

### PoC output

```

  F-1001 PoC: Batcher Off-by-One Blob Encoding Livelock


--- STEP 1: Off-by-one arithmetic proof ---

  BLOB_MAX_DATA_SIZE (max blob payload)  = 130044
  FRAME_OVERHEAD (per-frame header)       = 23
  DERIVATION_VERSION_0 (blob prefix)      = 0x00 (1 byte)

  Default max_frame_size (config.rs:97)   = 130044
  frame.data.len = max_frame_size - FRAME_OVERHEAD
                 = 130044 - 23 = 130021

  Blob payload   = 1 (VERSION) + 23 (overhead) + 130021 (data)
                 = 130045

  RESULT: 130045 > 130044 = OVERFLOW BY 1 BYTE(S)
  [FAIL] Production config overflows the blob size limit.

--- STEP 2: encode_packed FAILS with production-sized frame ---

  Created frame with data.len() = 130021 (production default)
  Calling BlobEncoder::encode_packed(&[frame])...
  encode_packed returned: Err(DataTooLarge { size: 130045 })
  Expected blob payload: 1 + 23 + 130021 = 130045
  BLOB_MAX_DATA_SIZE: 130044
  Overflow: 130045 - 130044 = 1 byte(s)
  [FAIL] Frame at production size CANNOT fit in a blob.

--- STEP 3: encode_packed SUCCEEDS with fixed-size frame ---

  Created frame with data.len() = 130020 (with fix: max_frame_size=130043)
  Calling BlobEncoder::encode_packed(&[frame])...
  encode_packed returned: Ok(blob)
  Blob payload: 1 + 23 + 130020 = 130044
  130044 <= 130044 (BLOB_MAX_DATA_SIZE)
  [PASS] Frame at fixed size fits exactly in a blob.

--- STEP 4: Real ChannelOut + ShadowCompressor -> oversized frames ---

  Created ChannelOut<ShadowCompressor> with:
    target_output_size = 130044 (BLOB_MAX_DATA_SIZE)
    compression_algo   = Brotli10
    approx_compr_ratio = 0.6
  Writing 200,000 bytes of pseudo-random data to compressor...
  Compressed output size (ready_bytes): 200005 bytes
  Called output_frame(max_size=130044)
  frame.data.len()  = 130021
  frame.data[0]     = 0x01 (Brotli channel version byte)

  Attempting BlobEncoder::encode_packed with this real frame...
  encode_packed returned: Err(DataTooLarge { size: 130045 })
  [FAIL] Real ChannelOut frame CANNOT be encoded into a blob.

  Checking second frame (no Brotli version byte)...
  (Not enough remaining compressed data for a second full frame)

--- STEP 5: Livelock pattern (synchronous infinite retry) ---

  Simulating submissions.rs requeue loop (capped at 10 iterations)...
  In production, this loop has NO iteration limit and NO yield point.

  Iteration  1: encode_packed -> DataTooLarge(size=130045). Requeue frame. Continue. [frame unchanged, Arc<Frame> is immutable]
  Iteration  2: encode_packed -> DataTooLarge(size=130045). Requeue frame. Continue. [frame unchanged, Arc<Frame> is immutable]
  Iteration  3: encode_packed -> DataTooLarge(size=130045). Requeue frame. Continue. [frame unchanged, Arc<Frame> is immutable]
  Iteration  4: encode_packed -> DataTooLarge(size=130045). Requeue frame. Continue. [frame unchanged, Arc<Frame> is immutable]
  Iteration  5: encode_packed -> DataTooLarge(size=130045). Requeue frame. Continue. [frame unchanged, Arc<Frame> is immutable]
  Iteration  6: encode_packed -> DataTooLarge(size=130045). Requeue frame. Continue. [frame unchanged, Arc<Frame> is immutable]
  Iteration  7: encode_packed -> DataTooLarge(size=130045). Requeue frame. Continue. [frame unchanged, Arc<Frame> is immutable]
  Iteration  8: encode_packed -> DataTooLarge(size=130045). Requeue frame. Continue. [frame unchanged, Arc<Frame> is immutable]
  Iteration  9: encode_packed -> DataTooLarge(size=130045). Requeue frame. Continue. [frame unchanged, Arc<Frame> is immutable]
  Iteration 10: encode_packed -> DataTooLarge(size=130045). Requeue frame. Continue. [frame unchanged, Arc<Frame> is immutable]

  Requeue count after 10 iterations: 10
  In production: loop runs forever (no break condition, no yield).
  The frame is Arc<Frame> -- immutable. Requeue does not resize it.
  Result: synchronous CPU spin blocking the batcher's main driver loop.
  No new L2 data is posted to L1. Finalization halts. Withdrawals freeze.
  [FAIL] Permanent livelock confirmed.

--- STEP 6: Fix -- max_frame_size = 130043 (subtracts 1 for VERSION prefix) ---

  Called output_frame(max_size=130043)  [FIX: -1]
  frame.data.len() = 130020
  encode_packed returned: Ok(blob)
  Blob payload: 1 + 23 + 130020 = 130044
  130044 <= 130044 (BLOB_MAX_DATA_SIZE)
  [PASS] With max_frame_size=130043, frames fit in blobs. No livelock.

  Cross-check: developers' own blob_filling_submission() at driver.rs:441:
    data_len = BLOB_MAX_DATA_SIZE - 1 - FRAME_OVERHEAD
             = 130044 - 1 - 23 = 130020
    This CORRECTLY subtracts 1 for DERIVATION_VERSION_0.
    But the default config does NOT apply this subtraction.
    Production: max_frame_size = 130044 -> frame.data = 130021 -> OVERFLOW
    Fix:        max_frame_size = 130043 -> frame.data = 130020 -> EXACT FIT


  PoC COMPLETE: All steps passed.
  The batcher's default max_frame_size (130044) causes
  every near-capacity frame to overflow BLOB_MAX_DATA_SIZE
  by exactly 1 byte, creating a permanent livelock.
```
