# 74427 sc low permanent freezing of first game bonds with no recovery path cascading liveness halt on descendants proof threshold 2&#x20;

**Submitted on Apr 22nd 2026 at 13:38:04 UTC by @stendal for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74427
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Permanent freezing of funds in the bridge or in dispute game bonds with no available recovery path

## Description

With `PROOF_THRESHOLD=2` (valid per the `InvalidProofThreshold` guard in the constructor), any first game whose parent is `ANCHOR_STATE_REGISTRY` and that reaches `gameOver()` with a single proof submitted cannot be resolved, cannot have credit claimed, and has no admin escape. The bond locks forever.

The exact chain through the code:

* `resolve()` reverts at `AggregateVerifier.sol:458` with `NotEnoughProofs` because `proofCount == 1 < PROOF_THRESHOLD == 2`.
* `claimCredit()` at `:613-617` branches on `expectedResolution == type(uint64).max`. That equality comes from `_getDelay()` at `:821` only when `proofCount == 0`. With `proofCount == 1`, `_getDelay()` returns `SLOW_FINALIZATION_DELAY` (7 days) — a finite value — so `claimCredit` stays on the `resolvedAt != 0` branch and reverts with `GameNotResolved`. The 14-day timeout fallback is unreachable.
* `verifyProposalProof()` at `:339` contains `if (gameOver()) revert GameOver()`. Once the 7-day window lapses, the missing proof type cannot be supplied — even if the second prover comes back online the next block.
* `challenge()` does not rescue the game either. `_checkIntermediateRoot` at `:1003` requires `intermediateRootToProve != intermediateOutputRoot(index)`. A late, honest proof that agrees with the claim is rejected.
* `nullify()` needs a second valid proof for a different state. Without a soundness break in SP1 or Nitro attestation, that path is closed.

Admin-level actions I checked all fail for first games:

* `blacklistDisputeGame(stuckGame)`: `resolve()` consults `isGameBlacklisted(parentGame)` inside `_getParentGameStatus` at `:938`. For a first game `parentAddress() == ANCHOR_STATE_REGISTRY`, so execution falls into the `return GameStatus.DEFENDER_WINS` branch at `:943` without ever checking the stuck game's own blacklist flag. Blacklisting is a no-op.
* `setRespectedGameType(other)`: irrelevant — `_getParentGameStatus` returns `DEFENDER_WINS` for `parent == ASR` regardless of the respected type.
* `updateRetirementTimestamp()`: retirement is checked only on parent games. The parent here is ASR, which is not a game.
* `setImplementation(newImpl)` on the factory: affects newly-created clones only. Existing EIP-1167 minimal proxies carry the old implementation address hardcoded in runtime bytecode and cannot be upgraded.

The comment inside `_increaseExpectedResolution` at `:809-811` reads:

> "we give enough time to resolve the issue and possibly blacklist this game"

The developer-documented recovery expectation does not hold for first games. Blacklisting the stuck game itself is a no-op (see above); the comment describes a mechanism the contract does not provide in this scenario.

### Cascade to descendant games

Stuck first games freeze the subtree built on top of them. Any child game that declares the stuck parent as its `parentAddress()` has `_getParentGameStatus` return `IN_PROGRESS` (live call to the stuck parent), and `resolve()` on the child reverts with `ParentGameNotResolved`. The child cannot close regardless of its own `proofCount`.

The only way to unblock descendants is an off-chain admin call to `blacklistDisputeGame(stuckParent)`. After that, the child's `_getParentGameStatus` hits the blacklist branch at `AggregateVerifier.sol:941-943` and returns `CHALLENGER_WINS`, the child's `resolve()` takes the branch at `:453-454` and sets `status = CHALLENGER_WINS` without overwriting `bondRecipient` (which was set to `gameCreator()` at `:408` during initialization). Descendant proposers get their bonds back after the manual admin step. The first game's bond remains frozen.

Net effect: a proof-system hiccup that leaves one first game with `proofCount=1` forces a manual admin response before the dispute tree below it can progress.

### Trigger conditions

`PROOF_THRESHOLD=2` plus one-of:

* ZK prover (SP1 / Succinct gateway) downtime past the 7-day window
* TEE outage (AWS Nitro availability, enclave key issues) with ZK in first
* Cert-chain expiry on Nitro between proof generation attempts
* Any deployment event where one proof type is available within 7 days and the other is not

The 7-day `SLOW_FINALIZATION_DELAY` is the grace period. If the second proof lands inside that window, `resolve()` closes normally. The bug is only reachable when the window elapses with `proofCount == 1`.

