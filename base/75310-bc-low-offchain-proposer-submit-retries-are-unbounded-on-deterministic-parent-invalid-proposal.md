# 75310 bc low offchain proposer submit retries are unbounded on deterministic parent invalid proposal reverts

**Submitted on Apr 28th 2026 at 13:05:56 UTC by @pks271 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75310
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk
  * Unbounded gas consumption in any in-scope contract function callable by external parties

## Description

## Bug Description

commit: `e3467a2048881213b56739a54a876efb9c6ea103` (`v0.8.0-rc.28`)

The off-chain proposer pipeline recovers the latest valid proposal chain, caches the recovered `parent_address`, generates the next output proof, and then submits the proof to L1 through `DisputeGameFactory.createWithInitData()`.

This flow assumes the recovered parent remains valid until the L1 proposal transaction is accepted. If the parent game becomes invalid after recovery/proving but before the proposal transaction lands, `AggregateVerifier.initializeWithInitData()` deterministically reverts with `InvalidParentGame`.

The parent can be successfully challenged and resolve `CHALLENGER_WINS`, or it can be blacklisted/retired by the registry, while the proposer still holds a proved child output using the stale parent.

After this deterministic submit failure, the proposer treats the failure as retryable, reinserts the same proof back into `state.proved`, clears `state.submitting`, and tries the same submission again on later ticks. The configured `max_retries` only bounds proof-generation failures in `handle_proof_result`, it is not applied to submit failures.

call trace:

`crates/proof/proposer/src/pipeline.rs::run -> try_recover_and_plan -> dispatch_proofs -> handle_proof_result stores proof in state.proved -> try_submit uses cached parent_address -> validate_and_submit -> OutputProposer.propose_output -> DisputeGameFactory.createWithInitData() -> AggregateVerifier.initializeWithInitData() revert InvalidParentGame -> SubmitOutcome::Failed -> state.proved.insert(target_block, proof) -> next tick repeats`

code:

```rust
// crates/proof/proposer/src/pipeline.rs
fn try_submit(&self, state: &mut PipelineState) {
    if state.submitting.is_some() || !state.submit_tasks.is_empty() {
        return;
    }

    let recovered = match &state.cached_recovery {
        Some(cached) => cached.state,
        _ => return,
    };

    let next_to_submit =
        match recovered.l2_block_number.checked_add(self.config.driver.block_interval) {
            Some(n) => n,
            None => return,
        };

    // @audit - remove the proof from the proved map
    let proof_result = match state.proved.remove(&next_to_submit) {
        Some(r) => r,
        None => return,
    };

    let parent_address = recovered.parent_address;
    state.submitting = Some(next_to_submit);

    state.submit_tasks.spawn(async move {
        let result =
            pipeline.validate_and_submit(&proof_result, next_to_submit, parent_address).await;
        match result {
            Ok(()) => SubmitOutcome::Success { target_block: next_to_submit },
            Err(SubmitAction::RootMismatch) => {
                SubmitOutcome::RootMismatch { target_block: next_to_submit }
            }
            Err(SubmitAction::Failed(e)) => {
                SubmitOutcome::Failed {
                    target_block: next_to_submit,
                    proof: proof_result,
                    error: e,
                }
            }
        }
    });
}
```

```rust
// crates/proof/proposer/src/pipeline.rs
SubmitOutcome::Failed { target_block, proof, error } => {
    Metrics::errors_total(error.metric_label()).increment(1);
    warn!(
        error = %error,
        target_block,
        "Submission failed, will retry"
    );
    // @audit - insert the proof back into the proved map
    state.proved.insert(target_block, proof);
    state.submitting = None;
    state.record_gauges();
    false
}
```

```solidity
// src/dispute/DisputeGameFactory.sol
function createWithInitData(
    GameType _gameType,
    Claim _rootClaim,
    bytes calldata _extraData,
    bytes calldata _initData
)
    external
    payable
    returns (IDisputeGame proxy_)
{
    proxy_ = _createGameImpl(_gameType, _rootClaim, _extraData);
    proxy_.initializeWithInitData{ value: msg.value }(_initData);
    _finalizeGameCreation(_gameType, _rootClaim, _extraData, proxy_);
}
```

