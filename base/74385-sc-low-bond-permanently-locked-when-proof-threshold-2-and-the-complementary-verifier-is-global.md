# 74385 sc low bond permanently locked when proof threshold 2 and the complementary verifier is globally nullified mid game

Submitted on Apr 22nd 2026 at 06:40:24 UTC by @humbleservant for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74385
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Permanent freezing of funds in the bridge or in dispute game bonds with no available recovery path

## Description

## Brief/Intro

`AggregateVerifier`'s constructor admits `PROOF_THRESHOLD` of either 1 or 2. On a clone deployed with threshold 2, once exactly one of the two proofs (TEE or ZK) has been accepted, if the complementary verifier is globally nullified via `Verifier.nullify()` before the second proof lands, the clone cannot resolve and cannot refund. The bond posted at init sits in `DELAYED_WETH` owned by an unreachable game address with no path back to the proposer. When threshold 2 ships to production, every in-flight game that is in the "one proof accepted, one still pending" state at the moment of a soundness-triggered `nullify()` has its bond permanently frozen with no available recovery path.

## Vulnerability Details

`Verifier.nullified` is a single boolean on the verifier contract. It is not per-game. Any proper, respected sibling dispute game in the anchor registry is authorized to call `nullify()`. The designed purpose is to respond to an SP1 soundness bug or an enclave image recall. The side effect is that every `AggregateVerifier` clone that shares that verifier has its subsequent `verify()` blocked by the `notNullified` modifier.

```solidity
// Verifier.sol

bool public nullified;                                        // line 14

modifier notNullified() { if (nullified) revert Nullified(); _; }   // lines 27-30

function nullify() external override {                        // lines 39-47
    if (
        !ANCHOR_STATE_REGISTRY.isGameProper(IDisputeGame(msg.sender))
            || !ANCHOR_STATE_REGISTRY.isGameRespected(IDisputeGame(msg.sender))
    ) revert NotProperGame();
    nullified = true;
    emit VerifierNullified(IDisputeGame(msg.sender));
}
```

On a threshold 2 clone, the constructor gate accepts the value:

```solidity
// AggregateVerifier.sol line 285
if (proofThreshold != 1 && proofThreshold != 2) revert InvalidProofThreshold();
```

When the proposer has already landed one proof and the other verifier is now nullified, three conditions combine to lock the bond:

First, `resolve()` reverts forever:

```solidity
// AggregateVerifier.sol line 458
if (proofCount < PROOF_THRESHOLD) revert NotEnoughProofs();
```

`proofCount` cannot reach 2 because the second `verifyProposalProof` reverts inside the nullified verifier. So `resolve()` is permanently wedged.

Second, `claimCredit()`'s 14-day creator refund fallback is gated on a state that the game cannot return to:

```solidity
// AggregateVerifier.sol lines 606-617
function claimCredit() external nonReentrant {
    if (bondClaimed) revert NoCreditToClaim();
    if (expectedResolution.raw() != type(uint64).max) {
        if (resolvedAt.raw() == 0) revert GameNotResolved();                    // line 614
    } else {
        if (block.timestamp < createdAt.raw() + 14 days) revert GameNotOver();  // line 616
    }
    // bond transfer
}
```

The active branch is the one at line 614 because `expectedResolution != type(uint64).max`. The only way `expectedResolution` returns to `type(uint64).max` is `_decreaseExpectedResolution()` observing `proofCount == 0`:

```solidity
// AggregateVerifier.sol lines 815-823
function _getDelay() internal view returns (uint64) {
    if (proofCount >= 2)      return FAST_FINALIZATION_DELAY;
    else if (proofCount == 1) return SLOW_FINALIZATION_DELAY;
    else                       return type(uint64).max;
}
```

After the first proof is accepted, `proofCount == 1`, `_decreaseExpectedResolution` writes `block.timestamp + SLOW_FINALIZATION_DELAY` into `expectedResolution`, and no subsequent call can push `proofCount` back down to 0. So the 14-day fallback at line 616 is structurally unreachable for this game.