### Impact scaling

* Per game: `bondAmount` frozen, permanently.
* Partial prover outage of `N` days at rate `K` first-games/day: roughly `N × K × bondAmount` permanently frozen, plus the full subtree below each stuck game requires manual admin blacklist to unblock.
* `proofThreshold == 2` is one of two branches explicitly admitted by the constructor. Whatever branch the contract admits must be safe under the audit scope.

## Proof of Concept

### Target

* Repo: `github.com/base/contracts` at tag `v8.1.0` (commit `01dad230390cd69bcf130b5fc7a7a580b31650a7`)
* File under test: `src/multiproof/AggregateVerifier.sol`
* Test file to add: `test/multiproof/StuckBond_PROOF_THRESHOLD_2.t.sol`
* Tooling: Foundry `forge 1.5.1-stable`, solc `0.8.15` (matches the contract pragma)

The PoC exercises the full production stack — real `AnchorStateRegistry`, `DisputeGameFactory`, `DelayedWETH`, and `AggregateVerifier` — all from `src/dispute/` and `src/multiproof/` of the audited repository. The only substitution is `src/multiproof/mocks/MockVerifier.sol`, whose `verify()` returns `true`. The stuck-bond mechanics live entirely inside the `AggregateVerifier` state machine (proofCount tracking, expectedResolution progression, gameOver gating, parent-status dispatch) and do not depend on TEE/ZK proof internals. Plugging in SP1 + Nitro in place of `MockVerifier` produces the same revert sequence.

### Setup

```bash
git clone https://github.com/base/contracts
cd contracts
git checkout v8.1.0
make deps    # or manually: forge install … (see Makefile `deps` target)
```

Save the file below as `test/multiproof/StuckBond_PROOF_THRESHOLD_2.t.sol`, then:

```bash
forge test --ffi \
  --match-path "test/multiproof/StuckBond_PROOF_THRESHOLD_2.t.sol" -vv
```

### Test file (runnable, complete)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import { BaseTest } from "./BaseTest.t.sol";
import { AggregateVerifier } from "src/multiproof/AggregateVerifier.sol";
import { IAnchorStateRegistry } from "interfaces/dispute/IAnchorStateRegistry.sol";
import { IDelayedWETH } from "interfaces/dispute/IDelayedWETH.sol";
import { IDisputeGame } from "interfaces/dispute/IDisputeGame.sol";
import { IVerifier } from "interfaces/multiproof/IVerifier.sol";
import { Claim, GameStatus, GameType, Hash, Timestamp } from "src/dispute/lib/Types.sol";
import { GameNotResolved } from "src/dispute/lib/Errors.sol";

