# 74742 bc low blob mode frame size mismatch can stall l1 blob publication before submission

**Submitted on Apr 24th 2026 at 15:57:29 UTC by @y4y for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74742
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.24>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

## Description

### Brief/Intro

The default blob-mode frame sizing allows a non-first frame whose serialized size already consumes the full `130044`-byte blob payload budget. Blob packing then adds the derivation prefix byte, producing a `130045`-byte payload. The batcher only detects that overflow when `BlobEncoder::encode_packed()` runs, after the frame has already been selected as the first submission in an L1 blob candidate. No new L1 tx is created for that frame; it is requeued and retried indefinitely.

### Vulnerability Details

The default encoder configuration makes the blob frame ceiling equal to the blob payload ceiling:

```rust
// base/crates/batcher/encoder/src/config.rs
impl Default for EncoderConfig {
    fn default() -> Self {
        Self {
            target_frame_size: 130_044,
            max_frame_size: 130_044,
            max_channel_duration: 2,
            sub_safety_margin: 0,
            target_num_frames: 1,
            batch_type: BatchType::Single,
            da_type: DaType::Blob,
            approx_compr_ratio: 0.6,
            max_l1_tx_size_bytes: None,
        }
    }
}
```

When a channel closes, the encoder drains frames using `self.config.max_frame_size` without reserving space for the blob derivation prefix:

```rust
// base/crates/batcher/encoder/src/encoder.rs
while open.out.ready_bytes() > 0 {
    match open.out.output_frame(self.config.max_frame_size) {
        Ok(frame) => frames.push(Arc::new(frame)),
        Err(e) => {
            warn!(error = %e, "failed to output frame during channel close");
            break;
        }
    }
}
```

`ChannelOut::output_frame()` subtracts the frame envelope and, only for the first frame, the compression-version byte. A non-first frame therefore carries `max_frame_size - 23` bytes of compressed data:

```rust
// base/crates/batcher/comp/src/channel_out.rs
let version_byte =
    if self.frame_number == 0 { self.compressor.channel_version_byte() } else { None };
let prefix_len = usize::from(version_byte.is_some());

let max_size = (max_size - FRAME_V0_OVERHEAD - prefix_len).min(self.ready_bytes());
```

Blob packing adds the derivation prefix byte and the per-frame envelope:

```rust
// base/crates/batcher/blobs/src/encoder.rs
pub const BLOB_MAX_DATA_SIZE: usize = (4 * 31 + 3) * 1024 - 4; // 130_044
pub const FRAME_OVERHEAD: usize = 23;

pub fn encode_packed(frames: &[Arc<Frame>]) -> Result<Box<Blob>, BlobEncodeError> {
    let encoded_size: usize = frames.iter().map(|f| Self::FRAME_OVERHEAD + f.data.len()).sum();
    let mut data = Vec::with_capacity(1 + encoded_size);
    data.push(DERIVATION_VERSION_0);
    for frame in frames {
        data.extend_from_slice(&frame.encode());
    }
    Self::encode(&data)
}

pub fn encode(data: &[u8]) -> Result<Box<Blob>, BlobEncodeError> {
    if data.len() > Self::BLOB_MAX_DATA_SIZE {
        return Err(BlobEncodeError::DataTooLarge { size: data.len() });
    }
    ...
}
```

Concrete example:

* default `max_frame_size = 130044`
* non-first frame data bytes = `130044 - 23 = 130021`
* serialized frame size = `23 + 130021 = 130044`
* blob payload size = `1 + 23 + 130021 = 130045`
* blob max payload size = `130044`

So the payload is deterministically one byte too large.

The submission queue does not reject an oversized first submission before blob encoding. The pre-pack check only runs once the candidate already contains at least one submission:

```rust
// base/crates/batcher/core/src/submissions.rs
while let Some(sub) = pipeline.next_submission() {
    let sub_frame_size: usize =
        sub.frames.iter().map(|f| BlobEncoder::FRAME_OVERHEAD + f.data.len()).sum();

    if !ids.is_empty()
        && payload_size + sub_frame_size > BlobEncoder::BLOB_MAX_DATA_SIZE
    {
        pipeline.requeue(sub.id);
        break;
    }

    payload_size += sub_frame_size;
    ids.push(sub.id);
    frames.extend(sub.frames);
}

let candidate = match da_type {
    DaType::Blob => match BlobEncoder::encode_packed(&frames) {
        Ok(blob) => TxCandidate { ... },
        Err(e) => {
            for id in ids {
                pipeline.requeue(id);
            }
            drop(permit);
            continue;
        }
    },
    ...
};
```

