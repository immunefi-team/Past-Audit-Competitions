# 90343 bc medium txpool skips maintain on every finality event watched txs never see finalized and are falsely timed out after 512 blocks despite being finalized

**Submitted on Aug 24th 2026 at 15:24:00 UTC by @Jornason for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #90343
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief/Intro

Quantus's vendored transaction pool — the SingleState implementation that is the default for every node (`client/cli/src/params/transaction_pool_params.rs:67`) — contains a maintenance guard, introduced publicly in PR #636, that skips pool maintenance whenever an incoming chain event is more than 20 blocks away from the pool's current best block (`client/transaction-pool/src/common/enactment_state.rs:31,107-123`). Quantus finalization deliberately lags the best block by exactly `MaxReorgDepth = 100` (`runtime/src/configs/mod.rs:161`, `client/consensus/qpow/src/chain_management.rs:203-222`), so every Finalized chain event arrives 100 blocks behind best and the guard discards every single one of them.

The pool therefore never processes finality: a transaction submitted through the standard `author_submitAndWatchExtrinsic` RPC goes `broadcast -> inBlock` and then never receives `Finalized`, even though the chain finalizes its inclusion block about 100 blocks later. Worse, after \~512 further transaction-bearing blocks the pool's finality watch list evicts the transaction and actively pushes `FinalityTimeout` — the protocol-level "transaction dropped, will not finalize" terminal signal — carrying the hash of a block that has in reality been finalized for roughly 412 blocks.

Measured on an unmodified node built from the frozen competition commit: a watched transfer included at height 4 received `finalityTimeout` at best height 516 while the node's own finalized height was 416. On the default configuration this deterministically breaks the transaction-lifecycle contract for every integrator that relies on watched-transaction RPC (wallets, exchanges, bridges, explorers, payment processors) with no attacker action required at all: confirmation flows never complete, and timeout-driven resubmission logic can execute the same payment a second time. This maps to the in-scope Medium impact Griefing.

### Vulnerability Details

All file references are to `audit-comp-quantus-chain` at the frozen commit `3b243b870a5442a6e5f443056f67bbba87d472f5`.

{% stepper %}
{% step %}

## The guard

`client/transaction-pool/src/common/enactment_state.rs` is Quantus's own addition to the vendored pool (upstream Substrate has no such guard; it appears in this repository with PR #636).

Line 31 sets `SKIP_MAINTENANCE_THRESHOLD = 20`. Lines 107-123 compute `skip_maintenance` from the absolute height distance between the incoming event's block and the pool's `recent_best_block`, skipping with `EnactmentAction::Skip` — and a debug log `"skip maintain: tree_route would be too long"` — when the distance exceeds 20, or (`_ => true` arm) whenever either height is unknown.

The code comment states the intent: cheaply reject deep forward jumps during full sync and deep backward reorgs. `force_update` (lines 192-196) records only `recent_finalized_block` on Finalized events, so `recent_best_block` keeps tracking `NewBestBlock` — meaning the guard compares every finalized block against the live best block height.
{% endstep %}

{% step %}

## Quantus finality is structurally 100 blocks behind best

`runtime/src/configs/mod.rs:161` sets `MaxReorgDepth = ConstU32<100>`; `client/consensus/qpow/src/chain_management.rs:203-222` reads that value at runtime (`finalize_depth = max_reorg_depth`) and finalizes exactly `best_number - 100` after every block import.

Consequently every `Finalized(#N)` event is delivered to the pool when the best block is approximately `#N+100`, so the height delta seen by the guard is \~100, which exceeds 20 every time.

The skip therefore fires on 100% of finality events in normal operation — not merely in the deep-reorg/full-sync corner the guard was designed for.
{% endstep %}

{% step %}

## Consequence A — Finalized is never delivered

Pool maintenance on finality is the only path that retires pruned transactions and fires watcher finalized statuses. With every finality event skipped, a watched transaction stalls at `inBlock` indefinitely, no matter how many blocks are finalized afterward.
{% endstep %}

{% step %}

## Consequence B — after the 512-entry bound, the pool actively lies

`client/transaction-pool/src/graph/listener.rs:85` sets `MAX_FINALITY_WATCHERS = 512`; `finality_watchers` maps each block hash to the watched transactions pruned in that block (lines 247-248), gaining at most one entry per block.

