# \#99219 [SC-Critical] A missing guard on the Junior assignment path mints shares at a near-zero price and dilutes existing Junior LPs out of their recovery

**Submitted on Nov 12th 2026 at 12:55:07 UTC by @RichJamo for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99219
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

`HousePool.assignUnassignedAssets` assigns stranded "unassigned" (ownerless) pool value into a tranche and mints matching shares. **The two tranche paths are not symmetric: the Senior branch is capacity-checked and reverts when the pool is wiped; the Junior branch has no such check.** So when a Junior tranche has been driven to zero principal by a *reversible* mark dip — while its LP shares are still outstanding — an assignment into it prices the mint against ~zero principal and mints the named receiver a share balance that dwarfs every existing holder. When the dip reverses and Junior principal is restored, that recovered value accrues to the receiver's shares and the existing Junior LPs are diluted to dust. The defect is a **missing guard on the Junior path**: the protocol fails to protect existing Junior claimant backing from a bootstrap mint — exactly the protection the Senior path gets. (The team's spec signals the same intent — `SECURITY.md:245` says these powers "do not permit withdrawal of existing claimant backing" — but the load-bearing point is the code's own Senior/Junior asymmetry, not that sentence.)

(Note on prior art: the Cecuro AI audit flagged a *different* bug in this same function — an unset `seedReceiver` deadlocking the refund-dust sweep. That is a lock on the refund path; this is a mispriced mint that transfers value. Different root cause, different impact.)

## Vulnerability Details

**Affected code** (`packages/perps/src`, commit `a82d4fef`): `HousePool.assignUnassignedAssets`, `TrancheVault.quoteBootstrapDeposit` / `bootstrapMint`.

**Root cause — the Junior assignment path has no capacity guard and mints at a wiped (~zero) price.** In `assignUnassignedAssets` (`HousePool.sol:660-704`), the deposit-capacity guard is applied only on the Senior branch:

```solidity
if (toSenior) {
    _requireSeniorDepositCapacity(amount, _getWaterfallState());   // Senior only
}
...
uint256 shares = ITrancheVaultBootstrap(targetVault).quoteBootstrapDeposit(amount);
...
} else {
    juniorPrincipal += amount;                                     // Junior branch: no guard
}
unassignedAssets = 0;
ITrancheVaultBootstrap(targetVault).bootstrapMint(shares, receiver);
```

The bootstrap quote is `assets * (accruedTotalSupply + VIRTUAL_SHARES) / (pricingAssets + 1)` (`TrancheVault.sol:1858`, `VIRTUAL_SHARES = 1000`). When the Junior tranche is wiped, `pricingAssets ≈ 0`, so the denominator is ~1 and the mint is `assets * (supply + 1000)` — compounded by 6-decimal assets against 18-decimal supply, the receiver's share balance exceeds every existing holder's by many orders of magnitude, so the receiver ends up owning essentially the whole tranche.

**Why the Senior path is safe and the Junior path is not — the asymmetry is the core of the finding.** The Senior guard is a hard revert, not a share adjustment: `_requireSeniorDepositCapacity` reverts `HousePool__ExceedsSeniorDepositCapacity` when the amount exceeds Senior deposit capacity (`HousePool.sol:677`). A wiped pool has zero Senior capacity — losses hit Junior first, and Senior is capped as a ratio of Junior — so a wiped-Senior assignment reverts *before any mint*. The Junior branch has no capacity constraint at all (Junior is first-loss and uncapped), so nothing stops the mint. The `VIRTUAL_SHARES` cushion (`+1000`) does not help: it defends the empty-vault first-depositor case (supply 0), but here the existing supply is non-zero, so `+1000` is negligible against `amount × supply`. On both paths the mint formula would dilute; the Senior path is saved only by the capacity revert, which the Junior path lacks. So this is a **missing enforcement in the code**, demonstrated by the protocol's own handling of the sibling path — not a reliance on owner behaviour.

**The design solved the wrong fairness problem.** The natspec states the function mints to a named receiver to "[prevent] later LPs from implicitly capturing previously ownerless value" (`HousePool.sol:653-657`) — i.e. to stop existing holders getting an *implicit windfall* if ownerless value were simply added to principal. That rationale fits a genuinely dead tranche being re-seeded. It inverts on a *populated* tranche: the same "give it an explicit owner" step hands the receiver the existing holders' own recoverable principal. The function applies bootstrap logic without checking whether the tranche actually has outstanding holders to protect.

**How it plays out.** A reversible mark dip drives Junior principal to zero while its share supply remains (Junior is first-loss; a dip-timed reconcile absorbs the loss into Junior). A standing `unassignedAssets` bucket exists — e.g. the remainder of an owner recapitalisation, which strands there (`HousePoolPendingPreviewLib.sol:135`). During the dip, the owner calls `assignUnassignedAssets(toSenior=false, receiver)`; the Junior branch mints the receiver a supply-dwarfing share balance with no capacity check. When the mark recovers, the restored Junior principal accrues pro-rata to shares — nearly all to the receiver — and the original Junior LPs are left with dust. The receiver is owner-named; the harm falls on the existing Junior LPs regardless of who it is.

