# 75256 bc medium base proposer uses the wrong parent after asr advancement plus full recovery

**Submitted on Apr 28th 2026 at 03:38:29 UTC by @johnluxe for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75256
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk
  * Temporary blocking of L2-to-L1 withdrawal finalization due to stalled output proposals after ASR advancement plus proposer full recovery.

## Description

## Brief/Intro

After `AnchorStateRegistry` advances to an `anchorGame`, `base-proposer` full recovery keeps the current anchor root and block but loses the anchor game address. It then submits the next dispute game with `parent = AnchorStateRegistry`. `AggregateVerifier` treats that parent as the original starting anchor, so the next valid output proposal reverts. In production, this stalls automated output proposal creation after cold start, cache reset, or stale-cache recovery, and withdrawals tied to newer outputs cannot finalize until a registered proposer submits the next game with the correct parent or the proposer is patched.

## Vulnerability Details

When ASR has no anchor game, `parent = AnchorStateRegistry` is correct. After ASR advances, `getAnchorRoot()` returns data from the current anchor game:

* `repos/contracts-v8.1.0/src/dispute/AnchorStateRegistry.sol:188`
* `repos/contracts-v8.1.0/src/dispute/AnchorStateRegistry.sol:195`

The ASR contract exposes the missing address through `anchorGame()`:

* `repos/contracts-v8.1.0/src/dispute/AnchorStateRegistry.sol:41`
* `repos/contracts-v8.1.0/interfaces/dispute/IAnchorStateRegistry.sol:24`

The Rust ASR binding used by `base-proposer` does not expose `anchorGame()`. It exposes `getAnchorRoot()`, and its `AnchorRoot` return type contains only `{ root, l2_block_number }`:

* `repos/base-v0.8.0-rc.28/crates/proof/contracts/src/anchor_state_registry.rs:16`
* `repos/base-v0.8.0-rc.28/crates/proof/contracts/src/anchor_state_registry.rs:43`
* `repos/base-v0.8.0-rc.28/crates/proof/contracts/src/anchor_state_registry.rs:76`

During full recovery, `recover_latest_state()` seeds the recovered parent with the ASR address:

* `repos/base-v0.8.0-rc.28/crates/proof/proposer/src/pipeline.rs:723`
* `repos/base-v0.8.0-rc.28/crates/proof/proposer/src/pipeline.rs:767`

The forward walk uses that parent when it builds the factory lookup key:

* `repos/base-v0.8.0-rc.28/crates/proof/proposer/src/pipeline.rs:811`
* `repos/base-v0.8.0-rc.28/crates/proof/proposer/src/pipeline.rs:869`

The lookup is parent-sensitive. Rust packs the parent into `extraData`, and the Solidity factory hashes `extraData` into the game UUID:

* `repos/base-v0.8.0-rc.28/crates/proof/contracts/src/dispute_game_factory.rs:170`
* `repos/base-v0.8.0-rc.28/crates/proof/contracts/src/dispute_game_factory.rs:184`
* `repos/contracts-v8.1.0/src/dispute/DisputeGameFactory.sol:262`
* `repos/contracts-v8.1.0/src/dispute/DisputeGameFactory.sol:271`

The recovered parent also reaches submission unchanged:

* `repos/base-v0.8.0-rc.28/crates/proof/proposer/src/pipeline.rs:466`
* `repos/base-v0.8.0-rc.28/crates/proof/proposer/src/pipeline.rs:1171`
* `repos/base-v0.8.0-rc.28/crates/proof/proposer/src/output_proposer.rs:103`
* `repos/base-v0.8.0-rc.28/crates/proof/proposer/src/output_proposer.rs:112`

`AggregateVerifier` interprets `parent = AnchorStateRegistry` as the starting anchor case. For any later game, the parent must be the previous game address:

* `repos/contracts-v8.1.0/src/multiproof/AggregateVerifier.sol:351`
* `repos/contracts-v8.1.0/src/multiproof/AggregateVerifier.sol:363`
* `repos/contracts-v8.1.0/src/multiproof/AggregateVerifier.sol:368`

Failure path:

1. A valid game resolves and advances ASR through `setAnchorState()`.
2. `base-proposer` takes full recovery.
3. Recovery starts from `(currentAnchorRoot, currentAnchorBlock, parent = ASR)`.
4. The next proposal is submitted with `parent = ASR`.
5. `AggregateVerifier.initializeWithInitData()` checks the block number against the original starting anchor and reverts with `UnexpectedBlockNumber`.

Using `parent = anchorGame` for the same next output succeeds.

## Impact Details

The automated proposer submits a valid next output with the wrong parent address, so dispute game creation reverts. After ASR advancement plus full recovery, this path cannot create newer output games.

Withdrawals that depend on outputs newer than the current anchor remain blocked while the proposer stays in this state. Progress resumes when a registered proposer submits the next game with `parent = anchorGame`, or when the recovery logic is patched.

The path uses normal ASR advancement and normal proposer recovery. It does not require an invalid proof, TEE/ZK key compromise, or malicious privileged action.

Impact duration depends on proposer operations. A second registered proposer or manual operator action can reduce the duration. The affected recovery path does not self-correct because it keeps recovering and submitting with the ASR parent.

## References

* Primary asset: `https://github.com/base/base/tree/v0.8.0-rc.28`
* Supporting: `https://github.com/base/contracts/tree/v8.1.0/src/multiproof`

## Link to Proof of Concept

<https://gist.github.com/John-Lussier/582174880e4091f575b2bd7f05293112>

## Proof of Concept

## Context required:

* ASR has advanced to a nonzero `anchorGame`.
* `base-proposer` takes full recovery after cold start, cache reset, or stale cached state.
* The proposer has a valid proof for the next output.

{% stepper %}
{% step %}

## Create and resolve the first game

1. Create `game1` as the first valid game after the starting anchor.
2. Resolve `game1` and call `AnchorStateRegistry.setAnchorState(game1)`.
   {% endstep %}

{% step %}

## Simulate proposer recovery with the wrong parent

3. Simulate `base-proposer` full recovery. The Rust recovery code reads the current anchor root and block but has no `anchorGame()` binding, so it sets `parent = ASR`.
4. Register or model the correct next child under `parent = game1`.
5. Run recovery. It misses that child because it builds the lookup key with `parent = ASR`.
   {% endstep %}

{% step %}

## Submit the next output

6. Submit the next output with `parent = ASR`. Game creation reverts with `UnexpectedBlockNumber`.
7. Submit the same next output with `parent = game1`. Game creation succeeds.
   {% endstep %}
   {% endstepper %}

The runnable PoC is attached as `proposer-recovery-parent-poc.patch`. It adds two local regression tests to the scoped repositories.

Apply from the workspace root:

```bash
git apply submissions/proposer-recovery-parent-poc.patch
```

Run the contract test:

```bash
cd repos/contracts-v8.1.0
forge test --match-test testCannotCreateNextGameFromAdvancedAnchorUsingRegistryAsParent -vv
```

Expected result: 1 passing test. It advances ASR to `game1`, proves that `game2` with `parent = ASR` reverts with `UnexpectedBlockNumber`, and proves that the same `game2` succeeds with `parent = game1`.

Run the proposer recovery test:

```bash
cd ../..
cargo test --manifest-path repos/base-v0.8.0-rc.28/Cargo.toml \
  -p base-proposer test_recovery_cold_start_from_advanced_anchor_loses_anchor_game_parent --lib
```

Expected result: 1 passing test. It sets ASR's root and block to an advanced anchor, registers the next child under the real anchor-game parent, and shows cold-start recovery returns `parent_address = AnchorStateRegistry`.
