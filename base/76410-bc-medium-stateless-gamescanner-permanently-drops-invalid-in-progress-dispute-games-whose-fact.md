# 76410 bc medium stateless gamescanner permanently drops invalid in progress dispute games whose factory index falls below gamecount lookback games after any single missed tick

*Submitted on May 4th 2026 at 10:26:48 UTC by @yesofcourse for* [*Audit Comp | Base Azul*](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76410
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Brief/Intro

The `base-challenger` service's `GameScanner` re-derives a trailing `lookback_games` window from the live `gameCount` on every tick with no monotonic watermark and no per-game backlog. The same crate's `BondManager` solves the equivalent discovery problem with a `bond_scan_head` watermark plus a periodic full rescan; the dispute scanner has neither. As a result, any in-progress invalid game whose first scan tick produces no candidate (because `Driver::validate_game` soft-skips on a transient `ValidatorError`) and whose factory index then falls below `gameCount - lookback_games` is excluded from every future scan. Game creation is permissionless on `DisputeGameFactory.createWithInitData`, so any L1 caller paying `initBonds(gameType)` can drive that index drift. The Base-run challenger then never validates the buried game, never requests a proof, and never submits a `nullify()` or `challenge()` transaction for it, even after the originating transient error has fully cleared.

## Vulnerability Details

The window arithmetic in `crates/proof/challenge/src/scanner.rs`, function `GameScanner::scan`, is recomputed from the live `gameCount` on every tick:

```rust
let game_count = self.factory_client.game_count().await?;
// ...
let end = game_count - 1;
let start = game_count.saturating_sub(self.config.lookback_games);

let results: Vec<(u64, Result<Option<CandidateGame>>)> = stream::iter(start..=end)
    .map(|i| async move { (i, self.evaluate_game(i).await) })
    .buffer_unordered(Self::SCAN_CONCURRENCY)
    .collect()
    .await;
```

There is no scan watermark and no per-game backlog. Once enough new games are appended to the factory, the index of an older in-progress game becomes strictly less than `start` and the scanner stops calling `evaluate_game(N)` for it.

The same crate's `BondManager` solves the equivalent discovery problem differently. In `crates/proof/challenge/src/bond.rs`, function `discover_claimable_games`, the doc comment is explicit:

```
    /// Discovers claimable games via two-tier scanning.
    ///
    /// **Incremental** (every call): scans from `bond_scan_head` to
    /// `game_count`, catching newly created games. Typically zero to a
    /// handful of games per tick, costing a single `game_count()` RPC
    /// when idle.
    ///
    /// **Periodic full rescan** (every `discovery_interval`):
    /// resets the watermark backward by `lookback` to re-evaluate games
    /// whose state may have changed (e.g. challenged or resolved by
    /// another actor since the last scan).
```

`BondManager` maintains a monotonic `bond_scan_head: u64` watermark, advances it forward after each scan, and resets it backward by `lookback` on a periodic timer, so a game once missed by index drift is unconditionally re-evaluated on the next `discovery_interval` boundary. `GameScanner` has neither.

The challenger ticks every `--poll-interval` seconds (default 12s), and at L1 throughput an attacker cannot append `lookback_games = 1000` games in one such window, so the bug only matters if the very first tick on the target index produces no `pending_proofs` entry. The driver has exactly such a path. In `crates/proof/challenge/src/driver.rs`, function `Driver::validate_game`:

```rust
match self.validator.validate_intermediate_roots(params).await {
    Ok(result) => Ok(Some((result, intermediate_roots))),
    Err(e) => {
        match &e {
            ValidatorError::BlockNotAvailable { .. } => { debug!(...); }
            _ => { warn!(...); }
        }
        Ok(None)
    }
}
```

Every `ValidatorError` variant collapses to `Ok(None)`. This includes `BlockNotAvailable` (an L2 RPC blip on `header_by_number` or `get_proof`, mid-import re-org, transient timeout) and the generic `Rpc(_)` variant. Then in `Driver::process_invalid_proposal`:

```rust
let result = match self.validate_game(&candidate).await? {
    Some((result, _)) => result,
    None => return Ok(()),
};
```

On `Ok(None)` the function returns early without calling `initiate_proof` and without inserting into `pending_proofs`. The same soft-skip applies to `Driver::process_fraudulent_zk_challenge`, so the bug applies symmetrically across the four `GameCategory` variants.

The burial chain in production:

1. An invalid in-progress game is created at factory index `N` via permissionless `DisputeGameFactory.createWithInitData`.
2. The Base challenger's first tick on `N` hits a transient L2 error on at least one checkpoint. `validate_game` returns `Ok(None)`. No `pending_proofs` entry is created.
3. Permissionless `createWithInitData` calls push `gameCount` past `N + lookback_games`. The attacker only needs one tick where the target was missed, and step 2 produces it.
4. L2 recovers.
5. Every subsequent tick recomputes `start = gameCount - lookback_games > N`. The scanner never includes `N` again. `pending_proofs` for the buried address remains empty for the lifetime of the running service.

No indirect recovery path exists. `Driver::poll_pending_proofs` only iterates `pending_proofs.addresses()`, so an empty entry is never re-touched. `Driver::discover_claimable_bonds` calls `BondManager::discover_claimable_games`, which has the watermark, but it concerns bond claiming and does not re-trigger dispute submission. There is no fallback discovery by `respectedGameType` lineage or by unresolved-status filter.

The Azul proofs spec describes challenger coverage without any window qualification. From the "Challenger" section: "Anyone can run a challenger. A challenger independently recomputes checkpoint output roots for in-progress games, identifies the first invalid claim, and submits the required dispute transaction." In the implementation the property only holds for indices within `gameCount - lookback_games` of the tip.

## Impact Details

The Base-run challenger, the official challenger that the Base team operates as the security backstop for the dispute system, can be permanently blinded to specific invalid in-progress dispute games. Both triggers are reachable without any privileged action: a transient `ValidatorError` on the first tick is a property of normal L2 RPC operation, and the index drift past `lookback_games` is reachable via permissionless `DisputeGameFactory.createWithInitData` calls.

This maps to the Blockchain / DLT row "A bug in the respective layer 0/1/2 network code that results in unintended smart-contract behavior with no concrete funds at direct risk". The qualifying behavior is that an in-progress dispute game whose `rootClaim` is provably wrong remains undisputed by the Base challenger, and the bond-slashing guarantee for its proposer does not materialize through the Base-operated dispute path, until a third-party challenger intervenes or an operator manually restarts the service with different configuration.

The bug does not by itself let an invalid output root finalize on-chain. Per "Anyone can run a challenger", a third-party challenger can still validate and dispute the buried game via the permissionless ZK path. The program's downgrade rule for reports that assume Base will not dispute or nullify an invalid proposal within the proof system therefore caps severity at Medium.

The post-Azul Sepolia deployment sets `INIT_BOND = 50000000000000000` (0.05 ETH) for `GAME_TYPE=621`, so an attacker locks up to `lookback_games * 0.05 ETH = 50 ETH` of recoverable bond capital across the appended games. Bonds return when each appended game resolves DEFENDER\_WINS, so the net economic cost is gas plus the opportunity cost of the bond lockup for one finality window. On Sepolia this is gas only.

## References

* Azul proofs spec, "Challenger" section: <https://specs.base.org/upgrades/azul/proofs>
* `GameScanner::scan` window arithmetic: <https://github.com/base/base/blob/v0.8.0-rc.28/crates/proof/challenge/src/scanner.rs>
* `BondManager::discover_claimable_games` watermark and periodic-rescan implementation: <https://github.com/base/base/blob/v0.8.0-rc.28/crates/proof/challenge/src/bond.rs>
* `Driver::validate_game` soft-skip path and `Driver::process_invalid_proposal` early-return on `Ok(None)`: <https://github.com/base/base/blob/v0.8.0-rc.28/crates/proof/challenge/src/driver.rs>
* `Driver::process_fraudulent_zk_challenge` symmetric soft-skip: <https://github.com/base/base/blob/v0.8.0-rc.28/crates/proof/challenge/src/driver.rs>
* `--lookback-games` CLI flag (default 1000): <https://github.com/base/base/blob/v0.8.0-rc.28/crates/proof/challenge/src/cli.rs>
* Existing soft-skip-on-`BlockNotAvailable` test (`test_step_validation_error_blocks_not_available`) that asserts the tick survives but not the post-recovery behavior: <https://github.com/base/base/blob/v0.8.0-rc.28/crates/proof/challenge/tests/driver.rs>
* Audit snapshot tag: <https://github.com/base/base/tree/v0.8.0-rc.28>
* Sepolia post-Azul deployment, `GAME_TYPE=621` and `INIT_BOND=50000000000000000`: <https://github.com/base/contract-deployments/blob/main/sepolia/2026-04-20-activate-multiproof/.env#L6-L17>

## Link to Proof of Concept

<https://gist.github.com/asendz/02e464a77824f7d050a83d94143345a0>

## Proof of Concept

### Running the PoC

Prerequisite: a Rust toolchain matching the workspace's `rust-toolchain.toml`. The workspace's `.cargo/config.toml` configures the macOS linker to `lld` at `/opt/homebrew/opt/lld/bin/ld64.lld`. Either install lld with `brew install lld`, or pass `RUSTFLAGS=""` to override the rustflags from `.cargo/config.toml` for this run.

The PoC test file ships at `crates/proof/challenge/tests/poc_lookback_burial.rs`. Run from the workspace root, in release mode:

```bash
cargo test -p base-challenger --release --test poc_lookback_burial -- --nocapture
```

If lld is not installed:

```bash
RUSTFLAGS="" cargo test -p base-challenger --release --test poc_lookback_burial -- --nocapture
```

PoC:

```rust
use std::{
    collections::{HashMap, HashSet},
    sync::{
        Arc, Mutex,
        atomic::{AtomicBool, AtomicU64, Ordering},
    },
    time::Duration,
};

use alloy_consensus::Header as ConsensusHeader;
use alloy_primitives::{Address, B256, Bytes, U256};
use alloy_rpc_types_eth::{EIP1186AccountProofResponse, Header as RpcHeader};
use async_trait::async_trait;
use base_challenger::{
    ChallengeSubmitter, Driver, DriverComponents, DriverConfig, GameScanner,
    IntermediateValidationParams, OutputValidator, ScannerConfig,
    test_utils::{
        DEFAULT_L1_HEAD, DEFAULT_TEE_PROVER, MockAggregateVerifier, MockGameState,
        MockTxManager, MockZkProofProvider, addr, build_test_header_and_account, factory_game,
        receipt_with_status,
    },
};
use base_proof_contracts::{
    AggregateVerifierClient, ContractError, DisputeGameFactoryClient, GameAtIndex, GameInfo,
};
use base_proof_rpc::{BaseBlock, L2Provider, RpcError, RpcResult};
use base_protocol::OutputRoot;
use base_tx_manager::{SendHandle, SendResponse, TxCandidate, TxManager};
use tokio_util::sync::CancellationToken;

const STORAGE_HASH: B256 = B256::repeat_byte(0xBB);
const BOGUS_ROOT: B256 = B256::repeat_byte(0xFF);
const VALID_GAME_TYPE: u32 = 1;
const STARTING_BLOCK: u64 = 10;
const ENDING_BLOCK: u64 = 20;
const INTERMEDIATE_INTERVAL: u64 = 5;
const LOOKBACK_GAMES: u64 = 1000;
const STATUS_DEFENDER_WINS: u8 = 2;

#[derive(Debug)]
struct GrowableFactory {
    games: Mutex<Vec<GameAtIndex>>,
}

impl GrowableFactory {
    fn new(initial: Vec<GameAtIndex>) -> Self {
        Self { games: Mutex::new(initial) }
    }

    fn append_game(&self, entry: GameAtIndex) {
        self.games.lock().unwrap().push(entry);
    }

    fn current_count(&self) -> u64 {
        self.games.lock().unwrap().len() as u64
    }
}

#[async_trait]
impl DisputeGameFactoryClient for GrowableFactory {
    async fn game_count(&self) -> Result<u64, ContractError> {
        Ok(self.games.lock().unwrap().len() as u64)
    }

    async fn game_at_index(&self, index: u64) -> Result<GameAtIndex, ContractError> {
        self.games
            .lock()
            .unwrap()
            .get(index as usize)
            .copied()
            .ok_or_else(|| ContractError::Validation(format!("index {index} out of bounds")))
    }

    async fn init_bonds(&self, _game_type: u32) -> Result<U256, ContractError> {
        Ok(U256::ZERO)
    }

    async fn game_impls(&self, _game_type: u32) -> Result<Address, ContractError> {
        Ok(Address::repeat_byte(0x11))
    }

    async fn games(
        &self,
        _game_type: u32,
        _root_claim: B256,
        _extra_data: Bytes,
    ) -> Result<Address, ContractError> {
        Ok(Address::ZERO)
    }
}

#[derive(Debug, Default)]
struct ToggleableL2 {
    headers: HashMap<u64, RpcHeader>,
    proofs: HashMap<B256, EIP1186AccountProofResponse>,
    error_blocks: Mutex<HashSet<u64>>,
}

impl ToggleableL2 {
    fn new() -> Self {
        Self::default()
    }

    fn insert_block(
        &mut self,
        block_number: u64,
        consensus_header: ConsensusHeader,
        account_result: EIP1186AccountProofResponse,
    ) {
        let block_hash = consensus_header.hash_slow();
        let rpc_header =
            RpcHeader { hash: block_hash, inner: consensus_header, ..Default::default() };
        self.headers.insert(block_number, rpc_header);
        self.proofs.insert(block_hash, account_result);
    }

    fn set_error_blocks(&self, blocks: &[u64]) {
        let mut set = self.error_blocks.lock().unwrap();
        set.clear();
        set.extend(blocks.iter().copied());
    }

    fn clear_error_blocks(&self) {
        self.error_blocks.lock().unwrap().clear();
    }
}

#[async_trait]
impl L2Provider for ToggleableL2 {
    async fn chain_config(&self) -> RpcResult<serde_json::Value> {
        Ok(serde_json::Value::Null)
    }

    async fn get_proof(
        &self,
        _address: Address,
        block_hash: B256,
    ) -> RpcResult<EIP1186AccountProofResponse> {
        self.proofs
            .get(&block_hash)
            .cloned()
            .ok_or_else(|| RpcError::ProofNotFound(format!("no proof for hash {block_hash}")))
    }

    async fn header_by_number(&self, number: Option<u64>) -> RpcResult<RpcHeader> {
        let block_number = number.unwrap_or(0);
        if self.error_blocks.lock().unwrap().contains(&block_number) {
            return Err(RpcError::BlockNotFound(format!("block {block_number} not available")));
        }
        self.headers
            .get(&block_number)
            .cloned()
            .ok_or_else(|| RpcError::HeaderNotFound(format!("no header for block {block_number}")))
    }

    async fn block_by_number(&self, _number: Option<u64>) -> RpcResult<BaseBlock> {
        Err(RpcError::BlockNotFound("not implemented in mock".into()))
    }

    async fn block_by_hash(&self, _hash: B256) -> RpcResult<BaseBlock> {
        Err(RpcError::BlockNotFound("not implemented in mock".into()))
    }
}

fn build_invalid_game_state(l2: &mut ToggleableL2) -> (MockGameState, B256) {
    let (header_15, account_15) = build_test_header_and_account(15, STORAGE_HASH);
    let root_15 =
        OutputRoot::from_parts(header_15.state_root, STORAGE_HASH, header_15.hash_slow()).hash();
    l2.insert_block(15, header_15, account_15);

    let (header_20, account_20) = build_test_header_and_account(ENDING_BLOCK, STORAGE_HASH);
    let expected_root_20 =
        OutputRoot::from_parts(header_20.state_root, STORAGE_HASH, header_20.hash_slow()).hash();
    l2.insert_block(ENDING_BLOCK, header_20, account_20);

    let state = MockGameState {
        status: 0,
        zk_prover: Address::ZERO,
        tee_prover: DEFAULT_TEE_PROVER,
        game_info: GameInfo {
            root_claim: BOGUS_ROOT,
            l2_block_number: ENDING_BLOCK,
            parent_address: Address::ZERO,
        },
        starting_block_number: STARTING_BLOCK,
        l1_head: DEFAULT_L1_HEAD,
        intermediate_output_roots: vec![root_15, BOGUS_ROOT],
        countered_index: 0,
        ..Default::default()
    };
    (state, expected_root_20)
}

#[derive(Debug)]
struct CountingMockTxManager {
    inner: MockTxManager,
    send_count: Arc<AtomicU64>,
}

impl CountingMockTxManager {
    fn new(inner: MockTxManager) -> (Self, Arc<AtomicU64>) {
        let send_count = Arc::new(AtomicU64::new(0));
        (Self { inner, send_count: Arc::clone(&send_count) }, send_count)
    }
}

impl TxManager for CountingMockTxManager {
    async fn send(&self, candidate: TxCandidate) -> SendResponse {
        self.send_count.fetch_add(1, Ordering::SeqCst);
        self.inner.send(candidate).await
    }

    async fn send_async(&self, candidate: TxCandidate) -> SendHandle {
        self.inner.send_async(candidate).await
    }

    fn sender_address(&self) -> Address {
        self.inner.sender_address()
    }
}

fn build_filler_game_state(block_number: u64) -> MockGameState {
    MockGameState {
        status: STATUS_DEFENDER_WINS,
        zk_prover: Address::ZERO,
        tee_prover: DEFAULT_TEE_PROVER,
        game_info: GameInfo {
            root_claim: B256::repeat_byte(block_number as u8),
            l2_block_number: block_number,
            parent_address: Address::ZERO,
        },
        starting_block_number: block_number.saturating_sub(INTERMEDIATE_INTERVAL),
        l1_head: DEFAULT_L1_HEAD,
        intermediate_output_roots: vec![B256::repeat_byte(block_number as u8)],
        countered_index: 0,
        ..Default::default()
    }
}

#[tokio::test(flavor = "current_thread")]
async fn burial_via_transient_validator_error_persists_post_recovery() {
    // Wire mocks. L2 starts in error mode for blocks 15 and 20.
    let mut l2 = ToggleableL2::new();
    let (invalid_state, expected_root_at_block_20) = build_invalid_game_state(&mut l2);
    let l2 = Arc::new(l2);
    l2.set_error_blocks(&[15, ENDING_BLOCK]);

    let factory: Arc<GrowableFactory> = Arc::new(GrowableFactory::new(Vec::new()));
    let verifier: Arc<MockAggregateVerifier> = Arc::new(MockAggregateVerifier::new(HashMap::new()));

    let inner_tx = MockTxManager::new(Ok(receipt_with_status(true, B256::repeat_byte(0xAA))));
    let (tx_manager, send_count) = CountingMockTxManager::new(inner_tx);

    let zk = Arc::new(MockZkProofProvider {
        session_id: "poc".to_string(),
        ..Default::default()
    });

    let scanner = GameScanner::new(
        Arc::clone(&factory) as Arc<dyn DisputeGameFactoryClient>,
        Arc::clone(&verifier) as Arc<dyn AggregateVerifierClient>,
        ScannerConfig { lookback_games: LOOKBACK_GAMES },
    );
    let validator = OutputValidator::new(Arc::clone(&l2));
    let submitter = ChallengeSubmitter::new(tx_manager);

    let mut driver: Driver<ToggleableL2, MockZkProofProvider, CountingMockTxManager> = Driver::new(
        DriverConfig {
            poll_interval: Duration::from_millis(10),
            cancel: CancellationToken::new(),
            ready: Arc::new(AtomicBool::new(false)),
        },
        DriverComponents {
            scanner,
            validator,
            zk_prover: zk,
            submitter,
            tee: None,
            verifier_client: Arc::clone(&verifier) as Arc<dyn AggregateVerifierClient>,
            bond_manager: None,
        },
    );

    assert_eq!(factory.current_count(), 0);
    assert_eq!(driver.pending_proofs.len(), 0);
    assert_eq!(send_count.load(Ordering::SeqCst), 0);

    // Invalid game created at index 0.
    verifier.update_game(addr(0), invalid_state);
    factory.append_game(factory_game(0, VALID_GAME_TYPE));
    assert_eq!(factory.current_count(), 1);

    // Tick 1: L2 errors on block 15, validator soft-skips, no pending entry.
    driver.step().await.expect("tick 1 succeeds (soft-skip path)");
    assert!(!driver.pending_proofs.contains_key(&addr(0)));
    assert_eq!(send_count.load(Ordering::SeqCst), 0);

    // Attacker appends LOOKBACK_GAMES filler games via permissionless
    // `DisputeGameFactory.createWithInitData`.
    for i in 1..=LOOKBACK_GAMES {
        let block = ENDING_BLOCK + i * INTERMEDIATE_INTERVAL;
        verifier.update_game(addr(i), build_filler_game_state(block));
        factory.append_game(factory_game(i, VALID_GAME_TYPE));
    }
    assert_eq!(factory.current_count(), 1 + LOOKBACK_GAMES);

    // L2 recovers.
    l2.clear_error_blocks();

    // Ticks 2-4: window is now [1..LOOKBACK_GAMES]; index 0 excluded.
    for tick in 2..=4 {
        driver.step().await.unwrap_or_else(|e| panic!("tick {tick}: {e}"));
        assert!(!driver.pending_proofs.contains_key(&addr(0)));
        assert_eq!(send_count.load(Ordering::SeqCst), 0);
    }
    assert_eq!(driver.pending_proofs.len(), 0);

    // Verifier state for the buried game is unchanged across all four ticks.
    {
        let live_state = verifier.games.lock().unwrap();
        let game = live_state.get(&addr(0)).expect("verifier still tracks addr(0)");
        assert_eq!(game.status, 0);
        assert_eq!(game.tee_prover, DEFAULT_TEE_PROVER);
        assert_eq!(game.intermediate_output_roots[1], BOGUS_ROOT);
    }

    // Direct evaluate_game(0) bypasses the window and confirms the candidate.
    let direct_eval = driver.scanner.evaluate_game(0).await.expect("evaluate_game(0)");
    let candidate = direct_eval.expect("Some(CandidateGame)");
    assert_eq!(candidate.index, 0);
    assert_eq!(candidate.factory.proxy, addr(0));
    assert_eq!(candidate.info.root_claim, BOGUS_ROOT);
    assert_eq!(candidate.info.l2_block_number, ENDING_BLOCK);
    assert_eq!(candidate.tee_prover, DEFAULT_TEE_PROVER);
    assert!(matches!(candidate.category, base_challenger::GameCategory::InvalidTeeProposal));

    // Validator on the recovered L2 catches the divergence directly.
    let intermediate_roots =
        verifier.intermediate_output_roots(addr(0)).await.expect("intermediate roots");
    let validation = driver
        .validator
        .validate_intermediate_roots(IntermediateValidationParams {
            game_address: addr(0),
            starting_block_number: STARTING_BLOCK,
            l2_block_number: ENDING_BLOCK,
            intermediate_block_interval: INTERMEDIATE_INTERVAL,
            claimed_root: BOGUS_ROOT,
            intermediate_roots: &intermediate_roots,
        })
        .await
        .expect("validator query");
    assert!(!validation.is_valid);
    assert_eq!(validation.invalid_intermediate_index, Some(1));
    assert_eq!(validation.expected_root, expected_root_at_block_20);

    assert_eq!(send_count.load(Ordering::SeqCst), 0);
}
```
