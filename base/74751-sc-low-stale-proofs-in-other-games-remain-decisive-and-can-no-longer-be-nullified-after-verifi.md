# 74751 sc low stale proofs in other games remain decisive and can no longer be nullified after verifier nullification

**Submitted on Apr 24th 2026 at 16:40:27 UTC by @silverologist for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74751
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Bypassing the soundness alert mechanism — two conflicting valid proofs of the same type (both TEE or both ZK) fail to trigger automatic game nullification
  * Circumventing the dispute/challenge mechanism to prevent correction of an invalid proposal before finalization
  * Forcing a dispute game into an incorrect resolved state (e.g., DEFENDER\_WINS when CHALLENGER\_WINS should apply, or vice versa)

## Description

## Summary

`Verifier::nullify()` is a global kill switch for a specific verifier. Once a contradiction has successfully nullified a verifier, fresh verification for that verifier is globally disabled.

However, `AggregateVerifier::nullify()` only removes the proof from the one game that executed the nullification. Other games that had already accepted a proof from that verifier keep their local prover slot, keep their `proofCount`, and can still resolve and be promoted to the anchor using that stale proof.

Worse, those other games can no longer use a contradiction to nullify their own stale proofs, because verification now reverts through the globally disabled verifier with `Nullified`.

As a result, games can resolve to an incorrect winner and the bond can be paid to the incorrect receiver.

## Detailed Description

The multiproof verifier contracts inherit from `Verifier`, which provides a single global `nullified` switch per verifier instance.

That switch is meant to be the proof-system emergency stop for fresh verification. Once a same-type contradiction is demonstrated, `Verifier::nullify()` permanently disables future calls to that verifier’s `verify()` entrypoint.

At the game level, `AggregateVerifier::nullify()` does three things:

1. verifies the conflicting same-type proof,
2. removes the existing proof from the current game,
3. calls the corresponding verifier’s global `nullify()`.

The problem is that step `2` is purely local to the calling game.

If another game had already accepted a proof of that same type before the global nullification happened, nothing revisits that already-accepted proof. That other game keeps:

* its existing `proofTypeToProver[proofType]`,
* its existing `proofCount`,
* its unchanged `expectedResolution`,
* and its ability to pass `resolve()`.

At the same time, that other game loses the normal same-type nullification path. `AggregateVerifier::nullify()` still requires a fresh conflicting proof to be verified through `_verifyProof(...)`, but once the shared verifier has been nullified, that verification step reverts before the stale local proof can be removed.

This happens because neither `AggregateVerifier::resolve()` nor `AggregateVerifier::closeGame()` re-check whether a previously accepted proof type has since been globally nullified.

So the multiproof state machine currently distinguishes between:

* **future proofs of that type**, which are blocked by the globally nullified verifier, and
* **already accepted proofs of that type in other games**, which remain sufficient for resolution and anchor promotion while no longer being same-type nullifiable.

That defeats the practical purpose of a global soundness alarm. If a verifier is nullified because it can justify contradictory claims, then games that were relying on already-accepted proofs from that now-untrusted path should not continue to independently resolve and anchor without being re-proven through another still-trusted path.

## Concrete Failure Sequence

Consider the following scenario:

1. Create `game A` with a ZK proof.
2. Create `game B` with a ZK proof.
3. In `game A`, provide a conflicting same-type ZK proof and call `AggregateVerifier::nullify(...)`.
4. This:
   * removes `game A`’s local ZK proof, and
   * globally nullifies `ZK_VERIFIER`.
5. `game B` still retains its already-accepted ZK proof and unchanged `proofCount`.
6. Try to nullify `game B` with a fresh conflicting ZK proof.
7. The call reverts with `Verifier::Nullified`, because the shared `ZK_VERIFIER` has already been globally disabled.
8. Wait for `game B`’s normal resolution delay to pass.
9. Call `game B.resolve()`.
10. Call `game B.closeGame()`.
11. `game B` becomes the new anchor even though the protocol has already globally nullified the ZK verifier for soundness.

Another example:

1. `game A` globally nullifies `ZK_VERIFIER`.
2. `game B` already has:
   * a TEE proof, and
   * a live ZK challenge.
3. `game B` can no longer nullify that stale ZK challenge, because fresh ZK verification now reverts with `Verifier::Nullified`.
4. `game B` still resolves `CHALLENGER_WINS` because the stale ZK challenge remains counted.
5. `bondRecipient` remains the challenger and the bond is paid to the challenger, even though the challenge now depends on a verifier that the protocol has already globally nullified for soundness. The bond is therefore paid to the wrong address.

## Impact

Once a verifier is nullified, other games that had already accepted its proofs can still treat those stale proofs as sufficient for resolution and anchor promotion. Similarly, a ZK challenge proof remains valid even after its verifier is nullified.

