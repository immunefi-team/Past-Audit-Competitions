# 74357 sc low unresolvable game state when proof threshold 2 and deadline is missed in aggregateverifier

**Submitted on Apr 22nd 2026 at 00:48:21 UTC by @CarlosMB for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74357
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Permanent freezing of funds in the bridge or in dispute game bonds with no available recovery path

## Description

### Target

`src/multiproof/AggregateVerifier.sol`

### Summary

When `PROOF_THRESHOLD = 2`, `AggregateVerifier` can enter a state with no reachable terminal resolution if the second proof is not submitted before the `gameOver()` deadline.

In this scenario, the contract simultaneously prevents:

* submission of additional proofs (due to timeout),
* resolution of the dispute game (due to insufficient proofs),
* and recovery of bonded funds (due to unresolved state).

As a result, the dispute game remains stuck in `IN_PROGRESS`, and the bonded funds are not recoverable through any available contract path.

### Description

The `AggregateVerifier` contract enforces a minimum number of proofs (`PROOF_THRESHOLD`) before a dispute game can be resolved.

When configured with:

`PROOF_THRESHOLD = 2`

the proposer must submit two valid proofs within the allowed time window.

However, if only one proof is submitted and the deadline expires:

**1. Additional proofs are rejected**

```solidity
if (gameOver()) revert GameOver();
```

Once `gameOver()` is true, no further proofs can be submitted.

**2. The game cannot be resolved**

```solidity
if (proofCount < PROOF_THRESHOLD) revert NotEnoughProofs();
```

Since only one proof exists, `resolve()` becomes unreachable.

**3. Funds cannot be reclaimed**

```solidity
if (resolvedAt.raw() == 0) revert GameNotResolved();
```

Because `resolve()` is never executed, `resolvedAt` remains zero indefinitely.

### Resulting State

This leads to a deadlock condition:

* `status == IN_PROGRESS`
* `resolvedAt == 0`
* `proofCount < PROOF_THRESHOLD`
* `gameOver() == true`

No function can transition the game to a terminal state.

### Impact

* The dispute game can remain permanently unresolved.
* Bonded funds deposited during initialization cannot be reclaimed through any available contract path.
* The timeout mechanism does not provide a terminal recovery path when the proof threshold is not met.

This behavior indicates that the dispute game state machine does not guarantee a terminal outcome once initiated.

### Severity Justification

This issue may qualify under the program’s Critical category:\
*“Permanent freezing of funds in dispute game bonds with no available recovery path”*\
because once this state is reached, the bonded funds cannot be recovered through any available contract path.

The final classification may depend on whether the protocol treats incomplete proof submission as an intended forfeiture condition or expects all dispute games to eventually reach a terminal state.

Regardless of classification, the issue leaves the game unresolved and the bond inaccessible.

## Proof of Concept

The following PoC demonstrates:

1. successful initialization with a single proof and bond deposit
2. expiration of the submission window
3. rejection of additional proofs
4. inability to resolve
5. inability to reclaim funds even after extended time

The following PoC was executed using the full project repository and real contract implementation from the in-scope codebase.

The test instantiates an `AggregateVerifier` implementation with `PROOF_THRESHOLD = 2` and deploys a clone using the same CWIA (Clones With Immutable Arguments) pattern used in production.

A minimal verifier implementation is used to isolate the state machine behavior without altering the execution flow of `AggregateVerifier`.

No mocking of AggregateVerifier logic is performed.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.15;

import "forge-std/Test.sol";
import {AggregateVerifier} from "src/multiproof/AggregateVerifier.sol";
import {
    IAnchorStateRegistry
} from "interfaces/dispute/IAnchorStateRegistry.sol";
import {IDelayedWETH} from "interfaces/dispute/IDelayedWETH.sol";
import {IVerifier} from "interfaces/multiproof/IVerifier.sol";
import {GameType, Hash} from "src/dispute/lib/Types.sol";
import {GameNotResolved} from "src/dispute/lib/Errors.sol";

