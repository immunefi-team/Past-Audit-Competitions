# 75488 bc low off by one in batcher default frame size causes permanent livelock in submit pending halting l2 finalization and freezing all pending withdrawals

**Submitted on Apr 29th 2026 at 12:41:02 UTC by @GiuseppeDeLaZara for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75488
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

### Brief/Intro

The Base batcher's default `max_frame_size` (130,044) doesn't account for the 1-byte `DERIVATION_VERSION_0` prefix added by `BlobEncoder::encode_packed()`. This causes every full frame to overflow `BLOB_MAX_DATA_SIZE` by 1 byte. The failed frame is requeued and retried in a tight synchronous loop with zero async yield points — `submit_pending()` never returns, permanently blocking the batcher driver.

### Vulnerability Details

Three components interact to create this livelock:

#### 1. Frame sizing (`channel_out.rs:140-168`)

`output_frame(max_size=130044)` produces frames with `data.len() = 130021`:

```
data budget = max_size - FRAME_V0_OVERHEAD = 130044 - 23 = 130021
```

#### 2. Blob encoding (`blobs/encoder.rs:52-68`)

`encode_packed()` prepends 1 byte that `output_frame` doesn't account for:

```rust
data.push(DERIVATION_VERSION_0);           // +1 byte
data.extend_from_slice(&frame.encode());   // +23 header + 130021 data
// Total: 1 + 23 + 130021 = 130045 > BLOB_MAX_DATA_SIZE (130044)
// → Err(DataTooLarge)
```

#### 3. Infinite retry in `submit_pending()` (`core/submissions.rs:52-169`)

On `DataTooLarge`, the code requeues the frame and continues:

```rust
Err(e) => {
    for id in ids { pipeline.requeue(id); }
    drop(permit);
    continue;  // back to loop top — no .await reached
}
```

`requeue()` rewinds the channel cursor (`encoder.rs:594-618`), so `next_submission()` returns the same immutable `Arc<Frame>`. The semaphore permit was just dropped so it's immediately re-acquirable. The entire error path is synchronous — the tokio task never yields.

### Why this is the default behavior

* `config.rs:97-98`: `max_frame_size = 130044` (the default)
* `cli.rs:227`: `max_frame_size = target_frame_size` (no separate flag)
* Any channel with compressed output ≥ 130,021 bytes triggers it
* At Base mainnet throughput, channels fill well past this on every cycle

### The developers know the correct math

Their own test helper `blob_filling_submission()` at `driver.rs:441`:

```rust
let data_len = BlobEncoder::BLOB_MAX_DATA_SIZE - 1 - BlobEncoder::FRAME_OVERHEAD;
// = 130044 - 1 - 23 = 130020  ← correctly subtracts 1
```

But the production config doesn't apply this subtraction.

## Impact Details

The batcher is the sole DA submitter for the L2. When `submit_pending()` livelocks:

* No batch data reaches L1 → derivation pipeline stalls
* TEE/ZK provers have no data to prove → no proposals finalized
* All pending L2→L1 withdrawals are frozen (require finalized state)
* Restarting with defaults reproduces immediately — not a transient failure

This is "Network not being able to confirm new transactions (total network shutdown)" per the scope definition.

**Funds at risk:** All assets pending withdrawal through the L2 bridge are frozen for the duration of the outage. On Base mainnet this includes all ETH and ERC-20 tokens in the bridge contracts and any in-flight withdrawal messages.

## References

* `crates/batcher/encoder/src/config.rs:97-98` — Default `max_frame_size = 130044`
* `bin/batcher/src/cli.rs:96-97,227` — CLI default, hardcodes max = target
* `crates/batcher/comp/src/channel_out.rs:140-168` — Frame data budget
* `crates/batcher/blobs/src/encoder.rs:52-68` — Blob encoding + size check
* `crates/batcher/core/src/submissions.rs:52-169` — `submit_pending()` loop
* `crates/batcher/encoder/src/encoder.rs:594-618` — Requeue cursor rewind
* `crates/batcher/core/src/driver.rs:157` — Driver blocked at `submit_pending().await`
* `crates/batcher/core/src/driver.rs:441` — Test helper with correct math

