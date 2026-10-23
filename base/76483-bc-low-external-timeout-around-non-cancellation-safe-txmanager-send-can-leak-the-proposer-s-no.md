# 76483 bc low external timeout around non cancellation safe txmanager send can leak the proposer s nonce and stall l1 proposal submission

**Submitted on May 4th 2026 at 16:28:45 UTC by @ZeroExRes for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76483
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Brief/Intro

The proposer wraps L1 dispute game creation in a 10-minute `tokio::time::timeout`, but the transaction path underneath it is not safe to cancel during initial preparation. If that outer timeout fires after the TxManager reserves a nonce but before the transaction is published, the nonce is lost from the in-memory nonce manager. After that, the proposer can keep signing later transactions with higher nonces that cannot be mined because the missing nonce was never published.

## Vulnerability Details

`SimpleTxManager::prepare()` documents the exact problem:

```rust
/// This future is **not** cancellation-safe.
/// ...
/// Callers must not wrap this future in `tokio::select!`,
/// `tokio::time::timeout`, or similar combinators that may drop it
/// mid-flight.
```

The proposer does exactly that during submission:

```rust
let propose_result = tokio::time::timeout(
    PROPOSAL_TIMEOUT,
    self.output_proposer.propose_output(...),
).await;
```

That call reaches:

```rust
ProposalSubmitter::propose_output()
    -> self.tx_manager.send(candidate).await
    -> send_tx(candidate, None)
    -> send_event_loop()
    -> prepare_with_initial_caps(..., nonce_override = None, ...)
```

Because `nonce_override` is `None`, `craft_tx_with_caps()` reserves a new nonce through `NonceManager::next_nonce()`:

```rust
let g = self.nonce_manager.next_nonce().await?;
let n = g.nonce();
```

The problem is what happens if the outer timeout drops the future here. `NonceGuard::Drop` does not roll the nonce back; it only logs that the nonce was consumed. Rollback only happens through an explicit `rollback()` call on the signing error path.

The TxManager has an internal cleanup path for `TxManagerError::SendTimeout`, but it is disabled by default. `tx_send_timeout` defaults to `Duration::ZERO`, and the CLI default is `"0s"`. So the internal timeout never fires, and the nonce reset logic only runs if `send_event_loop().await` returns. With the proposer’s outer timeout, the whole future is dropped before that cleanup can run.

## Impact Details

This is a **high-severity liveness issue** for the proposer. Once nonce `n` is leaked, every later proposal transaction is signed with nonce `n+1` or higher. Ethereum will not mine those transactions while nonce `n` is missing, so the proposer can get stuck behind a transaction that was never even published.

The practical result is ugly: L1 dispute game / output proposal submission can stop until someone notices and recovers the service by restarting, resetting nonce state, or manually clearing the gap. In the meantime, legitimate proposals are not created on L1. That means the smart-contract workflow is not just delayed by normal network conditions; it is completely wedged by the proposer’s own nonce state.

There is no direct theft here, and this does not halt the whole L1/L2 network. But for the proposal system, it is a hard availability failure. The trigger is realistic too: a congested L1, degraded RPC, repeated prepare retries, fee-bump churn, or a slow remote signer/HSM can all push the outer 10-minute timeout over the edge. With tx\_send\_timeout disabled by default, a stock proposer has no earlier internal timeout cleanup to catch it first. One timeout at the wrong point can turn into an indefinite submission outage for that proposer instance.

## References

* `crates/utilities/tx-manager/src/manager.rs`
* `crates/proof/proposer/src/pipeline.rs`
* `crates/proof/proposer/src/output_proposer.rs`
* `crates/utilities/tx-manager/src/nonce.rs`
* `crates/utilities/tx-manager/src/config.rs`
* `crates/utilities/tx-manager/src/macros.rs`

## Proof of Concept

Make these changes

