# 74367 sc low permanent bond locking when proof threshold 2 residual bug after audit 1 fix&#x20;

Submitted on Apr 22nd 2026 at 03:26:54 UTC by @M4v3r1ck for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74367
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Permanent freezing of funds in the bridge or in dispute game bonds with no available recovery path

## Description

## Bug Description

When `AggregateVerifier` is deployed with `PROOF_THRESHOLD = 2` and only one proof type is submitted (or the second proof is nullified), the deposited bond becomes **permanently unclaimable**. There is no code path that allows the bond to be recovered — not through `resolve()`, not through `claimCredit()`, and not through any emergency mechanism, even after arbitrary time.

### Relationship to prior audit finding

The Multiproof Audit 1 (Cantina, March 2026) reported a related issue: "Unconditional Proof Threshold Check in resolve Blocks Bond Recovery." That finding identified that `resolve()` enforced `NotEnoughProofs` even when the parent game was invalid (CHALLENGER\_WINS). The fix (commit `b11c86da`) correctly moved the `NotEnoughProofs` check inside the `else` block, so games with invalid parents can now resolve without meeting the proof threshold.

**However, the fix introduced a new dead state that was not present before.** The current code has a gap between `resolve()` and `claimCredit()` when the parent is valid but `proofCount < PROOF_THRESHOLD`:

* `resolve()` (line 458): `if (proofCount < PROOF_THRESHOLD) revert NotEnoughProofs();` — correctly placed inside the else block (parent is valid), but no fallback exists
* `claimCredit()` (line 613-617): The 14-day escape hatch only activates when `proofCount == 0` (`expectedResolution == type(uint64).max`), NOT when `0 < proofCount < threshold`

The root cause is in the `claimCredit()` escape hatch logic at `AggregateVerifier.sol:613-617`:

```solidity
if (expectedResolution.raw() != type(uint64).max) {
    if (resolvedAt.raw() == 0) revert GameNotResolved();  // ← ALWAYS reverts when proofCount > 0
} else {
    if (block.timestamp < createdAt.raw() + 14 days) revert GameNotOver();  // 14-day escape
}
```

When `proofCount == 1` and `PROOF_THRESHOLD == 2`:

* `expectedResolution` = finite timestamp (now + 7 days via `SLOW_FINALIZATION_DELAY`) — NOT `type(uint64).max`
* So `claimCredit()` takes the first branch and requires `resolvedAt != 0`
* But `resolve()` reverts with `NotEnoughProofs` because `1 < 2`
* Neither function succeeds → bond locked permanently

| proofCount          | expectedResolution | resolve()                     | claimCredit()                 | Result                  |
| ------------------- | ------------------ | ----------------------------- | ----------------------------- | ----------------------- |
| 0                   | type(uint64).max   | Reverts                       | Works after 14 days           | Bond recoverable ✓      |
| 1 (threshold=1)     | finite             | Works                         | Works after resolve           | Bond recoverable ✓      |
| **1 (threshold=2)** | **finite**         | **Reverts (NotEnoughProofs)** | **Reverts (GameNotResolved)** | **BOND LOCKED FOREVER** |
| 2 (threshold=2)     | finite             | Works                         | Works after resolve           | Bond recoverable ✓      |

This is a distinct bug from the Audit 1 finding:

* **Audit 1**: `resolve()` blocked by NotEnoughProofs when parent was invalid → Fix: skip threshold check when parent is CHALLENGER\_WINS
* **This finding**: `claimCredit()` escape hatch has a gap for `0 < proofCount < threshold` when parent is valid → No fix exists

## Severity

**High** — Permanent freezing of user funds with no recovery path.

## Impact

**Scenario A — No second proof submitted:**\
A proposer creates a game with a TEE proof and deposits their bond (e.g., 1 ETH). The ZK prover experiences downtime, or no one provides the second proof before the game expires. After 7 days, `gameOver() = true` and no more proofs can be submitted. The bond is locked forever.

**Scenario B — Proof nullification causes mass bond locking:**\
Many games exist with both TEE + ZK proofs. A legitimate soundness issue triggers `ZK_VERIFIER.nullify()` globally (as designed — `Verifier.sol:44`). All in-progress games lose their ZK proof (`proofCount` drops from 2 to 1). The ZK verifier is globally disabled, so no replacement ZK proof can be provided. All affected games' bonds are permanently locked.

This is particularly dangerous because:

