# 74791 bc medium stale intermediate block interval cache forces dispute games into incorrect resolved state

**Submitted on Apr 24th 2026 at 21:22:24 UTC by @abarbatei for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74791
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.24>
* **Impacts:**
  * Direct loss to Base or users ≥ 10% of funds held within Bridge.

## Description

### Brief/Intro

The challenger's `GameScanner` permanently caches the `INTERMEDIATE_BLOCK_INTERVAL` value per game type on first lookup and never invalidates it. After a legitimate governance upgrade via `DisputeGameFactory.setImplementation()` that changes the interval, the challenger computes the wrong expected checkpoint count for all new games, triggering a `CheckpointCountMismatch` error that causes the game to be silently skipped. A malicious proposer who creates a game during this post-upgrade window has their invalid game escape the entire challenge window, resolving as `DEFENDER_WINS` when `CHALLENGER_WINS` should apply.

## Vulnerability Details

### Affected Code

File: `crates/proof/challenge/src/scanner.rs`, function `resolve_intermediate_block_interval`:

```rust
async fn resolve_intermediate_block_interval(&self, game_type: u32) -> Result<u64> {
    {
        let cache = self.interval_cache.lock().unwrap();
        if let Some(&interval) = cache.get(&game_type) {
            return Ok(interval);  // BUG: never re-validated after contract upgrade
        }
    }
    let impl_address = self.factory_client.game_impls(game_type).await?;
    let interval = self.verifier_client
        .read_intermediate_block_interval(impl_address).await?;
    let mut cache = self.interval_cache.lock().unwrap();
    cache.insert(game_type, interval);
    Ok(interval)
}
```

`interval_cache` is a `Mutex<HashMap<u32, u64>>` that maps `game_type` to the interval value. It is populated on the first cache miss via an RPC call to the implementation contract's `INTERMEDIATE_BLOCK_INTERVAL` constant. Once populated, the cache entry persists for the entire process lifetime. No mechanism exists to detect that the implementation contract address has changed or that the on-chain interval value has been updated.

### Attack Sequence

1. **Challenger starts.** The first call to `resolve_intermediate_block_interval(game_type=1)` fetches `INTERMEDIATE_BLOCK_INTERVAL = 100` from `impl_v1`. The cache stores `{1 => 100}`.
2. **Governance upgrade.** `DisputeGameFactory.setImplementation(1, impl_v2)` is called, where `impl_v2` has `INTERMEDIATE_BLOCK_INTERVAL = 50`.
3. **Malicious proposer creates a game** with `starting_block = 1000`, `l2_block = 1200`, producing 4 intermediate roots based on the new interval of 50.
4. **Challenger evaluates the game.** `resolve_intermediate_block_interval(1)` hits the cache and returns the stale value `100`. The validator computes `expected_checkpoints = (1200 - 1000) / 100 = 2`, but the game has 4 roots. The result is `ValidatorError::CheckpointCountMismatch { expected: 2, actual: 4 }`.
5. **Silent skip.** The `validate_game` function logs a warning, returns `Ok(None)`, and the game receives no further processing. No challenge is submitted.
6. **Challenge window expires.** The game resolves as `DEFENDER_WINS`. The proposer's invalid state root is finalized.

### Note: Why This Is Not the Known Proposer Config Issue

The publicly disclosed known vulnerability "Proposer Config Values Require a Redeploy to Change" describes the **proposer** failing to update bond values and block intervals, causing the proposer's own transactions to revert (self-harm, liveness issue). This finding targets the **challenger** side, where the stale cache causes the challenger to silently skip a game that should be challenged. The failure mode is fundamentally different:

| Dimension       | Known Issue (Proposer)       | This Finding (Challenger)                   |
| --------------- | ---------------------------- | ------------------------------------------- |
| Component       | Proposer                     | Challenger scanner                          |
| Failure mode    | Proposer transactions revert | Game silently skipped                       |
| Who is harmed   | Proposer (self-harm)         | Protocol and users                          |
| Exploitability  | Not attacker-exploitable     | Attacker creates game during upgrade window |
| Security impact | Liveness (proposer restarts) | Safety (invalid game finalizes)             |