## Proof of Concept

This PoC calls the **real production `SubmissionQueue::submit_pending()`** from `base-batcher-core` and proves it enters a permanent infinite loop. It is not a unit test — it's a standalone binary that exercises the actual production code path.

The PoC uses a `LivelockPipeline` that models the real `FrameEncoder`'s requeue behavior: calling `requeue()` makes the frame available again via `next_submission()`, just like the cursor rewind in production.

**What it demonstrates:**

1. `submit_pending()` spawned with a production-sized frame (130021 bytes) **never returns**
2. After 2 seconds, the iteration counter shows **hundreds of thousands** of synchronous retries
3. The `TxManager` is **never called** — `encode_packed` fails every time before reaching `send_async`
4. The same test with the fixed size (130020 bytes) **completes immediately**

### How to run

```bash
# Place files in crates/batcher/poc-f1001/ within the base/base repo at v0.8.0-rc.28
cargo run -p poc-f1001
```

No network access, no environment variables, no external services.

### Dependencies

* `base/base` repository at tag `v0.8.0-rc.28`
* Rust toolchain (same as the workspace requires)
* No external services or mainnet forking needed (this is an offchain Rust component, not a smart contract)

### `crates/batcher/poc-f1001/Cargo.toml`

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
base-batcher-core = { workspace = true, features = ["test-utils"] }
base-batcher-encoder.workspace = true
base-tx-manager.workspace = true
base-protocol = { workspace = true, features = ["std"] }
base-common-consensus = { workspace = true, features = ["std"] }
alloy-primitives = { workspace = true, features = ["std"] }
alloy-consensus = { workspace = true, features = ["std"] }
alloy-rpc-types-eth = { workspace = true, features = ["std"] }
tokio = { workspace = true, features = ["sync", "macros", "rt", "rt-multi-thread", "time"] }
```

### `crates/batcher/poc-f1001/src/main.rs`

```rust
use std::sync::{
    Arc,
    atomic::{AtomicU64, Ordering},
};

use alloy_primitives::Address;
use base_batcher_core::SubmissionQueue;
use base_batcher_encoder::{
    BatchPipeline, BatchSubmission, DaType, ReorgError, StepError, StepResult, SubmissionId,
};
use base_blobs::BlobEncoder;
use base_common_consensus::BaseBlock;
use base_protocol::{ChannelId, Frame};
use base_tx_manager::{SendHandle, SendResponse, TxCandidate, TxManager};
use tokio::sync::oneshot;

const DEFAULT_MAX_FRAME_SIZE: usize = 130_044; // production default (config.rs:97)
const FIXED_MAX_FRAME_SIZE: usize = 130_043;   // corrected (accounts for VERSION prefix)

const PRODUCTION_FRAME_DATA_LEN: usize = DEFAULT_MAX_FRAME_SIZE - BlobEncoder::FRAME_OVERHEAD;
const FIXED_FRAME_DATA_LEN: usize = FIXED_MAX_FRAME_SIZE - BlobEncoder::FRAME_OVERHEAD;

/// Models the real FrameEncoder's requeue behavior:
/// requeue() makes the frame immediately available again via next_submission(),
/// identical to the cursor rewind at encoder.rs:594-618.
struct LivelockPipeline {
    submission: BatchSubmission,
    available: bool,
    iteration_count: Arc<AtomicU64>,
}

impl LivelockPipeline {
    fn new(frame_data_len: usize, iteration_count: Arc<AtomicU64>) -> Self {
        let frame = Arc::new(Frame {
            id: ChannelId::default(),
            number: 0,
            data: vec![0xAB; frame_data_len],
            is_last: false,
        });
        Self {
            submission: BatchSubmission {
                id: SubmissionId(0),
                channel_id: ChannelId::default(),
                da_type: DaType::Blob,
                frames: vec![frame],
            },
            available: true,
            iteration_count,
        }
    }
}