When a new entry pushes the map past 512, the oldest block's transactions are evicted and each receives `finality_timeout(evicted_block_hash)` (lines 255-263) — the terminal signal integrators interpret as "the transaction was dropped and will not finalize" (upstream `retracted()` at lines 267-269 removes entries on reorgs, which is the only reason the count can deviate by one).

By the time the timeout fires, the transaction's inclusion block has been finalized for \~412 blocks (512-entry lag minus the 100-block finality lag).
{% endstep %}

{% step %}

## Measured confirmation on the frozen commit

Main run (full artifacts in the linked Gist):

* Victim transfer included at height 4, index 1, hash `0x7b3bf1154e2801f043285916b420370fae0de9e8a9280c526b6f86301f4d375d`
* Watched stream sequence was exactly `broadcast -> inBlock -> finalityTimeout`
* `finalized` never observed
* `finalityTimeout` fired at best height 516 carrying that same inclusion block hash
* The node's finalized height at that moment was 416
* Finality lag was exactly 100
* 511 of the 512 newer blocks were transaction-bearing
* 0 filler resubmits
* 0 submit errors
* After the terminal status, `author_unwatchExtrinsic` returned `false`, i.e. the false timeout is also the last message the integrator ever receives

All six machine-checked assertions (A1-A6) in `stage2_summary.json` pass with verdict `CONFIRMED_FALSE_FINALITY_TIMEOUT`.

A corroboration run with `RUST_LOG=txpool=debug` captured exactly one `"skip maintain: tree_route would be too long"` debug line per finality event (12 lines for finalized heights 1-12), directly confirming the guard is the cause; the same log shows import-path maintenance (`Starting pruning of block`) running normally, so only the finality path is suppressed.
{% endstep %}

{% step %}

## Duplicate boundary

Checked 2026-08-24.

The guard was introduced publicly by PR #636 ("V12 dilithium", merged 2026-08-03) as a maintenance-cost mitigation; this report demonstrates that under Quantus's own finality policy the guard misfires on 100% of finality events — a defect introduced by a public change, which the program's Code Freeze Assurance treats as a new, valid bug.

Public commit history for both `enactment_state.rs` and `listener.rs` shows no commit after PR #636, and a public issue/PR search (`finality`, `txpool`, `watcher`, `FinalityTimeout`) shows no disclosure of this behavior.

The 512-entry eviction and `finality_timeout` mechanics themselves are upstream Substrate code; the in-scope root cause is Quantus's own guard interacting with Quantus's own `MaxReorgDepth = 100` finality policy, both inside the frozen competition repository. Private competition reports cannot be ruled out.
{% endstep %}
{% endstepper %}

### Impact Details

Selected impact: Griefing (Medium). The watched-transaction RPC stream is the chain's transaction-lifecycle contract with the outside world; `author_submitAndWatchExtrinsic` is how wallets, exchanges, bridges, block explorers, and payment processors learn that a transaction is final. On the default node configuration this contract is broken for 100% of watched traffic, deterministically, without any attacker:

* **Confirmations never complete.** No watched transaction ever emits `Finalized`. Any integration that waits for finality before acting (crediting a deposit, releasing goods, marking a withdrawal complete) hangs indefinitely or falls back to its own timeout logic.
* **The chain then actively reports the opposite of the truth.** About 512 transaction-bearing blocks after inclusion — \~102 minutes at the 12-second target block time (`runtime/src/lib.rs:87`, `TARGET_BLOCK_TIME_MS = 12_000`) — the pool emits `FinalityTimeout`, the terminal status that means "dropped, will not finalize," for a transaction that became final \~82 minutes earlier. Standard integrator playbooks respond to this status by rebuilding and resubmitting the payment under a new nonce. Because the original transaction is finalized (not dropped), both transactions execute: the same withdrawal or payment is carried out twice. An unscrupulous counterparty can additionally exploit this deterministically: any recipient of a payment knows the sender's wallet will be told the payment "timed out" roughly 102 minutes after inclusion and may be induced to send it again.
* **No mitigation on the integrator side is reliable.** After the false timeout the subscription is already consumed (`author_unwatchExtrinsic` returns `false`), so the false terminal status is the final word the integrator receives from the node.