At the same time, those games can no longer use a fresh contradiction to nullify the stale proof, because fresh verification, which is required during the nullification process, is now blocked by the global kill switch.

As a consequence, the bond payout can be sent to the wrong receiver and the game can have an incorrect resolution, both in favor of the creator or in favor of the challenger depending on the scenario.

These impacts map to the following severities in the contest scope:

* `Critical: Circumventing the dispute/challenge mechanism to prevent correction of an invalid proposal before finalization`
* `High: Bypassing the soundness alert mechanism — two conflicting valid proofs of the same type (both TEE or both ZK) fail to trigger automatic game nullification`
* `High: Forcing a dispute game into an incorrect resolved state (e.g., DEFENDER_WINS when CHALLENGER_WINS should apply, or vice versa)`

## Root Cause

The root cause is that verifier nullification is global at the verifier layer but local at the game layer.

* `Verifier::nullify()` flips a global kill switch for all games’ future `verify()` calls.
* `AggregateVerifier::nullify()` only deletes the proof from the current game.
* `AggregateVerifier::resolve()` relies on stored local game state (`proofCount`, `proofTypeToProver`, challenge marker) and does not re-check verifier nullification status.
* `AggregateVerifier::closeGame()` and `AnchorStateRegistry::setAnchorState()` likewise do not require that the proof types supporting a resolved game remain globally trusted at anchor time.

So the protocol globally records that the verifier can no longer be trusted for fresh verification, but games that had already accepted proofs from that verifier continue as if nothing changed for resolution purposes, even though the same proof type can no longer be used to nullify them.

## Recommended Fix

Once a verifier is globally nullified, already-accepted proofs of that type should stop being sufficient on their own to carry a game to resolution and anchor promotion.

`resolve()` and `closeGame()` should re-evaluate the game against the current global verifier-nullification state. If a supporting proof comes from a verifier that has since been nullified, that proof should stop counting.

That means:

* recomputing the effective trusted proof set at resolution time,
* deriving `proofCount`, outcome, and `bondRecipient` from the still-trusted proofs only,
* and preventing anchor promotion if a game still relies exclusively on stale proofs from a nullified verifier.

For example, if a game has:

* a still-trusted TEE proof,
* a stale ZK challenge proof,
* and `PROOF_THRESHOLD == 1`,

then after `ZK_VERIFIER` is nullified elsewhere, that stale ZK challenge should stop counting, the game should no longer resolve `CHALLENGER_WINS`, and `bondRecipient` should be updated accordingly.

## Proof of Concept

## PoC

