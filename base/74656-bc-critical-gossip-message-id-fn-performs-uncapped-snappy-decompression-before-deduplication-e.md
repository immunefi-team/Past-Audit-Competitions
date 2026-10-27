# 74656 bc critical gossip message id fn performs uncapped snappy decompression before deduplication enabling p2p memory amplification dos

Submitted on Apr 24th 2026 at 04:12:33 UTC by @Kopi for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74656
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/base/base/releases/tag/v0.8.0-rc.15>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## Brief / Intro

The consensus gossip layer computes libp2p gossipsub message IDs by fully decompressing inbound Snappy payloads with `snap::raw::Decoder::decompress_vec` before any duplicate-cache check or application validation. The configured 10 MiB `MAX_GOSSIP_SIZE` limit is applied to the compressed gossip frame, but no equivalent limit is enforced on the decompressed output used during `MessageId` computation.

As a result, any peer that can connect to the default P2P gossip port can send a Snappy payload below the wire-size limit that expands to a much larger plaintext during `compute_message_id`. The success path then copies the decompressed plaintext again into the SHA-256 input. This gives an unauthenticated P2P peer a memory-amplification primitive on the hot gossip receive path, before deduplication and before the validate queue throttles apply.

## Vulnerability Details

### 1. `compute_message_id` decompresses untrusted gossip payloads with no output-size cap

`default_config_builder()` registers `compute_message_id` as the gossipsub `message_id_fn`:

```rust
.message_id_fn(compute_message_id);
```

File:

`crates/consensus/gossip/src/config.rs:93`

`compute_message_id` then attempts to fully decompress the inbound message data:

```rust
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

File:

`crates/consensus/gossip/src/config.rs:104-122`

The `snap` decoder allocates the output vector from the Snappy-declared decompressed length:

```rust
pub fn decompress_vec(&mut self, input: &[u8]) -> Result<Vec<u8>> {
    let mut buf = vec![0; decompress_len(input)?];
    let n = self.decompress(input, &mut buf)?;
    buf.truncate(n);
    Ok(buf)
}
```

File:

`snap-1.1.1/src/decompress.rs:105-109`

This means the memory allocation is driven by untrusted gossip data. On the valid-Snappy path, the code then calls `.concat()` to prepend the 4-byte domain tag, copying the decompressed bytes again before hashing.

### 2. `MAX_GOSSIP_SIZE` limits the compressed frame, not the decompressed payload

The gossip constant is documented as limiting both RPC containers and decompressed messages:

```rust
/// The maximum gossip size.
/// Limits the total size of gossip RPC containers as well as decompressed individual messages.
pub const MAX_GOSSIP_SIZE: usize = 10 * (1 << 20);
```

File:

`crates/consensus/gossip/src/config.rs:13-15`

However, the code only applies this constant as the gossipsub transmit-size limit:

```rust
.max_transmit_size(MAX_GOSSIP_SIZE)
```

File:

`crates/consensus/gossip/src/config.rs:88`

libp2p checks the serialized gossip message size against the configured max:

```rust
if self
    .max_transmit_size_for_topic(&topic)
    .is_some_and(|max| message.get_size() > max)
{
    invalid_messages.push((message, ValidationError::MessageSizeTooLargeForTopic));
    continue;
}
```

File:

`libp2p-gossipsub-0.49.4/src/protocol.rs:284-300`

This check happens before `compute_message_id`, but it only constrains the compressed message bytes carried on the wire. It does not constrain the Snappy-decompressed output size later allocated by `Decoder::decompress_vec`.

The protocol batch reader shows the safer pattern used elsewhere in the repository: decompression with an explicit output limit.

```rust
decompress_to_vec_zlib_with_limit(&data, self.max_rlp_bytes_per_channel)
```

File:

`crates/consensus/protocol/src/batch/reader.rs:97-115`

The gossip `compute_message_id` path has no equivalent output-size limit.

### 3. The vulnerable decompression happens before duplicate-cache suppression

libp2p gossipsub handles a newly received message by first applying the inbound transform and then computing the message ID:

```rust
let message = match self.data_transform.inbound_transform(raw_message.clone()) {
    Ok(message) => message,
    Err(e) => {
        ...
        return;
    }
};

