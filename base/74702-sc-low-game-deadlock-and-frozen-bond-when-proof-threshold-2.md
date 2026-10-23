# 74702 sc low game deadlock and frozen bond when proof threshold 2

**Submitted on Apr 24th 2026 at 11:36:17 UTC by @silverologist for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74702
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Permanent freezing of funds in the bridge or in dispute game bonds with no available recovery path

## Description

## Summary

`AggregateVerifier` explicitly supports `PROOF_THRESHOLD == 2`, but its state machine does not actually support the case where a game still has only one proof when the single-proof timer expires. At that point, the game becomes permanently stuck: no second proof can ever be added, `resolve()` can never succeed, and `claimCredit()` can never release the bond.

The cleanest manifestation is a ZK-first game. A game initialized with a ZK proof gets the normal 7-day single-proof timer. After that timer expires, adding the second proof is blocked by `GameOver`, `resolve()` still reverts with `NotEnoughProofs`, and there is no `challenge()` escape hatch because `challenge()` only works when a TEE proof already exists. The dispute bond is therefore permanently frozen.

## Detailed Description

The constructor accepts both `1` and `2` as valid proof thresholds:

```solidity
if (proofThreshold != 1 && proofThreshold != 2) revert InvalidProofThreshold();
```

Whenever a proof is accepted, `_proofVerifiedUpdate()`:

1. stores the prover,
2. increments `proofCount`,
3. calls `_decreaseExpectedResolution()`.

For `proofCount == 1`, `_getDelay()` returns `SLOW_FINALIZATION_DELAY`, which is 7 days. That means a game with only one proof gets a finite resolution deadline even if `PROOF_THRESHOLD == 2`.

Once the 7-day timer expires, the second proof is blocked forever.

`verifyProposalProof()` begins with:

```solidity
if (gameOver()) revert GameOver();
```

And `gameOver()` is:

```solidity
return expectedResolution.raw() <= block.timestamp;
```

So once the game reaches its single-proof deadline, the second proof can no longer be added at all.

This is fatal under `PROOF_THRESHOLD == 2`, because the game still needs a second proof but the contract has already closed the proof-submission window.

Even though the proof-submission window has closed, `resolve()` still requires:

```solidity
if (proofCount < PROOF_THRESHOLD) revert NotEnoughProofs();
```

So the game enters a contradictory state:

* it is over for proof-submission purposes,
* but it is not complete enough for resolution purposes.

That is the deadlock.

`claimCredit()` has two modes:

* if `expectedResolution != type(uint64).max`, the game must already be resolved,
* otherwise, if `expectedResolution == type(uint64).max`, there is a 14-day fallback path.

But in this deadlocked state:

* `expectedResolution` is finite, because the game still has one proof,
* `resolvedAt` is zero, because `resolve()` can never succeed.

So `claimCredit()` keeps reverting with `GameNotResolved()` forever.

That is why the bond is permanently frozen.

## Concrete Exploit / Failure Sequence

The failure sequence is:

1. Deploy or configure `AggregateVerifier` with `proofThreshold = 2`.
2. Create a game initialized with a single ZK proof.
3. Wait until `SLOW_FINALIZATION_DELAY` elapses.
4. Attempt to submit the second proof: revert with `GameOver`.
5. Attempt to resolve: revert with `NotEnoughProofs`.
6. Wait arbitrarily long.
7. Attempt to claim the bond: revert with `GameNotResolved`.

At that point the game is stuck and the bond is permanently locked.

## Impact

A game can become permanently unresolvable and the bond can become permanently unrecoverable.

Under the contest's severity guide, this maps directly to `Permanent freezing of funds in the bridge or in dispute game bonds with no available recovery path`.

## Root Cause

The bug comes from three independent pieces of logic that are internally inconsistent when `PROOF_THRESHOLD == 2`:

1. **Timer assignment** is based on current `proofCount`.
2. **Proof submission closure** is based on `gameOver()`.
3. **Resolution eligibility** is based on `proofCount >= PROOF_THRESHOLD`.

For one-proof games under threshold two:

* the timer says the game is mature after 7 days,
* proof submission says the game is closed after 7 days,
* resolution says the game is still incomplete forever.

That combination should never be possible in a correct state machine.

## Recommended Fix

Once a one-proof game expires under `PROOF_THRESHOLD == 2`, the contract should move into an explicit terminal refund state.

## PoC

Add the following to ./contracts/test/multiproof/AggregateVerifierThresholdTwoPoC.t.sol and run as `forge test --match-path test/multiproof/AggregateVerifierThresholdTwoPoC.t.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.15;

import { IDisputeGame } from "interfaces/dispute/IDisputeGame.sol";
import { IAnchorStateRegistry } from "interfaces/dispute/IAnchorStateRegistry.sol";
import { IDelayedWETH } from "interfaces/dispute/IDelayedWETH.sol";
import { IDisputeGameFactory } from "interfaces/dispute/IDisputeGameFactory.sol";
import { Claim } from "src/dispute/lib/Types.sol";
import { AggregateVerifier } from "src/multiproof/AggregateVerifier.sol";
import { IVerifier } from "interfaces/multiproof/IVerifier.sol";
import { GameNotResolved } from "src/dispute/lib/Errors.sol";

import { BaseTest } from "./BaseTest.t.sol";

/// @notice PoC for: proofThreshold == 2 can permanently deadlock a game and its bond after the first proof ages out.
contract AggregateVerifierThresholdTwoPoCTest is BaseTest {
    function testThresholdTwoCanPermanentlyDeadlockGame() public {
        AggregateVerifier aggregateVerifierImpl = new AggregateVerifier(
            AGGREGATE_VERIFIER_GAME_TYPE,
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
            2
        );

        factory.setImplementation(AGGREGATE_VERIFIER_GAME_TYPE, IDisputeGame(address(aggregateVerifierImpl)));

        currentL2BlockNumber += BLOCK_INTERVAL;

        Claim rootClaim = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "zk-first-root")));
        bytes memory initialZkProof = _generateProof("zk-proof-1", AggregateVerifier.ProofType.ZK);

        AggregateVerifier game = _createAggregateVerifierGame(
            ZK_PROVER, rootClaim, currentL2BlockNumber, address(anchorStateRegistry), initialZkProof
        );

        bytes memory laterTeeProof = _generateProof("tee-proof-1", AggregateVerifier.ProofType.TEE);

        vm.warp(block.timestamp + 7 days + 1);

        vm.expectRevert(AggregateVerifier.GameOver.selector);
        vm.prank(TEE_PROVER);
        game.verifyProposalProof(laterTeeProof);

        vm.expectRevert(AggregateVerifier.NotEnoughProofs.selector);
        game.resolve();

        vm.warp(block.timestamp + 30 days);

        vm.expectRevert(GameNotResolved.selector);
        game.claimCredit();
    }
}
```

Observed result from validation:

```
Ran 1 test for test/multiproof/AggregateVerifierThresholdTwoPoC.t.sol:AggregateVerifierThresholdTwoPoCTest
[PASS] testThresholdTwoCanPermanentlyDeadlockGame() (gas: 3743661)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.94ms (466.95µs CPU time)
```