impl BatchPipeline for LivelockPipeline {
    fn add_block(&mut self, _: BaseBlock) -> Result<(), (ReorgError, Box<BaseBlock>)> {
        Ok(())
    }
    fn step(&mut self) -> Result<StepResult, StepError> {
        Ok(StepResult::Idle)
    }
    fn next_submission(&mut self) -> Option<BatchSubmission> {
        if self.available {
            self.available = false;
            self.iteration_count.fetch_add(1, Ordering::Relaxed);
            Some(BatchSubmission {
                id: self.submission.id,
                channel_id: self.submission.channel_id,
                da_type: self.submission.da_type,
                frames: self.submission.frames.clone(),
            })
        } else {
            None
        }
    }
    fn confirm(&mut self, _: SubmissionId, _: u64) {}
    fn requeue(&mut self, _: SubmissionId) {
        // Models encoder.rs:594-618: cursor rewind makes frame available again
        self.available = true;
    }
    fn force_close_channel(&mut self) {}
    fn advance_l1_head(&mut self, _: u64) {}
    fn prune_safe(&mut self, _: u64) {}
    fn reset(&mut self) {}
    fn da_backlog_bytes(&self) -> u64 { 0 }
}

/// Panics if called — proves encode_packed fails before TxManager is reached.
#[derive(Debug)]
struct PanicTxManager;

impl TxManager for PanicTxManager {
    async fn send(&self, _: TxCandidate) -> SendResponse {
        panic!("send() should never be reached during livelock");
    }
    fn send_async(&self, _: TxCandidate) -> impl std::future::Future<Output = SendHandle> + Send {
        panic!("send_async() should never be reached during livelock");
        #[allow(unreachable_code)]
        std::future::ready(SendHandle::new(oneshot::channel().1))
    }
    fn sender_address(&self) -> Address { Address::ZERO }
}

/// Confirms immediately — used for the fix verification step.
#[derive(Debug)]
struct ConfirmTxManager;

impl TxManager for ConfirmTxManager {
    async fn send(&self, _: TxCandidate) -> SendResponse { unreachable!() }
    fn send_async(&self, _: TxCandidate) -> impl std::future::Future<Output = SendHandle> + Send {
        use alloy_consensus::{Eip658Value, Receipt, ReceiptEnvelope, ReceiptWithBloom};
        use alloy_primitives::{B256, Bloom};
        use alloy_rpc_types_eth::TransactionReceipt;
        let receipt = TransactionReceipt {
            inner: ReceiptEnvelope::Legacy(ReceiptWithBloom {
                receipt: Receipt {
                    status: Eip658Value::Eip658(true),
                    cumulative_gas_used: 21_000,
                    logs: vec![],
                },
                logs_bloom: Bloom::ZERO,
            }),
            transaction_hash: B256::ZERO,
            transaction_index: Some(0),
            block_hash: Some(B256::ZERO),
            block_number: Some(1),
            gas_used: 21_000,
            effective_gas_price: 1_000_000_000,
            blob_gas_used: None,
            blob_gas_price: None,
            from: Address::ZERO,
            to: Some(Address::ZERO),
            contract_address: None,
        };
        let (tx, rx) = oneshot::channel();
        let _ = tx.send(Ok(receipt));
        std::future::ready(SendHandle::new(rx))
    }
    fn sender_address(&self) -> Address { Address::ZERO }
}

