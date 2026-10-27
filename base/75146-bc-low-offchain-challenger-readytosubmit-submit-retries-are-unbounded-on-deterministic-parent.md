# 75146 bc low offchain challenger readytosubmit submit retries are unbounded on deterministic parent invalid reverts

**Submitted on Apr 27th 2026 at 13:57:45 UTC by @pks271 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75146
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.24>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk
  * Unbounded gas consumption in any in-scope contract function callable by external parties

## Bug Description

In the challenger pipeline, each dispute game can reference a `parent` game and a `child` game. The parent-child relation is used to preserve dispute lineage and ensure a child cannot progress when its parent is already invalidated.

The off-chain challenger driver currently checks only the **child** game liveness (`status`, `tee_prover`, `zk_prover`) before submitting a `challenge()`, but does not pre-check parent validity. If the child remains `IN_PROGRESS` while the parent has already become invalid (`CHALLENGER_WINS`, blacklisted, or retired), on-chain `challenge()` deterministically reverts with `InvalidParentGame`.

After such revert, the driver keeps the entry in `ReadyToSubmit` and retries on every tick, but this path is not bounded by `MAX_PROOF_RETRIES` because `retry_count` only increments on proof job failures, not on submit failures.

`base/base` commit: `819ea306db40792a50626243034a62e3ca015ba6` (`v0.8.0-rc.24`)

call trace:

`crates/proof/challenge/src/driver.rs::step -> poll_pending_proofs -> poll_or_submit -> submit_dispute -> (L1) AggregateVerifier.challenge() revert InvalidParentGame -> Err branch leaves entry ReadyToSubmit -> next tick repeats`

code:

```rust
// crates/proof/challenge/src/driver.rs
pub const MAX_PROOF_RETRIES: u32 = 3;

let (status, tee_prover, zk_prover) = tokio::try_join!(
    self.verifier_client.status(game_address),
    self.verifier_client.tee_prover(game_address),
    self.verifier_client.zk_prover(game_address),
)?;

let already_resolved = match intent {
    DisputeIntent::Challenge => zk_prover != Address::ZERO || tee_prover == Address::ZERO,
    DisputeIntent::Nullify => { /* ... */ }
};

match result {
    Ok(_) => { self.pending_proofs.remove(&game_address); }
    Err(e) => {
        ...
        warn!(error = %e, game = %game_address, "dispute tx failed, will retry next tick");
        // Leave entry as ReadyToSubmit for retry.
    }
}
```

```rust
// crates/proof/challenge/src/pending.rs
ProofJobStatus::Failed => {
    pending.retry_count += 1;
    pending.phase = ProofPhase::NeedsRetry;
    ProofUpdate::NeedsRetry
}
```

```solidity
// base-contracts/src/multiproof/AggregateVerifier.sol
function challenge(...) external {
    if (_getParentGameStatus() == GameStatus.CHALLENGER_WINS) revert InvalidParentGame();
    ...
}

function _getParentGameStatus() internal view returns (GameStatus) {
    if (ANCHOR_STATE_REGISTRY.isGameBlacklisted(parentGame) || ANCHOR_STATE_REGISTRY.isGameRetired(parentGame)) {
        return GameStatus.CHALLENGER_WINS;
    }
    return parentGame.status();
}
```

## Impact

An external actor who can create this parent/child state progression can force the challenger to repeatedly send reverted L1 `challenge()` transactions for the same game cause gas burn and delay dispute resolution.

## Recommendation

Add a dedicated bounded submit retry mechanism (`MAX_SUBMIT_RETRIES` + per-entry `submit_retry_count`) for `ReadyToSubmit`, treat deterministic terminal reverts (for example `InvalidParentGame`) as drop conditions, and add a pre-submit parent-status validity check for `DisputeIntent::Challenge` consistent with on-chain parent validity rules.

## Proof of Concept

steps:

1. Create or wait for a child game that remains scanner-actionable (`IN_PROGRESS`, `tee_prover != 0`, `zk_prover == 0`).
2. Make the parent game invalid before the child submission attempt (for example parent resolves `CHALLENGER_WINS`, or parent becomes blacklisted/retired).
3. Let challenger reach `ReadyToSubmit`; each `challenge()` attempt reverts deterministically with `InvalidParentGame`.
4. Observe the same game remains in retry loop beyond `MAX_PROOF_RETRIES` because submit failures do not increment `retry_count`.

patch:

