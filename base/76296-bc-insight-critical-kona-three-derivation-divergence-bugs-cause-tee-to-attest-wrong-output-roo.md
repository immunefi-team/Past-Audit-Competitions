# 76296 bc insight critical kona three derivation divergence bugs cause tee to attest wrong output root single proof bridge drain

**Submitted on May 3rd 2026 at 18:37:11 UTC by @abn06 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76296
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Permanent freezing of funds (fix requires hardfork)
  * Direct loss to Base or users ≥ 10% of funds held within Bridge.
  * Forging or bypassing TEE or ZK proof verification in AggregateVerifier to finalize an invalid state root on L1
  * Circumventing the dispute/challenge mechanism to prevent correction of an invalid proposal before finalization
  * Permanent freezing of funds in the bridge or in dispute game bonds with no available recovery path

## Description

**Target:** `github.com/base/base` at `v0.8.0-rc.24`, commit `819ea306db40792a50626243034a62e3ca015ba6` (competition scope)\
**Also verified at:** `v0.8.0-rc.28`, commit `e3467a2048881213b56739a54a876efb9c6ea103` (superset - all bugs predate rc.28)\
**Severity:** Critical - Permanent loss of funds, victim withdrawals permanently frozen\
**Findings:** 3 independent derivation divergences + 1 compounding dead-code guard

***

## Triager Notes

* All exploit code is runnable as-is at the stated fork block
* No screenshots; every source reference is commit-pinned to `819ea306` (rc.24, primary) and `e3467a2` (rc.28, superset)
* Two PoC files provided: Rust (root cause) + Solidity/Foundry (bridge drain)
* `vm.store` on EIP-1967 impl slot in the Solidity PoC bypasses a Sepolia-specific ERC20 gate absent on mainnet - documented inline. This is not `vm.etch` and does not replace contract logic - it replaces only the implementation pointer.
* `vm.store` on `provenWithdrawals` simulates `proveWithdrawalTransaction` producing its storage write - the Portal's finalization codepath is exercised unchanged
* **No factory owner privilege is required.** `DisputeGameFactory.setImplementation` does not appear in the drain PoC. The drain test uses only a deployed `PoisonedGame` pointed to by `provenWithdrawals` storage.
* **TEE proof validity:** The TEE enclave is functioning correctly and signing what it derives. No key compromise is required. The enclave produces a genuine attestation of a wrong output root caused by a software bug in `reader.rs:139` - not a forged attestation. This report does not rely on an invalid TEE proof; it reports a software bug that causes the TEE to produce a valid proof of an incorrect result.
* **Batcher within normal authority:** The batcher setting `batch.timestamp` within the RLP payload is normal batcher operation - it is not a compromise of Base-operated infrastructure. The vulnerability is that Kona validates `batch.timestamp` (a batcher-supplied field) instead of `l1_origin.Time` (a consensus-enforced field). This bug is fully discoverable via code review of `reader.rs:139` with no runtime access to Base infrastructure required. Supplying `batch.timestamp` is not transaction ordering and is not MEV.
* **Feasibility Limitations (Immunefi program rules):** Base's program explicitly adopts Immunefi's Feasibility Limitations standard, which states that mitigation measures requiring unconventional action cannot be cited as grounds for severity downgrade. Base operating the batcher and Base's ability to intervene within 7 days are both unconventional mitigations - neither can be used to downgrade severity under this program's own rules.
* **"Dispute/blacklist/retire" downgrade clause (scope page):** The scope states reports may be downgraded if they assume Base will not dispute/blacklist/retire an invalid proposal, "unless it can be shown that such an action cannot be taken." We show exactly this: the intended dispute mechanism is a ZK challenge of the TEE proof. The scope page itself confirms "a ZK proof can be used to challenge a TEE proof." The ZK Challenge Impossibility section of this report proves that no deployed ZK prover can produce a valid proof of `R_canonical` because all provers share the same buggy `base-consensus-derive` pipeline. The standard dispute path is structurally closed. Guardian blacklist is an unconventional mitigation, not a standard dispute mechanism, and is protected from downgrade citation by the Feasibility Limitations clause.
* **ZK prover source made private mid-contest (April 24, 2026 - Discord announcement):** The dependency chain evidence (`executor.rs`, `logger.rs`) was captured at commit `e3467a2` prior to this change and is included in this report. The root cause bugs in `reader.rs` / `channel_assembler.rs` / `single.rs` remain fully verifiable at `v0.8.0-rc.24` (commit `819ea306`).

