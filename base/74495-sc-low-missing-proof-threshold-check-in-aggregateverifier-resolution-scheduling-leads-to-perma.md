# 74495 sc low missing proof threshold check in aggregateverifier resolution scheduling leads to permanent freezing of dispute game bonds

**Submitted on Apr 22nd 2026 at 23:46:52 UTC by @Web3Angel for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74495
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Permanent freezing of funds in the bridge or in dispute game bonds with no available recovery path

## Description

### Brief/Intro

`AggregateVerifier.sol` can put a game into a permanently unresolvable state when `PROOF_THRESHOLD` is configured to `2` and only one proof is present. The game receives a finite `expectedResolution` timestamp even though it does not have enough proofs to resolve. Once that timestamp passes, `AggregateVerifier::resolve()` reverts because the proof threshold is unmet, while `AggregateVerifier::claimCredit()` also reverts because the game has not resolved and the 14-day fallback is disabled. In production, this can permanently freeze dispute game bonds with no available on-chain recovery path.

### Vulnerability Details

`AggregateVerifier` supports both `PROOF_THRESHOLD = 1` and `PROOF_THRESHOLD = 2`:

```solidity
if (proofThreshold != 1 && proofThreshold != 2) revert InvalidProofThreshold();
```

However, its resolution clock logic is based on the absolute number of proofs, not on whether the configured threshold has been met.

During `AggregateVerifier::initializeWithInitData()`, the game starts by setting `expectedResolution` to the sentinel value:

```solidity
expectedResolution = Timestamp.wrap(type(uint64).max);
```

It then verifies the initializer proof and calls `AggregateVerifier::_proofVerifiedUpdate()`:

```solidity
_proofVerifiedUpdate(proofType, gameCreator());
```

`AggregateVerifier::_proofVerifiedUpdate()` increments `proofCount` and calls `AggregateVerifier::_decreaseExpectedResolution()`:

```solidity
function _proofVerifiedUpdate(ProofType proofType, address proposer) internal {
    proofTypeToProver[proofType] = proposer;
    proofCount += 1;

    _decreaseExpectedResolution();

    emit Proved(proposer, proofType);
}
```

The issue is in `AggregateVerifier::_getDelay()`. When `proofCount == 1`, it returns `SLOW_FINALIZATION_DELAY` even if `PROOF_THRESHOLD == 2`:

```solidity
function _getDelay() internal view returns (uint64) {
    if (proofCount >= 2) {
        return FAST_FINALIZATION_DELAY;
    } else if (proofCount == 1) {
        return SLOW_FINALIZATION_DELAY;
    } else {
        return type(uint64).max;
    }
}
```

As a result, a one-proof game under a two-proof threshold receives a finite `expectedResolution`:

```solidity
uint64 newResolution = uint64(block.timestamp) + delay;
expectedResolution = Timestamp.wrap(uint64(FixedPointMathLib.min(newResolution, expectedResolution.raw())));
```

After the slow finalization delay passes, the game appears to be over because `AggregateVerifier::gameOver()` returns true:

```solidity
function gameOver() public view returns (bool) {
    return expectedResolution.raw() <= block.timestamp;
}
```

But it still cannot resolve as `AggregateVerifier::resolve()` requires the configured proof threshold to be met:

```solidity
if (!gameOver()) revert GameNotOver();
if (proofCount < PROOF_THRESHOLD) revert NotEnoughProofs();
```

For `PROOF_THRESHOLD = 2`, a game with only one proof always reverts with `NotEnoughProofs`.

The bond recovery fallback in `AggregateVerifier::claimCredit()` is also unavailable. It only applies when `expectedResolution == type(uint64).max`:

```solidity
if (expectedResolution.raw() != type(uint64).max) {
    if (resolvedAt.raw() == 0) revert GameNotResolved();
} else {
    if (block.timestamp < createdAt.raw() + 14 days) revert GameNotOver();
}
```

Given that the one-proof game has a finite `expectedResolution` and never successfully resolves, `resolvedAt` remains zero forever. Therefore:

1. `AggregateVerifier::resolve()` reverts with `NotEnoughProofs`.
2. `AggregateVerifier::claimCredit()` reverts with `GameNotResolved`.
3. The 14-day fallback cannot be reached.
4. The dispute game bond remains locked permanently.

