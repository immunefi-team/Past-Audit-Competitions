# 76505 bc low proposer jit freshness check reuses cached result

## #76505 \[BC-Low] Proposer JIT Freshness Check Reuses Cached `output_at_block` Result

**Submitted on May 4th 2026 at 17:38:46 UTC by @v\_c0d35 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76505
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

### Description

## Proposer JIT Freshness Check Reuses Cached `output_at_block` Result

### Brief/Intro

The Base Azul proof proposer caches `optimism_outputAtBlock` responses by L2 block number and later performs its submit-time "JIT" freshness check through the same cached accessor. If the rollup RPC returns a stale output root once during proof construction, the proposer can reuse that stale cached value during `validate_and_submit` and treat the proposal as fresh without performing a new RPC read. In production, this can cause the proposer to submit a stale or incorrect output root as a dispute game; the demonstrated impact is limited to proposer-side freshness validation because the independent challenger is expected to recompute and dispute invalid games before finalization.

### Vulnerability Details

`RollupClient` stores `optimism_outputAtBlock` responses in an in-memory cache keyed only by the numeric L2 block number:

```rust
/// Cache for `optimism_outputAtBlock` responses keyed by L2 block number.
output_cache: MeteredCache<u64, OutputAtBlock>,
```

The cache is initialized as a capacity-bounded Moka cache, with no visible TTL, finality binding, RPC endpoint binding, L1-origin binding, or proof-context binding. `output_at_block()` checks this cache before making the `optimism_outputAtBlock` RPC call:

```rust
async fn output_at_block(&self, block_number: u64) -> RpcResult<OutputAtBlock> {
    if let Some(cached) = self.output_cache.get(&block_number).await {
        return Ok(cached);
    }

    let output = /* optimism_outputAtBlock RPC */;
    self.output_cache.insert(block_number, output).await;
    Ok(output)
}
```

This makes `output_at_block(N)` sticky for the lifetime of the cache entry. If the first response for block `N` is stale root `A`, later calls for the same `N` return `A` even if the rollup RPC would now return canonical root `B`.

The same accessor is used during proof construction. In `build_proof_request_for`, the proposer calls `self.rollup_client.output_at_block(target_block)` and puts the returned `claimed_output.output_root` into the proof request:

```rust
self.rollup_client.output_at_block(target_block).await
...
claimed_l2_output_root: claimed_output.output_root,
```

Later, `validate_and_submit` attempts to perform submit-time JIT validation:

```rust
// JIT validation: check that the proved output root still matches canonical.
let canonical_output = self
    .rollup_client
    .output_at_block(target_block)
    .await?;

if aggregate_proposal.output_root != canonical_output.output_root {
    return Err(SubmitAction::RootMismatch);
}
```

This looks like a fresh canonical read, but it is not guaranteed to be fresh. It calls the same cached `output_at_block()` method for the same `target_block`. Therefore, the following sequence is possible:

1. Proof construction calls `output_at_block(N)`.
2. The rollup RPC returns stale output root `A`.
3. `RollupClient` stores `A` in `output_cache[N]`.
4. The proof is built for root `A`.
5. The rollup RPC later would return canonical root `B` for block `N`.
6. `validate_and_submit` calls `output_at_block(N)` for its JIT check.
7. The cache returns `A`, so the proposer compares `A` against `A`.
8. The root mismatch is not detected, and the proposer proceeds to submit the game.

The follow-on intermediate-root validation also reuses `canonical_output.output_root` for the target block by inserting it into `canonical_map`, so the target block remains validated against the cached value rather than a newly observed canonical value.

If validation passes, `validate_and_submit` calls `self.output_proposer.propose_output(...)`. `ProposalSubmitter::propose_output` then ABI-encodes `proposal.output_root` into the dispute-game creation calldata and sends the transaction to the configured factory. This means the stale root that passed the cached JIT check can become the submitted root claim.

The core issue is not merely that the proposer trusts the rollup RPC for output roots. The narrower cache/JIT bug is that the code has an explicit submit-time freshness check, but that check can be reduced to comparing the proof result against a previously cached proof-construction value. JIT validation should bypass, clear, or context-bind this cache when freshness is required.

### Impact Details