/// @title StuckBondProofThreshold2Test
/// @notice Reproduces the PROOF_THRESHOLD=2 permanent bond freeze against the
///         real Base Azul multiproof infrastructure. Extends BaseTest and
///         overrides PROOF_THRESHOLD to 2 in setUp.
contract StuckBondProofThreshold2Test is BaseTest {
    uint256 internal constant OVERRIDE_PROOF_THRESHOLD = 2;

    function setUp() public override {
        _deployContractsAndProxies();
        _initializeProxies();

        // Redeploy AggregateVerifier with PROOF_THRESHOLD=2 (valid per InvalidProofThreshold guard)
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
            OVERRIDE_PROOF_THRESHOLD
        );

        factory.setImplementation(AGGREGATE_VERIFIER_GAME_TYPE, IDisputeGame(address(impl)));
        factory.setInitBond(AGGREGATE_VERIFIER_GAME_TYPE, INIT_BOND);
        anchorStateRegistry.setRespectedGameType(AGGREGATE_VERIFIER_GAME_TYPE);
        vm.warp(block.timestamp + 1);
    }

    function _createFirstGameWithTeeOnly() internal returns (AggregateVerifier game) {
        currentL2BlockNumber = BLOCK_INTERVAL;
        Claim rootClaim = Claim.wrap(keccak256(abi.encode("root", currentL2BlockNumber)));
        bytes memory teeProof = _generateProof(bytes("tee"), AggregateVerifier.ProofType.TEE);

        // parent = AnchorStateRegistry → first game
        game = _createAggregateVerifierGame(
            address(this),
            rootClaim,
            currentL2BlockNumber,
            address(anchorStateRegistry),
            teeProof
        );

        assertEq(game.proofCount(), 1, "initial TEE proof submitted");
        assertEq(game.PROOF_THRESHOLD(), OVERRIDE_PROOF_THRESHOLD, "threshold=2");
    }

    /// Core: at gameOver() with proofCount=1, both resolve() and claimCredit() revert.
    function test_StuckBond_ResolveAndClaimCreditBothRevert() public {
        AggregateVerifier game = _createFirstGameWithTeeOnly();
        vm.warp(game.expectedResolution().raw() + 1);
        assertTrue(game.gameOver());

        vm.expectRevert(AggregateVerifier.NotEnoughProofs.selector);
        game.resolve();

        vm.expectRevert(GameNotResolved.selector);
        game.claimCredit();
    }

    /// The 14-day timeout fallback is unreachable when expectedResolution != max.
    function test_StuckBond_StillStuckAfter14Days() public {
        AggregateVerifier game = _createFirstGameWithTeeOnly();
        vm.warp(game.expectedResolution().raw() + 1 + 14 days);

        vm.expectRevert(AggregateVerifier.NotEnoughProofs.selector);
        game.resolve();
        vm.expectRevert(GameNotResolved.selector);
        game.claimCredit();

        assertEq(game.bondAmount(), INIT_BOND);
    }

    /// verifyProposalProof blocks late second proof via `if (gameOver()) revert GameOver()`.
    function test_StuckBond_CannotAddSecondProofAfterGameOver() public {
        AggregateVerifier game = _createFirstGameWithTeeOnly();
        vm.warp(game.expectedResolution().raw() + 1);

        bytes memory zkProof = _generateProof(bytes("zk"), AggregateVerifier.ProofType.ZK);
        vm.expectRevert(AggregateVerifier.GameOver.selector);
        vm.prank(ZK_PROVER);
        game.verifyProposalProof(zkProof);
    }

    /// Admin blacklist of the stuck game is a no-op for first games.
    function test_StuckBond_BlacklistSelfDoesNotUnstick() public {
        AggregateVerifier game = _createFirstGameWithTeeOnly();
        vm.warp(game.expectedResolution().raw() + 1);

        anchorStateRegistry.blacklistDisputeGame(IDisputeGame(address(game)));
        assertTrue(anchorStateRegistry.isGameBlacklisted(IDisputeGame(address(game))));

        vm.expectRevert(AggregateVerifier.NotEnoughProofs.selector);
        game.resolve();
        vm.expectRevert(GameNotResolved.selector);
        game.claimCredit();
    }

    /// Changing respectedGameType cannot rescue — _getParentGameStatus hardcodes DEFENDER_WINS.
    function test_StuckBond_ChangeRespectedGameTypeDoesNotUnstick() public {
        AggregateVerifier game = _createFirstGameWithTeeOnly();
        vm.warp(game.expectedResolution().raw() + 1);

        anchorStateRegistry.setRespectedGameType(GameType.wrap(999));

        vm.expectRevert(AggregateVerifier.NotEnoughProofs.selector);
        game.resolve();
    }

    /// Retirement timestamp is parent-only; ASR parent is not a game.
    function test_StuckBond_RetirementTimestampDoesNotUnstick() public {
        AggregateVerifier game = _createFirstGameWithTeeOnly();
        vm.warp(game.expectedResolution().raw() + 1);

        anchorStateRegistry.updateRetirementTimestamp();

        vm.expectRevert(AggregateVerifier.NotEnoughProofs.selector);
        game.resolve();
    }

    /// Factory impl upgrade only affects new clones; existing EIP-1167 clones are immutable.
    function test_StuckBond_SetImplementationDoesNotUnstickExistingGame() public {
        AggregateVerifier game = _createFirstGameWithTeeOnly();
        vm.warp(game.expectedResolution().raw() + 1);

        // Deploy a hypothetical "fixed" impl with threshold=1
        AggregateVerifier fixedImpl = new AggregateVerifier(
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
            1
        );
        factory.setImplementation(AGGREGATE_VERIFIER_GAME_TYPE, IDisputeGame(address(fixedImpl)));

        vm.expectRevert(AggregateVerifier.NotEnoughProofs.selector);
        game.resolve();

        assertEq(game.PROOF_THRESHOLD(), 2, "old impl threshold baked into clone bytecode");
    }

    /// expectedResolution is finite (7d) — the 14-day recovery branch is unreachable.
    function test_StuckBond_ExpectedResolutionNotMax() public {
        AggregateVerifier game = _createFirstGameWithTeeOnly();

        uint64 er = game.expectedResolution().raw();
        assertTrue(er != type(uint64).max);
        assertTrue(er > 0);
    }

    /// Cascade: stuck parent blocks children via ParentGameNotResolved.
    /// Admin blacklist of parent unblocks children as CHALLENGER_WINS.
    /// Parent's own bond remains frozen.
    function test_StuckBond_CascadeToChildren() public {
        AggregateVerifier parent = _createFirstGameWithTeeOnly();
        vm.warp(parent.expectedResolution().raw() + 1);

        currentL2BlockNumber = BLOCK_INTERVAL * 2;
        Claim childClaim = Claim.wrap(keccak256(abi.encode("child-root", currentL2BlockNumber)));
        bytes memory teeProof = _generateProof(bytes("child-tee"), AggregateVerifier.ProofType.TEE);

        AggregateVerifier child = _createAggregateVerifierGame(
            address(this),
            childClaim,
            currentL2BlockNumber,
            address(parent),
            teeProof
        );

        // Give child both proofs
        bytes memory zkProof = _generateProof(bytes("child-zk"), AggregateVerifier.ProofType.ZK);
        _provideProof(child, ZK_PROVER, zkProof);
        assertEq(child.proofCount(), 2);

        vm.warp(child.expectedResolution().raw() + 1);

        // Child can't resolve — parent stuck in IN_PROGRESS forever
        vm.expectRevert(AggregateVerifier.ParentGameNotResolved.selector);
        child.resolve();

        // Admin blacklists stuck parent → child resolves as CHALLENGER_WINS
        anchorStateRegistry.blacklistDisputeGame(IDisputeGame(address(parent)));
        GameStatus childStatus = child.resolve();
        assertEq(uint256(childStatus), uint256(GameStatus.CHALLENGER_WINS));
        assertEq(child.bondRecipient(), address(this));

        // Parent's own bond still stuck forever
        vm.expectRevert(AggregateVerifier.NotEnoughProofs.selector);
        parent.resolve();
    }

    /// Scale: N first games during partial prover outage all get stuck.
    /// Funds-at-risk quantified: 10 games × 1 ETH = 10 ETH on one 7-day outage.
    function test_StuckBond_MultipleGamesStuckDuringOutage() public {
        uint256 numGames = 10;
        AggregateVerifier[] memory games = new AggregateVerifier[](numGames);

        // All first games must be at l2Seq == BLOCK_INTERVAL (next anchor step).
        // Competing proposers submit games with different rootClaims — all get stuck.
        currentL2BlockNumber = BLOCK_INTERVAL;
        for (uint256 i = 0; i < numGames; i++) {
            Claim rootClaim = Claim.wrap(keccak256(abi.encode("outage-root", i)));
            bytes memory teeProof = _generateProof(
                bytes.concat("outage-tee", bytes(abi.encode(i))),
                AggregateVerifier.ProofType.TEE
            );

            games[i] = _createAggregateVerifierGame(
                address(this),
                rootClaim,
                currentL2BlockNumber,
                address(anchorStateRegistry),
                teeProof
            );
        }

        // 7-day ZK outage: no ZK proofs submitted
        vm.warp(games[0].expectedResolution().raw() + 1);

        uint256 totalStuckBonds;
        for (uint256 i = 0; i < numGames; i++) {
            vm.expectRevert(AggregateVerifier.NotEnoughProofs.selector);
            games[i].resolve();
            totalStuckBonds += games[i].bondAmount();
        }

        // Asserts 10 bonds permanently frozen
        assertEq(totalStuckBonds, numGames * INIT_BOND);
        // 10 × 1 ETH = 10 ETH permanently stuck on a single 7-day partial prover outage
    }
}
```

### Expected output (actual `forge test` run against v8.1.0)

```
$ forge test --ffi --match-path "test/multiproof/StuckBond_PROOF_THRESHOLD_2.t.sol" -vvv

