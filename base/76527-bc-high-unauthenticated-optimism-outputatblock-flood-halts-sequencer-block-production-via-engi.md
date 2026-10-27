# 76527 bc high unauthenticated optimism outputatblock flood halts sequencer block production via engineactor head of line blocking

**Submitted on May 4th 2026 at 18:45:35 UTC by @ZeroExRes for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76527
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

`EngineActor` runs every node request through one `tokio::select!` loop. When the downstream RPC channel fills up, the active arm blocks on `rpc_tx.send().await`, and `select!` can't move to another branch until that await returns. An unauthenticated `optimism_outputAtBlock` flood on the public rollup-node RPC fills the channel, blocks the actor, and starves every build, seal, and block-processing request queued behind it. The sequencer stops confirming L2 transactions until the flood stops.

## Vulnerability Details

The loop in `crates/consensus/service/src/actors/engine/actor.rs`:

```rust
loop {
    tokio::select! {
        _ = self.cancellation_token.cancelled() => { ... }
        req = self.inbound_request_rx.recv() => {
            match request {
                EngineActorRequest::RpcRequest(rpc_req) => {
                    rpc_tx.send(*rpc_req).await...?;          // blocks here
                }
                // BuildRequest, SealRequest, ProcessUnsafe, Reset, GetPayload, ...
                _ = send_engine_processing_request(...).await?,
            }
        }
    }
}
```

