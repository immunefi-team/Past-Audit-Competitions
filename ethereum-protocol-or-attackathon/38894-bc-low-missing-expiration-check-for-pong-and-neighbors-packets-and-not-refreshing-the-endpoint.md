# #38894 \[BC-Low] Missing expiration check for Pong and Neighbors packets and not refreshing the endpoint proof

**Submitted on Jan 17th 2025 at 10:01:17 UTC by @Franfran for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38894
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/hyperledger/besu>
* **Impacts:**
  * (Specifications) A bug in specifications with no direct impact on client implementations

## Description

## Brief/Intro

There is an inconsistency with the discv4 specification regarding the `Pong` and `Neighbors` packet. Indeed, they contain an `expiration` field which should be checked according to the current time but is not. This makes Besu accepting outdated `Pong` and `Neighbors` packets from unresponsive or modified nodes, and might create a bonding with a now undesired connection after an unreasonable amount of time was elapsed with no bonding. Additionally, `Besu` never initiates a new bonding by itself even if it's outdated, making it a potential bad peer for other node implementations.

## Vulnerability Details

In the discv4 specification, the `Pong` and `Neighbors` packets contains an `expiration` field.\
<https://github.com/ethereum/devp2p/blob/master/discv4.md#pong-packet-0x02\\>
It is later explained in the ENRRequest packet specification the following: `The expiration field is an absolute UNIX time stamp. Packets containing a time stamp that lies in the past are expired may not be processed.` It is also explained that it is a [(minimal) protection against packet replay](https://github.com/ethereum/devp2p/blob/master/discv4.md#known-issues-in-the-current-version) `The "expiration" field present in all packets is supposed to prevent packet replay. Since it is an absolute time stamp, the node's clock must be accurate to verify it correctly.`

It seems that this is done for the `Ping` packet in <https://github.com/hyperledger/besu/blob/9c80c9bf42fea402f9bfeace7069ab76b2dc982f/ethereum/p2p/src/main/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/PeerDiscoveryController.java#L662-L665>, for the `FindNode` in <https://github.com/hyperledger/besu/blob/9c80c9bf42fea402f9bfeace7069ab76b2dc982f/ethereum/p2p/src/main/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/PeerDiscoveryController.java#L678-L680>, and for the `ENRRequest` in <https://github.com/hyperledger/besu/blob/9c80c9bf42fea402f9bfeace7069ab76b2dc982f/ethereum/p2p/src/main/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/PeerDiscoveryController.java#L695-L699.\\>
But not for the other packets containing an `expiration` field like `Pong`, and `Neighbors`.

### Other client implementations are handling the `expiration` field

#### Reth

1. Pong: <https://github.com/paradigmxyz/reth/blob/c5ab4243e74f66d06a14710d194d580faab938e2/crates/net/discv4/src/lib.rs#L1250-L1252>
2. Neighbors: <https://github.com/paradigmxyz/reth/blob/c5ab4243e74f66d06a14710d194d580faab938e2/crates/net/discv4/src/lib.rs#L1372-L1374\\>
   Reth also avoids to accumulate too much pending `Ping` requests by evicting old ones at a regular interval: <https://github.com/paradigmxyz/reth/blob/c5ab4243e74f66d06a14710d194d580faab938e2/crates/net/discv4/src/lib.rs#L1816> which means that if the `Pong` packet arrives after that time, the `Ping` will be missing and it won't complete the bonding: <https://github.com/paradigmxyz/reth/blob/c5ab4243e74f66d06a14710d194d580faab938e2/crates/net/discv4/src/lib.rs#L1265>.

#### Geth

1. Pong: <https://github.com/ethereum/go-ethereum/blob/6b61b54dc7f69cd091dcf3094ce19f26477f55a2/p2p/discover/v4\\_udp.go#L712-L714>
2. Neighbors: <https://github.com/ethereum/go-ethereum/blob/6b61b54dc7f69cd091dcf3094ce19f26477f55a2/p2p/discover/v4\\_udp.go#L776-L778>

### Besu does not initiate new bonding refreshing requests

In order to keep the [endpoint proof alive](https://github.com/ethereum/devp2p/blob/master/discv4.md#endpoint-proof), a peer need to have sent a valid `Pong` packet in the last 12 hours.\
Currently, Besu only relies on other node implementations to keep those endpoint proofs alive, which might be considered as an adversarial behavior for other nodes. Other implementations could stop initiating the bonding when needed and wait 12h + 1s as an argument to consider their Besu peer as bad. Indeed, the only places where Besu initiates a bonding is via the `bond` function and is either called when:

1. [A maintained connection peer is added](https://github.com/hyperledger/besu/blob/01126c0853e5a1152e760b4a5d1aa3862301e1c8/ethereum/p2p/src/main/java/org/hyperledger/besu/ethereum/p2p/network/DefaultP2PNetwork.java#L342)
2. [Refreshing the DNS record](https://github.com/hyperledger/besu/blob/01126c0853e5a1152e760b4a5d1aa3862301e1c8/ethereum/p2p/src/main/java/org/hyperledger/besu/ethereum/p2p/discovery/dns/DNSDaemon.java#L103), which calls [this](https://github.com/hyperledger/besu/blob/01126c0853e5a1152e760b4a5d1aa3862301e1c8/ethereum/p2p/src/main/java/org/hyperledger/besu/ethereum/p2p/network/DefaultP2PNetwork.java#L377)
3. [Receiving a `Ping` packet](https://github.com/hyperledger/besu/blob/9c80c9bf42fea402f9bfeace7069ab76b2dc982f/ethereum/p2p/src/main/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/PeerDiscoveryController.java#L329) which is what is called when another implementation refreshes the bonding by itself
4. [A new bonding round has started](https://github.com/hyperledger/besu/blob/9c80c9bf42fea402f9bfeace7069ab76b2dc982f/ethereum/p2p/src/main/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/RecursivePeerRefreshState.java#L123) but this [only applies to new peers, which are just `KNOWN`](https://github.com/hyperledger/besu/blob/9c80c9bf42fea402f9bfeace7069ab76b2dc982f/ethereum/p2p/src/main/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/RecursivePeerRefreshState.java#L334)

## Impact Details

Not handling the `expiration` field in the `Pong` and `Neighbors` messages means that outdated messages may be handled from unresponsive or malicious nodes which doesn't prevent packet replay.\
Not refreshing the bonding state breaks the endpoint proof according to the discv4 specifications, and might make Besu a bad peer for other node implementations.

## References

Links are provided when applicable.

## Fix proposal

1. Do not process `Pong` or `Neighbors` packets if the expiration is outdated compared to the local clock.
2. Do not process `Pong` or `Neighbors` packets if the matching `Ping` or `FindNeighbors` was sent too much time ago. You could use the reth implementation although it makes it impossible to differentiate an uninitiated `Ping`/`Pong` or `FindNeighbors`/`Neighbors` sequence and a timed-out response.
3. Attach an expiry to every bonds with other peers. If there is the need to send a packet like `FindNeighbors` or `EnrRequest` and that the last `Pong` packet was received more than 12 hours ago, initiate a new `Ping` / `Pong` sequence in order to refresh the bonding.

## Proof of Concept

## Proof of Concept

### Old `Pong` packets are processed even if they are expired

```diff
diff --git a/ethereum/p2p/src/main/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/PongPacketData.java b/ethereum/p2p/src/main/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/PongPacketData.java
index dd832c170..58cf7703c 100644
--- a/ethereum/p2p/src/main/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/PongPacketData.java
+++ b/ethereum/p2p/src/main/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/PongPacketData.java
@@ -54,6 +54,11 @@ public class PongPacketData implements PacketData {
     return new PongPacketData(to, pingHash, PacketData.defaultExpiration(), enrSeq);
   }
 
+  public static PongPacketData create2(
+          final Endpoint to, final Bytes pingHash, final long expiration, final UInt64 enrSeq) {
+    return new PongPacketData(to, pingHash, expiration, enrSeq);
+  }
+
   public static PongPacketData readFrom(final RLPInput in) {
     in.enterList();
     final Endpoint to = Endpoint.decodeStandalone(in);
```

Apply the current patch which is a slightly modified version of the above test named `bootstrapPeersPongReceived_HashMatched`.

```diff
diff --git a/ethereum/p2p/src/test/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/PeerDiscoveryControllerTest.java b/ethereum/p2p/src/test/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/PeerDiscoveryControllerTest.java
index 4005673e3..13727ff54 100644
--- a/ethereum/p2p/src/test/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/PeerDiscoveryControllerTest.java
+++ b/ethereum/p2p/src/test/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/PeerDiscoveryControllerTest.java
@@ -345,7 +345,7 @@ public class PeerDiscoveryControllerTest {
   }
 
   @Test
-  public void bootstrapPeersPongReceived_HashMatched() {
+  public void bootstrapPeersExpiredPongReceived_HashMatched() {
     // Create peers.
     final List<NodeKey> nodeKeys = PeerDiscoveryTestHelper.generateNodeKeys(3);
     final List<DiscoveryPeer> peers = helper.createDiscoveryPeers(nodeKeys);
@@ -378,7 +378,7 @@ public class PeerDiscoveryControllerTest {
     // Simulate PONG messages from all peers
     for (int i = 0; i < 3; i++) {
       final PongPacketData packetData =
-          PongPacketData.create(localPeer.getEndpoint(), mockPacket.getHash(), UInt64.ONE);
+              PongPacketData.create2(localPeer.getEndpoint(), mockPacket.getHash(), 1234, UInt64.ONE);
       final Packet packet0 = Packet.create(PacketType.PONG, packetData, nodeKeys.get(i));
       controller.onMessage(packet0, peers.get(i));
     }
```

The `timeout` was overidden thanks to the new exposed `create2` method and set to some outdated value, here `1234` (it expects an `UNIX` timestamp).
