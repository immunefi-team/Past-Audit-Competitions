# 76287 bc low async batcher l1 transactions permanently skip a dropped nonce after mempool deadline

**Submitted on May 3rd 2026 at 17:28:17 UTC by @shaXBT for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76287
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

## Description

The batcher uses the tx-manager asynchronous send path for L1 data submissions. That path reserves a nonce before spawning the background send task, and the nonce manager records a high-water mark so later async calls cannot reissue the same nonce after a reset. If the first transaction with nonce `N` is accepted by the L1 RPC/mempool but never receives a canonical receipt before `tx_not_in_mempool_timeout` expires, the send loop returns `MempoolDeadlineExpired` while `SendState::has_published()` remains true.

The cleanup logic treats this as a published async nonce and neither resets the nonce manager nor returns nonce `N` to the reuse pool. The next batch submission therefore reserves `N+1` while the L1 account nonce is still `N`. A local regression harness reproduces the stall without public-network traffic by using a deterministic provider mock that accepts the raw transaction for nonce `N`, never returns a receipt, and reports the account nonce as still `N`; the second and third async sends then attempt `N+1` and `N+2`, which are unmineable gap nonces. The bug has been confirmed end-to-end at the batcher integration boundary - not only at the tx-manager unit level - by the integration test at `crates/batcher/core/tests/lifecycle.rs:262`.

**Prerequisites / delivery / payload:** The triggering condition is an L1 mempool/provider state where a batch transaction is accepted once and then evicted or never propagated/mined before the default 120-second mempool deadline. The safe PoC payload is a local mocked L1 provider response sequence: accept nonce `N`, return no receipt until the deadline, keep `get_transaction_count` at `N`, then reject `N+1` as nonce-too-high. No live DoS traffic or public-network testing is required.

**Impact:** The batcher requeues the affected frames but cannot make progress because every later transaction is above the chain nonce. The default deadline is 120 seconds, already above the threshold for delaying block-related L1 data availability by at least 500% as the batcher instance is responsible for posting data. The issue could be downgraded if production supervision demonstrably restarts/repairs the nonce within the threshold, if production disables this async path.

**Rollback / containment:** pause batcher submissions, send a same-signer cancel/replacement transaction at nonce `N` or restart after clearing the tx-manager high-water state, verify the signer account pending nonce from a trusted L1 RPC, then requeue the pending frames. Rotate the L1 RPC endpoint if it accepted but failed to propagate the original transaction.

## Code Walkthrough Step-by-Step Exploitation

{% stepper %}
{% step %}

## Trigger an accepted-but-dropped L1 batch transaction

Under L1 congestion or a local regression mock, make `send_raw_transaction` accept the batcher transaction at nonce `N` but never return a canonical receipt before `tx_not_in_mempool_timeout`.
{% endstep %}

{% step %}

## Async nonce reservation occurs before publication

`crates/utilities/tx-manager/src/manager.rs:1606-1629`

```rust
async fn send_async(&self, candidate: TxCandidate) -> SendHandle {
    let (tx, rx) = oneshot::channel();

    let nonce = match self.nonce_manager.reserve_nonce().await {
        Ok(n) => Some(n),
        Err(e) => {
            let _ = tx.send(Err(e));
            return SendHandle::new(rx);
        }
    };

    let manager = self.clone();
    tokio::spawn(async move {
        let result = manager.send_tx(candidate, nonce).await;
        let _ = tx.send(result);
    });
    SendHandle::new(rx)
}
```

{% endstep %}

{% step %}

## The nonce manager records a high-water mark for async reservations

`crates/utilities/tx-manager/src/nonce.rs:321-327`

```rust
pub fn consume_reserved(mut self) -> Result<u64, TxManagerError> {
    let nonce = self.nonce;
    if let Some(mut guard) = self.guard.take() {
        let next = nonce.checked_add(1).ok_or(TxManagerError::NonceOverflow)?;
        guard.reserved_high_water = guard.reserved_high_water.max(next);
    }
    Ok(nonce)
}
```

{% endstep %}

{% step %}

## The accepted transaction marks the nonce as published

`crates/utilities/tx-manager/src/manager.rs:1337-1341`

```rust
match result {
    Ok(Ok(pending)) => {
        let tx_hash = *pending.tx_hash();
        send_state.record_successful_publish();
        info!(tx_hash = %tx_hash, "transaction published");
        Ok(tx_hash)
    }
```

{% endstep %}

{% step %}

## After the deadline, the send loop aborts as a non-retryable failure

