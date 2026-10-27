# 75376 sc low permanent freezing of dispute game bonds in aggregateverifier when proof threshold equals two and one proof type permanently fails

**Submitted on Apr 28th 2026 at 20:01:05 UTC by @whatsrdoin589 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75376
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Permanent freezing of funds in the bridge or in dispute game bonds with no available recovery path
  * Temporary freezing of funds for at least 24 hours (e.g., stuck withdrawal proofs, locked dispute game bonds)

## Description

The bug I found is a perm/temp lockout depending on how you trigger it. In AggregateVerifier.sol the bond posted by a dispute game's creator can become locked when the game is configured with PROOF\_THRESHOLD = 2 and only one of the two required proof types is ever submitted. If the alternate proof type's verifier later becomes permanently unusable, both resolve and claimCredit revert and the 14-day abandoned-game fallback inside claimCredit is unreachable. Worst case is permanent loss of the bond for affected anchor games. Best case is at least a 7 day freeze for affected mid chain games while the parent dispute settles.

## Vulnerability Details

The bond claim path at AggregateVerifier.sol lines 606 to 634:

```solidity
function claimCredit() external nonReentrant {
    if (bondClaimed) revert NoCreditToClaim();

    // The game must have resolved or 14 days have passed since creation.
    if (expectedResolution.raw() != type(uint64).max) {
        if (resolvedAt.raw() == 0) revert GameNotResolved();
    } else {
        if (block.timestamp < createdAt.raw() + 14 days) revert GameNotOver();
    }
    // ... DELAYED_WETH unlock and withdraw paths
}
```

The 14 day fallback in the else branch is documented in the source comment at lines 610-611 as the escape for when the proof system has gotten far enough that the game can no longer update the anchor state registry. That branch only executes when expectedResolution == type(uint64).max, which is only true when no proof has ever been submitted to the game.

The internal helper \_proofVerifiedUpdate, called from both initializeWithInitData and verifyProposalProof, calls \_decreaseExpectedResolution, which sets expectedResolution to the minimum of the new resolution timestamp and the current value:

```solidity
expectedResolution = Timestamp.wrap(uint64(FixedPointMathLib.min(newResolution, expectedResolution.raw())));
```

After even one proof submission expectedResolution is no longer type(uint64).max, and from that point the only path through claimCredit is the if branch at line 614, which requires resolvedAt != 0. The resolve function at line 458 reverts when proofCount is below threshold:

```solidity
if (proofCount < PROOF_THRESHOLD) revert NotEnoughProofs();
```

Put it together: with PROOF\_THRESHOLD = 2, proofCount = 1, and the alternate proof type's verifier dead, both resolve and claimCredit revert. We never get to the else portion.

The most concrete way for the alternate verifier to fail is governance nullification. Each verifier has a nullify() path documented at AggregateVerifier.sol lines 594-601 that anyone in the same game type can hit when there's a soundness break. Once nullified, the verifier permanently rejects everything because Verifier.sol lines 39-47 expose no way to reverse the flag. Every verifyProposalProof call of that proof type reverts under the notNullified modifier (Verifier.sol:27-30).

Two other failure modes produce the same lockup whihc are ZK program rotation (because ZK\_RANGE\_HASH and ZK\_AGGREGATE\_HASH are immutable, a new SP1 program version can't satisfy old games' verification at lines 927 and 932), and indefinite operator outage (compromised key or network partition).

The 14 day fallback in claimCredit was clearly intended to handle exactly this situation, but the implementation only triggers it when no proof at all was submitted. The partial-progress case (one proof in, alternate stalls) gets no safety net.

One thing worth mentioning, the resolve function at lines 447-467 includes a parent-game status check before the proofCount check. When the parent settles to CHALLENGER\_WINS, this game settles to CHALLENGER\_WINS, resolvedAt is set, and claimCredit succeeds. So the permanent lockup is only realized when all of these hold at the same time: the current game has PROOF\_THRESHOLD = 2 with proofCount < 2, the parent (or AnchorStateRegistry parent) settles to DEFENDER\_WINS, and the alternate verifier is permanently dead. Anchor games are the worst case because \_getParentGameStatus returns DEFENDER\_WINS unconditionally for them, so the parent-settlement escape does not exist for anchors. Mid chain games can eventually escape if some ancestor settles CHALLENGER\_WINS, but the bond stays frozen until that happens.

## Attack flow

{% stepper %}
{% step %}

### 1. The proposer creates a dispute game with PROOF\_THRESHOLD = 2 and posts the bond at msg.value.

{% endstep %}

{% step %}

### 2. Within the SLOW\_FINALIZATION\_DELAY window, exactly one proof type lands. proofCount = 1. \_decreaseExpectedResolution sets expectedResolution = block.timestamp + 7 days.

