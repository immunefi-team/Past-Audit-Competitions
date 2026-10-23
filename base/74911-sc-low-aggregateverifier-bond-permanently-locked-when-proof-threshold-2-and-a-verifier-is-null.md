# 74911 sc low aggregateverifier bond permanently locked when proof threshold 2 and a verifier is nullified resolve and claimcredit both revert

**Submitted on Apr 25th 2026 at 20:57:03 UTC by @Meadowlark14896 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74911
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hours (e.g., stuck withdrawal proofs, locked dispute game bonds)

## Description

### Summary

When `PROOF_THRESHOLD=2` (a value explicitly supported by the constructor validation at line 285), if either the TEE or ZK verifier is nullified, the game enters an unresolvable state where the bond is permanently locked. The `resolve()` function requires `proofCount >= PROOF_THRESHOLD`, but once a verifier is nullified, `proofCount` can never reach 2 again. Meanwhile, `claimCredit()` requires the game to be resolved (since `expectedResolution != type(uint64).max`), creating a deadlock where the bond cannot be claimed by anyone.

### Root Cause

In `AggregateVerifier.sol`, three conditions create the deadlock:

1. **Nullification kills a verifier permanently** (lines 594-601): `TEE_VERIFIER.nullify()` or `ZK_VERIFIER.nullify()` sets a global `nullified` flag with no undo mechanism. No further proofs of that type can be verified.
2. **`resolve()` requires `proofCount >= PROOF_THRESHOLD`** (line 458): With one verifier dead and `PROOF_THRESHOLD=2`, the game can never accumulate 2 proofs.
3. **`claimCredit()` 14-day fallback only works when `expectedResolution == type(uint64).max`** (lines 613-617): After a proof is submitted (moving `expectedResolution` away from `max`), the fallback path is disabled, and the function requires `resolvedAt != 0`.

```solidity
// Line 458 — resolve() blocks
if (proofCount < PROOF_THRESHOLD) revert NotEnoughProofs();

// Lines 613-617 — claimCredit() blocks
if (expectedResolution.raw() != type(uint64).max) {
    if (resolvedAt.raw() == 0) revert GameNotResolved();  // ← ALWAYS REVERTS
} else {
    if (block.timestamp < createdAt.raw() + 14 days) revert GameNotOver();
}
```

## Attack Scenarios

<details>

<summary>Path A — TEE nullified, then ZK proof submitted</summary>

1. Game created with TEE proof → `proofCount=1`, `expectedResolution = now + 7 days`
2. TEE proof nullified → `proofCount=0`, `expectedResolution = type(uint64).max`, `TEE_VERIFIER` dead
3. ZK proof submitted via `verifyProposalProof` → `proofCount=1`, `expectedResolution = now + 7 days`
4. After 7 days: `gameOver() = true`
5. `resolve()` → `proofCount(1) < PROOF_THRESHOLD(2)` → **REVERTS**
6. `claimCredit()` → `expectedResolution != max` → `resolvedAt == 0` → **REVERTS**
7. **Bond permanently locked**

</details>

<details>

<summary>Path B — ZK nullified after both proofs</summary>

1. TEE proof → `proofCount=1`
2. ZK proof → `proofCount=2`
3. ZK nullified → `proofCount=1`, `ZK_VERIFIER` dead
4. Cannot re-submit TEE (`AlreadyProven`) or ZK (`notNullified` modifier)
5. `resolve()` → `proofCount(1) < PROOF_THRESHOLD(2)` → **REVERTS**
6. **Bond permanently locked**

</details>

<details>

<summary>Path C — Challenge ZK nullified</summary>

1. TEE proof → `proofCount=1`
2. Challenge with ZK → `proofCount=2`, `counteredByIntermediateRootIndexPlusOne > 0`
3. ZK nullified (proves on-chain root correct) → `proofCount=1`, `ZK_VERIFIER` dead
4. `resolve()` → `proofCount(1) < PROOF_THRESHOLD(2)` → **REVERTS**
5. **Bond permanently locked**

</details>

## Impact

Permanent freezing of the proposer's bond. The only recovery path is `DelayedWETH.recover()` which requires the DelayedWETH owner (a multisig) to manually intervene. The `bondRecipient` (proposer or challenger) has no way to claim their funds.

The code explicitly supports `PROOF_THRESHOLD=2` via the constructor validation:

```solidity
if (proofThreshold != 1 && proofThreshold != 2) revert InvalidProofThreshold();
```

This means the protocol intends to support dual-proof configurations. Any deployment with `PROOF_THRESHOLD=2` is vulnerable to permanent bond locking whenever a verifier is nullified.

## Proof of Concept

Place in `test/multiproof/` and run with `forge test --match-test "test_BUG" -vvv`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Test, console} from "forge-std/Test.sol";
import {BaseTest} from "test/multiproof/BaseTest.t.sol";
import {AggregateVerifier} from "src/multiproof/AggregateVerifier.sol";
import {Claim, GameStatus, GameType, Hash, Proposal, Timestamp} from "src/dispute/lib/Types.sol";
import {IDisputeGame} from "interfaces/dispute/IDisputeGame.sol";
import {IVerifier} from "interfaces/multiproof/IVerifier.sol";
import {IAnchorStateRegistry} from "interfaces/dispute/IAnchorStateRegistry.sol";
import {IDelayedWETH} from "interfaces/dispute/IDelayedWETH.sol";
import {MockVerifier} from "src/multiproof/mocks/MockVerifier.sol";