The oracle must be live: `assignUnassignedAssets` is blocked during `oracleFrozen` (`ACCOUNTING_SPEC.md:319`), so the window is a live-market (weekday) dip, not a weekend freeze.

**This is not an owner-timing error.** The harmful moment is not reliably distinguishable from the legitimate one: a genuinely-wiped tranche (safe to re-seed) and a transiently-wiped tranche with outstanding holders (harmful) both present on-chain as `juniorPrincipal == 0`, and nothing surfaces whether the wipe will reverse. And the protocol does not leave the Senior equivalent to owner care — it *enforces* a check in code. The defect is that the Junior path has no such enforcement, not that an owner mis-timed a call. These recovery operations are used during distress, so an assignment landing while the mark is dipped is crisis response overlapping the volatility that caused the crisis, not a careless choice.

## Impact Details

Existing Junior LPs' recoverable backing is permanently **diluted and misallocated** to the assignment receiver. In the PoC a Junior LP who deposited 2,000 USDG and recovers 11,466 USDG without the assignment instead ends worth **1 atom (0.000001 USDG)** when the owner assigns during the wipe; the receiver, minted ~7.7×10²² shares against the LP's ~2×10¹² from a 25,803 USDG assignment, captures the recovered Junior principal. The pool total is conserved — the value moves between holders — and the loss falls on the pre-existing Junior cohort. The magnitude scales with the recoverable value of the wiped tranche; the assignment can be small relative to the value it captures. It is a defect of the code's own making — the Junior assignment path lacks the guard the Senior path has — not an accepted reallocation. (`SECURITY.md:245` signals the same intent, that these powers must not erode existing claimant backing, though it uses the narrower word "withdrawal"; the Senior/Junior asymmetry in the code is the stronger evidence.)

## References

- `HousePool.sol:660-704` (`assignUnassignedAssets`; Senior-only capacity guard at `:677` — the revert the Junior branch `:698-702` lacks), `:653-657` (natspec: minting to a named receiver "prevents later LPs from implicitly capturing previously ownerless value"), `TrancheVault.sol:1858` (bootstrap share quote), `HousePoolPendingPreviewLib.sol:135` (recap remainder strands in unassigned).
- Supporting intent: `SECURITY.md:244-245` (the assign/bootstrap powers "do not permit withdrawal of existing claimant backing"); `ACCOUNTING_SPEC.md:326` (a wiped tranche cannot be silently revived), `:319` (assignment blocked while `oracleFrozen`).
- Distinct from the Cecuro `assignUnassignedAssets` finding (seed-receiver refund deadlock) and from our own `RECAP-RESIDUAL-1` / `UNASSIGNED-DIP-1` (reconcile bugs).

## Recommendation (choice / harm / fix)

The **choice** — the owner may assign genuinely ownerless value to bootstrap a tranche — is intended and useful. The **harm** is that the Junior path mints at a wiped (~zero) price with no guard, so an assignment into a tranche that still has outstanding holders dilutes them out of their recovery. The real flaw is that the function applies bootstrap logic without distinguishing a *dead* tranche (no real holders — safe to re-seed to a named receiver) from a *populated* one (outstanding holders to protect). The **fix that keeps the choice**: make the Junior path refuse or re-price an assignment that would dilute existing holders — mirror the Senior path, which already reverts when the pool is wiped, or price the bootstrap mint against the tranche's pre-wipe basis rather than the live (~0) pricing assets. Genuine bootstrap of a dead tranche (no outstanding holders) is untouched; a populated tranche's existing claimant backing is preserved.

## Link to Proof of Concept

https://gist.github.com/RichJamo/b565395cdf6527b11135646fffd597db

## Proof of Concept

This proof of concept runs against the real `HousePool`, `CfdEngine` and both `TrancheVault` contracts, with no pool state forced via `vm.store`. It is two tests — a **control** and a **variant** — identical except for one step: whether the owner assigns during the wipe.

**Setup (both tests):** an old Junior LP makes a real deposit; a real owner recapitalisation strands value in `unassignedAssets`; a real open LONG, a real reversible mark dip, and real reconciles drive Junior principal to zero while its LP shares remain outstanding.

**Control** — `test_Control_NoAssignment_OldHolderRecovers`: no assignment is made. When the mark recovers, the old Junior LP recovers **11,466 USDG**.

**Variant** — `test_Variant_AssignmentDuringWipe_DilutesOldHolderOutOfRecovery`: during the wipe, the owner assigns the 25,803 USDG `unassignedAssets` into the wiped Junior tranche. The Senior call reverts on its capacity guard; the Junior call has no such guard and mints the receiver **~7.7×10²² shares** against the old holder's **~2×10¹²**. When the mark recovers, the restored Junior principal accrues to the receiver, and the old holder's shares are left worth **1 atom (0.000001 USDG)** — their 11,466 USDG recovery is gone.

The two tests differ only by that single assignment, so it is the sole cause of the loss.

**Reproducibility:** passes from a fresh `git archive` export of the target at commit `a82d4fef` (pinned submodules) on Forge 1.5.1 with via-IR. The PoC file (`AssignWipedJuniorDilutesHolder.t.sol`) and its run log are in this gist: https://gist.github.com/RichJamo/b565395cdf6527b11135646fffd597db.

