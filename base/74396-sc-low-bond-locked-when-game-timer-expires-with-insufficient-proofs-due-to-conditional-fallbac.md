# 74396 sc low bond locked when game timer expires with insufficient proofs due to conditional fallback gap in claimcredit

**Submitted on Apr 22nd 2026 at 08:53:01 UTC by @flopcatcher for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74396
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hours (e.g., stuck withdrawal proofs, locked dispute game bonds)

## Description

### Brief/Intro

The `claimCredit()` function has a 14-day bond recovery fallback, but it only activates when `expectedResolution == type(uint64).max` (i.e., zero proofs submitted). When at least one proof exists, `expectedResolution` is set to a finite timestamp (e.g., `block.timestamp + 7 days`), and the 14-day fallback path is unreachable.

### Vulnerability Details

With `PROOF_THRESHOLD=2`, if only one proof type is submitted before the 7-day `expectedResolution` timer expires, the game enters an irrecoverable dead state:

1. `resolve()` fails: `proofCount(1) < PROOF_THRESHOLD(2)` reverts with `NotEnoughProofs()`
2. `verifyProposalProof()` fails: `gameOver()` returns true, reverts with `GameOver()`
3. `claimCredit()` fails: `expectedResolution != type(uint64).max` enters the first branch, which requires `resolvedAt != 0`, but the game can never resolve

The code comment at line 610-612 states "The game must have resolved or 14 days have passed since creation" (an OR condition), but the implementation uses conditional branching on `expectedResolution` that restricts the 14-day fallback to only the no-proofs case.

The issue has the same root cause as `1.1 Unconditional Proof Threshold Check in resolve Blocks Normal Bond Recovery When Parent Game Is Invalid` from Cantina Multiproof security report, BUT the currently reported issue is a broader dead state scenario, (especially for first games or games with valid parents) and the specific `claimCredit()` conditional fallback gap. Fixe introduced in commit `dd587c9adc84a768eb540a88ef479275c5db97e9` addresses only the `resolve()` path when the parent is invalid.

### Impact Details

For first games (parent = AnchorStateRegistry), there is no instant recovery path since `_getParentGameStatus()` always returns `DEFENDER_WINS`. Impact is systemic when a verifier is globally nullified: all in-progress games needing the nullified verifier type and having only one proof submitted become unresolvable with temporaly locked bonds, until governance / ProxyAdmin owner use `hold` / `recover`. That is at least 24h in any realistic scenario.

## References

1. Cantina report: <https://cantina.xyz/portfolio/25ba64ea-d6f3-411e-8338-419ffc385ba6?utm\\_source=immunefi>

## Proof of Concept

* Add the following test to `test/multiproof`
* Run `forge test --mt test_PoC`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.15;

import { GameNotResolved } from "src/dispute/lib/Errors.sol";
import { IAnchorStateRegistry } from "interfaces/dispute/IAnchorStateRegistry.sol";
import { IDelayedWETH } from "interfaces/dispute/IDelayedWETH.sol";
import { IDisputeGame } from "interfaces/dispute/IDisputeGame.sol";
import { Claim } from "src/dispute/lib/Types.sol";

import { AggregateVerifier } from "src/multiproof/AggregateVerifier.sol";
import { IVerifier } from "interfaces/multiproof/IVerifier.sol";

import { BaseTest } from "./BaseTest.t.sol";

contract AggregateVerifierPoC is BaseTest {
    /// Register `AggregateVerifier` with `proofThreshold == 2`
    function setUp() public override {
        super.setUp();
        AggregateVerifier implWithThresholdTwo = new AggregateVerifier(
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
        factory.setImplementation(AGGREGATE_VERIFIER_GAME_TYPE, IDisputeGame(address(implWithThresholdTwo)));
    }

    function test_PoC() public {
        currentL2BlockNumber += BLOCK_INTERVAL;
        Claim rootClaim = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber)));
        bytes memory teeProof = _generateProof("tee-proof", AggregateVerifier.ProofType.TEE);
        bytes memory zkProof = _generateProof("zk-proof", AggregateVerifier.ProofType.ZK);

        AggregateVerifier game = _createAggregateVerifierGame(
            TEE_PROVER, rootClaim, currentL2BlockNumber, address(anchorStateRegistry), teeProof
        );

        assertEq(game.PROOF_THRESHOLD(), 2);
        assertEq(game.proofCount(), 1);
        assertTrue(game.expectedResolution().raw() != type(uint64).max);

        uint256 created = game.createdAt().raw();

        vm.warp(block.timestamp + 7 days + 1);
        assertTrue(game.gameOver());

        vm.expectRevert(AggregateVerifier.NotEnoughProofs.selector);
        game.resolve();

        vm.expectRevert(AggregateVerifier.GameOver.selector);
        _provideProof(game, ZK_PROVER, zkProof);

        vm.warp(created + 14 days + 1);
        vm.expectRevert(GameNotResolved.selector);
        game.claimCredit();

        assertEq(delayedWETH.balanceOf(address(game)), INIT_BOND);
    }
}
```