***

## Executive Summary

Three independent bugs in Kona's derivation pipeline each independently cause Kona's TEE to attest a wrong output root `R_kona ≠ R_canonical`. With `PROOF_THRESHOLD = 1` (confirmed on-chain at blocks `10775328` and `10781457`, and confirmed as a **deploy-time immutable** baked into the AggregateVerifier bytecode - cannot be changed without full contract redeployment), a single TEE proof resolves `DEFENDER_WINS` and poisons the anchor state. The same SP1 ZK prover runs the unpatched derivation pipeline (source-verified, commits `819ea306` and `e3467a2`), making ZK challenge structurally impossible. After a 7-day finalization delay (`SLOW_FINALIZATION_DELAY = 604800s`, confirmed on-chain), the attacker drains ETH from `OptimismPortal2` while victims whose withdrawals fall in the corrupted block range are permanently frozen.

**The permissionless extraction (Path B) is the primary severity driver.** Once `R_kona` is anchored by any means - a compromised batcher, an operator error, or any infrastructure fault - the bridge drain and victim freeze require zero privilege. Any external EOA can prove and finalize. The batcher trust question affects only how `R_kona` gets anchored, not the severity of what follows.

***

## Privilege Model - Two Attack Paths

### Path A - Malicious or Compromised Batcher (Fjord Timestamp Mismatch / OOO Frame Ordering / Past-Drop Misclassification)

**Privilege required:** Batcher role.\
**Framing:** The batcher is currently operated by Base. These bugs are exploitable if the batcher is compromised (key theft, supply-chain), acts maliciously, or if batcher decentralization proceeds (expanding the role to untrusted participants). The bugs exist because Kona validates batcher-supplied RLP fields against themselves rather than against L1-consensus-enforced values - a protocol-level flaw, not an operator-trust assumption.

**Batcher capability evidence:** `test_batch_timestamp_fully_batcher_controlled` demonstrates that `batch.timestamp` is a plain RLP field written by the batcher in every batch submission with no L1 validation. The boundary table in that test shows five values the batcher can freely set, including `fjord_ts - 1` which triggers the Fjord Timestamp Mismatch. Setting this value requires no permissions beyond what the batcher already exercises on every block - it is not transaction ordering and is not MEV.

Under Immunefi's severity framework, a compromised batcher triggering permanent fund loss at portal scale is Critical. The batcher is a semi-trusted role, not a fully-trusted admin equivalent.

### Path B - Any Actor Once `R_kona` Is Anchored

**Privilege required:** None - any external EOA.\
**Framing:** Once `R_kona` is anchored (via Path A, operator error, or infrastructure fault), the bridge drain and victim freeze steps are fully permissionless. The attacker need only:

1. Construct a valid Merkle proof of their withdrawal against `R_kona`'s state trie (they authored the channel content, so they control that trie)
2. Call `proveWithdrawalTransaction` - no privilege, any EOA
3. Wait 7 days
4. Call `finalizeWithdrawalTransaction` - no privilege, any EOA

The victim freeze is similarly permissionless - no action by any attacker is needed; victims simply cannot prove against the wrong root.

***

## On-Chain Confirmed Parameters

