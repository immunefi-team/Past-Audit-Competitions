# 74531 bc critical gossipsub message id computation snappy decompresses untrusted payloads without a decompressed size cap

**Submitted on Apr 23rd 2026 at 09:01:58 UTC by @oxeix for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74531
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/base/base/releases/tag/v0.8.0-rc.15>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

### Brief/Intro

Every Base consensus node derives a gossipsub `message-id` by Snappy-decompressing the raw message bytes inside `compute_message_id`, which runs on the hot, pre-validation path for every inbound gossip message from every mesh peer. Because the implementation calls `snap::raw::Decoder::decompress_vec`without first checking `snap::raw::decompress_len` against `MAX_GOSSIP_SIZE`, the output buffer is sized from an attacker-controlled 1–5-byte varint in the frame header. In production this lets any peer on the public gossip mesh publish tiny messages (\~a few bytes of header) that force every receiving Base node to perform multi-gigabyte heap allocations and bulk SHA-256 hashing per message, yielding a one-to-many CPU/memory amplification that trivially clears the "≥30% increase in node resource consumption" bar and, at the extreme, OOM-kills validator / sequencer / verifier nodes network-wide.

### Vulnerability Details

Consider the code:

```rust
pub fn default_config_builder() -> ConfigBuilder {
    let mut builder = ConfigBuilder::default();
    builder
        .mesh_n(DEFAULT_MESH_D)
        .mesh_n_low(DEFAULT_MESH_DLO)
        .mesh_n_high(DEFAULT_MESH_DHI)
        .gossip_lazy(DEFAULT_MESH_DLAZY)
        .heartbeat_interval(GOSSIP_HEARTBEAT)
        .fanout_ttl(Duration::from_secs(60))
        .history_length(12)
        .history_gossip(3)
        .flood_publish(false)
        .support_floodsub()
        .max_transmit_size(MAX_GOSSIP_SIZE)
        .duplicate_cache_time(Duration::from_secs(120))
        .connection_handler_queue_len(MAX_OUTBOUND_QUEUE)
        .validation_mode(libp2p::gossipsub::ValidationMode::None)
        .validate_messages()
        .message_id_fn(compute_message_id);

    builder
}
```

`compute_message_id` is installed as the gossipsub `message_id_fn`, so libp2p-gossipsub invokes it for every received message before deduplication (the seen-cache lookup is keyed on the id it returns) and before any semantic validation:

```rust
pub fn default_config_builder() -> ConfigBuilder {
    let mut builder = ConfigBuilder::default();
    builder
        .mesh_n(DEFAULT_MESH_D)
        .mesh_n_low(DEFAULT_MESH_DLO)
        .mesh_n_high(DEFAULT_MESH_DHI)
        .gossip_lazy(DEFAULT_MESH_DLAZY)
        .heartbeat_interval(GOSSIP_HEARTBEAT)
        .fanout_ttl(Duration::from_secs(60))
        .history_length(12)
        .history_gossip(3)
        .flood_publish(false)
        .support_floodsub()
        .max_transmit_size(MAX_GOSSIP_SIZE)
        .duplicate_cache_time(Duration::from_secs(120))
        .connection_handler_queue_len(MAX_OUTBOUND_QUEUE)
        .validation_mode(libp2p::gossipsub::ValidationMode::None)
        .validate_messages()
        .message_id_fn(compute_message_id);

    builder
}
```

The function itself is:

```rust
/// Computes the [`MessageId`] of a `gossipsub` message.
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
        |data| {
            let domain_valid_snappy: Vec<u8> = vec![0x1, 0x0, 0x0, 0x0];
            sha256([domain_valid_snappy.as_slice(), data.as_slice()].concat().as_slice())[..20]
                .to_vec()
        },
    );

    MessageId(id)
}
```

The crate defines a 10 MiB cap and its doc-comment explicitly claims it bounds decompressed individual messages:

```rust
/// The maximum gossip size.
/// Limits the total size of gossip RPC containers as well as decompressed individual messages.
pub const MAX_GOSSIP_SIZE: usize = 10 * (1 << 20);
```

but the only place this constant is wired into the gossipsub config is `.max_transmit_size(MAX_GOSSIP_SIZE)`. `max_transmit_size` is a libp2p-gossipsub setting that bounds the on-wire, compressed RPC size - not what `decompress_vec` allocates after the bytes are in memory. Nothing in `compute_message_id` consults `snap::raw::decompress_len` or caps the output buffer, so the claim in the doc-comment is not enforced on this path.