Third, no external function on the clone rescues the bond. Every external path was enumerated:

```
initializeWithInitData  : one-shot, guarded by the initialized bool
verifyProposalProof     : reverts Nullified() inside the other verifier
challenge               : restricted to ProofType.ZK, also calls ZK_VERIFIER.verify
nullify                 : global side effect, wedges sibling games too
resolve                 : wedged at line 458
closeGame               : requires resolvedAt != 0 at line 645
claimCredit             : wedged as described above
```

There is no owner, no guardian, no pause, and no `rescueBond`. `AnchorStateRegistry.isGameBlacklisted` and `isGameRetired` only govern whether child games reference this clone as a parent; they do not release the bond.

Self-nullify via `AggregateVerifier.nullify()` is not a recovery either. It globally nullifies the same-type verifier across every Azul game and requires a same-type counter-proof that a correct enclave will refuse to produce.

## Impact Details

Every deploy config shipped in v8.1.0 sets `multiproofProofThreshold = 1`:

```
deploy-config/sepolia.json:80:  "multiproofProofThreshold": 1,
deploy-config/hardhat.json:88:  "multiproofProofThreshold": 1
```

Other chain configs (`mainnet.json`, `internal-devnet.json`, `sepolia-devnet-0.json`) do not define a multiproof block. The bug cannot manifest on any current live deployment. It is latent until Base ships an implementation with `proofThreshold = 2`, which is the only reason the constructor admits that value.

Once threshold 2 is deployed, each wedged clone locks its full posted bond inside `DELAYED_WETH` with no reachable recovery. Total value at risk equals (number of concurrent threshold 2 games in the one-proof-accepted state at the nullify moment) multiplied by (bond per game). After a soundness disclosure, the expected population is every in-flight game that had already collected one proof but not both. With Base's observed game cadence that is on the order of dozens of games at any one time; with Optimism fault-dispute bonds historically at roughly 0.08 ETH to low single-digit ETH per game, the frozen value per soundness event lands in the low-single-digits to low-tens of ETH per chain.

Value is locked, not stolen. No attacker is required, no attacker profit. The finding fits impact category #6 Permanent freezing of funds in the bridge or in dispute game bonds with no available recovery path, because (a) the bond is a dispute game bond, (b) the freeze is permanent with no timeout or recovery function, and (c) every external function of the clone was shown unable to release the bond.

## References

* `src/multiproof/AggregateVerifier.sol` at <https://github.com/base/contracts/blob/v8.1.0/src/multiproof/AggregateVerifier.sol>
* `src/multiproof/Verifier.sol` at <https://github.com/base/contracts/blob/v8.1.0/src/multiproof/Verifier.sol>
* `src/multiproof/zk/ZKVerifier.sol` at <https://github.com/base/contracts/blob/v8.1.0/src/multiproof/zk/ZKVerifier.sol> (inherits `notNullified` on `verify()`)
* `src/multiproof/tee/TEEVerifier.sol` at <https://github.com/base/contracts/blob/v8.1.0/src/multiproof/tee/TEEVerifier.sol> (inherits `notNullified` on `verify()`)
* `deploy-config/sepolia.json` and `deploy-config/hardhat.json` at <https://github.com/base/contracts/tree/v8.1.0/deploy-config>
* Scope page: <https://immunefi.com/audit-competition/audit-comp-base-azul/scope/>

## Proof of Concept

Drop-in Foundry test that reuses the repo's own harness pattern (same topology as `test/multiproof/BaseTest.t.sol`, only PROOF\_THRESHOLD is changed to 2). Save as `test/multiproof/StuckBondThreshold2.t.sol` and run with `forge test --match-path "test/multiproof/StuckBondThreshold2.t.sol" -vv`. It compiles with `solc 0.8.15` and passes.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.15;
 
import { Test } from "forge-std/Test.sol";
 
