# 75312 bc low offchain challenger nullify retries are unbounded when dual proof game is externally challenged after proof readiness

Submitted on Apr 28th 2026 at 13:20:14 UTC by @pks271 for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75312
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk
  * Unbounded gas consumption in any in-scope contract function callable by external parties

## Description

### Bug Description

commit hash: `e3467a2048881213b56739a54a876efb9c6ea103` (`v0.8.0-rc.28`)

In the challenger dual-proof path (`InvalidDualProposal`), the driver can build a `nullify()` proof from an unchallenged game state (`tee_prover != 0`, `zk_prover != 0`, `countered_index == 0`) and move the entry to `ReadyToSubmit`.

If the game is externally challenged before the submit happens, the on-chain `nullify()` rules change immediately:

* `counteredByIntermediateRootIndexPlusOne > 0` requires nullifying the challenged index exactly;
* `intermediateRootToProve` must equal the challenged root;
* proof type must be `ZK`.

The current off-chain submit path does not re-validate `countered_index` compatibility before sending the pending `nullify()` call. It only checks `status`, `tee_prover`, and `zk_prover`. As a result, stale pre-challenge `(invalid_index, expected_root)` is repeatedly submitted and deterministically reverts.

On revert, the entry stays in `ReadyToSubmit` and is retried every tick. `MAX_PROOF_RETRIES` is not applied because `retry_count` only tracks proving retries, not submit retries.

### Call trace

`crates/proof/challenge/src/driver.rs::step -> process_invalid_proposal (InvalidDualProposal) -> initiate_proof(intent=Nullify) -> pending ReadyToSubmit -> external challenge sets countered index -> poll_or_submit submits stale nullify -> AggregateVerifier.nullify() challenged branch rejects -> Err branch keeps ReadyToSubmit -> next tick repeats`

### Code

```rust
// crates/proof/challenge/src/driver.rs
if self.pending_proofs.contains_key(&game_address) {
    debug!(game = %game_address, "skipping game with pending proof session");
    return Ok(());
}

let (status, tee_prover, zk_prover) = tokio::try_join!(
    self.verifier_client.status(game_address),
    self.verifier_client.tee_prover(game_address),
    self.verifier_client.zk_prover(game_address),
)?;

// no countered_index re-validation before submit
let result = self
    .submitter
    .submit_dispute(game_address, proof_bytes, invalid_index, expected_root, intent)
    .await;

match result {
    Ok(_) => { self.pending_proofs.remove(&game_address); }
    Err(e) => {
        warn!(error = %e, game = %game_address, "dispute tx failed, will retry next tick");
        // Leave entry as ReadyToSubmit for retry.
    }
}
```

```solidity
// base-contracts/src/multiproof/AggregateVerifier.sol
function nullify(
    bytes calldata proofBytes,
    uint256 intermediateRootIndex,
    bytes32 intermediateRootToProve
) external {
    ...
    if (counteredByIntermediateRootIndexPlusOne > 0) {
        if (intermediateRootIndex != counteredByIntermediateRootIndexPlusOne - 1) {
            revert InvalidIntermediateRootIndex();
        }
        if (intermediateRootToProve != intermediateOutputRoot(intermediateRootIndex)) {
            revert IntermediateRootMismatch(intermediateRootToProve, intermediateOutputRoot(intermediateRootIndex));
        }
        if (proofType != ProofType.ZK) revert InvalidProofType();
    } else {
        _checkIntermediateRoot(intermediateRootIndex, intermediateRootToProve);
    }
    ...
}
```

## Impact

An external actor can race the challenger by challenging a dual-proof game after proof readiness but before submit. This forces deterministic submit reverts on stale `nullify()` parameters. Because submit retries are unbounded in `ReadyToSubmit`, the challenger can enter a persistent gas/liveness degradation loop for the same game.

## Recommendation

* Add submit-side stale-context checks before `nullify()`:
  * fetch `countered_index`;
  * if challenged (`countered_index > 0`), ensure pending `invalid_index` and `expected_root` still match challenged index/root semantics;
  * otherwise drop/rebuild pending entry from fresh scan classification.
* Add `MAX_SUBMIT_RETRIES` (separate from proof retries) and stop retrying deterministic terminal reverts.

## Proof of Concept

steps:

1. Game is dual-proof and unchallenged (`tee_prover != 0`, `zk_prover != 0`, `countered_index == 0`).
2. Challenger validates roots and creates a `Nullify` pending proof with pre-challenge `(invalid_index, expected_root)`.
3. Before submit, external actor calls `challenge()`, moving the game into challenged state (`countered_index > 0`).
4. Challenger submits stale `nullify()` parameters; on-chain call reverts deterministically.
5. Driver keeps entry `ReadyToSubmit` and retries on every tick.

### Patch

