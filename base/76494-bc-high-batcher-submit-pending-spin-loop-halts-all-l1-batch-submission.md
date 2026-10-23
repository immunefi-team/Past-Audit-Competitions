# 76494 bc high batcher submit pending spin loop halts all l1 batch submission

**Submitted on May 4th 2026 at 17:13:40 UTC by @Blobism for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76494
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

The batcher's `SubmissionQueue::submit_pending` (`crates/batcher/core/src/submissions.rs:118-125`) contains a recovery path that does not yield: when `BlobEncoder::encode_packed` rejects a submission, the loop calls `pipeline.requeue(id); drop(permit); continue` with no `.await` in between. The production `BatchEncoder::requeue` rewinds the channel cursor so the very next `next_submission()` re-emits an identical-bytes submission. Because the failure path contains no yield point, the entire driver `tokio::select!` task wedges — admin RPC, cancellation, L1/L2 ingestion, and shutdown all starve. Once the batcher is wedged, no L1 batch transactions are submitted; after `MAX_SEQUENCER_DRIFT` the L2 safe head can no longer advance and the chain halts confirming new L1-derivable transactions.

## Vulnerability Details

### The defect: failure arm has no yield point and `requeue` rewinds the cursor

`submissions.rs:109-134`:

```rust
let candidate = match da_type {
    DaType::Blob => match BlobEncoder::encode_packed(&frames) {
        Ok(blob) => TxCandidate { /* ... */ },
        Err(e) => {
            warn!(error = %e, "failed to encode frames to blob, requeueing");
            for id in ids {
                pipeline.requeue(id);   // rewinds the channel cursor
            }
            drop(permit);               // releases the only outstanding permit
            continue;                   // back to `loop {`, no .await
        }
    },
    /* ... */
};
let handle = self.tx_manager.send_async(candidate).await;  // the ONLY .await
```

`BatchEncoder::requeue` rewinds the cursor:

```rust
if pending_ref.frame_start < channel.cursor {
    channel.cursor = pending_ref.frame_start;
}
```

so the next iteration's `next_submission()` re-emits the same submission bytes wrapped in a fresh `SubmissionId`. `try_acquire_owned()` then succeeds immediately because no other task holds a permit. The async task body never yields. Every other driver `select!` arm (block ingestion, L1 head updates, admin RPC, cancellation token, shutdown) is starved.

### Trigger

Any input that causes `BlobEncoder::encode_packed` to return an `Err` once is enough: the spin is self-perpetuating from that point on regardless of trigger. Reachable triggers exist on default config — for example, a single L2 transaction with \~200 KB of high-entropy calldata flows through `ChannelOut::output_frame` and the resulting frame slightly exceeds `BLOB_MAX_DATA_SIZE`, causing `encode_packed` to reject it. This requires no operator misconfiguration and no privileged role; gas cost is on the order of single-digit dollars at typical Base prices.

The root bug is in the recovery path. Any future change that introduces a new way for `encode_packed` (or any similar fallible encoding step) to return an error re-exposes the same wedge.

### Why the existing `test_blob_encoding_failure_requeues_submission` does not catch this

The existing test injects an oversized `BatchSubmission` into `TrackingPipeline`, whose `requeue` does not rewind anything — it just records the call in a `Vec`. After one requeue the inner queue is empty, the outer loop breaks via `if ids.is_empty() { break; }`, and the test asserts only that requeue was called once. It does not exercise the production rewind path; the test asserts the wrong invariant.

## Impact Details

Severity: **Critical — Network not being able to confirm new transactions (total network shutdown).**

While the batcher is wedged, no L1 inbox transactions are submitted. Once `MAX_SEQUENCER_DRIFT` (\~30 minutes of L2 time) elapses without a corresponding L1 batch, the sequencer cannot include new blocks without breaking the derivation invariant; the L2 safe head stops advancing and the chain effectively halts confirming new L1-derivable transactions.

Naive operator recovery (process restart) does not unstick the chain: the poison L2 block is already part of L2 history, and on restart the batcher rebuilds channels from unsubmitted L2 blocks and re-encodes the same submission, hitting the same failure. Recovery requires a code patch (e.g. a forced yield, or drop-after-N-failures, in the `submit_pending` failure arm) plus operator deployment. While the batcher is wedged, the admin RPC (`pause`, `resume`, `flush`, `set_throttle`) is also starved, so a graceful intervention is not available.

The minimal fix is one token at `submissions.rs:124`: change `continue` to `break` so the failure arm exits the inner submission loop and returns control to the driver `select!`, which can then surface the error, accept admin RPC commands, or shut down gracefully. A drop-after-N-failures policy is a reasonable hardening on top.

## References

`base/base` commit: `e3467a2048881213b56739a54a876efb9c6ea103` (`v0.8.0-rc.28`)

* Vulnerable failure path: `crates/batcher/core/src/submissions.rs:118-125`.
* `BatchEncoder::requeue` rewinds the channel cursor: `crates/batcher/encoder/src/encoder.rs:617-619`.

## Link to Proof of Concept

<https://gist.github.com/blobism/95cae5542f46cfb491fae267496415a2>

## Proof of Concept

{% hint style="info" %}
Note: ignore any references to bug 102, this is an arbitrary bug label
{% endhint %}

Get the PoC Gist: <https://gist.github.com/blobism/95cae5542f46cfb491fae267496415a2>

```bash
git clone git@github.com:base/base.git --branch v0.8.0-rc.28
cd base

# apply patch from Gist
git apply poc.diff

cargo test -p base-batcher-core --test poc_bug_102 -- --nocapture --test-threads 1
```

The branch adds one integration test file — `crates/batcher/core/tests/poc_bug_102.rs` — and two minimal dev-dependency lines to `crates/batcher/core/Cargo.toml` (`rt-multi-thread` feature on the existing tokio dev-dep, plus `base-consensus-genesis` so the test can construct a default `RollupConfig`). No source code under `crates/`, `bin/`, or any production manifest is modified. The file contains two tests.

### Test 1 — `poc_cycle_real_encoder_requeue_rewinds_forever`

Wraps a real, primed `BatchEncoder` in a `BatchPipeline` impl that delegates to `BatchEncoder` while counting `next_submission` and `requeue` calls. Runs the production failure-arm sequence — `next_submission → encode_packed → on Err, requeue` — synchronously for 1000 iterations and asserts every iteration returns identical submission bytes (with a fresh `SubmissionId`) and that `encode_packed` fails every time. This is what proves the cycle is self-perpetuating in production: the real `BatchEncoder::requeue` rewinds, so progress is never made.

```
cycle_proof: 1000 iterations → 1000 next_submission calls, 1000 requeues
test poc_cycle_real_encoder_requeue_rewinds_forever ... ok
```

### Test 2 — `poc_exploit_submit_pending_spins`

Wires the real primed `BatchEncoder` (via the counting wrapper) into the production `SubmissionQueue::new(tx_manager, Address::ZERO, /*max_pending=*/ 4)` and `tokio::spawn`s `submit_pending` on a 2-worker multi-thread runtime. After 250 ms of wall time, the test thread reads three counters, asserts a runaway spin, and detaches the worker via `shutdown_timeout(50 ms)` so the test does not hang on the non-yielding task.

```
submit_pending after 250ms:
  next_submission calls: 160127
  requeue calls:         160126
  send_async calls:      0
  task finished:         false
test poc_exploit_submit_pending_spins ... ok
```

Asserted invariants:

* `requeue calls > 1_000` — runaway spin (in practice >100k in 250 ms).
* `send_async calls == 0` — `submit_pending` never escapes the `Err` arm; the only `.await` in the loop body is on the success path and is never reached.
* `task finished == false` — the future never returns; with no `.await` in the failure path the task cannot be preempted.

Together the two tests demonstrate, against the production `BatchEncoder` and `SubmissionQueue`, that the production requeue path makes the failure cycle self-perpetuating, and that the live `submit_pending` future spins indefinitely without yielding to the runtime.

Expected full output:

```
running 2 tests
test poc_cycle_real_encoder_requeue_rewinds_forever ... cycle_proof: 1000 iterations → 1000 next_submission calls, 1000 requeues
ok
test poc_exploit_submit_pending_spins ... submit_pending after 250ms:
  next_submission calls: 160127
  requeue calls:         160126
  send_async calls:      0
  task finished:         false
ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

The exact `next_submission` / `requeue` counters in test 2 vary by host CPU (250 ms of busy loop). Any value > 1000 is a wedge.