```bash
diff --git a/crates/proof/challenge/tests/driver.rs b/crates/proof/challenge/tests/driver.rs
index b0dd37db..210d7ee8 100644
--- a/crates/proof/challenge/tests/driver.rs
+++ b/crates/proof/challenge/tests/driver.rs
@@ -208,6 +208,32 @@ fn invalid_game_mocks()
     (l2, factory, verifier)
 }
 
+/// mocks for a child game that is invalid and challengeable by scanner
+/// rules, while its parent game has already resolved as `CHALLENGER_WINS`.
+///
+/// models a deterministic on-chain `challenge()` revert path
+/// (`InvalidParentGame`) that is not reflected by the driver's pre-submit liveness checks
+/// (`status/teeProver/zkProver` on the child only).
+fn invalid_game_with_invalid_parent_mocks()
+-> (Arc<MockL2Provider>, Arc<MockDisputeGameFactory>, Arc<MockAggregateVerifier>) {
+    let (l2, factory, root_15, _root_20) = base_game_mocks();
+
+    let parent_addr = addr(999);
+
+    let mut child = game_state(20);
+    child.tee_prover = DEFAULT_TEE_PROVER;
+    child.intermediate_output_roots = vec![root_15, BOGUS_ROOT];
+    child.game_info.parent_address = parent_addr;
+
+    // parent is already invalidated/challenged.
+    let parent = mock_state(1, Address::ZERO, 15);
+
+    let verifier =
+        Arc::new(MockAggregateVerifier::new(HashMap::from([(addr(0), child), (parent_addr, parent)])));
+
+    (l2, factory, verifier)
+}
+
 /// Builds a driver with a single pending `ReadyToSubmit` proof at `addr(0)`
 /// whose verifier reports the given `game_state`.
 fn driver_with_ready_proof(
@@ -608,6 +634,47 @@ async fn test_step_proof_exceeds_max_retries() {
     );
 }
 
+#[tokio::test]
+async fn test_ready_to_submit_retry_remains_unbounded_when_parent_is_invalid() {
+    base_cli_utils::init_test_tracing();
+
+    // child game remains IN_PROGRESS and scanner-actionable, but parent has
+    // already resolved as CHALLENGER_WINS.
+    let (l2, factory, verifier) = invalid_game_with_invalid_parent_mocks();
+
+    let zk = succeeded_zk_prover("parent-invalid-retry", vec![0xDE, 0xAD]);
+
+    // deterministic on-chain reverts (e.g. InvalidParentGame in challenge()) across ticks
+    let total_submit_failures =
+        Driver::<MockL2Provider, MockZkProofProvider, MockTxManager>::MAX_PROOF_RETRIES as usize
+            + 2;
+    let tx_manager = MockTxManager::with_responses(
+        (0..total_submit_failures)
+            .map(|_| Ok(receipt_with_status(false, DEFAULT_TX_HASH)))
+            .collect(),
+    );
+
+    let mut driver = test_driver(factory, verifier, l2, zk, tx_manager);
+
+    // initiate proof session.
+    driver.step().await.unwrap();
+    assert!(
+        driver.pending_proofs.contains_key(&addr(0)),
+        "proof should be pending after initiation"
+    );
+
+    // repeatedly attempt submission and revert.
+    for _ in 0..total_submit_failures {
+        driver.step().await.unwrap();
+        let entry = driver.pending_proofs.get(&addr(0)).expect("entry should be preserved");
+        assert!(entry.is_ready(), "phase should remain ReadyToSubmit after revert");
+    }
+
+    // retries are independent from proving retries.
+    let entry = driver.pending_proofs.get(&addr(0)).expect("entry should still exist");
+    assert_eq!(entry.retry_count, 0);
+}
+
 // ── TEE-first proof sourcing tests ─────────────────────────────────────────
 
 #[tokio::test]
```

run:

```bash
cargo test -p base-challenger --test driver test_ready_to_submit_retry_remains_unbounded_when_parent_is_invalid -- --nocapture
```

result:

