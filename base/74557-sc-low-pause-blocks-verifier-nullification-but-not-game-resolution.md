# 74557 sc low pause blocks verifier nullification but not game resolution

Submitted on Apr 23rd 2026 at 12:30:42 UTC by @silverologist for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74557
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Bypassing the soundness alert mechanism — two conflicting valid proofs of the same type (both TEE or both ZK) fail to trigger automatic game nullification

## Description

## Summary

When the system is paused, `AggregateVerifier` can still resolve a game, but the defensive `nullify()` path is blocked because `Verifier::nullify` requires the calling game to still be `proper`. Since `AnchorStateRegistry::isGameProper` returns `false` while paused, pause disables the verifier kill switch without actually stopping a game from being resolved.

## Description

In `multiproof`, each `AggregateVerifier` instance is one dispute game for one proposed root. A proposer creates a game with `DisputeGameFactory.createWithInitData(...)`, supplying an initial proof. The game verifies that proof in `AggregateVerifier::initializeWithInitData`. Later, another actor may add a second proof with `AggregateVerifier::verifyProposalProof`. Once enough time passes and the proof threshold is met, `AggregateVerifier::resolve` decides the winner.

There are two different defensive mechanisms in the game:

* `AggregateVerifier::challenge` is the normal path for disputing a TEE-backed proposal. It is only available when a TEE proof already exists and no ZK proof has been submitted yet, so it cannot be used against a ZK-only game.
* `AggregateVerifier::nullify` is meant for cases where a verifier has justified conflicting outputs and must be invalidated. After checking the conflicting proof, `AggregateVerifier::nullify` calls the verifier’s global kill switch through `IVerifier::nullify`.

That kill switch is implemented in `Verifier::nullify`. It only succeeds if the caller is both a respected game and a proper game. Whether a game is proper is determined by `AnchorStateRegistry::isGameProper`, and that function returns `false` whenever the system is paused:

```solidity
if (paused()) {
    return false;
}
```

So once the system is paused, `Verifier::nullify` is blocked for all games.

The problem is that pause does not block the rest of the game lifecycle consistently. `AggregateVerifier::resolve` has no pause check at all. It only checks that the game is still in progress, that the parent has resolved, that the resolution time has passed, and that enough proofs exist. `AggregateVerifier::closeGame` does check `ANCHOR_STATE_REGISTRY.paused()`, but that only blocks anchoring while the pause is active. Once the pause is lifted, the already-resolved game can still be closed and promoted.

Pause blocks the same-type verifier-nullification path for both TEE and ZK proofs, but the practical risk is highest for ZK-only games because they do not have the separate TEE->ZK challenge path.

This creates the following sequence:

1. A ZK-only game is created.
2. The system is paused.
3. Defenders find a proof that nullifies the ZK verifier but cannot use `nullify()` because it now reverts due to the pause check in `AnchorStateRegistry::isGameProper`.
4. The game is eventually resolved successfully with the defender winning because `AggregateVerifier::resolve` does not care that the system is paused.
5. After the pause is lifted, `AggregateVerifier::closeGame` promotes the already-resolved game into the anchor state.

## Impact

A root may still be elevated to the anchor state even if the verifier behind the proof is invalid. This happens because the system's paused state disables the nullification process, while still allowing the game's resolution process to continue.

This is a perfect match for the following high severity impact listed as in-scope: `Bypassing the soundness alert mechanism — two conflicting valid proofs of the same type (both TEE or both ZK) fail to trigger automatic game nullification`.

## Recommendation

Allow verifier nullification even when the system is paused.

## Proof of Concept

What it demonstrates:

1. A ZK-only game is created.
2. Pause is enabled.
3. `nullify()` fails through `Verifier.NotProperGame`.
4. The same paused game still resolves.
5. Pause is lifted.
6. `closeGame()` succeeds.
7. The resolved game becomes the new anchor.

Run as `forge test --match-path test/multiproof/AggregateVerifierPausePoC.t.sol` on branch `v8.1.0` after adding the following to the POC file:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.15;

import { ISystemConfig } from "interfaces/L1/ISystemConfig.sol";
import { Claim, GameStatus, Hash } from "src/dispute/lib/Types.sol";
import { AggregateVerifier } from "src/multiproof/AggregateVerifier.sol";
import { Verifier } from "src/multiproof/Verifier.sol";

import { BaseTest } from "./BaseTest.t.sol";

/// @notice PoC for: pause blocks verifier nullification but does not stop a ZK-only game from
///         resolving and later becoming the anchor.
contract AggregateVerifierPausePoCTest is BaseTest {
    function testPauseBlocksNullifyButNotResolutionOrLaterAnchoring() public {
        currentL2BlockNumber += BLOCK_INTERVAL;

        // Step 1: create a ZK-only game. This is important because challenge() is not available
        // without a TEE proof, so nullify() is the only onchain defense path.
        Claim rootClaim = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "zk-root")));
        bytes memory initialZkProof = _generateProof("zk-proof-1", AggregateVerifier.ProofType.ZK);

        AggregateVerifier game = _createAggregateVerifierGame(
            ZK_PROVER, rootClaim, currentL2BlockNumber, address(anchorStateRegistry), initialZkProof
        );

        // Step 2: pause the system.
        vm.mockCall(address(systemConfig), abi.encodeWithSelector(ISystemConfig.paused.selector), abi.encode(true));

        // Step 3: try to nullify the ZK proof with a conflicting ZK proof.
        // This fails because Verifier.nullify() requires isGameProper(), and paused games are not proper.
        Claim conflictingRoot = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "conflicting-zk-root")));
        bytes memory conflictingZkProof = _generateProof("zk-proof-2", AggregateVerifier.ProofType.ZK);

        vm.expectRevert(Verifier.NotProperGame.selector);
        game.nullify(conflictingZkProof, BLOCK_INTERVAL / INTERMEDIATE_BLOCK_INTERVAL - 1, conflictingRoot.raw());

        // Step 4: even though nullify() is blocked by pause, resolve() is still allowed.
        vm.warp(block.timestamp + 7 days);
        game.resolve();

        assertEq(uint8(game.status()), uint8(GameStatus.DEFENDER_WINS));

        // Step 5: lift the pause.
        vm.mockCall(address(systemConfig), abi.encodeWithSelector(ISystemConfig.paused.selector), abi.encode(false));

        // Step 6: after unpause, the already-resolved game can still be closed and promoted.
        vm.warp(block.timestamp + 1);
        game.closeGame();

        // Step 7: the resolved game is now the global anchor.
        (Hash anchorRoot, uint256 anchorL2BlockNumber) = anchorStateRegistry.getAnchorRoot();
        assertEq(Hash.unwrap(anchorRoot), rootClaim.raw());
        assertEq(anchorL2BlockNumber, currentL2BlockNumber);
    }
}
```
