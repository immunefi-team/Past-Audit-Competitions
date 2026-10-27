# 75031 bc medium default blob frame sizing can create unencodable frames causing retry storms and 30 batcher resource amplification

**Submitted on Apr 26th 2026 at 19:56:52 UTC by @OxPrince for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75031
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.24>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

### Brief/Intro

The batcher can emit a blob frame that is valid according to the frame-size limit but impossible to pack into a single EIP-4844 blob after the submission layer adds its own derivation/frame overhead. When this one-byte-oversized frame reaches `SubmissionQueue::submit_pending`, blob encoding fails before any transaction is sent, the same frame is requeued, and the loop immediately retries it without backoff or quarantine. In production this can make the batcher repeatedly spend CPU on an unencodable frame and stop progressing normal L1 submissions, causing resource amplification and backlog growth.

### Vulnerability Details

The bug is an off-by-one mismatch between:

* the maximum frame payload emitted by `ChannelOut::output_frame`;
* the maximum payload accepted by the blob encoder;
* the extra wrapper bytes added when a frame is packed into a blob transaction.

The default batcher configuration uses blob DA and a `130,044` byte target frame size:

```rust
// crates/batcher/encoder/src/config.rs
impl Default for EncoderConfig {
    fn default() -> Self {
        Self {
            target_frame_size: 130_044,
            max_frame_size: 130_044,
            target_num_frames: 1,
            da_type: DaType::Blob,
            // ...
        }
    }
}
```

The CLI exposes the same defaults:

```rust
// bin/batcher/src/cli.rs
#[arg(long = "target-frame-size", default_value = "130044")]
pub target_frame_size: usize;

#[arg(long = "data-availability-type", default_value = "blobs")]
da_type: base_batcher_encoder::DaType;
```

The blob encoder also has a maximum payload size of `130,044` bytes:

```rust
// crates/batcher/blobs/src/encoder.rs
pub const BLOB_MAX_DATA_SIZE: usize = (4 * 31 + 3) * 1024 - 4; // 130_044
pub const FRAME_OVERHEAD: usize = 23;
```

When a channel is closed, `ChannelOut::output_frame` subtracts the frame metadata overhead from `max_frame_size`. The first frame also reserves one byte for the compression version. Later frames do not reserve this byte:

```rust
// crates/batcher/comp/src/channel_out.rs
let version_byte =
    if self.frame_number == 0 { self.compressor.channel_version_byte() } else { None };
let prefix_len = usize::from(version_byte.is_some());

let max_size = (max_size - FRAME_V0_OVERHEAD - prefix_len).min(self.ready_bytes());
```

Therefore, with default settings:

* first frame data length: `130,044 - 23 - 1 = 130,020`;
* non-first frame data length: `130,044 - 23 = 130,021`.

The first frame still fits into one blob transaction:

```
1 byte derivation version + 23 bytes frame overhead + 130,020 bytes data = 130,044
```

The non-first full frame does not fit:

```
1 byte derivation version + 23 bytes frame overhead + 130,021 bytes data = 130,045
```

That is one byte above `BlobEncoder::BLOB_MAX_DATA_SIZE`.

The submission loop does not reject an oversized first submission before attempting blob encoding:

```rust
// crates/batcher/core/src/submissions.rs
while let Some(sub) = pipeline.next_submission() {
    let sub_frame_size: usize =
        sub.frames.iter().map(|f| BlobEncoder::FRAME_OVERHEAD + f.data.len()).sum();

    if !ids.is_empty()
        && payload_size + sub_frame_size > BlobEncoder::BLOB_MAX_DATA_SIZE
    {
        pipeline.requeue(sub.id);
        break;
    }

    // ids is empty, so the oversized first submission is accepted here.
    payload_size += sub_frame_size;
    ids.push(sub.id);
    frames.extend(sub.frames);
}

let candidate = match da_type {
    DaType::Blob => match BlobEncoder::encode_packed(&frames) {
        Ok(blob) => { /* send tx */ }
        Err(e) => {
            warn!(error = %e, "failed to encode frames to blob, requeueing");
            for id in ids {
                pipeline.requeue(id);
            }
            drop(permit);
            continue;
        }
    },
    // ...
};
```