```solidity
// src/multiproof/AggregateVerifier.sol
if (parentAddress() != address(ANCHOR_STATE_REGISTRY)) {
    IDisputeGame parentGame = IDisputeGame(parentAddress());

    // Parent game must be registered, respected, not blacklisted, not retired, and not challenged.
    if (!_isValidGame(parentGame)) revert InvalidParentGame();

    startingOutputRoot = Proposal({
        l2SequenceNumber: parentGame.l2SequenceNumber(), root: Hash.wrap(parentGame.rootClaim().raw())
    });
}
```

## Impact

A parent-invalidating actor in the normal dispute flow can make the official proposer repeatedly submit the same already-doomed proposal transaction against the same stale parent. Each retry consumes L1 gas and prevents that proposer process from progressing to a fresh recovery/re-proof path for the next valid parent, cause gas burn and delay dispute resolution.

## Recommendation

Add a bounded submit retry counter separate from proof retries, classify deterministic terminal reverts such as `InvalidParentGame` as drop/reset/recover conditions instead of requeueing the same proof forever, and revalidate the cached parent immediately before submission.

## Proof of Concept

steps:

1. A parent game `P` is valid when the proposer performs recovery walk.
2. The proposer proves the child output using cached `parent_address = P`.
3. Before the proposal transaction is included, `P` becomes invalid through a normal protocol action, for example a valid challenge resolves `P` as `CHALLENGER_WINS`, or the registry blacklists/retires `P`.
4. The proposer submits the child proposal through `DisputeGameFactory.createWithInitData()`.
5. `AggregateVerifier.initializeWithInitData()` reverts with `InvalidParentGame`.
6. The proposer requeues the same proof and retries the same stale-parent submission on later ticks.

patch:

```diff
diff --git a/crates/proof/proposer/src/pipeline.rs b/crates/proof/proposer/src/pipeline.rs
index 013dceb3..a1e0b693 100644
--- a/crates/proof/proposer/src/pipeline.rs
+++ b/crates/proof/proposer/src/pipeline.rs
@@ -1302,7 +1302,14 @@ enum SubmitOutcome {
 
 #[cfg(test)]
 mod tests {
-    use std::{collections::HashMap, sync::Arc, time::Duration};
+    use std::{
+        collections::HashMap,
+        sync::{
+            Arc,
+            atomic::{AtomicUsize, Ordering},
+        },
+        time::Duration,
+    };
 
     use alloy_primitives::{Address, B256};
     use base_proof_primitives::{ProofResult, Proposal, ProverClient};
@@ -1310,6 +1317,7 @@ mod tests {
     use tokio_util::sync::CancellationToken;
 
     use super::*;
+    use crate::output_proposer::OutputProposer;
     use crate::test_utils::{
         MockAggregateVerifier, MockAnchorStateRegistry, MockDisputeGameFactory, MockL1, MockL2,
         MockOutputProposer, MockProver, MockRollupClient, test_anchor_root, test_proposal,
@@ -1564,6 +1572,82 @@ mod tests {
         assert!(result.is_ok());
     }
 
+    #[tokio::test(flavor = "current_thread", start_paused = true)]
+    async fn test_run_loop_retries_invalid_parent_reverts_beyond_max_retries() {
+        let cancel = CancellationToken::new();
+        let submit_attempts = Arc::new(AtomicUsize::new(0));
+        let unexpected_parent_calls = Arc::new(AtomicUsize::new(0));
+        let invalid_parent = proxy_addr(0);
+        let output_proposer: Arc<dyn OutputProposer> = Arc::new(InvalidParentOnlyOutputProposer {
+            invalid_parent,
+            attempts: Arc::clone(&submit_attempts),
+            unexpected_parent_calls: Arc::clone(&unexpected_parent_calls),
+        });
+        let (factory, output_roots) = game_chain_full(
+            1,
+            TEST_ANCHOR_BLOCK,
+            SUBMIT_BLOCK_INTERVAL,
+            SUBMIT_INTERMEDIATE_INTERVAL,
+        );
+
+        let l1 = Arc::new(MockL1 { latest_block_number: TEST_L1_BLOCK_NUMBER });
+        let l2 = Arc::new(MockL2 { block_not_found: true, canonical_hash: None });
+        let prover: Arc<dyn ProverClient> = Arc::new(MockProver {
+            delay: Duration::from_millis(1),
+            block_interval: SUBMIT_BLOCK_INTERVAL,
+        });
+        let rollup = Arc::new(MockRollupClient {
+            sync_status: test_sync_status(SUBMIT_BLOCK_INTERVAL * 2, B256::ZERO),
+            output_roots,
+            max_safe_block: None,
+        });
+        let anchor_registry =
+            Arc::new(MockAnchorStateRegistry { anchor_root: test_anchor_root(TEST_ANCHOR_BLOCK) });
+        let factory = Arc::new(factory);
+
+        let pipeline = ProvingPipeline::new(
+            PipelineConfig {
+                max_parallel_proofs: 1,
+                max_retries: 1,
+                recovery_scan_concurrency: 8,
+                tee_prover_registry_address: None,
+                driver: DriverConfig {
+                    poll_interval: Duration::from_millis(100),
+                    game_type: TEST_GAME_TYPE,
+                    block_interval: SUBMIT_BLOCK_INTERVAL,
+                    intermediate_block_interval: SUBMIT_INTERMEDIATE_INTERVAL,
+                    ..Default::default()
+                },
+            },
+            prover,
+            l1,
+            l2,
+            rollup,
+            anchor_registry,
+            factory,
+            Arc::new(MockAggregateVerifier::default()),
+            output_proposer,
+            cancel.clone(),
+        );
+
+        let handle = tokio::spawn(async move { pipeline.run().await });
+
+        tokio::time::sleep(Duration::from_secs(2)).await;
+        cancel.cancel();
+
+        let result = handle.await.expect("task should not panic");
+        assert!(result.is_ok(), "run() should stop cleanly after cancellation");
+        assert!(
+            submit_attempts.load(Ordering::SeqCst) > 1,
+            "submit attempts should exceed configured max_retries"
+        );
+        assert_eq!(
+            unexpected_parent_calls.load(Ordering::SeqCst),
+            0,
+            "all repeated submit attempts should target the same invalid parent"
+        );
+    }
+
     // ---- Recovery: empty factory ----
 
     #[tokio::test(flavor = "current_thread", start_paused = true)]
@@ -2116,6 +2200,30 @@ mod tests {
         ProofResult::Tee { aggregate_proposal: aggregate, proposals }
     }
 
+    #[derive(Debug)]
+    struct InvalidParentOnlyOutputProposer {
+        invalid_parent: Address,
+        attempts: Arc<AtomicUsize>,
+        unexpected_parent_calls: Arc<AtomicUsize>,
+    }
+
+    #[async_trait::async_trait]
+    impl OutputProposer for InvalidParentOnlyOutputProposer {
+        async fn propose_output(
+            &self,
+            _proposal: &Proposal,
+            parent_address: Address,
+            _intermediate_roots: &[B256],
+        ) -> Result<(), ProposerError> {
+            self.attempts.fetch_add(1, Ordering::SeqCst);
+            if parent_address == self.invalid_parent {
+                return Err(ProposerError::TxReverted("InvalidParentGame".into()));
+            }
+            self.unexpected_parent_calls.fetch_add(1, Ordering::SeqCst);
+            Ok(())
+        }
+    }
+
     #[tokio::test(flavor = "current_thread", start_paused = true)]
     async fn test_validate_and_submit_intermediate_roots_match() {
         // MockRollupClient returns B256::repeat_byte(n) for blocks without
```

Run:

```bash
CARGO_ENCODED_RUSTFLAGS='-Clink-arg=-fuse-ld=lld' cargo test -p base-proposer --lib test_run_loop_retries_invalid_parent_reverts_beyond_max_retries -- --nocapture
```

Result:

```
running 1 test
test pipeline::tests::test_run_loop_retries_invalid_parent_reverts_beyond_max_retries ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 74 filtered out; finished in 0.00s
```