That produces the observed loop:

```
failed to encode frames to blob; requeueing without L1 tx submission
error=data too large: 130045 bytes exceeds maximum 130044
submissions=1
frames=1
blob_payload_size=130045
blob_max_data_size=130044
first_frame_number=Some(0)
first_frame_data_len=Some(130021)
```

This is a pre-broadcast failure. It is not a malformed L1 blob tx that gets sent and later ignored. No new L1 tx hash is created for the oversized frame.

The pre-condition for such issue to happen is:

* Blob DA mode is active.
* The effective frame ceiling allows a serialized frame of `130044` bytes (`target_frame_size = 130044` by default, and CLI wiring maps it to `max_frame_size`).
* A channel produces enough compressed bytes after its first frame to create a full non-first frame.
* That frame is selected as the first submission in an empty blob candidate.

### Impact Details

The batcher can enter a deterministic encode-fail/requeue loop and stop making DA publication progress for the affected channel. Unsafe L2 blocks behind that channel are not published to L1 until the operator changes configuration or a code fix is deployed. Restarting with the same configuration does not solve the issue, because the same frame is derived again and fails at the same boundary.

This is operationally severe because the sequencer can continue accepting L2 transactions while the L1 publication path is stalled for that channel.

### References

* base/crates/batcher/encoder/src/config.rs:94-134
* base/crates/batcher/encoder/src/encoder.rs:229-239
* base/crates/batcher/comp/src/channel\_out.rs:140-171
* base/crates/batcher/blobs/src/encoder.rs:30-68
* base/crates/batcher/core/src/submissions.rs:61-133

## Proof of Concept

There are multiple changes to the codebase:\
This change improves observability only. It does not tighten limits or alter blob packing behavior.

```diff
diff --git a/base/crates/batcher/core/src/submissions.rs b/base/crates/batcher/core/src/submissions.rs
--- a/base/crates/batcher/core/src/submissions.rs
+++ b/base/crates/batcher/core/src/submissions.rs
@@
-                    Err(e) => {
-                        warn!(error = %e, "failed to encode frames to blob, requeueing");
+                    Err(e) => {
+                        warn!(
+                            error = %e,
+                            submissions = %ids.len(),
+                            frames = %frames.len(),
+                            blob_payload_size = %payload_size,
+                            blob_max_data_size = %BlobEncoder::BLOB_MAX_DATA_SIZE,
+                            first_frame_number = ?frames.first().map(|f| f.number),
+                            first_frame_data_len = ?frames.first().map(|f| f.data.len()),
+                            "failed to encode frames to blob; requeueing without L1 tx submission"
+                        );
                         for id in ids {
                             pipeline.requeue(id);
                         }
```

***

File: `base/crates/batcher/core/tests/h01_blob_frame_size_poc.rs`