contract ProofThreshold2Test is BaseTest {
    // Override PROOF_THRESHOLD to 2
    function setUp() public override {
        // Deploy everything manually with PROOF_THRESHOLD=2
        _deployContractsAndProxies();
        _initializeProxies();

        // Deploy AggregateVerifier with PROOF_THRESHOLD=2
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
            2  // PROOF_THRESHOLD = 2
        );

        factory.setImplementation(AGGREGATE_VERIFIER_GAME_TYPE, IDisputeGame(address(aggregateVerifierImpl)));
        factory.setInitBond(AGGREGATE_VERIFIER_GAME_TYPE, INIT_BOND);
        anchorStateRegistry.setRespectedGameType(AGGREGATE_VERIFIER_GAME_TYPE);
        vm.warp(block.timestamp + 1);
    }

    function test_BUG_bondLockedWhenVerifierNullified() public {
        // Step 1: Create game with TEE proof
        currentL2BlockNumber += BLOCK_INTERVAL;
        Claim rootClaim = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber)));
        bytes memory teeProof = _generateProof("tee", AggregateVerifier.ProofType.TEE);

        AggregateVerifier game = _createAggregateVerifierGame(
            TEE_PROVER, rootClaim, currentL2BlockNumber, address(anchorStateRegistry), teeProof
        );
        assertEq(game.proofCount(), 1);
        console.log("1. Game created with TEE proof, proofCount=1");

        // Step 2: Submit ZK proof
        bytes memory zkProof = _generateProof("zk", AggregateVerifier.ProofType.ZK);
        vm.prank(ZK_PROVER);
        game.verifyProposalProof(zkProof);
        assertEq(game.proofCount(), 2);
        console.log("2. ZK proof submitted, proofCount=2");

        // Step 3: ZK verifier is nullified (simulates ZK compromise detection)
        zkVerifier.setNullified(true);

        // Simulate nullify call effect on the game
        // In reality this happens via game.nullify() but we need a valid proof
        // Instead, let's demonstrate the state: if proofCount drops to 1 and ZK is dead

        // For this PoC, we directly show the deadlock condition:
        // After nullification, proofCount=1, PROOF_THRESHOLD=2, ZK_VERIFIER is dead

        // Step 4: Wait for game to be over
        vm.warp(block.timestamp + 7 days + 1);
        assertTrue(game.gameOver(), "Game should be over");

        // With proofCount=2, resolve works fine
        game.resolve();
        assertEq(uint8(game.status()), uint8(GameStatus.DEFENDER_WINS));
        console.log("3. This game resolves (proofCount=2 >= threshold=2)");

        // Now demonstrate the BROKEN case: create a new game
        currentL2BlockNumber += BLOCK_INTERVAL;
        rootClaim = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber)));
        teeProof = _generateProof("tee2", AggregateVerifier.ProofType.TEE);

        AggregateVerifier game2 = _createAggregateVerifierGame(
            TEE_PROVER, rootClaim, currentL2BlockNumber, address(game), teeProof
        );
        assertEq(game2.proofCount(), 1);
        console.log("4. New game created with TEE proof, proofCount=1");

        // ZK verifier is nullified — cannot submit ZK proof
        bytes memory zkProof2 = _generateProof("zk2", AggregateVerifier.ProofType.ZK);
        vm.prank(ZK_PROVER);
        vm.expectRevert(); // notNullified modifier blocks this
        game2.verifyProposalProof(zkProof2);
        console.log("5. ZK proof BLOCKED by notNullified — proofCount stuck at 1");

        // Wait for game to be over
        vm.warp(block.timestamp + 7 days + 1);
        assertTrue(game2.gameOver());

        // resolve() fails — proofCount < PROOF_THRESHOLD
        vm.expectRevert(AggregateVerifier.NotEnoughProofs.selector);
        game2.resolve();
        console.log("6. resolve() REVERTS: proofCount(1) < PROOF_THRESHOLD(2)");

        // claimCredit() fails — game not resolved
        vm.expectRevert(abi.encodeWithSignature("GameNotResolved()"));
        game2.claimCredit();
        console.log("7. claimCredit() REVERTS: game not resolved");

        // Even after 14 days — still fails because expectedResolution != max
        vm.warp(block.timestamp + 14 days);
        vm.expectRevert(abi.encodeWithSignature("GameNotResolved()"));
        game2.claimCredit();
        console.log("8. claimCredit() STILL REVERTS after 14 days");
        console.log("");
        console.log("RESULT: Bond is permanently locked. No recovery path for bondRecipient.");
    }
}
```

## Why This Is Not a Duplicate

* This is a code logic bug in `AggregateVerifier.sol` (`src/multiproof/`), not a deployment configuration issue.
* Not listed in the Known Vulnerabilities PDF.
* Not related to the `anchorGame` reset issue or any other previously reported finding.
* The code explicitly supports `PROOF_THRESHOLD=2` via constructor validation.

## Recommendation

Add a recovery mechanism when a verifier is nullified. For example, reduce `PROOF_THRESHOLD` dynamically when a verifier is nullified, or allow `claimCredit()` to use the 14-day fallback regardless of `expectedResolution`:

```solidity
function claimCredit() external nonReentrant {
    if (bondClaimed) revert NoCreditToClaim();

    // Allow 14-day fallback regardless of expectedResolution
    bool gameResolved = resolvedAt.raw() != 0;
    bool fallbackExpired = block.timestamp >= createdAt.raw() + 14 days;

    if (!gameResolved && !fallbackExpired) {
        revert GameNotResolved();
    }

    // ... rest of function
}
```

## Link to Proof of Concept

<https://gist.github.com/bigfootbjj/4c22790fc3dc4dfb30c06cf55292fa30>