{% hint style="info" %}
**Boundaries of the claim:** on-chain execution is unaffected — transactions are included and finalized normally; there is no crash, halt, consensus failure, or protocol-layer fund loss, so no High/Critical impact is claimed; the duplicate-payment exposure is at the integrator layer (a user or service acting on the false signal), not a direct protocol-layer theft; no resource-consumption impact is claimed. The defect fires by itself on default settings — no attack traffic, malformed input, or special node configuration is required.
{% endhint %}

### References

* `client/cli/src/params/transaction_pool_params.rs:67` — SingleState is the default transaction pool
* `client/transaction-pool/src/common/enactment_state.rs:31` (`SKIP_MAINTENANCE_THRESHOLD = 20`), `:107-123` (height-delta skip guard), `:139-143` (route-length bound), `:192-196` (`force_update` keeps `recent_best_block` on finality events)
* `client/transaction-pool/src/graph/listener.rs:85` (`MAX_FINALITY_WATCHERS = 512`), `:247-263` (per-block watcher entries and eviction firing `finality_timeout` with the evicted block hash), `:267-269` (retracted removal)
* `runtime/src/configs/mod.rs:161` (`MaxReorgDepth = 100`), `runtime/src/lib.rs:87` (`TARGET_BLOCK_TIME_MS = 12_000`)
* `client/consensus/qpow/src/chain_management.rs:203-222` (finalizes exactly `best - max_reorg_depth` after each import)
* `pallets/qpow/src/lib.rs:92-93,373` (chain-spec difficulty override hook; 131,072 floor) — harness note only, not part of the defect
* Official PRs in the competition repository: #636 (publicly introduced the guard whose interaction with Quantus finality causes this defect; merged 2026-08-03), #595 (vendored pool import)

## Link to Proof of Concept

<https://gist.github.com/Jornason/75a2e6dd5c3ec7d74d222165db7e0de6>

## Proof of Concept

For Audit Competitions, step-by-step explanations are provided below; the complete reproducer source is in the linked secret Gist. Everything runs on loopback against a locally built node; no public network, bootnode, telemetry, or project infrastructure is contacted.

Host used: macOS on Apple Silicon, zsh, python 3 with the websockets package, rustc stable; the node is a debug build of the frozen commit (build profile does not affect the pool/finality logic under test).

### Harness disclosure

{% hint style="info" %}
The defect is driven by block count, not wall-clock difficulty: the guard's trigger is finality lag (100) > threshold (20), and the eviction trigger is 512 block-hash entries.
{% endhint %}