`crates/utilities/tx-manager/src/send_state.rs:138-160`

```rust
pub fn critical_error(&self) -> Option<TxManagerError> {
    let now = Instant::now();
    let inner = self.inner.lock().expect("SendState mutex poisoned");
    if !inner.mined_txs.is_empty() {
        return None;
    }
    if inner.already_reserved {
        return Some(TxManagerError::AlreadyReserved);
    }
    if inner.nonce_too_high {
        return Some(TxManagerError::NonceTooHigh);
    }
    if !inner.has_published && inner.nonce_too_low_count > 0 {
        return Some(TxManagerError::NonceTooLow);
    }
    if inner.nonce_too_low_count >= self.safe_abort_nonce_too_low_count {
        return Some(TxManagerError::NonceTooLow);
    }
    if let Some(deadline) = inner.mempool_deadline
        && now >= deadline
    {
        return Some(TxManagerError::MempoolDeadlineExpired);
    }
```

{% endstep %}

{% step %}

## The cleanup path does not reset or return nonce `N`

`nonce_override.is_some()` suppresses reset for all errors except `NonceTooHigh`, and `has_published=true` suppresses returning the reserved nonce (`crates/utilities/tx-manager/src/manager.rs:973-1022`):

```rust
fn should_reset_nonce_on_send_error<T>(
    result: &TxManagerResult<T>,
    send_state: &SendState,
    nonce_override: Option<u64>,
) -> bool {
    match result {
        Ok(_) => false,
        Err(TxManagerError::NonceTooHigh) => true,
        _ if nonce_override.is_some() => false,
        Err(TxManagerError::SendTimeout) => true,
        Err(_) => !send_state.has_published(),
    }
}

fn should_return_reserved_nonce<T>(
    result: &TxManagerResult<T>,
    send_state: &SendState,
) -> bool {
    match result {
        Ok(_) | Err(TxManagerError::NonceTooHigh | TxManagerError::NonceTooLow) => false,
        Err(_) => !send_state.has_published(),
    }
}
```

{% endstep %}

{% step %}

## Reset cannot recover the skipped nonce once the high-water mark moved

Future reservations use `max(chain_nonce, reserved_high_water)`, so even a later `NonceTooHigh` reset will skip over nonce `N` (`crates/utilities/tx-manager/src/nonce.rs:210-224`):

```rust
let effective = nonce.max(guard.reserved_high_water);
if effective != nonce {
    debug!(
        chain_nonce = nonce,
        high_water = guard.reserved_high_water,
        effective,
        "high-water mark advanced nonce past chain value",
    );
}
let next = effective.checked_add(1).ok_or(TxManagerError::NonceOverflow)?;
guard.nonce = Some(next);
```

{% endstep %}

{% step %}

## Batcher requeues but cannot progress

`crates/batcher/core/src/submissions.rs:149-175`, `220-228`

```rust
let handle = self.tx_manager.send_async(candidate).await;
// ... outcome handling ...
Err(e) => {
    warn!(error = %e, "submission failed");
    TxOutcome::Failed
}
```

```rust
TxOutcome::Failed => {
    let count = ids.len();
    for id in ids {
        pipeline.requeue(id);
    }
    warn!(submissions = %count, "submission failed, requeued for retry");
}
```

{% endstep %}
{% endstepper %}

## As comparison with Optimism reference (`op-service/txmgr`)

The Optimism Go reference implementation does not have this bug. Three structural invariants diverge between the two codebases.

`ErrMempoolDeadlineExpired` is gated on never-published in Optimism. In `op-service/txmgr/send_state.go:130`, `CriticalError` only returns `ErrMempoolDeadlineExpired` while `successfulPublishCount == 0`:

```go
case s.successfulPublishCount == 0 && s.now().After(s.txInMempoolDeadline):
    return ErrMempoolDeadlineExpired
```

Once any publish succeeds, the deadline branch is permanently disarmed - the transaction loop continues rebroadcasting indefinitely. Base equivalent at `crates/utilities/tx-manager/src/send_state.rs:138-160` falls through to `MempoolDeadlineExpired` even after `has_published=true` because there is no equivalent `successfulPublishCount` guard on the deadline arm. This is the direct trigger of F-5: a transaction that was accepted by the RPC but not mined will eventually hit the deadline and abort, even though it was already in the mempool.