let msg_id = self.config.message_id(&message);
```

File:

`libp2p-gossipsub-0.49.4/src/behaviour.rs:1775-1792`

Only after the message ID is computed does libp2p check and insert into the duplicate cache:

```rust
if !self.duplicate_cache.insert(msg_id.clone()) {
    tracing::debug!(message_id=%msg_id, "Message already received, ignoring");
    ...
    return;
}
```

File:

`libp2p-gossipsub-0.49.4/src/behaviour.rs:1825-1830`

Therefore, duplicate suppression cannot prevent the allocation. The node must decompress the attacker-controlled payload first in order to learn whether the message is a duplicate.

### 4. Validate-queue limits and application validation also happen after message-ID computation

The Base gossip config enables validation and defines validate-queue constants:

```rust
pub const MAX_VALIDATE_QUEUE: usize = 256;
pub const GLOBAL_VALIDATE_THROTTLE: usize = 512;
```

File:

`crates/consensus/gossip/src/config.rs:24-28`

But those limits apply after gossipsub has computed the message ID. The vulnerable allocation occurs before the message can be admitted to, rejected from, or throttled by the validation pipeline.

This also means block-signature validation does not protect the path. Even an invalid block payload must pass through `compute_message_id` first.

### 5. The P2P gossip port is externally reachable by default

The CLI defaults the P2P listen address and TCP port to all interfaces on port `9222`:

```rust
#[arg(long = "p2p.listen.ip", default_value = "0.0.0.0", env = "BASE_NODE_P2P_LISTEN_IP", value_parser = resolve_host)]
pub listen_ip: IpAddr;

#[arg(long = "p2p.listen.tcp", default_value = "9222", env = "BASE_NODE_P2P_LISTEN_TCP_PORT")]
pub listen_tcp_port: u16;
```

File:

`crates/client/cli/src/p2p.rs:97-101`

The network config then builds the gossip listen address from these values:

```rust
let mut gossip_address = libp2p::Multiaddr::from(self.listen_ip);
gossip_address.push(libp2p::multiaddr::Protocol::Tcp(self.listen_tcp_port));
```

File:

`crates/client/cli/src/p2p.rs:481-482`

No sequencer private key, admin RPC access, JWT, or application-layer credential is required to trigger this path. A peer only needs to complete normal libp2p connection and gossipsub message delivery to the node.

## Impact Details

This issue gives an unauthenticated P2P peer a memory-amplification denial-of-service primitive against consensus gossip nodes.

The direct exploit path is:

1. Connect to the node's libp2p gossip port, which defaults to `0.0.0.0:9222`.
2. Send a gossipsub publish message whose compressed payload is below `MAX_GOSSIP_SIZE`.
3. Choose Snappy data that expands to a much larger decompressed plaintext.
4. Force `compute_message_id` to allocate the full decompressed output before deduplication.
5. On the valid-Snappy path, force an additional copy of the decompressed output through `.concat()` before hashing.
6. Repeat with distinct messages so the duplicate cache cannot suppress the workload.

The demonstrated amplification is:

```
compressed   = 1,524,708 bytes
decompressed = 32,505,856 bytes
```

This is a roughly 21x expansion before accounting for the additional `.concat()` copy. Depending on host memory, peer concurrency, and message rate, repeated payloads can cause severe memory pressure, allocator churn, gossip task stalls, or process termination.

## References

`crates/consensus/gossip/src/config.rs:13-15`

`crates/consensus/gossip/src/config.rs:24-28`

`crates/consensus/gossip/src/config.rs:88-93`

`crates/consensus/gossip/src/config.rs:104-122`

`crates/client/cli/src/p2p.rs:97-101`

`crates/client/cli/src/p2p.rs:481-482`

`crates/consensus/protocol/src/batch/reader.rs:97-115`

`libp2p-gossipsub-0.49.4/src/protocol.rs:284-300`

`libp2p-gossipsub-0.49.4/src/behaviour.rs:1775-1792`

`libp2p-gossipsub-0.49.4/src/behaviour.rs:1825-1830`

`snap-1.1.1/src/decompress.rs:105-109`

## Link to Proof of Concept

<https://gist.github.com/sebastian-osec/016936985ecfe8485bb5c788bff8e489>

## Proof of Concept

## Goal

Demonstrate that:

1. a compressed gossip payload below the configured 10 MiB `MAX_GOSSIP_SIZE` wire limit can decompress to a much larger plaintext,
2. the real configured `message_id_fn` fully decompresses that oversized plaintext,
3. the resulting `MessageId` is computed from the full decompressed payload, proving that no decompressed-size limit is enforced, and
4. this occurs in the same callback that libp2p invokes before duplicate-cache insertion.

## PoC artifacts

The attached PoCs exercise the issue at two levels:

1. a direct unit test that invokes the production `message_id_fn` through `default_config().message_id(...)`, and
2. a local two-peer libp2p gossipsub network test where an attacker swarm publishes the oversized Snappy payload over loopback TCP and the victim swarm computes the received `MessageId` from the full decompressed payload.

PoC file:

`crates/consensus/gossip/src/config.rs`

PoC tests:

`test_message_id_fn_decompresses_payloads_larger_than_max_gossip_size`

`test_local_gossipsub_network_computes_message_id_from_oversized_snappy_payload`

## Run command

The workspace defaults to `mold` for linking. In this environment `mold` was not installed, so I used `lld`:

```bash
CARGO_TARGET_X86_64_UNKNOWN_LINUX_GNU_RUSTFLAGS='-C link-arg=-fuse-ld=lld' \
cargo test -p base-consensus-gossip \
  test_message_id_fn_decompresses_payloads_larger_than_max_gossip_size -- --nocapture
