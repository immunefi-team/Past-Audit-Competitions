# #41594 \[BC-Insight] Invalid URL format in TcpListener binding prevents REST API from starting

**Submitted on Mar 16th 2025 at 20:03:47 UTC by @Rhaydden for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41594
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/movement-rest>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk
  * RPC API crash affecting programs with greater than or equal to 25% of the market capitalization on top of the respective layer

## Description

## Brief/Intro

`MovementRest` service fails to start because the URL provided for binding the TCP listener includes the `http://` scheme. The `poem::listener::TcpListener::bind()` method, which is used to bind the service to a network address, expects a socket address in the format `"host:port"` and not a full URL. This prevents the entire REST API service from starting in any production environment, rendering all API endpoints inaccessible. Any operation built on top of Movement that rely on this REST API would be completely non-functional.

## Vulnerability Details

The issue is in the `MovementRest::try_from_env()` method where the default URL for the service is defined, and in the `run_service()` method where this URL is used to bind the `TcpListener`.

In `MovementRest::try_from_env()`:

```rust
let url = env::var(Self::MOVEMENT_REST_ENV_VAR)
    .unwrap_or_else(|_| "http://0.0.0.0:30832".to_string());
```

The default value for the `url` field is set to `"http://0.0.0.0:30832"`, which includes the `"http://"` scheme.

In `run_service()`:

```rust
Server::new(TcpListener::bind(self.url.clone()))
    .run(movement_rest)
    .map_err(Into::into)
```

The `self.url` (which defaults to `"http://0.0.0.0:30832"`) is directly passed to `TcpListener::bind()`.

According to the rust doc for `std::net::TcpListener::bind()` (which is the underlying listener used by Poem). See here:

<https://doc.rust-lang.org/std/net/struct.TcpListener.html>

> `pub fn bind<A: ToSocketAddrs>(addr: A) -> Result<TcpListener>`
>
> Creates a new `TcpListener` which will be bound to the specified address.
>
> The address type can be any implementor of `ToSocketAddrs` trait. See the doc for concrete examples.
>
> Examples\
> Creates a TCP listener bound to 127.0.0.1:80:
>
> ```
> use std::net::TcpListener;
>
> let listener = TcpListener::bind("127.0.0.1:80").unwrap();
> ```

The doc and examples clearly show that `TcpListener::bind()` expects a string representing a socket address in the format `"host:port"`. Passing a full URL with a scheme like `"http://"` will cause the binding process to fail as it cannot be parsed into a valid socket address. This will result in an error at runtime, preventing the `MovementRest` service from starting and listening for incoming connections.

This is a bit tricky because iff we look at the test here, the unit tests pass successfully. This is because the tests use `TestClient`, which doesn't actually bind to a TCP port:

```rust
#[tokio::test]
	async fn test_health_endpoint() {
		let rest_service = MovementRest::try_from_env().expect("Failed to create MovementRest");
		assert_eq!(rest_service.url, "http://0.0.0.0:30832");
		// Create a test client
		let client = TestClient::new(rest_service.create_routes());

		// Test the /health endpoint
		let response = client.get("/health").send().await;
		assert!(response.0.status().is_success());
	}
}
```

## Impact Details

I categorised this under "RPC API crash affecting programs with greater than or equal to 25% of the market capitalization on top of the respective layer" for the following reasons:

The REST service would fail to start in any production environment making all API endpoints inaccessible, including:

* The /health endpoint for monitoring
* The /movement/v1/state-root-hash/:blockheight endpoint for retrieving state root hashes

Any applications, wallets, or services built on top of Movement that rely on this REST API would be completely non-functional.

## References

<https://github.com/immunefi-team/attackathon-movement//blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/movement-rest/src/lib.rs>

<https://doc.rust-lang.org/std/net/struct.TcpListener.html>

## Proof of Concept

## Proof of concept

Build and run the Movement REST service in a real environment (not a test environment):

```rust
// This would fail with an error like:
// Error: failed to parse socket address: invalid socket address syntax
let rest_service = MovementRest::try_from_env().expect("Failed to create MovementRest");
rest_service.run_service().await.unwrap();
```

The error would occur because `TcpListener::bind("http://0.0.0.0:30832")` fails to parse the string into a valid `SocketAddr`.

Here's a simple test that demonstrates the failure:

```rust
#[tokio::test]
async fn test_server_binding_fails() {
    let rest_service = MovementRest::try_from_env().expect("Failed to create MovementRest");
    
    // Attempt to start the service
    let result = rest_service.run_service().await;
    
    // The binding should fail due to invalid URL format
    assert!(result.is_err());
    assert!(result.unwrap_err().to_string().contains("invalid socket address"));
}
```

## Fix

Fix is straightforward. Remove the HTTP scheme prefix from the default URL:

```diff
pub fn try_from_env() -> Result<Self, Error> {
		let url = env::var(Self::MOVEMENT_REST_ENV_VAR)
-			.unwrap_or_else(|_| "http://0.0.0.0:30832".to_string());
+           .unwrap_or_else(|_| "0.0.0.0:30832".to_string()); 
		Ok(Self { url, context: None })
	}
```

Or paarse the URL and extract only the host and port if a full URL is expected from the environment variable.