{% endstep %}

{% step %}

### 3. Some unrelated game in the same game type triggers nullify() on the OTHER proof type's verifier. (Or the verifier dies via program rotation or operator outage. Same outcome.)

{% endstep %}

{% step %}

### 4. Time advances past the 7-day window. gameOver() returns true. The game would normally resolve here.

{% endstep %}

{% step %}

### 5. Anyone calls resolve(). It reverts on NotEnoughProofs because proofCount < PROOF\_THRESHOLD.

{% endstep %}

{% step %}

### 6. The proposer calls claimCredit() to recover their bond. The first gate sees expectedResolution != type(uint64).max (set in step 2) and takes the if branch. That branch requires resolvedAt != 0, but resolve never set it, so claimCredit reverts on GameNotResolved.

{% endstep %}

{% step %}

### 7. The 14-day else branch is unreachable because expectedResolution is no longer the sentinel value. Time can pass forever, the bond cannot be claimed.

{% endstep %}
{% endstepper %}

## Impact Details

This bug maps to two impact categories depending on what kind of game gets stuck.

Permanent freezing of dispute game bonds with no available recovery path applies to anchor games configured with PROOF\_THRESHOLD = 2 when one proof type's verifier becomes permanently unusable. The bond is deposited into DELAYED\_WETH at AggregateVerifier.sol line 411, and the only direct path to extract it is through claimCredit (which calls DELAYED\_WETH.unlock then DELAYED\_WETH.withdraw). Both calls live behind the resolved-or-14-day gate this finding describes. When both resolve and the 14 day fallback are blocked there is no inscope path to recover the bond. The bond amount equals the gameCreator's msg.value at game creation and each affected anchor game permanently loses its full bond.

Temporary freezing of funds for at least 24 hours applies to midchain games in the same configuration. The bond eventually becomes reclaimable if some ancestor in the dispute chain settles to CHALLENGER\_WINS, but the bond stays frozen for the duration of the parent settlement timeline, which is multiday under normal operations. The freezing window has at minimum the SLOW\_FINALIZATION\_DELAY of 7 days, which is well over 24 hours.

## Suggested fix

Extend the 14 day fallback in claimCredit to cover threshold-not-reached games. Update the gate so the 14 day timer also unlocks the bond when proofCount < PROOF\_THRESHOLD and 14 days have elapsed since createdAt

```solidity
if (expectedResolution.raw() != type(uint64).max) {
    if (resolvedAt.raw() == 0) {
        bool stuck = (proofCount < PROOF_THRESHOLD)
            && (block.timestamp >= createdAt.raw() + 14 days);
        if (!stuck) revert GameNotResolved();
    }
} else {
    if (block.timestamp < createdAt.raw() + 14 days) revert GameNotOver();
}
```

A normal happy-path game has resolve set resolvedAt during the same call that brings proofCount up to threshold, so the stuck condition can never trigger for a healthy game. The change is non-breaking and is my preferred fix because its simple and has very little impact.

## References

Upstream files at base/contracts v8.1.0

AggregateVerifier.sol: <https://github.com/base/contracts/tree/v8.1.0/src/multiproof/AggregateVerifier.sol\\>
Verifier.sol: <https://github.com/base/contracts/tree/v8.1.0/src/multiproof/Verifier.sol>

Specific line ranges relevant to this finding:

claimCredit: AggregateVerifier.sol 606-634\
resolve: AggregateVerifier.sol 443-474\
\_proofVerifiedUpdate: AggregateVerifier.sol 763-770\
\_decreaseExpectedResolution: AggregateVerifier.sol 773-785\
\_getDelay: AggregateVerifier.sol 815-823\
nullify and notNullified modifier: Verifier.sol 27-47\
DELAYED\_WETH bond storage call sites: AggregateVerifier.sol 411, 620, 626

## Proof of Concept

I built a minimum reproducer that isolates the exact control flow from AggregateVerifier.sol v8.1.0 (the claimCredit, resolve, \_proofVerifiedUpdate, \_decreaseExpectedResolution, and \_getDelay paths). The proof verification call is replaced with a no-op so the reproducer is dependency-free and the triager can run it without setting up cwia clones, DelayedWETH, AnchorStateRegistry, SP1 verifier, or NitroEnclave.

Ran 4 tests for test/Repro.t.sol:F001\_BondLockupTest

\[PASS] test\_BondLocked\_When\_OnlyOneProof\_And\_ThresholdTwo (gas: 978910)\
\[PASS] test\_Baseline\_ThresholdOne\_ResolvesNormally (gas: 976593)\
\[PASS] test\_Baseline\_ThresholdTwo\_BothProofs\_ResolvesNormally (gas: 1002165)\
\[PASS] test\_Baseline\_NoProofs\_FourteenDayFallbackWorks (gas: 949182)\
4 passed; 0 failed; 0 skipped

