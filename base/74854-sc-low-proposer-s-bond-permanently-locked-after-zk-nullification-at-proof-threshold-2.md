# 74854 sc low proposer s bond permanently locked after zk nullification at proof threshold 2

**Submitted on Apr 25th 2026 at 11:04:37 UTC by @hexproof for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74854
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Permanent freezing of funds in the bridge or in dispute game bonds with no available recovery path

## Description

### Brief/Intro

When `AggregateVerifier` is deployed with `PROOF_THRESHOLD=2`, a ZK nullification that drops `proofCount` from `2` to `1` leaves `expectedResolution` at a finite value (`now + SLOW_FINALIZATION_DELAY`, i.e. now + 7 days) instead of `type(uint64).max`.

Both on-chain exits for the proposer's `INIT_BOND` are then permanently closed:

* `resolve()` reverts `NotEnoughProofs` forever
* `claimCredit()`'s 14-day escape hatch is gated out because it requires `expectedResolution == type(uint64).max`

The proposer's bond is structurally inaccessible for the lifetime of the contract for first-game deployments. The defect is asymmetric: the analogous TEE-side nullification (`1 → 0`) correctly sets `expectedResolution = type(uint64).max` and the bond returns.

The trigger is the protocol's own documented response to a soundness issue, permissionlessly invocable via `nullify()` at L548 with no caller modifier.

### Vulnerability Details

Three pieces of source must be read together. Line numbers reference `github.com/base/contracts/tree/v8.1.0/src/multiproof/AggregateVerifier.sol`.

1. The constructor accepts threshold `= 2` (L285):

```solidity
if (proofThreshold != 1 && proofThreshold != 2)
    revert InvalidProofThreshold();
```

`PROOF_THRESHOLD=2` is a first-class supported configuration — not an unreachable code path.

2. `_proofRefutedUpdate` decrements `proofCount`, then calls `_increaseExpectedResolution` (L788-798):

```solidity
function _proofRefutedUpdate(ProofType proofType) internal {
    delete proofTypeToProver[proofType];
    if (proofCount == 0) revert NotEnoughProofs();
    unchecked { proofCount -= 1; }
    _increaseExpectedResolution();
}
```

`_increaseExpectedResolution` (L800-823) reads `_getDelay()` and branches:

```solidity
function _increaseExpectedResolution() internal {
    uint64 delay = _getDelay();
    if (delay == type(uint64).max) {
        expectedResolution = Timestamp.wrap(type(uint64).max);
        return;
    }
    expectedResolution = Timestamp.wrap(uint64(block.timestamp) + delay);
}

function _getDelay() internal view returns (uint64) {
    if (proofCount >= 2)      return FAST_FINALIZATION_DELAY;
    else if (proofCount == 1) return SLOW_FINALIZATION_DELAY;
    else                      return type(uint64).max;
}
```

At `PROOF_THRESHOLD=2`, the `2 → 1` ZK nullify lands `proofCount` on `1`, so `_getDelay` returns `SLOW_FINALIZATION_DELAY` (7 days). The `uint64.max` branch is **not** taken. `expectedResolution` is now `+ 7d`.

3. `claimCredit()` at L606-630 then has no working exit:

```solidity
function claimCredit() external nonReentrant {
    if (bondClaimed) revert NoCreditToClaim();
    if (expectedResolution.raw() != type(uint64).max) {
        if (resolvedAt.raw() == 0) revert GameNotResolved();
    } else {
        if (block.timestamp < createdAt.raw() + 14 days)
            revert GameNotOver();
    }
    ...
}
```

The `else` branch (the 14-day escape hatch) is gated on `expectedResolution == type(uint64).max`. Since step 2 leaves it finite, control enters the `if` branch and reverts `GameNotResolved` at L614 because `resolve()` at L458 cannot satisfy `proofCount >= PROOF_THRESHOLD`.

#### Asymmetry

A TEE-side nullification (`proofCount 1 → 0`) puts `proofCount` in the `else` branch of `_getDelay`, which returns `type(uint64).max`. That sets `expectedResolution = type(uint64).max`, the 14-day escape hatch opens, and the bond returns.

Only the ZK-side `2 → 1` transition leaves the bond trapped.

