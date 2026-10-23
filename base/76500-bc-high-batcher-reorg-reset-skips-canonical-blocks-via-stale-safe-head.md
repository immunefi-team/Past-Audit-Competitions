# 76500 bc high batcher reorg reset skips canonical blocks via stale safe head

\#76500 \[BC-High] Batcher Reorg Reset Skips Canonical Blocks via Stale Safe Head

Submitted on May 4th 2026 at 17:35:16 UTC by @v\_c0d35 for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76500
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

### Description

## Batcher Reorg Reset Skips Canonical Blocks via Stale Safe Head

### Brief/Intro

The Base batcher receives an L2 reorg event that includes the new post-reorg L2 head, but the driver ignores that event head and resets catchup from a separately watched safe-head number. Because the watched safe-head value is only updated when it increases, it can remain stale-high after the rollup node's actual `safe_l2` regresses. In production, this can make the batcher skip canonical L2 blocks after a reorg and submit later blocks that verifiers cannot safely derive until the missing prefix is posted, delaying L1 data availability and safe/finalized progress for affected user transactions.

### Vulnerability Details

The reorg event type carries the block reference that should drive the reset. `L2BlockEvent::Reorg` is documented as a signal that all state should be rewound to `new_safe_head`:

```rust
/// An L2 reorg was detected; all state should be rewound to `new_safe_head`.
Reorg {
    /// The new safe head after the reorg.
    new_safe_head: L2BlockInfo,
},
```

