# 75433 sc low permanent bond lock when proof threshold 2 and only one proof is submitted in aggregateverifier&#x20;

Submitted on Apr 29th 2026 at 04:34:48 UTC by @Jornason for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75433
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Permanent freezing of funds in the bridge or in dispute game bonds with no available recovery path

## Description

## Brief/Intro

When the `AggregateVerifier` dispute game is deployed with `PROOF_THRESHOLD = 2` (the dual-proof configuration that grants 1-day fast finalization) and only the FIRST proof is ever submitted — i.e., a TEE proof via `initializeWithInitData(...)` lands but no companion ZK proof arrives via `verifyProposalProof(...)` before the 7-day SLOW window elapses — the proposer's bond becomes **permanently irrecoverable**. The lock is a deterministic state-machine consequence of three independent facts: `resolve()` is gated by `proofCount >= PROOF_THRESHOLD`, `claimCredit()`'s 14-day fallback is gated by `expectedResolution == type(uint64).max` (which a single proof submission moves off the sentinel), and self-rescue via `nullify()` / `challenge()` requires a counter-proof an honest proposer does not possess. The bond stays trapped in `DelayedWETH` indefinitely.

## Vulnerability Details

**Code citations against commit `0618859` of the contracts repo:**

1. `_proofVerifiedUpdate` (lines 763-770) calls `_decreaseExpectedResolution`, which moves `expectedResolution` from `type(uint64).max` to `block.timestamp + SLOW_FINALIZATION_DELAY` (7 d) on the first proof.
2. `resolve()` (lines 443-474) — when the parent game is **healthy** (`_getParentGameStatus() == DEFENDER_WINS`), the function falls through to `if (proofCount < PROOF_THRESHOLD) revert NotEnoughProofs;`. Multiproof Audit 1 finding #1 already fixed the analogous issue for `parent == CHALLENGER_WINS` (lines 453-455 short-circuit when parent invalid), but the healthy-parent path retains the unconditional threshold check.
3. `claimCredit()` (lines 606-634) — branch:

   ```solidity
   if (expectedResolution.raw() != type(uint64).max) {
       if (resolvedAt.raw() == 0) revert GameNotResolved();
   } else {
       if (block.timestamp < createdAt.raw() + 14 days) revert GameNotOver();
   }
   ```

   Because `expectedResolution` was moved off the sentinel by step 1, the 14-day fallback never triggers.
4. `challenge()` (lines 480-541) and `nullify()` (lines 548-602) both require a valid counter-proof against an intermediate root that **differs** from the proposed one (see `_checkIntermediateRoot` at lines 1001-1006). An honest proposer cannot produce such a counter-proof.

The bond stays escrowed in `DelayedWETH` (deposited at `AggregateVerifier.sol:412`). The only recovery paths are `DelayedWETH.recover()` (privileged ProxyAdmin owner — out-of-scope per Immunefi rules) or per-game `blacklistDisputeGame`, neither of which is part of normal protocol operation.

**This case is NOT covered by Multiproof Audit 1 finding #1.** That fix at lines 453-455 only short-circuits when `_getParentGameStatus() == CHALLENGER_WINS`. The case demonstrated here has a healthy parent and falls through to the threshold revert.

## Impact Details

**Estimated economic damage**

* **Per stuck game**: One full proposer bond is locked permanently. Base's `DisputeGameFactory.initBonds()` for the multiproof game type determines the exact value. The protocol's own test harness (`BaseTest.t.sol`) uses `INIT_BOND = 1 ether` as a stand-in. At current ETH prices (\~$1,800), each occurrence locks **≥ $1,800** with no recovery path. The actual production bond may be higher — Base has historically used bonds in the 0.08–1 ETH range for its OP-Stack dispute games.
* **Cumulative exposure**: A proposer operating a fleet of N concurrent games is exposed to N × bond value. If the ZK prover network goes offline for 24 hours and the proposer has submitted TEE proofs for 10 games in that window, the total locked capital is **10 × bond** with zero on-chain recourse.
* **Liveness impact**: Repeated bond loss exhausts the proposer's working capital, degrading the chain's L1 finalization liveness. Base currently operates with a small set of Coinbase-run proposers — each lost bond directly reduces available bonding capacity.

**Direct impact**

* One full proposer bond is locked indefinitely per occurrence. There is no time-based escape, no self-rescue path, and no on-chain recovery mechanism within the in-scope attack surface.
* A TEE proposer that submits a proof but loses contact with the ZK prover network (no public-permissionless ZK prover available, ZK service down, network partition) cannot recover its bond.

**Indirect impact**

* There is no on-chain economic incentive for a permissionless ZK prover to supply the missing companion proof — submitting it costs gas, salvages someone else's bond, and earns no on-chain reward.
* Composes with any future `PROOF_THRESHOLD = 2` deployment: every active proposer becomes simultaneously exposed.
* If a proof-system soundness incident triggers `IVerifier.nullify()` on the ZK verifier, every TEE-only game in flight falls into this lock.

**Severity classification**

The PDF text of the C6 entry under "Smart Contract — Critical" is: *"Permanent freezing of bridge funds / dispute bonds with no recovery."* This finding is the literal definition of that text — dispute bonds are permanently frozen, with no on-chain recovery path inside the in-scope attack surface. The `DelayedWETH.recover()` admin function exists but is privileged and out-of-scope per competition rules.

## Recommendation

Two non-mutually-exclusive options, in increasing strength:

### Option A — Time-based escape inside `resolve()`

