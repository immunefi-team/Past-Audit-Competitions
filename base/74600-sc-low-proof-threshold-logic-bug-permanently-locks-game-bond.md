# 74600 sc low proof threshold logic bug permanently locks game bond

Submitted on Apr 23rd 2026 at 18:21:49 UTC by @magtentic for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74600
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Permanent freezing of funds in the bridge or in dispute game bonds with no available recovery path

## Description

### Brief/Intro

In `AggregateVerifier`, games with `PROOF_THRESHOLD = 2` must receive two proofs before resolving. The first proof resets the finalization delay (from ∞ to 7 days) but without a second proof, the game can never satisfy `resolve()`. The bond then becomes irrecoverable, freezing ETH indefinitely.

### Vulnerability Details

Upon initialization, `expectedResolution` is set to `type(uint64).max`.

```solidity
        // Set expected resolution.
        expectedResolution = Timestamp.wrap(type(uint64).max);
```

When a valid proof is submitted (`_proofVerifiedUpdate` is called), `proofCount` increments to 1 and `_decreaseExpectedResolution` sets `expectedResolution = block.timestamp + SLOW_FINALIZATION_DELAY` (7 days).

```solidity
        proofCount += 1;
        _decreaseExpectedResolution();
```

```solidity
    function _decreaseExpectedResolution() internal {
        uint64 delay = _getDelay();
...
        uint64 newResolution = uint64(block.timestamp) + delay;
        expectedResolution = Timestamp.wrap(uint64(FixedPointMathLib.min(newResolution, expectedResolution.raw())));
    }
```

```solidity
    function _getDelay() internal view returns (uint64) {
...
        } else if (proofCount == 1) {
            return SLOW_FINALIZATION_DELAY;
...
    }
```

Thus after the first proof, `expectedResolution` is finite (7 days). However, `resolve()` requires `gameOver()` and `proofCount >= PROOF_THRESHOLD`.

```solidity
    function resolve() external returns (GameStatus) {
...
            // Game must be completed with a valid proof and enough proofs.
            if (!gameOver()) revert GameNotOver();
            if (proofCount < PROOF_THRESHOLD) revert NotEnoughProofs();
...
    }
```

If no second proof arrives, `proofCount < 2` and `resolve()` always reverts `NotEnoughProofs()`.

Meanwhile, `claimCredit()` checks if `expectedResolution != max` and requires the game to have `resolvedAt != 0`.

```solidity
    function claimCredit() external nonReentrant {
...
        if (expectedResolution.raw() != type(uint64).max) {
            if (resolvedAt.raw() == 0) revert GameNotResolved();
...
```

Since `resolvedAt` remains 0 (game never resolves), `claimCredit()` also reverts. The 14-day fallback timeout is only reachable when `expectedResolution == max`, but here it was lowered, so that branch is skipped.

```solidity
        } else {
            if (block.timestamp < createdAt.raw() + 14 days) revert GameNotOver();
        }
```

The net effect is a permanent deadlock: the game stays `IN_PROGRESS`, the bond is locked in `DelayedWETH`, and no child games can proceed.

## Impact Details

An honest proposer who posts one proof will have their entire ETH bond effectively locked forever if the second proof is missing or too costly to obtain. This violates in-scope impacts (“Permanent freezing of funds in dispute game bonds”) and blocks all dependent games. The protocol cannot finalize this proposal nor refund the bond without an explicit fix.

## References

[AggregateVerifier](https://github.com/base/contracts/blob/v8.1.0/src/multiproof/AggregateVerifier.sol)

* `initializeWithInitData` sets `expectedResolution = max`
* `_proofVerifiedUpdate` lowers it to `block.timestamp + delay`
* In `resolve()`, the code requires `proofCount >= PROOF_THRESHOLD`
* In `claimCredit()`, if `expectedResolution` is not `max`, `resolvedAt` must be non-zero

## Proof of Concept

Copy the below PoC to `test/AggregateVerifierThresholdTwoBondLockTest .t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.15;

import { GameNotResolved } from "src/dispute/lib/Errors.sol";
import { Claim, GameStatus } from "src/dispute/lib/Types.sol";
import { IDisputeGame } from "interfaces/dispute/IDisputeGame.sol";
import { AggregateVerifier } from "src/multiproof/AggregateVerifier.sol";
import { IAnchorStateRegistry } from "interfaces/dispute/IAnchorStateRegistry.sol";
import { IDelayedWETH } from "interfaces/dispute/IDelayedWETH.sol";
import { IVerifier } from "interfaces/multiproof/IVerifier.sol";

import { BaseTest } from "./multiproof/BaseTest.t.sol";

contract AggregateVerifierThresholdTwoBondLockTest is BaseTest {
    function setUp() public override {
        super.setUp();

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
    }

    function test_singleProofCannotResolveOrRecoverBondAndBlocksChildResolution() public {
        currentL2BlockNumber += BLOCK_INTERVAL;
        Claim parentRootClaim = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "parent")));
        bytes memory parentTeeProof = _generateProof("parent-tee", AggregateVerifier.ProofType.TEE);

        AggregateVerifier parentGame = _createAggregateVerifierGame(
            TEE_PROVER, parentRootClaim, currentL2BlockNumber, address(anchorStateRegistry), parentTeeProof
        );

        assertEq(parentGame.proofCount(), 1);
        assertEq(uint8(parentGame.status()), uint8(GameStatus.IN_PROGRESS));
        assertEq(parentGame.resolvedAt().raw(), 0);
        assertEq(parentGame.expectedResolution().raw(), block.timestamp + 7 days);

        currentL2BlockNumber += BLOCK_INTERVAL;
        Claim childRootClaim = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "child")));
        bytes memory childTeeProof = _generateProof("child-tee", AggregateVerifier.ProofType.TEE);
        bytes memory childZkProof = _generateProof("child-zk", AggregateVerifier.ProofType.ZK);

        AggregateVerifier childGame =
            _createAggregateVerifierGame(TEE_PROVER, childRootClaim, currentL2BlockNumber, address(parentGame), childTeeProof);
        _provideProof(childGame, ZK_PROVER, childZkProof);

        assertEq(childGame.proofCount(), 2);
        assertEq(childGame.expectedResolution().raw(), block.timestamp + 1 days);

        vm.warp(block.timestamp + 7 days + 1);

        vm.expectRevert(AggregateVerifier.NotEnoughProofs.selector);
        parentGame.resolve();

        assertEq(uint8(parentGame.status()), uint8(GameStatus.IN_PROGRESS));
        assertEq(parentGame.resolvedAt().raw(), 0);

        vm.expectRevert(AggregateVerifier.ParentGameNotResolved.selector);
        childGame.resolve();

        vm.warp(parentGame.createdAt().raw() + 14 days + 1);

        vm.expectRevert(GameNotResolved.selector);
        parentGame.claimCredit();

        assertEq(uint8(parentGame.status()), uint8(GameStatus.IN_PROGRESS));
        assertEq(parentGame.resolvedAt().raw(), 0);
        assertEq(delayedWETH.balanceOf(address(parentGame)), INIT_BOND);
    }
}
```

Run the below to execute

```bash
forge test --match-contract AggregateVerifierThresholdTwoBondLockTest -vvvv
```
