# 89171 bc high missing pending connection cap and per ip limit in litep2p tcp transport causes 30 resource increase and 500 block delays via zombie peer flood

**Submitted on Aug 17th 2026 at 15:44:57 UTC by @craftsoldier for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89171
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

## Description

## Summary

litep2p's ConnectionLimits only tracks established (post-handshake) connections; there is no pending-connection cap and no per-IP limit. An attacker can open a large number of TCP connections that either stall before the handshake or complete the full pqXX Noise handshake to become established. Each connection pins memory on the victim, and without any per-IP or pre-handshake throttle, a single source IP can monopolize the connection pool.

The PoC demonstrates two directly measured High-tier impacts: node RSS increases by more than 30% relative to baseline, and transaction inclusion latency exceeds 500% of normal. On a 4-core Docker container with 5,000 concurrent zombie peers, `docker stats` shows RSS growing from \~166MB to 3.18GB (\~1,900% increase) and canary inclusion latency reaching 57.8s. On a local Mac with 10,000 zombies, RSS peaked at 3.1GB (from \~200MB baseline) and mean canary latency degraded from 0.76s baseline to 8.6s (11x). The attack is remote, unauthenticated, and uses litep2p's own handshake code, so the zombie peers are indistinguishable from real peers at the protocol level.

## Vulnerability Detail

The ConnectionLimits struct in client/litep2p/src/transport/manager/limits.rs tracks incoming connections in a single HashSet called incoming\_connections. A connection ID is only inserted into this set by accept\_established\_connection(), which is called after the full handshake completes; the pre-handshake gate on\_incoming() checks incoming\_connections.len() against the configured maximum, meaning connections that have not yet completed their handshake are invisible to the limit.

When a PendingInboundConnection event arrives (client/litep2p/src/transport/manager/mod.rs:1371), the manager calls on\_pending\_incoming\_connection() which delegates to the same established-only on\_incoming() check; if it passes, accept\_pending() pushes an uncapped async task onto pending\_connections (client/litep2p/src/transport/tcp/mod.rs:153) with no semaphore, no queue depth limit, no per-IP throttle.

Quantus's fork of litep2p replaces standard Noise XX with a pqXX handshake using ML-KEM-768 + Dilithium-87 (client/litep2p/src/crypto/noise/mod.rs). After the handshake completes, a NoiseSocket is allocated with 455KB of buffers (contains 5 × 65KB read-ahead plus 2 × 65KB write), as documented in client/litep2p/src/transport/tcp/config.rs:52-69. Upstream litep2p with plain Noise XX has approximately 2KB per-connection cost; **Quantus's PQ modification raises this to 455KB, a 200× increase that turns a mild upstream gap into a severe resource-exhaustion vector.**

The node configures max\_incoming\_connections = 10,000 (client/network/src/lib.rs:310, applied at client/network/src/litep2p/mod.rs:520). Zombie peers that complete the handshake become established and count toward this cap; at \~600KB–1MB per established connection, 10,000 zombies could consume 6–10GB, which is larger than the RAM available to many nodes. The PoC directly measured 3.1GB–3.18GB consumption at 5,000–10,000 zombies on hardware with ample memory, confirming the severe per-connection cost and resource-exhaustion potential.

The correct defense already exists in the codebase for the QUIC listener: node/src/miner\_server.rs:576 implements MAX\_UNAUTHENTICATED\_CONNECTIONS, a semaphore capping pre-auth concurrency with a 5s auth deadline. The TCP transport lacks the equivalent defense.

## Impact

Two High-tier impacts were directly measured in the PoC:

1. "Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours." On a 4-core Docker container with 5,000 zombies, `docker stats` recorded RSS growing from \~166MB baseline to 3.18GB peak (\~1,900% increase). On a local Mac with 10,000 zombies, RSS peaked at 3.1GB from a \~200MB baseline (\~1,500% increase). Both far exceed the 30% threshold.
2. "Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments." On the 5,000-zombie Docker run, real Dilithium-signed balance transfer canaries took up to 57.8s to be included. On the 10,000-zombie Mac run, mean canary latency was 8.6s vs a 0.76s baseline (11x) and max latency reached 18.9s (25x). These latencies represent severe transaction freezing well beyond 500% of normal.

The attack is remote (TCP to the public P2P port), unauthenticated (any valid Dilithium keypair works), and indistinguishable from legitimate peers (real pqXX handshake, real identify response, real ping echoes). All testing was performed locally on --dev nodes and Docker containers; no mainnet or public testnet was involved.

## Fix Recommendation

Add a pending-inbound connection cap by inserting a semaphore at on\_pending\_incoming\_connection() time, mirroring miner\_server.rs's MAX\_UNAUTHENTICATED\_CONNECTIONS pattern; acquire before spawning the handshake task, release on timeout or failure, and convert to an established slot on success. Additionally, add per-IP connection limiting to prevent a single source from monopolizing the connection pool, and consider lowering the 10,000 established cap given the per-connection memory cost.

## Link to Proof of Concept

<https://gist.github.com/craftsoldier/57d25519d0bb99eb4f8e211639da3041>

## Proof of Concept

All testing was performed on local --dev nodes and local Docker containers; no mainnet or public testnet was involved.

### Step-by-step

{% stepper %}
{% step %}

## Generate unique Dilithium keypairs

The attacker generates 10,000 unique Dilithium keypairs, because litep2p limits one connection per peer ID and the peer ID is derived from the public key (client/litep2p/src/peer\_id.rs).
{% endstep %}

{% step %}

## Initiate TCP connections and negotiate Noise

