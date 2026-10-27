# 76066 bc low base proofs history eth getproof override removes upstream resource guards allowing low concurrency rpc requests to inflate node cpu and rss

**Submitted on May 2nd 2026 at 13:56:08 UTC by @joohhnnn8 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76066
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

### Brief / Intro

When a Base node is started with the historical proofs ExEx enabled (`--proofs-history`), Base replaces reth's upstream `eth_getProof` implementation with a custom override in `crates/execution/rpc/src/eth/proofs.rs`.

The custom implementation drops the main resource guards used by the upstream reth path:

1. a per-RPC semaphore permit,
2. a max-proof-window check before historical proof generation,
3. a `spawn_blocking` offload for synchronous trie/proof work.

As a result, a remote client can issue a small number of valid `eth_getProof` JSON-RPC requests, each with a large storage-key array. The requests are below the server request-body limit and do not require brute-force traffic. The expensive synchronous proof generation runs directly inside the async RPC handler, which can hold tokio runtime workers and inflate process memory.

This is not a bandwidth-flood or high-rate DDoS claim. The PoC uses N=1, N=2, and N=4 concurrent legitimate RPC calls. The impact comes from Base's override removing upstream RPC-side resource protections around a heavy synchronous proof-generation path.

### Attack Preconditions

* The victim runs a Base node with the historical proofs ExEx enabled (`--proofs-history`).
* The node exposes the `eth_getProof` RPC method to the attacker.
* The attacker sends valid JSON-RPC requests below the configured request body limit.
* No sequencer compromise, leaked key, privileged role, public mainnet/testnet testing, or chain transaction is required.

### Non-Claims

I am not claiming a full network halt, sequencer stall, chain-wide transaction freeze, fund loss, or chain split. The demonstrated impact is resource consumption and service degradation on an affected Base archive/proofs RPC node.

### Vulnerability Details

The affected override is in `crates/execution/rpc/src/eth/proofs.rs:58-91`:

```rust
async fn get_proof(
    &self,
    address: Address,
    keys: Vec<JsonStorageKey>,
    block_number: Option<BlockId>,
) -> RpcResult<EIP1186AccountProofResponse> {
    let start = Instant::now();
    EthApiExtMetrics::get_proof_requests().increment(1);

    let storage_keys = keys.iter().map(|key| key.as_b256()).collect::<Vec<_>>();

    let result = async {
        let proof = self
            .state_provider_factory
            .state_provider(block_number)
            .await
            .map_err(Into::into)?
            .proof(Default::default(), address, &storage_keys)
            .map_err(Into::into)?;

        Ok(proof.into_eip1186_response(keys))
    }
    .await;

    // metrics ...
    result
}
```

`StateProofProvider::proof` is synchronous. It is invoked directly inside this async function, with no `spawn_blocking` offload, no semaphore permit, no storage-key count cap, and no upstream-style proof-window check.

The override is registered at `crates/client/proofs/src/proofs.rs:83-94`:

```rust
let api_ext = EthApiExt::new(ctx.registry.eth_api().clone(), storage.clone());
let debug_ext = DebugApiExt::new(/* ... */);

ctx.modules.replace_configured(api_ext.into_rpc())?;
ctx.modules.replace_configured(debug_ext.into_rpc())?;
```

`replace_configured` replaces reth's upstream `eth_getProof` method with the Base implementation when the historical proofs ExEx is loaded. With the ExEx loaded, affected `eth_getProof` requests hitting the node go through the unprotected Base path.

In upstream reth v1.11.3, the corresponding implementation in `crates/rpc/rpc-eth-api/src/helpers/state.rs` uses three protections before or around proof generation:

1. `acquire_owned_tracing()` to acquire a semaphore permit,
2. `max_proof_window()` to reject overly historical proof requests before proof generation,
3. `spawn_blocking_io_fut(...)` to offload synchronous state/proof work away from tokio runtime workers.

In an upstream reth control configuration, the same class of historical proof request is rejected before proof generation by `max_proof_window`. Even when a proof request is allowed, upstream reth still protects the runtime with a semaphore and `spawn_blocking`; the Base override bypasses all three protections.

The same Base RPC crate already uses similar protections elsewhere, which shows this is an omitted guard rather than required behavior:

* `crates/execution/rpc/src/debug.rs:140` uses a semaphore for debug tracing endpoints.
* `crates/execution/rpc/src/witness.rs:37` uses a semaphore for witness endpoints.
* `crates/execution/rpc/src/eth/transaction.rs:154` uses blocking offload for transaction lookups.
* `crates/execution/rpc/src/eth/mod.rs:232` defines `max_proof_window()`, but the proofs-history override does not enforce it.

### v0.8.0-rc.28 applicability