```diff
diff --git a/crates/proof/challenge/tests/driver.rs b/crates/proof/challenge/tests/driver.rs
index 341b0d6e5..0a9b6e2e4 100644
--- a/crates/proof/challenge/tests/driver.rs
+++ b/crates/proof/challenge/tests/driver.rs
@@ -1164,6 +1164,56 @@ async fn test_step_dual_proof_tee_fails_falls_back_to_zk_nullify() {
     );
 }

+#[tokio::test]
+async fn test_dual_proof_ready_to_submit_retry_remains_unbounded_after_external_challenge() {
+    base_cli_utils::init_test_tracing();
+
+    // AUDIT: start from a dual-proof, unchallenged game where the challenger
+    // enters the Nullify path and builds a pending proof from pre-challenge state.
+    let (l2, factory, root_15, _root_20) = base_game_mocks();
+    let initial_state = MockGameState {
+        tee_prover: DEFAULT_TEE_PROVER,
+        zk_prover: ZK_PROVER_ADDR,
+        intermediate_output_roots: vec![root_15, BOGUS_ROOT],
+        countered_index: 0,
+        ..game_state(20)
+    };
+    let verifier = single_game_verifier(initial_state.clone());
+
+    let zk = succeeded_zk_prover("dual-race-retry", vec![0xDE, 0xAD]);
+
+    // AUDIT: model deterministic nullify() submit reverts once the game is
+    // externally challenged before the pending submission lands.
+    let total_submit_failures =
+        Driver::<MockL2Provider, MockZkProofProvider, MockTxManager>::MAX_PROOF_RETRIES as usize
+            + 2;
+    let tx_manager = MockTxManager::with_responses(
+        (0..total_submit_failures)
+            .map(|_| Ok(receipt_with_status(false, DEFAULT_TX_HASH)))
+            .collect(),
+    );
+
+    let mut driver = test_driver(factory, Arc::clone(&verifier), l2, zk, tx_manager);
+
+    // AUDIT: dual-proof candidate should initiate ZK Nullify proof flow.
+    driver.step().await.unwrap();
+    let entry = driver.pending_proofs.get(&addr(0)).expect("proof should be pending after initiation");
+    assert!(matches!(entry.phase, ProofPhase::AwaitingProof { .. }));
+    assert_eq!(entry.intent, DisputeIntent::Nullify);
+
+    // AUDIT: external challenge flips countered_index semantics before submit.
+    let mut challenged_state = initial_state;
+    challenged_state.countered_index = 2; // 1-based => challenged index 1
+    verifier.update_game(addr(0), challenged_state);
+
+    for _ in 0..total_submit_failures {
+        driver.step().await.unwrap();
+        let entry = driver.pending_proofs.get(&addr(0)).expect("entry should be preserved");
+        assert!(entry.is_ready(), "phase should remain ReadyToSubmit after revert");
+        assert_eq!(entry.retry_count, 0, "submit retries are not bounded by proof retry counter");
+    }
+}
+
 // ──────────────────────────────────────────────────────────────────────────
 // Bond lifecycle integration tests
```

Run:

```bash
CARGO_ENCODED_RUSTFLAGS='-Clink-arg=-fuse-ld=lld' cargo test -p base-challenger --test driver test_dual_proof_ready_to_submit_retry_remains_unbounded_after_external_challenge -- --nocapture
```

Result:

```
running 1 test
2026-04-28T12:34:49.931248Z  INFO base_challenger::scanner: scan complete games_found=1 scan_head=0 games_scanned=1
2026-04-28T12:34:49.931270Z  INFO base_challenger::validator: validating intermediate output roots game=0x0000000000000000000000000000000000000000 starting_block=10 end_block=20 interval=5 intermediate_count=2
2026-04-28T12:34:49.931544Z  WARN base_challenger::validator: invalid output root detected game=0x0000000000000000000000000000000000000000 block=20 index=1 expected=0x401da1446febeae61109677bc2f52e9e7bbb8313a6f3507529006ef0b49c2f60 claimed=0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
2026-04-28T12:34:49.931567Z  INFO base_challenger::driver: invalid intermediate root detected, requesting proof game=0x0000000000000000000000000000000000000000 invalid_index=1 expected_root=0x401da1446febeae61109677bc2f52e9e7bbb8313a6f3507529006ef0b49c2f60 intent=Nullify
2026-04-28T12:34:49.931623Z  INFO base_challenger::driver: proof job initiated game=0x0000000000000000000000000000000000000000 session_id=dual-race-retry
2026-04-28T12:34:49.931651Z  INFO base_challenger::driver: proof ready, submitting dispute transaction game=0x0000000000000000000000000000000000000000 proof_len=3 action="nullify"
2026-04-28T12:34:49.931684Z  INFO base_challenger::submitter: submitting dispute transaction game=0x0000000000000000000000000000000000000000 action="nullify" intermediate_root_index=1 intermediate_root_to_prove=0x401da1446febeae61109677bc2f52e9e7bbb8313a6f3507529006ef0b49c2f60 calldata_len=164
2026-04-28T12:34:49.931706Z  WARN base_challenger::driver: dispute tx failed, will retry next tick error=transaction reverted: 0xdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd game=0x0000000000000000000000000000000000000000
...
```
