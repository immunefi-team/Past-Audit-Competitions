# 74620 bc critical unbounded snappy decompression in base gossip message id computation increases node resource consumption

**Submitted on Apr 23rd 2026 at 20:29:26 UTC by @joohhnnn8 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74620
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/base/base/releases/tag/v0.8.0-rc.15>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## Brief / Intro

Base consensus gossip computes message IDs by fully Snappy-decompressing inbound gossip message data. The configured gossip size limit is applied to the compressed wire payload, but the decompressed output is not bounded before allocation.

A malicious P2P / gossipsub peer can therefore send a valid compressed message below the 10 MiB transmit limit that expands to a much larger buffer during message ID calculation, increasing memory and CPU usage of the receiving Base gossip node before the payload is application-validated.

## Vulnerability Details

The issue is in `crates/consensus/gossip/src/config.rs`.

`MAX_GOSSIP_SIZE` is set to 10 MiB:

```rust
pub const MAX_GOSSIP_SIZE: usize = 10 * (1 << 20);
```

This value is configured as the libp2p gossipsub transmit limit:

```rust
.max_transmit_size(MAX_GOSSIP_SIZE)
.message_id_fn(compute_message_id);
```

The transmit limit bounds the message size on the wire. However, Base's custom message ID function then attempts to fully decompress valid Snappy data:

```rust
fn compute_message_id(msg: &Message) -> MessageId {
    let mut decoder = Decoder::new();
    let id = decoder.decompress_vec(&msg.data).map_or_else(
        |_| {
            let domain_invalid_snappy: Vec<u8> = vec![0x0, 0x0, 0x0, 0x0];
            sha256([domain_invalid_snappy.as_slice(), msg.data.as_slice()].concat().as_slice())
                [..20]
                .to_vec()
        },
        |data| {
            let domain_valid_snappy: Vec<u8> = vec![0x1, 0x0, 0x0, 0x0];
            sha256([domain_valid_snappy.as_slice(), data.as_slice()].concat().as_slice())[..20]
                .to_vec()
        },
    );

    MessageId(id)
}
```

`decompress_vec()` allocates the full decompressed buffer. There is no check that the decompressed size is less than `MAX_GOSSIP_SIZE`.

I verified this with a real peer-to-peer integration test, not only by calling the function directly. The victim used `base_consensus_gossip::Behaviour` with Base's `default_config()` and Base block topic subscription. The attacker was a separate libp2p gossipsub peer connected over localhost TCP.

When the attacker published a valid Snappy payload below the wire limit, the Base victim received the gossip message and produced the expected valid-snappy message ID, which proves that the victim decompressed the full payload while handling peer-delivered gossip.

## Impact Details

Selected impact:

Increasing network processing node resource consumption by at least 30% without brute force actions.

Measured result from the PoC:

| Metric                                   |                       Value |
| ---------------------------------------- | --------------------------: |
| Configured gossip transmit limit         |                      10 MiB |
| Compressed payload sent by peer          |  6,295,556 bytes / 6.00 MiB |
| Decompressed payload processed by victim | 134,217,728 bytes / 128 MiB |
| Amplification                            |                       21.3x |
| Time until victim produced message ID    |                     \~0.73s |
| Maximum RSS during timed run             |                  327,704 KB |

This is a node resource-consumption issue. I am not claiming a full network halt or proven chain-wide transaction freeze. The demonstrated impact is that a malicious gossip peer can make a receiving Base network node allocate and process data based on the decompressed size, even though the message is below the configured transmit limit on the wire.

Repeated delivery or multiple malicious peers could further increase memory / CPU pressure and degrade gossip processing for affected nodes.

No funds are directly at risk.

## References

* Asset: <https://github.com/base/base/releases/tag/v0.8.0-rc.15>
* Affected file: `crates/consensus/gossip/src/config.rs`
* Affected function: `compute_message_id`

## Proof of Concept

This PoC was run locally on an isolated test machine against `base/base` `v0.8.0-rc.15`. It does not interact with Base mainnet or any public testnet.

The test creates two real libp2p swarms over localhost TCP:

* victim: `base_consensus_gossip::Behaviour` using Base's `default_config()`
* attacker: normal libp2p gossipsub peer

The victim is subscribed to the Base block gossip topic:

`/optimism/8453/0/blocks`

The attacker publishes a valid Snappy-compressed payload:

* compressed size: 6.00 MiB
* decompressed size: 128 MiB
* below `MAX_GOSSIP_SIZE = 10 MiB` on the wire

The test then checks that the victim receives the network-delivered gossip message and that the message ID equals the Base valid-snappy hash over the decompressed payload. This confirms the vulnerable path is reached through peer-delivered gossip, not only through a direct unit test call.

Command:

```bash
cargo test -p base-consensus-gossip --test poc_h2_peer -- --nocapture
```

Output:

```
running 1 test
[poc_h2_peer] publishing compressed=6295556 bytes (6.00 MiB), decompressed=128 MiB, ratio=21.3x
[poc_h2_peer] Base Behaviour victim received network message and produced Base valid-snappy message_id after 737.758272ms
[poc_h2_peer] matching message_id proves victim decompressed 128 MiB while handling peer-delivered gossip
test test_poc_h2_real_peer_delivery_triggers_base_message_id_decompression ... ok
```

Resource measurement command:

```bash
/usr/bin/time -v cargo test -p base-consensus-gossip --test poc_h2_peer -- --nocapture
```

Relevant output:

```
Maximum resident set size (kbytes): 327704
Elapsed (wall clock) time: 0:02.47
Exit status: 0
```

Expected result:

A single malicious peer-delivered gossip message below the 10 MiB transmit limit causes the Base gossip victim to decompress and process a 128 MiB buffer during message ID computation.

Recommended fix:

Use bounded Snappy decompression for message ID calculation. Valid-snappy messages whose decompressed size exceeds `MAX_GOSSIP_SIZE` should be rejected or hashed through a bounded / streaming path before allocating the full decompressed output.
