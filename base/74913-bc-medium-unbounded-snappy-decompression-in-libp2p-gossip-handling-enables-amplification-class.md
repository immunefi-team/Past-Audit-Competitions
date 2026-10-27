# 74913 bc medium unbounded snappy decompression in libp2p gossip handling enables amplification class resource exhaustion of base consensus follower fleet

**Submitted on Apr 25th 2026 at 21:02:27 UTC by @gzeon for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74913
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.24>
* **Impacts:**
  * Shutdown of greater than or equal to 30% of network processing nodes without brute force actions, but does not shut down the network

## Description

## Brief / Intro

Base's libp2p gossipsub handling decompresses every incoming gossip message **without any decompressed-size cap**, despite the `MAX_GOSSIP_SIZE = 10 MB` constant already enforced on the *compressed* side. Two distinct decompression sinks fire per delivered message — one in `compute_message_id` (on every received message, before dedup), and one in `BlockHandler::handle` via `NetworkPayloadEnvelope::decode_v{1..4}`. Snappy's raw format permits up to 21.3:1 expansion under that 10 MB cap. Empirically: **a single 9.98 MB compressed payload commits 213 MB of RSS and 0.18 s of CPU on every receiving peer per sink**.

Attackers are unauthenticated libp2p peers joining the gossip mesh; no Base credentials, no infra access, no spoofing. On Base's recommended hardware (32 GB RAM, 8+ vCPU per [Base node-operator docs](https://docs.base.org/base-chain/node-operators/performance-tuning#hardware)), a single attacker at \~4 Gbps uplink saturates ≥30 % of the gossipsub task's CPU on every in-mesh peer and pressures co-resident `base-reth-node`, RPC, and flashblocks services with 213 MB RSS spikes per message — qualifying for **MEDIUM** under "Increasing network processing node resource consumption by at least 30 % without brute force actions."

The fix is a 5-line `decompress_len` pre-check using the existing `MAX_GOSSIP_SIZE` constant.

## Vulnerability Details

### The two decompression sinks

**Sink 1.** `crates/consensus/gossip/src/config.rs:104-119` — `compute_message_id` runs for every received gossip message before the duplicate cache is consulted. Decompression therefore fires on first receipt regardless of later dedup:

```rust
fn compute_message_id(msg: &Message) -> MessageId {
    let mut decoder = Decoder::new();
    let id = decoder.decompress_vec(&msg.data).map_or_else(
        |_| { /* fallback hash on raw bytes */ },
        |data| { /* hash on decompressed bytes */ },
    );
    MessageId(id)
}
```

Registered as the gossipsub message-id function at `crates/consensus/gossip/src/config.rs:88-93` alongside `.max_transmit_size(MAX_GOSSIP_SIZE)` (compressed-only cap of 10 MB).

**Sink 2.** `crates/consensus/gossip/src/handler.rs:50-72` — `BlockHandler::handle` selects one of `decode_v{1..4}` based on the message's topic and runs it on the same compressed bytes:

```rust
let decoded = if msg.topic == self.blocks_v1_topic.hash() {
    NetworkPayloadEnvelope::decode_v1(&msg.data)
} else if msg.topic == self.blocks_v2_topic.hash() {
    NetworkPayloadEnvelope::decode_v2(&msg.data)
} ... ;
```

Each `decode_vN` calls `snap::raw::Decoder::decompress_vec(data)` again at:

* `crates/common/rpc-types-engine/src/envelope.rs:258` (`decode_v1`)
* `crates/common/rpc-types-engine/src/envelope.rs:302` (`decode_v2`)
* `crates/common/rpc-types-engine/src/envelope.rs:346` (`decode_v3`)
* `crates/common/rpc-types-engine/src/envelope.rs:402` (`decode_v4`)

For each gossip message that survives id-dedup and is routed to its topic handler, both decompression sinks fire in sequence on the same byte buffer.

### Topic subscription matrix

Topic instances created in `BlockHandler::new` (`crates/consensus/gossip/src/handler.rs:99-102`); all four returned as the accepted topic set by `topics()` (`handler.rs:79-87`):

```rust
blocks_v1_topic: IdentTopic::new(format!("/optimism/{chain_id}/0/blocks")),
blocks_v2_topic: IdentTopic::new(format!("/optimism/{chain_id}/1/blocks")),
blocks_v3_topic: IdentTopic::new(format!("/optimism/{chain_id}/2/blocks")),
blocks_v4_topic: IdentTopic::new(format!("/optimism/{chain_id}/3/blocks")),
```