Compiling 1 files with Solc 0.8.15
Solc 0.8.15 finished in 3.36s
Compiler run successful!

Ran 10 tests for test/multiproof/StuckBond_PROOF_THRESHOLD_2.t.sol:StuckBondProofThreshold2Test
[PASS] test_StuckBond_BlacklistSelfDoesNotUnstick()                  (gas:   568 634)
[PASS] test_StuckBond_CannotAddSecondProofAfterGameOver()            (gas:   510 928)
[PASS] test_StuckBond_CascadeToChildren()                            (gas: 1 045 646)
[PASS] test_StuckBond_ChangeRespectedGameTypeDoesNotUnstick()        (gas:   522 030)
[PASS] test_StuckBond_ExpectedResolutionNotMax()                     (gas:   508 071)
[PASS] test_StuckBond_MultipleGamesStuckDuringOutage()               (gas: 4 224 409)
[PASS] test_StuckBond_ResolveAndClaimCreditBothRevert()              (gas:   535 562)
[PASS] test_StuckBond_RetirementTimestampDoesNotUnstick()            (gas:   521 875)
[PASS] test_StuckBond_SetImplementationDoesNotUnstickExistingGame()  (gas: 3 725 402)
[PASS] test_StuckBond_StillStuckAfter14Days()                        (gas:   535 569)

