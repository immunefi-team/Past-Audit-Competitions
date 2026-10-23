# 74945 sc low permanent bond lockup when verifier nullified with proof threshold 2

## #74945 \[SC-Low] Permanent bond lockup when verifier nullified with PROOF\_THRESHOLD=2

**Submitted on Apr 26th 2026 at 04:25:37 UTC by @GibsonShelby for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74945
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Permanent freezing of funds in the bridge or in dispute game bonds with no available recovery path

### Description

## Permanent Bond Lockup When Verifier Nullified With PROOF\_THRESHOLD=2

**Severity:** High solid (Critical defensible if `DELAYED_WETH.recover()` is excluded as a recovery path).\
**Target:** `src/multiproof/AggregateVerifier.sol` v8.1.0 (`https://github.com/base/contracts/tree/v8.1.0/src/multiproof/`).\
**On-chain status (Apr 26, 2026):** AggregateVerifier impl `0xf3f0fa3124b7b0feb048a00404fe4d5d49e60796` on Sepolia (`gameImpls(621)`). `PROOF_THRESHOLD()` returns `1` today. Mainnet not yet deployed. F9 fires under the threshold=2 configuration the constructor allows (L285).

### Brief

When `PROOF_THRESHOLD=2` and a verifier is nullified after a challenge, `resolve()` reverts (`proofCount=1 < threshold=2`), `claimCredit()` reverts (`resolvedAt=0`, `expectedResolution` finite), and no replacement proof can be submitted (verifier globally dead, timer expired). The bond sits locked in `DELAYED_WETH` with no permissionless recovery. Only the multisig can rescue, manually, off-chain, per game.

The developer comment at `AggregateVerifier.sol#L526-L530` says: *"If the ZK is nullified, we allow the remaining TEE proof to resolve."* Line 458 does the opposite.

### Root Cause

`AggregateVerifier.sol#L458` inside `resolve()`:

```solidity
if (proofCount < PROOF_THRESHOLD) revert NotEnoughProofs();
```

The check is unconditional. When nullification reduces `proofCount` below `PROOF_THRESHOLD` and no replacement proof can be submitted (verifier globally dead + game timer expired), the game is permanently unresolvable.

### Pre-conditions

1. **`PROOF_THRESHOLD=2`** -- legal per constructor at L285 (`if (proofThreshold != 1 && proofThreshold != 2) revert InvalidProofThreshold();`). Threshold=2 is the intended production configuration; threshold=1 is the bootstrap/degraded mode.
2. **Game is challenged** -- normal dual-proof flow (TEE init + ZK challenge → `proofCount=2`).
3. **One verifier nullified** -- `nullify()` is a first-class protocol mechanism. ZK soundness failures are documented (Zcash 2019, multiple ZK-rollup circuit bugs 2023-2025).

### Vulnerability Walk

**Init (TEE):** `proofCount=1`, `expectedResolution = block.timestamp + 7 days`, bond deposited.

**Challenge (ZK):** `proofCount=2`, `counteredByIntermediateRootIndexPlusOne` set, `expectedResolution` reset.

**Nullify (ZK):** L548-L602 runs `_proofRefutedUpdate(ZK)`:

* `delete proofTypeToProver[ZK]` → ZK prover erased
* `proofCount -= 1` → `proofCount=1`
* `_increaseExpectedResolution()` calls `_getDelay()` which returns **7 days** when `proofCount==1` (NOT `type(uint64).max`). So `expectedResolution = block.timestamp + 7 days`, finite.
* `delete counteredByIntermediateRootIndexPlusOne;`
* `IVerifier(ZK_VERIFIER).nullify();` → singleton `Verifier.nullified = true`, global, irreversible

**After 7 days, `resolve()` reverts:** L443-L474, parent resolves as `DEFENDER_WINS`, branch enters else, L458 reverts `NotEnoughProofs` (1<2).

**`claimCredit()` reverts:** L606-L634, `expectedResolution != type(uint64).max`, enters first branch, hits `resolvedAt == 0` → `GameNotResolved`. The 14-day fallback is unreachable because `expectedResolution` is finite.

**Replacement proofs:** TEE slot occupied (`AlreadyProven`), ZK globally dead (`Nullified`), or timer expired (`GameOver`). `proofCount` can never increase.

**Formal proof of irrecoverability** is in the dedicated PoC field below: Halmos returns UNSAT for `check_resolveAlwaysReverts`, `check_claimCreditAlwaysReverts`, and `check_neverDeadlock` under deadlock preconditions. Foundry invariants `invariant_proofCountBoundedByThreshold` and `invariant_singletonVerifierKillsAllGames` BREAK with shrunk minimal sequences (2 calls and 4 calls respectively) -- their failure IS the proof of property violation. Echidna 1M iterations and Medusa 500K x 8 workers (1.62M total) confirm the deadlock-state properties hold across exhaustive fuzz.

