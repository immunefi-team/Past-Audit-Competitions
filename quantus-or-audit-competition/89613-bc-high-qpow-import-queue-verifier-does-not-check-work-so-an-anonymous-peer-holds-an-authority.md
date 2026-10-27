# 89613 bc high qpow import queue verifier does not check work so an anonymous peer holds an authority in major sync and halts its block production indefinitely at zero hash power

**Submitted on Aug 20th 2026 at 06:18:05 UTC by @zealynx for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89613
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

## Description

## Brief/Intro

An anonymous network peer with no valid proof-of-work and near-zero bandwidth can hold a Quantus authority in major sync indefinitely, which pauses the authority's block production. The peer advertises a tip at least six blocks ahead and repeatedly serves a valid-looking block range whose first importable block carries an invalid 64-byte QPoW seal. Because Quantus's import-queue verifier does not check work — the work check happens later, in block import, and surfaces as a generic error that does not blame the peer — the malicious source is never dropped. Its advertised height keeps the node above the major-sync threshold, and it can repeat forever at about 8 KiB/s. In the reproduction, one attacker held a fresh authority at height 0 for 125 seconds across 600 rejected imports; disconnecting only the attacker let the unchanged authority mine block 1 in 62 seconds.

## Vulnerability Details

The root cause is an in-scope QPoW defect: the import-queue verifier does not perform a peer-attributable work check.

`SimplePowVerifier::verify` only extracts the final 64-byte seal (`client/consensus/qpow/src/lib.rs:450`); it does not verify the work. The actual consensus check runs later, inside `PowBlockImport::import_block` (`lib.rs:266-296`), where invalid work returns a generic importer error.

That placement loses the peer attribution the drop path needs. `sc-consensus` maps **verifier** errors to `BlockImportError::VerificationFailed(peer, …)` — which `ChainSync` punishes by dropping the peer — but maps **importer** errors to `BlockImportError::Other`, for which `ChainSync` merely restarts synchronization **without** dropping the source (`client/network-sync/src/strategy/chain_sync.rs:747` vs `:777`). The request transport itself is also counted as peer success before import completes, so the retained peer serves the same invalid range repeatedly with no accumulating failures. Its advertised height stays in `median_seen()` (upper median), and a tip six blocks ahead keeps `is_major_syncing()` true because `MAJOR_SYNC_BLOCKS = 5`. The production mining loop pauses while that flag is true (`worker.rs:169-175`), and mined-result submission independently rejects candidates during major sync.

