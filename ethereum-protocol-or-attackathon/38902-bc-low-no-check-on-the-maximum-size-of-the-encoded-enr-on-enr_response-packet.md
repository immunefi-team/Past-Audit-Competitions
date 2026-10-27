# #38902 \[BC-Low] No check on the maximum size of the encoded ENR on ENR\_RESPONSE packet

**Submitted on Jan 17th 2025 at 13:39:03 UTC by @Franfran for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38902
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/hyperledger/besu>
* **Impacts:**
  * (Specifications) A bug in specifications with no direct impact on client implementations

## Description

## Brief/Intro

When asking for an Ethereum Node Record (ENR) of another node, this one should respond with a packet containing this information.\
It is specified that ENR should be long of at most 300 bytes but this check is missing when decoding these packets.

## Vulnerability Details

In the [EIP-778](https://eips.ethereum.org/EIPS/eip-778#rlp-encoding) defining ENR, it is stated that "The maximum encoded size of a node record is 300 bytes. Implementations should reject records larger than this size."\
When receiving an `ENR_RESPONSE` packet, the packet is first deserialized with the [`ENRResponsePacketData::readFrom` function](https://github.com/hyperledger/besu/blob/98383c5777ef54007a0e3cb0d51ea9016c6e6d27/ethereum/p2p/src/main/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/PacketType.java#L31).\
This function RLP decodes the packet data from raw RLP by using the [discovery library](https://github.com/Consensys/discovery) by calling its function [fromBytes](https://github.com/hyperledger/besu/blob/a052148dfd08fa9d3713a05d9c04b6cdb4135480/ethereum/p2p/src/main/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/ENRResponsePacketData.java#L49) but this library contains a bug.\
Indeed, it doesn't check if the RLP-encoded packet data size exceeds 300 bytes.[Here is the definition of the `fromBytes` function](https://github.com/Consensys/discovery/blob/9bb8ad3942f81450fe54a78c66f0b61e58c45b0b/src/main/java/org/ethereum/beacon/discovery/schema/NodeRecordFactory.java#L76) which seems to check for the [size of some data not exceeding 300 bytes](https://github.com/Consensys/discovery/blob/9bb8ad3942f81450fe54a78c66f0b61e58c45b0b/src/main/java/org/ethereum/beacon/discovery/schema/NodeRecordFactory.java#L97), but this is only checking that the value of the `id` in the key-value store of the ENR does not exceed 300 bytes. This seems to be a misunderstanding with the specification, that says that the RLP-encoded record should not exceed 300 bytes.\
To conclude, a peer can send an unexpectedly big ENR to our node when responding with the `ENRResponse` packet and it's going to be accepted and stored.

## Impact Details

This is a drift from the specification and will make the implementation waste resources by allowing huge UDP packets of the `ENRResponse` as well as storing abnormally big records on disk.

## References

Links added where applicable

## Proof of Concept

Allow serialization or the ENR without limit check in the call to `Packet.create` in order to build a simple POC

```diff
diff --git a/ethereum/p2p/src/main/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/ENRResponsePacketData.java b/ethereum/p2p/src/main/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/ENRResponsePacketData.java
index d71c7e1e0..642dc603c 100644
--- a/ethereum/p2p/src/main/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/ENRResponsePacketData.java
+++ b/ethereum/p2p/src/main/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/ENRResponsePacketData.java
@@ -55,7 +55,8 @@ public class ENRResponsePacketData implements PacketData {
   public void writeTo(final RLPOutput out) {
     out.startList();
     out.writeBytes(requestHash);
-    out.writeRLPBytes(enr.serialize());
+//    out.writeRLPBytes(enr.serialize());
+    out.writeRLPBytes(enr.asRlp());
     out.endList();
   }
```

Add this test

```diff
diff --git a/ethereum/p2p/src/test/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/PeerDiscoveryControllerTest.java b/ethereum/p2p/src/test/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/PeerDiscoveryControllerTest.java
index 4005673e3..3d09f6d21 100644
--- a/ethereum/p2p/src/test/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/PeerDiscoveryControllerTest.java
+++ b/ethereum/p2p/src/test/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/PeerDiscoveryControllerTest.java
@@ -16,6 +16,7 @@ package org.hyperledger.besu.ethereum.p2p.discovery.internal;
 
 import static com.google.common.base.Preconditions.checkNotNull;
 import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatNoException;
 import static org.assertj.core.api.Assertions.assertThatThrownBy;
 import static org.mockito.ArgumentMatchers.any;
 import static org.mockito.ArgumentMatchers.anyLong;
@@ -1575,6 +1576,30 @@ public class PeerDiscoveryControllerTest {
     verify(controller, times(1)).connectOnRlpxLayer(eq(maybePeer.get()));
   }
 
+  @Test
+  public void canRespondWithBigENR() {
+    final List<NodeKey> nodeKeys = PeerDiscoveryTestHelper.generateNodeKeys(1);
+    final NodeKey nodeKey = nodeKeys.getFirst();
+    final List<DiscoveryPeer> peers = helper.createDiscoveryPeers(nodeKeys);
+    final DiscoveryPeer sender = peers.getFirst();
+
+    final NodeRecord nodeRecord = createNodeRecord(nodeKey, true);
+    nodeRecord.set("garbage1", Bytes.of(new byte[100]));
+    nodeRecord.set("garbage2", Bytes.of(new byte[100]));
+    assertThat(nodeRecord.asRlp().size()).isGreaterThan(300);
+
+    final Bytes requestHash = Bytes.of(0x00, 0x01, 0x02, 0x03);
+    final ENRResponsePacketData enrResponsePacketData =
+            ENRResponsePacketData.create(
+                    requestHash, nodeRecord);
+
+    prepareForForkIdCheck(nodeKeys, sender, false);
+    final Packet enrPacket =
+            Packet.create(PacketType.ENR_RESPONSE, enrResponsePacketData, nodeKey);
+
+    assertThatNoException().isThrownBy(() -> controller.onMessage(enrPacket, sender));
+  }
+
   @Nonnull
   private Packet prepareForForkIdCheck(
       final List<NodeKey> nodeKeys, final DiscoveryPeer sender, final boolean sendForkId) {
```

***

If you're worried about the patch to the `writeTo` making the POC possible, revert this patch and instead build the packet from the RLP data, which doesn't check the size of the ENR record as expected.

```diff
diff --git a/ethereum/p2p/src/main/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/Packet.java b/ethereum/p2p/src/main/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/Packet.java
index e7e113bfb..6d39702ab 100644
--- a/ethereum/p2p/src/main/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/Packet.java
+++ b/ethereum/p2p/src/main/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/Packet.java
@@ -54,12 +54,11 @@ public class Packet {
   private final SECPSignature signature;
   private final SECPPublicKey publicKey;
 
-  private Packet(final PacketType type, final PacketData data, final NodeKey nodeKey) {
+  private Packet(final PacketType type, final PacketData data, final NodeKey nodeKey, final Bytes dataBytes) {
     this.type = type;
     this.data = data;
 
     final Bytes typeBytes = Bytes.of(this.type.getValue());
-    final Bytes dataBytes = RLP.encode(this.data::writeTo);
 
     this.signature = nodeKey.sign(keccak256(Bytes.wrap(typeBytes, dataBytes)));
     this.hash = keccak256(Bytes.concatenate(encodeSignature(signature), typeBytes, dataBytes));
@@ -96,7 +95,12 @@ public class Packet {
 
   public static Packet create(
       final PacketType packetType, final PacketData packetData, final NodeKey nodeKey) {
-    return new Packet(packetType, packetData, nodeKey);
+    return new Packet(packetType, packetData, nodeKey, RLP.encode(packetData::writeTo));
+  }
+
+  public static Packet create2(
+          final PacketType packetType, final PacketData packetData, final NodeKey nodeKey, final Bytes dataBytes) {
+    return new Packet(packetType, packetData, nodeKey, dataBytes);
   }
 
   public static Packet decode(final Buffer message) {
```

And the updated test:

```diff
diff --git a/ethereum/p2p/src/test/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/PeerDiscoveryControllerTest.java b/ethereum/p2p/src/test/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/PeerDiscoveryControllerTest.java
index 4005673e3..2cd679f9c 100644
--- a/ethereum/p2p/src/test/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/PeerDiscoveryControllerTest.java
+++ b/ethereum/p2p/src/test/java/org/hyperledger/besu/ethereum/p2p/discovery/internal/PeerDiscoveryControllerTest.java
@@ -16,6 +16,7 @@ package org.hyperledger.besu.ethereum.p2p.discovery.internal;
 
 import static com.google.common.base.Preconditions.checkNotNull;
 import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatNoException;
 import static org.assertj.core.api.Assertions.assertThatThrownBy;
 import static org.mockito.ArgumentMatchers.any;
 import static org.mockito.ArgumentMatchers.anyLong;
@@ -45,6 +46,10 @@ import org.hyperledger.besu.ethereum.p2p.permissions.PeerPermissions;
 import org.hyperledger.besu.ethereum.p2p.permissions.PeerPermissions.Action;
 import org.hyperledger.besu.ethereum.p2p.permissions.PeerPermissionsDenylist;
 import org.hyperledger.besu.ethereum.p2p.rlpx.RlpxAgent;
+import org.hyperledger.besu.ethereum.rlp.BytesValueRLPOutput;
+import org.hyperledger.besu.ethereum.rlp.RLP;
+import org.hyperledger.besu.ethereum.rlp.RLPInput;
+import org.hyperledger.besu.ethereum.rlp.RLPOutput;
 import org.hyperledger.besu.metrics.noop.NoOpMetricsSystem;
 
 import java.time.Instant;
@@ -1575,6 +1580,38 @@ public class PeerDiscoveryControllerTest {
     verify(controller, times(1)).connectOnRlpxLayer(eq(maybePeer.get()));
   }
 
+  @Test
+  public void canRespondWithBigENR() {
+    final List<NodeKey> nodeKeys = PeerDiscoveryTestHelper.generateNodeKeys(1);
+    final NodeKey nodeKey = nodeKeys.getFirst();
+    final List<DiscoveryPeer> peers = helper.createDiscoveryPeers(nodeKeys);
+    final DiscoveryPeer sender = peers.getFirst();
+
+    /* NodeRecord nodeRecord = createNodeRecord(nodeKey, true);
+    nodeRecord.set("garbage1", Bytes.of(new byte[100]));
+    nodeRecord.set("garbage2", Bytes.of(new byte[100]));
+    assertThat(nodeRecord.asRlp().size()).isGreaterThan(300);
+
+    final Bytes requestHash = Bytes.of(0x00, 0x01, 0x02, 0x03);
+    final ENRResponsePacketData enrResponsePacketData =
+            ENRResponsePacketData.create(
+                    requestHash, nodeRecord);
+    final BytesValueRLPOutput out = new BytesValueRLPOutput();
+    enrResponsePacketData.writeTo(out);
+    final Bytes packetData = out.encoded();
+    System.err.println(packetData.toHexString());*/
+
+    final Bytes enrDataRlp = Bytes.fromHexString("0xf901808400010203f90178b84022416f7c1a2339b553d3849b6da9a6572898d41273fd569613c10afe5504ca2306a698da3d8b4941d017ec7e66bc69902313e2c5c322caf3d4c56c840c2cfd4a0183657468cac984fc64ec0483118c30886761726261676531b86400000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000886761726261676532b86400000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000826964827634826970847f00000189736563703235366b31a102839d567c7a7b9886c64229d62c6839075e7574153b8346bc84ad841b4afd7c0e83746370821ed283756470821307");
+    final ENRResponsePacketData enrResponsePacketData = ENRResponsePacketData.readFrom(RLP.input(enrDataRlp));
+    assertThat(enrResponsePacketData.getEnr().asRlp().size()).isGreaterThan(300);
+
+    prepareForForkIdCheck(nodeKeys, sender, false);
+    final Packet enrPacket =
+            Packet.create2(PacketType.ENR_RESPONSE, enrResponsePacketData, nodeKey, enrDataRlp);
+
+    assertThatNoException().isThrownBy(() -> controller.onMessage(enrPacket, sender));
+  }
+
   @Nonnull
   private Packet prepareForForkIdCheck(
       final List<NodeKey> nodeKeys, final DiscoveryPeer sender, final boolean sendForkId) {
```