### First-game scoping (Rule A pre-emption)

The Audit Comp Base Azul Scope tab states under “What emergency actions may you want to use as a reason to downgrade an otherwise valid bug report?”:

> "Any report that assumes we will not dispute/blacklist/retire an invalid proposal within the proof system, unless it can be shown that such an action cannot be taken."

For this finding, that action cannot recover the bond, and the in-scope source proves it. `_getParentGameStatus` at L938-950:

```solidity
function _getParentGameStatus() internal view returns (GameStatus) {
    if (parentAddress() != address(ANCHOR_STATE_REGISTRY)) {
        IDisputeGame parentGame = IDisputeGame(parentAddress());
        if (
            ANCHOR_STATE_REGISTRY.isGameBlacklisted(parentGame)
            || ANCHOR_STATE_REGISTRY.isGameRetired(parentGame)
        ) {
            return GameStatus.CHALLENGER_WINS;
        }
        return parentGame.status();
    }
    // First dispute game branch:
    return GameStatus.DEFENDER_WINS;            // L949
}
```

For a first dispute game — the canonical case for any new output proposal posted against the current anchor — `parentAddress() == address(ANCHOR_STATE_REGISTRY)`, so L949 returns `DEFENDER_WINS` unconditionally.

The blacklist/retire branch at L941-944 is unreachable, because the `ANCHOR_STATE_REGISTRY` is not itself a blacklistable parent game. `resolve()` therefore enters the else branch (L455-467), gated by `proofCount >= PROOF_THRESHOLD`, and reverts `NotEnoughProofs` forever.

For chained games (`parentAddress() != ASR`) the parent-blacklist short-circuit does fire, and the bond is not re-trapped because `bondRecipient` is only flipped to ZK in the else branch (L461-463) — not in the parent-`CHALLENGER_WINS` branch (L453-454). The lock is therefore confined to first-game deployments, but in that configuration it is genuinely unrecoverable: the protocol cannot “blacklist the parent” because the parent **is** the `ANCHOR_STATE_REGISTRY`.

### Realisation conditions

* `PROOF_THRESHOLD` must be `2`. The constructor at L285 supports it as a first-class configuration. Sepolia activate-multiproof currently configures `=1` (defect dormant on current testnet, per `github.com/base/contract-deployments/blob/main/sepolia/2026-04-20-activate-multiproof/.env`); mainnet's threshold is not predetermined from in-scope source.
* A ZK nullification must occur. `nullify()` at L548 has no caller modifier — anonymous EOAs with a contradiction proof can trigger it. The Scope tab's “Soundness alert mechanism” explicitly describes the trigger as “two valid same-type proofs of conflicting roots” (see Impact Details for the verbatim quote).
* Both conditions are protocol-documented operational realities, not unrealistic adversarial setups.

### Affected

Every **first** `AggregateVerifier` dispute game (`parent = ASR`) deployed with `PROOF_THRESHOLD=2` once a ZK nullification has dropped `proofCount` from `2` to `1`. The honest proposer who posted `INIT_BOND` at L411 is the loss-bearing party.

Chained games are recoverable through the parent-blacklist exit.

### What is NOT vulnerable

* `PROOF_THRESHOLD=1` deployments — the `2 → 1` transition is not part of the state machine. Test 3 in the PoC confirms.
* TEE-side nullification path (`proofCount 1 → 0`). The else branch of `_getDelay` returns `type(uint64).max`; the 14-day escape hatch opens; bond returns. Test 2 confirms.
* Chained dispute games (`parent = another AggregateVerifier` game): when the parent can be blacklisted/retired via the `AnchorStateRegistry`'s privileged blacklist/retire role, `_getParentGameStatus` returns `CHALLENGER_WINS`, `resolve()` short-circuits at L453-454 to `CHALLENGER_WINS` without flipping `bondRecipient`. Bond returns via the standard claim path.

## Impact Details

Immunefi taxonomy: **Permanent freezing of funds**.

The proposer's posted `INIT_BOND` becomes structurally inaccessible for first-game deployments (`parent = ANCHOR_STATE_REGISTRY`):