The root cause is that the finalization delay is computed from `proofCount` alone rather than from whether `proofCount >= PROOF_THRESHOLD`.

### Impact Details

Selected impact:

> Permanent freezing of funds in the bridge or in dispute game bonds with no available recovery path

The affected funds are the dispute game initialization bond deposited into `DelayedWETH` during `AggregateVerifier::initializeWithInitData()`:

```solidity
bondAmount = msg.value;
DELAYED_WETH.deposit{ value: msg.value }();
```

If `PROOF_THRESHOLD = 2`, a game initialized with one accepted proof can become permanently unable to return its bond if the second proof does not arrive before the finite deadline. This can happen without privileged access and without compromising any key. It is sufficient for normal protocol operation to create a game with one proof while the second proof path is delayed, unavailable, or intentionally not submitted.

Once the game reaches this state, the creator cannot recover the bond through the normal resolution path or through the 14-day fallback. The loss is bounded per affected game by the configured initialization bond, but the issue can affect every one-proof game created under a two-proof threshold. If many games are created while one proof system is unavailable or delayed, the locked bond amount can accumulate across multiple games.

This is not direct theft, and it does not by itself forge state roots or drain the bridge. The severity comes from permanent fund freezing with no on-chain recovery path.

### References

* `AggregateVerifier::initializeWithInitData()` sets the sentinel and then calls `AggregateVerifier::_proofVerifiedUpdate()`: `src/multiproof/AggregateVerifier.sol`
* `AggregateVerifier::_proofVerifiedUpdate()`, `AggregateVerifier::_decreaseExpectedResolution()`, and `AggregateVerifier::_getDelay()` set a finite deadline for one-proof games: `src/multiproof/AggregateVerifier.sol`
* `AggregateVerifier::resolve()` requires `proofCount >= PROOF_THRESHOLD`: `src/multiproof/AggregateVerifier.sol`
* `AggregateVerifier::claimCredit()` only permits unresolved fallback when `expectedResolution == type(uint64).max`: `src/multiproof/AggregateVerifier.sol`
* Standalone PoC: `test/OneProofDeadStateStandalone.t.sol`
* Protocol-level PoC: `test/protocol/AggregateVerifierFindingsProtocol.t.sol`

## Proof of Concept

The PoC demonstrates the vulnerable state transition: one accepted proof under `PROOF_THRESHOLD = 2` gives the game a finite deadline, but after that deadline `AggregateVerifier::resolve()` still reverts with `NotEnoughProofs` and `AggregateVerifier::claimCredit()` still reverts with `GameNotResolved`.

Two PoC variants are provided:

{% stepper %}
{% step %}

## Standalone scoped PoC

This is the primary reproduction for the platform-provided scope, which is limited to `src/multiproof`. The in-scope package "<https://github.com/base/c...ree/v8.1.0/src/multiproof>" does not include the broader Base repository tests `BaseTest`, or the protocol deployment helpers needed to instantiate the full dispute-game stack. For that reason, this PoC deploys a minimal harness that copies only the affected `AggregateVerifier` state transitions needed for this finding: initialization, proof counting, `expectedResolution` updates, `resolve()`, and `claimCredit()`. This keeps the reproduction self-contained within the submitted scope while still demonstrating the exact dead state.

### Standalone PoC

The standalone test defines a minimal `AggregateVerifierOneProofHarness` that copies the relevant logic from:

* `AggregateVerifier::initializeWithInitData()`
* `AggregateVerifier::_proofVerifiedUpdate()`
* `AggregateVerifier::_decreaseExpectedResolution()`
* `AggregateVerifier::_getDelay()`
* `AggregateVerifier::resolve()`
* `AggregateVerifier::claimCredit()`

The test then:

1. Deploys the harness with `PROOF_THRESHOLD = 2`.
2. Initializes it with one proof and a `1 ether` bond.
3. Confirms `proofCount == 1`, `expectedResolution` is finite, and `resolvedAt == 0`.
4. Warps past 7 days and confirms `resolve()` reverts with `NotEnoughProofs`.
5. Warps past 14 days and confirms `claimCredit()` reverts with `GameNotResolved`.

#### File: `test/OneProofDeadStateStandalone.t.sol`