```diff
diff --git a/crates/utilities/tx-manager/tests/common/mod.rs b/crates/utilities/tx-manager/tests/common/mod.rs
index af01f7026..704e78449 100644
--- a/crates/utilities/tx-manager/tests/common/mod.rs
+++ b/crates/utilities/tx-manager/tests/common/mod.rs
@@ -4,7 +4,7 @@
 //! binary uses every item.
 #![allow(dead_code, unreachable_pub)]
 
-use std::sync::Arc;
+use std::{future, sync::Arc};
 
 use alloy_consensus::SignableTransaction;
 use alloy_network::{EthereumWallet, TxSigner};
@@ -14,6 +14,7 @@ use alloy_provider::{Provider, RootProvider};
 use alloy_signer_local::PrivateKeySigner;
 use async_trait::async_trait;
 use base_tx_manager::{NoopTxMetrics, SendState, SimpleTxManager, TxCandidate, TxManagerConfig};
+use tokio::sync::Notify;
 
 pub const TEST_RECIPIENT: Address = Address::with_last_byte(0x42);
 pub const SAFE_ABORT_DEPTH: u64 = 3;
@@ -112,3 +113,46 @@ pub async fn setup_with_failing_signer(
     .expect("should create manager with failing signer");
     (manager, anvil)
 }
+
+/// Signer that hangs once signing starts.
+pub struct BlockingSigner {
+    pub address: Address,
+    pub started: Arc<Notify>,
+}
+
+#[async_trait]
+impl TxSigner<Signature> for BlockingSigner {
+    fn address(&self) -> Address {
+        self.address
+    }
+
+    async fn sign_transaction(
+        &self,
+        _tx: &mut dyn SignableTransaction<Signature>,
+    ) -> alloy_signer::Result<Signature> {
+        self.started.notify_one();
+        future::pending().await
+    }
+}
+
+/// Creates a manager whose signer hangs.
+pub async fn setup_with_blocking_signer(
+    config: TxManagerConfig,
+) -> (SimpleTxManager<RootProvider>, Arc<Notify>, alloy_node_bindings::AnvilInstance) {
+    let (provider, _, anvil) = setup_anvil();
+    let started = Arc::new(Notify::new());
+    let wallet = EthereumWallet::from(BlockingSigner {
+        address: anvil.addresses()[0],
+        started: Arc::clone(&started),
+    });
+    let manager = SimpleTxManager::from_wallet(
+        provider,
+        wallet,
+        config,
+        anvil.chain_id(),
+        Arc::new(NoopTxMetrics),
+    )
+    .await
+    .expect("should create manager with blocking signer");
+    (manager, started, anvil)
+}
diff --git a/crates/utilities/tx-manager/tests/send_lifecycle.rs b/crates/utilities/tx-manager/tests/send_lifecycle.rs
index 679946ba1..53beff36e 100644
--- a/crates/utilities/tx-manager/tests/send_lifecycle.rs
+++ b/crates/utilities/tx-manager/tests/send_lifecycle.rs
@@ -16,8 +16,8 @@ use alloy_primitives::B256;
 use alloy_provider::{Provider, RootProvider};
 use base_tx_manager::{SendState, SimpleTxManager, TxManager, TxManagerConfig, TxManagerError};
 use common::{
-    SAFE_ABORT_DEPTH, mine_block, publish_simple_tx, setup_with_config, setup_with_failing_signer,
-    simple_tx_candidate,
+    SAFE_ABORT_DEPTH, mine_block, publish_simple_tx, setup_with_blocking_signer, setup_with_config,
+    setup_with_failing_signer, simple_tx_candidate,
 };
 use rstest::rstest;
 use tokio::sync::mpsc;
@@ -106,6 +106,33 @@ async fn send_resets_nonce_manager_on_pre_publish_failure(#[case] tx_send_timeou
     assert_send_error_resets_nonce(config).await;
 }
 
+/// Dropping `send()` mid-signing leaks the reserved nonce.
+#[tokio::test]
+async fn poc_outer_timeout_during_initial_send_leaks_nonce() {
+    let config = TxManagerConfig { tx_send_timeout: Duration::ZERO, ..fast_send_config() };
+    let (manager, signing_started, _anvil) = setup_with_blocking_signer(config).await;
+    let send_manager = manager.clone();
+
+    let send_task = tokio::spawn(async move {
+        tokio::time::timeout(Duration::from_millis(250), send_manager.send(simple_tx_candidate()))
+            .await
+    });
+
+    tokio::time::timeout(Duration::from_secs(5), signing_started.notified())
+        .await
+        .expect("send should reach the signer after reserving a nonce");
+
+    let result = send_task.await.expect("send task should not panic");
+    assert!(result.is_err(), "outer timeout should cancel the in-flight send");
+
+    let guard = manager.nonce_manager().next_nonce().await.expect("should reserve next nonce");
+    assert_eq!(
+        guard.nonce(),
+        1,
+        "nonce 0 was leaked when the outer timeout dropped send() before TxManager cleanup"
+    );
+}
+
 // ── publish_tx() ──────────────────────────────────────────────────────
```

```bash
cargo test -p base-tx-manager --test send_lifecycle \
  poc_outer_timeout_during_initial_send_leaks_nonce -- --nocapture
```

which outputs

`test poc_outer_timeout_during_initial_send_leaks_nonce ... ok`

The test uses a signer that blocks after signing starts, after the nonce has already been reserved. The outer timeout cancels `send()`, and the next nonce returned by the manager is `1`, showing nonce `0` was leaked without publication. This test demonstrates the TxManager failure mode directly; the production proposer reaches the same `send()` path through `ProposalSubmitter::propose_output()` under the outer timeout in `pipeline.rs`.
