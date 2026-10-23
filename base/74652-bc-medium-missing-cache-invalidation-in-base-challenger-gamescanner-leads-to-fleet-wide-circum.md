# 74652 bc medium missing cache invalidation in base challenger gamescanner leads to fleet wide circumvention of the dispute challenge mechanism after governance setimplementation&#x20;

> Submitted on Apr 24th 2026 at 02:57:29 UTC by @Lucky8 for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74652
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/releases/tag/v0.8.0-rc.15>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Summary

GameScanner in the base-challenger (Rust) caches `INTERMEDIATE_BLOCK_INTERVAL()` per GameType in a `Mutex<HashMap<u32, u64>>` that is populated lazily and never invalidated. After a routine governance `DisputeGameFactory.setImplementation(GameType, newImpl)` call where `newImpl.INTERMEDIATE_BLOCK_INTERVAL()` differs from the prior impl, every challenger process already running holds a stale value. From that point on, every new dispute game of the upgraded GameType returns `ValidatorError::CheckpointCountMismatch` inside `validate_intermediate_roots`, which `driver.rs::validate_game` routes through a generic `_ => warn!()` arm that returns `Ok(None)` and silently skips the game. The scanner is stateless, so it re-evaluates the game on every tick — but it re-reads the same cached stale interval, so the skip is permanent per process. Any invalid proposal submitted after the governance upgrade is non-challengeable by that challenger instance until restart. Because every challenger runs identical code with identical cache semantics, a single governance upgrade desynchronises the entire challenger fleet simultaneously.

## Vulnerable code (v0.8.0-rc.15, SHA de349fc9e8bf61531ce36ca57572345b03b2b097)

* `crates/proof/challenge/src/scanner.rs:140` — `interval_cache: Mutex<HashMap<u32, u64>>` (declaration).
* `crates/proof/challenge/src/scanner.rs:338-366` — `resolve_intermediate_block_interval`: the cache-hit path on lines 340-343 returns the cached value unconditionally and never revalidates against the on-chain impl.
* `crates/proof/challenge/src/validator.rs:308-344` — `validate_intermediate_roots`: the stale interval feeds into `expected_count = span / intermediate_block_interval`; when the on-chain count (computed with the new impl’s interval) disagrees, the function returns `ValidatorError::CheckpointCountMismatch`.
* `crates/proof/challenge/src/driver.rs:288-328` — `validate_game`: the `_ => warn!()` arm (lines 317-323) treats every non-`BlockNotAvailable` validator error as a transient skip, returning `Ok(None)`. This silently swallows the mismatch.

There is no cache TTL, no `ImplementationSet` event subscription, and no cross-checking of the impl address against the cache key. A process restart is the only action that clears the cache.

## Preconditions (all routine)

1. A challenger instance is running before the `setImplementation` call (standard 24/7 security-fleet posture).
2. The new implementation has a different `INTERMEDIATE_BLOCK_INTERVAL()` than the old one. The Azul upgrade spec establishes that `AggregateVerifier` configuration is immutable per deployment with explicit per-deployment variation ("Their addresses are immutable on the AggregateVerifier implementation, so each deployment has an explicit verifier set" — `docs/specs/pages/upgrades/azul/proofs.md:60`). The same pattern is documented for `BLOCK_INTERVAL` at `crates/proof/contracts/src/aggregate_verifier.rs:41` ("immutable on the implementation") and governs `INTERMEDIATE_BLOCK_INTERVAL` by the same ABI shape. The proposer reads both values per-impl-address at startup and runtime-checks their divisibility (`crates/proof/proposer/src/service.rs:117-131`), corroborating per-impl variability. Operationally no deployed impl yet uses a non-default interval; this is a latent activation bug, not presently exploitable on today's chain.
3. An invalid proposal of the upgraded GameType is created post-upgrade. The chained drain scenario requires a compromised or rogue authorized prover; whether that is in-scope for this program determines whether the chained impact applies.

## Impact

Primary (Medium): a bug in L2 network code that produces unintended dispute-game behaviour with no concrete funds directly at risk. The scope-named impact "Circumventing the dispute/challenge mechanism to prevent correction of an invalid proposal before finalization" is instantiated precisely.

Fleet-correlation argument (High): because every challenger runs identical code with identical cache semantics and the cache is only invalidated by process restart, a single governance upgrade produces a simultaneous fleet-wide silent challenger brownout for the upgraded GameType. This violates the optimistic design's assumption of uncorrelated challenger failure and cannot be mitigated by adding redundancy.

Chained (Critical, conditional): when combined with an invalid proposal from any authorized prover, the stale cache is the mechanism by which a forged state root reaches finalization. At finalisation an attacker can execute invalid withdrawals through the L1 bridge ("Draining or stealing funds from the L1 bridge portal through invalid withdrawal proofs constructed against a forged finalized state").

## Severity claim

Medium primary, chainable to Critical per the bridge-drain scope item, with High defensible on the fleet-correlation argument. Deferring to triage on program posture regarding a compromised-prover threat model.

## Fix (for reference)

Key the cache on `(game_type, impl_address)` or subscribe to `DisputeGameFactory.ImplementationSet` for invalidation. A short TTL is a partial bound. Ranked options are in the full report.

## Link to Proof of Concept

<https://gist.github.com/trgarrett/22e3ce37e4e5091561a00f1497f078b0>

## Proof of Concept

{% stepper %}
{% step %}
T0: governance registered game\_type=0 -> impl\_v1 (interval=10)

T1: challenger's FIRST resolve(game\_type=0) = 10 (cached)
{% endstep %}

{% step %}
T2: `DisputeGameFactory.setImplementation(game_type=0, impl_v2)` called by owner.\
On-chain: game\_type=0 -> impl\_v2 (interval=50)
{% endstep %}

{% step %}
T3: challenger's resolve(game\_type=0) AFTER upgrade = 10 (still stale from cache)
{% endstep %}

{% step %}
T4: new game of type 0 created. starting=100, l2\_block=600.\
Impl-v2 stores 10 intermediate roots on-chain.\
Challenger's cached interval = 10.
{% endstep %}

{% step %}
\==== VULNERABILITY TRIGGERED ====

ValidatorError (matches `_ => warn!()` arm in `driver.rs:317-323`):\
`CheckpointCountMismatch`: expected 50 (using cached interval 10), actual 10 (stored by on-chain impl)

`driver.rs::validate_game` at this point returns `Ok(None)`, silently skipping the game.

Scanner is stateless and re-evaluates every tick, but every re-evaluation hits the SAME cached (stale) interval value, so the game is PERMANENTLY skipped until the challenger process restarts. Any invalid proposal submitted after the governance upgrade is non-challengeable by this challenger instance.
{% endstep %}
{% endstepper %}