1. The `PROOF_THRESHOLD` constructor parameter accepts both 1 and 2 as valid values
2. The bond locking is silent — there's no revert message indicating the state is unrecoverable
3. The proposer has no way to distinguish between "wait longer" and "your bond is gone forever"
4. Scenario B can affect many games simultaneously from a single nullification event

## Recommendation

Extend the escape hatch in `claimCredit()` to cover the case where the game is expired and unresolvable:

```solidity
if (expectedResolution.raw() != type(uint64).max) {
    if (resolvedAt.raw() == 0) {
        // Allow claiming if game is expired but unresolvable (proofCount < threshold)
        if (!gameOver() || block.timestamp < createdAt.raw() + 14 days) {
            revert GameNotResolved();
        }
    }
} else {
    if (block.timestamp < createdAt.raw() + 14 days) revert GameNotOver();
}
```

## Proof of Concept

File: `test/multiproof/PoCBondLocked.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.15;

import { Test } from "forge-std/Test.sol";

import {
    BondTransferFailed,
    ClaimAlreadyResolved,
    GameNotResolved,
    NoCreditToClaim
} from "src/dispute/lib/Errors.sol";
import { AnchorStateRegistry } from "src/dispute/AnchorStateRegistry.sol";
import { DelayedWETH } from "src/dispute/DelayedWETH.sol";
import { DisputeGameFactory } from "src/dispute/DisputeGameFactory.sol";
import { IAnchorStateRegistry } from "interfaces/dispute/IAnchorStateRegistry.sol";
import { IDelayedWETH } from "interfaces/dispute/IDelayedWETH.sol";
import { IDisputeGame } from "interfaces/dispute/IDisputeGame.sol";
import { IDisputeGameFactory } from "interfaces/dispute/IDisputeGameFactory.sol";
import { ISystemConfig } from "interfaces/L1/ISystemConfig.sol";
import { Claim, GameStatus, GameType, Hash, Proposal, Timestamp } from "src/dispute/lib/Types.sol";

import { ProxyAdmin } from "src/universal/ProxyAdmin.sol";
import { TransparentUpgradeableProxy } from "@openzeppelin/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";

import { AggregateVerifier } from "src/multiproof/AggregateVerifier.sol";
import { IVerifier } from "interfaces/multiproof/IVerifier.sol";
import { MockSystemConfig } from "src/multiproof/mocks/MockSystemConfig.sol";
import { MockVerifier } from "src/multiproof/mocks/MockVerifier.sol";
import { LibClone } from "@solady/utils/LibClone.sol";

/// @title PoC: Permanent Bond Locking with PROOF_THRESHOLD = 2
/// @notice When PROOF_THRESHOLD = 2 and only one proof is submitted, the bond
///         becomes permanently unclaimable. resolve() reverts with NotEnoughProofs,
///         and claimCredit()'s 14-day escape hatch doesn't activate because
///         expectedResolution != type(uint64).max.
contract PoCBondLocked is Test {
    using LibClone for address;

    GameType public constant GAME_TYPE = GameType.wrap(621);
    uint256 public constant L2_CHAIN_ID = 8453;
    uint256 public constant BLOCK_INTERVAL = 100;
    uint256 public constant INTERMEDIATE_BLOCK_INTERVAL = 10;
    uint256 public constant INIT_BOND = 1 ether;
    uint256 public constant DELAYED_WETH_DELAY = 1 days;
    uint256 public constant FINALITY_DELAY = 0 days;

    // THE KEY: PROOF_THRESHOLD = 2
    uint256 public constant PROOF_THRESHOLD = 2;

    address public immutable PROPOSER = makeAddr("proposer");
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
        AnchorStateRegistry _asr = new AnchorStateRegistry(FINALITY_DELAY);
        DelayedWETH _dweth = new DelayedWETH(DELAYED_WETH_DELAY);
        DisputeGameFactory _factory = new DisputeGameFactory();
        proxyAdmin = new ProxyAdmin(address(this));

        TransparentUpgradeableProxy asrProxy = new TransparentUpgradeableProxy(address(_asr), address(proxyAdmin), "");
        anchorStateRegistry = AnchorStateRegistry(address(asrProxy));

        TransparentUpgradeableProxy factoryProxy = new TransparentUpgradeableProxy(address(_factory), address(proxyAdmin), "");
        factory = DisputeGameFactory(address(factoryProxy));

        TransparentUpgradeableProxy dwethProxy = new TransparentUpgradeableProxy(address(_dweth), address(proxyAdmin), "");
        delayedWETH = DelayedWETH(payable(address(dwethProxy)));

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

        // Deploy AggregateVerifier with PROOF_THRESHOLD = 2
        AggregateVerifier impl = new AggregateVerifier(
            GAME_TYPE,
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
            PROOF_THRESHOLD // = 2
        );

        factory.setImplementation(GAME_TYPE, IDisputeGame(address(impl)));
        factory.setInitBond(GAME_TYPE, INIT_BOND);
        anchorStateRegistry.setRespectedGameType(GAME_TYPE);
        vm.warp(block.timestamp + 1);
    }

    /// @notice Core PoC: Bond permanently locked when only 1 of 2 required proofs submitted
    function testPoCBondPermanentlyLocked() public {
        // ============================================================
        // Step 1: Proposer creates game with TEE proof, deposits 1 ETH bond
        // ============================================================
        currentL2BlockNumber += BLOCK_INTERVAL;
        Claim rootClaim = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber)));
        bytes memory teeProof = _generateProof("tee", AggregateVerifier.ProofType.TEE);

        AggregateVerifier game = _createGame(PROPOSER, rootClaim, currentL2BlockNumber, teeProof);

        assertEq(game.proofCount(), 1);
        assertEq(game.PROOF_THRESHOLD(), 2);
        assertEq(game.bondAmount(), INIT_BOND);

        // expectedResolution is finite (now + 7 days), NOT type(uint64).max
        assertTrue(game.expectedResolution().raw() != type(uint64).max);

        // ============================================================
        // Step 2: Time passes — no second proof is submitted
        //         After 7 days, gameOver() = true, no more proofs accepted
        // ============================================================
        vm.warp(block.timestamp + 7 days + 1);
        assertTrue(game.gameOver());

        // Can't add second proof — game is over
        bytes memory zkProof = _generateProof("zk", AggregateVerifier.ProofType.ZK);
        vm.expectRevert(AggregateVerifier.GameOver.selector);
        game.verifyProposalProof(zkProof);

        // ============================================================
        // Step 3: resolve() ALWAYS reverts — proofCount < PROOF_THRESHOLD
        // ============================================================
        vm.expectRevert(AggregateVerifier.NotEnoughProofs.selector);
        game.resolve();

        // ============================================================
        // Step 4: claimCredit() ALWAYS reverts — resolvedAt == 0 but
        //         expectedResolution != type(uint64).max, so the 14-day
        //         escape hatch does NOT activate
        // ============================================================
        vm.expectRevert(GameNotResolved.selector);
        game.claimCredit();

        // ============================================================
        // Step 5: Even after 14 days, 30 days, 365 days — still locked
        // ============================================================
        vm.warp(block.timestamp + 14 days);
        vm.expectRevert(GameNotResolved.selector);
        game.claimCredit();

        vm.warp(block.timestamp + 365 days);
        vm.expectRevert(GameNotResolved.selector);
        game.claimCredit();

        // resolve() still fails forever
        vm.expectRevert(AggregateVerifier.NotEnoughProofs.selector);
        game.resolve();

        // Bond is permanently locked — unrecoverable
        assertFalse(game.bondClaimed());
        assertFalse(game.bondUnlocked());
        assertEq(game.resolvedAt().raw(), 0);
    }

    /// @notice Variant: Bond locked after proof nullification (more realistic scenario)
    function testPoCBondLockedAfterNullification() public {
        // ============================================================
        // Step 1: Game with both proofs (proofCount = 2, threshold = 2)
        // ============================================================
        currentL2BlockNumber += BLOCK_INTERVAL;
        Claim rootClaim = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber)));
        bytes memory teeProof = _generateProof("tee", AggregateVerifier.ProofType.TEE);

        AggregateVerifier game = _createGame(PROPOSER, rootClaim, currentL2BlockNumber, teeProof);

        // Add ZK proof
        bytes memory zkProof = _generateProof("zk", AggregateVerifier.ProofType.ZK);
        vm.prank(makeAddr("zk-prover"));
        game.verifyProposalProof(zkProof);

        assertEq(game.proofCount(), 2);
        // expectedResolution decreased to now + 1 day (FAST_FINALIZATION_DELAY)
        assertTrue(game.expectedResolution().raw() != type(uint64).max);

        // ============================================================
        // Step 2: ZK proof is nullified (legitimate soundness issue found)
        // ============================================================
        Claim differentRoot = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "different")));
        bytes memory nullifyProof = _generateProof("nullify-zk", AggregateVerifier.ProofType.ZK);
        bytes memory fullNullifyProof = abi.encodePacked(uint8(AggregateVerifier.ProofType.ZK), nullifyProof);

        // Nullify ZK proof: proves a different intermediate root
        vm.prank(makeAddr("nullifier"));
        // Need to construct proper nullify call
        game.nullify(
            fullNullifyProof,
            BLOCK_INTERVAL / INTERMEDIATE_BLOCK_INTERVAL - 1,
            differentRoot.raw()
        );

        // proofCount back to 1, but ZK_VERIFIER is now globally nullified
        assertEq(game.proofCount(), 1);
        assertTrue(zkVerifier.nullified());

        // expectedResolution is finite (now + 7 days), NOT max
        assertTrue(game.expectedResolution().raw() != type(uint64).max);

        // ============================================================
        // Step 3: Can't get ZK proof again — verifier is globally nullified
        //         Game expires with proofCount = 1 < threshold = 2
        // ============================================================
        vm.warp(block.timestamp + 7 days + 1);
        assertTrue(game.gameOver());

        // resolve() reverts: not enough proofs
        vm.expectRevert(AggregateVerifier.NotEnoughProofs.selector);
        game.resolve();

        // claimCredit() reverts: game not resolved, escape hatch inactive
        vm.expectRevert(GameNotResolved.selector);
        game.claimCredit();

        // Bond permanently locked
        vm.warp(block.timestamp + 365 days);
        vm.expectRevert(GameNotResolved.selector);
        game.claimCredit();
    }

    // ===================== Helpers =====================

    function _createGame(
        address creator,
        Claim rootClaim,
        uint256 l2BlockNumber,
        bytes memory proof
    ) internal returns (AggregateVerifier game) {
        bytes memory intermediateRoots = abi.encodePacked(
            _generateIntermediateRootsExceptLast(l2BlockNumber),
            rootClaim.raw()
        );
        bytes memory extraData = abi.encodePacked(uint256(l2BlockNumber), address(anchorStateRegistry), intermediateRoots);

        vm.deal(creator, INIT_BOND);
        vm.prank(creator);
        return AggregateVerifier(
            address(factory.createWithInitData{ value: INIT_BOND }(GAME_TYPE, rootClaim, extraData, proof))
        );
    }

    function _generateProof(bytes memory salt, AggregateVerifier.ProofType proofType) internal view returns (bytes memory) {
        bytes32 l1OriginHash = blockhash(block.number - 1);
        uint256 l1OriginNumber = block.number - 1;
        bytes memory signature = abi.encodePacked(salt, bytes32(0), bytes32(0), uint8(27));
        return abi.encodePacked(uint8(proofType), l1OriginHash, l1OriginNumber, signature);
    }

    function _generateIntermediateRootsExceptLast(uint256 l2BlockNumber) internal pure returns (bytes memory) {
        bytes memory roots;
        uint256 start = l2BlockNumber - BLOCK_INTERVAL;
        for (uint256 i = 1; i < BLOCK_INTERVAL / INTERMEDIATE_BLOCK_INTERVAL; i++) {
            roots = abi.encodePacked(roots, keccak256(abi.encode(start + INTERMEDIATE_BLOCK_INTERVAL * i)));
        }
        return roots;
    }
}
```

The test deploys `AggregateVerifier` with `PROOF_THRESHOLD = 2` (custom setUp, not using BaseTest defaults). Two tests demonstrate the bug:

1. **`testPoCBondPermanentlyLocked`**: Creates a game with only 1 TEE proof. After 7 days, `gameOver() = true` but `resolve()` reverts with `NotEnoughProofs`. `claimCredit()` reverts with `GameNotResolved`. Even after 365 days, both functions still revert. Bond permanently locked.
2. **`testPoCBondLockedAfterNullification`**: Creates a game with both proofs (proofCount=2). ZK proof is nullified (legitimate soundness issue). `proofCount` drops to 1, ZK verifier is globally dead. Game expires, resolve fails, claimCredit fails. Bond permanently locked.

Run: `forge test --match-path test/multiproof/PoCBondLocked.t.sol -vv`

Both tests PASS, confirming the bond is unrecoverable in both scenarios.
