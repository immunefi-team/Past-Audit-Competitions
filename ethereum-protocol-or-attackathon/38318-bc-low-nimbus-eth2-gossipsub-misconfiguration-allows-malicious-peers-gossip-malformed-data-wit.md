# #38318 \[BC-Low] nimbus-eth2: Gossipsub misconfiguration allows malicious peers gossip malformed data without penalization

**Submitted on Dec 30th 2024 at 22:58:51 UTC by @alpharush for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38318
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/status-im/nimbus-eth2>
* **Impacts:**
  * denial of service

## Description

## Brief/Intro

Peers can gossip snappy compressed data that is corrupted or data that is not properly SSZ-encoded and nimbus-eth2 will not reject them, allowing spam.

## Vulnerability Details

Peers can gossip snappy compressed data that is corrupted or data that is not properly SSZ-encoded and nimbus-eth2 will not apply the libp2p invalid message penalty due to the topic weight being set to zero (`basicParams` uses `TopicParam.init` which defaults to zero).

Although these message are rejected as invalid, the penalty applied to the peer's score is zero (the product of the topic weight, zero, and the number of invalid messages sent by the peer). This applies to all of the topic subscriptions with the exception of `AggregateAndProofsTopic` and `BeaconBlocksTopic` which do not use `basicParams`.

<https://github.com/status-im/nimbus-eth2/blob/f54a0366abfba26c7694fad1e14350d19aa0a228/beacon\\_chain/networking/topic\\_params.nim#L58-L58> <https://github.com/search?q=repo%3Astatus-im%2Fnimbus-eth2+basicParams\\&type=code> <https://github.com/vacp2p/nim-libp2p/blob/8855bce0854ecf4adad7a0556bb2b2d2f98e0e20/libp2p/protocols/pubsub/gossipsub/scoring.nim#L75-L77> <https://github.com/vacp2p/nim-libp2p/blob/8855bce0854ecf4adad7a0556bb2b2d2f98e0e20/libp2p/protocols/pubsub/gossipsub/scoring.nim#L209-L216>

## Impact Details

A malicious node can get away with behavior that other nodes won't penalize and that Nimbus can't decode, degrading the node's ability to follow the network and validate the beacon chain. Instead, Nimbus will waste compute and bandwidth deserializing data it can't decoded and have to wait for other non-malicious to gossip e.g. the signed beacon block contents the validators needs to attest to for the current slot. This negatively affects attestation latency and the corresponding validator earnings.

## Fix

Use non-zero topic weights like Prysm for the other topics\
<https://github.com/prysmaticlabs/prysm/blob/80cafaa6dffe12c18091390e8f6b55036db3af67/beacon-chain/p2p/gossip\\_scoring\\_params.go#L20-L46>

## Proof of Concept

## Proof of Concept

I modified prysm (diff: <https://gist.github.com/0xalpharush/519f349791cc0b5f47cd6e873e978440#file-encodegossip-diff>) to send occasionally send random junk in the gossip encoder of Prysm to send junk and ran a Kurtosis network. I observed that the `nbc_gossip_failed_ssz` and `nbc_gossip_failed_snappy` counters climb in Prometheus but Nimbus does not disconnect.

<https://github.com/prysmaticlabs/prysm/blob/c0f9689e303a30dcd63ac82af860062673439afc/beacon-chain/p2p/encoder/ssz.go#L38-L51>

If a modify the Request/Response encoding for RPC like the ping instead, the Nimbus node will stop peering with the bad node. This is correctly penalized here (<https://github.com/status-im/nimbus-eth2/blob/f54a0366abfba26c7694fad1e14350d19aa0a228/beacon\\_chain/networking/eth2\\_network.nim#L1028-L1030>)

1. Apply the above patch to prysm
2. Run a kurtosis network with the configuration file:\
   <https://gist.github.com/0xalpharush/519f349791cc0b5f47cd6e873e978440#file-network\\_params-json>
3. Check the aforementioned prometheus metrics and see that the spam continues indefinitely