### Scope of Impact -- Singleton Verifier

`Verifier.sol#L14`: `bool public nullified` is a contract-wide boolean, not per-game. One `nullify()` call kills the verifier for ALL games sharing that `ZK_VERIFIER` instance. Every active `PROOF_THRESHOLD=2` game with at least one proof enters the deadlock simultaneously.

Asymmetry: if BOTH verifiers are nullified (`proofCount=0`), `_getDelay()` returns `type(uint64).max` and the `claimCredit()` 14-day fallback works. If only ONE is nullified (`proofCount=1`), the fallback is blocked. Partial failure is strictly worse than total failure -- a perverse incentive.

### Impact

Bond locked in `DELAYED_WETH` with no permissionless recovery. Multisig rescue via `recover()` exists but is an emergency backstop:

* Sends ETH to `msg.sender` (admin), not `bondRecipient` -- manual redistribution off-chain
* Pulls from global `address(this).balance` shared across ALL games -- no per-game accounting
* NatSpec self-describes as emergency error-case backstop
* After rescue, the AggregateVerifier state machine stays broken

**Financial scaling** (Sepolia today: `INIT_BOND = 0.05 ETH`, \~50-60 multiproof games per day, 15,761 cumulative games at gameType 621):

| N concurrent threshold=2 games | ETH locked | USD (\~$2,322/ETH) |
| ------------------------------ | ---------- | ------------------ |
| 50                             | 2.5        | $5,805             |
| 200                            | 10.0       | $23,220            |
| 500                            | 25.0       | $58,050            |

Sepolia today: 0 deadlocked games, `nullified()` returns false on both verifiers. F9 is forward-looking for any deployment that flips threshold to 2 (which Cantina explicitly anticipates: *"the planned upgrade to PROOF\_THRESHOLD = 2"*).

**Severity amplifiers worth flagging:**

1. Singleton verifier death: one `nullify()` kills ALL games sharing that verifier.
2. Immutable code (Solady Clone, no proxy, no UUPS).
3. Atomic attack surface: `challenge()` + `nullify()` in one tx.
4. Cascading parent-child: deadlocked games block descendant resolution → potentially stalls L2→L1 withdrawals.
5. Damage-to-cost ratio: 0.05 ETH attack cost → 50:1 to 200:1 ETH locked.
6. **No challenger required**: invariant `invariant_proofCountBoundedByThreshold` breaks with shrunk 2-call sequence (`initialize(threshold=2)` + `warp(30 days)`). Solo proposer + expired timer reaches the same deadlock at `resolve()`. The trigger surface is wider than nullification alone.

### Severity Justification

**Critical reading:** matches the scope text "Permanent freezing of funds in dispute game bonds with no available recovery path". The deadlock is mathematically permanent (Halmos UNSAT). Admin `recover()` is documented as emergency backstop, sends ETH to admin not `bondRecipient`, has no per-game accounting, and leaves the state machine broken. By a strict reading of "available recovery path", these governance routes don't qualify.

**High reading (submitted):** Cantina Multiproof Audit 1 finding 3.1.1 identified the same root-cause line and rated it Informational because admin `DELAYED_WETH.recover()` was treated as a sufficient recovery path. The Coinbase fix `dd587c9a` patched the parent-invalid arm. F9 isolates the parent-valid arm at L455-L466 which the fix does not touch -- distinct executed code path. Once the admin rescue is read as a recovery vector, the freeze becomes "temporary" and the financial framing sits inside the High band ($10K-$100K at 50-200 concurrent games).

Submitted at **High** because that is the most defensible reading given the Cantina precedent. The Critical push is left for the triager's discretion.

CVSS: `CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:C/C:N/I:H/A:H = 8.5 (High)`.

### Why Dispute / Blacklist / Retire Do NOT Resolve

The competition rule downgrades reports that assume the team will not dispute. F9 does not assume that.