```rust
//! PoC for H-01: a production-sized non-first blob frame is too large once the
//! blob derivation prefix is added, so the submission loop requeues it forever
//! before any L1 transaction is handed to the tx manager.

use std::{
    future::Future,
    panic::AssertUnwindSafe,
    sync::{
        Arc,
        atomic::{AtomicUsize, Ordering},
    },
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
use futures::FutureExt;
use tokio::sync::oneshot;

#[derive(Debug, Default, Clone)]
struct RecordingTxManager {
    sends: Arc<AtomicUsize>,
}

impl TxManager for RecordingTxManager {
    fn send(&self, _: TxCandidate) -> impl Future<Output = SendResponse> + Send {
        async { unreachable!("the PoC only exercises send_async") }
    }

    fn send_async(&self, _: TxCandidate) -> impl Future<Output = SendHandle> + Send {
        self.sends.fetch_add(1, Ordering::SeqCst);
        let (_tx, rx) = oneshot::channel();
        std::future::ready(SendHandle::new(rx))
    }

    fn sender_address(&self) -> Address {
        Address::ZERO
    }
}

#[derive(Debug)]
struct RequeueingBlobPipeline {
    frame: Arc<Frame>,
    available: bool,
    requeues: Arc<AtomicUsize>,
}

impl RequeueingBlobPipeline {
    fn new(frame: Arc<Frame>, requeues: Arc<AtomicUsize>) -> Self {
        Self { frame, available: true, requeues }
    }
}

impl BatchPipeline for RequeueingBlobPipeline {
    fn add_block(&mut self, _: BaseBlock) -> Result<(), (ReorgError, Box<BaseBlock>)> {
        Ok(())
    }

    fn step(&mut self) -> Result<StepResult, StepError> {
        Ok(StepResult::Idle)
    }

    fn next_submission(&mut self) -> Option<BatchSubmission> {
        if !self.available {
            return None;
        }
        self.available = false;
        Some(BatchSubmission {
            id: SubmissionId(1),
            channel_id: ChannelId::default(),
            da_type: DaType::Blob,
            frames: vec![Arc::clone(&self.frame)],
        })
    }

    fn confirm(&mut self, _: SubmissionId, _: u64) {
        panic!("oversized blob frame must never be confirmed");
    }

    fn requeue(&mut self, id: SubmissionId) {
        assert_eq!(id, SubmissionId(1));
        self.available = true;
        let count = self.requeues.fetch_add(1, Ordering::SeqCst) + 1;

        // Production BatchEncoder::requeue rewinds the ready-channel cursor, so
        // the same impossible frame is returned again. Stop the PoC after the
        // second requeue to prove the actual submission loop has repeated.
        if count >= 2 {
            panic!("PoC stop: oversized blob frame was requeued repeatedly");
        }
    }

    fn advance_l1_head(&mut self, _: u64) {}

    fn force_close_channel(&mut self) {}

    fn reset(&mut self) {}

    fn prune_safe(&mut self, _: u64) {}

    fn da_backlog_bytes(&self) -> u64 {
        0
    }
}

#[tokio::test(flavor = "current_thread")]
async fn h01_oversized_blob_frame_requeues_before_any_l1_tx_send() {
    // A full non-first frame produced with max_frame_size = 130_044 carries
    // 130_044 - 23 bytes of data. Blob packing then adds one derivation-version
    // byte plus the 23-byte frame envelope:
    //
    //     1 + 23 + 130_021 = 130_045
    //
    // That is one byte above BlobEncoder::BLOB_MAX_DATA_SIZE (130_044).
    let frame_data_len = BlobEncoder::BLOB_MAX_DATA_SIZE - BlobEncoder::FRAME_OVERHEAD;
    let oversized_payload_len = 1 + BlobEncoder::FRAME_OVERHEAD + frame_data_len;
    assert_eq!(oversized_payload_len, BlobEncoder::BLOB_MAX_DATA_SIZE + 1);

    let frame = Arc::new(Frame::new(ChannelId::default(), 1, vec![0x42; frame_data_len], false));
    assert!(
        BlobEncoder::encode_packed(&[Arc::clone(&frame)]).is_err(),
        "the exact frame selected for a blob tx is too large after the derivation prefix"
    );

    let requeues = Arc::new(AtomicUsize::new(0));
    let mut pipeline = RequeueingBlobPipeline::new(Arc::clone(&frame), Arc::clone(&requeues));

    let sends = Arc::new(AtomicUsize::new(0));
    let tx_manager = RecordingTxManager { sends: Arc::clone(&sends) };
    let mut queue = SubmissionQueue::new(tx_manager, Address::ZERO, 1);

    let result = AssertUnwindSafe(queue.submit_pending(&mut pipeline)).catch_unwind().await;

    assert!(
        result.is_err(),
        "the PoC intentionally stops after proving the real submission loop repeats"
    );
    assert_eq!(
        requeues.load(Ordering::SeqCst),
        2,
        "the impossible frame was requeued and selected again"
    );
    assert_eq!(
        sends.load(Ordering::SeqCst),
        0,
        "no L1 tx candidate reached the tx manager before the requeue loop"
    );
}
```

***

## To run the PoC

{% stepper %}
{% step %}

## Start the single-sequencer devnet

```bash
just devnet up-single
just devnet status
```

{% endstep %}

{% step %}

## Stop the compose batcher so the PoC logs stay isolated

```bash
docker compose --env-file etc/docker/devnet-env -f etc/docker/docker-compose.yml stop base-batcher
```

{% endstep %}

{% step %}

## Run the one-off batcher against the stable client