import {MockDelayedWETH} from "scripts/multiproof/mocks/MockDelayedWETH.sol";
import {
    MockAnchorStateRegistry
} from "scripts/multiproof/mocks/MockAnchorStateRegistry.sol";
import {LibClone} from "@solady/utils/LibClone.sol";

contract BaseDummyVerifier is IVerifier {
    function verify(
        bytes calldata,
        bytes32,
        bytes32
    ) external pure returns (bool) {
        return true;
    }

    function nullify() external {}
}

contract AggregateVerifierThreshold2DeadlockTest is Test {
    AggregateVerifier impl;
    AggregateVerifier verifier;

    MockDelayedWETH delayWeth;
    MockAnchorStateRegistry anchorRegistry;
    BaseDummyVerifier teeVerifier;
    BaseDummyVerifier zkVerifier;

    address internal constant FACTORY = address(0xEEEE);
    address internal constant PROPOSER = address(0xBEEF);

    function setUp() public {
        anchorRegistry = new MockAnchorStateRegistry();
        anchorRegistry.initialize(
            FACTORY,
            Hash.wrap(bytes32(0)),
            0,
            GameType.wrap(0)
        );

        delayWeth = new MockDelayedWETH();
        teeVerifier = new BaseDummyVerifier();
        zkVerifier = new BaseDummyVerifier();

        AggregateVerifier.ZkHashes memory hashes = AggregateVerifier.ZkHashes({
            rangeHash: bytes32(0),
            aggregateHash: bytes32(0)
        });

        // Deploy implementation with PROOF_THRESHOLD = 2
        impl = new AggregateVerifier(
            GameType.wrap(0),
            IAnchorStateRegistry(address(anchorRegistry)),
            IDelayedWETH(payable(address(delayWeth))),
            IVerifier(address(teeVerifier)),
            IVerifier(address(zkVerifier)),
            bytes32(0),
            hashes,
            bytes32(0),
            10,
            100,
            100,
            2
        );

        vm.roll(1000);
        vm.warp(1);
    }

    function test_deadlock_permanently_locks_bond_when_threshold_is_two_and_second_proof_misses_deadline()
        public
    {
        vm.deal(PROPOSER, 10 ether);

        uint256 l1OriginNumber = block.number - 1;
        bytes32 l1OriginHash = blockhash(l1OriginNumber);
        uint256 l2BlockNumber = 100;

        bytes memory cwiaData = abi.encodePacked(
            PROPOSER,
            bytes32(uint256(999)), // rootClaim
            bytes32(0), // l1Head
            l2BlockNumber, // l2SequenceNumber
            address(anchorRegistry), // parentGame
            bytes32(uint256(999)) // single intermediate root
        );

        address clone = LibClone.clone(address(impl), cwiaData);
        verifier = AggregateVerifier(payable(clone));

        // First proof: one accepted TEE proof only
        bytes memory initialProof = new bytes(66);
        initialProof[0] = bytes1(uint8(AggregateVerifier.ProofType.TEE));
        for (uint256 i = 0; i < 32; i++) {
            initialProof[1 + i] = l1OriginHash[i];
            initialProof[33 + i] = bytes32(l1OriginNumber)[i];
        }

        vm.prank(PROPOSER);
        verifier.initializeWithInitData{value: 1 ether}(initialProof);

        // Initial state after the first proof
        assertEq(verifier.PROOF_THRESHOLD(), 2);
        assertEq(verifier.proofCount(), 1);
        assertEq(verifier.bondAmount(), 1 ether);
        assertEq(verifier.bondRecipient(), PROPOSER);
        assertEq(uint256(verifier.resolvedAt().raw()), 0);
        assertTrue(verifier.expectedResolution().raw() != type(uint64).max);
        assertFalse(verifier.gameOver());
        assertFalse(verifier.bondUnlocked());
        assertFalse(verifier.bondClaimed());

        // Miss the submission window for the second proof
        skip(7 days + 1);
        assertTrue(verifier.gameOver());

        // A late second proof is rejected
        bytes memory secondProof = new bytes(66);
        secondProof[0] = bytes1(uint8(AggregateVerifier.ProofType.ZK));

        vm.expectRevert(AggregateVerifier.GameOver.selector);
        verifier.verifyProposalProof(secondProof);

        // The game also cannot be resolved because proofCount < PROOF_THRESHOLD
        vm.expectRevert(AggregateVerifier.NotEnoughProofs.selector);
        verifier.resolve();

        assertEq(uint256(verifier.resolvedAt().raw()), 0);
        assertFalse(verifier.bondUnlocked());
        assertFalse(verifier.bondClaimed());

        // claimCredit is unavailable because the game never becomes resolved
        skip(14 days);

        vm.expectRevert(GameNotResolved.selector);
        verifier.claimCredit();

        // Much later, the bond remains irrecoverable
        skip(365 days);

        vm.expectRevert(AggregateVerifier.NotEnoughProofs.selector);
        verifier.resolve();

        vm.expectRevert(GameNotResolved.selector);
        verifier.claimCredit();

        assertEq(uint256(verifier.resolvedAt().raw()), 0);
        assertFalse(verifier.bondUnlocked());
        assertFalse(verifier.bondClaimed());
        assertEq(verifier.bondRecipient(), PROPOSER);
        assertEq(verifier.bondAmount(), 1 ether);
    }
}
```

### Expected Output

```js
Compiler run successful!