```
unning 1 test
2026-04-27T13:50:00.452242Z  INFO base_challenger::scanner: scan complete games_found=1 scan_head=0 games_scanned=1
2026-04-27T13:50:00.452269Z  INFO base_challenger::validator: validating intermediate output roots game=0x0000000000000000000000000000000000000000 starting_block=10 end_block=20 interval=5 intermediate_count=2
2026-04-27T13:50:00.452511Z  WARN base_challenger::validator: invalid output root detected game=0x0000000000000000000000000000000000000000 block=20 index=1 expected=0x401da1446febeae61109677bc2f52e9e7bbb8313a6f3507529006ef0b49c2f60 claimed=0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
2026-04-27T13:50:00.452535Z  INFO base_challenger::driver: invalid intermediate root detected, requesting proof game=0x0000000000000000000000000000000000000000 invalid_index=1 expected_root=0x401da1446febeae61109677bc2f52e9e7bbb8313a6f3507529006ef0b49c2f60 intent=Challenge
2026-04-27T13:50:00.452569Z  INFO base_challenger::driver: proof job initiated game=0x0000000000000000000000000000000000000000 session_id=parent-invalid-retry
2026-04-27T13:50:00.452601Z  INFO base_challenger::driver: proof ready, submitting dispute transaction game=0x0000000000000000000000000000000000000000 proof_len=3 action="challenge"
2026-04-27T13:50:00.452636Z  INFO base_challenger::submitter: submitting dispute transaction game=0x0000000000000000000000000000000000000000 action="challenge" intermediate_root_index=1 intermediate_root_to_prove=0x401da1446febeae61109677bc2f52e9e7bbb8313a6f3507529006ef0b49c2f60 calldata_len=164
2026-04-27T13:50:00.452658Z  WARN base_challenger::driver: dispute tx failed, will retry next tick error=transaction reverted: 0xdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd game=0x0000000000000000000000000000000000000000
2026-04-27T13:50:00.452687Z  INFO base_challenger::scanner: scan complete games_found=1 scan_head=0 games_scanned=1
2026-04-27T13:50:00.452695Z  INFO base_challenger::driver: proof ready, submitting dispute transaction game=0x0000000000000000000000000000000000000000 proof_len=3 action="challenge"
2026-04-27T13:50:00.452719Z  INFO base_challenger::submitter: submitting dispute transaction game=0x0000000000000000000000000000000000000000 action="challenge" intermediate_root_index=1 intermediate_root_to_prove=0x401da1446febeae61109677bc2f52e9e7bbb8313a6f3507529006ef0b49c2f60 calldata_len=164
2026-04-27T13:50:00.452739Z  WARN base_challenger::driver: dispute tx failed, will retry next tick error=transaction reverted: 0xdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd game=0x0000000000000000000000000000000000000000
2026-04-27T13:50:00.452763Z  INFO base_challenger::scanner: scan complete games_found=1 scan_head=0 games_scanned=1
2026-04-27T13:50:00.452770Z  INFO base_challenger::driver: proof ready, submitting dispute transaction game=0x0000000000000000000000000000000000000000 proof_len=3 action="challenge"
2026-04-27T13:50:00.452790Z  INFO base_challenger::submitter: submitting dispute transaction game=0x0000000000000000000000000000000000000000 action="challenge" intermediate_root_index=1 intermediate_root_to_prove=0x401da1446febeae61109677bc2f52e9e7bbb8313a6f3507529006ef0b49c2f60 calldata_len=164
2026-04-27T13:50:00.452810Z  WARN base_challenger::driver: dispute tx failed, will retry next tick error=transaction reverted: 0xdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd game=0x0000000000000000000000000000000000000000
2026-04-27T13:50:00.452833Z  INFO base_challenger::scanner: scan complete games_found=1 scan_head=0 games_scanned=1
2026-04-27T13:50:00.452839Z  INFO base_challenger::driver: proof ready, submitting dispute transaction game=0x0000000000000000000000000000000000000000 proof_len=3 action="challenge"
2026-04-27T13:50:00.452858Z  INFO base_challenger::submitter: submitting dispute transaction game=0x0000000000000000000000000000000000000000 action="challenge" intermediate_root_index=1 intermediate_root_to_prove=0x401da1446febeae61109677bc2f52e9e7bbb8313a6f3507529006ef0b49c2f60 calldata_len=164
2026-04-27T13:50:00.452878Z  WARN base_challenger::driver: dispute tx failed, will retry next tick error=transaction reverted: 0xdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd game=0x0000000000000000000000000000000000000000
2026-04-27T13:50:00.452901Z  INFO base_challenger::scanner: scan complete games_found=1 scan_head=0 games_scanned=1
2026-04-27T13:50:00.452907Z  INFO base_challenger::driver: proof ready, submitting dispute transaction game=0x0000000000000000000000000000000000000000 proof_len=3 action="challenge"
2026-04-27T13:50:00.452926Z  INFO base_challenger::submitter: submitting dispute transaction game=0x0000000000000000000000000000000000000000 action="challenge" intermediate_root_index=1 intermediate_root_to_prove=0x401da1446febeae61109677bc2f52e9e7bbb8313a6f3507529006ef0b49c2f60 calldata_len=164
2026-04-27T13:50:00.452946Z  WARN base_challenger::driver: dispute tx failed, will retry next tick error=transaction reverted: 0xdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd game=0x0000000000000000000000000000000000000000
2026-04-27T13:50:00.452969Z  INFO base_challenger::scanner: scan complete games_found=1 scan_head=0 games_scanned=1
test test_ready_to_submit_retry_remains_unbounded_when_parent_is_invalid ... ok
```

The test demonstrates that submit retries continue even after more than `MAX_PROOF_RETRIES`, with the pending proof still in `ReadyToSubmit` and `retry_count == 0`.