Create a test folder and copy the following Solidity test into `test/OneProofDeadStateStandalone.t.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.15;

import { Test, console2 as console } from "forge-std/Test.sol";

/// @notice Self-contained PoC for the one-proof dead-state finding.
/// @dev This file intentionally avoids importing BaseTest or any non-multiproof contracts.
///      It models only the AggregateVerifier state transitions needed for the finding:
///      initializeWithInitData -> _proofVerifiedUpdate -> _decreaseExpectedResolution ->
///      resolve/claimCredit. The relevant logic mirrors src/multiproof/AggregateVerifier.sol.
contract AggregateVerifierOneProofHarness {
    enum ProofType {
        TEE,
        ZK
    }

    enum GameStatus {
        IN_PROGRESS,
        CHALLENGER_WINS,
        DEFENDER_WINS
    }

    uint64 public constant SLOW_FINALIZATION_DELAY = 7 days;
    uint64 public constant FAST_FINALIZATION_DELAY = 1 days;

    error GameNotOver();
    error GameNotResolved();
    error NoCreditToClaim();
    error NotEnoughProofs();

    uint256 public immutable PROOF_THRESHOLD;

    uint64 public createdAt;
    uint64 public resolvedAt;
    uint64 public expectedResolution;
    uint8 public proofCount;
    bool public bondClaimed;
    uint256 public bondAmount;
    GameStatus public status;
    mapping(ProofType => address) public proofTypeToProver;

    constructor(uint256 proofThreshold) {
        PROOF_THRESHOLD = proofThreshold;
    }

    /// @notice Mirrors the relevant effects of AggregateVerifier::initializeWithInitData().
    function initializeWithInitData(ProofType proofType, address proposer) external payable {
        createdAt = uint64(block.timestamp);
        expectedResolution = type(uint64).max;

        _proofVerifiedUpdate(proofType, proposer);

        bondAmount = msg.value;
    }

    /// @notice Mirrors the relevant checks/effects of AggregateVerifier::resolve().
    function resolve() external returns (GameStatus) {
        if (!gameOver()) revert GameNotOver();
        if (proofCount < PROOF_THRESHOLD) revert NotEnoughProofs();

        status = GameStatus.DEFENDER_WINS;
        resolvedAt = uint64(block.timestamp);
        return status;
    }

    /// @notice Mirrors the liveness gate in AggregateVerifier::claimCredit().
    function claimCredit() external {
        if (bondClaimed) revert NoCreditToClaim();

        if (expectedResolution != type(uint64).max) {
            if (resolvedAt == 0) revert GameNotResolved();
        } else {
            if (block.timestamp < createdAt + 14 days) revert GameNotOver();
        }

        bondClaimed = true;
    }

    function gameOver() public view returns (bool) {
        return expectedResolution <= block.timestamp;
    }

    /// @notice Mirrors AggregateVerifier::_proofVerifiedUpdate().
    function _proofVerifiedUpdate(ProofType proofType, address proposer) internal {
        proofTypeToProver[proofType] = proposer;
        proofCount += 1;
        _decreaseExpectedResolution();
    }

    /// @notice Mirrors AggregateVerifier::_decreaseExpectedResolution().
    function _decreaseExpectedResolution() internal {
        uint64 delay = _getDelay();

        if (delay == type(uint64).max) {
            expectedResolution = type(uint64).max;
            return;
        }

        uint64 newResolution = uint64(block.timestamp) + delay;
        expectedResolution = newResolution < expectedResolution ? newResolution : expectedResolution;
    }

    /// @notice Mirrors the vulnerable AggregateVerifier::_getDelay() logic.
    function _getDelay() internal view returns (uint64) {
        if (proofCount >= 2) {
            return FAST_FINALIZATION_DELAY;
        } else if (proofCount == 1) {
            return SLOW_FINALIZATION_DELAY;
        } else {
            return type(uint64).max;
        }
    }
}

contract OneProofDeadStateStandaloneTest is Test {
    address internal proposer = address(0xA11CE);

    function testOneProofGameWithThresholdTwoCannotResolveOrRecoverBond() public {
        AggregateVerifierOneProofHarness game = new AggregateVerifierOneProofHarness(2);

        vm.deal(proposer, 1 ether);
        vm.prank(proposer);
        game.initializeWithInitData{ value: 1 ether }(AggregateVerifierOneProofHarness.ProofType.ZK, proposer);

        console.log("proofThreshold", game.PROOF_THRESHOLD());
        console.log("proofCount", game.proofCount());
        console.log("expectedResolution", game.expectedResolution());
        console.log("resolvedAt", game.resolvedAt());
        console.log("bondAmount", game.bondAmount());

        assertEq(game.PROOF_THRESHOLD(), 2);
        assertEq(game.proofCount(), 1);
        assertEq(game.expectedResolution(), block.timestamp + 7 days);
        assertEq(game.resolvedAt(), 0);

        vm.warp(block.timestamp + 7 days);
        console.log("gameOver after 7 days", game.gameOver());
        vm.expectRevert(AggregateVerifierOneProofHarness.NotEnoughProofs.selector);
        game.resolve();

        vm.warp(block.timestamp + 7 days);
        console.log("after 14 days total, expectedResolution is still finite");
        console.log("claimCredit reverts GameNotResolved because resolvedAt is zero");
        vm.expectRevert(AggregateVerifierOneProofHarness.GameNotResolved.selector);
        game.claimCredit();
    }
}
```

