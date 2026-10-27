# 75294 bc low the proposer retries an invalid cached parent instead of refreshing recovery state

**Submitted on Apr 28th 2026 at 11:14:23 UTC by @Jolyon for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75294
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

### Brief/Intro

The proposer in `base/base` reconstructs its next parent by walking UUID-linked dispute games from the anchor, but it does not verify that the recovered games are still valid parent candidates onchain. If a previously cached chain later becomes invalid because an ancestor resolves `CHALLENGER_WINS`, is blacklisted, or is retired, the proposer can keep retrying the same stale parent after submission failures instead of refreshing recovery state. This can delay or stall proposal progression until operator intervention or a later cache reset.

### Vulnerability Details

The issue is caused by two coupled assumptions in the proposer’s recovery and retry flow.

First, recovery treats UUID reachability as sufficient for parent selection. In `base/base`, `recover_latest_state()` reads the anchor root and `game_count`, then calls `forward_walk()` to reconstruct the latest tip by repeatedly querying `DisputeGameFactory.games(...)` with canonical roots and the previous parent address. This logic caches the latest UUID-linked descendant as the next parent candidate, but it does not verify whether that recovered game is still a valid parent candidate onchain at the time of recovery.

Relevant code:

* Recovery caches the latest UUID-linked descendant without checking current game validity:\
  `base/base/crates/proof/proposer/src/pipeline.rs`\
  [pipeline.rs#L713-L777](https://github.com/base/base/blob/v0.8.0-rc.28/crates/proof/proposer/src/pipeline.rs#L713-L777)
* Forward walk relies only on canonical roots and UUID lookups:\
  `base/base/crates/proof/proposer/src/pipeline.rs`\
  [pipeline.rs#L805-L895](https://github.com/base/base/blob/v0.8.0-rc.28/crates/proof/proposer/src/pipeline.rs#L805-L895)

Second, retry logic reuses cached recovery state after failure. In `try_submit()`, the proposer takes the cached `parent_address` and uses it directly for the next submission attempt. If submission fails in the ordinary `SubmitOutcome::Failed` path, `handle_submit_result()` requeues the same proof but does not invalidate or refresh `state.cached_recovery`. The next retry therefore uses the same stale parent again.

Relevant code:

* Cached `parent_address` is used directly for submission:\
  `base/base/crates/proof/proposer/src/pipeline.rs`\
  [pipeline.rs#L445-L503](https://github.com/base/base/blob/v0.8.0-rc.28/crates/proof/proposer/src/pipeline.rs#L445-L503)
* Failed submissions requeue without refreshing recovery state:\
  `base/base/crates/proof/proposer/src/pipeline.rs`\
  [pipeline.rs#L579-L589](https://github.com/base/base/blob/v0.8.0-rc.28/crates/proof/proposer/src/pipeline.rs#L579-L589)

This becomes unsafe because the supplied parent must still be valid onchain. In `base/contracts`, `AggregateVerifier.initialize()` rejects a non-anchor parent with `InvalidParentGame()` if `_isValidGame(parentGame)` is false. `_isValidGame()` requires the parent game to still be registered, respected, not blacklisted, not retired, and not already resolved as `CHALLENGER_WINS`.

Relevant code:

* Parent validation during initialization:\
  `base/contracts/src/multiproof/AggregateVerifier.sol`\
  [AggregateVerifier.sol#L351-L360](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/multiproof/AggregateVerifier.sol#L351-L360)
* `_isValidGame()` definition:\
  `base/contracts/src/multiproof/AggregateVerifier.sol`\
  [AggregateVerifier.sol#L952-L957](https://github.com/base/contracts/blob/01dad230390cd69bcf130b5fc7a7a580b31650a7/src/multiproof/AggregateVerifier.sol#L952-L957)

In short, the root cause is that recovery caches a descendant based only on UUID reachability, while retry logic assumes that cached parent remains valid across submission failures. Once a previously cached dispute-game chain later becomes invalid onchain, the proposer can keep retrying the same stale parent instead of forcing a fresh recovery.

## Impact Details

This issue is a bug in Base’s L2 proposer / network-side code that results in unintended smart contract behavior with no concrete funds at direct risk.

Once the proposer cache contains a dispute-game descendant whose ancestry later becomes invalid onchain, subsequent proposal attempts can continue reusing that stale parent across failed submissions. Because the normal `SubmitOutcome::Failed` retry path does not invalidate or refresh `cached_recovery`, the proposer can keep retrying an invalid parent candidate while the cache key remains unchanged.

The result is not direct fund theft or a proof soundness failure. Instead, the proposer can continue attempting dispute-game creation with an onchain-invalid parent, causing unintended dispute-game submission behavior and delaying creation of the next valid game until recovery state is refreshed, the service is restarted, or another operator action occurs.

This is therefore best classified as a Medium-severity network / proposer logic bug that causes unintended smart contract behavior without concrete funds directly at risk.

## Proof of Concept

Repository/revision: `base/base@v0.8.0-rc.28`

Apply the following additions within the existing `#[cfg(test)] mod tests` block in `crates/proof/proposer/src/pipeline.rs`.

```rust
use std::sync::Mutex;
use async_trait::async_trait;
use base_proof_contracts::{ContractError, GameInfo};
```

```rust
#[derive(Debug)]
struct PanicAggregateVerifier;

#[async_trait]
impl AggregateVerifierClient for PanicAggregateVerifier {
    async fn game_info(&self, _game_address: Address) -> Result<GameInfo, ContractError> {
        panic!("unexpected AggregateVerifierClient::game_info call in recovery PoC");
    }

    async fn status(&self, _game_address: Address) -> Result<u8, ContractError> {
        panic!("unexpected AggregateVerifierClient::status call in recovery PoC");
    }

    async fn zk_prover(&self, _game_address: Address) -> Result<Address, ContractError> {
        panic!("unexpected AggregateVerifierClient::zk_prover call in recovery PoC");
    }

    async fn tee_prover(&self, _game_address: Address) -> Result<Address, ContractError> {
        panic!("unexpected AggregateVerifierClient::tee_prover call in recovery PoC");
    }

    async fn starting_block_number(
        &self,
        _game_address: Address,
    ) -> Result<u64, ContractError> {
        panic!("unexpected AggregateVerifierClient::starting_block_number call in recovery PoC");
    }

    async fn l1_head(&self, _game_address: Address) -> Result<B256, ContractError> {
        panic!("unexpected AggregateVerifierClient::l1_head call in recovery PoC");
    }

    async fn read_block_interval(&self, _impl_address: Address) -> Result<u64, ContractError> {
        panic!("unexpected AggregateVerifierClient::read_block_interval call in recovery PoC");
    }

    async fn read_intermediate_block_interval(
        &self,
        _impl_address: Address,
    ) -> Result<u64, ContractError> {
        panic!(
            "unexpected AggregateVerifierClient::read_intermediate_block_interval call in recovery PoC"
        );
    }

    async fn intermediate_output_roots(
        &self,
        _game_address: Address,
    ) -> Result<Vec<B256>, ContractError> {
        panic!(
            "unexpected AggregateVerifierClient::intermediate_output_roots call in recovery PoC"
        );
    }

    async fn intermediate_output_root(
        &self,
        _game_address: Address,
        _index: u64,
    ) -> Result<B256, ContractError> {
        panic!(
            "unexpected AggregateVerifierClient::intermediate_output_root call in recovery PoC"
        );
    }

    async fn countered_index(&self, _game_address: Address) -> Result<u64, ContractError> {
        panic!("unexpected AggregateVerifierClient::countered_index call in recovery PoC");
    }

    async fn game_over(&self, _game_address: Address) -> Result<bool, ContractError> {
        panic!("unexpected AggregateVerifierClient::game_over call in recovery PoC");
    }

    async fn resolved_at(&self, _game_address: Address) -> Result<u64, ContractError> {
        panic!("unexpected AggregateVerifierClient::resolved_at call in recovery PoC");
    }

    async fn bond_recipient(&self, _game_address: Address) -> Result<Address, ContractError> {
        panic!("unexpected AggregateVerifierClient::bond_recipient call in recovery PoC");
    }

    async fn bond_unlocked(&self, _game_address: Address) -> Result<bool, ContractError> {
        panic!("unexpected AggregateVerifierClient::bond_unlocked call in recovery PoC");
    }

    async fn bond_claimed(&self, _game_address: Address) -> Result<bool, ContractError> {
        panic!("unexpected AggregateVerifierClient::bond_claimed call in recovery PoC");
    }

    async fn expected_resolution(
        &self,
        _game_address: Address,
    ) -> Result<u64, ContractError> {
        panic!("unexpected AggregateVerifierClient::expected_resolution call in recovery PoC");
    }

    async fn proof_count(&self, _game_address: Address) -> Result<u8, ContractError> {
        panic!("unexpected AggregateVerifierClient::proof_count call in recovery PoC");
    }

    async fn created_at(&self, _game_address: Address) -> Result<u64, ContractError> {
        panic!("unexpected AggregateVerifierClient::created_at call in recovery PoC");
    }

    async fn delayed_weth(&self, _game_address: Address) -> Result<Address, ContractError> {
        panic!("unexpected AggregateVerifierClient::delayed_weth call in recovery PoC");
    }
}

#[derive(Debug, Default)]
struct RecordingFailingOutputProposer {
    parent_addresses: Mutex<Vec<Address>>,
}

impl RecordingFailingOutputProposer {
    fn recorded_parent_addresses(&self) -> Vec<Address> {
        self.parent_addresses
            .lock()
            .expect("recording proposer mutex poisoned")
            .clone()
    }
}

#[async_trait]
impl OutputProposer for RecordingFailingOutputProposer {
    async fn propose_output(
        &self,
        _proposal: &Proposal,
        parent_address: Address,
        _intermediate_roots: &[B256],
    ) -> Result<(), ProposerError> {
        self.parent_addresses
            .lock()
            .expect("recording proposer mutex poisoned")
            .push(parent_address);
        Err(ProposerError::TxReverted(
            "mock InvalidParentGame-style submit failure".into(),
        ))
    }
}

fn recovery_pipeline_full_with_clients(
    factory: MockDisputeGameFactory,
    output_roots: HashMap<u64, B256>,
    anchor_block: u64,
    block_interval: u64,
    intermediate_block_interval: u64,
    verifier_client: Arc<dyn AggregateVerifierClient>,
    output_proposer: Arc<dyn OutputProposer>,
) -> TestPipeline {
    let cancel = CancellationToken::new();
    let l1 = Arc::new(MockL1 { latest_block_number: TEST_L1_BLOCK_NUMBER });
    let l2 = Arc::new(MockL2 { block_not_found: true, canonical_hash: None });
    let prover: Arc<dyn ProverClient> =
        Arc::new(MockProver { delay: MOCK_PROVER_DELAY, block_interval });
    let rollup = Arc::new(MockRollupClient {
        sync_status: test_sync_status(0, B256::ZERO),
        output_roots,
        max_safe_block: None,
    });
    let anchor_registry =
        Arc::new(MockAnchorStateRegistry { anchor_root: test_anchor_root(anchor_block) });

    ProvingPipeline::new(
        PipelineConfig {
            max_parallel_proofs: 1,
            max_retries: 1,
            recovery_scan_concurrency: 8,
            tee_prover_registry_address: None,
            driver: DriverConfig {
                game_type: TEST_GAME_TYPE,
                block_interval,
                intermediate_block_interval,
                ..Default::default()
            },
        },
        prover,
        l1,
        l2,
        rollup,
        anchor_registry,
        Arc::new(factory),
        verifier_client,
        output_proposer,
        cancel,
    )
}

#[tokio::test(flavor = "current_thread", start_paused = true)]
async fn test_poc_recovery_seeds_poisoned_parent_and_retry_reuses_it() {
    let (factory, output_roots) = game_chain_full(
        3,
        TEST_ANCHOR_BLOCK,
        SUBMIT_BLOCK_INTERVAL,
        SUBMIT_INTERMEDIATE_INTERVAL,
    );
    let recording_proposer = Arc::new(RecordingFailingOutputProposer::default());
    let pipeline = recovery_pipeline_full_with_clients(
        factory,
        output_roots,
        TEST_ANCHOR_BLOCK,
        SUBMIT_BLOCK_INTERVAL,
        SUBMIT_INTERMEDIATE_INTERVAL,
        Arc::new(PanicAggregateVerifier),
        recording_proposer.clone(),
    );

    let mut cache: Option<CachedRecovery> = None;
    let recovered = pipeline.recover_latest_state(&mut cache).await.unwrap();

    assert_eq!(
        recovered.parent_address,
        proxy_addr(2),
        "recovery should advance to the latest UUID-linked descendant"
    );
    assert_eq!(
        recovered.l2_block_number,
        SUBMIT_BLOCK_INTERVAL * 3,
        "recovery should cache the latest descendant block"
    );
    assert!(cache.is_some(), "recovery should populate cached state");

    let target_block = recovered.l2_block_number + SUBMIT_BLOCK_INTERVAL;
    let start_block = target_block
        .checked_sub(SUBMIT_BLOCK_INTERVAL)
        .expect("target block must be at least one submit interval")
        + 1;
    let proposals: Vec<Proposal> = (start_block..=target_block).map(test_proposal).collect();
    let proof_result = ProofResult::Tee {
        aggregate_proposal: test_proposal(target_block),
        proposals,
    };

    let mut state = PipelineState::new();
    state.cached_recovery = cache;
    state.proved.insert(target_block, proof_result);

    pipeline.try_submit(&mut state);
    let first = state
        .submit_tasks
        .join_next()
        .await
        .expect("first submit task should exist");
    assert!(
        !pipeline.handle_submit_result(first, &mut state).await,
        "failed submission should defer retry to a later tick"
    );

    assert!(
        state.proved.contains_key(&target_block),
        "failed submission should requeue the proof for retry"
    );
    assert_eq!(
        state.cached_recovery
            .as_ref()
            .expect("cached recovery missing")
            .state
            .parent_address,
        recovered.parent_address,
        "failed submission should keep the recovered parent cached"
    );

    pipeline.try_submit(&mut state);
    let second = state
        .submit_tasks
        .join_next()
        .await
        .expect("second submit task should exist");
    assert!(
        !pipeline.handle_submit_result(second, &mut state).await,
        "second failed submission should also defer retry"
    );

    assert_eq!(
        recording_proposer.recorded_parent_addresses(),
        vec![recovered.parent_address, recovered.parent_address],
        "retry should keep reusing the same recovered parent address"
    );
}
```

Run:

```bash
cargo test -p base-proposer pipeline::tests::test_poc_recovery_seeds_poisoned_parent_and_retry_reuses_it -- --exact --nocapture
```

Observed output:

```
Finished `test` profile [unoptimized + debuginfo] target(s) in 1.57s
Running unittests src/lib.rs (target/debug/deps/base_proposer-f731c12d87d98136)

running 1 test
test pipeline::tests::test_poc_recovery_seeds_poisoned_parent_and_retry_reuses_it ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 74 filtered out; finished in 0.00s
```

This PoC intentionally mocks the submission failure path; the production invalid-parent rejection is enforced by `AggregateVerifier.initialize()` and `_isValidGame()` in `base/contracts`.
