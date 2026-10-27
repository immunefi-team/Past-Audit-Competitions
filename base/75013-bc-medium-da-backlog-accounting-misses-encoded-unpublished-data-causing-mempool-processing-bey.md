# 75013 bc medium da backlog accounting misses encoded unpublished data causing mempool processing beyond configured throttle limits

**Submitted on Apr 26th 2026 at 17:14:01 UTC by @OxPrince for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75013
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.24>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

### Brief/Intro

The batcher DA throttle only counts user transactions that are still waiting to be encoded. Once a block is encoded into a channel, the same DA payload is no longer counted by `da_backlog_bytes()` even when it has not been published or confirmed on L1. As a result, the batch driver can tell the block builder to use unthrottled DA limits while unpublished DA backlog is already above the configured threshold, causing network processing nodes to process mempool transactions beyond the intended DA throttle parameters.

### Vulnerability Details

The batcher exposes a DA throttle intended to reduce builder throughput when unsubmitted DA backlog exceeds a configured threshold. The default threshold is `1,000,000` bytes, and the default throttled limits are much lower than the unthrottled limits:

```rust
// crates/batcher/core/src/throttle.rs
threshold_bytes: 1_000_000,
block_size_lower_limit: 2_000,
block_size_upper_limit: 130_000,
tx_size_lower_limit: 150,
tx_size_upper_limit: 20_000,
```

The CLI documents the same behavior:

```rust
// bin/batcher/src/cli.rs
/// When the estimated unsubmitted DA backlog exceeds this value, the batcher
/// signals the sequencer to reduce block throughput.
```

However, the driver drains encoding before applying the throttle:

```rust
// crates/batcher/core/src/driver.rs
self.drain_encoding()?;
self.throttle.apply(self.pipeline.da_backlog_bytes()).await;
self.submissions.recover_txpool().await;
self.submissions.submit_pending(&mut self.pipeline).await;
```

The encoder's backlog function only counts blocks at and after `block_cursor`:

```rust
// crates/batcher/encoder/src/encoder.rs
fn da_backlog_bytes(&self) -> u64 {
    self.blocks
        .iter()
        .skip(self.block_cursor)
        .flat_map(|b| &b.body.transactions)
        .filter(|tx| !matches!(tx, BaseTxEnvelope::Deposit(_)))
        .map(|tx| tx.encode_2718_len() as u64)
        .sum()
}
```

This excludes DA that has already been encoded into the current channel, ready channels, pending submissions, or in-flight L1 transactions. Those bytes are still unpublished and still consume DA capacity, but they are invisible to the throttle signal.

The vulnerable sequence is:

1. User transactions create DA backlog above the configured throttle threshold.
2. `BatchDriver::run()` calls `drain_encoding()` first.
3. Encoding advances `block_cursor` and moves the bytes into unpublished channel/frame state.
4. `BatchDriver::run()` then calls `da_backlog_bytes()`.
5. The backlog now reports `0` or an artificially low value.
6. `DaThrottle` applies unthrottled builder limits even though unpublished DA backlog remains above the threshold.

This is not an admin misconfiguration. The default config enables throttling, and the bug appears in the normal driver/encoder accounting path.

## Impact Details

The relevant set parameters are the DA throttle limits configured through `ThrottleConfig` and applied to the block builder. When backlog is above the threshold, the system is expected to reduce builder DA limits. With the default config, the relevant difference is:

| Condition   | max\_tx\_size | max\_block\_size |
| ----------- | ------------: | ---------------: |
| Throttled   |           150 |            2,000 |
| Unthrottled |        20,000 |          130,000 |

The PoC demonstrates a state where unpublished DA backlog is above `1 MB`, but the driver still applies the unthrottled limits `(20,000, 130,000)`. This means the builder can continue accepting and processing mempool transactions at normal DA size limits even though the throttle condition is already met.

The practical security risk is liveness degradation under L1 publication pressure. If L1 submissions are delayed, blocked, or slow to confirm, encoded unpublished DA can accumulate while the builder continues producing blocks at unthrottled DA limits. This defeats the backpressure mechanism that is supposed to keep DA publication backlog bounded.

## References

