# \#97958 [SC-High] Residual pending recapitalization is reserved against cash that has already left the pool, so LP epoch settlement reverts with no-progress indefinitely and no permissionless action can clear it

**Submitted on Nov 4th 2026 at 03:48:28 UTC by @risinghorse for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97958
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief / Intro

When a terminal deficit is recovered, the owner calls `recapitalizePool`, which records the injected cash as a `pendingRecapitalizationAssets` bucket that the next LP settlement is meant to apply. The SC-L1 fix caps how much of that bucket a single settlement may apply to the amount currently *distributable*, and reserves the rest from free cash. But `pendingRecapitalizationAssets` records cash that arrived at recap time, and nothing decrements it when that same cash later leaves the pool — for example when the trader claim the recap was meant to service is paid. Once the residual bucket exceeds distributable value, free cash is computed as `… − pendingBucket` and reaches zero even though the pool still holds cash. The Senior redemption budget is then zero, the settlement makes no progress, and it reverts `HousePool__NoLpEpochProgress` — which rolls back the very reconcile that would have burned the residual bucket down. Because the only function that decrements the bucket (`reconcile`) has no production caller other than that reverting settlement path, LP redemptions are frozen until the owner injects *more* capital or waits out repeated 72-hour rate-change timelocks.

## Vulnerability Details

**The residual is reserved against nothing.** `recapitalizePool` adds the full injected amount to `pendingRecapitalizationAssets` ([`HousePool.sol:1799`](../../../src/HousePool.sol); queued from [`CfdEngine.sol:632-653`](../../../src/CfdEngine.sol)). A reconcile applies only the *settleable* part, capped at distributable value by `_settleablePendingClaimantBuckets` ([`HousePool.sol:1829`](../../../src/HousePool.sol)), and decrements the bucket by exactly that part at `:1790`. The remainder stays in `pendingRecapitalizationAssets`. Free cash for redemptions subtracts the whole pending bucket:

```solidity
// HousePool.settleLpEpoch (:842-843)
uint256 freeAssets =
    _buildWithdrawalSnapshot(accountingSnapshot, unassignedAssets, _pendingClaimantBucketAssets()).freeAssets;
```

**Nothing reduces the bucket when the cash it represents leaves.** The recap is injected to service a trader claim. When `settleTraderClaim` pays that claim, pool cash `P` and claims `C` both fall, but `pendingRecapitalizationAssets` is unchanged — it still records the original injection. So once the bucket exceeds distributable `D`, the excess `R = pending − D` is reserved against cash that is already gone. On a flat book this bites whenever `pending ≥ 2·cash`: free cash saturates to 0.

**The no-progress revert rolls back the only fix.** With free cash 0, the Senior redemption budget `_min(freeAssets, seniorPrincipal)` is 0 ([`HousePool.sol:848`](../../../src/HousePool.sol)); `_fundRedemptionPhase` funds nothing, the Junior phase (gated behind the Senior backlog, `:878-882`) also funds nothing, and `_emitLpEpochSettled` reverts `HousePool__NoLpEpochProgress` ([`HousePool.sol:1184`](../../../src/HousePool.sol)). That revert undoes the reconcile that ran earlier in the same call — the one place the bucket is decremented. The decrement therefore never persists.

**No permissionless path can clear it.** `reconcile()` is `onlyVault` ([`HousePool.sol:1471`](../../../src/HousePool.sol)) and `TrancheVault` never calls it (grep of `src` finds no production caller), so the only way to persist a reconcile is `settleLpEpoch` — which reverts. The pending bucket also gates LP entry (`:1108`), so deposits are refused too. Matured redemption requests cannot be cancelled. The deadlock is stable.

**The sequence the PoC performs (no attacker; the documented recovery, `test_A`):**
1. A terminal deficit is reached by a ~100% basket round trip (1.00 → 1.50 → ~0.20): a winner's realized payout exceeds pool cash, creating a trader claim `C > P`. Degraded mode latches.
2. A Senior LP has a matured redemption queued.
3. The owner follows the documented terminal-deficit runbook (SECURITY.md:1499-1512): `recapitalizePool(deficit + small surplus)`, the claimant calls `settleTraderClaim` (paid, `P` and `C` fall together), `clearDegradedMode`.
4. The pool now holds a small surplus, `C = 0`, `L = 0`, but `pendingRecapitalizationAssets` still records the full injection. Free cash is 0.
5. `settleLpEpoch` reverts `HousePool__NoLpEpochProgress` on every call thereafter; the Senior LP receives nothing.

A control (`test_D2`) confirms the ordering dependence: if a small Senior redemption matures *between* the recap and the loss, the reconcile persists and there is no deadlock. A revenue-bucket variant with no owner recap at all is in `leftover/RevenueBucket.t.sol`.

## Impact Details

All figures from `EF1Strict.t.sol` on the strict release fixture, USDG.