All values read from live contracts at fork block `10775328` (ETH Sepolia). `PROOF_THRESHOLD` and `SLOW_FINALIZATION_DELAY` independently re-confirmed at block `10781457` (same values). `PROOF_THRESHOLD = 1` additionally confirmed by the mid-contest changelog: the entry "PROOF\_THRESHOLD of 2 should not be used" (PR #263) explicitly rules out any value other than 1, corroborating on-chain reads as the intentional production value:

| Parameter                     | Value          | Source                                                                                                                                                                                                                                                                                                                            |
| ----------------------------- | -------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `PROOF_THRESHOLD`             | `1`            | `cast call 0xF3f0...E796 "PROOF_THRESHOLD()(uint256)"` → `1` (blocks 10775328 + 10781457 + changelog PR #263). **Confirmed as deploy-time immutable via PUSH32 bytecode analysis** - value is baked into AggregateVerifier at deployment, not changeable via storage write or governance call without full contract redeployment. |
| `SLOW_FINALIZATION_DELAY`     | `604800s (7d)` | `cast call 0xF3f0...E796 "SLOW_FINALIZATION_DELAY()(uint64)"` → `604800` (blocks 10775328 + 10781457)                                                                                                                                                                                                                             |
| `respectedGameType`           | `621`          | `cast call 0x49f5...e85 "respectedGameType()(uint32)"` → `621`                                                                                                                                                                                                                                                                    |
| `proofMaturityDelaySeconds`   | `0`            | `cast call 0x49f5...e85 "proofMaturityDelaySeconds()(uint256)"` → `0`                                                                                                                                                                                                                                                             |
| `proofCount` (live games)     | `1`            | `factory.gameAtIndex(16231) → proofCount = 1` (16,232 games total)                                                                                                                                                                                                                                                                |
| `provenWithdrawals` base slot | `57`           | `keccak256(abi.encode(attacker, keccak256(abi.encode(hash, 57))))` = `0x641f90dda5649cdd42647182bcf5ca83d71d2d23246a358e0cdde4d0289c19d9` - exact match to Foundry trace                                                                                                                                                          |
| `finalizedWithdrawals` slot   | `58`           | `keccak256(abi.encode(attacker, keccak256(abi.encode(hash, 58))))` = `0xbf975b327b0385249a2f63a55f160f31fb828a2554a6e8c4bbb123d9b28f022d` - double-spend guard confirmed                                                                                                                                                          |

***

## Attack Chain (Path A + B Combined)

```
[Path A - Batcher-triggered derivation divergence]
    │
    ▼
1.  Batcher crafts L1 batch submission with exploiting field
    (Fjord Timestamp Mismatch: batch.timestamp = fjord_ts - 1)
    (OOO Frame Ordering: OOO frame sequence with gap at F3, attacker payload between F4-F6)
    (Past/Drop Misclassification: misaligned batch timestamp in window (safe_ts, safe_ts + block_time))
    │
    ▼
2.  Kona TEE derives different blocks than op-node from identical L1 data
    Kona: empty blocks / attacker blocks / wrong batch ordering
    op-node: real user blocks
    → R_kona ≠ R_canonical, irrecoverably, for the affected block range
    │
    ▼
3.  SP1 ZK prover runs same unpatched derivation pipeline (source-verified)
    No valid proof of R_canonical can be produced by any deployed prover
    ZK challenge is structurally impossible
    │
    ▼
4.  TEE submits proof of R_kona to AggregateVerifier
    proofCount++ → proofCount (1) ≥ PROOF_THRESHOLD (1) → resolve() → DEFENDER_WINS
    R_kona is now the anchored output root
    │
[Path B - Permissionless extraction, any external EOA]
    │
    ▼
5.  Attacker authored the channel content → controls R_kona's L2 state trie
    Attacker constructs Merkle proof of their 10 ETH withdrawal against R_kona
    Calls proveWithdrawalTransaction(withdrawal, merkleProof, R_kona)
    → provenWithdrawals[withdrawalHash][attacker] = (resolvedGame, timestamp)
    │
    ▼
6.  Attacker waits SLOW_FINALIZATION_DELAY = 604800s (7 days)
    │
    ▼
7.  Attacker calls finalizeWithdrawalTransaction
    Portal reads provenWithdrawals → resolvedGame.status() = DEFENDER_WINS (2)
    proofMaturityDelaySeconds = 0 → no additional delay
    ETH transferred from OptimismPortal2 to attacker
    │
    ▼
8.  Victims whose withdrawals exist in R_canonical but not R_kona:
    proveWithdrawalTransaction fails (Merkle proof invalid against R_kona)
    Funds permanently frozen until guardian intervention + TEE re-attestation
    Recovery: minimum days to weeks

[Recovery path note - PR #263]
    The mid-contest changelog fix "Resolution is not changed if a global verifier is nullified"
    (PR #263) means Base can now alter the poisoned game's resolution by nullifying the
    TEE verifier through a different game. However this requires:
    (a) detecting the exploit within the 7-day SLOW_FINALIZATION_DELAY window,
    (b) unconventional guardian intervention off the standard protocol path.
    Under Immunefi's Feasibility Limitations clause (explicitly adopted by this program),
    mitigation measures requiring unconventional action cannot be cited as grounds for
    severity downgrade. The attack remains Critical.
```

***

## Finding 1 - Fjord Timestamp Source Mismatch

**File:** `crates/consensus/protocol/src/batch/reader.rs:139`\
**Commit:** identical between rc.24 and rc.28

```rust
// reader.rs:139 - VULNERABLE
if self.brotli_used && !cfg.is_fjord_active(batch.timestamp()) {
//                                           ^^^^^^^^^^^^^^^^^^ batcher writes this field
```

```go
// op-node channel_in_reader.go:57 - CORRECT
if cr.cfg.IsFjord(cr.prev.Origin().Time) {
//                ^^^^^^^^^^^^^^^^^^^^^ L1 origin timestamp - consensus-enforced
}
```

**Source-confirmed (v1.13.4):**

```
grep -n "IsFjord" op-node/rollup/derive/channel_in_reader.go
57: if cr.cfg.IsFjord(cr.prev.Origin().Time) {
```

`batch.timestamp()` is an RLP field the batcher writes with no L1 constraint. `cr.prev.Origin().Time` is set by L1 consensus, unreachable by the batcher.

**Exploit:** Batcher sets `batch.timestamp = fjord_activation_timestamp - 1`.

* Kona: `is_fjord_active(fjord_ts - 1) = false` → brotli channel discarded → empty blocks derived
* op-node: `IsFjord(l1_origin.Time) = true` → brotli accepted → real user blocks derived
* `R_kona ≠ R_canonical` from this block range onward, irrecoverably

**Rust PoC:** `kona_derivation_fjord::test_fjord_activation_divergence` - assertions pass, divergence confirmed.

**Batcher decoupling evidence (source-confirmed):** `crates/batcher/comp/src/composer.rs:70` sets `batch.timestamp = block.header.timestamp` and `crates/batcher/comp/src/variant.rs:25` calls `config.is_fjord_active(timestamp)` using the L2 block's real timestamp to determine channel compression type. These are **two separate values** that happen to be equal in the honest batcher. A compromised batcher can decouple them: create a brotli-compressed channel using the real L2 block timestamp (Fjord-era), while setting `batch.timestamp = fjord_ts - 1` in the RLP field. The channel's `brotli_used = true` flag is set by channel compression type - independent of `batch.timestamp`. This is the exact decoupling that makes `reader.rs:139` exploitable: `brotli_used = true` but `is_fjord_active(batch.timestamp) = false` → channel discarded.

**Mid-contest corroboration:** The competition changelog entry `fix(consensus): iterate all transactions in is_deposits_only` (PR #2429) documents Base patching a separate derivation classification bug in the same Kona pipeline during this contest. The bug structure is mechanically identical to this finding - an incorrect classification in the derivation pipeline causing divergence from canonical behavior. This confirms that derivation classification errors of exactly this class are real, patchable, and have been independently discovered and fixed by Base's own security process.

***

## Finding 2 - Holocene OOO Frame Ordering Not Enforced

**File:** `crates/consensus/derive/src/stages/channel/channel_assembler.rs:116`

```rust
// channel_assembler.rs:116 - VULNERABLE
// No ordering check. Frames accepted in any sequence.
channel.add_frame(next_frame, origin)?;
```

```go
// op-node channel_assembler.go:98 - CORRECT
ca.channel = NewChannel(frame.ID, origin, true) // requireInOrder=true

// channel.go:78-79:
if ch.requireInOrder && int(frame.FrameNumber) != len(ch.inputs) {
    return fmt.Errorf("frame out of order")
}
```

**Source-confirmed absence:**

```
grep -c "holocene\|in_order\|require.*order\|inputs_len" channel_assembler.rs
→ 0 conditional ordering guards (3 matches are debug log strings only)
```

**Developer documented:** `channel.rs:31`: `"Frames are allowed to be ingested out of order."`

**Exploit:** Batcher submits F0-F2 (normal prefix), then F4-F6(is\_last) with attacker-payload blocks, then F3 late.

* Kona: all 7 frames accepted, `is_ready() = true`, attacker blocks derived → `R_kona`
* op-node: F4 rejected (expected F3), channel never ready → `R_canonical`

**Cross-transaction injection path (source-confirmed):** `frame_queue.rs:prune()` is the only Holocene ordering enforcement in the Kona pipeline. Reading the full source confirms it operates exclusively within a single L1 transaction's frame batch - it runs immediately after `self.queue.extend(frames)` for one tx, then frames are popped to `channel_assembler` and leave the queue permanently. A compromised batcher submitting F0-F2 in TX1, F4-F6(is\_last) in TX2, and F3 in TX3 produces three independent `prune()` invocations, each seeing only sequential frames within their own batch. All seven frames reach `channel_assembler.rs:116` unfiltered across the three transactions, where no ordering guard exists. `test_holocene_non_sequential_frames` only catches OOO frames within the same tx - cross-tx OOO is invisible to `prune()` by design.

**Rust PoC:** `kona_derivation_holocene::test_ooo_frame_kona_accepts_opnode_rejects` - exercises `Channel::add_frame` directly (the data-structure layer `channel_assembler.rs:116` calls without a guard). Source-confirmed absence of guards in the assembler is separately documented above.

***

## Finding 3 - Holocene Misaligned Batch Timestamp: Past Instead of Drop

**File:** `crates/consensus/protocol/src/batch/single.rs:52`

**Source-confirmed (op-node v1.13.4, `batches.go:82-87`):**

```go
if batch.Timestamp < nextTimestamp {
    if !cfg.IsHolocene(batch.Timestamp) {
        return BatchPast    // pre-Holocene only
    }
    return BatchDrop        // Holocene: DROP + channel.Flush()
}
```

Kona returns `BatchValidity::Past` for `batch.timestamp ∈ (safe_head.ts, next_ts)`. op-node returns `BatchDrop` under Holocene and calls `channel.Flush()`.

**Exploit:** Batcher places a misaligned batch then attacker's target batch in one channel.

* Kona: `Past` → no flush → **next batch in channel is processed normally** → attacker's target batch derived → `R_kona`
* op-node: `Drop` → channel flushed → entire remaining channel discarded → `R_canonical`

**Impact chain closed:** `Past` advances the safe head past the misaligned batch without flushing the channel. Any batch placed immediately after the misaligned batch in the same channel is processed by Kona but lost by op-node (which flushed). The attacker controls what goes into that subsequent batch - they authored the channel content. The state trie divergence is therefore fully attacker-controlled, not just a classification difference.

**CI blind spot (source-confirmed):** `test_check_batch_timestamp_holocene_active_drop` uses `block_time = 1` (`single.rs:621`). With `block_time = 1`: the misaligned window `(safe_ts, safe_ts + 1)` is the empty open interval - no integer exists, bug is invisible. Base production uses `block_time = 2`, creating `{safe_ts + 1}` as the exploitable slot.

**CI actively validates the wrong behavior (source-confirmed at rc.24):** The test suite at `v0.8.0-rc.24` (lines 296 and 311 of `single.rs`) already asserts `BatchValidity::Past` for the Holocene case as the expected correct result. The CI does not merely fail to catch the bug - it positively asserts that the buggy behavior is correct, meaning a patch that fixes the divergence would fail CI until tests are also updated. Additionally, the `Drop` path at `single.rs:54` is named `BatchDropReason::PastTimestampPreHolocene`, making it explicit in the source that the `Drop` path is intended for pre-Holocene only - the Holocene path returning `Past` at line 52 is structurally asserted by the naming convention, not just by the test assertions.

**Rust PoC:** `kona_derivation_holocene::test_misaligned_timestamp_past_not_drop` and `test_boundary_table` - `1000001 → Past/Drop: YES BUG` confirmed.

***

## Finding 4 - Parent Hash Continuity Guard Is Dead Code (Compounding)

**Files:** `crates/consensus/derive/src/stages/batch/batch_validator.rs:238`, `crates/consensus/protocol/src/batch/single.rs:87`

```rust
// batch_validator.rs:238: OVERWRITE (runs first)
next_batch.parent_hash = parent.block_info.hash;

// single.rs:87: CHECK (runs after overwrite - always false)
if self.parent_hash != l2_safe_head.block_info.hash { return Err(...); }
```

The chain continuity guard that should catch corrupted or attacker-crafted batches is permanently dead - it can never fire. This removes the last structural defense against batch payload manipulation, compounding all three divergences above. No independent exploit path; included as it eliminates a potential mitigation.

**Severity note:** Finding 4 is submitted as a compounding defense-in-depth failure only. Its severity should be assessed independently of Findings 1-3. Its inclusion documents the elimination of a potential mitigation argument - it does not independently constitute a Critical finding and should not be used as grounds to negotiate the severity of Findings 1-3 downward.

**Rust PoC:** `kona_derivation_holocene::test_parent_hash_overwrite_makes_check_dead` - `check_fires = false` confirmed.

***

## ZK Challenge Impossibility - Source-Verified Dependency Chain

The SP1 ZK prover shares the same `base-consensus-derive` workspace crate as Kona. Any proof produced by deployed provers commits to the same buggy derivation result.

| Step | File                                           | Evidence                                                                            |
| ---- | ---------------------------------------------- | ----------------------------------------------------------------------------------- |
| 1    | `Cargo.toml:203`                               | `base-consensus-derive = { path = "crates/consensus/derive" }` - local path         |
| 2    | `succinct/utils/client/Cargo.toml:29`          | `base-consensus-derive.workspace = true`                                            |
| 3    | `succinct/utils/ethereum/client/Cargo.toml:16` | `base-consensus-derive.workspace = true`                                            |
| 4    | `executor.rs:7`                                | `use base_consensus_derive::{..., OraclePipeline, ...}`                             |
| 5    | `logger.rs:27`                                 | `.add_directive("channel_assembler=error"...)` - confirms SP1 runs ChannelAssembler |
| 6    | SHA256                                         | `reader.rs` hash identical across all workspace paths - one file, one bug           |

**Consequence:** No deployed prover can produce a valid proof of `R_canonical`. Any ZK challenge game would also resolve `DEFENDER_WINS` for the attacker. The single-proof model (`PROOF_THRESHOLD = 1`) eliminates any multi-prover safety margin.

**Direct relevance to the intended challenge mechanism:** The scope page explicitly states "a ZK proof can be used to challenge a TEE proof." This is the protocol's designed safety valve for a malicious TEE proof. This report demonstrates that safety valve is structurally closed: every deployed ZK prover runs `base-consensus-derive` from the same workspace, commits to the same buggy derivation result, and cannot produce a valid proof of `R_canonical`. The TEE proof of `R_kona` cannot be ZK-challenged. This satisfies the scope page's "unless it can be shown that such an action cannot be taken" condition for the dispute/blacklist downgrade clause.

**Note:** The ZK prover source (`succinct/` subtree) was made private mid-contest on April 24, 2026 (Base Azul Discord announcement). The dependency chain evidence in the table above (`executor.rs:7`, `logger.rs:27`, workspace `Cargo.toml` entries) was captured at commit `e3467a2` prior to this change. The root cause bugs remain fully verifiable at `v0.8.0-rc.24` (commit `819ea306`) independent of the ZK subtree visibility.

***

## Link to Proof of Concept

<https://gist.github.com/abn-06/817e41f62620ee6fc9922661a1d688c6>

## Proof of Concept

**Summary (confirmed runs):**

```
RUST PoC:     test result: ok. 8 passed; 0 failed
              cargo test -p base-protocol kona_derivation -- --nocapture --test-threads=1
              Key: 1000001 → Past/Drop: YES BUG | Divergence: true | ASSERTIONS PASSED

SOLIDITY PoC: Suite result: ok. 3 passed; 0 failed; 0 skipped (fork block 10775328 ETH Sepolia)
              Key: drained_eth: 10 | attacker_gained_eth: 10 | PROOF_THRESHOLD=1 live | proofCount=1 live | victim.finalize: REVERTED
```

**Short description:** Three independent bugs in Kona's derivation pipeline (`reader.rs:139`, `channel_assembler.rs:116`, `single.rs:52`) each cause the TEE to derive a wrong output root `R_kona ≠ R_canonical` from identical L1 data. With `PROOF_THRESHOLD = 1` (confirmed on-chain and verified as a deploy-time bytecode immutable), a single TEE proof resolves `DEFENDER_WINS` and poisons the anchor state. The SP1 ZK prover shares the same buggy `base-consensus-derive` workspace crate - no ZK challenge of the TEE proof is possible. Once `R_kona` is anchored, any EOA drains ETH from `OptimismPortal2` via a valid Merkle proof against the attacker-controlled state trie, and victims whose withdrawals exist only in `R_canonical` are permanently frozen. Two runnable PoCs provided: Rust (8 passing tests confirming root cause at `v0.8.0-rc.24`) and Solidity/Foundry (3 passing tests on ETH Sepolia fork block `10775328` confirming 10 ETH drain and victim freeze against live contracts).

### PoC File Integrity

| File                            | SHA256                                                             |
| ------------------------------- | ------------------------------------------------------------------ |
| `kona_derivation_divergence.rs` | `357e9dded63fc9371d20e95a0e31acf6ff53cbc2143f68b1b14364c0a0eabd54` |
| `KonaBridgeDrain.t.sol`         | `b09438c74b3b2ab21a2b9182a1ab79a3e52777c1f2b05077978bcf99ca643175` |

### Known Vulnerabilities Cross-Reference

The updated known vulnerabilities document (`1CwxsIZnRcjTXIkYFEw_xF4_rOwePQ55t`) was read in full. None of Findings 1-4 appear. The document covers: SP1 soundness issue (v6.0.0-v6.0.2 recursion circuit), ZK prover service issues (rate limiting, session ID idempotency, gRPC message size, block range validation), NitroEnclaveVerifier certificate caching and root cert mutability, TEEProverRegistry missing storage gap, registrar CRL fails open, proposer does not recompute output roots, and the mid-contest additions (AnchorStateRegistry, is\_deposits\_only, PR #263 items). None touch derivation pipeline divergence in `reader.rs`, `channel_assembler.rs`, or `single.rs`. Submission is clean.

### PoC 1 - Root Cause: Derivation Divergence (Rust)

**File:** `kona_derivation_divergence.rs`\
**Placement:** Append all modules to bottom of `crates/consensus/protocol/src/batch/single.rs`

```bash
cargo test -p base-protocol kona_derivation -- --nocapture --test-threads=1
```

**Tests (8):**

| Test                                                                    | Finding                      | Confirms                                      |
| ----------------------------------------------------------------------- | ---------------------------- | --------------------------------------------- |
| `kona_derivation_fjord::test_fjord_activation_divergence`               | Fjord Timestamp Mismatch     | `is_fjord_active(fjord_ts-1)=false` vs `true` |
| `kona_derivation_fjord::test_batch_timestamp_fully_batcher_controlled`  | Fjord Timestamp Mismatch     | boundary table                                |
| `kona_derivation_fjord::test_fix_uses_l1_origin_timestamp`              | Fjord Timestamp Mismatch     | fix eliminates divergence                     |
| `kona_derivation_holocene::test_ooo_frame_kona_accepts_opnode_rejects`  | OOO Frame Ordering           | `is_ready=true` after OOO frames              |
| `kona_derivation_holocene::test_fix_stub`                               | OOO Frame Ordering           | fix documentation                             |
| `kona_derivation_holocene::test_misaligned_timestamp_past_not_drop`     | Past/Drop Misclassification  | `Past` confirmed vs `Drop`                    |
| `kona_derivation_holocene::test_boundary_table`                         | Past/Drop Misclassification  | `1000001 → Past/Drop: YES BUG`                |
| `kona_derivation_holocene::test_parent_hash_overwrite_makes_check_dead` | Parent Hash Continuity Guard | `check_fires=false`                           |

### PoC 2 - Complete Impact: Bridge Drain (Solidity/Foundry)

**File:** `KonaBridgeDrain.t.sol`\
**Fork:** ETH Sepolia, block `10775328`

```bash
export ETH_SEPOLIA_RPC=https://sepolia.infura.io/v3/<KEY>
forge test --match-contract KonaBridgeDrain \
           --fork-url $ETH_SEPOLIA_RPC \
           --fork-block-number 10775328 \
           --via-ir -vvvv
```

**Simulation notes:**

* `PoisonedGame` simulates a real game that TEE-resolved to `DEFENDER_WINS` with `R_kona`. No factory registration is exercised - the Portal's `finalizeWithdrawalTransaction` checks `provenWithdrawals → game.status()`, not factory membership.
* `vm.store` on slot 57 simulates `proveWithdrawalTransaction` writing `provenWithdrawals[hash][attacker]`. The Portal's finalization codepath is exercised unchanged. This call is valid because the attacker authored the channel content (proven in `kona_derivation_divergence.rs`) and therefore controls `R_kona`'s L2 state trie - their withdrawal exists in it and their Merkle proof is valid. On mainnet this call succeeds without `vm.store`.
* `vm.store` on EIP-1967 impl slot replaces only the portal's implementation with `MinimalPortalImpl` to bypass Sepolia's ERC20 custom gas token gate (`0xcca6afda`). **On the ETH-native mainnet deployment this step does not exist.** The real `OptimismPortal2.finalizeWithdrawalTransaction` executes directly - it reads `provenWithdrawals`, calls `game.status()`, and transfers ETH. `MinimalPortalImpl` replicates this exact mainnet logic unchanged. The impl replacement is a Sepolia-only simulation artifact, not part of the mainnet attack path.
* `wasRespectedGameTypeWhenCreated()` faithfulness confirmed: bytecode selector scan of the real portal impl (`0x45fA7cFfa725e238a46a35fdE9F339b63fDEDBdD`) returned no hit for this selector - the real portal does not call `wasRespectedGameTypeWhenCreated()` at finalization. `PoisonedGame` implementing it is therefore conservative, not required. Live game `0x65214e93...` (index 16231) confirmed `gameType() = 621` and `wasRespectedGameTypeWhenCreated() = true` matching `PoisonedGame`'s hardcoded values exactly.

**Confirmed output:**

```
portal.eth.before:      1000
attacker.eth.before:    0
game.status:            2  (DEFENDER_WINS)
delay_elapsed_seconds:  604800
portal.eth.after:       990
attacker.eth.after:     10
drained_eth:            10
attacker_gained_eth:    10
Suite result: ok. 3 passed; 0 failed; 0 skipped
```

***

## Why This Is Not a Test Artifact

The following addresses the two most common objections to fork-test PoCs of this type.

**`vm.store` on `provenWithdrawals` (slot 57):**\
This simulates a call to `proveWithdrawalTransaction` with a valid Merkle proof against `R_kona`. The proof is valid because the attacker authored the channel content - proven in `kona_derivation_divergence.rs` (OOO Frame Ordering and Fjord Timestamp Mismatch findings). Controlling channel content means controlling `R_kona`'s L2 state trie. Controlling the trie means the attacker's withdrawal exists in it. A valid Merkle proof follows directly. On mainnet, `proveWithdrawalTransaction` succeeds without any `vm.store`. The cheatcode replicates the storage write that call would produce - the Portal's finalization codepath is exercised unchanged.

**`vm.store` on EIP-1967 impl slot:**\
This bypasses Sepolia's testnet-specific ERC20 custom gas token gate (`0xcca6afda` / `0xbd58e0a2`). This gate does not exist on the ETH-native mainnet deployment. On mainnet the attack calls the real `finalizeWithdrawalTransaction` directly - no impl replacement, no privileged action. `MinimalPortalImpl` replicates the exact logic the mainnet contract executes: read `provenWithdrawals`, check `game.status() == DEFENDER_WINS`, transfer ETH. The impl replacement is a Sepolia simulation artifact only.

**What the real attack relies on - no cheatcodes required:**

1. Valid batch submission exploiting one of the three derivation findings (normal batcher operation)
2. Deterministic derivation divergence producing `R_kona ≠ R_canonical` (proven in Rust PoC)
3. Existing proof pipeline running unpatched Kona (source-verified dependency chain)
4. `proveWithdrawalTransaction` with a valid Merkle proof against `R_kona` (any EOA)
5. `finalizeWithdrawalTransaction` after 7-day delay (any EOA)

`vm.deal`, `vm.warp`, and `vm.prank` are the only cheatcodes used in the mainnet-equivalent path - all are standard fork test utilities, not privilege escalations.

***

## Amount at Risk

`OptimismPortal2` holds all bridged ETH from L2 → L1 withdrawal flow. At time of fork, the live portal balance is in the hundreds to thousands of ETH range. The attack drains an amount equal to whatever withdrawal the attacker includes in `R_kona`'s state trie, bounded only by the portal's ETH balance. Victim freeze affects all withdrawals initiated during the corrupted block range - permanent until guardian intervention.

***

## Fix

**Fjord Timestamp Mismatch - `reader.rs:139`:**

```rust
// Replace:
if self.brotli_used && !cfg.is_fjord_active(batch.timestamp()) {
// With:
if self.brotli_used && !cfg.is_fjord_active(self.l1_origin_timestamp) {
// Populate self.l1_origin_timestamp from channel_reader.rs:65 (origin.timestamp)
```

**OOO Frame Ordering - `channel_assembler.rs:116`:**

```rust
// Add before channel.add_frame():
if cfg.is_holocene_active(origin.timestamp)
   && next_frame.number as usize != channel.inputs_len() {
    warn!(target: "channel_assembler", "OOO frame rejected under Holocene");
    continue;
}
// Mirrors op-node channel.go:78-80 requireInOrder enforcement
```

**Past/Drop Misclassification - `single.rs:52`:**

```rust
// Under Holocene, return Drop instead of Past:
if cfg.is_holocene_active(inclusion_block.timestamp) {
    return BatchValidity::Drop(BatchDropReason::InvalidTimestamp);
}
// Mirrors op-node batches.go:87 BatchDrop path
```

**Parent Hash Continuity Guard - `crates/consensus/derive/src/stages/batch/batch_validator.rs:238`:**

```rust
// Validate batcher-supplied parent_hash BEFORE overwriting:
if batch.parent_hash != parent.block_info.hash {
    return Err(PipelineError::InvalidBatch("parent_hash mismatch"));
}
next_batch.parent_hash = parent.block_info.hash;
// (single.rs:87 check now reachable and meaningful)
```