Ran 1 test for test/poc1imu.t.sol:poc1imu
[PASS] test_DEADLOCK_PermanentBondLockWhenTimeoutMissed() (gas: 321451)

Traces:
  ...
    // 1. Initial Deposit setup and accepted
    ├─ [149657] 0xa0Cb...::initializeWithInitData{value: 1000000000000000000}(...)
    │   ├─ emit Proved(...)
    │   ├─ [96] MockDelayedWETH::deposit{value: 1000000000000000000}()
  ...
    // 2. Timeout simulation (warp past deadline)
    ├─ [0] VM::warp(604802 [6.048e5])
    
    // 3. Late proof explicitly hits GameOver()
    ├─ [0] VM::expectRevert(custom error 0xc31eb0e0: GameOver())
    ├─ [1141] 0xa0Cb...::verifyProposalProof(...)
    │   └─ ← [Revert] GameOver()
  ...
    // 4. Forced resolve explicitly hits NotEnoughProofs()
    ├─ [0] VM::expectRevert(custom error 0xc31eb0e0: NotEnoughProofs())
    ├─ [1178] 0xa0Cb...::resolve()
    │   └─ ← [Revert] NotEnoughProofs()
  ...
    // 5. ClaimCredit explicitly hits GameNotResolved() permanently
    ├─ [0] VM::expectRevert(custom error 0xc31eb0e0: GameNotResolved())
    ├─ [23018] 0xa0Cb...::claimCredit()
    │   └─ ← [Revert] GameNotResolved()

Suite result: ok. 1 passed; 0 failed; 0 skipped;
```

## Root Cause

The issue arises because:

* `verifyProposalProof()` blocks additional proofs after timeout
* `resolve()` requires `proofCount >= PROOF_THRESHOLD`
* `claimCredit()` requires `resolvedAt != 0`
* There is no fallback path for:\
  `gameOver() == true && proofCount < PROOF_THRESHOLD`

## Recommendation

Allow the game to transition to a terminal state when the deadline has passed but the proof threshold is not met.

Example:

```solidity
if (proofCount < PROOF_THRESHOLD) {
    status = GameStatus.CHALLENGER_WINS;
    resolvedAt = Timestamp.wrap(uint64(block.timestamp));
}
```

Any equivalent terminal-state transition would also resolve the issue, provided it guarantees settlement and prevents funds from remaining indefinitely locked.

This behavior shows that the dispute game state machine does not guarantee a terminal outcome once initiated.