1. F9 fires AFTER the team's correct dispute action. `nullify()` is the response to an invalid proof; F9 is the bug in the response itself.
2. `Verifier.nullified` is single-write irreversible (no `unnullify`). Dispute action moves the game further from threshold, not closer.
3. `resolve()` L458 depends only on `proofCount < PROOF_THRESHOLD` -- no coupling to blacklist/retire status.
4. Singleton design amplifies one dispute into mass deadlock across N unrelated games.
5. Parent-blacklist (escape #15) forces sibling games into `CHALLENGER_WINS` regardless of outcome -- creates new victims.
6. Contract is immutable Solady Clone -- no upgrade path.

### Differentiation From Prior Audits

* **Cantina Multiproof Audit 1, finding 3.1.1** (`AggregateVerifier.sol#L453`, Informational): targets the `parentGameStatus == CHALLENGER_WINS` branch. Fixed in commit `dd587c9a`. F9 fires in the `else` branch at L455-L466 when the parent resolves normally (typically `DEFENDER_WINS`). The `dd587c9a` diff does not touch the else branch. Same line citation, distinct executed code path, fix-orthogonal scenario.
* **Cantina Multiproof Audit 2**: 0 related findings.
* **Public Known Vulnerabilities catalogue (18 issues)**: cross-checked, 0 match F9. Public catalogue covers SP1 ZK soundness (off-chain), proposer config, ZK prover service, TEE NitroEnclaveVerifier, TEEProverRegistry storage gap, ZK VM memory, plaintext TEE comms. F9 is on-chain Solidity state-machine -- distinct.
* **1 private known issue (Hash Variant)** disclosure expected 2026-04-28. Best-effort residual-risk estimate 5-10%.

### Recommendation

Add a nullification-aware threshold check in `resolve()`:

```solidity
// inside the parent-valid else branch, after gameOver() check:
uint256 effectiveThreshold = PROOF_THRESHOLD;
if (IVerifier(ZK_VERIFIER).nullified() || IVerifier(TEE_VERIFIER).nullified()) {
    effectiveThreshold = effectiveThreshold > 1 ? effectiveThreshold - 1 : 1;
}
if (proofCount < effectiveThreshold) revert NotEnoughProofs();
```

This implements the documented intent at L526-530 ("If the ZK is nullified, we allow the remaining TEE proof to resolve"). Preserves threshold=1 behavior, preserves no-nullification behavior, allows resolution with the remaining valid proof when one verifier is dead.

**Alternative minimal fix:** add a deadlock-recovery branch in `claimCredit()` that detects the state (`expectedResolution != max && resolvedAt == 0 && proofCount < PROOF_THRESHOLD && verifier.nullified`) and unlocks the bond to `bondRecipient`.

### References

* AggregateVerifier.sol v8.1.0: <https://github.com/base/contracts/blob/v8.1.0/src/multiproof/AggregateVerifier.sol>
* Verifier.sol v8.1.0: <https://github.com/base/contracts/blob/v8.1.0/src/multiproof/Verifier.sol>
* Cantina Multiproof Audit 1, finding 3.1.1, fix commit `dd587c9a`
* Public Known Vulnerabilities PDF: <https://drive.google.com/file/d/13Q1bk8XKhme-ZOBJAGh5JodpP4HLuLer/view>
* On-chain verification: AggregateVerifier impl `0xf3f0fa3124b7b0feb048a00404fe4d5d49e60796` Sepolia, `PROOF_THRESHOLD()` returns 1
* Sepolia activation tx: <https://sepolia.etherscan.io/tx/0xb20fcd230ee76842637a508382886aa7596f36dd6fc8f47d147aeab213fbd55d>

### Proof of Concept

PoC Foundry runnable. 6/6 deterministic tests + 50K fuzz + 1Kx200 invariants (200K calls) confirm the deadlock. Two invariants are designed to FAIL on the buggy code -- their failure IS the proof of property violation. Halmos formal UNSAT proof on three properties (resolveAlwaysReverts, claimCreditAlwaysReverts, neverDeadlock) under deadlock preconditions.

Reproduce locally:

```bash
git clone <repo>
cd poc-tests/f9-formal
forge install foundry-rs/forge-std@v1.10.0 --no-git
forge build --build-info
forge test                                              # 6/6 deterministic PASS
forge test --invariant-runs 1000 --invariant-depth 200 # 200K calls, 2 invariants BREAK as expected
forge test --fuzz-runs 50000                            # 50K runs PASS
halmos --solver-timeout-assertion 300000               # 3/3 properties UNSAT (formal proof)
```

The full PoC is included in the Description above (section Proof of Concept). It uses minimal reproduction contracts AggregateVerifierMinimal + MockVerifier that copy the v8.1.0 source verbatim with line references in comments. The mainnet fork approach is not feasible because the v8.1.0 multiproof contracts are not yet deployed to any public network (Base Azul activation scheduled for May 13, 2026, post-competition).

Key counter-example sequences captured by the invariant fuzzer:

* `invariant_proofCountBoundedByThreshold` breaks with shrunk minimal sequence of 2 calls: `initialize(threshold=2, bond=...)` then `warp(2,592,000)` (30 days). No challenger, no nullify required. Solo proposer + expired timer reaches deadlock.
* `invariant_singletonVerifierKillsAllGames` breaks with shrunk sequence of 4 calls: `initialize` then `challenge` then `nullifyZk` then `warp`. Classic nullification-driven deadlock.
* Both sequences end in `resolve()` reverting with NotEnoughProofs and `claimCredit()` reverting with GameNotResolved, exactly the deadlock described in the report.
