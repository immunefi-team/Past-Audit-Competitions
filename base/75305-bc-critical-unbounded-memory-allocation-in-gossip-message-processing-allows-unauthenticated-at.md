# 75305 bc critical unbounded memory allocation in gossip message processing allows unauthenticated attacker to freeze block propagation

**Submitted on Apr 28th 2026 at 12:33:10 UTC by @HarryBarz for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75305
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

## Description

## Brief/Intro

The Base node's gossip layer decompresses every incoming network message without first checking how large the decompressed output will be. An attacker with a basic internet connection can craft a message that is under 10 MB on the wire but forces the node to allocate and process 200 MB of memory to decompress it. Because the node handles all gossip messages one at a time in a single processing loop, each crafted message freezes the entire loop for 230–545 milliseconds while the decompression runs. An attacker sending these messages continuously keeps the loop permanently occupied, preventing the node from validating or propagating legitimate blocks. On the Base network, where a new block is produced every 2 seconds, a sustained flood from a single unauthenticated attacker can delay block processing by thousands of percent above the normal block time, effectively halting the node's participation in the network.

## Vulnerability Details

The Base consensus node receives blocks from other peers over a gossip network (libp2p gossipsub). Every message sent over this network is compressed using the Snappy algorithm to save bandwidth. When a message arrives, the node must decompress it to read the contents.

The Snappy format works by including a header at the start of the compressed data that declares how large the decompressed output will be. The node reads this declared size and allocates that much memory before doing any decompression work.

The problem is that the node never checks whether the declared size is reasonable before allocating the memory.

<https://github.com/base/base/blob/1c45bc58d1f69716f5e26a5bf2947a17a86f15a6/crates/consensus/gossip/src/config.rs#L104-L122>

```rust
fn compute_message_id(msg: &Message) -> MessageId {
    let mut decoder = Decoder::new();
    let id = decoder.decompress_vec(&msg.data).map_or_else(   // // ← no size check
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

This function is called for every single message the node receives, before any validation, before any queue, before any authentication check. Its job is to compute a unique ID for the message. To do that, it fully decompresses the message, with no limit on how large the decompressed output is allowed to be.

An attacker can send a message that is 9.84 MB on the wire but decompresses to 200 MB. The node allocates 200 MB of memory, computes the hash, then frees it. This happens for every received message, and it happens before the node even decides whether to accept or reject the message.

<https://github.com/base/base/blob/1c45bc58d1f69716f5e26a5bf2947a17a86f15a6/crates/common/rpc-types-engine/src/envelope.rs#L399-L402>

```rust
    pub fn decode_v4(data: &[u8]) -> Result<Self, PayloadEnvelopeError> {
        use ssz::Decode;
        let mut decoder = snap::raw::Decoder::new();
        let decompressed = decoder.decompress_vec(data)?; // ← no size check
```

After the message ID is computed, the message is passed to the block handler which calls `decode_v4` to read the block contents. This is a second, separate 200 MB allocation for the same message. Same root cause, no size check before decompressing.

The same pattern exists in `decode_v1 (line 258)`, <https://github.com/base/base/blob/1c45bc58d1f69716f5e26a5bf2947a17a86f15a6/crates/common/rpc-types-engine/src/envelope.rs#L255-L258> `decode_v2 (line 302)`, <https://github.com/base/base/blob/1c45bc58d1f69716f5e26a5bf2947a17a86f15a6/crates/common/rpc-types-engine/src/envelope.rs#L299-L302> and `decode_v3 (line 346)` <https://github.com/base/base/blob/1c45bc58d1f69716f5e26a5bf2947a17a86f15a6/crates/common/rpc-types-engine/src/envelope.rs#L343-L346> — all four decode paths are affected.

The node is configured with `MAX_GOSSIP_SIZE = 10 MB`, intended to limit message size. However this only applies to outbound messages. The inbound size check in libp2p-gossipsub uses a per-topic lookup that returns empty for this configuration, so the check never fires on incoming messages.

The attack payload is 9.84 MB on the wire, under the 10 MB limit anyway. The vulnerability is not about the wire size. It is about what happens after the node receives a legitimately-sized message and tries to decompress it.

#### NB

`compute_message_id` is registered as the message ID function at [config.rs:93:](https://github.com/base/base/blob/1c45bc58d1f69716f5e26a5bf2947a17a86f15a6/crates/consensus/gossip/src/config.rs#L93)

```rust
.message_id_fn(compute_message_id)
```

This means gossipsub calls it automatically on every message that arrives off the wire, before any validation, before any authentication, and before any decision to accept or reject it. There is no gate between a raw TCP connection and this function.

### How the Attack Works

1. Attacker opens a TCP connection to the node's gossip port. No authentication is required, and this port is publicly accessible by design.
2. Attacker sends a Snappy-compressed message containing 9.84 MB of compressed zeros. This expands to 200 MB when decompressed (20:1 ratio).
3. Node receives the message, calls `compute_message_id`, allocates 200 MB, computes a hash, frees the memory. All of this happens inside the gossip event loop.
4. Node then calls `decode_v4` on the same message — another 200 MB allocation and free.
5. Because the event loop is single-threaded, no other events are processed during these allocations.
6. Attacker repeats continuously.

## Impact Details

### Direct Impact on the Node

The Base consensus node processes gossip messages one at a time in a single loop. Each bomb message forces the node to decompress 200 MB, which blocks the entire loop for the duration of that operation. During that freeze, the node cannot validate incoming blocks, propagate blocks to other peers, or respond to any network activity.

Base produces a block every 2,000ms. The PoC measured each bomb message blocking the event loop for 230–545ms on a local debug build. At a sustained flood rate of 2 messages per second, the event loop is frozen for 460–1,090ms out of every 1,000ms, the node spends more time processing attack messages than doing actual work, and falls permanently behind the chain tip for the duration of the attack.

### Network-Wide Amplification

The gossip network is built on a mesh topology where each node maintains connections to 8–12 peers. When any peer sends a message, gossipsub automatically forwards it to all other mesh members.

One attacker sending a single bomb message does not just affect one node; it is re-propagated across the entire mesh. Every node that receives it runs the same unbounded decompression, freezing every node's event loop simultaneously. A single attacker with a basic internet connection can degrade the entire gossip mesh with one message.

### No Authentication Required

The gossip port is publicly accessible by design. Nodes must accept connections from unknown peers to participate in the network. The attacker does not need a validator key, staked funds, a prior relationship with any node, or any special permission. Any machine on the internet that can open a TCP connection to the gossip port can trigger this.

### Severity Mapping

The Immunefi impact criteria states:

High: Temporary freezing of network transactions by delaying one block by 500% or more of the average block time

* Base block time: 2,000ms
* 500% threshold: 10,000ms cumulative delay
* At 2 bomb messages per second: event loop frozen 460–1,090ms per second
* Over a 10-second window: cumulative freeze of 4,600–10,900ms — meeting and exceeding the 10,000ms threshold
* The attack requires no special resources and can be sustained indefinitely

This vulnerability does not allow an attacker to steal funds or manipulate state. The impact is availability; the network's ability to confirm transactions is degraded or halted for the duration of the attack. Transactions submitted during the attack period will be delayed but not lost.

## References

Links in the code

## Link to Proof of Concept

<https://gist.github.com/HarryBarz/568f6ad35cf92c13d484c6d9b1241a11>

## Proof of Concept

The full POC details can be found on the gist link