import { AnchorStateRegistry } from "src/dispute/AnchorStateRegistry.sol";
import { DelayedWETH } from "src/dispute/DelayedWETH.sol";
import { DisputeGameFactory } from "src/dispute/DisputeGameFactory.sol";
import { IAnchorStateRegistry } from "interfaces/dispute/IAnchorStateRegistry.sol";
import { IDelayedWETH } from "interfaces/dispute/IDelayedWETH.sol";
import { IDisputeGame } from "interfaces/dispute/IDisputeGame.sol";
import { IDisputeGameFactory } from "interfaces/dispute/IDisputeGameFactory.sol";
import { ISystemConfig } from "interfaces/L1/ISystemConfig.sol";
import { Claim, GameType, Hash, Proposal } from "src/dispute/lib/Types.sol";
import { GameNotResolved } from "src/dispute/lib/Errors.sol";
 
import { ProxyAdmin } from "src/universal/ProxyAdmin.sol";
import { TransparentUpgradeableProxy } from "@openzeppelin/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";
 
import { AggregateVerifier } from "src/multiproof/AggregateVerifier.sol";
import { Verifier } from "src/multiproof/Verifier.sol";
import { IVerifier } from "interfaces/multiproof/IVerifier.sol";
 
import { MockSystemConfig } from "src/multiproof/mocks/MockSystemConfig.sol";
import { MockVerifier } from "src/multiproof/mocks/MockVerifier.sol";
 