Use the following PoC file at `contracts/test/multiproof/AggregateVerifierGlobalNullificationPoC.t.sol` and run it with `forge test --match-path test/multiproof/AggregateVerifierGlobalNullificationPoC.t.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.15;

import { Claim, GameStatus, Hash } from "src/dispute/lib/Types.sol";
import { AggregateVerifier } from "src/multiproof/AggregateVerifier.sol";
import { Verifier } from "src/multiproof/Verifier.sol";

import { BaseTest } from "./BaseTest.t.sol";

/// @notice PoCs for global verifier nullification not being retroactive across games.
/// @dev Scenario 1: a different game globally nullifies ZK verification, but a stale already-accepted
///      ZK proof in another game still remains sufficient for that game to resolve and anchor.
/// @dev Scenario 2: a game with a TEE proof is challenged by a ZK proof, then a different game globally
///      nullifies the shared ZK verifier. The challenged game can no longer nullify the stale ZK challenge,
///      yet the stale challenge still wins the game and receives the bond.
contract AggregateVerifierGlobalNullificationPoC is BaseTest {
    function testGlobalZkNullificationDoesNotInvalidateAlreadyAcceptedProofInOtherGame() public {
        currentL2BlockNumber += BLOCK_INTERVAL;

        Claim game1RootClaim = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "game1-root")));
        bytes memory game1ZkProof = _generateProof("zk-proof-1", AggregateVerifier.ProofType.ZK);
        AggregateVerifier game1 = _createAggregateVerifierGame(
            ZK_PROVER, game1RootClaim, currentL2BlockNumber, address(anchorStateRegistry), game1ZkProof
        );

        Claim game2RootClaim = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "game2-root")));
        bytes memory game2ZkProof = _generateProof("zk-proof-2", AggregateVerifier.ProofType.ZK);
        AggregateVerifier game2 = _createAggregateVerifierGame(
            ZK_PROVER, game2RootClaim, currentL2BlockNumber, address(anchorStateRegistry), game2ZkProof
        );

        Claim conflictingGame1Root = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "conflicting-root")));
        bytes memory conflictingZkProof = _generateProof("zk-proof-3", AggregateVerifier.ProofType.ZK);
        game1.nullify(conflictingZkProof, BLOCK_INTERVAL / INTERMEDIATE_BLOCK_INTERVAL - 1, conflictingGame1Root.raw());

        assertTrue(zkVerifier.nullified());
        assertEq(game2.proofCount(), 1);
        assertEq(address(game2.zkProver()), ZK_PROVER);

        Claim conflictingGame2Root = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "game2-conflicting-root")));
        bytes memory conflictingGame2ZkProof = _generateProof("zk-proof-4", AggregateVerifier.ProofType.ZK);

        vm.expectRevert(Verifier.Nullified.selector);
        game2.nullify(
            conflictingGame2ZkProof, BLOCK_INTERVAL / INTERMEDIATE_BLOCK_INTERVAL - 1, conflictingGame2Root.raw()
        );

        vm.warp(block.timestamp + 7 days);
        game2.resolve();
        assertEq(uint8(game2.status()), uint8(GameStatus.DEFENDER_WINS));

        vm.warp(block.timestamp + 1);
        game2.closeGame();

        (Hash anchorRoot, uint256 anchorL2BlockNumber) = anchorStateRegistry.getAnchorRoot();
        assertEq(Hash.unwrap(anchorRoot), game2RootClaim.raw());
        assertEq(anchorL2BlockNumber, currentL2BlockNumber);
    }

    function testStaleZkChallengeStillWinsAndReceivesBondAfterGlobalZkNullification() public {
        currentL2BlockNumber += BLOCK_INTERVAL;

        Claim nullifierGameRootClaim = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "nullifier-root")));
        bytes memory nullifierGameZkProof = _generateProof("zk-proof-a", AggregateVerifier.ProofType.ZK);
        AggregateVerifier nullifierGame = _createAggregateVerifierGame(
            ZK_PROVER, nullifierGameRootClaim, currentL2BlockNumber, address(anchorStateRegistry), nullifierGameZkProof
        );

        Claim challengedGameRootClaim = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "challenged-root")));
        bytes memory challengedGameTeeProof = _generateProof("tee-proof-a", AggregateVerifier.ProofType.TEE);
        AggregateVerifier challengedGame = _createAggregateVerifierGame(
            TEE_PROVER,
            challengedGameRootClaim,
            currentL2BlockNumber,
            address(anchorStateRegistry),
            challengedGameTeeProof
        );

        Claim conflictingChallengeRoot = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "challenge-root")));
        bytes memory challengeProof = _generateProof("zk-proof-b", AggregateVerifier.ProofType.ZK);

        vm.prank(ZK_PROVER);
        challengedGame.challenge(
            challengeProof, BLOCK_INTERVAL / INTERMEDIATE_BLOCK_INTERVAL - 1, conflictingChallengeRoot.raw()
        );

        Claim conflictingNullifierRoot =
            Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "nullifier-conflicting-root")));
        bytes memory conflictingNullifierProof = _generateProof("zk-proof-c", AggregateVerifier.ProofType.ZK);
        nullifierGame.nullify(
            conflictingNullifierProof, BLOCK_INTERVAL / INTERMEDIATE_BLOCK_INTERVAL - 1, conflictingNullifierRoot.raw()
        );

        assertTrue(zkVerifier.nullified());

        uint256 challengedIntermediateRootIndex = BLOCK_INTERVAL / INTERMEDIATE_BLOCK_INTERVAL - 1;
        Claim conflictingChallengeRoot2 = Claim.wrap(challengedGame.intermediateOutputRoot(challengedIntermediateRootIndex));
        bytes memory conflictingChallengeProof2 = _generateProof("zk-proof-d", AggregateVerifier.ProofType.ZK);

        vm.expectRevert(Verifier.Nullified.selector);
        challengedGame.nullify(
            conflictingChallengeProof2, challengedIntermediateRootIndex, conflictingChallengeRoot2.raw()
        );

        vm.warp(block.timestamp + 7 days);
        challengedGame.resolve();

        assertEq(uint8(challengedGame.status()), uint8(GameStatus.CHALLENGER_WINS));
        assertEq(challengedGame.bondRecipient(), ZK_PROVER);

        uint256 teeProverBalanceBefore = TEE_PROVER.balance;
        uint256 zkProverBalanceBefore = ZK_PROVER.balance;

        challengedGame.claimCredit();
        vm.warp(block.timestamp + DELAYED_WETH_DELAY);
        challengedGame.claimCredit();

        assertEq(TEE_PROVER.balance, teeProverBalanceBefore);
        assertEq(ZK_PROVER.balance, zkProverBalanceBefore + INIT_BOND);
    }
}
```