* `resolve()` reverts forever with `NotEnoughProofs` (L458).
* `claimCredit()` reverts forever with `GameNotResolved` (L614).
* No admin / governance / owner / forceResolve path exists. Verified by inspection of every external function on `AggregateVerifier` (`initializeWithInitData`, `verifyProposalProof`, `challenge`, `nullify`, `resolve`, `claimCredit`). PoC Test 1 exhausts every remaining external call from the locked state and confirms each one reverts.
* The parent-blacklist exit is unreachable for first games (per L949 — see Vulnerability Details).
* No state-only path returns the bond. The only theoretical exit is a TEE-signed counter-proof producing a TEE-side `proofCount 1 → 0` transition, but an honest TEE will never sign a journal asserting an intermediate root the proposer never submitted (`_verifyTeeProof` L870-895 forces `intermediateRootToProve` into the journal). Recovery requires TEE compromise on top of the trigger — outside the protocol's intended threat model.

Financial impact: the bond is whatever `msg.value` the factory enforces at game creation (`INIT_BOND`, L411). Larger bonds make the loss larger; pool-of-bonds across many concurrent games makes the systemic loss worse.

### Rule B pre-emption (invalid-proof reliance)

The Scope tab states:

> "Any report relying on an invalid TEE or ZK proof will be downgraded, especially TEE proofs unless it can be shown that a key compromise is unnecessary."

This finding does not rely on an invalid proof. The Scope tab's own description of the protocol mechanism establishes the correct framing:

> "Soundness alert mechanism — if two different state roots both have valid proofs of the same type (i.e. both are TEE or both are ZK), the game can be automatically nullified"

Per the program's own description, `nullify()` fires from two valid proofs of the same type asserting different state roots — not from an invalid proof. Two independent ZK provers (or one prover producing two outputs across a software-revision or input-divergence event) producing valid proofs of contradicting intermediate roots is sufficient. No private key compromise of any prover is required.

The Scope tab also enumerates this as a primary concern:

> "Proof system integration — TEE/ZK dispute game submission logic, verifier contract bugs, and soundness issues in ZK recursion"

The on-chain evidence matches: `nullify()` at L548 has no caller modifier; `AggregateVerifier.nullify()` can be invoked by any address that produces a verifying counter-proof of a different intermediate root, cascading into `ZK_VERIFIER.nullify()` at L598. PoC Test 4 demonstrates an anonymous EOA with zero balance triggering this cascade and reaching the lock state — no privileged role, no guardian, no key compromise.

The trigger condition is therefore a “valid-proofs-collide” event the program explicitly enumerates as the Soundness alert mechanism, which the contract correctly implements at L594-598. The finding is the dead-end the contract enters **after** nullification — the absence of a recovery path for the proposer's bond once the protocol's own designed-for-soundness-issues mechanism has fired.

## Recommended Mitigation

Either:

1. drop the asymmetry by making `_getDelay` return `type(uint64).max` whenever `proofCount` drops below threshold from a higher value, or
2. widen the `claimCredit` escape hatch gate to include `expectedResolution finite but unreachable` states.

Concrete patch (option a) — change `_increaseExpectedResolution` at L800-823:

```solidity
function _increaseExpectedResolution() internal {
    // After a nullification, if proofs are insufficient to
    // resolve, freeze resolution and let the 14-day claimCredit
    // hatch run.
    if (proofCount < PROOF_THRESHOLD) {
        expectedResolution = Timestamp.wrap(type(uint64).max);
        return;
    }
    uint64 delay = _getDelay();
    expectedResolution = Timestamp.wrap(uint64(block.timestamp) + delay);
}
```

Equivalent fix (option b) — change `claimCredit` at L613:

```solidity
if (expectedResolution.raw() != type(uint64).max
    && proofCount >= PROOF_THRESHOLD) {
    if (resolvedAt.raw() == 0) revert GameNotResolved();
} else {
    if (block.timestamp < createdAt.raw() + 14 days)
        revert GameNotOver();
}
```

Either change closes the trap. Option (a) is preferred because it keeps the invariant local to the resolution-time bookkeeping.

## References

### In-scope source

All line citations below are verifiable against the live files:

* `AggregateVerifier.sol` — <https://github.com/base/contracts/blob/v8.1.0/src/multiproof/AggregateVerifier.sol>
* `L285` — constructor accepts `proofThreshold ∈ {1, 2}`
* `L411` — `INIT_BOND` deposited into `DELAYED_WETH`
* `L458` — `resolve()` reverts `NotEnoughProofs` if `proofCount < PROOF_THRESHOLD`
* `L548` — `nullify()` (no caller modifier)
* `L594-598` — ZK-nullify branch: delete countered + `ZK_VERIFIER.nullify()`
* `L613-617` — `claimCredit` two-branch gate
* `L788-798` — `_proofRefutedUpdate` decrements `proofCount`
* `L800-823` — `_increaseExpectedResolution` + `_getDelay`
* `L938-950` — `_getParentGameStatus` (`L949` unconditional `DEFENDER_WINS` for first games)
* `Verifier.sol` — <https://github.com/base/contracts/blob/v8.1.0/src/multiproof/Verifier.sol>
* `L13` — “used to prevent further proof verification after a soundness issue is found”

### Deployment / configuration evidence

* <https://github.com/base/contract-deployments/blob/main/sepolia/2026-04-20-activate-multiproof/.env> — `PROOF_THRESHOLD=1` (current Sepolia configuration; defect dormant)
* <https://github.com/base/contract-deployments/blob/main/sepolia/2026-04-20-activate-multiproof/addresses.json> — `AggregateVerifier` deployed at `0x498313fB340CD5055c5568546364008299A47517`

### Program rule citations

All from the Audit Comp Base Azul Scope tab:

* “Soundness alert mechanism” — protocol-documented `nullify` trigger
* “Proof system integration — TEE/ZK dispute game submission logic, verifier contract bugs, and soundness issues in ZK recursion” — primary concern naming
* Rule A (parent-blacklist downgrade) and Rule B (invalid-proof downgrade) — pre-empted in Vulnerability Details and Impact Details respectively

## Link to Proof of Concept

<https://gist.github.com/calebtkjordan/df10023d5df1a7760cedc5ff3e7eae59>

## Proof of Concept