Optimism re-broadcasts the same nonce indefinitely until the outer context cancels. `txmgr.go:680-749` shows `sendTx` as an unbounded `for {}` loop driven by two tickers: `rebroadcastInterval` (line 710) for re-submission attempts and `bumpFeeTicker` (lines 736-738) for fee escalation. The loop exits only on `CriticalError()`, a successful receipt, or `ctx.Done()` (the outer `TxSendTimeout`). The comment at `txmgr.go:767-771` is explicit: a previously successfully published tx can get dropped from the mempool, and if resubmission is not attempted the system can end up waiting on it to get mined indefinitely. `SendAsync` at `txmgr.go:333-380` calls `resetNonce()` on any error from `sendTx` (lines 371-373), including timeouts, forcing the next call to re-read chain state via `NonceAt`. Base fee-bump loop is bounded by the 120-second `tx_not_in_mempool_timeout`, producing only 2-3 resubmission attempts before permanently abandoning the nonce.

Optimism has no async-reservation high-water mark. `txmgr.go:631-668` shows `signWithNextNonce` holding `m.nonceLock`, fetching once via `NonceAt(latest)`, and incrementing a single `m.nonce` counter. On signing failure it decrements back (line 663). `resetNonce()` at `txmgr.go:670-676` sets `m.nonce = nil`, forcing the next call to re-read chain state. `SendAsync` at `txmgr.go:333-380` calls `resetNonce()` on any error from `sendTx` (lines 371-373). Base port introduced a `reserved_high_water` field that ratchets forward in `consume_reserved` (`nonce.rs:321-327`) and is never lowered on reset - `advance_nonce` (`nonce.rs:210-224`) uses `max(chain_nonce, reserved_high_water)`. Combined with `should_reset_nonce_on_send_error` returning false when `has_published=true` (`manager.rs:973-1022`), the nonce gap becomes permanent.

Optimism invariant: once the chain has accepted a nonce, keep retrying until it lands or the operator-supplied outer context kills the whole call, and nonce state is always re-derived from `NonceAt(latest)` on any failure. Base reth-based port introduced an async high-water mark to enforce ordered concurrent reservations (a valid concern in the Rust async model) and a separate mempool deadline that fires post-publication. The two mechanisms combined create a state where the manager abandons a published nonce without recycling it - a state Optimism design forbids by construction. This is an OP-Stack Rust/Go parity divergence.

## Location

`crates/utilities/tx-manager/src/manager.rs:973-1022` is the root cause: async nonce cleanup refuses both reset and nonce return after an accepted-but-unmined publish. `crates/utilities/tx-manager/src/nonce.rs:210-224` makes the condition persistent by honoring `reserved_high_water` even after reset. `crates/batcher/core/src/submissions.rs:149-175` is the runtime entry point using `send_async` for batch submissions.

## Recommendation

Track async nonce state separately for accepted-but-not-mined transactions. When `MempoolDeadlineExpired` fires and no canonical receipt exists, the manager should either continue replacing/cancelling nonce `N` until it lands, or explicitly return/repair the nonce only after proving the transaction is absent from the local and upstream mempool. Do not advance to `N+1` until nonce `N` is mined, cancelled, or safely reissued.

The most direct fix aligned with Optimism invariant is to guard `MempoolDeadlineExpired` on the absence of a successful publish - mirroring the Go `successfulPublishCount == 0` check in `send_state.go:130`. If the deadline fires after a successful publish, the send loop should continue rebroadcasting (like Optimism unbounded `sendTx` loop) until the outer context deadline cancels the entire operation, rather than aborting and leaking the nonce.

A minimal regression test suite should cover: (1) mock provider accepts raw transaction for nonce `N`, returns no receipt until a short deadline expires, keeps `get_transaction_count` at `N`, and asserts the next batcher submission retries/cancels nonce `N` rather than signing `N+1`; (2) the same scenario followed by a `NonceTooHigh` response to ensure `reserved_high_water` is lowered or bypassed safely.

The integration test (`test_mempool_deadline_critical_error_requeues_without_txpool_recovery`) extends coverage to the full batcher-side path: it uses `anvil --no-mining` to accept-but-not-mine, emulates mempool eviction via `drop_anvil_tx`, and asserts that the second batcher submission signs nonce `N+1` while the canonical L1 nonce is still `N` and the tx-manager next reserved nonce has advanced to `N+2`.

## Proof of Concept

Files changed:

1. `crates/batcher/core/Cargo.toml`
2. `crates/batcher/core/tests/lifecycle.rs`

Cargo.toml change:\
Add this under \[dev-dependencies]:

```bash
alloy-rpc-types = { workspace = true, features = ["txpool"] }
```