```bash
docker run --rm --name h01-batcher-poc-client --network docker_default \
  base-batcher:local \
  --l1-rpc-url http://l1-el:4545 \
  --l2-rpc-url http://base-client:8545 \
  --rollup-rpc-url http://base-client-cl:8549 \
  --private-key 0x92db14e403b83dfe3df233f83dfa3a0d7096f21ca9b0d6d6b8d88b2b4ec1564e \
  --max-channel-duration 2 \
  --poll-interval 1 \
  --sub-safety-margin 0 \
  --num-confirmations 1
```

{% endstep %}

{% step %}

## Confirm the batcher backfills from the stable client

```
fetched safe L2 head safe_l2=498
starting sequential backfill ... latest_l2=543
```

{% endstep %}

{% step %}

## If needed, send large normal L2 transactions while the one-off batcher is running

```bash
source etc/docker/devnet-env
COUNT=18
BYTES=16384
GAS_LIMIT=5000000

for i in $(seq 1 "$COUNT"); do
  DATA="0x$(openssl rand -hex "$BYTES")"
  cast send "$ANVIL_ACCOUNT_2_ADDR" 'f(bytes)' "$DATA" \
    --rpc-url "$L2_BUILDER_RPC_URL" \
    --private-key "$ANVIL_ACCOUNT_1_KEY" \
    --gas-limit "$GAS_LIMIT"
done
```

{% endstep %}

{% step %}

## Expected sequence

1. The sequencer accepts the large `f(bytes)` L2 transactions.
2. The batcher continues to publish normal L1 blob txs.
3. The next selected frame reaches `blob_payload_size=130045`.
4. The batcher loops on the encode failure without creating a new L1 tx for the offending frame.
   {% endstep %}

{% step %}

## Restore the normal compose batcher after the run

```bash
docker compose --env-file etc/docker/devnet-env -f etc/docker/docker-compose.yml up -d base-batcher
```

{% endstep %}

{% step %}

## Console output

<details>

<summary>This should be in the console output/log</summary>

```
2026-04-24T15:22:00.363756Z  WARN base_batcher_core::submissions: failed to encode frames to blob; requeueing without L1 tx submission error=data too large: 130045 bytes exceeds maximum 130044 submissions=1 frames=1 blob_payload_size=130045 blob_max_data_size=130044 first_frame_number=Some(0) first_frame_data_len=Some(130021)
2026-04-24T15:22:00.363769Z  WARN base_batcher_core::submissions: failed to encode frames to blob; requeueing without L1 tx submission error=data too large: 130045 bytes exceeds maximum 130044 submissions=1 frames=1 blob_payload_size=130045 blob_max_data_size=130044 first_frame_number=Some(0) first_frame_data_len=Some(130021)
2026-04-24T15:22:00.363782Z  WARN base_batcher_core::submissions: failed to encode frames to blob; requeueing without L1 tx submission error=data too large: 130045 bytes exceeds maximum 130044 submissions=1 frames=1 blob_payload_size=130045 blob_max_data_size=130044 first_frame_number=Some(0) first_frame_data_len=Some(130021)
2026-04-24T15:22:00.363795Z  WARN base_batcher_core::submissions: failed to encode frames to blob; requeueing without L1 tx submission error=data too large: 130045 bytes exceeds maximum 130044 submissions=1 frames=1 blob_payload_size=130045 blob_max_data_size=130044 first_frame_number=Some(0) first_frame_data_len=Some(130021)
2026-04-24T15:22:00.363808Z  WARN base_batcher_core::submissions: failed to encode frames to blob; requeueing without L1 tx submission error=data too large: 130045 bytes exceeds maximum 130044 submissions=1 frames=1 blob_payload_size=130045 blob_max_data_size=130044 first_frame_number=Some(0) first_frame_data_len=Some(130021)
2026-04-24T15:22:00.363822Z  WARN base_batcher_core::submissions: failed to encode frames to blob; requeueing without L1 tx submission error=data too large: 130045 bytes exceeds maximum 130044 submissions=1 frames=1 blob_payload_size=130045 blob_max_data_size=130044 first_frame_number=Some(0) first_frame_data_len=Some(130021)
2026-04-24T15:22:00.363835Z  WARN base_batcher_core::submissions: failed to encode frames to blob; requeueing without L1 tx submission error=data too large: 130045 bytes exceeds maximum 130044 submissions=1 frames=1 blob_payload_size=130045 blob_max_data_size=130044 first_frame_number=Some(0) first_frame_data_len=Some(130021)
```

</details>
{% endstep %}
{% endstepper %}
