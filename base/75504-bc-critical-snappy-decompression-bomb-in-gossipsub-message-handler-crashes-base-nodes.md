# 75504 bc critical snappy decompression bomb in gossipsub message handler crashes base nodes

Submitted on Apr 29th 2026 at 14:21:53 UTC by @DeltaXV for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75504
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * RPC API crash affecting programs with greater than or equal to 25% of the market capitalization on top of the respective layer
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

## Description

### Brief/Intro

`base-consensus` does not validate the declared decompressed size of snappy-compressed gossipsub messages before decompressing them. The OP implementation for instance checks and ensures `DecodedLen <= maxGossipSize` before decompression. With current implementation an attacker can send malformed p2p messages that validly decompresses to 191 MB, blocking the event loop crashing the P2P RPC and stalling the block processing for 30+ seconds.

### Vulnerability Details

`base-consensus` does not validate the declared decompressed size of snappy-compressed gossipsub messages before decompressing them. The OP reference implementation checks `DecodedLen <= maxGossipSize` before decompression — Base omits this guard. An attacker can send a 9 MB gossip message that validly decompresses to 191 MB, blocking the event loop, stalling block delivery for 28+ seconds, and crashing the P2P RPC.

Every inbound gossipsub message passes through `compute_message_id` in [`config.rs`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/gossip/src/config.rs), which calls `snap::raw::Decoder::decompress_vec()` unconditionally:

```rust
fn compute_message_id(msg: &Message) -> MessageId {
    let mut decoder = Decoder::new();
    let id = decoder.decompress_vec(&msg.data).map_or_else(
        |_| { /* hash raw data */ },
        |data| { /* hash decompressed data */ },
    );
    MessageId(id)
}
```

The `snap` crate (v1.1.1) reads a varint header from the compressed input declaring the output size, allocates a `Vec<u8>` of that size, then decompresses into it. Snappy achieves a consistent 21:1 compression ratio on identical-byte data, so 9 MB of compressed input produces 191 MB of valid decompressed output where every page is written to.

`compute_message_id` is registered as the `message_id_fn` on the gossipsub config (line 93). libp2p-gossipsub v0.49.4 calls it **synchronously** inside `handle_received_message` on the main event loop — there is no async yield point. The entire tokio task blocks while 191 MB is allocated, decompressed, and then SHA-256 hashed.

After `compute_message_id` returns, the message is delivered to the `BlockHandler`, which calls `decode_v1`–`decode_v4` in [`envelope.rs`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/common/rpc-types-engine/src/envelope.rs). These functions call `decompress_vec` a **second time** on the same data, doubling the work.

While the event loop is blocked processing bomb messages, legitimate gossip blocks from the sequencer cannot be received or forwarded to the engine. The P2P RPC endpoint, which shares the same tokio runtime, becomes unresponsive.

The OP reference implementation (`op-node/p2p/gossip.go`) guards both paths:

**Message ID function** (line 123):

```go
dLen, err := snappy.DecodedLen(pmsg.Data)
if err == nil && dLen <= maxGossipSize { /* only then decompress */ }
```

**Block validator** (line 289):

```go
if outLen > maxGossipSize {
    log.Warn("possible snappy zip bomb, decoded length is too large")
    return pubsub.ValidationReject
}
```

Base omits both guards.

### Impact Details

A single attacker can repeatedly crash the P2P RPC of any base-consensus node and delay block processing by 30+ seconds by sending crafted gossipsub messages from the mesh.

## Link to Proof of Concept

<https://gist.github.com/DeltaXV/f08fdfcd01d934a6e2cbdfbb62030aa9>

## Proof of Concept

{% stepper %}
{% step %}

### Start the devnet

```bash
cd base
just -f etc/docker/Justfile up-single
```

{% endstep %}

{% step %}

### Wait for all containers to be healthy

```bash
docker compose --env-file etc/docker/devnet-env \
  -f etc/docker/docker-compose.yml ps
```

Create a `snappy_bomb/monitor.sh` file and paste the script from the gist. <https://gist.github.com/DeltaXV/f08fdfcd01d934a6e2cbdfbb62030aa9>
{% endstep %}

{% step %}

### Run the monitor in a separate terminal

```bash
./snappy_bomb/monitor.sh
```

Create the `snappy_bomb/src/main.rs` file and paste the following code from the gist and do the same for `snappy_bomb/cargo.toml`: <https://gist.github.com/DeltaXV/f08fdfcd01d934a6e2cbdfbb62030aa9>
{% endstep %}

{% step %}

### Build the PoC

```bash
cd snappy_bomb && cargo build --release
```

{% endstep %}

{% step %}

### Run the attack

5 concurrent peers, 5 messages each:

```bash
for i in 1 2 3 4 5; do
  ./target/release/snappy_bomb_poc "/ip4/127.0.0.1/tcp/8003" 191 5 &
done
```

{% endstep %}

{% step %}

### Observe the monitor

The unsafe head stalls for 28+ seconds and the RPC becomes unresponsive:

Here's a video reproducing the exploit -> <https://mega.nz/file/if4zxSxC#C1HGlnYoWhm6Yri8fNG5etJD1-GiizmTD2hzLZjaNSU>

```
TIME       FDs      PEERS        UNSAFE_HEAD
---------- ------------ -------- --------------
16:01:09   32       6            215
16:01:11   32       6            216
16:01:13   32       6            217
16:01:15   32       6            218
16:01:17   33       6            219
16:01:19   32       ERR          220
16:01:24   32       ERR          220 STALLED
16:01:28   32       ERR          220 STALLED
16:01:32   32       ERR          220 STALLED
16:01:37   32       ERR          220 STALLED
16:01:41   32       ERR          220 STALLED
16:01:45   32       ERR          220 STALLED
16:01:49   32       ERR          220 STALLED
^C
```

Each attacker sends a valid gossipsub message containing 9 MB of snappy-compressed identical bytes. The decompression succeeds, the event loop stalls, and legitimate blocks cannot be processed.
{% endstep %}
{% endstepper %}

### Remediation

Add a `decompress_len` check before calling `decompress_vec`, matching the OP reference:

```rust
fn compute_message_id(msg: &Message) -> MessageId {
    let decompressed_ok = snap::raw::decompress_len(&msg.data)
        .map(|len| len <= MAX_GOSSIP_SIZE)
        .unwrap_or(false);

    if decompressed_ok {
        // safe to decompress
    } else {
        // hash raw compressed data (invalid snappy domain)
    }
}
```

Apply the same guard to `decode_v1` through `decode_v4` in `envelope.rs`.