The problematic behavior is the `Err(e) => ... continue` path. For an unencodable frame, requeueing does not change the frame size. The next loop iteration acquires the permit again, dequeues the same frame again, attempts to encode the same impossible blob payload again, fails again, and repeats.

No L1 transaction is submitted in this path, so no receipt, confirmation, failure, or txpool state can break the loop. There is also no retry counter, sleep, backoff, or failed-frame quarantine.

### Reachability

The issue is reachable with default blob settings. It does not require an invalid configuration.

The default encoder can produce the problematic non-first full frame when a channel fragments into multiple frames. A large poorly-compressible L2 transaction payload is enough to create this situation. This is realistic for calldata-heavy or high-entropy transaction input.

The following reachability PoC proves this:

```rust
#[test]
fn poc_default_framing_emits_blob_oversized_non_first_frame() {
    const FRAME_V0_OVERHEAD: usize = 23;
    const NON_FIRST_FRAME_DATA_BYTES: usize = 130_044 - FRAME_V0_OVERHEAD;

    let mut encoder = default_encoder();
    encoder.add_block(make_block_with_large_user_tx(B256::ZERO, 400_000)).unwrap();

    assert_eq!(encoder.step().unwrap(), StepResult::BlockEncoded);
    encoder.force_close_channel();

    let channel = encoder.ready_channels.front().expect("channel must be ready");
    assert!(channel.frames.len() > 1, "large block must fragment into multiple frames");
    assert_eq!(channel.frames[1].data.len(), NON_FIRST_FRAME_DATA_BYTES);
}
```

This confirms that normal default framing can emit a non-first frame with `130,021` bytes of data.

## Impact Details

**Medium: Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours**

The resource increase comes from repeated blob-encoding attempts against a frame that can never fit. The batcher repeatedly:

1. dequeues the same frame;
2. builds a blob payload;
3. calls `BlobEncoder::encode_packed`;
4. receives `DataTooLarge`;
5. requeues the frame;
6. immediately repeats the same work.

This is not brute force in the sense of many external requests or repeated network calls. The amplification is internal: one reachable bad frame causes the batcher to spin in its own submission loop.

This is far above the `30%` Medium threshold. In the real `BatchEncoder`/`SubmissionQueue` path, the same frame remains available after each requeue, and there is no built-in maximum retry count on this encode-failure path.

Operationally, this can cause:

* elevated CPU consumption in the batcher process;
* failure to make progress on L1 batch submission while the impossible frame remains at the front of the queue;
* backlog growth behind the stuck frame;
* partial node/operator degradation during calldata-heavy or high-entropy traffic.

## References

* `crates/batcher/encoder/src/config.rs`: default `target_frame_size = 130_044`, `max_frame_size = 130_044`, `da_type = Blob`.
* `bin/batcher/src/cli.rs`: CLI defaults for `--target-frame-size=130044` and `--data-availability-type=blobs`.
* `crates/batcher/comp/src/channel_out.rs`: non-first frames reserve no compression-version byte, allowing `130,021` bytes of frame data under default frame sizing.
* `crates/batcher/blobs/src/encoder.rs`: blob payload limit is `130,044`; blob frame packing adds `1 + 23` bytes of overhead.
* `crates/batcher/core/src/submissions.rs`: oversized first submission is accepted, `encode_packed` fails, the same frame is requeued, and the loop immediately continues.
* `crates/batcher/encoder/src/encoder.rs`: `poc_default_framing_emits_blob_oversized_non_first_frame`.
* `crates/batcher/core/src/submissions.rs`: `poc_oversized_blob_frame_requeues_without_backoff` and `poc_oversized_blob_frame_resource_amplification`.

## Proof of Concept