The attack itself goes as following:

an attacker-published gossip message carrying only:

```
[0x80, 0x80, 0x80, 0x80, 0x0F, <minimal body>]
```

declares declared\_len ≈ 4 GiB in 5 bytes of header. Every receiving Base node, on seeing this message for dedup, runs:

1. libp2p-gossipsub transport accepts it (≪ 10 MiB, so max\_transmit\_size is satisfied).
2. `compute_message_id` is invoked.
3. `decompress_vec` reads the varint, calls vec!\[0; \~4 GiB] — a multi-gigabyte zero-initialised heap allocation that touches every page (real RSS, real wall time). Either the body is valid and the full decompressed buffer is then fed through sha256(\[0x01, 0x00, 0x00, 0x00] || data) (more CPU), or the body is invalid, the allocation is dropped, and the invalid-snappy branch is taken — but the allocation already happened.

## Impact Details

Increasing network processing node resource consumption by at least 30%

## References

*

## Proof of Concept

Add the test to the `gossip/config.rs`:

```rust
  /// Regression test for: "Gossipsub message-id generation decompresses
    /// untrusted Snappy payloads without a decompressed-size cap".
    ///
    /// `compute_message_id` is executed for every received gossip message
    /// before any semantic validation, as part of dedup / message-id
    /// derivation. The current implementation calls
    /// `snap::raw::Decoder::decompress_vec` directly, which allocates a
    /// buffer sized from the attacker-supplied uncompressed-length field
    /// in the Snappy frame. Nothing caps that value at `MAX_GOSSIP_SIZE`
    /// (the `.max_transmit_size(MAX_GOSSIP_SIZE)` builder setting only
    /// bounds the *transport* size, not the decompressed allocation), so
    /// a single peer can force every receiver to perform an oversized
    /// allocation + decompression per message — a cheap CPU/memory
    /// amplification vector.
    ///
    /// After the fix, `compute_message_id` must refuse to decompress any
    /// frame whose decompressed size would exceed `MAX_GOSSIP_SIZE` and
    /// fall through to the invalid-snappy hashing branch instead (or
    /// equivalently, decompress into a buffer capped at
    /// `MAX_GOSSIP_SIZE` and treat overflow as invalid).
    #[test]
    fn test_compute_message_id_rejects_oversized_decompressed_payload() {
        // Build a *valid* Snappy frame that decompresses to one byte over
        // MAX_GOSSIP_SIZE. The payload is highly compressible (a single
        // repeated byte), so the compressed form stays small (~hundreds of
        // KiB) and keeps the test cheap while still tripping the cap that
        // the fix must introduce.
        let oversized = MAX_GOSSIP_SIZE + 1;
        let raw = vec![0x42u8; oversized];
        let compressed = snap::raw::Encoder::new()
            .compress_vec(&raw)
            .expect("snappy encode of repeated byte must succeed");

        let declared = snap::raw::decompress_len(&compressed)
            .expect("snappy header must be well-formed");
        assert!(
            declared > MAX_GOSSIP_SIZE,
            "fixture decompressed length ({declared}) must exceed MAX_GOSSIP_SIZE ({MAX_GOSSIP_SIZE})",
        );

        let msg = Message {
            source: None,
            data: compressed.clone(),
            sequence_number: None,
            topic: libp2p::gossipsub::TopicHash::from_raw("test"),
        };

        let id = compute_message_id(&msg);

        // the oversized frame must be handled
        let invalid_domain: [u8; 4] = [0x0, 0x0, 0x0, 0x0];
        let expected_invalid = sha256(
            [invalid_domain.as_slice(), compressed.as_slice()].concat().as_slice(),
        )[..20]
            .to_vec();

        // And it must NOT match the valid-snappy hash, which is what the
        // current (vulnerable) implementation produces after performing the
        // oversized allocation + decompression
        let valid_domain: [u8; 4] = [0x1, 0x0, 0x0, 0x0];
        let vulnerable_valid_hash = sha256(
            [valid_domain.as_slice(), raw.as_slice()].concat().as_slice(),
        )[..20]
            .to_vec();

        assert_ne!(
            id.0, vulnerable_valid_hash,
            "compute_message_id must not decompress frames whose declared size exceeds MAX_GOSSIP_SIZE",
        );
        assert_eq!(
            id.0, expected_invalid,
            "oversized snappy frames must be routed through the invalid-snappy branch without allocating",
        );
    }
```