(The `Other`-vs-`VerificationFailed` mapping and `median_seen` live in vendored upstream Substrate and are correct under their assumption that importer errors are local, not peer-attributable. The in-scope defect is that Quantus's verifier pushes a *peer-attributable* failure — bad work — down the importer path instead of failing it in the verifier. The fix is Quantus-side.)

The block is never imported — the QPoW check correctly rejects it before fork choice. The vulnerability is the **loss of peer attribution at that boundary**, which turns invalid consensus data into an indefinitely retryable sync error.

### Vulnerable scenario

{% stepper %}
{% step %}

## An inbound peer advertises a future tip

An ordinary inbound PQ-noise peer advertises itself as a full peer at height 6. The handshake does not prove the advertised tip's work.
{% endstep %}

{% step %}

## The peer serves malformed work data

It returns six SCALE-decodable, correctly numbered and parent-linked headers in descending order. Block 1 is a genuinely-mined production-valid Planck block with only its final 64-byte seal replaced.
{% endstep %}

{% step %}

## The verifier accepts the block

`SimplePowVerifier` accepts the block after extracting the well-shaped seal.
{% endstep %}

{% step %}

## The importer rejects work without attributing the peer

`PowBlockImport` rejects the invalid work, but the queue reports `Other`, not `VerificationFailed(peer, …)`.
{% endstep %}

{% step %}

## Synchronization restarts and authoring remains paused

`ChainSync` restarts without evicting the source. The peer's height keeps the authority in major sync, so the request/reject cycle repeats while authoring stays paused.
{% endstep %}
{% endstepper %}

No attacker balance, trusted/authority key, valid proof-of-work, local access, exposed RPC, custom chain spec, debug build, unsafe victim option, or hash power is required.

### Recommended fix

Perform peer-attributable QPoW validation in the import-queue `Verifier`, so an invalid engine, seal length, seal value, or parent-state difficulty returns from `Verifier::verify` and reaches sync as `BlockImportError::VerificationFailed(peer, …)`; keep the importer check as defense in depth. Do **not** blanket-drop peers for every `Other` (that also covers innocent local DB/runtime failures) — preserve a typed invalid-consensus-data error with its source peer and map only that class to the existing `VERIFICATION_FAIL` disconnect path.

## Impact Details

An anonymous peer can prevent a normally-configured Planck authority from producing canonical blocks for an unbounded period, at \~8 KiB/s and zero hash power.

* **Demonstrated:** one attacker plus one honest genesis peer connected; the attacker held a fresh authority at height 0 across **600** production import rejections for the 125 s the harness ran (the mechanism has no time limit — it persists as long as the attacker serves). One 1,770-byte response \~4.8×/second.
* **Causal control:** disconnecting **only** the attacker let the unchanged authority + external miner produce block 1 in 62 s — so the stall is caused by peer retention, not by mining difficulty.
* **Amplification:** `median_seen` is the upper median, so against an authority with more honest sync peers the attacker needs an equal number of cheap Sybil identities — no hash power. Repeated against active authorities, this halts chain progress network-wide.
* **Fix-efficacy (disclosed honestly):** the verifier fix drops the offending peer on its first invalid block and, at the production reputation cost `VERIFICATION_FAIL = -(1<<29)` against the ban threshold `71*(i32::MIN/100)`, **bans it after three** — so its tip leaves `median_seen` and the authority exits major sync and authors. A variant that rotates a **fresh identity per invalid block** is not fully closed by dropping alone; it needs the paired mitigation (rate-limit per-peer sync-restart churn, or exclude never-work-valid tips from the major-sync height determination). This is stated so the single-peer demo is not over-claimed as covering identity rotation.
* **Prerequisites:** none beyond the ability to open an inbound sync connection — permissionless.

## References

Frozen revision: `quantus-chain @ 3b243b870a5442a6e5f443056f67bbba87d472f5`

* <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/client/consensus/qpow/src/lib.rs#L266-L296> — work check runs in `import_block`, not the verifier
* <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/client/consensus/qpow/src/lib.rs#L402-L452> — `SimplePowVerifier::verify` extracts the seal only
* <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/client/consensus/qpow/src/worker.rs#L169-L175> — authoring paused while `is_major_syncing()`
* <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/client/network-sync/src/strategy/chain\\_sync.rs#L747-L781> — `VerificationFailed` drops the peer; `Other` only restarts
* <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/client/network-sync/src/strategy/chain\\_sync.rs#L1573-L1584> — `median_seen` upper median

## Link to Proof of Concept

<https://gist.github.com/vendrell46/bca03145d14505e08222bf25cd7f888a>

## Proof of Concept

Everything runs locally against commit `3b243b8`, the shipped `planck_live_spec`, and production `MaxReorgDepth = 100`. **No test was run against mainnet or any public testnet.** The harness traverses the real PQ-noise transport, block-sync protobuf/SCALE decoders, production import queue, runtime QPoW verifier, sync strategy, authoring loop, and external-miner interface. No production source file is modified. The complete harness (6 files across two helper crates, `poc-invalid-work/` and `poc-miner/`) is in the secret gist linked at the end; steps, commands, and observed output are below.

### Step by step

{% stepper %}
{% step %}

## Fixture (baseline control)

A release authority + external miner mine one genuinely protocol-valid Planck block #1; only its final 64-byte seal is later overwritten with `0xa5…`. This is imported cleanly first, so the only change under test is the seal.
{% endstep %}

{% step %}

## Attacker peer

A standalone litep2p node advertises a full-peer handshake at height 6 and, on every sync range request, returns six linked headers (only #1 reaches import) with the corrupted seal — at \~5 small replies/second (rate-limited, so the impact is from retention, not flooding).
{% endstep %}

{% step %}

## Victim authority

A fresh production authority + miner boots with the attacker **and** an honest genesis peer connected (`peers: 2`, `isSyncing: true`, height 0).
{% endstep %}

{% step %}

## Assert the stall

For the hold window, victim height stays 0, major sync stays active, the attacker connection stays open, and hundreds of real `PoW validation error: invalid seal` import failures accrue against the same retained peer.
{% endstep %}

{% step %}

## Causal control

Disconnect **only** the attacker; assert the unchanged authority/miner reach height ≥1 (honest peer still connected) — isolating peer retention as the cause.
{% endstep %}
{% endstepper %}

### Build & run

Helper crates are siblings of the audited `chain/` checkout.

```sh
cd poc-invalid-work
cargo build --release
cargo build --release --manifest-path ../poc-miner/Cargo.toml
(cd ../chain && cargo build --release -p quantus-node)
ATTACK_SECONDS=125 THREADS=8 ./run.sh
```

### Observed output

```
node: quantus-node 0.8.0-exodus-3b243b870a5
chain: planck_live_spec; production QPoW import; MaxReorgDepth=100
attacker: /ip4/127.0.0.1/tcp/57198/p2p/QmZQ9M...VDbu (no valid PoW mined)
during attack: height=0 health={"isSyncing": true, "peers": 2, "shouldHavePeers": true} responses=601 response_bytes=1770 invalid_imports=600 peer_reconnections=0 attacker_alive=yes
PASS
expected-safe: invalid QPoW is rejected and its source peer is dropped, allowing the authority to mine
harmful-actual: the same peer was retained across 600 rejected imports and kept height at 0 for 125s
causal-control: after attacker disconnect, the unchanged authority/miner reached height=1 in 62s
```

**Full PoC files (poc-invalid-work: Cargo.toml, src/main.rs, rpc.py, run.sh; poc-miner: Cargo.toml, src/main.rs):** <https://gist.github.com/vendrell46/bca03145d14505e08222bf25cd7f888a>
