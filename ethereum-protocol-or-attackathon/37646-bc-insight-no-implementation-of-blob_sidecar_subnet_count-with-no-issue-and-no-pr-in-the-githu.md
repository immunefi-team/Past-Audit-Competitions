# #37646 \[BC-Insight] No implementation of BLOB\_SIDECAR\_SUBNET\_COUNT with no issue and no PR in the GitHub

**Submitted on Dec 11th 2024 at 14:10:06 UTC by @Pig46940 for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37646
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/chainsafe/lodestar>
* **Impacts:**
  * (Specifications) A bug in specifications with no direct impact on client implementations

## Description

## Brief/Intro

In the consensus specifications, BLOB\_SIDECAR\_SUBNET\_COUNT should be implemented; however, it is not implemented in the Lodestar codebase.

<https://github.com/ethereum/consensus-specs/blob/d8276acf06a05cf396951687119de55b725ca120/specs/deneb/p2p-interface.m&#x64;_\\[New> in Deneb:EIP4844]\_

| Name                        | Value | Description                                                        |
| --------------------------- | ----- | ------------------------------------------------------------------ |
| `BLOB_SIDECAR_SUBNET_COUNT` | `6`   | The number of blob sidecar subnets used in the gossipsub protocol. |

## Vulnerability Details

The coment left in the test code `// TODO DENEB: Configure the blob subnets in a followup PR`. BUT, I could not find the implementation in entire repository.

The `BLOB_SIDECAR_SUBNET_COUNT` defines the blob sidecar subnet count in the Gossipsub protocol. However, no implementation is found in the entire codebase.

I carefully checked the following code, which should contain the relevant logic, but found no implementation:\
<https://github.com/ChainSafe/lodestar/blob/dad9037e7739d5bcbccfe627e715ef40e9ba935b/packages/beacon-node/src/network/gossip/gossipsub.ts>

The value is only defined in interopConfigs.ts with no implementation:\
<https://github.com/ChainSafe/lodestar/blob/dad9037e7739d5bcbccfe627e715ef40e9ba935b/packages/validator/test/unit/utils/interopConfigs.ts#L54>

I could not understand why your project does not implement this specification logic though the GitHub:\
<https://github.com/search?q=repo%3AChainSafe%2Flodestar%20BLOB\\_SIDECAR\\_SUBNET\\_COUNT\\&type=code>

Additionally, the following comment is left in the test code:\
// TODO DENEB: Configure the blob subnets in a followup PR\
However, I could not find the implementation in the entire repository.

### Other clients

Other consensus clients have implemented this logic.

* Prysm

<https://github.com/prysmaticlabs/prysm/blob/008f157e17e625e44ec076c79aae3a91c0a3f977/beacon-chain/sync/subscriber.go#L146>

* Lighthouse

<https://github.com/sigp/lighthouse/blob/c042dc14d74352512b7632e0ee6ec07f1aa26b3a/beacon\\_node/lighthouse\\_network/src/types/topics.rs#L56>

## Impact Details

It is difficult to clearly understand the full impact; however, the blob sidecar network will increase certain limits when interacting with other clients. This may lead to an increase in P2P network traffic to some extent, potentially exposing a vulnerability that attackers could exploit.

## References

<https://github.com/ethereum/consensus-specs/blob/d8276acf06a05cf396951687119de55b725ca120/specs/deneb/p2p-interface.md#configuration>

## Proof of Concept

## Proof of Concept

Should be in like following code

<https://github.com/ChainSafe/lodestar/blob/dad9037e7739d5bcbccfe627e715ef40e9ba935b/packages/beacon-node/src/network/gossip/gossipsub.ts#L6>

```
     for (const [fork, peersByBeaconBlobSidecarSubnet] of peersByBeaconBlobSidecarSubnetByFork.map) {
        for (let subnet = 0; subnet < BLOB_SIDECAR_SUBNET_COUNT; subnet++) {
          metricsGossip.peersByBeaconBlobSidecarSubnet.set(
            {fork, subnet: attSubnetLabel(subnet)},
            peersByBeaconBlobSidecarSubnet[subnet] ?? 0
          );
        }
      }
```
