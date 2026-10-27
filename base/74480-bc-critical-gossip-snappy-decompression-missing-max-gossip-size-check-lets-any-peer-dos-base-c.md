# 74480 bc critical gossip snappy decompression missing max gossip size check lets any peer dos base consensus

**Submitted on Apr 22nd 2026 at 21:07:41 UTC by @QED for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74480
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/base/base/releases/tag/v0.8.0-rc.15>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## Brief/Intro

`base-consensus` snappy-decompresses inbound gossip messages twice per packet without checking decoded size against `MAX_GOSSIP_SIZE`. A wire-valid 9.4 MiB packet decompresses to 200 MiB at each of two sites. Any anonymous libp2p peer can trigger it. Sustained attack from 4 unauthenticated peers halts the sequencer's block propagation 100% against a local devnet running the official Base sequencer stack.

## Vulnerability Details

### Invariant the code claims

```rust
// crates/consensus/gossip/src/config.rs:13-15
/// The maximum gossip size.
/// Limits the total size of gossip RPC containers as well as decompressed individual messages.  // <-- CLAIMED INVARIANT
pub const MAX_GOSSIP_SIZE: usize = 10 * (1 << 20);
```

The cap is enforced only on compressed wire bytes (`.max_transmit_size(MAX_GOSSIP_SIZE)`), never on decoded size.

### Root cause #1 — `compute_message_id` decompresses every inbound message

```rust
// crates/consensus/gossip/src/config.rs:75-101
builder
    .max_transmit_size(MAX_GOSSIP_SIZE)         // <-- COMPRESSED bytes only
    .validation_mode(libp2p::gossipsub::ValidationMode::None)
    .validate_messages()
    .message_id_fn(compute_message_id);         // <-- runs on EVERY received message

// crates/consensus/gossip/src/config.rs:104-122
fn compute_message_id(msg: &Message) -> MessageId {
    let mut decoder = Decoder::new();
    let id = decoder.decompress_vec(&msg.data).map_or_else(  // <-- ROOT CAUSE: no decoded-size cap
        |_| { /* invalid-snappy domain tag */ },
        |data| { /* valid-snappy domain tag */ },
    );
    MessageId(id)
}
```

Called before topic filtering: every gossipsub message on any topic the node hears about pays the decompression cost.

### Root cause #2 — `decode_v*` decompresses block-topic messages again

```rust
// crates/consensus/gossip/src/handler.rs:49-60  (BlockHandler::handle)
let decoded = if msg.topic == self.blocks_v1_topic.hash() {
    NetworkPayloadEnvelope::decode_v1(&msg.data)
} else if msg.topic == self.blocks_v2_topic.hash() {
    NetworkPayloadEnvelope::decode_v2(&msg.data)
} else if msg.topic == self.blocks_v3_topic.hash() {
    NetworkPayloadEnvelope::decode_v3(&msg.data)
} else if msg.topic == self.blocks_v4_topic.hash() {
    NetworkPayloadEnvelope::decode_v4(&msg.data)
} else { ... };
```

```rust
// crates/common/rpc-types-engine/src/envelope.rs:255-275  (decode_v1; v2/v3/v4 are identical)
pub fn decode_v1(data: &[u8]) -> Result<Self, PayloadEnvelopeError> {
    let mut decoder = snap::raw::Decoder::new();
    let decompressed = decoder.decompress_vec(data)?;   // <-- ROOT CAUSE: no decoded-size cap

    if decompressed.len() < 66 {                        // length check runs AFTER alloc
        return Err(PayloadEnvelopeError::InvalidLength);
    }
    // signature / SSZ checks also run AFTER alloc
```

`decode_v2`/`v3`/`v4` at `envelope.rs:299`, `:343`, `:399` follow the same pattern.

### Why the signer check doesn't gate the allocation