#### Run

Run it with a minimal Foundry config that compiles only this PoC test:

```bash
mkdir -p test
mkdir -p poc-foundry
cat > poc-foundry/foundry.toml <<'EOF'
[profile.default]
src = "../test"
test = "../test"
script = "../test"
out = "../forge-artifacts-poc"
libs = ["../lib"]
remappings = ["forge-std/=../lib/forge-std/src"]
EOF

forge test --config-path poc-foundry/foundry.toml --match-test testOneProofGameWithThresholdTwoCannotResolveOrRecoverBond -vv
```

The PoC only needs `forge-std`; it does not import the broader protocol test suite. The temporary `poc-foundry/foundry.toml` prevents Foundry from compiling unrelated scoped-repo files with missing dependencies.

If the reviewer is running inside a clean Foundry checkout whose default config does not compile unrelated missing dependencies, they can alternatively copy the Solidity code above into `test/OneProofDeadStateStandalone.t.sol` and run:

```bash
forge test --match-test testOneProofGameWithThresholdTwoCannotResolveOrRecoverBond -vv
```

Expected console output:

```javascript
proofThreshold 2
proofCount 1
expectedResolution <finite timestamp>
resolvedAt 0
bondAmount 1000000000000000000
gameOver after 7 days true
after 14 days total, expectedResolution is still finite
claimCredit reverts GameNotResolved because resolvedAt is zero
```

{% endstep %}

{% step %}

## Protocol-level PoC

The protocol-level PoC below is provided only as optional integration evidence. It uses the broader Base repository test harness, including `test/BaseTest.t.sol`, to deploy the real `AggregateVerifier`. That broader repository is different from the competition scope: the in-scope package is only `src/multiproof`, which does not include `BaseTest` or the rest of the protocol test suite. Therefore, reviewers using only the scoped repository should run the standalone PoC above. Reviewers with access to the broader repository can additionally run this protocol-level test.

This test repeats the standalone sequence against a real `AggregateVerifier` deployment: create a one-proof game with `PROOF_THRESHOLD = 2`, confirm its finite `expectedResolution`, then show that `resolve()` and `claimCredit()` both remain unavailable.

The protocol-level test:

1. Deploys an `AggregateVerifier` implementation with `PROOF_THRESHOLD = 2`.
2. Creates a game through `DisputeGameFactory::createWithInitData()` using one ZK proof.
3. Confirms `proofCount == 1` and `expectedResolution == block.timestamp + 7 days`.
4. Warps past 7 days and confirms `AggregateVerifier::resolve()` reverts with `NotEnoughProofs`.
5. Warps past 14 days total and confirms `AggregateVerifier::claimCredit()` reverts with `GameNotResolved`.

#### File: `test/protocol/AggregateVerifierFindingsProtocol.t.sol`

Create `test/protocol/AggregateVerifierFindingsProtocol.t.sol` and copy the code below:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.15;

import { GameNotResolved } from "src/dispute/lib/Errors.sol";
import { console2 as console } from "forge-std/console2.sol";
import { IAnchorStateRegistry } from "interfaces/dispute/IAnchorStateRegistry.sol";
import { IDelayedWETH } from "interfaces/dispute/IDelayedWETH.sol";
import { IDisputeGame } from "interfaces/dispute/IDisputeGame.sol";
import { IDisputeGameFactory } from "interfaces/dispute/IDisputeGameFactory.sol";
import { Claim } from "src/dispute/lib/Types.sol";