The directly demonstrated impact is that the proposer-side freshness guard can be bypassed by a stale resident cache entry. A stale `optimism_outputAtBlock` response can pass through proof construction and then self-confirm during `validate_and_submit`, allowing the proposer to submit an invalid or stale dispute-game proposal that should have been rejected by a fresh submit-time read.

This weakens the protocol's defense-in-depth around output proposal freshness. The immediate operational consequence is creation of an invalid dispute game and associated proposal gas/bond handling that should have been avoided by the JIT validation step. In the default architecture, the challenger independently recomputes output roots from L2 headers and account/storage proofs, so the expected recovery path is for the invalid game to be detected and disputed before it becomes final.

### References

* [`RollupClient` cache field keyed by L2 block number](https://github.com/base/base/blob/v0.8.0-rc.28/crates/proof/rpc/src/rollup_client.rs#L75-L80)
* [`RollupClient::new` creates `rollup_output_at_block` cache](https://github.com/base/base/blob/v0.8.0-rc.28/crates/proof/rpc/src/rollup_client.rs#L93-L117)
* [`RollupClient::output_at_block` returns cached value before making RPC call](https://github.com/base/base/blob/v0.8.0-rc.28/crates/proof/rpc/src/rollup_client.rs#L169-L197)
* [`MeteredCache::with_capacity`, `get`, and `insert`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/proof/rpc/src/cache.rs#L94-L143)
* [`build_proof_request_for` sources `claimed_l2_output_root` from `output_at_block`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/proof/proposer/src/pipeline.rs#L957-L991)
* [`validate_and_submit` JIT validation also calls `output_at_block`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/proof/proposer/src/pipeline.rs#L1047-L1078)
* [`validate_and_submit` reuses the target block's `canonical_output` in `canonical_map`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/proof/proposer/src/pipeline.rs#L1094-L1102)
* [`validate_and_submit` proceeds to `propose_output` after validation](https://github.com/base/base/blob/v0.8.0-rc.28/crates/proof/proposer/src/pipeline.rs#L1158-L1177)
* [`ProposalSubmitter::propose_output` encodes and submits `proposal.output_root`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/proof/proposer/src/output_proposer.rs#L103-L132)

### Proof of Concept

## PoC: Proposer JIT Freshness Check Reuses Cached `output_at_block` Result

### Goal

This PoC runs entirely against the public `base/base` repository at tag `v0.8.0-rc.28`. It demonstrates that `RollupClient::output_at_block(N)` caches the first `optimism_outputAtBlock(N)` response and that a later call for the same block, modeling `validate_and_submit` JIT validation, returns the cached stale root without re-querying the rollup RPC. A fresh client with an empty cache then observes the updated RPC response, proving the cache is the reason the JIT freshness read stayed stale.

### What This Reproduces

The production proposer code path uses `RollupClient::output_at_block` in both places relevant to the bug:

* Proof construction: [`build_proof_request_for`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/proof/proposer/src/pipeline.rs#L957-L991) calls `self.rollup_client.output_at_block(target_block)` and stores the returned root as `claimed_l2_output_root`.
* Submit-time JIT validation: [`validate_and_submit`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/proof/proposer/src/pipeline.rs#L1047-L1078) calls `self.rollup_client.output_at_block(target_block)` again and compares the proposal root against that value.

The PoC tests the shared vulnerable primitive directly. It uses a local mock rollup RPC that returns root `A` on the first `optimism_outputAtBlock(N)` call and root `B` on the next network call for the same block.

Expected vulnerable behavior:

1. First call, modeling proof construction, receives root `A`.
2. Second call on the same `RollupClient`, modeling JIT validation, returns cached root `A`.
3. The mock RPC request counter remains `1`, proving no fresh RPC read occurred during the JIT-model call.
4. A fresh `RollupClient` with an empty cache receives root `B`, proving the server had changed and the stale result came from the cache.

### Prerequisites

* Rust/Cargo toolchain installed.
* Network access for the initial clone and dependency download.
* No Base node, rollup node, L1 RPC, L2 RPC, deployment artifact, or private test fixture is required.

{% stepper %}
{% step %}

### Clone The Public Repository

```bash
git clone https://github.com/base/base.git
cd base
git checkout v0.8.0-rc.28
```

The tag is annotated, so Git may print a detached-HEAD notice. That is expected.
{% endstep %}

{% step %}

### Enable Tokio Test Runtime Features

Open `crates/proof/rpc/Cargo.toml` and replace the existing `[dev-dependencies]` Tokio line:

```toml
tokio = { workspace = true, features = ["test-util"] }
```

with:

```toml
tokio = { workspace = true, features = ["macros", "rt-multi-thread", "test-util"] }
```

This only enables the async test runtime for the PoC integration test.
{% endstep %}

{% step %}

### Add The PoC Test File

Save the following file to `crates/proof/rpc/tests/output_at_block_jit_cache.rs`:

```rust
#![allow(missing_docs)]

use std::{
    io::{Read, Write},
    net::{TcpListener, TcpStream},
    sync::{
        atomic::{AtomicUsize, Ordering},
        Arc,
    },
    thread::{self, JoinHandle},
    time::Duration,
};

use alloy_primitives::B256;
use base_proof_rpc::{RollupClient, RollupClientConfig, RollupProvider};
use url::Url;

fn repeated_hex(byte: &str) -> String {
    format!("0x{}", byte.repeat(32))
}

fn response_body(root: &str, block_number: u64) -> String {
    format!(
        r#"{{"jsonrpc":"2.0","id":1,"result":{{"outputRoot":"{root}","blockRef":{{"hash":"0x3333333333333333333333333333333333333333333333333333333333333333","number":{block_number},"parentHash":"0x2222222222222222222222222222222222222222222222222222222222222222","timestamp":1234567890,"l1origin":{{"hash":"0x1111111111111111111111111111111111111111111111111111111111111111","number":100}},"sequenceNumber":0}}}}}}"#
    )
}

fn read_request(stream: &mut TcpStream) -> String {
    let mut bytes = Vec::new();
    let mut buf = [0u8; 1024];

    loop {
        let n = stream.read(&mut buf).expect("read request");
        if n == 0 {
            break;
        }
        bytes.extend_from_slice(&buf[..n]);

        let Some(headers_end) = bytes.windows(4).position(|w| w == b"\r\n\r\n") else {
            continue;
        };
        let headers = String::from_utf8_lossy(&bytes[..headers_end]);
        let content_length = headers
            .lines()
            .find_map(|line| {
                let (name, value) = line.split_once(": ")?;
                name.eq_ignore_ascii_case("content-length")
                    .then(|| value.parse::<usize>().expect("content length"))
            })
            .unwrap_or(0);

        if bytes.len() >= headers_end + 4 + content_length {
            break;
        }
    }

    String::from_utf8(bytes).expect("request is utf8")
}

fn start_sequential_rollup_rpc(
    stale_root: String,
    fresh_root: String,
    block_number: u64,
) -> (Url, Arc<AtomicUsize>, JoinHandle<()>) {
    let listener = TcpListener::bind("127.0.0.1:0").expect("bind mock RPC");
    let addr = listener.local_addr().expect("local address");
    let calls = Arc::new(AtomicUsize::new(0));
    let calls_for_thread = Arc::clone(&calls);

    let handle = thread::spawn(move || {
        for stream in listener.incoming().take(2) {
            let mut stream = stream.expect("accept request");
            let request = read_request(&mut stream);
            assert!(request.contains("optimism_outputAtBlock"), "unexpected request: {request}");
            assert!(
                request.contains(&format!("0x{block_number:x}")),
                "request did not ask for target block {block_number}: {request}"
            );

            let call_index = calls_for_thread.fetch_add(1, Ordering::SeqCst);
            let root = if call_index == 0 { &stale_root } else { &fresh_root };
            let body = response_body(root, block_number);
            let response = format!(
                "HTTP/1.1 200 OK\r\ncontent-type: application/json\r\ncontent-length: {}\r\nconnection: close\r\n\r\n{}",
                body.len(),
                body
            );
            stream.write_all(response.as_bytes()).expect("write response");
        }
    });

    (Url::parse(&format!("http://{addr}")).expect("mock URL"), calls, handle)
}

#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
async fn output_at_block_cache_defeats_submit_time_freshness_model() {
    let target_block = 12_345u64;
    let stale_root_hex = repeated_hex("11");
    let fresh_root_hex = repeated_hex("22");
    let stale_root: B256 = stale_root_hex.parse().expect("stale root");
    let fresh_root: B256 = fresh_root_hex.parse().expect("fresh root");

    let (rpc_url, rpc_calls, server) =
        start_sequential_rollup_rpc(stale_root_hex, fresh_root_hex, target_block);

    let proposer_client = RollupClient::new(
        RollupClientConfig::new(rpc_url.clone()).with_timeout(Duration::from_secs(5)),
    )
    .expect("proposer rollup client");

    // This models proof construction: the first RPC answer for the target block is stale root A.
    let proof_construction_output = proposer_client
        .output_at_block(target_block)
        .await
        .expect("proof construction output");
    assert_eq!(proof_construction_output.output_root, stale_root);
    assert_eq!(rpc_calls.load(Ordering::SeqCst), 1);

    // This models validate_and_submit JIT validation. The mock RPC would now return fresh root B,
    // but the same RollupClient returns cached A without making a second RPC request.
    let jit_validation_output = proposer_client
        .output_at_block(target_block)
        .await
        .expect("jit validation output");
    assert_eq!(jit_validation_output.output_root, stale_root);
    assert_eq!(
        rpc_calls.load(Ordering::SeqCst),
        1,
        "JIT validation reused the cached output and did not re-query the rollup RPC"
    );
    assert_eq!(proof_construction_output.output_root, jit_validation_output.output_root);

    // A fresh client with an empty cache observes the updated RPC answer B for the same block.
    // This proves that the cached value, rather than the mocked RPC, is why JIT saw A.
    let fresh_client = RollupClient::new(
        RollupClientConfig::new(rpc_url).with_timeout(Duration::from_secs(5)),
    )
    .expect("fresh rollup client");
    let fresh_output = fresh_client
        .output_at_block(target_block)
        .await
        .expect("fresh output");
    assert_eq!(fresh_output.output_root, fresh_root);
    assert_eq!(rpc_calls.load(Ordering::SeqCst), 2);

    server.join().expect("mock RPC server exits");
}
```

{% endstep %}

{% step %}

### Run The PoC

Run only the new integration test:

```bash
cargo test -p base-proof-rpc --test output_at_block_jit_cache output_at_block_cache_defeats_submit_time_freshness_model -- --nocapture
```

Expected output:

```
running 1 test
test output_at_block_cache_defeats_submit_time_freshness_model ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

{% endstep %}
{% endstepper %}

### Why The Passing Test Demonstrates The Vulnerability

The mock RPC server is programmed to return a stale root first and a fresh root on the next network request for the same block. After the first call, `RollupClient` stores the stale root in `output_cache` under the bare L2 block number. The second call uses the same `RollupClient` and same block number. The test proves that:

* the second call returns the stale root;
* the mock RPC request counter is still `1`, so no fresh network request occurred;
* a fresh `RollupClient` with an empty cache then observes the fresh root from the same mock RPC.

This reproduces the submit-time freshness failure because the production JIT check calls the same cached `output_at_block(target_block)` method used during proof construction. A stale output root can therefore compare equal to itself during JIT validation.

### Relevant Public Source Links

* [`RollupClient` cache field keyed by L2 block number](https://github.com/base/base/blob/v0.8.0-rc.28/crates/proof/rpc/src/rollup_client.rs#L75-L80)
* [`RollupClient::output_at_block` returns a cached value before making the RPC call](https://github.com/base/base/blob/v0.8.0-rc.28/crates/proof/rpc/src/rollup_client.rs#L169-L197)
* [`MeteredCache::with_capacity`, `get`, and `insert`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/proof/rpc/src/cache.rs#L94-L143)
* [`build_proof_request_for` sources `claimed_l2_output_root` from `output_at_block`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/proof/proposer/src/pipeline.rs#L957-L991)
* [`validate_and_submit` JIT validation also calls `output_at_block`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/proof/proposer/src/pipeline.rs#L1047-L1078)
* [`ProposalSubmitter::propose_output` submits `proposal.output_root`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/proof/proposer/src/output_proposer.rs#L103-L132)

### Verification Notes

These steps were verified from a fresh public `v0.8.0-rc.28` checkout. The final test run completed successfully with:

```
running 1 test
test output_at_block_cache_defeats_submit_time_freshness_model ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.04s
```