[lifecycle.rs](http://lifecycle.rs/) change:\
Add the PoC helpers/imports from the patch, then add the test named:

place the following test under `crates/batcher/core/tests/lifecycle.rs`:

```bash
/// PoC exploitation steps:
///
/// 1. Run the batcher submission path against an L1 provider that accepts
///    transactions but does not mine them (`anvil --no-mining`).
/// 2. Submit one batch through `SubmissionQueue::submit_pending`, which is the
///    batcher entry point that calls `TxManager::send_async`. This reserves
///    nonce N, publishes the transaction, and then fails with
///    `MempoolDeadlineExpired` because no canonical receipt appears.
/// 3. Verify the accepted nonce-N transaction reached the L1 txpool, then drop
///    it before mining to emulate provider/mempool eviction.
/// 4. Verify the canonical account nonce is still N.
/// 5. Let the batcher handle the failed outcome. It requeues the submission
///    instead of entering txpool recovery.
/// 6. Submit the requeued batch. The vulnerable nonce cleanup path has not
///    returned or repaired nonce N, so the retry signs and publishes nonce N+1.
/// 7. Verify the exploit condition: the L1 txpool contains nonce N+1 while the
///    canonical account nonce is still N, and the tx-manager's next reserved
///    nonce has advanced to N+2.
#[tokio::test]
async fn test_mempool_deadline_critical_error_requeues_without_txpool_recovery() {
    eprintln!("[PoC step 1] starting no-mining L1 provider");
    let anvil = Anvil::new().arg("--no-mining").spawn();
    let tx_not_in_mempool_timeout = Duration::from_millis(200);
    let config = TxManagerConfig {
        num_confirmations: 1,
        receipt_query_interval: Duration::from_millis(50),
        resubmission_timeout: Duration::from_millis(500),
        tx_not_in_mempool_timeout,
        confirmation_timeout: Duration::from_secs(5),
        tx_send_timeout: Duration::ZERO,
        ..TxManagerConfig::default()
    };
    let manager = manager_from_anvil(&anvil, config).await;
    let address = manager.sender_address();
    let chain_nonce_before =
        manager.provider().get_transaction_count(address).await.expect("should fetch tx count");
    assert_eq!(chain_nonce_before, 0, "fresh Anvil signer should start at nonce N=0");
    eprintln!(
        "[PoC step 1] signer={address}, canonical_nonce={chain_nonce_before}, \
         mempool_deadline={:?}",
        tx_not_in_mempool_timeout,
    );

    let last_error = Arc::new(Mutex::new(None));
    let recording_manager =
        RecordingTxManager { inner: manager.clone(), last_error: Arc::clone(&last_error) };

    let recorded = Arc::new(Mutex::new(Recorded::default()));
    let mut pipeline = TrackingPipeline::new(Arc::clone(&recorded));
    pipeline.submissions.push_back(calldata_submission(0));

    let mut queue = SubmissionQueue::new(recording_manager, Address::ZERO, 1);
    eprintln!(
        "[PoC step 2] submitting first batch: SubmissionQueue::submit_pending -> \
         TxManager::send_async"
    );
    queue.submit_pending(&mut pipeline).await;

    let (ids, outcome) = tokio::time::timeout(Duration::from_secs(5), queue.next_settled())
        .await
        .expect("batcher should receive the tx-manager critical error")
        .expect("submission should settle");

    assert_eq!(ids, vec![SubmissionId(0)]);
    assert_eq!(
        *last_error.lock().unwrap(),
        Some(TxManagerError::MempoolDeadlineExpired),
        "real tx-manager send loop must abort through SendState::critical_error"
    );
    assert_eq!(
        outcome,
        TxOutcome::Failed,
        "batcher must classify MempoolDeadlineExpired as a failed submission"
    );
    eprintln!(
        "[PoC step 2] first submission settled: ids={ids:?}, outcome={outcome:?}, \
         tx_manager_error={:?}",
        *last_error.lock().unwrap(),
    );

    let first_txpool = txpool_entries(manager.provider(), address).await;
    assert_eq!(
        txpool_nonces(&first_txpool),
        vec![0],
        "first real batcher submission must have published nonce N into the L1 txpool"
    );

    let first_hash = first_txpool[0].hash;
    eprintln!(
        "[PoC step 3] accepted tx is in L1 txpool: hash={first_hash}, nonce={}",
        first_txpool[0].nonce,
    );
    drop_anvil_tx(manager.provider(), first_hash).await;
    eprintln!("[PoC step 3] dropped nonce-N tx from the L1 txpool before mining");
    assert!(
        txpool_entries(manager.provider(), address).await.is_empty(),
        "PoC precondition: the accepted nonce-N tx was evicted/dropped before it mined"
    );

    let chain_nonce_after_drop =
        manager.provider().get_transaction_count(address).await.expect("should fetch tx count");
    assert_eq!(
        chain_nonce_after_drop, 0,
        "dropping the tx leaves the canonical L1 account nonce at N"
    );
    eprintln!("[PoC step 4] canonical nonce after txpool eviction: {chain_nonce_after_drop}");

    eprintln!("[PoC step 5] handling failed outcome; batcher requeues the submission");
    queue.handle_outcome(&mut pipeline, ids, outcome);
    pipeline.submissions.push_back(calldata_submission(0));
    *last_error.lock().unwrap() = None;
    eprintln!(
        "[PoC step 6] resubmitting requeued batch through the same \
         TxManager::send_async path"
    );
    queue.submit_pending(&mut pipeline).await;

    let (ids2, outcome2) = tokio::time::timeout(Duration::from_secs(5), queue.next_settled())
        .await
        .expect("second batcher submission should also settle")
        .expect("second submission should produce an outcome");

    assert_eq!(ids2, vec![SubmissionId(0)]);
    assert_eq!(
        *last_error.lock().unwrap(),
        Some(TxManagerError::MempoolDeadlineExpired),
        "retry must hit the same real critical error after signing a gap nonce"
    );
    assert_eq!(outcome2, TxOutcome::Failed);
    eprintln!(
        "[PoC step 6] retry settled: ids={ids2:?}, outcome={outcome2:?}, \
         tx_manager_error={:?}",
        *last_error.lock().unwrap(),
    );

    let retry_txpool = txpool_entries(manager.provider(), address).await;
    assert_eq!(
        txpool_nonces(&retry_txpool),
        vec![1],
        "exploit reproduced: retry published nonce N+1 while L1 still expects N"
    );
    eprintln!(
        "[PoC step 7] retry txpool state: published_nonce={}, canonical_nonce_still_expected=0",
        retry_txpool[0].nonce,
    );

    queue.handle_outcome(&mut pipeline, ids2, outcome2);

    let chain_nonce =
        manager.provider().get_transaction_count(address).await.expect("should fetch tx count");
    let next_guard = manager.nonce_manager().next_nonce().await.expect("should reserve next nonce");
    let next_nonce = next_guard.nonce();
    drop(next_guard);

    let r = recorded.lock().unwrap();
    assert_eq!(r.dequeued, vec![SubmissionId(0), SubmissionId(0)]);
    assert_eq!(
        r.requeued,
        vec![SubmissionId(0), SubmissionId(0)],
        "failed submissions must be requeued without entering txpool recovery"
    );
    assert_eq!(chain_nonce, 0, "no-mining Anvil must keep the L1 account nonce at N");
    assert_eq!(
        next_nonce, 2,
        "two real batcher submissions advanced the async nonce high-water mark to N+2"
    );

    eprintln!(
        "[PoC result] reproduced: accepted nonce 0 tx {first_hash}, dropped it before mining, \
         retry queued nonce {}, chain_nonce={chain_nonce}, tx_manager_next_nonce={next_nonce}",
        retry_txpool[0].nonce,
    );
}
```

```bash
proxy cargo test -p base-batcher-core --features test-utils --test lifecycle test_mempool_deadline_critical_error_requeues_without_txpool_recovery -- --nocapture
```

Expected PoC output includes:

```bash
[PoC step 1] starting no-mining L1 provider
[PoC step 2] submitting first batch: SubmissionQueue::submit_pending -> TxManager::send_async
[PoC step 2] first submission settled: ids=[SubmissionId(0)], outcome=Failed, tx_manager_error=Some(MempoolDeadlineExpired)
[PoC step 3] accepted tx is in L1 txpool: hash=..., nonce=0
[PoC step 3] dropped nonce-N tx from the L1 txpool before mining
[PoC step 4] canonical nonce after txpool eviction: 0
[PoC step 5] handling failed outcome; batcher requeues the submission
[PoC step 6] resubmitting requeued batch through the same TxManager::send_async path
[PoC step 6] retry settled: ids=[SubmissionId(0)], outcome=Failed, tx_manager_error=Some(MempoolDeadlineExpired)
[PoC step 7] retry txpool state: published_nonce=1, canonical_nonce_still_expected=0
[PoC result] reproduced: accepted nonce 0 tx ..., dropped it before mining, retry queued nonce 1, chain_nonce=0, tx_manager_next_nonce=2
```