Every follower subscribes to all four topics. `compute_message_id` hashes only `msg.data` (topic is not part of the digest), so an attacker who publishes the byte-identical payload on multiple topics gets dedup'd globally after the first decompression. The PoC therefore varies one byte per topic so each topic-message has a distinct id and runs its own decompression — a 4-byte total cost to the attacker per round.

### Why `snap::raw::Decoder::decompress_vec` is unbounded

Source-level behaviour (`snap-1.1.1/src/decompress.rs:105-110`):

```rust
pub fn decompress_vec(&mut self, input: &[u8]) -> Result<Vec<u8>> {
    let mut buf = vec![0; decompress_len(input)?];        // pre-allocate declared size
    let n = self.decompress(input, &mut buf)?;
    buf.truncate(n);
    Ok(buf)
}
```

The varint header carries the declared decompressed length, capped only by `MAX_INPUT_SIZE = u32::MAX` (\~4 GB). No application-level cap exists in Base's call sites; both sinks pass attacker-controlled bytes directly to `decompress_vec`.

`vec![0; N]` calls `__rust_alloc_zeroed`, which on glibc/Linux above the `M_MMAP_THRESHOLD = 128 KB` issues `mmap(MAP_ANONYMOUS)`. Pages remain kernel zero pages until first written. The decoder then writes the decompressed payload into the buffer, **physically committing the pages**. A "tiny varint header declaring 4 GB" attack errors out fast (HeaderMismatch) without committing many pages. A **legit-expansion** attack — chained `copy-2-byte len=64 offset=1` instructions — produces a snappy-spec-valid stream that commits real RSS at the snappy max ratio of **21.3:1**.

### Snappy-spec ceiling on per-message decompressed size

| Snappy op                                  | Compressed | Max decompressed | Ratio      |
| ------------------------------------------ | ---------- | ---------------- | ---------- |
| copy-1-byte (3-bit len, 11-bit offset)     | 2          | 11               | 5.5:1      |
| **copy-2-byte (6-bit len, 16-bit offset)** | **3**      | **64**           | **21.3:1** |
| copy-4-byte (6-bit len, 32-bit offset)     | 5          | 64               | 12.8:1     |

Under Base's `MAX_GOSSIP_SIZE = 10 MB` cap, **per-message decompressed size is bounded at \~213 MB**. This is the load-bearing constant for the rest of the report.

### Per-decompression cost

Single-process measurement on x86\_64 Linux (release build):

```
Compressed: 9984378 B (9.52 MB)
Target decompressed: 213000000 B (203.13 MB)
10 decompressions in 1.804986475s
Per-decompression: 180.498647ms
Maximum resident set size (kbytes): 219868   # 213 MB committed real RSS
```

Each gossip message processed through one sink costs the receiving peer **\~0.18 s CPU + 213 MB peak RSS**. With both sinks routed (sink-1 dedup miss followed by sink-2 handler), a single delivered message is **\~0.36 s CPU + 213 MB peak**.

### Attacker reachability

libp2p gossipsub admits any peer that completes the standard noise + yamux + identify handshake. No Base-specific credentials, no bearer tokens, no IP allowlist. The attacker reaches the gossip mesh exactly as any honest follower does. Discoverable via code review alone; not "compromising Base-operated infrastructure."

## Impact Details

### Attack profile against Base's recommended hardware

Base's [node-operator performance-tuning page](https://docs.base.org/base-chain/node-operators/performance-tuning#hardware) recommends **32 GB RAM, 8+ vCPU**.

The CPU-saturation threshold for "≥30 % resource consumption" on 8 cores = 2.4 cores busy.

| Step                                                                       | Value                                                            |
| -------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| Per-message CPU cost                                                       | 0.18 s × 2 sinks = **0.36 cores-second per delivered message**   |
| Per-publish CPU cost on each receiver (4 topics, salted)                   | 4 × 0.36 = **1.44 cores-second per attacker round per receiver** |
| Threshold publishes/sec to saturate 30 % of the gossipsub task             | 2.4 / 1.44 ≈ **1.67 publishes/sec**                              |
| Attacker uplink (mesh fan-out: `mesh_D × topics × publishes/s × msg_size`) | 1.67 × 4 × 8 × 9.98 MB ≈ **534 MB/s ≈ 4.3 Gbps**                 |