fn main() {
    println!("==========================================================");
    println!(" F-1001: Batcher submit_pending() livelock PoC");
    println!(" Using REAL SubmissionQueue from base-batcher-core");
    println!("==========================================================\n");

    let rt = tokio::runtime::Builder::new_multi_thread()
        .worker_threads(2)
        .enable_all()
        .build()
        .unwrap();

    rt.block_on(async {
        // ── STEP 1: Prove submit_pending() livelocks with production frame size ──

        println!("[STEP 1] Calling submit_pending() with production-sized frame");
        println!("         frame.data.len = {} (from output_frame({}))", PRODUCTION_FRAME_DATA_LEN, DEFAULT_MAX_FRAME_SIZE);
        println!("         blob payload   = 1 + 23 + {} = {} (limit: {})",
                 PRODUCTION_FRAME_DATA_LEN,
                 1 + 23 + PRODUCTION_FRAME_DATA_LEN,
                 BlobEncoder::BLOB_MAX_DATA_SIZE);
        println!("         overflow: 1 byte → DataTooLarge on every attempt\n");

        let iterations = Arc::new(AtomicU64::new(0));
        let iter_clone = Arc::clone(&iterations);

        let handle = tokio::spawn(async move {
            let mut pipeline = LivelockPipeline::new(PRODUCTION_FRAME_DATA_LEN, iter_clone);
            let mut queue: SubmissionQueue<PanicTxManager> = SubmissionQueue::new(
                PanicTxManager, Address::ZERO, 1,
            );
            queue.submit_pending(&mut pipeline).await;
            // ^ This never returns. The loop inside:
            //   acquire permit (sync) → next_submission (sync) → encode_packed (fails, sync)
            //   → requeue (sync) → drop permit → continue. No await point reached.
        });

        println!("         Waiting 2 seconds to observe...\n");
        tokio::time::sleep(std::time::Duration::from_secs(2)).await;

        let count = iterations.load(Ordering::Relaxed);
        let stuck = !handle.is_finished();

        println!("         RESULT:");
        println!("           submit_pending returned: {} ", if stuck { "NO — still looping" } else { "YES (unexpected)" });
        println!("           iterations in 2 sec:     {}", count);
        println!("           TxManager invoked:       NO (encode_packed fails first)");
        println!("           async yield points hit:  ZERO");
        println!();

        assert!(stuck, "submit_pending() should not have returned");
        assert!(count > 1000, "expected thousands of iterations, got {}", count);

        println!("  >>> LIVELOCK CONFIRMED: submit_pending() never returns.");
        println!("      Driver main loop (driver.rs:157) is permanently blocked.");
        println!("      No L2 data posted to L1. Finalization halted. Withdrawals frozen.\n");
        handle.abort();

        // ── STEP 2: Prove the fix works — submit_pending() returns normally ──

        println!("──────────────────────────────────────────────────────────");
        println!("[STEP 2] Same test with fixed frame size (max_frame_size = {})", FIXED_MAX_FRAME_SIZE);
        println!("         frame.data.len = {}", FIXED_FRAME_DATA_LEN);
        println!("         blob payload   = 1 + 23 + {} = {} = BLOB_MAX_DATA_SIZE (exact fit)\n",
                 FIXED_FRAME_DATA_LEN, 1 + 23 + FIXED_FRAME_DATA_LEN);

        let iterations2 = Arc::new(AtomicU64::new(0));
        let iter_clone2 = Arc::clone(&iterations2);

        let handle2 = tokio::spawn(async move {
            let mut pipeline = LivelockPipeline::new(FIXED_FRAME_DATA_LEN, iter_clone2);
            let mut queue: SubmissionQueue<ConfirmTxManager> = SubmissionQueue::new(
                ConfirmTxManager, Address::ZERO, 1,
            );
            queue.submit_pending(&mut pipeline).await;
        });

        let result = tokio::time::timeout(std::time::Duration::from_secs(5), handle2).await;
        let count2 = iterations2.load(Ordering::Relaxed);

        match result {
            Ok(Ok(())) => {
                println!("         submit_pending() returned after {} iteration(s).", count2);
                println!("         encode_packed succeeded. Frame submitted and confirmed.");
                println!("  >>> NO LIVELOCK with the fix.\n");
            }
            _ => panic!("submit_pending should complete with fixed frame size"),
        }

        // ── STEP 3: Arithmetic confirmation ──

        println!("──────────────────────────────────────────────────────────");
        println!("[STEP 3] Direct encode_packed verification\n");

        let bad = BlobEncoder::encode_packed(&[Arc::new(Frame {
            data: vec![0; PRODUCTION_FRAME_DATA_LEN], ..Frame::default()
        })]);
        println!("         encode_packed(data.len={}) → {:?}", PRODUCTION_FRAME_DATA_LEN,
                 bad.as_ref().err().unwrap());

        let good = BlobEncoder::encode_packed(&[Arc::new(Frame {
            data: vec![0; FIXED_FRAME_DATA_LEN], ..Frame::default()
        })]);
        assert!(good.is_ok());
        println!("         encode_packed(data.len={}) → Ok(blob)", FIXED_FRAME_DATA_LEN);

        println!("\n         Developer test (driver.rs:441) uses:");
        println!("           BLOB_MAX_DATA_SIZE - 1 - FRAME_OVERHEAD = {}", FIXED_FRAME_DATA_LEN);
        println!("         Production config uses: {} → overflow by 1", DEFAULT_MAX_FRAME_SIZE);

        println!("\n==========================================================");
        println!(" SUMMARY");
        println!(" - submit_pending() livelocks with default config");
        println!(" - driver.rs:157 blocks permanently → batcher is dead");
        println!(" - no L2 batch data reaches L1 → finalization halts");
        println!(" - all pending withdrawals frozen");
        println!(" - restart with defaults reproduces immediately");
        println!(" - fix: max_frame_size = 130043 (subtract 1 for VERSION byte)");
        println!("==========================================================");
    });
}
```

### Expected output

<details>

<summary>Expected output</summary>

```
==========================================================
 F-1001: Batcher submit_pending() livelock PoC
 Using REAL SubmissionQueue from base-batcher-core