* `crates/batcher/core/src/driver.rs`: `BatchDriver::run()` drains encoding before throttle application.
* `crates/batcher/encoder/src/encoder.rs`: `BatchEncoder::da_backlog_bytes()` only counts blocks after `block_cursor`.
* `crates/batcher/core/src/throttle.rs`: default throttle threshold and throttled/unthrottled DA limits.
* `bin/batcher/src/cli.rs`: CLI documents that DA backlog above threshold should reduce block throughput.

## Proof of Concept

`poc_driver_leaves_builder_unthrottled_with_unpublished_da_backlog`

```rust
//! Integration tests for DA throttle behaviour in [`BatchDriver`].

use std::{
    sync::{Arc, Mutex},
    time::Duration,
};

use alloy_primitives::Address;
use async_trait::async_trait;
use base_batcher_core::{
    BatchDriver, BatchDriverConfig, DaThrottle, ThrottleConfig, ThrottleController,
    ThrottleStrategy,
    test_utils::{
        ImmediateConfirmTxManager, PendingL1HeadSource, PendingSource, Recorded, TrackingPipeline,
        TrackingThrottleClient,
    },
};
use base_batcher_encoder::{
    BatchPipeline, BatchSubmission, ReorgError, StepError, StepResult, SubmissionId,
};
use base_batcher_source::{L2BlockEvent, SourceError, UnsafeBlockSource};
use base_common_consensus::BaseBlock;
use base_runtime::{
    Cancellation, Clock, Spawner,
    deterministic::{Config, Runner},
};
use tokio::sync::mpsc;

/// When the DA backlog exceeds the threshold, the driver must call
/// `set_max_da_size` on the throttle client with reduced limits.
#[test]
fn test_throttle_client_called_on_high_backlog() {
    Runner::start(Config::seeded(0), |ctx| async move {
        let recorded = Arc::new(Mutex::new(Recorded::default()));
        // 2 MB backlog — above the default 1 MB threshold.
        let pipeline = TrackingPipeline::new(Arc::clone(&recorded)).with_da_backlog(2_000_000);

        let throttle = ThrottleController::new(ThrottleConfig::default(), ThrottleStrategy::Linear);
        let (throttle_client, throttle_recorded) = TrackingThrottleClient::new();

        let driver = BatchDriver::new(
            ctx.clone(),
            pipeline,
            PendingSource,
            ImmediateConfirmTxManager { l1_block: 1 },
            BatchDriverConfig {
                inbox: Address::ZERO,
                max_pending_transactions: 1,
                drain_timeout: Duration::from_millis(10),
            },
            DaThrottle::new(throttle, Arc::new(throttle_client)),
            PendingL1HeadSource,
        );
        let handle = ctx.spawn(driver.run());

        ctx.sleep(Duration::from_millis(50)).await;
        ctx.cancel();
        assert!(handle.await.unwrap().is_ok());

        let calls = throttle_recorded.lock().unwrap();
        assert!(!calls.is_empty(), "throttle client must be called when backlog is high");
        let (max_tx_size, max_block_size) = calls[0];
        assert!(
            max_block_size < 130_000,
            "max_block_size should be below upper limit when throttled, got {max_block_size}"
        );
        assert!(
            max_tx_size < 20_000,
            "max_tx_size should be below upper limit when throttled, got {max_tx_size}"
        );
    });
}

/// When the DA backlog is zero (below threshold), the driver must call
/// `set_max_da_size` with the upper limits to reset any previous throttle.
#[test]
fn test_throttle_client_called_with_upper_limits_on_zero_backlog() {
    Runner::start(Config::seeded(0), |ctx| async move {
        let recorded = Arc::new(Mutex::new(Recorded::default()));
        let pipeline = TrackingPipeline::new(Arc::clone(&recorded)).with_da_backlog(0);

        let throttle = ThrottleController::new(ThrottleConfig::default(), ThrottleStrategy::Linear);
        let (throttle_client, throttle_recorded) = TrackingThrottleClient::new();

        let driver = BatchDriver::new(
            ctx.clone(),
            pipeline,
            PendingSource,
            ImmediateConfirmTxManager { l1_block: 1 },
            BatchDriverConfig {
                inbox: Address::ZERO,
                max_pending_transactions: 1,
                drain_timeout: Duration::from_millis(10),
            },
            DaThrottle::new(throttle, Arc::new(throttle_client)),
            PendingL1HeadSource,
        );
        let handle = ctx.spawn(driver.run());

        ctx.sleep(Duration::from_millis(50)).await;
        ctx.cancel();
        assert!(handle.await.unwrap().is_ok());

        let calls = throttle_recorded.lock().unwrap();
        assert!(!calls.is_empty(), "throttle client must be called even with zero backlog");
        let (max_tx_size, max_block_size) = calls[0];
        assert_eq!(
            max_block_size, 130_000,
            "max_block_size should be the upper limit when not throttling"
        );
        assert_eq!(
            max_tx_size, 20_000,
            "max_tx_size should be the upper limit when not throttling"
        );
    });
}

/// `set_max_da_size` must be called exactly once when limits do not change
/// between driver loop iterations.
#[test]
fn test_throttle_not_called_redundantly() {
    Runner::start(Config::seeded(0), |ctx| async move {
        let recorded = Arc::new(Mutex::new(Recorded::default()));
        let pipeline = TrackingPipeline::new(Arc::clone(&recorded)).with_da_backlog(0);

        let throttle = ThrottleController::new(ThrottleConfig::default(), ThrottleStrategy::Linear);
        let (throttle_client, throttle_recorded) = TrackingThrottleClient::new();

        let driver = BatchDriver::new(
            ctx.clone(),
            pipeline,
            PendingSource,
            ImmediateConfirmTxManager { l1_block: 1 },
            BatchDriverConfig {
                inbox: Address::ZERO,
                max_pending_transactions: 1,
                drain_timeout: Duration::from_millis(10),
            },
            DaThrottle::new(throttle, Arc::new(throttle_client)),
            PendingL1HeadSource,
        );
        let handle = ctx.spawn(driver.run());

        // Run for 100ms to allow multiple loop iterations.
        ctx.sleep(Duration::from_millis(100)).await;
        ctx.cancel();
        assert!(handle.await.unwrap().is_ok());

        let calls = throttle_recorded.lock().unwrap();
        assert_eq!(
            calls.len(),
            1,
            "set_max_da_size must be called exactly once when limits do not change, got {}",
            calls.len()
        );
    });
}

/// With the Step strategy and full intensity, when backlog is above the
/// threshold, the driver must apply the lower DA limits.
#[test]
fn test_step_strategy_full_intensity_applies_lower_limits() {
    Runner::start(Config::seeded(0), |ctx| async move {
        let recorded = Arc::new(Mutex::new(Recorded::default()));
        // Backlog of 100 — above threshold of 1.
        let pipeline = TrackingPipeline::new(Arc::clone(&recorded)).with_da_backlog(100);

        let config =
            ThrottleConfig { threshold_bytes: 1, max_intensity: 1.0, ..Default::default() };
        let throttle = ThrottleController::new(config, ThrottleStrategy::Step);
        let (throttle_client, throttle_recorded) = TrackingThrottleClient::new();

        let driver = BatchDriver::new(
            ctx.clone(),
            pipeline,
            PendingSource,
            ImmediateConfirmTxManager { l1_block: 1 },
            BatchDriverConfig {
                inbox: Address::ZERO,
                max_pending_transactions: 1,
                drain_timeout: Duration::from_millis(10),
            },
            DaThrottle::new(throttle, Arc::new(throttle_client)),
            PendingL1HeadSource,
        );
        let handle = ctx.spawn(driver.run());

        ctx.sleep(Duration::from_millis(50)).await;
        ctx.cancel();
        assert!(handle.await.unwrap().is_ok());

        let calls = throttle_recorded.lock().unwrap();
        assert!(!calls.is_empty(), "throttle client must be called with Step strategy");
        let (max_tx_size, max_block_size) = calls[0];
        assert_eq!(
            max_block_size, 2_000,
            "Step strategy at full intensity must apply block_size_lower_limit"
        );
        assert_eq!(
            max_tx_size, 150,
            "Step strategy at full intensity must apply tx_size_lower_limit"
        );
    });
}

/// Verifies that when the DA backlog transitions from above the threshold
/// (throttle active) to zero (throttle inactive), the driver makes exactly
/// two RPC calls: one with reduced limits and one resetting to upper limits.
#[test]
fn test_throttle_transitions_from_active_to_inactive() {
    // Pipeline whose DA backlog is controlled from the test via a shared lock.
    struct DynamicPipeline {
        backlog: Arc<Mutex<u64>>,
    }

    impl BatchPipeline for DynamicPipeline {
        fn add_block(&mut self, _: BaseBlock) -> Result<(), (ReorgError, Box<BaseBlock>)> {
            Ok(())
        }

        fn step(&mut self) -> Result<StepResult, StepError> {
            Ok(StepResult::Idle)
        }

        fn next_submission(&mut self) -> Option<BatchSubmission> {
            None
        }

        fn confirm(&mut self, _: SubmissionId, _: u64) {}
        fn requeue(&mut self, _: SubmissionId) {}
        fn force_close_channel(&mut self) {}
        fn advance_l1_head(&mut self, _: u64) {}
        fn prune_safe(&mut self, _: u64) {}
        fn reset(&mut self) {}

        fn da_backlog_bytes(&self) -> u64 {
            *self.backlog.lock().unwrap()
        }
    }

    // Source driven by an mpsc channel so the test can wake the driver loop
    // by sending a dummy block event after changing the backlog.
    struct ChannelSource {
        rx: mpsc::UnboundedReceiver<L2BlockEvent>,
    }

    #[async_trait]
    impl UnsafeBlockSource for ChannelSource {
        async fn next(&mut self) -> Result<L2BlockEvent, SourceError> {
            match self.rx.recv().await {
                Some(event) => Ok(event),
                // Channel closed: park until the driver is cancelled.
                None => std::future::pending().await,
            }
        }
    }

    Runner::start(Config::seeded(0), |ctx| async move {
        let (source_tx, source_rx) = mpsc::unbounded_channel();

        // Start with 2 MB backlog — above the default 1 MB threshold.
        let backlog = Arc::new(Mutex::new(2_000_000u64));
        let pipeline = DynamicPipeline { backlog: Arc::clone(&backlog) };

        let throttle = ThrottleController::new(ThrottleConfig::default(), ThrottleStrategy::Linear);
        let (throttle_client, throttle_recorded) = TrackingThrottleClient::new();

        let driver = BatchDriver::new(
            ctx.clone(),
            pipeline,
            ChannelSource { rx: source_rx },
            ImmediateConfirmTxManager { l1_block: 1 },
            BatchDriverConfig {
                inbox: Address::ZERO,
                max_pending_transactions: 1,
                drain_timeout: Duration::from_millis(10),
            },
            DaThrottle::new(throttle, Arc::new(throttle_client)),
            PendingL1HeadSource,
        );
        let handle = ctx.spawn(driver.run());

        // First iteration fires immediately on startup; give it time to complete.
        ctx.sleep(Duration::from_millis(30)).await;

        // Drop the backlog to zero, then wake the driver by delivering a dummy
        // block so the select! arm fires and the loop re-runs the throttle check.
        *backlog.lock().unwrap() = 0;
        source_tx.send(L2BlockEvent::Block(Box::default())).unwrap();

        ctx.sleep(Duration::from_millis(30)).await;
        ctx.cancel();
        assert!(handle.await.unwrap().is_ok());

        let calls = throttle_recorded.lock().unwrap();
        assert!(
            calls.len() >= 2,
            "expected at least 2 throttle calls (activate + deactivate), got {}",
            calls.len()
        );

        // First call must have reduced limits (throttle active, backlog was high).
        let (first_tx, first_block) = calls[0];
        assert!(
            first_block < 130_000,
            "first call should apply throttled block limit, got {first_block}"
        );
        assert!(first_tx < 20_000, "first call should apply throttled tx limit, got {first_tx}");

        // Last call must reset to upper limits (throttle deactivated).
        let (last_tx, last_block) = *calls.last().unwrap();
        assert_eq!(last_block, 130_000, "last call should reset block limit to upper bound");
        assert_eq!(last_tx, 20_000, "last call should reset tx limit to upper bound");
    });
}

#[derive(Debug, Default)]
struct EncodedButUnpublishedState {
    queued_da_bytes: u64,
    unpublished_da_bytes: u64,
    step_calls: usize,
}

#[derive(Debug)]
struct EncodedButUnpublishedPipeline {
    state: Arc<Mutex<EncodedButUnpublishedState>>,
}

impl BatchPipeline for EncodedButUnpublishedPipeline {
    fn add_block(&mut self, _: BaseBlock) -> Result<(), (ReorgError, Box<BaseBlock>)> {
        Ok(())
    }

    fn step(&mut self) -> Result<StepResult, StepError> {
        let mut state = self.state.lock().unwrap();
        if state.step_calls == 0 {
            state.step_calls += 1;
            state.unpublished_da_bytes = state.queued_da_bytes;
            state.queued_da_bytes = 0;
            return Ok(StepResult::BlockEncoded);
        }
        Ok(StepResult::Idle)
    }

    fn next_submission(&mut self) -> Option<BatchSubmission> {
        None
    }

    fn confirm(&mut self, _: SubmissionId, _: u64) {}
    fn requeue(&mut self, _: SubmissionId) {}
    fn force_close_channel(&mut self) {}
    fn advance_l1_head(&mut self, _: u64) {}
    fn prune_safe(&mut self, _: u64) {}
    fn reset(&mut self) {}

    fn da_backlog_bytes(&self) -> u64 {
        self.state.lock().unwrap().queued_da_bytes
    }
}

/// PoC for the driver-side impact of the DA backlog blind spot. The driver
/// drains encoding before applying throttle. If encoding moves DA from the
/// queued set into unpublished frames, `da_backlog_bytes()` can report zero
/// while the data is still waiting for L1 publication, so the builder is left
/// at unthrottled limits despite backlog above the default 1 MB threshold.
#[test]
fn poc_driver_leaves_builder_unthrottled_with_unpublished_da_backlog() {
    const DEFAULT_THROTTLE_THRESHOLD: u64 = 1_000_000;
    const UNTHROTTLED_MAX_TX_SIZE: u64 = 20_000;
    const UNTHROTTLED_MAX_BLOCK_SIZE: u64 = 130_000;

    Runner::start(Config::seeded(0), |ctx| async move {
        let state = Arc::new(Mutex::new(EncodedButUnpublishedState {
            queued_da_bytes: 2 * DEFAULT_THROTTLE_THRESHOLD,
            ..Default::default()
        }));
        let pipeline = EncodedButUnpublishedPipeline { state: Arc::clone(&state) };

        let throttle = ThrottleController::new(ThrottleConfig::default(), ThrottleStrategy::Linear);
        let (throttle_client, throttle_recorded) = TrackingThrottleClient::new();

        let driver = BatchDriver::new(
            ctx.clone(),
            pipeline,
            PendingSource,
            ImmediateConfirmTxManager { l1_block: 1 },
            BatchDriverConfig {
                inbox: Address::ZERO,
                max_pending_transactions: 1,
                drain_timeout: Duration::from_millis(10),
            },
            DaThrottle::new(throttle, Arc::new(throttle_client)),
            PendingL1HeadSource,
        );
        let handle = ctx.spawn(driver.run());

        ctx.sleep(Duration::from_millis(50)).await;
        ctx.cancel();
        assert!(handle.await.unwrap().is_ok());

        let state = state.lock().unwrap();
        assert_eq!(state.step_calls, 1, "encoding must run before throttle is applied");
        assert!(
            state.unpublished_da_bytes > DEFAULT_THROTTLE_THRESHOLD,
            "DA remains unpublished and above the throttle threshold"
        );
        assert_eq!(
            state.queued_da_bytes, 0,
            "the backlog signal no longer includes encoded unpublished DA"
        );

        let calls = throttle_recorded.lock().unwrap();
        assert!(!calls.is_empty(), "driver must apply DA limits on startup");
        assert_eq!(
            calls[0],
            (UNTHROTTLED_MAX_TX_SIZE, UNTHROTTLED_MAX_BLOCK_SIZE),
            "builder is left unthrottled while unpublished DA exceeds the threshold"
        );
    });
}
```