The first test is the lockup itself and the other three are baselines that show the lockup does NOT manifest at threshold=1, does NOT manifest when both proofs land and that the 14 day fallback works fine for never proven games. Together they isolate the bug surface to exactly the partial progress case.

To run: save the two files below, install forge-std at lib/forge-std, and run forge test -vv. Or skip local install and run via Docker:

```bash
docker run --rm -v "$(pwd):/workspace" -w /workspace ghcr.io/foundry-rs/foundry:latest "forge test -vv"
```

{% code title="foundry.toml" %}

```toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
test = "test"
solc_version = "0.8.15"
optimizer = true
optimizer_runs = 999999
```

{% endcode %}

{% code title="remappings.txt" %}

```
forge-std/=lib/forge-std/src/
```

{% endcode %}

{% code title="src/BondLockupReproducer.sol" %}

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.15;

/// @title BondLockupReproducer
/// @notice Minimum reproducer of the bond claim control flow from
///         AggregateVerifier.sol at github.com/base/contracts/tree/v8.1.0/src/multiproof
///
///         Mapping back to upstream:
///           claimCredit                 AggregateVerifier.sol:606-634
///           resolve                     AggregateVerifier.sol:443-474
///           _proofVerifiedUpdate        AggregateVerifier.sol:763-770
///           _decreaseExpectedResolution AggregateVerifier.sol:773-785
///           _getDelay                   AggregateVerifier.sol:815-823
contract BondLockupReproducer {
    enum ProofType { TEE, ZK }

    // NOTE: upstream names the first enum value IN_PROGRESS. Renamed to
    // ACTIVE here only so the bounty platform's markdown editor does not
    // auto link the GameStatus.IN substring as a URL. Behavior, ordering,
    // and default zero value are unchanged.
    enum GameStatus { ACTIVE, CHALLENGER_WINS, DEFENDER_WINS }

    uint64 public constant SLOW_FINALIZATION_DELAY = 7 days;
    uint64 public constant FAST_FINALIZATION_DELAY = 1 days;

    uint256 public PROOF_THRESHOLD;

    uint64  public createdAt;
    uint64  public resolvedAt;
    uint64  public expectedResolution;
    GameStatus public status;
    uint8   public proofCount;
    bool    public initialized;
    bool    public bondClaimed;
    bool    public bondUnlocked;
    uint256 public bondAmount;
    address public bondRecipient;
    mapping(ProofType => address) internal proofTypeToProver;

    error AlreadyInitialized();
    error GameNotResolved();
    error GameNotOver();
    error NotEnoughProofs();
    error NoCreditToClaim();
    error AlreadyProven(ProofType proofType);
    error InvalidProofThreshold();

    event Resolved(GameStatus status);
    event Proved(address indexed proposer, ProofType indexed proofType);

    constructor(uint256 proofThreshold) {
        if (proofThreshold != 1 && proofThreshold != 2) revert InvalidProofThreshold();
        PROOF_THRESHOLD = proofThreshold;
    }

    function initializeWith(ProofType proofType) external payable {
        if (initialized) revert AlreadyInitialized();
        initialized = true;
        createdAt = uint64(block.timestamp);
        expectedResolution = type(uint64).max;
        bondAmount = msg.value;
        bondRecipient = msg.sender;
        _proofVerifiedUpdate(proofType, msg.sender);
    }

    function initializeOnly() external payable {
        if (initialized) revert AlreadyInitialized();
        initialized = true;
        createdAt = uint64(block.timestamp);
        expectedResolution = type(uint64).max;
        bondAmount = msg.value;
        bondRecipient = msg.sender;
    }

    function submitOtherProof(ProofType proofType) external {
        if (status != GameStatus.ACTIVE) revert GameNotResolved();
        if (proofTypeToProver[proofType] != address(0)) revert AlreadyProven(proofType);
        _proofVerifiedUpdate(proofType, msg.sender);
    }

    function resolve() external returns (GameStatus) {
        if (status != GameStatus.ACTIVE) revert GameNotResolved();
        if (!gameOver()) revert GameNotOver();
        if (proofCount < PROOF_THRESHOLD) revert NotEnoughProofs();
        status = GameStatus.DEFENDER_WINS;
        resolvedAt = uint64(block.timestamp);
        emit Resolved(status);
        return status;
    }

    function claimCredit() external {
        if (bondClaimed) revert NoCreditToClaim();
        if (expectedResolution != type(uint64).max) {
            if (resolvedAt == 0) revert GameNotResolved();
        } else {
            if (block.timestamp < createdAt + 14 days) revert GameNotOver();
        }
        if (!bondUnlocked) {
            bondUnlocked = true;
            return;
        }
        bondClaimed = true;
    }

    function gameOver() public view returns (bool) {
        return expectedResolution <= block.timestamp;
    }

    function _proofVerifiedUpdate(ProofType proofType, address proposer) internal {
        proofTypeToProver[proofType] = proposer;
        proofCount += 1;
        _decreaseExpectedResolution();
        emit Proved(proposer, proofType);
    }

    function _decreaseExpectedResolution() internal {
        uint64 delay = _getDelay();
        if (delay == type(uint64).max) {
            expectedResolution = type(uint64).max;
            return;
        }
        uint64 newResolution = uint64(block.timestamp) + delay;
        expectedResolution = _min(newResolution, expectedResolution);
    }

    function _getDelay() internal view returns (uint64) {
        if (proofCount >= 2) return FAST_FINALIZATION_DELAY;
        if (proofCount == 1) return SLOW_FINALIZATION_DELAY;
        return type(uint64).max;
    }

    function _min(uint64 a, uint64 b) internal pure returns (uint64) {
        return a < b ? a : b;
    }
}
```

{% endcode %}

{% code title="test/Repro.t.sol" %}

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.15;

import "forge-std/Test.sol";
import {BondLockupReproducer} from "../src/BondLockupReproducer.sol";

contract F001_BondLockupTest is Test {
    BondLockupReproducer rep;
    address proposer = address(0xA11CE);
    uint256 constant BOND = 1 ether;

    function setUp() public {
        // deal is the StdCheats helper inherited via Test. Using it instead
        // of vm.deal so the bounty platform's markdown editor does not auto
        // link the .deal substring as a URL.
        deal(proposer, 10 ether);
    }

    function test_BondLocked_When_OnlyOneProof_And_ThresholdTwo() public {
        rep = new BondLockupReproducer(2);

        vm.prank(proposer);
        rep.initializeWith{value: BOND}(BondLockupReproducer.ProofType.TEE);

        assertEq(rep.bondAmount(), BOND);
        assertEq(rep.bondRecipient(), proposer);
        assertEq(uint256(rep.proofCount()), 1);
        assertTrue(rep.expectedResolution() != type(uint64).max);

        skip(30 days);

        vm.expectRevert(BondLockupReproducer.NotEnoughProofs.selector);
        rep.resolve();

        assertEq(uint256(rep.resolvedAt()), 0);

        vm.prank(proposer);
        vm.expectRevert(BondLockupReproducer.GameNotResolved.selector);
        rep.claimCredit();

        skip(365 days);
        vm.prank(proposer);
        vm.expectRevert(BondLockupReproducer.GameNotResolved.selector);
        rep.claimCredit();
    }

    function test_Baseline_ThresholdOne_ResolvesNormally() public {
        rep = new BondLockupReproducer(1);
        vm.prank(proposer);
        rep.initializeWith{value: BOND}(BondLockupReproducer.ProofType.TEE);
        skip(7 days + 1);
        BondLockupReproducer.GameStatus s = rep.resolve();
        assertEq(uint256(s), uint256(BondLockupReproducer.GameStatus.DEFENDER_WINS));
        assertGt(uint256(rep.resolvedAt()), 0);
        vm.prank(proposer);
        rep.claimCredit();
        assertTrue(rep.bondUnlocked());
        vm.prank(proposer);
        rep.claimCredit();
        assertTrue(rep.bondClaimed());
    }

    function test_Baseline_NoProofs_FourteenDayFallbackWorks() public {
        rep = new BondLockupReproducer(2);
        vm.prank(proposer);
        rep.initializeOnly{value: BOND}();
        assertEq(rep.expectedResolution(), type(uint64).max);
        assertEq(uint256(rep.proofCount()), 0);
        skip(14 days + 1);
        vm.prank(proposer);
        rep.claimCredit();
        assertTrue(rep.bondUnlocked());
        vm.prank(proposer);
        rep.claimCredit();
        assertTrue(rep.bondClaimed());
    }

    function test_Baseline_ThresholdTwo_BothProofs_ResolvesNormally() public {
        rep = new BondLockupReproducer(2);
        vm.prank(proposer);
        rep.initializeWith{value: BOND}(BondLockupReproducer.ProofType.TEE);
        rep.submitOtherProof(BondLockupReproducer.ProofType.ZK);
        skip(1 days + 1);
        BondLockupReproducer.GameStatus s = rep.resolve();
        assertEq(uint256(s), uint256(BondLockupReproducer.GameStatus.DEFENDER_WINS));
        vm.prank(proposer);
        rep.claimCredit();
        assertTrue(rep.bondUnlocked());
        vm.prank(proposer);
        rep.claimCredit();
        assertTrue(rep.bondClaimed());
    }
}
```

{% endcode %}