import { AggregateVerifier } from "src/multiproof/AggregateVerifier.sol";

import { BaseTest } from "../BaseTest.t.sol";

contract AggregateVerifierFindingsProtocolTest is BaseTest {
    function testFinding_OneProofInitIsUnresolvableWhenThresholdIsTwo() public {
        _deployAggregateVerifierWithProofThreshold(2);

        currentL2BlockNumber += BLOCK_INTERVAL;
        Claim rootClaim = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber, "zk")));
        bytes memory zkProof = _generateProof("zk-proof", AggregateVerifier.ProofType.ZK);

        AggregateVerifier game = _createAggregateVerifierGame(
            ZK_PROVER, rootClaim, currentL2BlockNumber, address(anchorStateRegistry), zkProof
        );

        console.log("one-proof init: proofCount", game.proofCount());
        console.log("one-proof init: expectedResolution", game.expectedResolution().raw());
        console.log("one-proof init: resolvedAt", game.resolvedAt().raw());
        console.log("one-proof init: threshold", uint256(2));

        assertEq(game.proofCount(), 1);
        assertEq(game.expectedResolution().raw(), block.timestamp + 7 days);

        vm.warp(block.timestamp + 7 days);
        console.log("after deadline: resolve reverts NotEnoughProofs because proofCount < threshold");
        vm.expectRevert(AggregateVerifier.NotEnoughProofs.selector);
        game.resolve();

        vm.warp(block.timestamp + 7 days);
        console.log("after 14 days: claimCredit reverts because expectedResolution is finite and resolvedAt is zero");
        vm.expectRevert(GameNotResolved.selector);
        game.claimCredit();
    }

    function _deployAggregateVerifierWithProofThreshold(uint256 proofThreshold) internal {
        AggregateVerifier aggregateVerifierImpl = new AggregateVerifier(
            AGGREGATE_VERIFIER_GAME_TYPE,
            IAnchorStateRegistry(address(anchorStateRegistry)),
            IDelayedWETH(payable(address(delayedWETH))),
            teeVerifier,
            zkVerifier,
            TEE_IMAGE_HASH,
            AggregateVerifier.ZkHashes(ZK_RANGE_HASH, ZK_AGGREGATE_HASH),
            CONFIG_HASH,
            L2_CHAIN_ID,
            BLOCK_INTERVAL,
            INTERMEDIATE_BLOCK_INTERVAL,
            proofThreshold
        );

        IDisputeGameFactory(address(factory)).setImplementation(
            AGGREGATE_VERIFIER_GAME_TYPE, IDisputeGame(address(aggregateVerifierImpl))
        );
    }
}
```

Run:

```bash
forge test --config-path protocol-foundry/foundry.toml --offline -vv \
  --match-test testFinding_OneProofInitIsUnresolvableWhenThresholdIsTwo
```

Expected console output:

```
one-proof init: proofCount 1
one-proof init: expectedResolution <finite timestamp>
one-proof init: resolvedAt 0
one-proof init: threshold 2
after deadline: resolve reverts NotEnoughProofs because proofCount < threshold
after 14 days: claimCredit reverts because expectedResolution is finite and resolvedAt is zero
```

{% endstep %}
{% endstepper %}

## Recommended Mitigation

The game should not receive a finite finalization deadline until it has enough proofs to resolve under the configured threshold.

One possible fix is to make `AggregateVerifier::_getDelay()` threshold-aware:

```solidity
function _getDelay() internal view returns (uint64) {
    if (proofCount < PROOF_THRESHOLD) {
        return type(uint64).max;
    }

    if (proofCount >= 2) {
        return FAST_FINALIZATION_DELAY;
    }

    return SLOW_FINALIZATION_DELAY;
}
```

With this change, a one-proof game under `PROOF_THRESHOLD = 2` keeps `expectedResolution = type(uint64).max` until the second proof arrives. If the second proof never arrives, the existing 14-day fallback in `AggregateVerifier::claimCredit()` remains available.

Alternatively, `AggregateVerifier::claimCredit()` could explicitly allow bond recovery for games that are past a safety timeout and have `proofCount < PROOF_THRESHOLD`, while ensuring such games cannot update the anchor state.