`rpc_tx` has capacity 1024. The `EngineRpcProcessor` that drains it holds 16 concurrent permits, and its drain loop calls `recv()` *before* `acquire_owned().await`, so when permits are all in use, items stop coming off the channel. `EngineQueries::OutputAtBlock` runs `l2_block_by_label` against the EL (a DB lookup that's slow for cold/archived blocks); pre-Isthmus it additionally runs `eth_getProof`. Either path is enough to saturate the 16-permit semaphore under sustained load. `optimism_outputAtBlock` is served by `RollupRpc` with no auth; `l2_jwt_secret` only covers the engine RPC (op-node ↔ EL).

Failure sequence:

1. Attacker spams cold-block `optimism_outputAtBlock` faster than the drain rate (a few hundred req/s).
2. The 16 permits fill with slow `eth_getProof`. `rpc_tx` reaches capacity.
3. The actor pulls the next RPC request, hits `rpc_tx.send().await`, blocks.
4. `select!` can't re-enter while that arm is awaiting. Non-RPC requests queued behind it are starved even though their downstream channel is empty.

## Impact Details

"Network not being able to confirm new transactions (total network shutdown)"

With Base's single sequencer, blocking the actor halts block production while the flood is active.

"Temporary freezing of network transactions by delaying one block by 500% or more"

Base targets 2-second blocks. Even 10 seconds of sustained traffic trips the 500% threshold.

Attack profile:

* Unauthenticated. Reachable from any host that can hit the rollup-node JSON-RPC.
* A few hundred cold-block queries per second is enough.
* Liveness recovers within seconds once the flood stops.
* Funds aren't at risk; L1 force-include remains available.

## References

* `crates/consensus/service/src/actors/engine/actor.rs` — vulnerable select loop
* `crates/consensus/service/src/actors/engine/rpc_request_processor.rs` — semaphore + recv-then-acquire
* `crates/consensus/engine/src/query.rs` — slow EL calls behind `OutputAtBlock`
* `crates/consensus/rpc/src/rollup.rs` — public unauthenticated RPC entrypoint

## Proof of Concept

Make these changes

```diff
diff --git a/crates/consensus/service/src/actors/engine/actor.rs b/crates/consensus/service/src/actors/engine/actor.rs
index 1a3f1ee24..adc253be1 100644
--- a/crates/consensus/service/src/actors/engine/actor.rs
+++ b/crates/consensus/service/src/actors/engine/actor.rs
@@ -168,3 +168,145 @@ where
         }
     }
 }
+
+#[cfg(test)]
+mod tests {
+    use std::{
+        sync::{
+            Arc,
+            atomic::{AtomicUsize, Ordering},
+        },
+        time::Duration,
+    };
+
+    use base_consensus_engine::EngineQueries;
+    use tokio::{
+        sync::oneshot,
+        task::JoinHandle,
+        time::{sleep, timeout},
+    };
+
+    use super::*;
+    use crate::{EngineProcessingRequest, EngineRpcRequest};
+
+    struct StalledRpcReceiver;
+
+    impl EngineRpcRequestReceiver for StalledRpcReceiver {
+        fn start(
+            self,
+            request_channel: mpsc::Receiver<EngineRpcRequest>,
+        ) -> JoinHandle<Result<(), EngineError>> {
+            tokio::spawn(async move {
+                let _kept = request_channel;
+                std::future::pending::<()>().await;
+                Ok(())
+            })
+        }
+    }
+
+    struct CountingProcessor(Arc<AtomicUsize>);
+
+    impl EngineRequestReceiver for CountingProcessor {
+        fn start(
+            self,
+            mut request_channel: mpsc::Receiver<EngineProcessingRequest>,
+        ) -> JoinHandle<Result<(), EngineError>> {
+            let counter = self.0;
+            tokio::spawn(async move {
+                while request_channel.recv().await.is_some() {
+                    counter.fetch_add(1, Ordering::SeqCst);
+                }
+                Ok(())
+            })
+        }
+    }
+
+    fn rpc_request() -> EngineActorRequest {
+        let (tx, _rx) = oneshot::channel();
+        EngineActorRequest::RpcRequest(Box::new(EngineRpcRequest::EngineQuery(Box::new(
+            EngineQueries::State(tx),
+        ))))
+    }
+
+    fn processing_request() -> EngineActorRequest {
+        EngineActorRequest::ProcessFinalizedL2BlockNumberRequest(Box::new(42))
+    }
+
+    #[tokio::test(flavor = "multi_thread", worker_threads = 4)]
+    async fn rpc_backpressure_stalls_engine_processing_routing() {
+        let cancellation = CancellationToken::new();
+        // Inbound capacity mirrors the production wiring.
+        let (inbound_tx, inbound_rx) = mpsc::channel(1024);
+        let processed = Arc::new(AtomicUsize::new(0));
+
+        let actor = EngineActor::new(
+            cancellation.clone(),
+            inbound_rx,
+            CountingProcessor(Arc::clone(&processed)),
+            StalledRpcReceiver,
+        );
+        let actor_handle = tokio::spawn(actor.start(()));
+
+        // Fill the actor's internal rpc_tx (capacity 1024).
+        for _ in 0..1024 {
+            inbound_tx.send(rpc_request()).await.unwrap();
+        }
+        sleep(Duration::from_millis(100)).await;
+
+        // The 1025th RPC parks the actor on rpc_tx.send().await.
+        inbound_tx.send(rpc_request()).await.unwrap();
+        sleep(Duration::from_millis(100)).await;
+
+        // engine_processing_tx is empty, but the parked select! arm can't pick this up.
+        inbound_tx.send(processing_request()).await.unwrap();
+        sleep(Duration::from_millis(500)).await;
+
+        let routed = processed.load(Ordering::SeqCst);
+        assert_eq!(routed, 0, "processing request blocked by RPC backpressure (routed={routed})");
+
+        // Cancellation drops rpc_tx, releasing the parked send so the actor exits.
+        cancellation.cancel();
+        let _ = timeout(Duration::from_secs(2), actor_handle).await;
+    }
+
+    #[tokio::test(flavor = "multi_thread", worker_threads = 4)]
+    async fn engine_processing_routes_when_rpc_drains() {
+        struct DrainingRpcReceiver;
+        impl EngineRpcRequestReceiver for DrainingRpcReceiver {
+            fn start(
+                self,
+                mut request_channel: mpsc::Receiver<EngineRpcRequest>,
+            ) -> JoinHandle<Result<(), EngineError>> {
+                tokio::spawn(async move {
+                    while request_channel.recv().await.is_some() {}
+                    Ok(())
+                })
+            }
+        }
+
+        let cancellation = CancellationToken::new();
+        let (inbound_tx, inbound_rx) = mpsc::channel(1024);
+        let processed = Arc::new(AtomicUsize::new(0));
+
+        let actor = EngineActor::new(
+            cancellation.clone(),
+            inbound_rx,
+            CountingProcessor(Arc::clone(&processed)),
+            DrainingRpcReceiver,
+        );
+        let actor_handle = tokio::spawn(actor.start(()));
+
+        for _ in 0..1024 {
+            inbound_tx.send(rpc_request()).await.unwrap();
+        }
+        inbound_tx.send(rpc_request()).await.unwrap();
+        inbound_tx.send(processing_request()).await.unwrap();
+        sleep(Duration::from_millis(200)).await;
+
+        let routed = processed.load(Ordering::SeqCst);
+        assert_eq!(routed, 1, "processing request routes when RPC drains");
+
+        cancellation.cancel();
+        let _ = timeout(Duration::from_secs(2), actor_handle).await;
+    }
+}
```

```bash
$ cargo test -p base-consensus-node --lib actors::engine::actor::tests
running 2 tests
test actors::engine::actor::tests::engine_processing_routes_when_rpc_drains ... ok
test actors::engine::actor::tests::rpc_backpressure_stalls_engine_processing_routing ... ok
```

`rpc_backpressure_stalls_engine_processing_routing`: real `EngineActor` with a stalled mock RPC receiver. Once `rpc_tx` is full and the actor is blocked, a `ProcessFinalizedL2BlockNumberRequest` sent on the inbound channel never reaches the engine processor, which is idle with an empty queue.

`engine_processing_routes_when_rpc_drains`: same harness, draining receiver. The same processing request routes through. One variable changed, opposite results.