To make the required 520+ blocks practical on a workstation, the run uses a chain spec exported from the stock `--dev` spec via `quantus-node build-spec --dev` with exactly one field changed: `genesis.runtimeGenesis.patch.qPoW.initialDifficulty` lowered from `0x3d0900` (4,000,000) to `0x20000` (131,072, the pallet's own floor — `pallets/qpow/src/lib.rs:373`, `get_min_difficulty`).

The override hook is explicitly supported by the pallet ("This allows chain-spec overrides of initial difficulty", `pallets/qpow/src/lib.rs:92-93`), and the per-block retarget is bounded to +/-1/2048, so difficulty can drift at most \~1.35x over the run.

The binary is the unmodified stock build; the pool, guard, finality policy, and watcher code paths are untouched, and the genesis hash changes only because the difficulty value lives in genesis storage. A first run at the stock difficulty was also performed and abandoned only because \~120 s/block on the shared host made 512 blocks exceed any practical wall clock; its partial trace showed identical behavior.

Launch flags differ from `--dev` only where the CLI requires it outside dev mode:

* `--validator` — mining is enabled only for authorities (`node/src/service.rs:790`)
* `--rewards-inner-hash` — mandatory outside `--dev` (`node/src/command.rs:657`; a throwaway local address — the rewards destination is irrelevant to the mechanism)
* `--unsafe-force-node-key-generation` — the node refuses to start as a non-dev authority without a persisted Dilithium network key (`client/cli/src/params/node_key_params.rs:118-123`)

The transaction pool type is the default SingleState (confirmed in the startup log line `"Creating transaction pool txpool_type=SingleState"`).

{% stepper %}
{% step %}

## Build the frozen node and the transaction generator

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-chain
cd audit-comp-quantus-chain
git checkout 3b243b870a5442a6e5f443056f67bbba87d472f5
cp /path/to/gist/gen_transfers.rs runtime/examples/gen_transfers.rs
cargo build -p quantus-node --bin quantus-node
cargo build -p quantus-runtime --example gen_transfers
```

`gen_transfers.rs` builds ordinary `Balances::transfer_keep_alive` extrinsics from the well-known dev account `crystal_alice()` to `dilithium_bob()`, signed with Dilithium87 exactly the way the node's own transaction extensions expect, and prints them as JSON lines (`{"nonce":N,"bytes":B,"hex":"0x.."}`).

Each generated transaction is 7,296 bytes. Nothing about the transactions is special — that is the point: the defect fires on completely ordinary traffic.
{% endstep %}

{% step %}

## Run the main scenario

```bash
export WORKDIR=/tmp/qstage2 QUANTUS_REPO=$PWD
export REWARDS_INNER=0x<throwaway inner hash from: quantus-node key quantus --scheme wormhole>
zsh run_stage2.zsh
```

The runner exports the stock dev spec, patches the single difficulty field, starts one victim node plus six mining helpers on loopback (default pool, default finality), waits for block production, generates 620 sequential-nonce transactions, and launches `stage2_driver.py`.

The driver submits nonce 0 through `author_submitAndWatchExtrinsic` and keeps the subscription open; once it is included at height H, it submits exactly one ordinary transfer per new head so that each block adds one finality-watcher entry; it records every victim status, every `(best, finalized)` pair, and per-block occupancy into `trace.jsonl`, and finishes with a machine-checked `summary.json`.
{% endstep %}

{% step %}

## Expected results

| Measurement                             | Result                                                                      |
| --------------------------------------- | --------------------------------------------------------------------------- |
| Victim inclusion                        | height 4, index 1, hash `0x7b3bf1...4d375d`                                 |
| Watched stream sequence                 | `broadcast -> inBlock -> finalityTimeout` (`finalized` never observed)      |
| `finalityTimeout` fired at              | best = 516 (511 newer transaction-bearing blocks recorded; 512-entry bound) |
| Hash carried by the timeout             | the inclusion block hash (assertion A4)                                     |
| Node finalized height at that moment    | 416 — the inclusion block had been finalized for \~412 blocks               |
| Finality lag at that moment             | exactly 100 (= `MaxReorgDepth`)                                             |
| Filler resubmits / submit errors        | 0 / 0                                                                       |
| Post-terminal `author_unwatchExtrinsic` | false (subscription already consumed)                                       |
| Verdict                                 | `CONFIRMED_FALSE_FINALITY_TIMEOUT` (A1-A6 pass, see `stage2_summary.json`)  |
| Wall time                               | \~54 minutes for 516 blocks (\~6.3 s/block average)                         |
| {% endstep %}                           |                                                                             |

{% step %}

## Confirm the cause

```bash
zsh corrob_guardlog.zsh
```

This starts the same network with `RUST_LOG=txpool=debug` on the victim and waits for `best >= 108`.

Observed: exactly one `"skip maintain: tree_route would be too long"` debug line per finality event (12 lines for finalized heights 1-12; excerpt in `corrob_skip_maintain_excerpt.txt`), while import-path maintenance (`txpool: Starting pruning of block`) keeps running normally — proving that only the finality path is suppressed, and by the guard.
{% endstep %}
{% endstepper %}

### Suggested fix

Make the guard finality-aware instead of best-relative: for Finalized events, either exempt them from the height-delta skip (the actual work is already bounded by the tree-route length check at `enactment_state.rs:139-143`), or measure the delta against `recent_finalized_block` rather than `recent_best_block`, or raise the threshold above `MaxReorgDepth` with margin.

Add a regression test asserting that a transaction submitted via `author_submitAndWatchExtrinsic` receives `finalized` within `MaxReorgDepth + k` blocks on a default-config node.

## Gist

<https://gist.github.com/Jornason/75a2e6dd5c3ec7d74d222165db7e0de6> (secret; verified public=false, 8 files: README.md, stage2\_driver.py, run\_stage2.zsh, corrob\_guardlog.zsh, gen\_transfers.rs, stage2\_summary.json, stage2\_trace.jsonl, corrob\_skip\_maintain\_excerpt.txt)