## Impact Details

A malicious proposer who monitors governance upgrade transactions can exploit this vulnerability to finalize a game with an incorrect state root. The impact chain is:

1. **Invalid state root finalized on L1** - the game resolves `DEFENDER_WINS` without challenge, anchoring a fraudulent L2 state in the `AnchorStateRegistry`.
2. **Fraudulent withdrawals** - with a finalized invalid state root, the attacker can prove withdrawals against fabricated L2 state via `OptimismPortal.proveWithdrawalTransaction()`, potentially draining bridge funds.
3. **All games of the upgraded type are affected** - every game created after the upgrade but before the challenger is manually restarted is silently skipped. The window is unbounded until operator intervention.

This matches the in-scope Blockchain/DLT impact: **"Direct loss to Base or users ≥ 10% of funds held within Bridge."** The attacker controls the fabricated L2 state root and can prove withdrawals for arbitrary amounts against it via `OptimismPortal.proveWithdrawalTransaction()`, putting the entire bridge balance at risk - well above the 10% threshold.

## References

* Vulnerable code: `crates/proof/challenge/src/scanner.rs` lines 338-366 in <https://github.com/base/base/releases/tag/v0.8.0-rc.15>
* `interval_cache` retained as optimization in PR #2219 (stateless scanner, 2026-04-16) - the cache was not changed
* No fix found in any PR up to #2336
* Related (but distinct): Known vulnerability "Proposer Config Values Require a Redeploy to Change"

## Proof of Concept

The PoC code is provided after the setup and containing two unit tests and all supporting mock types. To integrate and run:

### Setup

1. Copy the `UpgradeableImplMockFactory` struct and its `DisputeGameFactoryClient` impl into `crates/proof/challenge/src/test_utils.rs`, alongside the existing mock types (after `MockDisputeGameFactory`).
2. Add the `impl_interval_map: HashMap<Address, u64>` field to the existing `MockAggregateVerifier` struct, add the `with_impl_intervals` constructor, and update `read_intermediate_block_interval` to use it (all shown in the PoC file).
3. Paste the two `poc_stale_interval_*` test functions into the `#[cfg(test)] mod tests` block at the end of the same file.

### Run

```bash
cd base/
cargo test -p base-challenger poc_stale_interval
```

### What the tests demonstrate

**Test 1 - `poc_stale_interval_cache_populated_from_impl`** (baseline): On a fresh `GameScanner`, the first `scan()` call fetches `INTERMEDIATE_BLOCK_INTERVAL` from the implementation contract (`IMPL_OLD`) and caches `INTERVAL_OLD = 100`. Confirms the cache-population path works correctly.

**Test 2 - `poc_stale_interval_cache_after_governance_upgrade`** (exploit): After the first scan populates the cache, `upgrade_impl(IMPL_NEW)` simulates a governance call to `DisputeGameFactory.setImplementation()`. The second `scan()` call hits the stale cache and returns `INTERVAL_OLD = 100` instead of querying the new implementation which would return `INTERVAL_NEW = 50`. The test asserts:

1. Post-upgrade, the cached interval is still `100` (stale) - not `50` (current on-chain value)
2. The scanner never re-queries `game_impls()` or `read_intermediate_block_interval()` for a game type that was already cached
3. Any game created with the new interval would have a checkpoint count computed from `INTERVAL_NEW`, but the challenger validates against `INTERVAL_OLD` - producing a `CheckpointCountMismatch` that causes the game to be silently skipped

```rust
// PoC: Stale `intermediate_block_interval` Cache Forces Dispute Games
// Into Incorrect Resolved State
//
// Integration steps:
//   1. Add `UpgradeableImplMockFactory` (Section A) into
//      `crates/proof/challenge/src/test_utils.rs` alongside the existing
//      mock types (after `MockDisputeGameFactory`).
//   2. Apply the three modifications to `MockAggregateVerifier` (Section B):
//      new field, new constructor, updated trait method.
//   3. Paste the two test functions (Section C) into the existing
//      `#[cfg(test)] mod tests { ... }` block in the same file.
//
// Run:
//   cd base/
//   cargo test -p base-challenger poc_stale_interval