The dynamic resource measurements below were taken on a local isolated devnet using Base v0.8.0-rc.15. I also verified that the same vulnerable proofs-history override remains present in Base v0.8.0-rc.28: `crates/execution/rpc/src/eth/proofs.rs:58-91` still invokes synchronous proof generation directly inside the async RPC handler, and `crates/client/proofs/src/proofs.rs:83-94` still registers the override through `replace_configured`. No semaphore, `spawn_blocking`, storage-key count cap, or upstream-style proof-window check was added to this override path. Therefore, the same root cause and attack path apply to v0.8.0-rc.28. No public Base mainnet or public testnet endpoint was tested.

### Impact Details

Selected impact:

Increasing network processing node resource consumption by at least 30% without brute force actions.

The attacker is a single host issuing N concurrent legitimate `eth_getProof` requests against the affected Base archive client RPC port. Each request carries 200,000 storage keys on a deployed contract and targets a recently produced block, such as `latest - 30`, that exists inside the Base proofs-history DB range. Each request body is approximately 14.0 MiB, below the default jsonrpsee `max_request_body_size` of 15 MiB.

The baseline below is a controlled no-attack baseline from the same local devnet environment.

Baseline with no attacker traffic:

* `eth_blockNumber` p50: 11.9 ms
* `eth_blockNumber` max: 15.6 ms
* idle RSS: approximately 280 MiB

Under attack, a separate control client sampled `eth_blockNumber` every 200 ms for 60 seconds. Container CPU and RSS were sampled every second with `docker stats`.

Results:

| Case                               | control p50 | control p95 | control p99 | control max | CPU peak |  RSS peak |
| ---------------------------------- | ----------: | ----------: | ----------: | ----------: | -------: | --------: |
| Baseline                           |     11.9 ms |           - |           - |     15.6 ms |        - | \~280 MiB |
| Base archive client, N=1 attacker  |     13.4 ms |     30.7 ms |    333.9 ms |    384.5 ms |     323% |   500 MiB |
| Base archive client, N=2 attackers |     13.6 ms |     55.0 ms |    484.9 ms |    545.0 ms |     495% |   600 MiB |
| Base archive client, N=4 attackers |     14.9 ms |     43.4 ms |    532.4 ms |    827.5 ms |     618% |   897 MiB |
| Upstream reth control, N=4         |     14.5 ms |     35.0 ms |    111.0 ms |    151.3 ms |     311% |   387 MiB |

Attacker request behavior on the same runs:

| Target                            | Single-request behavior                         | Response                                                            |
| --------------------------------- | ----------------------------------------------- | ------------------------------------------------------------------- |
| Base archive client override path | Did not return within the 60 second test window | curl timed out after 60002 ms with 0 bytes received                 |
| Upstream reth control path        | Returned in approximately 1.2-2.6 seconds       | error -32602: distance to target block exceeds maximum proof window |

Key observations:

* RSS increased from approximately 280 MiB idle to 897 MiB with only 4 concurrent attacker requests.
* The absolute RSS increase was approximately 620 MiB, or about 3.2x baseline.
* CPU reached approximately 618%, meaning around 6 CPU cores were busy.
* p99 latency for an unrelated lightweight RPC method, `eth_blockNumber`, increased from approximately 15 ms baseline to 532 ms.
* A single attacker request was already enough to raise CPU to 323% and RSS to 500 MiB.
* The affected Base override accepted the request and spent resources on proof generation, while the upstream reth control rejected the same class of request before proof generation.

This exceeds the selected 30% node resource-consumption threshold under a low-concurrency workload of valid RPC requests.

## Proof of Concept

The PoC has two parts.

Part 1 is the main reproduction: a real Base archive/proofs client in a local isolated devnet.

Part 2 is an optional mechanism-isolation test that demonstrates why running synchronous proof work directly inside an async RPC handler can starve unrelated RPC calls.

No public Base mainnet or public Base testnet endpoint is used.

### Part 1 - Real Base archive client devnet reproduction

{% stepper %}
{% step %}

## Check out the target release

```bash
git clone https://github.com/base/base.git
cd base
git checkout v0.8.0-rc.28
```

The resource measurements reported below were collected on v0.8.0-rc.15. The affected code path and line references were also checked against v0.8.0-rc.28 and remain the same for the vulnerable override.
{% endstep %}

{% step %}

## Start a local Base devnet from the repository

The Base client service must be started with historical proofs enabled:

```txt
--proofs-history
--proofs-history.storage-path=/data/proofs-history
```

In my local docker compose setup, these flags were added to the `base-client` command.
{% endstep %}

{% step %}

## Deploy a simple storage-heavy contract