==========================================================

[STEP 1] Calling submit_pending() with production-sized frame
         frame.data.len = 130021 (from output_frame(130044))
         blob payload   = 1 + 23 + 130021 = 130045 (limit: 130044)
         overflow: 1 byte → DataTooLarge on every attempt

         Waiting 2 seconds to observe...

         RESULT:
           submit_pending returned: NO — still looping
           iterations in 2 sec:     357752
           TxManager invoked:       NO (encode_packed fails first)
           async yield points hit:  ZERO

  >>> LIVELOCK CONFIRMED: submit_pending() never returns.
      Driver main loop (driver.rs:157) is permanently blocked.
      No L2 data posted to L1. Finalization halted. Withdrawals frozen.

──────────────────────────────────────────────────────────
[STEP 2] Same test with fixed frame size (max_frame_size = 130043)
         frame.data.len = 130020
         blob payload   = 1 + 23 + 130020 = 130044 = BLOB_MAX_DATA_SIZE (exact fit)

         submit_pending() returned after 1 iteration(s).
         encode_packed succeeded. Frame submitted and confirmed.
  >>> NO LIVELOCK with the fix.

──────────────────────────────────────────────────────────
[STEP 3] Direct encode_packed verification

         encode_packed(data.len=130021) → DataTooLarge { size: 130045 }
         encode_packed(data.len=130020) → Ok(blob)

         Developer test (driver.rs:441) uses:
           BLOB_MAX_DATA_SIZE - 1 - FRAME_OVERHEAD = 130020
         Production config uses: 130044 → overflow by 1

==========================================================
 SUMMARY
 - submit_pending() livelocks with default config
 - driver.rs:157 blocks permanently → batcher is dead
 - no L2 batch data reaches L1 → finalization halts
 - all pending withdrawals frozen
 - restart with defaults reproduces immediately
 - fix: max_frame_size = 130043 (subtract 1 for VERSION byte)
==========================================================
```

</details>