// =========================================================================
// Section A — New mock type (add at module level in test_utils.rs)
// =========================================================================

/// Mock factory where the `game_impls()` return address can be changed at
/// runtime to simulate a governance upgrade via
/// `DisputeGameFactory.setImplementation()`.
#[derive(Debug)]
pub struct UpgradeableImplMockFactory {
    pub inner: MockDisputeGameFactory,
    pub current_impl: Mutex<Address>,
}

impl UpgradeableImplMockFactory {
    pub fn new(games: Vec<GameAtIndex>, initial_impl: Address) -> Self {
        Self { inner: MockDisputeGameFactory { games }, current_impl: Mutex::new(initial_impl) }
    }

    /// Simulates `DisputeGameFactory.setImplementation(game_type, new_impl)`.
    pub fn upgrade_impl(&self, new_impl: Address) {
        *self.current_impl.lock().unwrap() = new_impl;
    }
}

#[async_trait]
impl DisputeGameFactoryClient for UpgradeableImplMockFactory {
    async fn game_count(&self) -> Result<u64, ContractError> {
        self.inner.game_count().await
    }

    async fn game_at_index(&self, index: u64) -> Result<GameAtIndex, ContractError> {
        self.inner.game_at_index(index).await
    }

    async fn init_bonds(&self, game_type: u32) -> Result<U256, ContractError> {
        self.inner.init_bonds(game_type).await
    }

    async fn game_impls(&self, _game_type: u32) -> Result<Address, ContractError> {
        Ok(*self.current_impl.lock().unwrap())
    }

    async fn games(
        &self,
        game_type: u32,
        root_claim: B256,
        extra_data: Bytes,
    ) -> Result<Address, ContractError> {
        self.inner.games(game_type, root_claim, extra_data).await
    }
}

// =========================================================================
// Section B — Modifications to existing MockAggregateVerifier
// =========================================================================
//
// 1. Add field to the struct:
//
//        pub struct MockAggregateVerifier {
//            pub games: Mutex<HashMap<Address, MockGameState>>,
//    +       pub impl_interval_map: HashMap<Address, u64>,   // <-- new
//        }
//
// 2. Add constructor (inside `impl MockAggregateVerifier`):
//
//        pub fn with_impl_intervals(
//            games: HashMap<Address, MockGameState>,
//            impl_interval_map: HashMap<Address, u64>,
//        ) -> Self {
//            Self { games: Mutex::new(games), impl_interval_map }
//        }
//
// 3. Update existing `new()` to initialize the new field:
//
//        pub fn new(games: HashMap<Address, MockGameState>) -> Self {
//            Self { games: Mutex::new(games), impl_interval_map: HashMap::new() }
//        }
//
// 4. Replace `read_intermediate_block_interval` in the
//    `AggregateVerifierClient` impl:
//
//        async fn read_intermediate_block_interval(
//            &self,
//            impl_address: Address,
//        ) -> Result<u64, ContractError> {
//            Ok(self.impl_interval_map.get(&impl_address).copied().unwrap_or(5))
//        }

// =========================================================================
// Section C — PoC tests (add inside `#[cfg(test)] mod tests { ... }`)
// ==========================================================================
//
// Required imports (already present in the test module):
//   use super::*;
//   use crate::scanner::{GameScanner, ScannerConfig};
// =========================================================================

// `GameScanner::resolve_intermediate_block_interval` caches the result of
// `factory.game_impls(game_type)` -> `verifier.read_intermediate_block_interval(impl)`
// in `interval_cache: Mutex<HashMap<u32, u64>>`. Once populated, the cache
// is NEVER invalidated.
//
// Attack scenario:
//   1. Governance calls `DisputeGameFactory.setImplementation(game_type, new_impl)`
//      where `new_impl` has a different `INTERMEDIATE_BLOCK_INTERVAL`.
//   2. Malicious proposer creates a game under `new_impl` with checkpoint
//      count matching the new interval.
//   3. Challenger's stale cache computes wrong expected checkpoint count
//      -> `CheckpointCountMismatch` -> game silently skipped.
//   4. Challenge window expires -> game resolves DEFENDER_WINS uncontested.