For each keypair, the attacker opens a TCP connection to the victim's P2P port (default 30333) and sends the multistream-select V1 header plus the /noise protocol name. The header /multistream/1.0.0\n is 19 bytes (varint 0x13) and /noise\n is 7 bytes (varint 0x07). Both are varint-prefixed unsigned LEB128.

```rust
let header = b"\x13/multistream/1.0.0\n";  // 20 bytes total
let proto  = b"\x07/noise\n";               // 8 bytes total
stream.write_all(&[header, proto].concat()).await?;
stream.flush().await?;
let mut hdr_echo = [0u8; 20];
stream.read_exact(&mut hdr_echo).await?;
let mut proto_echo = [0u8; 8];
stream.read_exact(&mut proto_echo).await?;
```

{% endstep %}

{% step %}

## Perform the pqXX handshake

The victim's listener\_select\_proto (client/litep2p/src/multistream\_select/listener\_select.rs:58) echoes the header, accepts /noise, and echoes the protocol name. After that, the attacker calls litep2p's own handshake() function with Role::Dialer. This performs the full four-message pqXX handshake (ML-KEM-768 + Dilithium-87) on the exact same code path as a legitimate peer.

```rust
let (mut noise_socket, _peer_id) = handshake(
    stream.compat_write(),
    &keypair,
    Role::Dialer,
    5, 2,
    Duration::from_secs(10),
    HandshakeTransport::Tcp,
).await?;
```

{% endstep %}

{% step %}

## Allocate the NoiseSocket buffers

Once the handshake completes, the victim allocates a NoiseSocket with 455KB of buffers (5 × 65KB read-ahead + 2 × 65KB write, client/litep2p/src/crypto/noise/mod.rs:354 and client/litep2p/src/transport/tcp/config.rs:52-69). Upstream litep2p with plain Noise XX costs about 2KB per connection; Quantus's PQ modification raises this to 455KB, a 200× amplification.
{% endstep %}

{% step %}

## Negotiate yamux

The attacker then negotiates /yamux/1.0.0 on the encrypted NoiseSocket. Without this second multistream-select, the victim's negotiate\_connection (client/litep2p/src/transport/tcp/connection.rs:467) times out after 5 seconds and closes the connection, freeing the 455KB.

```rust
let yamux_header = b"\x13/multistream/1.0.0\n";
let yamux_proto  = b"\x0d/yamux/1.0.0\n";  // 14 bytes total
noise_socket.write_all(&[yamux_header, yamux_proto].concat()).await?;
noise_socket.flush().await?;
```

{% endstep %}

{% step %}

## Maintain the session

After yamux is accepted, the victim creates a yamux::Connection and opens substreams for identify, ping, and notification protocols. The attacker maintains the session by reading 12-byte yamux frame headers and ACKing every substream SYN, responding to every yamux PING, and never sending FIN or RST. On ping substreams the attacker echoes 32-byte payloads; on identify substreams it sends a minimal valid protobuf response so the victim creates a full peer store entry.
{% endstep %}

{% step %}

## Retain established connections

Because ConnectionLimits only counts established connections (client/litep2p/src/transport/manager/limits.rs:106,145) and there is no pending cap or per-IP throttle (client/litep2p/src/transport/tcp/mod.rs:153), all of these connections remain alive indefinitely and continue pinning memory.
{% endstep %}

{% step %}

## Run concurrent zombie loops

The attacker runs 5,000–10,000 of these loops concurrently. The PoC reconnects automatically if any connection drops, maintaining the concurrent count.
{% endstep %}
{% endstepper %}

### Measured results

#### 5,000 zombies on a 4-core Docker container

* RSS measured via `docker stats` grew from \~166MB to 3.18GB (\~1,900% increase)
* CPU reached 204.6%
* Dilithium-signed balance transfer canaries took up to 57.8s to be included (vs 7s baseline)

#### 10,000 zombies on a local Mac

* RSS peaked at 3.1GB (from \~200MB baseline)
* CPU reached 228.9%
* Mean canary latency was 8.6s vs 0.76s baseline (11x); max 18.9s
* Peak open sockets: 10,000, filling max\_incoming\_connections

### Build and run

```bash
cd quantus-chain
cargo build --release -p poc-litep2p-preauth-dos
cargo build --release -p quantus-node
./target/release/quantus-node --dev --port 30333 --rpc-port 9944 --no-prometheus --in-peers 100
# in another terminal:
ulimit -n 65536
./target/release/poc-litep2p-preauth-dos --handshake \
  --concurrent 5000 --duration-secs 180 \
  --evidence-dir poc-litep2p-preauth-dos/evidence
```

### Code references

* client/litep2p/src/transport/manager/limits.rs:106 — on\_incoming() checks established set only
* client/litep2p/src/transport/manager/limits.rs:145 — accept\_established\_connection(), the only insertion point
* client/litep2p/src/transport/manager/mod.rs:1371 — pending inbound uses the same established-only check
* client/litep2p/src/transport/tcp/mod.rs:153 — unbounded pending\_connections.push, no semaphore
* client/litep2p/src/crypto/noise/mod.rs:145 — ML-KEM keypair generated per inbound connection
* client/litep2p/src/crypto/noise/mod.rs:354 — NoiseSocket::new, 455KB buffer allocation
* client/litep2p/src/transport/tcp/config.rs:52-69 — 455KB per-connection buffer documentation
* client/litep2p/src/transport/tcp/connection.rs:467 — /yamux/1.0.0 negotiation after Noise
* client/network/src/lib.rs:310 — MAX\_CONNECTIONS\_ESTABLISHED\_INCOMING = 10,000
* client/network/src/litep2p/mod.rs:520 — cap applied, established only, no pending dimension
* node/src/miner\_server.rs:576 — correct pre-auth semaphore pattern exists for QUIC only
