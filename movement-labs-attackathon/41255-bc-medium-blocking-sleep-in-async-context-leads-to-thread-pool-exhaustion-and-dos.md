# #41255 \[BC-Medium] Blocking sleep in async context leads to thread pool exhaustion and DoS

**Submitted on Mar 13th 2025 at 03:25:35 UTC by @Rhaydden for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41255
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/>
* **Impacts:**
  * Shutdown of greater than or equal to 30% of network processing nodes without brute force actions, but does not shut down the network
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## Brief/Intro

`try_http2` function in `MovementDaLightNodeClient` uses `std::thread::sleep` within its asynchronous retry loop. This is a blocking operation bein used in an asynchronous context causing the entire thread to pause instead of allowing the asynchronous runtime to proceed with other tasks. In a production environment, this could lead to performance degradation, especially under load as it can stall the thread pool used by the async runtime, leading to reduced responsiveness and throughput for HTTP/2 connections. This can lead to thread pool exhaustion and denial of service, allowing an attacker to paralyze the entire node's async runtime and prevent legitimate transactions from being processed

## Vulnerability Details

The `try_http2` function is dessigned to establish an HTTP/2 connection to a light node service with retry logic. It iterates up to 5 times, attempting to connect. If a connection fails, it currently uses `std::thread::sleep` to pause before the next retry attempt:

```rust
32: 	/// Creates an http2 connection to the light node service.
33: 	pub async fn try_http2(connection_string: &str) -> Result<Self, anyhow::Error> {
34: 		for _ in 0..5 {
35: 			match http2::Http2::connect(connection_string).await {
36: 				Ok(result) => return Ok(Self::Http2(result)),
37: 				Err(err) => {
38: 					tracing::warn!("DA Http2 connection failed: {}. Retrying in 5s...", err);
39: 					std::thread::sleep(std::time::Duration::from_secs(5));  ❌
40: 				}
41: 			}
42: 		}
43: 		return Err(
44: 			anyhow::anyhow!("Error DA Http2 connection failed more than 5 time aborting.",),
45: 		);
46: 	}
```

`std::thread::sleep` is a blocking function. When called within an `async` function, it blocks the entire operating system thread on which the asynchronous task is running. This is an isssue because asynchronous runtimes like Tokio (commonly used in Rust async applications, and likely underlying `http2::Http2::connect` and tonic/gRPC used here) are designed to efficiently manage a pool of threads. They expect tasks to yield control back to the runtime when waiting for I/O or other operations, allowing other tasks to run on the same thread.

Using `std::thread::sleep` prevents this efficient scheduling. While the function is sleeping, the thread is completely stalled, unable to process other asynchronous tasks.

As a result,

* Each failed connection blocks a thread for 5 seconds
* The method retries up to 5 times, blocking a thread for 25 seconds total
* The blocking occurs in the core connection handling code
* It affects both new connections and existing stream processing

## Fix

Coorect approach for asynchronous delays would be to use a non-blocking sleep function provided by the asynchronous runtime. For Tokio, this is `tokio::time::sleep`. The fix is to replace `std::thread::sleep` with `tokio::time::sleep`

```diff
pub async fn try_http2(connection_string: &str) -> Result<Self, anyhow::Error> {
    for _ in 0..5 {
        match http2::Http2::connect(connection_string).await {
            Ok(result) => return Ok(Self::Http2(result)),
            Err(err) => {
                tracing::warn!("DA Http2 connection failed: {}. Retrying in 5s...", err);
-               std::thread::sleep(std::time::Duration::from_secs(5));
+               tokio::time::sleep(std::time::Duration::from_secs(5)).await;
            }
        }
    }
    Err(anyhow::anyhow!("Error DA Http2 connection failed more than 5 times, aborting."))
}
```

## Impact Details

Impact is primarily related to performance and efficiency, particularly under load when establishing HTTP/2 connections might require retries. Increased latency for operations relying on HTTP/2 connections as threads are unnecessarily blocked during retry attempts. This could manifest as slower response times for user requests or delays in processing data. Thread pool exhaustion preventing transaction processing. Also existing streams becoming unresponsive.\
The attack doesnt require much to execute but can cause widespread disruption to the protocol's operation.

## References

<https://github.com/immunefi-team/attackathon-movement//blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/client/src/lib.rs#L32-L46>

<https://docs.rs/tokio/latest/tokio/runtime/index.html>

<https://rust-lang.github.io/async-book/04\\_pinning/01\\_chapter.html>

## Proof of Concept

## Proof of Concept

Here's how the blocking sleep in the async `try_http2` function couold be exploited:

1. Setup Phase:

```rust
// Attacker creates multiple concurrent connection attempts
async fn launch_attack(target: &str, num_connections: usize) {
    let handles: Vec<_> = (0..num_connections)
        .map(|_| {
            tokio::spawn(async move {
                MovementDaLightNodeClient::try_http2(target).await
            })
        })
        .collect();
}
```

2. Attack Sequence:

* Set up a malicious server that deliberately fails HTTP/2 connection attempts
* The server should respond with a TCP connection but fail the HTTP/2 handshake
* Launch 100+ concurrent connection attempts

3. Exploitation Steps:

```rust
// Example malicious server behavior
async fn malicious_server() {
    let listener = TcpListener::bind("0.0.0.0:8080").await.unwrap();
    loop {
        let (socket, _) = listener.accept().await.unwrap();
        // Accept connection but fail HTTP/2 handshake
        // This triggers the retry mechanism
    }
}

// Attack execution
#[tokio::main]
async fn main() {
    // Launch 100 concurrent connection attempts
    launch_attack("http://malicious-server:8080", 100).await;
}
```

4. What happen:

* Each failed connection triggers the retry loop
* Each retry calls `std::thread::sleep(Duration::from_secs(5))`
* With 100 connections:
* * 100 threads are blocked for 5 seconds
* * This repeats 5 times per connection
* * Total blocking time = 100 \* 5 \* 5 = 2500 thread-seconds

5. Impact:

* Tokio's thread pool (default size usually matches CPU cores) gets exhausted
* All async tasks in the application stall
* Memory usage increases as tasks queue up
* Application becomes unresponsive

6.

```rust
// Add this to verify impact
async fn verify_attack() {
    // Monitor thread pool stats before attack
    let before = tokio::runtime::Handle::current().metrics();
    
    // Launch attack
    launch_attack("malicious-server:8080", 100).await;
    
    // Monitor after - should see blocked threads
    let after = tokio::runtime::Handle::current().metrics();
    println!("Blocked threads: {}", after.blocked_threads());
}
```

7. Why exploit is possible:

* The async function uses blocking thread::sleep
* Each blocked thread occupies a slot in Tokio's thread pool
* Once thread pool is exhausted, all other async operations stall
* Application can't process legitimate requests

In reality, service becomes unresponsive to legit clients. Memory usage grows with queued tasks. Also Protocol's resources get exhausted.