contract StuckBondThreshold2 is Test {
    GameType public constant AGGREGATE_VERIFIER_GAME_TYPE = GameType.wrap(621);
    uint256 public constant L2_CHAIN_ID = 8453;
    uint256 public constant BLOCK_INTERVAL = 100;
    uint256 public constant INTERMEDIATE_BLOCK_INTERVAL = 10;
    uint256 public constant INIT_BOND = 1 ether;
    uint256 public constant DELAYED_WETH_DELAY = 1 days;
    uint256 public constant FINALITY_DELAY = 0 days;
    uint256 public constant PROOF_THRESHOLD = 2; // the only change vs BaseTest
 
    address public immutable TEE_PROVER = makeAddr("tee-prover");
    address public immutable ZK_PROVER = makeAddr("zk-prover");
 
    bytes32 public immutable TEE_IMAGE_HASH = keccak256("tee-image");
    bytes32 public immutable ZK_RANGE_HASH = keccak256("zk-range");
    bytes32 public immutable ZK_AGGREGATE_HASH = keccak256("zk-aggregate");
    bytes32 public immutable CONFIG_HASH = keccak256("config");
 
    ProxyAdmin public proxyAdmin;
    MockSystemConfig public systemConfig;
 
    DisputeGameFactory public factory;
    AnchorStateRegistry public anchorStateRegistry;
    DelayedWETH public delayedWETH;
 
    MockVerifier public teeVerifier;
    MockVerifier public zkVerifier;
 
    uint256 public currentL2BlockNumber = 0;
 
    function setUp() public {
        systemConfig = new MockSystemConfig();
        AnchorStateRegistry _anchorStateRegistry = new AnchorStateRegistry(FINALITY_DELAY);
        DelayedWETH _delayedWETH = new DelayedWETH(DELAYED_WETH_DELAY);
        DisputeGameFactory _factory = new DisputeGameFactory();
 
        proxyAdmin = new ProxyAdmin(address(this));
 
        TransparentUpgradeableProxy anchorStateRegistryProxy =
            new TransparentUpgradeableProxy(address(_anchorStateRegistry), address(proxyAdmin), "");
        anchorStateRegistry = AnchorStateRegistry(address(anchorStateRegistryProxy));
 
        TransparentUpgradeableProxy factoryProxy =
            new TransparentUpgradeableProxy(address(_factory), address(proxyAdmin), "");
        factory = DisputeGameFactory(address(factoryProxy));
 
        TransparentUpgradeableProxy delayedWETHProxy =
            new TransparentUpgradeableProxy(address(_delayedWETH), address(proxyAdmin), "");
        delayedWETH = DelayedWETH(payable(address(delayedWETHProxy)));
 
        teeVerifier = new MockVerifier(IAnchorStateRegistry(address(anchorStateRegistry)));
        zkVerifier = new MockVerifier(IAnchorStateRegistry(address(anchorStateRegistry)));
 
        anchorStateRegistry.initialize(
            ISystemConfig(address(systemConfig)),
            IDisputeGameFactory(address(factory)),
            Proposal({ root: Hash.wrap(keccak256(abi.encode(currentL2BlockNumber))), l2SequenceNumber: currentL2BlockNumber }),
            GameType.wrap(0)
        );
        factory.initialize(address(this));
        delayedWETH.initialize(ISystemConfig(address(systemConfig)));
 
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
            PROOF_THRESHOLD
        );
 
        factory.setImplementation(AGGREGATE_VERIFIER_GAME_TYPE, IDisputeGame(address(aggregateVerifierImpl)));
        factory.setInitBond(AGGREGATE_VERIFIER_GAME_TYPE, INIT_BOND);
        anchorStateRegistry.setRespectedGameType(AGGREGATE_VERIFIER_GAME_TYPE);
 
        vm.warp(block.timestamp + 1);
    }
 
    function test_stuckBond_whenComplementaryVerifierIsGloballyNullified() public {
        // Both games are siblings that build on the anchor root at block 0.
        currentL2BlockNumber += BLOCK_INTERVAL;
 
        // Game A: lands a ZK proof. proofCount = 1, waiting on TEE.
        Claim rootA = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "gameA-zk")));
        bytes memory zkProofA = _generateProof("gameA-zk", AggregateVerifier.ProofType.ZK);
        AggregateVerifier gameA =
            _createAggregateVerifierGame(ZK_PROVER, rootA, currentL2BlockNumber, address(anchorStateRegistry), zkProofA);
        assertEq(gameA.proofCount(), 1, "gameA should have 1 proof after init");
        assertLt(gameA.expectedResolution().raw(), type(uint64).max, "gameA expectedResolution is not max");
 
        // Sibling game B: lands a TEE proof, then submits a conflicting TEE proof via nullify().
        // This globally flips Verifier(teeVerifier).nullified = true.
        Claim rootB1 = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "gameB-tee1")));
        bytes memory teeProofB1 = _generateProof("gameB-tee1", AggregateVerifier.ProofType.TEE);
        AggregateVerifier gameB =
            _createAggregateVerifierGame(TEE_PROVER, rootB1, currentL2BlockNumber, address(anchorStateRegistry), teeProofB1);
 
        Claim rootB2 = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "gameB-tee2")));
        bytes memory teeProofB2 = _generateProof("gameB-tee2", AggregateVerifier.ProofType.TEE);
        gameB.nullify(teeProofB2, BLOCK_INTERVAL / INTERMEDIATE_BLOCK_INTERVAL - 1, rootB2.raw());
        assertTrue(Verifier(address(teeVerifier)).nullified(), "TEE verifier must now be globally nullified");
 
        // Game A tries to land the TEE proof. notNullified modifier blocks it.
        bytes memory teeProofA = _generateProof("gameA-tee", AggregateVerifier.ProofType.TEE);
        vm.expectRevert(Verifier.Nullified.selector);
        gameA.verifyProposalProof(teeProofA);
 
        // After SLOW_FINALIZATION_DELAY, resolve() is still wedged: proofCount < PROOF_THRESHOLD.
        vm.warp(block.timestamp + 7 days + 1);
        vm.expectRevert(AggregateVerifier.NotEnoughProofs.selector);
        gameA.resolve();
 
        // claimCredit is wedged forever: expectedResolution != type(uint64).max, so the
        // GameNotResolved branch at line 614 is the one that fires, every time.
        vm.expectRevert(GameNotResolved.selector);
        gameA.claimCredit();
        vm.warp(block.timestamp + 365 days);
        vm.expectRevert(GameNotResolved.selector);
        gameA.claimCredit();
        vm.warp(block.timestamp + 10 * 365 days);
        vm.expectRevert(GameNotResolved.selector);
        gameA.claimCredit();
 
        // Bond is permanently locked in DelayedWETH under gameA's address.
        assertEq(delayedWETH.balanceOf(address(gameA)), INIT_BOND, "bond stuck in DelayedWETH under gameA");
    }
 
    // Helpers (same pattern as BaseTest, inlined so the file is self-contained).
    function _createAggregateVerifierGame(
        address creator,
        Claim rootClaim,
        uint256 l2BlockNumber,
        address parentAddress,
        bytes memory proof
    ) internal returns (AggregateVerifier game) {
        bytes memory intermediateRoots =
            abi.encodePacked(_generateIntermediateRootsExceptLast(l2BlockNumber), rootClaim.raw());
        bytes memory extraData = abi.encodePacked(uint256(l2BlockNumber), parentAddress, intermediateRoots);
 
        vm.deal(creator, INIT_BOND);
        vm.prank(creator);
        return AggregateVerifier(
            address(
                factory.createWithInitData{ value: INIT_BOND }(
                    AGGREGATE_VERIFIER_GAME_TYPE, rootClaim, extraData, proof
                )
            )
        );
    }
 
    function _generateProof(bytes memory salt, AggregateVerifier.ProofType proofType)
        internal view returns (bytes memory)
    {
        bytes32 l1OriginHash = blockhash(block.number - 1);
        uint256 l1OriginNumber = block.number - 1;
        bytes memory signature = abi.encodePacked(salt, bytes32(0), bytes32(0), uint8(27));
        return abi.encodePacked(uint8(proofType), l1OriginHash, l1OriginNumber, signature);
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

Run output:

```
Ran 1 test for test/multiproof/StuckBondThreshold2.t.sol:StuckBondThreshold2
[PASS] test_stuckBond_whenComplementaryVerifierIsGloballyNullified() (gas: 1049557)
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

The trace the test encodes:

1. Proposer of game A posts bond and a valid ZK proof. `proofCount = 1`, `expectedResolution = block.timestamp + 7 days`.
2. A sibling game B is a proper, respected dispute game in the anchor registry. It submits two conflicting TEE proofs via `nullify()`, which calls `TEE_VERIFIER.nullify()`. `Verifier(teeVerifier).nullified = true` globally.
3. Game A tries to land the TEE proof. Reverts `Nullified()` inside `TEEVerifier.verify`.
4. Seven days elapse. `resolve()` reverts `NotEnoughProofs()` at line 458.
5. `claimCredit()` reverts `GameNotResolved()` at line 614. The 14-day fallback at line 616 is not taken because `expectedResolution != type(uint64).max`.
6. Arbitrary time elapses (365 days, then 10 years). `claimCredit()` still reverts `GameNotResolved()`. Time does not help.
7. The bond stays with the clone in `DELAYED_WETH` permanently.

The same trace works with the proof types swapped (ZK accepted first on game A, then ZK verifier globally nullified by game B).

### Recommended fix

Treat "the only missing proof type's verifier is globally nullified" as an escape condition and route the proposer through the 14-day creator refund that already exists.

```solidity
// AggregateVerifier.sol
 
+ function _otherTypeVerifierNullified() internal view returns (bool) {
+     if (PROOF_THRESHOLD != 2 || proofCount != 1) return false;
+     ProofType accepted = proofs[0].proofType;
+     address otherVerifier = accepted == ProofType.TEE
+         ? address(ZK_VERIFIER)
+         : address(TEE_VERIFIER);
+     return IVerifier(otherVerifier).nullified();
+ }
 
  function claimCredit() external nonReentrant {
      if (bondClaimed) revert NoCreditToClaim();
      if (expectedResolution.raw() != type(uint64).max) {
-         if (resolvedAt.raw() == 0) revert GameNotResolved();
+         if (resolvedAt.raw() == 0 && !_otherTypeVerifierNullified()) {
+             revert GameNotResolved();
+         }
+         if (resolvedAt.raw() == 0 && _otherTypeVerifierNullified()) {
+             if (block.timestamp < createdAt.raw() + 14 days) revert GameNotOver();
+         }
      } else {
          if (block.timestamp < createdAt.raw() + 14 days) revert GameNotOver();
      }
      // bond transfer
  }
```