Single attacker, \~4 Gbps uplink, every in-mesh peer simultaneously pays ≥30 % gossipsub-task CPU. The figure is for a single attacker meshed at one position; multiple sock-puppets at different mesh positions distribute the same total uplink without changing the per-receiver impact.

Memory side-effect: each delivered gossip message commits a 213 MB RSS spike that lives for the duration of the snappy decompression (\~0.18 s) plus any handler-side validation. On a co-resident `base-consensus + base-reth-node + RPC + flashblocks` deployment (the standard configuration for Base followers), every spike forces the kernel to evict EL state-DB hot pages from page cache, slowing block sync. With swap disabled (universal for blockchain nodes — see "kernel/system configurations" below) the kernel cannot ease the pressure by paging.

### Why this is amplification, not brute force

Per-byte ratios on Base's recommended 32 GB / 8-core hardware:

```
Per-receiver, per attacker byte:
   memory:  9.98 MB compressed : 426 MB committed (dual-sink) ≈ 1 : 44
   CPU:     9.98 MB compressed : 0.36 cores-second           ≈ 36 ns CPU per byte

Network-wide via mesh fan-out (mesh_D = 8, ~50 in-mesh peers):
   memory:  ≈ 1 : 1100
   CPU:     1 publish (80 MB attacker uplink) → 50 receivers × 0.36 cores-sec
            ≈ 18 cores-seconds per 80 MB attacker uplink
            ≈ 225 milli-cores-seconds per MB attacker uplink
```

The discriminator between amplification and brute force is whether attacker resource cost is symmetric to victim cost. This finding's per-byte ratios are asymmetric by 1–3 orders of magnitude. Two further structural points:

* **A code-level fix exists and closes the entire amplification factor** (5-line patch using a constant Base already defines). Pure brute-force floods can only be mitigated at the network edge.
* **Mesh fan-out decouples attacker uplink from victim count.** A bandwidth flood scales linearly with victim count; this attack's uplink is bounded by `mesh_D × topics × stacking-rate × msg_size`, **independent of the number of in-mesh peers paying CPU + memory cost**.

### Kernel and system configurations that lower the threshold

Common production-server defaults that increase the bug's blast radius (no operator misconfiguration required):

| Configuration                                                     | Effect                                                                                                                                                      |
| ----------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Swap disabled / `vm.swappiness=0`                                 | No swap relief on memory pressure → process competes for committed RSS. Universal on blockchain nodes (operators disable swap to avoid latency variance).   |
| Co-resident `base-consensus + base-reth-node + RPC + flashblocks` | Per-process memory budget shrinks; gossipsub task's 213 MB spikes evict EL state-DB hot pages from kernel page cache. Universal in single-host deployments. |
| `cgroup v2` with `memory.swap.max=0`                              | Container-level no-swap. Docker default for `mem_limit`.                                                                                                    |
| `vm.overcommit_memory=2` (strict accounting)                      | Allocations rejected before they can be paged out → fast-fail OOM at lower RSS.                                                                             |
| `systemd-oomd` enabled                                            | Proactive OOM kill at lower memory-pressure thresholds. Increasingly common on Ubuntu/Fedora servers.                                                       |

Under the standard "swap=0 + co-resident services" production deployment, every gossip message processed forces a memory-pressure shock to the EL state cache, slowing block sync below sustainable rate.

### Severity classification

Per Base's bounty severity table:

> **MEDIUM**: "Increasing network processing node resource consumption by at least 30 % without brute force actions, compared to the preceding 24 hours."

This is the load-bearing classification. On 32 GB / 8-core followers, a single attacker at \~4 Gbps causes ≥30 % gossipsub-task CPU saturation on every in-mesh peer per the derivation above.

Two subordinate points:

* The bug also satisfies the alternative MEDIUM criterion "Shutdown of ≥30 % of network processing nodes without brute force actions" on lower-RAM victims (cgroup containers, low-spec follower deployments). Empirically reproduced at 1 GB cgroup with explicit `container oom` events; see PoC.
* The finding does **not** rise to HIGH: no direct ≥500 % block-production delay (Base's sequencer is centralized; gossip layer is not on the L2 block-production critical path), no chain-split, no RPC-API crash on programs with ≥25 % market cap.

## References

### Source-level evidence (Base v0.8.0-rc.24, commit `819ea30`)

* `crates/consensus/gossip/src/config.rs:15` — `pub const MAX_GOSSIP_SIZE: usize = 10 * (1 << 20);`
* `crates/consensus/gossip/src/config.rs:31` — `pub const DEFAULT_MESH_D: usize = 8;`
* `crates/consensus/gossip/src/config.rs:88` — `.max_transmit_size(MAX_GOSSIP_SIZE)` (compressed cap binding)
* `crates/consensus/gossip/src/config.rs:104-119` — `compute_message_id` (Sink 1)
* `crates/consensus/gossip/src/handler.rs:50-72` — `BlockHandler::handle` (Sink 2 dispatcher)
* `crates/consensus/gossip/src/handler.rs:79-87` — accepted-topics matrix (`topics()` returns all four)
* `crates/consensus/gossip/src/handler.rs:99-102` — topic instances created in `BlockHandler::new`
* `crates/common/rpc-types-engine/src/envelope.rs:258, 302, 346, 402` — four `decode_vN::decompress_vec` sites

### Upstream evidence

* `snap-1.1.1/src/decompress.rs:105-110` — `decompress_vec` allocates `vec![0; decompress_len(input)?]` before validating payload completeness
* `snap-1.1.1/src/lib.rs:93` — `MAX_INPUT_SIZE = u32::MAX` allows up to 4 GB declared decompressed size
* libp2p-gossipsub mesh-D fan-out: one publish reaches all in-mesh peers in O(log\_D N) hops via mesh forwarding
* Snappy format spec: `copy-2-byte len=64 offset=1` yields 21.3:1 ratio per chained instruction
* Base hardware recommendations: <https://docs.base.org/base-chain/node-operators/performance-tuning#hardware>

### Bug introduction

The vulnerable pattern was introduced at the initial Base Consensus port in commit `feat(consensus): Base Consensus (#898)` — the gossip config and envelope decoders were ported from op-node without a decompressed-size cap. Pattern unchanged through `v0.8.0-rc.24`. No fix commit observed in the rc.1–rc.24 range.

### In-scope confirmation

* `crates/consensus/gossip/` and `crates/common/rpc-types-engine/` are inside `crates/`, not in any out-of-scope directory.
* Reachable via libp2p gossip mesh, no Base-operated infrastructure compromise required.
* Discoverable via code review alone — both sinks are 5–20 line public Rust functions.
* Not present in the publicly-disclosed known-issues list.

## Link to Proof of Concept

<https://gist.github.com/gzeoneth/9c756cb5dcc09ebf29514bb2856faa4f>

## Proof of Concept

* **Single-process measurement** — `snap-bomb-poc.rs`. Demonstrates the snappy-level 21:1 ratio, 213 MB committed RSS, and 0.18 s CPU per `decompress_vec` call. Run on Linux with `cargo run --release` followed by `/usr/bin/time -v target/release/snap-bomb-poc`.

```rust
fn varint(mut v: u64) -> Vec<u8> {
    let mut out = Vec::new();
    loop {
        let b = (v & 0x7f) as u8;
        v >>= 7;
        if v == 0 { out.push(b); break; }
        out.push(b | 0x80);
    }
    out
}

fn build_legit_bomb(target_size: u64) -> Vec<u8> {
    let mut out = varint(target_size);
    out.push(0x00); out.push(0xAA);
    let copies = (target_size - 1) / 64;
    for _ in 0..copies {
        out.push((63 << 2) | 0b10); out.push(0x01); out.push(0x00);
    }
    out
}

fn main() {
    // Target: max compressed ~9 MB → ~600 MB decompressed (via 64x copy ratio)
    let target: u64 = 213_000_000;
    let bomb = build_legit_bomb(target);
    println!("Compressed: {} B ({:.2} MB), target: {} B ({:.2} MB), ratio: {:.0}:1",
        bomb.len(), bomb.len() as f64 / 1_048_576.0,
        target, target as f64 / 1_048_576.0,
        target as f64 / bomb.len() as f64);
    if bomb.len() > 10 * 1024 * 1024 {
        println!("WARNING: compressed exceeds MAX_GOSSIP_SIZE (10 MB)");
    }
    let t = std::time::Instant::now();
    let mut d = snap::raw::Decoder::new();
    let res = d.decompress_vec(&bomb);
    let dt = t.elapsed();
    match res {
        Ok(v) => println!("Decompressed: {} B in {:?}", v.len(), dt),
        Err(e) => println!("Err: {:?}", e),
    }
}
```

* **Multi-node end-to-end reproduction** — `poc-docker` <https://gist.github.com/gzeoneth/9c756cb5dcc09ebf29514bb2856faa4f> A 4-container Docker Compose stack running the EXACT vulnerable `compute_message_id` pattern + the `decode_vN` second sink, demonstrating OOM kill of a 1 GB cgroup follower under realistic Base-shaped attacker config (3 sock-puppets × 4 topics × 2 publishes/s × 2 sinks). Captured 2026-04-25:

```
2026-04-25T07:40:29.225182946Z container oom 9bb562e60ee6...
  com.docker.compose.service=victim-a-r
  name=poc-docker-victim-a-r-1

2026-04-25T07:40:29.622919395Z container die 9bb562e60ee6...
  execDuration=204
  exitCode=137                            # SIGKILL by OOM killer

$ docker inspect poc-docker-victim-a-r-1 --format \
    "R={{.RestartCount}} OOMKilled={{.State.OOMKilled}}"
R=1 OOMKilled=false                       # OOMKilled flag clears post-restart
```

`OOMKilled=false` is normal post-restart docker behaviour; `RestartCount=1` and the explicit `container oom` event are the load-bearing evidence.

The cgroup-OOM scenario demonstrates that the bug's per-message peak RSS (213 MB) is sufficient to OOM-kill containers below \~1 GB. The primary impact on **production 32 GB hardware** is the CPU + memory-pressure profile derived in "Impact Details" above, not direct OOM.

### Reproduction commands

```bash
cd poc-docker
docker compose --profile realistic up -d
sleep 180
docker events --since 5m --filter event=oom
docker inspect poc-docker-victim-a-r-1 --format \
    "RestartCount={{.RestartCount}} OOMKilled={{.State.OOMKilled}}"
```

### PoC adaptations explicitly disclosed

Two cosmetic adaptations were made to the PoC; **neither affects the structural attack**:

* `mesh_n=2/n_low=1/n_high=4/outbound_min=0` (vs. Base's production 8/6/12). Required because libp2p-gossipsub validates `D_low * 2 ≤ D` and a 4-node mesh cannot satisfy 6/12. Mesh-D affects fan-out efficiency, not whether the bug exists. The 32 GB-hardware derivation in "Impact Details" uses Base's production `mesh_D=8`.
* `add_explicit_peer()` is called after dial completes. In production Base, gossipsub naturally meshes within \~2 heartbeats once SUBSCRIBE control messages exchange.

## The fix

The existing `MAX_GOSSIP_SIZE = 10 MB` constant is already imported at both vulnerable sites. `snap::raw::decompress_len` reads only the ≤5-byte varint header and performs no allocation, so it is safe to call before `decompress_vec`:

```rust
fn bounded_decompress(input: &[u8]) -> Result<Vec<u8>, snap::Error> {
    let len = snap::raw::decompress_len(input)?;
    if len > MAX_GOSSIP_SIZE {                          // <-- the missing check
        return Err(snap::Error::TooBig {
            given: len as u64,
            max: MAX_GOSSIP_SIZE as u64,
        });
    }
    snap::raw::Decoder::new().decompress_vec(input)
}
```

Apply in:

* `crates/consensus/gossip/src/config.rs:104-119` (`compute_message_id`)
* `crates/common/rpc-types-engine/src/envelope.rs:258, 302, 346, 402` (all four `decode_vN`)

A `NetworkPayloadEnvelope` v4 carrying a full-size Isthmus block (30 M gas) is well under 10 MB SSZ-encoded, so 10 MB decompressed accommodates every legitimate payload. Using the same constant as the compressed cap forces effective ratio ≤ 1:1 (decompressed cannot exceed compressed cap), structurally eliminating the amplification factor regardless of future Snappy version changes.

### Fix verification

Apply the patch above to `vulnerable_message_id` in `poc-docker/src/main.rs` and rebuild:

```bash
docker compose --profile realistic up --build
```

Victim RSS stays under \~50 MB regardless of attacker rate. `RestartCount` remains at 0. Attacker `publish` calls succeed at the wire level but receivers reject the bombs at message-id computation, so the RSS impact is zero.