Source: [`crates/batcher/source/src/event.rs#L7-L15`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/batcher/source/src/event.rs#L7-L15)

The hybrid L2 block source creates this event when it observes the same block number with a different hash. It constructs `new_safe_head` directly from the reorg block and emits it as the event payload:

```rust
Some(_) => {
    // Same number, different hash - reorg detected.
    ...
    let block_info = BlockInfo::from(&block);
    ...
    let new_safe_head = ... L2BlockInfo::new(block_info, ...);
    ...
    Some(L2BlockEvent::Reorg { new_safe_head })
}
```

Source: [`crates/batcher/source/src/hybrid.rs#L76-L118`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/batcher/source/src/hybrid.rs#L76-L118)

The driver receives the event head, but does not use it for the reset. The event is converted into `DriverEvent::Reorg(new_safe_head)`:

```rust
Ok(L2BlockEvent::Reorg { new_safe_head }) => DriverEvent::Reorg(new_safe_head),
```

Source: [`crates/batcher/core/src/driver.rs#L345-L351`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/batcher/core/src/driver.rs#L345-L351)

In the actual reset handler, the event head is only logged as `reorg_head`. The reset target is computed from `self.safe_head_rx.borrow()` instead:

```rust
DriverEvent::Reorg(head) => {
    let safe_head = self.safe_head_rx.as_ref().map(|rx| *rx.borrow()).unwrap_or(0);
    let catchup_from = safe_head + 1;
    warn!(
        reorg_head = %head.block_info.number,
        safe_head = %safe_head,
        catchup_from = %catchup_from,
        "L2 reorg detected, resetting pipeline and catching up from safe head"
    );
    self.submissions.discard();
    self.pipeline.reset();
    self.source.reset_catchup(catchup_from);
}
```

Source: [`crates/batcher/core/src/driver.rs#L182-L193`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/batcher/core/src/driver.rs#L182-L193)

The same pattern also exists in the block-ingestion reorg path, where `add_block` detects a parent mismatch but catchup is again reset from the watched safe head rather than from the returned block/reorg context:

```rust
Err((e, _block)) => {
    let safe_head = self.safe_head_rx.as_ref().map(|rx| *rx.borrow()).unwrap_or(0);
    let catchup_from = safe_head + 1;
    ...
    self.submissions.discard();
    self.pipeline.reset();
    self.source.reset_catchup(catchup_from);
}
```

Source: [`crates/batcher/core/src/driver.rs#L250-L268`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/batcher/core/src/driver.rs#L250-L268)

This would be less dangerous if the watched safe head could never be stale above the rollup node's actual safe head. However, the watched value is explicitly an advancing-only `u64`. At startup, the batcher reads `optimism_syncStatus.safe_l2.block_info.number`:

```rust
let sync_status = rollup_client
    .sync_status()
    .await
    .map_err(|e| eyre::eyre!("optimism_syncStatus RPC failed: {e}"))?;
let safe_l2_number = sync_status.safe_l2.block_info.number;
```

Source: [`crates/batcher/service/src/service.rs#L309-L318`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/batcher/service/src/service.rs#L309-L318)

It then creates a watch channel from that number and starts a poller:

```rust
let (safe_head_tx, safe_head_rx) = watch::channel::<u64>(safe_l2_number);
...
SafeHeadPoller::new(rollup_client, self.config.poll_interval, safe_head_tx)
    .spawn(runtime.token().clone());
```

Source: [`crates/batcher/service/src/service.rs#L452-L461`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/batcher/service/src/service.rs#L452-L461)

The poller obtains the number from `optimism_syncStatus.safe_l2`:

```rust
let status = self.sync_status().await?;
Ok(status.safe_l2.block_info.number)
```

Source: [`crates/batcher/service/src/safe_head_poller.rs#L20-L25`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/batcher/service/src/safe_head_poller.rs#L20-L25)

But it only mutates the watched value when the new number is greater than the old value:

```rust
self.safe_head_tx.send_if_modified(|old| {
    if n > *old {
        *old = n;
        true
    } else {
        false
    }
});
```

Source: [`crates/batcher/service/src/safe_head_poller.rs#L63-L74`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/batcher/service/src/safe_head_poller.rs#L63-L74)

The rollup RPC's `safe_l2` is not monotonic in the same way. The RPC response is derived from the engine sync state's `safe_head()`:

```rust
unsafe_l2: l2_sync_status.sync_state.unsafe_head(),
local_safe_l2: l2_sync_status.sync_state.local_safe_head(),
safe_l2: l2_sync_status.sync_state.safe_head(),
finalized_l2: l2_sync_status.sync_state.finalized_head(),
```

Source: [`crates/consensus/rpc/src/rollup.rs#L59-L72`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/rpc/src/rollup.rs#L59-L72)

`EngineSyncState::apply_update` replaces `safe_head` with an incoming `safe_head` update without an increasing-number guard:

```rust
Self {
    unsafe_head: sync_state_update.unsafe_head.unwrap_or(self.unsafe_head),
    local_safe_head: sync_state_update.local_safe_head.unwrap_or(self.local_safe_head),
    safe_head: sync_state_update.safe_head.unwrap_or(self.safe_head),
    finalized_head: sync_state_update.finalized_head.unwrap_or(self.finalized_head),
}
```

Source: [`crates/consensus/engine/src/state/core.rs#L77-L108`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/engine/src/state/core.rs#L77-L108)

As a result, a reorg or safe-head reconciliation can make `optimism_syncStatus.safe_l2` move from `S` down to `R`, while the batcher's watch channel remains stuck at `S`. If a reorg event with `new_safe_head = R` is delivered while the watch still contains `S`, the driver calls `reset_catchup(S + 1)` instead of `reset_catchup(R + 1)`.

The catchup source obeys the supplied start number exactly. While `next_sequential` is set, it fetches block `n`, increments to `n + 1`, and returns the block:

```rust
if let Some(n) = sequential {
    let latest_number = self.provider.get_block_number().await?;
    if n > latest_number {
        *self.next_sequential.lock().unwrap() = None;
    } else {
        let block = self.provider.get_block_by_number(n.into()).full().await?...;
        *self.next_sequential.lock().unwrap() = Some(n + 1);
        return Ok(block);
    }
}
...
fn reset_catchup(&self, start_from: u64) {
    *self.next_sequential.lock().unwrap() = Some(start_from);
}
```

Source: [`crates/batcher/service/src/source.rs#L45-L88`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/batcher/service/src/source.rs#L45-L88)

The hybrid source's reset path simply clears deduplication state and delegates to that poller:

```rust
fn reset_catchup(&mut self, start_from: u64) {
    self.seen.clear();
    self.poller.reset_catchup(start_from);
}
```

Source: [`crates/batcher/source/src/hybrid.rs#L130-L138`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/batcher/source/src/hybrid.rs#L130-L138)

The encoder reset removes the previous tip and pending channel state:

```rust
self.blocks.clear();
self.block_cursor = 0;
self.tip = B256::ZERO;
self.current_channel = None;
self.ready_channels.clear();
self.pending.clear();
```

Source: [`crates/batcher/encoder/src/encoder.rs#L644-L656`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/batcher/encoder/src/encoder.rs#L644-L656)

After that reset, the encoder's parent-hash continuity check only runs when `self.blocks` is non-empty:

```rust
fn add_block(&mut self, block: BaseBlock) -> Result<(), (ReorgError, Box<BaseBlock>)> {
    if !self.blocks.is_empty() && block.header.parent_hash != self.tip {
        return Err((ReorgError::ParentMismatch { expected: self.tip, got: block.header.parent_hash }, Box::new(block)));
    }
    ...
    self.blocks.push_back(block);
}
```

Source: [`crates/batcher/encoder/src/encoder.rs#L339-L351`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/batcher/encoder/src/encoder.rs#L339-L351)

This means the first block delivered after the wrong reset point is accepted as the beginning of a new channel even if the canonical blocks between the reorg head and the stale watched safe head were never re-delivered to the encoder.

A minimal deterministic sequence is:

1. The safe-head watch records `S = 42`.
2. The engine/RPC safe head later regresses to `R = 5`.
3. The safe-head poller receives `5`, but ignores it because `5 > 42` is false.
4. The L2 block source emits `L2BlockEvent::Reorg { new_safe_head: R }`.
5. The driver logs `reorg_head = 5`, but calls `reset_catchup(43)`.
6. Sequential catchup fetches and encodes blocks `43, 44, 45`.
7. Canonical blocks `6..42` are never fed to the encoder in that reset cycle.

This directly violates the batcher protocol requirement that channels cover contiguous, non-overlapping L2 block ranges and that no blocks be skipped between channels. The public Base batcher spec states that after a reorg the batcher must discard pending encoding/submission state and restart from the new canonical chain tip, and that no blocks may be skipped between consecutive channels:

Source: [`docs/specs/pages/protocol/batcher.md#L39-L45`](https://github.com/base/base/blob/v0.8.0-rc.28/docs/specs/pages/protocol/batcher.md#L39-L45)

The downstream derivation rules then prevent later batches from advancing the safe head until the missing prefix is available. A batch whose timestamp is ahead of the next expected L2 timestamp is treated as `future`, and a batch whose parent hash does not equal the current safe L2 head hash is dropped:

Source: [`docs/specs/pages/protocol/consensus/derivation.md#L624-L631`](https://github.com/base/base/blob/v0.8.0-rc.28/docs/specs/pages/protocol/consensus/derivation.md#L624-L631)

### Impact Details

The batcher is the data availability service that posts L2 sequencer data to L1 so validators can reconstruct the L2 chain from L1. The public protocol overview describes the batcher as compressing L2 transaction data into channel frames and posting them to L1, allowing validators to independently reconstruct the L2 chain:

Source: [`docs/specs/pages/protocol/overview.md#L209-L213`](https://github.com/base/base/blob/v0.8.0-rc.28/docs/specs/pages/protocol/overview.md#L209-L213)

When this bug is triggered, the affected batcher can omit the canonical block range `R+1..S` after a reorg and continue submitting later blocks from `S+1`. Those later submissions do not repair the gap because the derivation pipeline expects the next batch after the current safe head. The missing prefix prevents safe derivation from progressing through the skipped user transactions until the omitted canonical range is posted in a valid channel and included on L1.

The user-visible consequences are delayed L1 data availability, delayed safe/finalized progress for the affected L2 transactions, delayed downstream proof/proposer workflows that depend on safe or finalized L2 state, and wasted L1 submission fees for later channels that cannot be used until the missing prefix is supplied. If the sequencer continues producing blocks while the batcher is posting from the wrong cursor, the unsafe/safe gap can grow and the batcher's data availability backlog can increase.

The delay scales with `S - R`. In the concrete sequence above, the batcher skips 37 L2 blocks (`6..42`) and first submits block `43`. The Base derivation spec describes the L2 block time as a configurable parameter and notes 2 seconds for the Optimism/Base-style block cadence:

Source: [`docs/specs/pages/protocol/consensus/derivation.md#L86-L100`](https://github.com/base/base/blob/v0.8.0-rc.28/docs/specs/pages/protocol/consensus/derivation.md#L86-L100)

At a 2 second cadence, the example 37-block gap corresponds to 74 seconds of L2 block data whose canonical batcher data is omitted before the first post-reset submission. Larger stale-safe-head gaps produce proportionally larger delays. Recovery requires the missing canonical range to be reposted from the correct cursor, for example through a corrected reset path or operator recovery that starts from the actual post-reorg safe head.

The default batcher CLI configuration polls the safe head every 1 second and uses a 48 second transaction resubmission timeout:

Sources: [`bin/batcher/src/cli.rs#L79-L81`](https://github.com/base/base/blob/v0.8.0-rc.28/bin/batcher/src/cli.rs#L79-L81), [`bin/batcher/src/cli.rs#L139-L145`](https://github.com/base/base/blob/v0.8.0-rc.28/bin/batcher/src/cli.rs#L139-L145)

Those defaults do not correct the skipped canonical range: polling lower `safe_l2` values is suppressed by the safe-head poller, and resubmission only retries pending submissions from the current encoder/submission state. The stale high reset cursor remains the root cause until the batcher is made to replay from the actual reorg head.

### References

* [`crates/batcher/source/src/event.rs#L7-L15`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/batcher/source/src/event.rs#L7-L15)
* [`crates/batcher/source/src/hybrid.rs#L76-L118`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/batcher/source/src/hybrid.rs#L76-L118)
* [`crates/batcher/core/src/driver.rs#L182-L193`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/batcher/core/src/driver.rs#L182-L193)
* [`crates/batcher/core/src/driver.rs#L250-L268`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/batcher/core/src/driver.rs#L250-L268)
* [`crates/batcher/service/src/safe_head_poller.rs#L20-L25`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/batcher/service/src/safe_head_poller.rs#L20-L25)
* [`crates/batcher/service/src/safe_head_poller.rs#L63-L74`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/batcher/service/src/safe_head_poller.rs#L63-L74)
* [`crates/consensus/rpc/src/rollup.rs#L59-L72`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/rpc/src/rollup.rs#L59-L72)
* [`crates/consensus/engine/src/state/core.rs#L77-L108`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/engine/src/state/core.rs#L77-L108)
* [`crates/batcher/service/src/source.rs#L45-L88`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/batcher/service/src/source.rs#L45-L88)
* [`crates/batcher/encoder/src/encoder.rs#L339-L351`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/batcher/encoder/src/encoder.rs#L339-L351)
* [`crates/batcher/encoder/src/encoder.rs#L644-L656`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/batcher/encoder/src/encoder.rs#L644-L656)
* [`docs/specs/pages/protocol/batcher.md#L39-L45`](https://github.com/base/base/blob/v0.8.0-rc.28/docs/specs/pages/protocol/batcher.md#L39-L45)
* [`docs/specs/pages/protocol/consensus/derivation.md#L624-L631`](https://github.com/base/base/blob/v0.8.0-rc.28/docs/specs/pages/protocol/consensus/derivation.md#L624-L631)
* [`docs/specs/pages/protocol/overview.md#L209-L213`](https://github.com/base/base/blob/v0.8.0-rc.28/docs/specs/pages/protocol/overview.md#L209-L213)
* [Base v1 upgrade documentation](https://docs.base.org/base-chain/node-operators/base-v1-upgrade)

### Proof of Concept

## Proof of Concept: Batcher Reorg Reset Skips Canonical Blocks via Stale Safe Head

This guide reproduces the vulnerability on the public `base/base` repository at tag `v0.8.0-rc.28`. It adds three deterministic tests:

* `base-consensus-engine`: proves engine sync state can move `safe_l2` downward.
* `base-batcher-service`: proves the batcher safe-head watch ignores that downward move and remains stale-high.
* `base-batcher-core`: proves a reorg event then resets catchup from the stale watched safe head, and a real `BatchEncoder` submits later blocks while omitting the canonical range after the reorg head.

{% stepper %}
{% step %}

### 1. Checkout the Public Repository

Run:

```bash
git clone --branch v0.8.0-rc.28 --depth 1 https://github.com/base/base.git base-reorg-reset-poc
cd base-reorg-reset-poc
```

{% endstep %}

{% step %}

### 2. Add One Test Dependency

Open `crates/batcher/core/Cargo.toml` and add this line under `[dev-dependencies]`:

```toml
base-consensus-genesis.workspace = true
```

For example, the `[dev-dependencies]` section should become:

```toml
[dev-dependencies]
rstest.workspace = true
async-trait.workspace = true
alloy-consensus = { workspace = true, features = ["std"] }
base-runtime = { workspace = true, features = ["test-utils"] }
alloy-rpc-types-eth = { workspace = true, features = ["std"] }
base-consensus-genesis.workspace = true
tokio = { workspace = true, features = ["macros", "rt", "time"] }
```

This dependency is already part of the workspace; it is only needed by the test below to construct a real `BatchEncoder`.
{% endstep %}

{% step %}

### 3. Add Engine Safe-Head Regression Test

Save the following file to `crates/consensus/engine/tests/safe_head_regression.rs`:

```rust
//! Regression coverage for engine safe-head state after a reorg-style downward update.

use alloy_primitives::B256;
use base_consensus_engine::{EngineSyncState, EngineSyncStateUpdate};
use base_protocol::{BlockInfo, L2BlockInfo};

fn l2_info(number: u64) -> L2BlockInfo {
    L2BlockInfo::new(
        BlockInfo::new(B256::with_last_byte(number as u8), number, B256::ZERO, number * 2),
        Default::default(),
        0,
    )
}

#[test]
fn engine_sync_state_accepts_lower_safe_head_update() {
    let high = l2_info(42);
    let low = l2_info(5);

    let state = EngineSyncState::default()
        .apply_update(EngineSyncStateUpdate { safe_head: Some(high), ..Default::default() })
        .apply_update(EngineSyncStateUpdate { safe_head: Some(low), ..Default::default() });

    assert_eq!(
        state.safe_head().block_info.number,
        5,
        "engine/RPC state can move safe_l2 downward after a reorg"
    );
}
```

{% endstep %}

{% step %}

### 4. Add Batcher Safe-Head Watch Regression Test

Save the following file to `crates/batcher/service/tests/safe_head_regression.rs`:

```rust
//! Regression coverage for safe-head watch updates across downward sync-status moves.

use std::{
    collections::VecDeque,
    sync::{Arc, Mutex},
    time::Duration,
};

use base_batcher_service::{SafeHeadPoller, SafeHeadProvider};
use tokio::sync::watch;
use tokio_util::sync::CancellationToken;

#[derive(Clone, Debug)]
struct RegressingProvider {
    values: Arc<Mutex<VecDeque<u64>>>,
}

impl SafeHeadProvider for RegressingProvider {
    async fn safe_l2_number(&self) -> Result<u64, Box<dyn std::error::Error + Send + Sync>> {
        Ok(self.values.lock().unwrap().pop_front().unwrap_or(5))
    }
}

#[tokio::test]
async fn poller_keeps_stale_high_value_when_sync_status_regresses() {
    let provider =
        RegressingProvider { values: Arc::new(Mutex::new(VecDeque::from([42, 5, 5, 5]))) };
    let (tx, mut rx) = watch::channel(0u64);
    let cancellation = CancellationToken::new();

    let handle =
        SafeHeadPoller::new(provider, Duration::from_millis(1), tx).spawn(cancellation.clone());

    tokio::time::timeout(Duration::from_millis(200), rx.changed())
        .await
        .expect("poller should observe the first high safe head")
        .expect("watch sender should stay live");
    assert_eq!(*rx.borrow_and_update(), 42);

    tokio::time::sleep(Duration::from_millis(30)).await;
    assert_eq!(
        *rx.borrow(),
        42,
        "lower optimism_syncStatus.safe_l2 values are ignored, leaving stale-high watch state"
    );

    let changed = tokio::time::timeout(Duration::from_millis(5), rx.changed()).await;
    assert!(changed.is_err(), "safe-head watch should not notify on the downward regression");

    cancellation.cancel();
    handle.await.unwrap();
}
```

{% endstep %}

{% step %}

### 5. Add Driver and Real Encoder Reorg Reset PoC

Save the following file to `crates/batcher/core/tests/reorg_reset_mismatch.rs`:

```rust
//! Deterministic reproduction for reorg reset source mismatch.

use std::{
    collections::{BTreeMap, VecDeque},
    sync::{Arc, Mutex},
    time::Duration,
};

use alloy_consensus::{BlockBody, Eip658Value, Header, Receipt, ReceiptEnvelope, ReceiptWithBloom};
use alloy_primitives::{Address, B256, Bloom, Sealed};
use alloy_rpc_types_eth::TransactionReceipt;
use async_trait::async_trait;
use base_batcher_core::{
    BatchDriver, BatchDriverConfig, DaThrottle, NoopThrottleClient, ThrottleController,
    test_utils::PendingL1HeadSource,
};
use base_batcher_encoder::{
    BatchEncoder, BatchPipeline, BatchSubmission, DaType, EncoderConfig, ReorgError, StepError,
    StepResult, SubmissionId,
};
use base_batcher_source::{L2BlockEvent, SourceError, UnsafeBlockSource};
use base_common_consensus::{BaseBlock, BaseTxEnvelope, TxDeposit};
use base_consensus_genesis::RollupConfig;
use base_protocol::{BlockInfo, L1BlockInfoBedrock, L1BlockInfoTx, L2BlockInfo};
use base_runtime::{
    Cancellation, Clock, Spawner,
    deterministic::{Config, Runner},
};
use base_tx_manager::{SendHandle, SendResponse, TxCandidate, TxManager};
use tokio::sync::{oneshot, watch};

#[derive(Debug)]
struct CatchupSource {
    events: VecDeque<L2BlockEvent>,
    blocks: BTreeMap<u64, BaseBlock>,
    latest: u64,
    next_catchup: Option<u64>,
    reset_args: Arc<Mutex<Vec<u64>>>,
}

impl CatchupSource {
    fn new(
        events: Vec<L2BlockEvent>,
        blocks: BTreeMap<u64, BaseBlock>,
    ) -> (Self, Arc<Mutex<Vec<u64>>>) {
        let reset_args = Arc::new(Mutex::new(Vec::new()));
        let latest = blocks.keys().next_back().copied().unwrap_or_default();
        (
            Self {
                events: VecDeque::from(events),
                blocks,
                latest,
                next_catchup: None,
                reset_args: Arc::clone(&reset_args),
            },
            reset_args,
        )
    }
}

#[async_trait]
impl UnsafeBlockSource for CatchupSource {
    async fn next(&mut self) -> Result<L2BlockEvent, SourceError> {
        if let Some(n) = self.next_catchup {
            if n <= self.latest {
                self.next_catchup = Some(n + 1);
                let block =
                    self.blocks.get(&n).unwrap_or_else(|| panic!("missing canonical block {n}"));
                return Ok(L2BlockEvent::Block(Box::new(block.clone())));
            }
            self.next_catchup = None;
        }

        if let Some(event) = self.events.pop_front() {
            return Ok(event);
        }

        std::future::pending::<Result<L2BlockEvent, SourceError>>().await
    }

    fn reset_catchup(&mut self, start_from: u64) {
        self.reset_args.lock().unwrap().push(start_from);
        self.next_catchup = Some(start_from);
    }
}

#[derive(Debug)]
struct RecordingPipeline {
    added_blocks: Arc<Mutex<Vec<u64>>>,
    resets: Arc<Mutex<usize>>,
}

impl RecordingPipeline {
    fn new() -> (Self, Arc<Mutex<Vec<u64>>>, Arc<Mutex<usize>>) {
        let added_blocks = Arc::new(Mutex::new(Vec::new()));
        let resets = Arc::new(Mutex::new(0));
        (
            Self { added_blocks: Arc::clone(&added_blocks), resets: Arc::clone(&resets) },
            added_blocks,
            resets,
        )
    }
}

impl BatchPipeline for RecordingPipeline {
    fn add_block(&mut self, block: BaseBlock) -> Result<(), (ReorgError, Box<BaseBlock>)> {
        self.added_blocks.lock().unwrap().push(block.header.number);
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

    fn reset(&mut self) {
        *self.resets.lock().unwrap() += 1;
    }

    fn da_backlog_bytes(&self) -> u64 {
        0
    }
}

#[derive(Debug)]
struct ObservedEncoder {
    inner: BatchEncoder,
    accepted_blocks: Arc<Mutex<Vec<u64>>>,
    submission_snapshots: Arc<Mutex<Vec<Vec<u64>>>>,
}

impl ObservedEncoder {
    fn new() -> (Self, Arc<Mutex<Vec<u64>>>, Arc<Mutex<Vec<Vec<u64>>>>) {
        let accepted_blocks = Arc::new(Mutex::new(Vec::new()));
        let submission_snapshots = Arc::new(Mutex::new(Vec::new()));
        let encoder_config = EncoderConfig {
            da_type: DaType::Calldata,
            target_num_frames: 1,
            target_frame_size: 130_044,
            max_frame_size: 130_044,
            max_channel_duration: 100,
            ..EncoderConfig::default()
        };
        (
            Self {
                inner: BatchEncoder::new(Arc::new(RollupConfig::default()), encoder_config),
                accepted_blocks: Arc::clone(&accepted_blocks),
                submission_snapshots: Arc::clone(&submission_snapshots),
            },
            accepted_blocks,
            submission_snapshots,
        )
    }
}

impl BatchPipeline for ObservedEncoder {
    fn add_block(&mut self, block: BaseBlock) -> Result<(), (ReorgError, Box<BaseBlock>)> {
        let number = block.header.number;
        match self.inner.add_block(block) {
            Ok(()) => {
                self.accepted_blocks.lock().unwrap().push(number);
                Ok(())
            }
            Err(e) => Err(e),
        }
    }

    fn step(&mut self) -> Result<StepResult, StepError> {
        self.inner.step()
    }

    fn next_submission(&mut self) -> Option<BatchSubmission> {
        let submission = self.inner.next_submission();
        if submission.is_some() {
            self.submission_snapshots
                .lock()
                .unwrap()
                .push(self.accepted_blocks.lock().unwrap().clone());
        }
        submission
    }

    fn confirm(&mut self, id: SubmissionId, l1_block: u64) {
        self.inner.confirm(id, l1_block);
    }

    fn requeue(&mut self, id: SubmissionId) {
        self.inner.requeue(id);
    }

    fn force_close_channel(&mut self) {
        self.inner.force_close_channel();
    }

    fn advance_l1_head(&mut self, l1_block: u64) {
        self.inner.advance_l1_head(l1_block);
    }

    fn prune_safe(&mut self, safe_l2_number: u64) {
        self.inner.prune_safe(safe_l2_number);
    }

    fn reset(&mut self) {
        self.inner.reset();
    }

    fn da_backlog_bytes(&self) -> u64 {
        self.inner.da_backlog_bytes()
    }
}

#[derive(Clone, Debug)]
struct RecordingConfirmTxManager {
    l1_block: u64,
    candidates: Arc<Mutex<Vec<TxCandidate>>>,
}

impl RecordingConfirmTxManager {
    fn new(l1_block: u64) -> (Self, Arc<Mutex<Vec<TxCandidate>>>) {
        let candidates = Arc::new(Mutex::new(Vec::new()));
        (Self { l1_block, candidates: Arc::clone(&candidates) }, candidates)
    }
}

impl TxManager for RecordingConfirmTxManager {
    async fn send(&self, candidate: TxCandidate) -> SendResponse {
        self.candidates.lock().unwrap().push(candidate);
        Ok(stub_receipt(self.l1_block))
    }

    fn send_async(
        &self,
        candidate: TxCandidate,
    ) -> impl std::future::Future<Output = SendHandle> + Send {
        self.candidates.lock().unwrap().push(candidate);
        let (tx, rx) = oneshot::channel();
        let _ = tx.send(Ok(stub_receipt(self.l1_block)));
        std::future::ready(SendHandle::new(rx))
    }

    fn sender_address(&self) -> Address {
        Address::ZERO
    }
}

fn stub_receipt(block_number: u64) -> TransactionReceipt {
    let inner = ReceiptEnvelope::Legacy(ReceiptWithBloom {
        receipt: Receipt {
            status: Eip658Value::Eip658(true),
            cumulative_gas_used: 21_000,
            logs: vec![],
        },
        logs_bloom: Bloom::ZERO,
    });
    TransactionReceipt {
        inner,
        transaction_hash: B256::ZERO,
        transaction_index: Some(0),
        block_hash: Some(B256::ZERO),
        block_number: Some(block_number),
        gas_used: 21_000,
        effective_gas_price: 1_000_000_000,
        blob_gas_used: None,
        blob_gas_price: None,
        from: Address::ZERO,
        to: Some(Address::ZERO),
        contract_address: None,
    }
}

fn make_deposit_tx() -> BaseTxEnvelope {
    let calldata = L1BlockInfoTx::Bedrock(L1BlockInfoBedrock::default()).encode_calldata();
    BaseTxEnvelope::Deposit(Sealed::new(TxDeposit { input: calldata, ..Default::default() }))
}

fn make_block(number: u64, parent_hash: B256) -> BaseBlock {
    BaseBlock {
        header: Header { number, parent_hash, timestamp: number * 2, ..Default::default() },
        body: BlockBody { transactions: vec![make_deposit_tx()], ..Default::default() },
    }
}

fn canonical_blocks(start: u64, latest: u64) -> BTreeMap<u64, BaseBlock> {
    let mut blocks = BTreeMap::new();
    let mut parent_hash = B256::with_last_byte((start.saturating_sub(1) % 255) as u8);
    for number in start..=latest {
        let block = make_block(number, parent_hash);
        parent_hash = block.header.hash_slow();
        blocks.insert(number, block);
    }
    blocks
}

fn make_l2_info(number: u64) -> L2BlockInfo {
    L2BlockInfo::new(BlockInfo::new(B256::ZERO, number, B256::ZERO, 0), Default::default(), 0)
}

#[test]
fn reorg_event_resets_from_watched_safe_head_and_skips_modeled_canonical_range() {
    Runner::start(Config::seeded(0), |ctx| async move {
        let reorg_event_head = 5;
        let watched_safe_head = 42;
        let latest = 45;

        let event = L2BlockEvent::Reorg { new_safe_head: make_l2_info(reorg_event_head) };
        let (source, reset_args) = CatchupSource::new(vec![event], canonical_blocks(6, latest));
        let (pipeline, added_blocks, resets) = RecordingPipeline::new();
        let (safe_head_tx, safe_head_rx) = watch::channel::<u64>(watched_safe_head);

        let driver = BatchDriver::new(
            ctx.clone(),
            pipeline,
            source,
            RecordingConfirmTxManager::new(1).0,
            BatchDriverConfig {
                inbox: Address::ZERO,
                max_pending_transactions: 1,
                drain_timeout: Duration::from_millis(10),
            },
            DaThrottle::new(ThrottleController::noop(), Arc::new(NoopThrottleClient)),
            PendingL1HeadSource,
        )
        .with_safe_head_rx(safe_head_rx);

        let handle = ctx.spawn(driver.run());
        ctx.sleep(Duration::from_millis(50)).await;
        ctx.cancel();

        drop(safe_head_tx);
        assert!(handle.await.unwrap().is_ok());

        let reset_args = reset_args.lock().unwrap().clone();
        let added_blocks = added_blocks.lock().unwrap().clone();

        assert_eq!(*resets.lock().unwrap(), 1, "reorg event must reset the pipeline");
        assert_eq!(reset_args, vec![watched_safe_head + 1]);
        assert_ne!(reset_args[0], reorg_event_head + 1);
        assert_eq!(added_blocks, vec![43, 44, 45]);
        assert!(
            (reorg_event_head + 1..=watched_safe_head).all(|n| !added_blocks.contains(&n)),
            "modeled canonical blocks after the reorg event head were not re-delivered"
        );
    });
}

#[test]
fn reorg_event_with_real_encoder_submits_from_watched_head_and_omits_event_range() {
    Runner::start(Config::seeded(0), |ctx| async move {
        let reorg_event_head = 5;
        let watched_safe_head = 42;
        let latest = 45;
        let l2_block_time_secs = 2;

        let event = L2BlockEvent::Reorg { new_safe_head: make_l2_info(reorg_event_head) };
        let (source, reset_args) = CatchupSource::new(
            vec![event, L2BlockEvent::Flush],
            canonical_blocks(reorg_event_head + 1, latest),
        );
        let (pipeline, accepted_blocks, submission_snapshots) = ObservedEncoder::new();
        let (tx_manager, tx_candidates) = RecordingConfirmTxManager::new(7);
        let (safe_head_tx, safe_head_rx) = watch::channel::<u64>(watched_safe_head);

        let driver = BatchDriver::new(
            ctx.clone(),
            pipeline,
            source,
            tx_manager,
            BatchDriverConfig {
                inbox: Address::ZERO,
                max_pending_transactions: 1,
                drain_timeout: Duration::from_millis(10),
            },
            DaThrottle::new(ThrottleController::noop(), Arc::new(NoopThrottleClient)),
            PendingL1HeadSource,
        )
        .with_safe_head_rx(safe_head_rx);

        let handle = ctx.spawn(driver.run());
        ctx.sleep(Duration::from_millis(100)).await;
        ctx.cancel();

        drop(safe_head_tx);
        assert!(handle.await.unwrap().is_ok());

        let reset_args = reset_args.lock().unwrap().clone();
        let accepted_blocks = accepted_blocks.lock().unwrap().clone();
        let submission_snapshots = submission_snapshots.lock().unwrap().clone();
        let tx_candidates = tx_candidates.lock().unwrap().clone();

        assert_eq!(reset_args, vec![watched_safe_head + 1]);
        assert_eq!(accepted_blocks, vec![43, 44, 45]);
        assert!(
            (reorg_event_head + 1..=watched_safe_head).all(|n| !accepted_blocks.contains(&n)),
            "real encoder was never fed the canonical range after the reorg event head"
        );
        assert!(!tx_candidates.is_empty(), "force-flushed real encoder must submit to L1");
        assert_eq!(submission_snapshots.first(), Some(&vec![43, 44, 45]));
        assert!(
            tx_candidates.iter().all(|candidate| !candidate.tx_data.is_empty()),
            "calldata-mode submissions should carry encoded batch frames"
        );

        let expected_first_l2 = reorg_event_head + 1;
        let actual_first_submitted_l2 = submission_snapshots[0][0];
        let skipped_blocks_before_first_submission = actual_first_submitted_l2 - expected_first_l2;
        let modeled_delay_secs = skipped_blocks_before_first_submission * l2_block_time_secs;

        assert_eq!(skipped_blocks_before_first_submission, 37);
        assert_eq!(modeled_delay_secs, 74);
    });
}
```

{% endstep %}

{% step %}

### 6. Run the PoC

From the repository root, run:

```bash
cargo test -p base-consensus-engine --test safe_head_regression -- --nocapture
cargo test -p base-batcher-service --test safe_head_regression -- --nocapture
cargo test -p base-batcher-core --features test-utils --test reorg_reset_mismatch -- --nocapture
```

The first run may compile a large part of the workspace. If the checkout is on a small temporary filesystem, set `CARGO_TARGET_DIR` to a path with enough free space before running the commands, for example:

```bash
export CARGO_TARGET_DIR="$HOME/base-poc-target"
```

Expected result:

```
test engine_sync_state_accepts_lower_safe_head_update ... ok
test poller_keeps_stale_high_value_when_sync_status_regresses ... ok
test reorg_event_resets_from_watched_safe_head_and_skips_modeled_canonical_range ... ok
test reorg_event_with_real_encoder_submits_from_watched_head_and_omits_event_range ... ok
```

The final `base-batcher-core` test proves the full vulnerable transition:

* reorg event head `R = 5`
* watched safe head `S = 42`
* actual reset target recorded as `43`
* expected reset target would be `6`
* real encoder accepts and submits blocks `[43, 44, 45]`
* canonical blocks `6..42` are omitted from the encoder/submission path
* the modeled omitted range is 37 L2 blocks, equal to 74 seconds at a 2 second L2 block cadence
  {% endstep %}

{% step %}

### 7. Why This Reproduces the Vulnerability

The three tests connect the complete state transition:

1. `base-consensus-engine` shows the rollup node state can move `safe_l2` downward.
2. `base-batcher-service` shows the batcher watch channel ignores that downward value and remains stale-high.
3. `base-batcher-core` shows the driver uses the stale watched value instead of the reorg event head, then the real encoder submits later blocks while skipping the canonical range after the reorg head.

This is the same logic used by the public `v0.8.0-rc.28` batcher source paths:

* Reorg event payload: `crates/batcher/source/src/event.rs`
* Reorg detection and event emission: `crates/batcher/source/src/hybrid.rs`
* Reorg reset decision: `crates/batcher/core/src/driver.rs`
* Safe-head watch updates: `crates/batcher/service/src/safe_head_poller.rs`
* Sequential catchup: `crates/batcher/service/src/source.rs`
* Real batch encoder reset and block acceptance: `crates/batcher/encoder/src/encoder.rs`
  {% endstep %}
  {% endstepper %}