```

The local network PoC can be run with:

```bash
CARGO_TARGET_X86_64_UNKNOWN_LINUX_GNU_RUSTFLAGS='-C link-arg=-fuse-ld=lld' \
cargo test -p base-consensus-gossip \
  test_local_gossipsub_network_computes_message_id_from_oversized_snappy_payload -- --nocapture
```

## Observed result

Direct `message_id_fn` PoC:

```
running 1 test
compressed=1524708 decompressed=32505856
test config::tests::test_message_id_fn_decompresses_payloads_larger_than_max_gossip_size ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 44 filtered out
```

Local two-peer gossipsub network PoC:

```
running 1 test
test config::tests::test_local_gossipsub_network_computes_message_id_from_oversized_snappy_payload ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 45 filtered out
```

I also ran the full gossip crate test suite:

```bash
CARGO_TARGET_X86_64_UNKNOWN_LINUX_GNU_RUSTFLAGS='-C link-arg=-fuse-ld=lld' \
cargo test -p base-consensus-gossip
```

Observed result:

```
test result: ok. 46 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## What the PoC proves

{% stepper %}
{% step %}

## Builds a Snappy plaintext of `MAX_GOSSIP_SIZE * 3 + 1 MiB`, which is `32,505,856` bytes.

The direct unit test does this to create an oversized plaintext for compression.
{% endstep %}

{% step %}

## Compresses it with `snap::raw::Encoder`.

This produces the Snappy payload used in the test.
{% endstep %}

{% step %}

## Confirms the compressed payload is only `1,524,708` bytes, which is below the 10 MiB gossip transmit-size limit.

This shows the wire payload stays under `MAX_GOSSIP_SIZE`.
{% endstep %}

{% step %}

## Confirms the decompressed length is larger than `MAX_GOSSIP_SIZE`.

This verifies the plaintext expands beyond the configured gossip size.
{% endstep %}

{% step %}

## Invokes the real configured gossipsub message-ID callback with `default_config().message_id(&test_message(compressed))`.

This calls the production `message_id_fn` directly.
{% endstep %}

{% step %}

## Computes the expected valid-Snappy message ID as:

```
sha256(0x01_00_00_00 || full_decompressed_payload)[..20]
```

This is the ID format used for the valid-Snappy path.
{% endstep %}

{% step %}

## Confirms the callback returns that exact ID.

The assertion only passes if `compute_message_id` fully decompresses and hashes the oversized plaintext. This validates the core issue: the gossip path enforces the compressed wire-size limit but does not enforce a decompressed-size limit before allocating memory and computing the deduplication key.
{% endstep %}
{% endstepper %}

The local network test then validates the same behavior over an actual libp2p connection:

1. Starts a victim gossipsub swarm with the production `default_config()`.
2. Starts an attacker gossipsub swarm with a separate, self-generated libp2p identity.
3. Makes the victim listen on loopback TCP and has the attacker dial it.
4. Waits for the peers to connect and exchange topic subscriptions.
5. Publishes the same oversized Snappy payload from the attacker swarm.
6. Confirms the victim receives a gossipsub message whose `MessageId` equals:

```
sha256(0x01_00_00_00 || full_decompressed_payload)[..20]
```

That assertion only passes if the victim's real gossipsub receive path invokes the configured `message_id_fn` and computes the ID from the oversized decompressed payload before yielding the message to the application.

## Minimal attack shape

A live-network attacker would send gossipsub publish messages with payloads shaped like the PoC payload:

```
snappy_compressed_payload = snappy_compress(repeated_bytes_larger_than_MAX_GOSSIP_SIZE)
assert len(snappy_compressed_payload) < MAX_GOSSIP_SIZE

publish_to_gossipsub_topic(snappy_compressed_payload)
```

No sequencer signature is needed to reach the vulnerable allocation, because signature and block validation occur after message-ID computation.