Suite result: ok. 10 passed; 0 failed; 0 skipped; finished in 7.91ms (31.72ms CPU time)
```

### What each test demonstrates

| Test                                          | Demonstrates                                                                                                                                                                         |
| --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `ResolveAndClaimCreditBothRevert`             | Core: at `gameOver()` with `proofCount=1`, both `resolve()` and `claimCredit()` revert. Bond inaccessible.                                                                           |
| `StillStuckAfter14Days`                       | 14-day timeout fallback in `claimCredit` is unreachable when `expectedResolution != max`.                                                                                            |
| `CannotAddSecondProofAfterGameOver`           | `verifyProposalProof` blocks late second proof via `if (gameOver()) revert GameOver()`. Missing proof cannot be supplied.                                                            |
| `BlacklistSelfDoesNotUnstick`                 | Admin blacklist of the stuck game is a no-op — `resolve` consults `isGameBlacklisted(parentGame)` at `AggregateVerifier.sol:941`, not the game itself.                               |
| `ChangeRespectedGameTypeDoesNotUnstick`       | `_getParentGameStatus` hardcodes `DEFENDER_WINS` for `parent == ASR` regardless of respected type.                                                                                   |
| `RetirementTimestampDoesNotUnstick`           | Retirement check is parent-only; ASR parent is not a game.                                                                                                                           |
| `SetImplementationDoesNotUnstickExistingGame` | Factory impl upgrade applies to new clones only. Existing EIP-1167 proxies retain old bytecode and `PROOF_THRESHOLD`.                                                                |
| `ExpectedResolutionNotMax`                    | Confirms `expectedResolution` is finite (7d) — closes the only recovery branch in `claimCredit`.                                                                                     |
| `CascadeToChildren`                           | Stuck first game propagates to descendants. Child `resolve` reverts with `ParentGameNotResolved`. Admin must blacklist parent to unblock children. Parent's own bond stays frozen.   |
| `MultipleGamesStuckDuringOutage`              | Quantified funds-at-risk: **10 first games × 1 ETH = 10 ETH permanently locked** on one 7-day partial prover outage. Asserted on-chain by `totalStuckBonds == numGames * INIT_BOND`. |

### Funds at risk

* Per stuck game: `bondAmount` frozen permanently.
* Outage-driven scaling: `N_days × K_games_per_day × bondAmount` frozen during a partial prover outage. Each stuck first game also requires off-chain admin `blacklistDisputeGame` to release its descendant subtree.
* Asserted in the PoC: **10 ETH** locked per one 7-day outage with 10 first-games/day at 1 ETH bond. At mainnet production bond sizes (OP Stack precedent: 0.08–10 ETH per game) the impact grows proportionally.
* `proofThreshold == 2` is one of two branches explicitly admitted by the constructor (`InvalidProofThreshold` guard permits only `1` or `2`). Whatever configuration the contract admits must be safe under the audit scope.

### Independent validation on live fork (secondary, optional)

The same revert sequence reproduces on a Hoodi fork against the deployed DGF / ASR / DelayedWETH (`FindingB_LiveExploit.t.sol`):

```bash
forge test --ffi \
  --match-path "test/multiproof/FindingB_LiveExploit.t.sol" \
  --fork-url https://ethereum-hoodi-rpc.publicnode.com -vv
```

Deployed addresses exercised: DGF `0x154972aB98A5e321a9c7aB7677973f1F501a8090`, ASR `0x60A6C389F0BC5cE4269A40d9695927bC58700328`, DelayedWETH from the live implementation. Result: `NotEnoughProofs` on `resolve()`, `GameNotResolved` on `claimCredit()`, bond `50000000000000000 wei` stuck on-chain after `vm.warp` of 100+ days.

### Environment

```
repo:      github.com/base/contracts
tag:       v8.1.0
commit:    01dad230390cd69bcf130b5fc7a7a580b31650a7
foundry:   forge 1.5.1-stable (b0a9dd9ce)
compiler:  solc 0.8.15 (matches AggregateVerifier pragma)
test file: test/multiproof/StuckBond_PROOF_THRESHOLD_2.t.sol
suite:     StuckBondProofThreshold2Test
result:    10 passed / 0 failed / 0 skipped
```
