# \#100085 [SC-Insight] A matured Junior LP deposit can neither activate nor be cancelled during degraded mode or an oracle freeze, temporarily freezing segregated escrow

**Submitted on Nov 16th 2026 at 08:01:30 UTC by @RichJamo for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100085
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

A Junior LP becomes an LP in two steps: a deposit request, cancellable up to maturity, then activation into shares at the next hourly LP settlement. Normally maturity and activation happen in the same settlement. In degraded mode (the pool's insolvency circuit breaker) and under an oracle freeze (every weekend), activation is blocked — and a matured deposit cannot be cancelled either, because the cancel gate has no unlock for those states. The depositor's escrowed USDG is frozen with no exit for the whole blocked window: unbounded in degraded mode, and ~48-72 hours every weekend. The escrow sits segregated from pool backing, so returning it would reduce no one's solvency; the trap is a missing exit, not a protection.

## Vulnerability Details

Activation is blocked whenever `_entryStatusBlocked` is true (`HousePool.sol:1139-1145`):

```solidity
return statusSnapshot.oracleFrozen || statusSnapshot.degradedMode
    || HousePoolAccountingLib.buildReconcileSnapshot(accountingSnapshot).deficit != 0;
```

In degraded mode the LP settlement additionally hard-reverts (`HousePool.sol:832`, `:1864-1870`); under an oracle freeze redemptions still fund and only deposit activation defers. Either way a matured deposit does not become shares.

The matured-deposit cancel gate has no unlock for these states (`TrancheVault.sol:851-860`):

```solidity
if (
    (IS_SENIOR || currentLpEpoch() >= requestId) && !queueState.rejected
        && (!IS_SENIOR || !POOL.lpEpochSettlementPaused()) && !_projectedTerminalWipe()
        && !POOL.isSeniorImpairedAfterPendingDepositReconcile()
) {
    if (IS_SENIOR) {
        revert TrancheVault__SeniorDepositCommitted();
    }
    revert TrancheVault__DepositEpochAlreadyActive();
}
```

The only unlocks are rejection, a projected terminal wipe (needs `juniorPricingAssets == 0`), Senior impairment, or a Senior-only settlement pause. With Junior NAV positive and Senior intact, none fires, so a matured Junior deposit in degraded mode or a freeze reverts `TrancheVault__DepositEpochAlreadyActive`. The escrow stays put: `pendingDepositEscrowAssets` is held in the vault outside pool `accountedAssets` (`TrancheVault.sol:133`, `HousePool.sol:426-428`), so returning it changes no LP's backing and lets the depositor escape no loss.

**Why this is not the intended design.** The gate already enables a Senior deposit's cancel precisely when settlement is paused — so the design grants an exit when activation is blocked; the degraded and frozen cases simply were not given the equivalent. The documents never state a matured deposit must stay trapped here: a settlement hold "permits controllers to recover pending escrow" (`ACCOUNTING_SPEC.md:440`), emergency containment keeps "eligible cancellations operating" and stops "only settlement broadcasts" (`SECURITY.md:1471`), and the one line that sends a matured deposit to "later activation after recovery" (`SECURITY.md:493`) is about the guardian pause, not degraded insolvency. The documented reason for deferring activation is pricing — "symmetric entry pricing requires the live terminal-NAV gate" (`ACCOUNTING_SPEC.md:554`) — which bites activation, not a cancel that simply returns the escrow at no price.

## Impact Details

Temporary freezing of user funds. The frozen instance is routine and bounded: a Junior deposit placed before the Friday close matures into the weekend and can neither activate nor cancel until the market reopens, ~48-72 hours, every weekend — comfortably past the 24-hour High threshold. The degraded instance is unbounded: it clears only on owner recapitalisation (or enough trader closes) plus a reconcile, `clearDegradedMode`, and an owner `assignUnassignedAssets`, so the depositor cannot self-rescue. A depositor who requested entry while the pool was healthy is force-held, at the moment the pool is flagged insolvent, into the first-loss Junior tranche with no way to withdraw escrow that bears none of the pool's risk. A cohort of matured depositors is caught, not a single account, and the trigger is the market calendar or ordinary trading, not the victim.

## Recommendation

Make the cancel gate track the activation-blocked condition instead of a hand-picked unlock list: allow `cancelPendingDeposit` (and `cancelRedeemRequest`) for a matured request whenever `_entryStatusBlocked` is true (degraded / oracle-frozen / deficit), mirroring the existing `!lpEpochSettlementPaused()` unlock, and return the segregated escrow. This keeps every protection of degraded mode and the freeze — entry stays blocked, settlement stays frozen, no LP can activate or run — and drains only escrow that was never at risk; it touches no pool accounting.

## References

- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L851-L860
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1139-L1145
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L426-L428
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L493
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L440


## Link to Proof of Concept

https://gist.github.com/RichJamo/fbdf0eb3e6b984c0358cfbcb1364615d

## Proof of Concept

## Proof of Concept

Two passing tests, one per blocking state, against a clone of the audit repository at commit
`a82d4fefb02768e5b3bddcc97dee371eee24fea8`. No in-scope contract is modified.

Files in the gist: `FrozenDepositTrap.t.sol`, `DegradedDepositTrap.t.sol`, `README.md`.

Gist: https://gist.github.com/RichJamo/fbdf0eb3e6b984c0358cfbcb1364615d

Setup:

```
cp FrozenDepositTrap.t.sol DegradedDepositTrap.t.sol \
   <repo>/packages/perps/test/perps/accounting/waterfall/
cd <repo>/packages/perps
forge test --match-path "test/perps/accounting/waterfall/*DepositTrap.t.sol" -vv
```

**`test_MaturedJuniorDepositTrappedWhileOracleFrozen` (oracle freeze).** A Junior LP requests a 500,000 USDG
deposit while the market is open, solvent and non-degraded. About one hour later the real `MarketCalendarLib`
reports `isOracleFrozen() == true` and `degradedMode() == false`. A live `settleLpEpoch` does not revert — it
funds a maturing redemption in the same call — but the Junior deposit does not activate (claimable 0, the full
500,000 still escrowed, depositor shares 0). `cancelPendingDeposit` reverts
`TrancheVault__DepositEpochAlreadyActive`. Warping past the freeze to a weekday, the same deposit finalises and
mints shares — isolating `oracleFrozen` as the sole cause.

**`test_MaturedJuniorDepositTrappedInDegradedMode` (degraded mode).** The pool is driven into degraded mode with
Junior principal above zero and Senior at or above its high-water mark. The matured deposit cannot activate — a
live `settleLpEpoch` reverts `HousePool__DegradedMode` — and cannot cancel. Only after recapitalisation, a
reconcile, `clearDegradedMode`, and an owner `assignUnassignedAssets` does the same deposit finalise and the
depositor claim shares, confirming the freeze is temporary and owner-gated, not something the depositor can end.

Both tests pass.