/// Baseline: on a fresh scanner, the interval is fetched from the
/// implementation contract and cached correctly.
#[tokio::test]
async fn poc_stale_interval_cache_populated_from_impl() {
    const IMPL_OLD: Address = Address::repeat_byte(0x11);
    const IMPL_NEW: Address = Address::repeat_byte(0x22);
    const INTERVAL_OLD: u64 = 100;
    const INTERVAL_NEW: u64 = 50;

    let factory = Arc::new(UpgradeableImplMockFactory::new(
        vec![factory_game(0, 1)],
        IMPL_OLD,
    ));

    let mut impl_intervals = HashMap::new();
    impl_intervals.insert(IMPL_OLD, INTERVAL_OLD);
    impl_intervals.insert(IMPL_NEW, INTERVAL_NEW);

    let mut verifier_games = HashMap::new();
    verifier_games.insert(addr(0), mock_state(0, Address::ZERO, 100));

    let verifier = Arc::new(MockAggregateVerifier::with_impl_intervals(
        verifier_games,
        impl_intervals,
    ));

    let scanner = GameScanner::new(factory, verifier, ScannerConfig { lookback_games: 1000 });

    let candidates = scanner.scan().await.unwrap();
    assert_eq!(candidates.len(), 1);
    assert_eq!(
        candidates[0].intermediate_block_interval,
        INTERVAL_OLD,
        "first scan must populate cache with interval from IMPL_OLD"
    );
}

/// Exploit: after a governance upgrade, the scanner returns the STALE
/// cached interval instead of re-querying the new implementation.
///
/// The stale value causes the driver to compute wrong expected checkpoint
/// counts, producing `CheckpointCountMismatch` and silently skipping the
/// game — allowing it to resolve DEFENDER_WINS uncontested.
#[tokio::test]
async fn poc_stale_interval_cache_after_governance_upgrade() {
    const IMPL_OLD: Address = Address::repeat_byte(0x11);
    const IMPL_NEW: Address = Address::repeat_byte(0x22);
    const INTERVAL_OLD: u64 = 100;
    const INTERVAL_NEW: u64 = 50;

    let factory_concrete = Arc::new(UpgradeableImplMockFactory::new(
        vec![factory_game(0, 1)],
        IMPL_OLD,
    ));
    let factory: Arc<dyn DisputeGameFactoryClient> = factory_concrete.clone();

    let mut impl_intervals = HashMap::new();
    impl_intervals.insert(IMPL_OLD, INTERVAL_OLD);
    impl_intervals.insert(IMPL_NEW, INTERVAL_NEW);

    let mut verifier_games = HashMap::new();
    verifier_games.insert(addr(0), mock_state(0, Address::ZERO, 100));

    let verifier = Arc::new(MockAggregateVerifier::with_impl_intervals(
        verifier_games,
        impl_intervals,
    ));

    let scanner = GameScanner::new(factory, verifier, ScannerConfig { lookback_games: 1000 });

    // Tick 1: cache populated with INTERVAL_OLD for game_type=1.
    let tick1 = scanner.scan().await.unwrap();
    assert_eq!(tick1.len(), 1);
    assert_eq!(tick1[0].intermediate_block_interval, INTERVAL_OLD);

    // Governance upgrade: factory.game_impls(1) now returns IMPL_NEW.
    // A fresh (uncached) query would return INTERVAL_NEW = 50.
    factory_concrete.upgrade_impl(IMPL_NEW);

    // Tick 2: cache hit — stale INTERVAL_OLD returned instead of INTERVAL_NEW.
    let tick2 = scanner.scan().await.unwrap();
    assert_eq!(tick2.len(), 1);
    assert_eq!(
        tick2[0].intermediate_block_interval,
        INTERVAL_OLD, // BUG: stale value, should be INTERVAL_NEW
        "BUG confirmed: interval_cache not invalidated after governance upgrade — \
         stale interval {INTERVAL_OLD} returned instead of current {INTERVAL_NEW}"
    );

    assert_ne!(
        tick2[0].intermediate_block_interval,
        INTERVAL_NEW,
        "if this fails, the bug was fixed — interval_cache is now invalidated on upgrade"
    );
}
```