Once the local chain is producing blocks, deploy a simple storage-heavy contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract BigStorage {
    mapping(uint256 => uint256) public slots;

    function fill(uint256 start, uint256 count) external {
        for (uint256 i = 0; i < count; i++) {
            slots[start + i] = block.number * 1000000 + (start + i);
        }
    }
}
```

{% endstep %}

{% step %}

## Fill the contract with storage entries

In my run, I filled 1,000 mapping entries using 10 batches of 100 writes.
{% endstep %}

{% step %}

## Build an `eth_getProof` JSON-RPC request with 200,000 storage keys

The generated request body was:

```txt
payload: 14000122 bytes
```

This is below the default jsonrpsee 15 MiB request body limit.

The request shape is:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "eth_getProof",
  "params": [
    "<BigStorage contract address>",
    ["0x00...", "0x01...", "... 200000 storage keys ..."],
    "<recent block, e.g. latest - 30>"
  ]
}
```

{% endstep %}

{% step %}

## Run N concurrent attacker requests against the affected Base archive client RPC port

Cases tested:

```txt
N=1
N=2
N=4
```

At the same time, run a separate control sampler that calls `eth_blockNumber` every 200 ms for 60 seconds.

Also sample container CPU and RSS every second using `docker stats`.
{% endstep %}

{% step %}

## Expected output from the affected Base archive client

```txt
=== BASELINE (no attack, 10 samples) ===
baseline eth_blockNumber n=10 p50=11.9ms max=15.6ms

=== BASE_N1 N=1 target=http://127.0.0.1:8545 ===
control eth_blockNumber n=258 p50=13.4ms p95=30.7ms p99=333.9ms max=384.5ms
max CPU%: 323.32
max RSS: 500.8 MiB

=== BASE_N2 N=2 target=http://127.0.0.1:8545 ===
control eth_blockNumber n=250 p50=13.6ms p95=55.0ms p99=484.9ms max=545.0ms
max CPU%: 494.66
max RSS: 602.7 MiB

=== BASE_N4 N=4 target=http://127.0.0.1:8545 ===
control eth_blockNumber n=251 p50=14.9ms p95=43.4ms p99=532.4ms max=827.5ms
max CPU%: 618.06
max RSS: 897.8 MiB
```

The attacker requests against the Base archive client did not return within the 60 second test window:

```txt
curl: Operation timed out after 60002 milliseconds with 0 bytes received
```

{% endstep %}

{% step %}

## Upstream reth control result

Running the same N=4 request class against a non-override upstream reth RPC returned:

```txt
error -32602: distance to target block exceeds maximum proof window
```

The control RPC remained substantially healthier:

```txt
control eth_blockNumber n=259 p50=14.5ms p95=35.0ms p99=111.0ms max=151.3ms
max CPU%: 311.39
max RSS: 387.3 MiB
```

This shows the Base proofs-history override accepts expensive proof work that the upstream guarded path rejects before proof generation, and it performs the synchronous work without semaphore or blocking-pool protection.
{% endstep %}
{% endstepper %}

### Part 2 - Optional runtime mechanism isolation test

This test isolates the runtime behavior without chain state. It runs two JSON-RPC handlers on the same jsonrpsee HTTP server:

1. A Base-pattern handler:
   * synchronous work directly inside async fn
   * no semaphore
   * no `spawn_blocking`
2. A reth-pattern handler:
   * semaphore with 3 permits
   * `spawn_blocking` for CPU-bound work

With four tokio runtime workers and eight concurrent slow requests:

Base-pattern result:

```txt
control eth_blockNumber wall = 1.970s
```

Reth-pattern result:

```txt
control eth_blockNumber wall = 833µs
```

The control RPC latency differs by approximately 2370x.

This test is not the main impact proof. It is included only to explain the mechanism: the affected Base override performs synchronous proof work directly inside the async RPC path, while the upstream pattern isolates heavy work behind a semaphore and blocking pool.

### References

Asset: <https://github.com/base/base/tree/v0.8.0-rc.28>

Dynamic measurement environment: Base v0.8.0-rc.15 local isolated devnet

Affected file in v0.8.0-rc.28: crates/execution/rpc/src/eth/proofs.rs

Affected function: EthApiExt::get\_proof

Affected lines in v0.8.0-rc.28: crates/execution/rpc/src/eth/proofs.rs:58-91

Override registration in v0.8.0-rc.28: crates/client/proofs/src/proofs.rs:83-94

Related Base files showing the protective pattern exists elsewhere: crates/execution/rpc/src/debug.rs crates/execution/rpc/src/witness.rs crates/execution/rpc/src/eth/transaction.rs crates/execution/rpc/src/eth/mod.rs

Upstream guarded implementation: reth v1.11.3 crates/rpc/rpc-eth-api/src/helpers/state.rs EthState::get\_proof