The PoC is a pure in-memory Foundry test (no fork, no RPC). It uses only the in-scope `MockVerifier` (Base's own test harness) plus minimal plumbing harnesses under `test/harness/` that satisfy the `DELAYED_WETH` and `AnchorStateRegistry` interfaces. Every state transition that demonstrates the bug is executed by the unmodified in-scope `AggregateVerifier.sol`.

The reference Foundry project is standalone (`solc = 0.8.15`, `evm_version = "london"`) because Base's multiproof contracts require those settings.

### Step-by-step reproduction

Set up a Foundry project configured with `solc=0.8.15 / evm_version=london`. Vendor minimal Solady (`utils/Clone`, `LibClone`, `FixedPointMathLib`, `ReentrancyGuard`, `EnumerableSetLib`) and stub the Optimism dispute interfaces (`IAnchorStateRegistry`, `IDelayedWETH`, `IDisputeGame`, `IDisputeGameFactory`). Drop the attached PoC files into `test/baseazul/` and `test/harness/`.

Run the suite (4 tests):

```bash
forge test --match-contract BaseAzul_BondLockOnZKNullify -vvv
```

Expected output (all 4 pass):

#### Test 1 — `test_ZK_nullification_locks_bond_at_threshold_2` (the lock demo)

End-to-end lock sequence: init at `PROOF_THRESHOLD=2` → ZK challenge → guardian/anon ZK-nullify (`2 → 1`) → warp → `resolve()` reverts `NotEnoughProofs` (L458) → `claimCredit()` reverts `GameNotResolved` (L614).

Phase prints under `-vvv` show `proofCount` and `expectedResolution` at each step:

* `post-init : proofCount=1, expectedResolution=now+7d`
* `post-challenge : proofCount=2, expectedResolution=now+7d`
* `post-zk-nullify: proofCount=1, expectedResolution=now+7d`
* `(NOT type(uint64).max — the lock)`

Final assertions: `proposer.balance == pre - INIT_BOND`, the `INIT_BOND` is still parked in `DELAYED_WETH`, and the game is stuck `IN_PROGRESS`.

#### Test 2 — `test_TEE_nullification_bond_recovers_at_threshold_2_comparison` (the asymmetry comparison)

Same harness, but a TEE-side nullification (`1 → 0`). Phase print shows `expectedResolution` flips to `type(uint64).max`, so the 14-day `claimCredit` escape hatch opens and the bond returns. Documents the asymmetry — same contract, same bond, same threshold, only the nullified side differs.

#### Test 3 — `test_threshold_1_unaffected` (defect dormant at threshold=1)

`PROOF_THRESHOLD=1` deployment. Init → warp → `resolve()` succeeds (`DEFENDER_WINS`) → `claimCredit` returns the bond. Confirms the defect is dormant when threshold=1 (the current Sepolia configuration per `github.com/base/contract-deployments/blob/main/sepolia/2026-04-20-activate-multiproof/.env`).

#### Test 4 — `test_nullify_is_callable_under_realistic_conditions` (Path b — anon-permissionless trigger)

Pre-empts the “is `nullify()` really reachable?” rebuttal in two parts.

1. `Verifier.sol:39-47` access gate rejects unauthorized callers (EOA, unregistered contract, half-registered contract) with `NotProperGame`.
2. `AggregateVerifier.nullify()` at L548 has **no modifier** — an anonymous EOA with no balance and no privilege can submit a contradiction proof and cascade into `ZK_VERIFIER.nullify()` because the `AggregateVerifier` clone is itself a registered proper+respected dispute game.

After the anon-triggered cascade the post-state matches Test 1's lock state exactly.

### Source pointers verifiable against the unmodified in-scope source

#### `AggregateVerifier.sol`

* `L285` — constructor accepts `proofThreshold` in `{1, 2}`
* `L383` — init sets `expectedResolution = type(uint64).max`
* `L411-412` — `INIT_BOND` deposited into `DELAYED_WETH`
* `L458` — `resolve()` reverts `NotEnoughProofs`
* `L534` — challenge sets `expectedResolution = now + SLOW`
* `L548` — `nullify()` — no caller modifier
* `L569` — nullify-when-countered requires `ProofType.ZK`
* `L594-598` — ZK-nullify branch
* `L613-617` — `claimCredit` two-branch gate
* `L614` — `GameNotResolved` revert
* `L788-798` — `_proofRefutedUpdate`
* `L800-823` — `_increaseExpectedResolution` + `_getDelay`
* `L938-950` — `_getParentGameStatus` (`L949` unconditional `DEFENDER_WINS` for first games)

#### `Verifier.sol`

* `L13` — “used to prevent further proof verification after a soundness issue is found”
* `L39-47` — `nullify()` access gate

### PoC files

The complete Foundry test + harnesses are available at:

<https://gist.github.com/calebtkjordan/df10023d5df1a7760cedc5ff3e7eae59>

Files in the Gist:

* `BaseAzul_BondLockOnZKNullify.t.sol` — the PoC (4 tests)
* `AnchorStateRegistryHarness.sol` — plumbing harness (ASR + DGF mocks)
* `DelayedWETHHarness.sol` — plumbing harness (DelayedWETH mock)
* `foundry.toml` — confirms the `solc=0.8.15 / evm_version=london` settings the PoC requires

### How to run

Create a Foundry project with the layout below and the attached `foundry.toml`.

Vendor minimal Solady (`utils/Clone`, `LibClone`, `FixedPointMathLib`, `ReentrancyGuard`, `EnumerableSetLib`) under `lib/solady/`, and stub the in-scope dispute interfaces under `interfaces/dispute/` and `interfaces/multiproof/`.

Symlink the in-scope `AggregateVerifier.sol`, `Verifier.sol`, `MockVerifier.sol`, `MockSystemConfig.sol` under `srcs/`.

Run:

```bash
forge test --match-path test/baseazul/BaseAzul_BondLockOnZKNullify.t.sol -vv
```

Expected: `4/4` passing. The lock demo is `test_ZK_nullification_locks_bond_at_threshold_2`.

### Gist filename → on-disk layout

* `test/baseazul/BaseAzul_BondLockOnZKNullify.t.sol`
* `test/harness/AnchorStateRegistryHarness.sol`
* `test/harness/DelayedWETHHarness.sol`
* `foundry.toml`