Add a `STALE_GAME_WINDOW` (e.g. 14-30 d) that, after expiry with insufficient proofs, resolves the game as `CHALLENGER_WINS` so the bond is at least claimable through the existing `claimCredit()` path (or burned, designer's choice):

```solidity
function resolve() external returns (GameStatus) {
    if (status != GameStatus.IN_PROGRESS) revert ClaimAlreadyResolved();
    GameStatus parentGameStatus = _getParentGameStatus();
    if (parentGameStatus == GameStatus.IN_PROGRESS) revert ParentGameNotResolved();

    if (parentGameStatus == GameStatus.CHALLENGER_WINS) {
        status = GameStatus.CHALLENGER_WINS;
    } else {
        if (!gameOver()) revert GameNotOver();
        if (proofCount < PROOF_THRESHOLD) {
+           if (block.timestamp >= createdAt.raw() + STALE_GAME_WINDOW) {
+               status = GameStatus.CHALLENGER_WINS;
+               resolvedAt = Timestamp.wrap(uint64(block.timestamp));
+               emit Resolved(status);
+               return status;
+           }
            revert NotEnoughProofs();
        }
        ...
    }
    ...
}
```

### Option B — Make the `claimCredit()` 14-day fallback unconditional after the cliff

```solidity
function claimCredit() external nonReentrant {
    if (bondClaimed) revert NoCreditToClaim();

-   if (expectedResolution.raw() != type(uint64).max) {
-       if (resolvedAt.raw() == 0) revert GameNotResolved();
-   } else {
-       if (block.timestamp < createdAt.raw() + 14 days) revert GameNotOver();
-   }
+   bool resolvedNormally = expectedResolution.raw() != type(uint64).max && resolvedAt.raw() != 0;
+   bool fallbackElapsed = block.timestamp >= createdAt.raw() + 14 days;
+   if (!resolvedNormally && !fallbackElapsed) revert GameNotResolved();
    ...
}
```

Both fixes are localized and consistent with the existing audit-driven design intent (Audit 1's CHALLENGER\_WINS short-circuit was clearly meant to ensure bond recovery is always reachable).

## References

Repository: `https://github.com/base/contracts` (Implementation Contracts, in scope per Immunefi competition page)

* `src/multiproof/AggregateVerifier.sol#L443-L474` — `resolve()` body, threshold revert site
* `src/multiproof/AggregateVerifier.sol#L606-L634` — `claimCredit()` body, sentinel-gated fallback
* `src/multiproof/AggregateVerifier.sol#L763-L770` — `_proofVerifiedUpdate` (moves `expectedResolution` off the sentinel)
* `src/multiproof/AggregateVerifier.sol#L773-L785` — `_decreaseExpectedResolution`
* `src/multiproof/AggregateVerifier.sol#L412` — bond escrow into DelayedWETH
* `src/dispute/DelayedWETH.sol#L96-L104` — withdraw path showing bonds are stuck without `unlock`

Audit cross-reference: Multiproof Audit 1 (Cantina, 2026-03-23) — finding #1 ("Unconditional proof threshold check") fixed the parent-CHALLENGER-WINS case only. The healthy-parent case proven here was not covered.

## Link to Proof of Concept

<https://gist.github.com/Jornason/ee71634a3490f9ea8566228990138361>

## Proof of Concept

A runnable Foundry test is provided at:

**File:** `contracts/test/multiproof/poc/HypothesisH_B_BondLockSingleProof.t.sol`

The test inherits from the protocol's own `contracts/test/multiproof/BaseTest.t.sol` harness (which already wires up `MockVerifier` returning `true` for any input, the AnchorStateRegistry / DisputeGameFactory / DelayedWETH proxy chain, and `MockSystemConfig`), then redeploys the `AggregateVerifier` implementation with `PROOF_THRESHOLD = 2` and walks the lock end-to-end.

### Reproduction

```bash
# One-time setup: install upstream deps
cd contracts
# On Windows, switch git's SSL backend to OpenSSL first to avoid schannel handshake failures:
#   git config --global http.sslBackend openssl
make deps
# If `make` is not in PATH on Windows, copy the forge install commands from contracts/Makefile.

# Run the PoC. FOUNDRY_PROFILE=ci is required because the default `lite` profile fails
# to compile some upstream Safe / OpenZeppelin contracts with stack-too-deep errors.
FOUNDRY_PROFILE=ci forge test \
    --match-contract HypothesisHB_BondLockSingleProof -vv
```

### Expected output (verbatim)

```
Ran 3 tests for test/multiproof/poc/HypothesisH_B_BondLockSingleProof.t.sol:HypothesisHB_BondLockSingleProof
[PASS] test_HB_secondProof_breaksTheLock_whenAvailable() (gas: 635106)
[PASS] test_HB_singleTEEProof_permanentlyLocksBond() (gas: 593356)
[PASS] test_HB_thresholdOne_singleTEEProof_resolvesCleanly() (gas: 3792559)
Suite result: ok. 3 passed; 0 failed; 0 skipped
```

### What each test asserts

* `test_HB_singleTEEProof_permanentlyLocksBond` — **the primary PoC**. After a single TEE proof is submitted to a `PROOF_THRESHOLD = 2` game, the test fast-forwards to the deadline, asserts `resolve()` reverts with `NotEnoughProofs`, then advances by 7 d, 14 d, and 1 year — `claimCredit()` reverts with `GameNotResolved` at every checkpoint. Final assertion: `address(delayedWETH).balance == INIT_BOND`, i.e., the bond is still trapped in `DelayedWETH`.
* `test_HB_thresholdOne_singleTEEProof_resolvesCleanly` — sanity check: the same flow with `PROOF_THRESHOLD = 1` recovers the bond cleanly, isolating the lock to the threshold = 2 case.
* `test_HB_secondProof_breaksTheLock_whenAvailable` — sanity check: a permissionless ZK proof submission via `verifyProposalProof` unsticks the bond, confirming the lock manifests precisely when no companion proof arrives within the SLOW window.
