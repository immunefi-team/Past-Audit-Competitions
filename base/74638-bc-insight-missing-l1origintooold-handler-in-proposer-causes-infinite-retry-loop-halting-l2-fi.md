# 74638 bc insight missing l1origintooold handler in proposer causes infinite retry loop halting l2 finality and l2 l1 withdrawals

Submitted on Apr 24th 2026 at 00:01:37 UTC by @AlexMyas for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74638
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/base/base/releases/tag/v0.8.0-rc.15>

## Description

### Brief/Intro

The Base Azul proposer enters a permanent infinite retry loop when a proof's `l1_head` expires after \~27h, preventing new dispute games from being created and blocking all L2→L1 withdrawals until a manual restart. This is the sibling of the stall fixed in PR #2221 — `L1OriginTooOld` was not handled alongside `GameAlreadyExists`.

### Vulnerability Details

`l1_head` is baked into the proof at dispatch time and cannot be refreshed (`pipeline.rs:957`, `server.rs:171`). After 8,191 L1 blocks (\~27h), the contract rejects it:

```solidity
if (blockAge > EIP2935_WINDOW) revert L1OriginTooOld(); // AggregateVerifier.sol:963
```

`estimate_gas` catches this before broadcast. The error hits `classify_tx_manager_error()` (`output_proposer.rs:25`), which only special-cases `GameAlreadyExists` — `L1OriginTooOld` falls through to `SubmitAction::Failed` (`pipeline.rs:1196`).

The `Failed` handler re-queues the same stale proof:

```rust
// pipeline.rs:579
state.proved.insert(target_block, proof); // stale proof re-inserted, loop repeats
```

No escape exists: `prune_stale` only removes entries ≤ `recovered_block`; `dispatch_proofs` skips blocks in `state.proved`; no retry cap on submission failures.

**Fix:** route `L1OriginTooOld` to `SubmitAction::Discard` — drops the stale proof so a fresh one is requested on the next tick.

```rust
} else if e.is_l1_origin_too_old() {
    Err(SubmitAction::Discard(e))
}
```

### Impact Details

* No new dispute games → L2 finality stalls, all L2→L1 withdrawals blocked.
* L2 sequencer unaffected; only finality and bridges halt.
* Recovery: operator restart (minutes); proof regenerates automatically with fresh `l1_head`.
* Trigger: prover or L1 RPC outage lasting >27h while a proof is in-flight.

### References

* `crates/proof/proposer/src/pipeline.rs` L579, L1196
* `crates/proof/proposer/src/output_proposer.rs` L25
* `src/multiproof/AggregateVerifier.sol` L963
* PR #2221: `fix(proposer): break GameAlreadyExists stall` — analogous fix

### Proof of Concept

1. Proposer dispatches proof for L2 block N; `l1_head` is baked in at this moment (`pipeline.rs:957`).
2. Prover/L1 RPC outage lasts >8,191 L1 blocks (\~27h). Proof lands in `state.proved`.
3. `try_submit` → `estimate_gas` → reverts `L1OriginTooOld` (`AggregateVerifier.sol:963`).
4. `classify_tx_manager_error()` has no case for it → `SubmitAction::Failed` → stale proof re-inserted into `state.proved` (`pipeline.rs:586`).
5. Next tick (12s): same proof, same revert. Loop repeats indefinitely.
6. `prune_stale` never clears block N; `dispatch_proofs` never re-proves it. Proposer halted until manual restart.