**Claim path (`test_A`):** terminal deficit 6,940.96; owner recapitalizes 10,411.44 (deficit + 3,470.48 surplus); the claim is paid and the latch cleared. Pool cash is then 3,470.48, `pendingRecapitalizationAssets` is 10,411.44, free assets 0.
- **720 hourly `settleLpEpoch` attempts over 30 days all revert** `HousePool__NoLpEpochProgress` (720 keeper attempts, 0 successes; 194 of them in frozen-market hours). The Senior LP's queued redemption (≈ 8,472 at full value) funds 0 the entire time.
- Frozen value 3,470.48, against 10,411.44 of reserved-but-spent recap.
- **Owner-only exits:** a further recap above `pending − 2·cash` (here frozen value + 1 USDG) clears it and the Senior redemption funds within 1 hour; or donation + `accountExcess` at half the cost (1,735.24); or **zero-capital Senior-rate-change finalizations**, which take multiple 72-hour timelock cycles — `test_I` shows one cycle at 95 hours still not progressing, so ≥ 144 hours in general.
- A non-owner cure exists but requires genuine adverse trading: `test_E` shows a trader loss of 1,735.24 to the pool restores progress — not something an LP can rely on.

The CLAIMS-2 form of the same bug (terminal deficit 59,738.23, pending 89,738.23, free 0; the same cash routed through `accountExcess` in the control funds 29,999.92 immediately) scales the frozen amount with the deficit.

**Severity, honestly:**
- **Literal High:** LP redemption settlement is frozen for well over 24 hours (measured 30 days with no owner action; the zero-capital exit is ≥ 144 hours), and the program assumes no response/detection deadline.
- **Realistic Medium:** the owner can end it in one transaction by injecting a little more capital (or half via donation + `accountExcess`), and the frozen value is the owner's own recovery remainder plus later revenue — LP principal was already wiped by the deficit, so no pre-existing user principal is trapped. A reviewer who credits the owner's one-transaction cure lands at Medium (or at "unable to operate due to lack of token funds," though the cash is present).
- No theft and no permanent freeze: a 72-hour rate change always eventually clears it.

## Eligibility (important)

This is the SC Audit Studio pre-audit finding **SC-L1** (marked **Fixed**) turned against itself. SC-L1's fix capped settleable pending buckets at distributable value and reserves the remainder "to be applied later once it's backed." This report shows the fix **introduces a new bug**: once the recap cash has left the pool (paid to the claimant it was injected for), the residual can never become "backed," and the no-progress revert prevents the one reconcile that would retire it. Under the program's rule — "a report on the same root cause is valid only if it shows that the fix … is incomplete or introduces a new bug" — it is eligible on that basis. It is not in the named SECURITY.md known-limitation sections.

## References

- `packages/perps/src/HousePool.sol:1799` (recap increments the bucket), `:1790` (only decrement, inside `reconcile`), `:842-843` (free cash subtracts the pending bucket), `:848`/`:878-882` (Senior/Junior budgets), `:1184` (`HousePool__NoLpEpochProgress` rolls back the reconcile), `:1471` (`reconcile` is `onlyVault`, no production caller), `:1108` (pending bucket blocks LP entry), `:1829` (`_settleablePendingClaimantBuckets` caps to distributable).
- `packages/perps/src/CfdEngine.sol:632-653` — `recapitalizePool` queues the bucket; `settleTraderClaim` later lowers pool cash without touching the bucket.
- Documented runbook followed: SECURITY.md "Explicit terminal-deficit cascade" (1499-1512).
- Origin: SC Audit Studio pre-audit **SC-L1** (Fixed) — this report shows the fix introduces the deadlock.

## Proof of Concept

## Proof of Concept

Runs in the repository's Foundry suite at the in-scope commit, no in-scope contract modified; only Pyth and USDG are mocked. All steps use public entrypoints and EOA signers; the owner calls are the documented `recapitalizePool`, `clearDegradedMode`, `accountExcess` and the timelocked pool-config change. No `vm.store`, `vm.etch`, `mockCall`, or prank as router/engine/pool/vault.

**File:** `packages/perps/test/audit_poc/recap_wedge/EF1Strict.t.sol` (with the shared release fixture it imports). 10/10 tests pass. A no-recap revenue-bucket variant is in `packages/perps/test/audit_poc/leftover/RevenueBucket.t.sol`.

**Run:**
```bash
cd packages/perps && "$HOME/.foundry/bin/forge" test --match-path "test/audit_poc/recap_wedge/EF1Strict.t.sol" --skip "test/perps/**/*.t.sol" --skip "script/**" -vv
```

**Key tests and what they show:**
- `test_A_ClaimPath` — the deadlock: after the documented recovery, 720/720 hourly settlements over 30 days revert `NoLpEpochProgress`; the Senior LP receives 0; cures priced (recap, donation+accountExcess, rate cycles).
- `test_D2_PlainRecap_ReconcilePersisted` — control: a Senior redemption maturing before the loss persists the reconcile and avoids the deadlock (ordering dependence).
- `test_I_RateChangeCycles` — the zero-capital exit takes multiple 72-hour cycles (≥ 144 hours).
- `test_E_OrganicCure` — only a genuine trader loss to the pool restores progress without the owner.
- `test_G_JuniorRefundCyclesWithNoSeniorHead`, `test_H_SeniorHeadBlocksJuniorCycles` — a matured Senior head is what makes it stick; with none, worthless Junior refunds eventually clear it.

### Suggested remediation (directional)

Decrement `pendingRecapitalizationAssets` when the cash it represents leaves the pool (e.g. when the serviced trader claim is paid), or write the residual bucket down to distributable value at reconcile instead of carrying an unbacked remainder, or let a reconcile that consumes a pending bucket persist even when the redemption phases make no funding progress — so a spent recapitalization cannot reserve cash that is no longer there.