Main poc: `poc_oversized_blob_frame_resource_amplification`

<details>

<summary>Open proof of concept</summary>

```rust
//! Submission lifecycle management for the batch driver.

use std::{future::Future, pin::Pin, sync::Arc};

use alloy_primitives::{Address, Bytes, U256};
use base_batcher_encoder::{BatchPipeline, BatcherMetrics, DaType, FrameEncoder, SubmissionId};
use base_blobs::BlobEncoder;
use base_tx_manager::{TxCandidate, TxManager, TxManagerError};
use futures::stream::{FuturesUnordered, StreamExt};
use tokio::sync::Semaphore;
use tracing::{info, warn};

use crate::TxOutcome;

/// Type alias for the in-flight receipt future collection.
type InFlight =
    FuturesUnordered<Pin<Box<dyn Future<Output = (Vec<SubmissionId>, TxOutcome)> + Send>>>;

/// Manages the full submission lifecycle for the batch driver.
///
/// Owns capacity management (semaphore), in-flight receipt tracking
/// ([`FuturesUnordered`]), txpool blockage state, the [`TxManager`], and the
/// batcher inbox address. These were previously loose fields on [`BatchDriver`].
#[derive(Debug)]
pub struct SubmissionQueue<TM: TxManager> {
    tx_manager: TM,
    in_flight: InFlight,
    semaphore: Arc<Semaphore>,
    inbox: Address,
    txpool_blocked: bool,
}

impl<TM: TxManager> SubmissionQueue<TM> {
    /// Create a new [`SubmissionQueue`].
    pub fn new(tx_manager: TM, inbox: Address, max_pending: usize) -> Self {
        Self {
            tx_manager,
            in_flight: FuturesUnordered::new(),
            semaphore: Arc::new(Semaphore::new(max_pending)),
            inbox,
            txpool_blocked: false,
        }
    }

    /// Submit all ready frames that fit within semaphore capacity.
    ///
    /// For each available semaphore permit (= one L1 transaction), packs as many
    /// pending frames as fit into a single blob payload (up to
    /// [`BlobEncoder::BLOB_MAX_DATA_SIZE`] bytes), then submits one L1 tx carrying
    /// that blob. Loops until the semaphore is exhausted, the pipeline has no
    /// ready submissions, or the txpool is blocked.
    pub async fn submit_pending<P: BatchPipeline>(&mut self, pipeline: &mut P) {
        loop {
            if self.txpool_blocked {
                break;
            }
            let Ok(permit) = Arc::clone(&self.semaphore).try_acquire_owned() else {
                break;
            };

            // Collect as many submissions as fit into one blob payload.
            // payload_size tracks: 1 (DERIVATION_VERSION_0) + sum of frame.encode() sizes.
            let mut ids: Vec<SubmissionId> = Vec::new();
            let mut frames = Vec::new();
            let mut payload_size: usize = 1; // DERIVATION_VERSION_0 prefix
            let mut frame_bytes: usize = 0;
            let mut da_type = DaType::Blob;

            while let Some(sub) = pipeline.next_submission() {
                // Calculate the encoded byte cost of this submission's frames.
                let sub_frame_size: usize =
                    sub.frames.iter().map(|f| BlobEncoder::FRAME_OVERHEAD + f.data.len()).sum();

                // If the blob already has at least one submission and this one doesn't fit,
                // put it back and stop packing.
                if !ids.is_empty()
                    && payload_size + sub_frame_size > BlobEncoder::BLOB_MAX_DATA_SIZE
                {
                    pipeline.requeue(sub.id);
                    break;
                }

                if ids.is_empty() {
                    da_type = sub.da_type;
                } else if sub.da_type != da_type {
                    pipeline.requeue(sub.id);
                    break;
                }
                frame_bytes += sub.frames.iter().map(|f| f.data.len()).sum::<usize>();
                payload_size += sub_frame_size;
                ids.push(sub.id);
                frames.extend(sub.frames);

                // Calldata mode: exactly one frame per L1 transaction (protocol requirement).
                if matches!(da_type, DaType::Calldata) {
                    break;
                }
            }

            if ids.is_empty() {
                drop(permit);
                break;
            }

            let da_type_label = match da_type {
                DaType::Blob => BatcherMetrics::DA_TYPE_BLOB,
                DaType::Calldata => BatcherMetrics::DA_TYPE_CALLDATA,
            };
            let candidate = match da_type {
                DaType::Blob => match BlobEncoder::encode_packed(&frames) {
                    Ok(blob) => TxCandidate {
                        to: Some(self.inbox),
                        tx_data: Bytes::new(),
                        value: U256::ZERO,
                        gas_limit: 0,
                        blobs: Arc::from(vec![blob]),
                    },
                    Err(e) => {
                        warn!(error = %e, "failed to encode frames to blob, requeueing");
                        for id in ids {
                            pipeline.requeue(id);
                        }
                        drop(permit);
                        continue;
                    }
                },
                DaType::Calldata => TxCandidate {
                    to: Some(self.inbox),
                    tx_data: FrameEncoder::to_calldata(&frames[0]),
                    value: U256::ZERO,
                    gas_limit: 0,
                    blobs: vec![].into(),
                },
            };
            info!(
                submissions = %ids.len(),
                da_type = %da_type_label,
                frame_bytes = %frame_bytes,
                "submitting packed batch frames to L1"
            );
            BatcherMetrics::submission_total(BatcherMetrics::OUTCOME_SUBMITTED)
                .increment(ids.len() as u64);
            BatcherMetrics::da_bytes_submitted_total(da_type_label).increment(frame_bytes as u64);
            BatcherMetrics::in_flight_submissions().increment(1.0);
            let handle = self.tx_manager.send_async(candidate).await;
            let fut: Pin<Box<dyn Future<Output = (Vec<SubmissionId>, TxOutcome)> + Send>> =
                Box::pin(async move {
                    let outcome = match handle.await {
                        Ok(receipt) => {
                            let l1_block = receipt.block_number.unwrap_or_else(|| {
                                warn!("confirmed receipt missing block number; l1_head will not advance");
                                0
                            });
                            TxOutcome::Confirmed { l1_block }
                        }
                        Err(TxManagerError::AlreadyReserved) => {
                            warn!("txpool nonce slot already reserved");
                            TxOutcome::TxpoolBlocked
                        }
                        Err(e) => {
                            warn!(error = %e, "submission failed");
                            TxOutcome::Failed
                        }
                    };
                    drop(permit);
                    (ids, outcome)
                });
            self.in_flight.push(fut);
        }
    }

    /// Attempt to clear a txpool blockage by cancelling the stuck transaction.
    ///
    /// No-op if the txpool is not currently blocked. On success, clears the
    /// blocked flag so submission can resume.
    pub async fn recover_txpool(&mut self) {
        if !self.txpool_blocked {
            return;
        }
        match self.tx_manager.cancel_tx().await {
            Ok(()) => {
                self.txpool_blocked = false;
                info!("txpool unblocked after cancellation tx");
            }
            Err(e) => {
                warn!(error = %e, "cancel_tx failed, txpool remains blocked");
            }
        }
    }

    /// Handle a settled in-flight receipt.
    ///
    /// On confirmation, calls `pipeline.confirm` for each packed submission and
    /// `pipeline.advance_l1_head` once. On failure, requeues all. On txpool
    /// blockage, requeues all and sets the blocked flag.
    pub fn handle_outcome<P: BatchPipeline>(
        &mut self,
        pipeline: &mut P,
        ids: Vec<SubmissionId>,
        outcome: TxOutcome,
    ) {
        BatcherMetrics::in_flight_submissions().decrement(1.0);
        match outcome {
            TxOutcome::Confirmed { l1_block } => {
                for id in &ids {
                    pipeline.confirm(*id, l1_block);
                }
                pipeline.advance_l1_head(l1_block);
                BatcherMetrics::submission_total(BatcherMetrics::OUTCOME_CONFIRMED)
                    .increment(ids.len() as u64);
                info!(submissions = %ids.len(), l1_block = %l1_block, "submission confirmed on L1");
            }
            TxOutcome::Failed => {
                let count = ids.len();
                for id in ids {
                    pipeline.requeue(id);
                }
                BatcherMetrics::submission_total(BatcherMetrics::OUTCOME_FAILED)
                    .increment(count as u64);
                warn!(submissions = %count, "submission failed, requeued for retry");
            }
            TxOutcome::TxpoolBlocked => {
                let count = ids.len();
                for id in ids {
                    pipeline.requeue(id);
                }
                self.txpool_blocked = true;
                BatcherMetrics::submission_total(BatcherMetrics::OUTCOME_REQUEUED)
                    .increment(count as u64);
                warn!(submissions = %count, "submission blocked by txpool nonce slot, requeued");
            }
        }
    }

    /// Drain all in-flight futures up to the given deadline.
    ///
    /// Confirmed receipts call `pipeline.confirm` + `pipeline.advance_l1_head`.
    /// Failed or txpool-blocked submissions are logged and abandoned — no requeue
    /// because the process is shutting down.
    pub async fn drain<P: BatchPipeline>(
        &mut self,
        pipeline: &mut P,
        mut timeout_fut: Pin<Box<dyn Future<Output = ()> + Send>>,
    ) {
        loop {
            if self.in_flight.is_empty() {
                break;
            }
            tokio::select! {
                _ = &mut timeout_fut => {
                    warn!(remaining = %self.in_flight.len(), "drain timeout reached, abandoning in-flight submissions");
                    break;
                }
                Some((ids, outcome)) = self.in_flight.next() => {
                    BatcherMetrics::in_flight_submissions().decrement(1.0);
                    match outcome {
                        TxOutcome::Confirmed { l1_block } => {
                            for id in &ids {
                                pipeline.confirm(*id, l1_block);
                            }
                            pipeline.advance_l1_head(l1_block);
                            BatcherMetrics::submission_total(BatcherMetrics::OUTCOME_CONFIRMED).increment(ids.len() as u64);
                            info!(submissions = %ids.len(), l1_block = %l1_block, "submission confirmed on L1 during drain");
                        }
                        TxOutcome::Failed => {
                            BatcherMetrics::submission_total(BatcherMetrics::OUTCOME_FAILED).increment(ids.len() as u64);
                            warn!(submissions = %ids.len(), "submission failed during drain, abandoning");
                        }
                        TxOutcome::TxpoolBlocked => {
                            BatcherMetrics::submission_total(BatcherMetrics::OUTCOME_REQUEUED).increment(ids.len() as u64);
                            warn!(submissions = %ids.len(), "submission txpool-blocked during drain, abandoning");
                        }
                    }
                }
            }
        }
    }

    /// Discard all in-flight futures, returning their semaphore permits.
    ///
    /// Used on reorg to prevent stale completions from modifying the freshly
    /// reset pipeline.
    pub fn discard(&mut self) {
        let discarded = self.in_flight.len();
        if discarded > 0 {
            warn!(discarded = %discarded, "discarding in-flight submissions due to reorg");
            BatcherMetrics::in_flight_submissions().set(0.0);
        }
        self.in_flight = FuturesUnordered::new();
    }

    /// Returns a future for the next settled `(ids, outcome)` pair.
    ///
    /// Resolves immediately to `None` when in-flight is empty; safe to use as
    /// a `select!` arm with a `Some(...)` pattern guard.
    pub fn next_settled(
        &mut self,
    ) -> impl Future<Output = Option<(Vec<SubmissionId>, TxOutcome)>> + '_ {
        self.in_flight.next()
    }

    /// Returns the number of currently in-flight submissions.
    pub fn in_flight_count(&self) -> usize {
        self.in_flight.len()
    }
}

#[cfg(test)]
mod tests {
    use std::{
        sync::{Arc, Mutex},
        time::{Duration, Instant},
    };

    use alloy_primitives::Address;
    use base_batcher_encoder::{
        BatchSubmission, DaType, ReorgError, StepError, StepResult, SubmissionId,
    };
    use base_common_consensus::BaseBlock;
    use base_protocol::{ChannelId, Frame};
    use base_tx_manager::{SendHandle, SendResponse, TxCandidate, TxManager};
    use tokio::sync::oneshot;

    use super::*;

    #[derive(Debug, Default)]
    struct PipelineCounters {
        next_submissions: usize,
        requeues: usize,
    }

    #[derive(Debug)]
    struct OneFrameRetryPipeline {
        counters: Arc<Mutex<PipelineCounters>>,
        available: bool,
        frame: Arc<Frame>,
        max_requeues: usize,
    }

    impl OneFrameRetryPipeline {
        fn new(
            counters: Arc<Mutex<PipelineCounters>>,
            frame_data_len: usize,
            max_requeues: usize,
        ) -> Self {
            Self {
                counters,
                available: true,
                frame: Arc::new(Frame {
                    data: vec![0; frame_data_len],
                    ..Frame::default()
                }),
                max_requeues,
            }
        }
    }

    impl BatchPipeline for OneFrameRetryPipeline {
        fn add_block(&mut self, _: BaseBlock) -> Result<(), (ReorgError, Box<BaseBlock>)> {
            Ok(())
        }

        fn step(&mut self) -> Result<StepResult, StepError> {
            Ok(StepResult::Idle)
        }

        fn next_submission(&mut self) -> Option<BatchSubmission> {
            self.counters.lock().unwrap().next_submissions += 1;
            if !self.available {
                return None;
            }
            self.available = false;

            Some(BatchSubmission {
                id: SubmissionId(0),
                channel_id: ChannelId::default(),
                da_type: DaType::Blob,
                frames: vec![Arc::clone(&self.frame)],
            })
        }

        fn confirm(&mut self, _: SubmissionId, _: u64) {}

        fn requeue(&mut self, _: SubmissionId) {
            let mut counters = self.counters.lock().unwrap();
            counters.requeues += 1;
            self.available = counters.requeues < self.max_requeues;
        }

        fn force_close_channel(&mut self) {}
        fn advance_l1_head(&mut self, _: u64) {}
        fn prune_safe(&mut self, _: u64) {}
        fn reset(&mut self) {}

        fn da_backlog_bytes(&self) -> u64 {
            0
        }
    }

    #[derive(Debug, Default)]
    struct RecordingTxManager {
        sends: Arc<Mutex<usize>>,
    }

    impl TxManager for RecordingTxManager {
        async fn send(&self, _: TxCandidate) -> SendResponse {
            unreachable!("oversized blob payload must fail before tx submission")
        }

        fn send_async(&self, _: TxCandidate) -> impl Future<Output = SendHandle> + Send {
            *self.sends.lock().unwrap() += 1;
            let (tx, rx) = oneshot::channel();
            drop(tx);
            std::future::ready(SendHandle::new(rx))
        }

        fn sender_address(&self) -> Address {
            Address::ZERO
        }
    }

    #[derive(Debug)]
    struct ResourceSample {
        elapsed: Duration,
        next_submissions: usize,
        requeues: usize,
        sends: usize,
        encoded_payload_bytes: usize,
    }

    impl ResourceSample {
        const fn operation_count(&self) -> usize {
            self.next_submissions + self.requeues + self.sends
        }
    }

    const fn valid_max_blob_frame_data_len() -> usize {
        BlobEncoder::BLOB_MAX_DATA_SIZE - BlobEncoder::FRAME_OVERHEAD - 1
    }

    const fn oversized_blob_frame_data_len() -> usize {
        // This is the size produced by a full non-first frame when max_frame_size is
        // the default 130_044: 130_044 - FRAME_OVERHEAD = 130_021.
        // Blob packing adds DERIVATION_VERSION_0 plus FRAME_OVERHEAD, so the
        // payload is 130_045 bytes and cannot fit in one blob.
        BlobEncoder::BLOB_MAX_DATA_SIZE - BlobEncoder::FRAME_OVERHEAD
    }

    const fn blob_payload_bytes(frame_data_len: usize) -> usize {
        1 + BlobEncoder::FRAME_OVERHEAD + frame_data_len
    }

    async fn run_resource_case(frame_data_len: usize, max_requeues: usize) -> ResourceSample {
        let counters = Arc::new(Mutex::new(PipelineCounters::default()));
        let sends = Arc::new(Mutex::new(0));
        let mut pipeline =
            OneFrameRetryPipeline::new(Arc::clone(&counters), frame_data_len, max_requeues);
        let mut queue = SubmissionQueue::new(
            RecordingTxManager { sends: Arc::clone(&sends) },
            Address::ZERO,
            1,
        );

        let start = Instant::now();
        queue.submit_pending(&mut pipeline).await;
        let elapsed = start.elapsed();

        let counters = counters.lock().unwrap();
        let sends = *sends.lock().unwrap();
        let encode_attempts = counters.requeues + sends;
        ResourceSample {
            elapsed,
            next_submissions: counters.next_submissions,
            requeues: counters.requeues,
            sends,
            encoded_payload_bytes: encode_attempts * blob_payload_bytes(frame_data_len),
        }
    }

    #[tokio::test]
    async fn poc_oversized_blob_frame_requeues_without_backoff() {
        const MAX_REQUEUES: usize = 8;

        let counters = Arc::new(Mutex::new(PipelineCounters::default()));
        let sends = Arc::new(Mutex::new(0));
        let mut pipeline = OneFrameRetryPipeline::new(
            Arc::clone(&counters),
            oversized_blob_frame_data_len(),
            MAX_REQUEUES,
        );
        let mut queue = SubmissionQueue::new(
            RecordingTxManager { sends: Arc::clone(&sends) },
            Address::ZERO,
            1,
        );

        queue.submit_pending(&mut pipeline).await;

        assert_eq!(
            counters.lock().unwrap().requeues,
            MAX_REQUEUES,
            "the same unencodable frame was requeued repeatedly in one submit_pending call"
        );
        assert_eq!(
            *sends.lock().unwrap(),
            0,
            "oversized payload never reaches the tx manager, so receipts cannot break the loop"
        );
    }

    #[tokio::test]
    async fn poc_oversized_blob_frame_resource_amplification() {
        const MAX_REQUEUES: usize = 512;

        let normal = run_resource_case(valid_max_blob_frame_data_len(), 0).await;
        let oversized = run_resource_case(oversized_blob_frame_data_len(), MAX_REQUEUES).await;

        assert_eq!(normal.sends, 1, "valid max-size frame must submit once");
        assert_eq!(normal.requeues, 0, "valid max-size frame must not requeue");
        assert_eq!(oversized.sends, 0, "oversized frame never reaches the tx manager");
        assert_eq!(
            oversized.requeues, MAX_REQUEUES,
            "oversized frame is retried repeatedly in one submit_pending call"
        );

        assert!(
            oversized.operation_count() * 10 >= normal.operation_count() * 13,
            "oversized path should exceed the 30% resource-increase threshold: normal={normal:?}, oversized={oversized:?}"
        );
        assert!(
            oversized.encoded_payload_bytes * 10 >= normal.encoded_payload_bytes * 13,
            "oversized path should exceed the 30% encoded-byte work threshold: normal={normal:?}, oversized={oversized:?}"
        );

        println!("normal={normal:?}");
        println!("oversized={oversized:?}");
        println!(
            "elapsed: normal={:?}, oversized={:?}",
            normal.elapsed, oversized.elapsed
        );
    }
}

```

</details>
