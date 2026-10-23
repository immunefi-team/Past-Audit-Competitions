# #37594 \[SC-Insight] Nimbus incorrectly rejects non-minimally encoded snappy data length's due to spec. ambiguity

**Submitted on Dec 10th 2024 at 04:40:19 UTC by @alpharush for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37594
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/ethereum/consensus-specs>
* **Impacts:**
  * (Specifications) A bug in specifications with direct impact on client implementations

## Description

## Brief/Intro

The consensus spec does not specify that the varint length of snappy encoded p2p messages must be minimally encoded. Nimbus’ implementation makes this assumption and will reject messages that other clients accept.

Here the spec should clarify whether the varint must be minimally encoded and clients should uniformly error, if so:\
<https://github.com/ethereum/consensus-specs/blob/83a8042c0d67452c6f25f15ce613085f2b508295/specs/phase0/p2p-interface.md?plain=1#L667>

## Vulnerability Details

The root cause is that Nim’s snappy implementation cannot decode varint’s encoded as more than 5 bytes (uint32) while the spec supports up to 10 bytes.\
(<https://github.com/status-im/nim-snappy/blob/0c308d34241c9f0764f6d111a0288428ded173bc/snappy/codec.nim#L134>)

## Impact Details

Normally, this would cause a fork in the chain due to peer penalization for sending messages that can’t be decoded. However, Nimbus’ peer scoring is broken and won’t disconnect. The misconfiguration of nim-libp2p has been reported previously to the EF and is apparently a known issue (<https://github.com/status-im/nimbus-eth2/pull/3029>). Nimbus clients will just have to wait for other peers to send the correct message in order to keep up with the chain.

This spec-implementation divergence should be clarified and patched in clients appropriately. If Nimbus’ peer penalization was, it would cause forks in the p2p network similar to this [post](https://www.asymmetric.re/blog/ghost-in-the-block-ethereum-consensus-vulnerability). Currently, a malicious peer can gossip messages all clients except Nimbus can decode by non-minimally encoding the length prefix of their snappy-encoded data.

## References

Add any relevant links to documentation or code

## Link to Proof of Concept

<https://gist.github.com/0xalpharush/519f349791cc0b5f47cd6e873e978440>

## Proof of Concept

## Proof of Concept

This behavior is evident by testing the Nim implementation of snappy directly. Apply this [diff](https://gist.github.com/0xalpharush/519f349791cc0b5f47cd6e873e978440#file-test_snappy-nim-diff) to [nim-snappy](https://github.com/status-im/nim-snappy/). Run `nim c -r tests/test_snappy.nim "malformed data"`

Observe that the length is incorrect:

```
snappy lenU32 0
bytesRead -6
none()
```

Prysm’s implementation will accept this input as `24023`. Run this [go script](https://gist.github.com/0xalpharush/519f349791cc0b5f47cd6e873e978440#file-prysm_varint-go).

```
go mod init test
go mod tidy
go run main.go
```

The expected output:

```
24023
24023
24023
```

Thus, it’s clear Prysm’s gossip decoding of snappy encoded data data not require the varint length prefix to be minimally encoded.\
<https://github.com/gogo/protobuf/blob/f67b8970b736e53dbd7d0a27146c8f1ac52f74e5/proto/decode.go#L51-L72>

Lighthouse’s snappy decoding also does not.\
<https://github.com/BurntSushi/rust-snappy/blob/a65ad09a96568bb162b23c89636601a30a40013e/src/varint.rs#L14-L31>

To see the impact on clients in a production setting, I modified Prysm to send non-minimally encoded snappy data in the [gossip encoder](https://gist.github.com/0xalpharush/519f349791cc0b5f47cd6e873e978440#file-encodegossip-diff) of Prysm and ran a Kurtosis network. I observed that the `nbc_gossip_failed_snappy` [counters](https://github.com/status-im/nimbus-eth2/blob/d2d02bd68c45729c5457235f72807f3427e0a3b0/beacon_chain/networking/eth2_network.nim#L2433-L2448) climb in Prometheus but Nimbus does not disconnect.

```
kurtosis run --enclave testnet \
  github.com/kurtosis-tech/ethereum-package@b0820ddae77e7d45d090c00e47aa3e8d3832e194 \
  --args-file ./network_params.json
```

Changing the [RPC encoding](https://gist.github.com/0xalpharush/519f349791cc0b5f47cd6e873e978440#file-encodewithmaxlength-diff) will cause disconnects as it correctly penalizes peers. Re-run the kurtosis network and observe the peers disconnect.