```rust
// crates/consensus/gossip/src/block_validity.rs:108-115
/// The block encoding/compression are assumed to be valid at this point (they are first checked
/// in the handle).                                                        // <-- decompression happened BEFORE we got here
pub fn block_valid(&mut self, envelope: &NetworkPayloadEnvelope) -> Result<(), BlockInvalidError> { ... }

// crates/consensus/gossip/src/block_validity.rs:218-230  (inside validate_block_internal)
let msg = envelope.payload_hash.signature_message(self.rollup_config.l2_chain_id.id());
let block_signer = *self.signer_recv.borrow();
let Ok(msg_signer) = envelope.signature.recover_address_from_prehash(&msg) else {
    return Err(BlockInvalidError::Signature);          // <-- signer recovery runs LATER, long after alloc
};
if msg_signer != block_signer {
    return Err(BlockInvalidError::Signer { expected: block_signer, received: msg_signer });
}
```

### No ingress-level gate

```rust
// crates/consensus/gossip/src/behaviour.rs:54
MessageAuthenticity::Anonymous            // <-- no per-publisher auth

// crates/client/cli/src/p2p.rs:93-99  (P2PArgs defaults)
#[arg(long = "p2p.listen.ip",  default_value = "0.0.0.0")]   // <-- public bind
#[arg(long = "p2p.listen.tcp", default_value = "9222")]
#[arg(long = "p2p.ban.peers",  default_value = "false")]     // <-- banning OFF
```

### Reachability

`crates/common/chains/src/config.rs:254-257` hardcodes two Sepolia bootnodes. Both accept anonymous libp2p on 9222/tcp; probe against live in-scope hosts:

```
2026-04-22 18:44  /ip4/18.210.176.114/tcp/9222  (base)  -> publish(100 KiB snappy) -> Ok
2026-04-22 18:45  /ip4/107.21.251.55/tcp/9222   (base)  -> publish(100 KiB snappy) -> Ok
```

Fresh secp256k1 keypair, `MessageAuthenticity::Anonymous`, 100 KiB benign payload (not the bomb). TCP + Noise + yamux + gossipsub + topic-mesh admission all succeed. No credentials of any kind required.

## Impact Details

**Network not being able to confirm new transactions (total network shutdown).**

Each inbound packet costs the sequencer's single-threaded gossip task \~130 ms of CPU and \~464 MiB of transient heap (9.4 MiB wire → 200 MiB decoded, double-decompressed). 4 anonymous peers pushing \~75 Mbps aggregate (8 bombs/s, unique per message) saturate that task continuously, so the sequencer's own `publish()` — which runs on the same task — never gets a turn. Result: block-v4 propagation drops from 0.500 blocks/s to 0.000 blocks/s for the full attack window.

### Measured against the real sequencer stack

Run `measure-rate` against `DevnetBuilder::new().build()` (real L1 anvil + real base-reth-node EL + real in-process `base-consensus-node` in sequencer mode):

```
baseline:  10 blocks in 20s = 0.500 blocks/s   (matches Base's 2 s block cadence)
attack:     0 blocks in 45s = 0.000 blocks/s
drop:       100.0%
```

**Attacker capability:** 4 anonymous libp2p peers, each publishing a 200 MiB-decoded / 9.4 MiB-compressed snappy bomb every 500 ms with a per-message nonce byte. Aggregate wire \~75 Mbps. No credentials.

**Per-packet cost at the victim** (measured in `e2e-localnet`):

```
compressed on wire       : 9.38 MiB    (passes max_transmit_size)
declared decoded         : 200.00 MiB
transient heap bump      : 463.70 MiB  (both decompression sites fire)
per-message CPU          : ~130 ms (memset + decompress) on the swarm task
```

Gossipsub's behaviour task is single-threaded, so sustained delivery saturates the sequencer's swarm-task CPU. The sequencer's own `publish()` runs on that same task, so its block broadcast waits behind the attacker backlog. Measurement shows the backlog fully starves block propagation.

### Prior art

Ethereum consensus-specs' [p2p-interface](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/p2p-interface.md) mandates a `decompress_len` cap before allocation. Prysm, Lighthouse, Teku, Nimbus, and paradigmxyz/reth (RLPx path) all implement it. `base-consensus` is the only Rust-Ethereum snappy consumer on this review surface that skips it. Direct class precedent: **CVE-2023-34455** (snappy-java) and **Lodestar GHSA-53rv-hcvm-rpp9**.

## References

Task commit `de349fc9e8bf61531ce36ca57572345b03b2b097`:

* `crates/consensus/gossip/src/config.rs:13-15` — `MAX_GOSSIP_SIZE` + claimed invariant
* `crates/consensus/gossip/src/config.rs:75-101` — builder wiring only `max_transmit_size`
* `crates/consensus/gossip/src/config.rs:104-122` — `compute_message_id` (ROOT CAUSE #1)
* `crates/consensus/gossip/src/handler.rs:49-60` — dispatch to `decode_v*`
* `crates/common/rpc-types-engine/src/envelope.rs:255,299,343,399` — `decode_v1..v4` (ROOT CAUSE #2)
* `crates/consensus/gossip/src/block_validity.rs:108-115,218-230` — signer check runs post-decompress
* `crates/consensus/gossip/src/behaviour.rs:54` — `MessageAuthenticity::Anonymous`
* `crates/client/cli/src/p2p.rs:93-99` — `0.0.0.0:9222`, banning off
* `crates/common/chains/src/config.rs:254-257` — Sepolia bootnodes
* `snap-1.1.1/src/lib.rs:93` — snap's `MAX_INPUT_SIZE = u32::MAX`

External:

* Eth consensus-specs p2p-interface: <https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/p2p-interface.md>
* Snappy format spec: <https://github.com/google/snappy/blob/main/format\\_description.txt>
* Lodestar GHSA-53rv-hcvm-rpp9: <https://github.com/ChainSafe/lodestar/security/advisories/GHSA-53rv-hcvm-rpp9>
* reth RLPx size-check reference: <https://github.com/paradigmxyz/reth/blob/main/crates/net/eth-wire/src/p2pstream.rs>

## Link to Proof of Concept

<https://gist.github.com/x-qedaudit/cfce5f46348b92ced0ca8220fe13bf9d>

## Proof of Concept

Two binaries in a single Cargo crate.

### Shared `Cargo.toml`

```toml
[package]
name = "poc-tbfyfhvd"
version = "0.0.0"
edition = "2024"

[dependencies]
snap = "1"
base-common-rpc-types-engine = { git = "https://github.com/base/base.git", rev = "de349fc9e8bf61531ce36ca57572345b03b2b097", package = "base-common-rpc-types-engine" }
base-consensus-gossip        = { git = "https://github.com/base/base.git", rev = "de349fc9e8bf61531ce36ca57572345b03b2b097", package = "base-consensus-gossip" }
devnet                       = { git = "https://github.com/base/base.git", rev = "de349fc9e8bf61531ce36ca57572345b03b2b097", package = "devnet" }
libp2p = { version = "0.56", features = ["tcp", "noise", "yamux", "gossipsub", "identify", "tokio", "macros", "secp256k1"] }
tokio  = { version = "1", features = ["full"] }

[[bin]] name = "e2e-localnet" ; path = "e2e_localnet.rs"
[[bin]] name = "measure-rate" ; path = "measure_rate.rs"
```

### `e2e-localnet` — per-packet heap measurement

Real-wire demo against a victim wired with Base's production `default_config()`. Victim subscribes to all four block topics and dispatches inbound messages to `NetworkPayloadEnvelope::decode_v{1..4}` exactly like `BlockHandler::handle`. Sender publishes one bomb over real libp2p. Tracking `GlobalAlloc` reports peak heap.

```
$ gen-bomb --decoded-size 200M --output /tmp/bomb.snappy     # inline in e2e_localnet.rs
  compressed: 9.38 MiB   (< MAX_GOSSIP_SIZE)
  decoded:    200.00 MiB

$ e2e-localnet
  [victim] received 9830406 B on /optimism/1337/3/blocks, decode_v4 = Some(BrokenSszEncoding)
  heap peak during run     : 473.40 MiB
  attack-attributable bump : 463.70 MiB
```

\~49× wire-to-heap amplification. 2 × 200 MiB = both decompression sites fire before SSZ rejection.

### `measure-rate` — block-propagation halt against the official sequencer

Spins up the real devnet (L1 anvil + base-reth-node EL + in-process `base-consensus` sequencer), connects our observer to the sequencer's libp2p port, runs an attacker pool, and measures inbound block-v4 arrival rate.

```
$ measure-rate --baseline-secs 20 --attack-secs 45 --attackers 4
[measure] builder-consensus p2p_addr = /ip4/127.0.0.1/tcp/NNNNN/p2p/16Uiu2HA...
[measure] BASELINE: observing for 20s
[observer] block-v4 msg #1 … #20       (one per ~2 s, as expected)
[measure] baseline: 10 blocks in 20s = 0.500 blocks/s
[measure] ATTACK: launching 4 attackers, bomb 200 MiB decoded, interval 500 ms
[attacker 0] publish_ok=97, errs={}
[attacker 1] publish_ok=97, errs={}
[attacker 2] publish_ok=97, errs={}
[attacker 3] publish_ok=97, errs={}
[measure] attack  : 0 blocks in 45s = 0.000 blocks/s
  block-arrival-rate drop      : 100.0%
```

Each attacker uses Base's own `default_config()` and a per-message seed-byte nonce so each bomb has a unique message-id and bypasses gossipsub's duplicate cache. All 388 bombs delivered to the sequencer; zero publish errors; the sequencer's block-v4 broadcast went to zero for the full 45 s window.

Full source including the inline bomb generator, tracking allocator, and event-loop handling: [gist](https://gist.github.com/x-qedaudit/cfce5f46348b92ced0ca8220fe13bf9d).

## Mitigation

Call `snap::raw::decompress_len` first — it parses only the varu32 preamble and never allocates — and reject when the declared length exceeds `MAX_GOSSIP_SIZE`. Apply at both decompression sites.

### Canonical fix: paradigmxyz/reth (same `snap` crate, same input class)

```rust
// paradigmxyz/reth : crates/net/eth-wire/src/p2pstream.rs:34
const MAX_PAYLOAD_SIZE: usize = 16 * 1024 * 1024;   // EIP-706 snappyMaxMessage

// :437-445 (per-frame)
let decompressed_len = snap::raw::decompress_len(&bytes[1..])?;   // <-- O(1) varint check, no alloc
if decompressed_len > MAX_PAYLOAD_SIZE {
    return Poll::Ready(Some(Err(P2PStreamError::MessageTooBig {
        message_size: decompressed_len, max_size: MAX_PAYLOAD_SIZE,
    })))
}
let mut decompress_buf = BytesMut::zeroed(decompressed_len + 1);
this.decoder.decompress(&bytes[1..], &mut decompress_buf[1..])...
```

### Port to Base

**`compute_message_id`** (`crates/consensus/gossip/src/config.rs:104`):

```rust
fn compute_message_id(msg: &Message) -> MessageId {
    // <-- FIX: reject when declared decoded length exceeds MAX_GOSSIP_SIZE
    if matches!(snap::raw::decompress_len(&msg.data), Ok(n) if n > MAX_GOSSIP_SIZE) {
        let domain_invalid: Vec<u8> = vec![0x0, 0x0, 0x0, 0x0];
        return MessageId(
            sha256([domain_invalid.as_slice(), msg.data.as_slice()].concat().as_slice())[..20].to_vec(),
        );
    }
    // … existing body …
}
```

**`NetworkPayloadEnvelope::decode_v{1..4}`** (`crates/common/rpc-types-engine/src/envelope.rs:255,299,343,399`):

```rust
pub fn decode_v1(data: &[u8]) -> Result<Self, PayloadEnvelopeError> {
    // <-- FIX: cap decoded size before alloc
    let declared = snap::raw::decompress_len(data).map_err(PayloadEnvelopeError::from)?;
    if declared > MAX_DECOMPRESSED_ENVELOPE_BYTES {
        return Err(PayloadEnvelopeError::DecodedTooLarge { given: declared, max: MAX_DECOMPRESSED_ENVELOPE_BYTES });
    }
    let decompressed = snap::raw::Decoder::new().decompress_vec(data)?;
    // … existing body …
}
```

### Regression tests

* Snappy blob with 1 MiB compressed / 1 GiB declared decoded must be rejected by `compute_message_id` and every `decode_v*` with no 1 GiB allocation.
* `measure-rate` against a patched sequencer reports `drop ≤ 5%`.
