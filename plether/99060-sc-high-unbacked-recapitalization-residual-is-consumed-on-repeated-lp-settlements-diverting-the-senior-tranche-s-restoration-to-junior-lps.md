# \#99060 [SC-High] Unbacked recapitalization residual is consumed on repeated LP settlements, diverting the Senior tranche's restoration to Junior LPs

**Submitted on Nov 11th 2026 at 17:38:21 UTC by @mhkarazeybek for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99060
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro
After an LP deficit, a documented recovery is `CfdEngine.recapitalizePool` followed by `clearDegradedMode`. The part of the recapitalization that pool equity does not yet back stays in a pending bucket. `HousePool._reconcile` caps that bucket against gross distributable equity, which already backs the Senior principal assigned by the previous reconcile. While the residual is at least that equity, each later LP-epoch settlement writes Senior principal off, rebuilds it from the bucket and lowers the Senior high-water mark, with no economic change in between. When value returns to the pool it is credited to Junior instead of restoring Senior, and a Junior LP can send those settlements himself. For queue progress they rely on zero-payout exit requests, an open Cecuro finding on the program's known-issues list ("MED-24 and MED-29 (LP refund epochs and redemption rounding)").

The bug is reachable only after the pool is in deficit and, in the sequence tested here, the owner has recapitalized by at most twice the deficit. In the PoC the deficit takes a price path of 1.00 → 0.50 → 1.90 on a book at 88.6% utilisation, an extreme path for an FX index. If the price then returns to 1.00 and the remaining position closes flat, a Senior LP who deposited 50,000 USDG receives 22,814.80 instead of 52,814.75, and a Junior LP who deposited 100,000 receives 109,622.43 instead of 46,831.07.

## Vulnerability Details
Commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src`.

`HousePool._reconcile` (`HousePool.sol:1555-1599`) computes the settleable part of the pending buckets by capping them against the whole distributable amount:

```solidity
// HousePool.sol:1829-1834, body of _settleablePendingClaimantBuckets
return HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, reconcileSnapshot.distributable);
```

`planReconcile` then removes that amount from distributable before it reprices the principal that is already claimed:

```solidity
// libraries/HousePoolReconcilePlanLib.sol:68-71
if (pendingBucketAssets > 0) {
    snapshot.distributable =
        snapshot.distributable > pendingBucketAssets ? snapshot.distributable - pendingBucketAssets : 0;
}
```

and `_applyPendingClaimantBucketsLive` (`HousePool.sol:1759-1776`) applies the same amount through `applyClaimantRecapitalizationIntent`, whose empty-bootstrap branch overwrites the high-water mark:

```solidity
// libraries/HousePoolPendingPreviewLib.sol:118-121
if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
    state.waterfall.seniorPrincipal += remaining;
    state.waterfall.seniorHighWaterMark =
        recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
```

Let the owner recapitalize `R` while pool equity is negative by `deficit`. After the first reconcile, distributable is `D = R - deficit`, Senior principal is `D`, the high-water mark is `R`, and the residual bucket is `r = deficit`. The repository asserts this state as intended (`test/perps/accounting/waterfall/HousePoolWaterfall.t.sol:201`: "Residual recap should preserve senior's recovery target"). The same test shows the defect without our fixtures (its own writes the trader-claim liability to storage): duplicating its final `pool.reconcile()` call (lines 214-215) above line 213, so that one reconcile runs with nothing changed, makes its next check fail with `Residual recap should settle once backing becomes available: 50040000000 != 100000000000`.

On the next reconcile nothing has changed, yet the residual is capped against the same `D`:

1. `min(r, D)` is subtracted from distributable, although that `D` is what backs Senior's principal.
2. The waterfall sees less equity than principal and charges the difference to Senior. If `r >= D`, Senior principal goes to zero.
3. The bucket is applied. Both principals are zero, so the empty-bootstrap branch runs again and sets the high-water mark to the residual as it stood before this step, which is lower than `R`.

Each repetition with `r >= D` lowers the high-water mark by `D` and consumes `D` of the bucket. The last one, with `r < D`, only drains what is left. The bucket ends empty without having been backed, and from then on the waterfall restores Senior only up to the lowered mark and sends the rest to Junior (and to unassigned assets for any part of the bucket still pending).

`settleLpEpoch` has no once-per-epoch or once-per-block guard. It needs LP queue progress, and one call processes at most 16 matured epochs per phase (`HousePool.sol:180`). `TrancheVault.requestRedeem` stays open while the pool is degraded, so exit requests filed during that time form a backlog that allows several settlements back to back.

The cap was added in commit `31a38bb7` (2026-05-08, "Cap pending claimant buckets before reconcile"), the fix for pre-audit finding SC-L1. The fix note says the unsettleable part "stays pending, stays reserved everywhere, and gets applied later once it’s backed". That does not hold when the residual is at least the equity already assigned. The behaviour also contradicts `ACCOUNTING_SPEC.md` waterfall rules 3 ("Losses do not reduce the Senior high-water mark") and 5 ("Reserving them in step 2 prevents counting the same inflow twice").

## Impact Details
The figures are USDG received in wallets after both LPs redeem, for the same trades, prices and recapitalization. In forms A and B the price returns to 1.00 in one update after 1.90 and the SHORT closes flat.

| Form | Senior LP: control → attack | Junior LP who sends the settlements: control → attack |
|---|---|---|
| A. Three extra settlements, three hours apart | 52,814.75 → 22,814.80 (−29,999.95) | 46,831.07 → 109,622.43 (+62,791.36) |
| B. One transaction, 33-epoch backlog | as A | 46,815.61 → 102,352.72 (+55,537.11) |
| B. One transaction, 49-epoch backlog | as A | 46,808.12 → 109,568.68 (+62,760.56) |
| C. SHORT takes its profit at 1.90; later trading revenue | 38,211.02 → 22,814.80 (−15,396.22) | 0 → 14,074.06 |

In form A the control pays the Senior LP 52,814.75 for a 50,000 deposit because the first settlement after recovery sets the high-water mark to `R`; the attack's 22,814.80 is 27,185.20 below the deposit. Against the control the Senior tranche loses 30,000.55 and the Junior tranche gains 67,815.39. The other 37,814.84 of Junior's gain is recapitalization cash left in `unassignedAssets` in the control. In form C the revenue is the losses of four further traders (about 22.8k); without it Senior stays at about 15,000 in both.

At the release Senior share limit a Junior exit is capped while Senior stays in, which these figures (both LPs exiting in one epoch) do not show: with only the attacker exiting in form A, the control pays 36,905.82 (78.8% of its shares; the rest unpaid in the 8 epochs run) and the attack 109,622.43 in full (separate run; not in the gist).

Who receives Junior's gain depends on how the mark returns. In a further suite re-run on this fixture (separate run; not in the gist), with the mark falling 0.001 per hour while the SHORT is open, a 1 USDG Junior deposit request pending since before the deficit, settled in the second Junior deposits reopen, receives 115,891.25, the LP who sent the settlements 2,317.48 and the Senior LP 22,815.15. In the 21 of its 23 tests that pass there (two stop at a helper assertion), the extra settlements leave the Senior LP's Senior payout between 22,814.80 and 22,815.36 wherever value returns; only on gradual paths can a Senior LP holding a Junior deposit request take value back, from Junior holders.

Who can do it. Any holder of Junior shares (one transferred sliver is enough), with no role and no further capital. While Junior principal is zero the exit requests are zero-value and are refunded, and such refund epochs count as queue progress; the cost is gas plus the Pyth fee per settlement. In form B the attacker files one exit request per hourly epoch from sub-accounts holding share slivers while the pool is degraded, then sends all settlements as his first transaction after the owner's `clearDegradedMode`. An attacker is not required: Junior exit requests filed by other LPs before each keeper settlement leave Senior 30,000 lower (S1). The control in form B is a single settlement on the same backlog.

Recovery and defences.

- With the bucket drained, `assignUnassignedAssets` reverts with `HousePool__NoUnassignedAssets` (S3a). A later owner top-up of 30,000 does not restore Senior principal: relative to the same run without it, 7,742.48 goes to Junior and 22,185.16 to unassigned assets, which the owner can assign as Senior shares to a named receiver (S3b). We found no owner function that moves Junior principal back to Senior.
- An owner top-up of `D` before the extra settlements prevents the loss at 1.86 and 1.90, but the loss is still 34,000 at 1.94 and 40,600 at 1.98 (also with `2D`) (S3c, S3d).
- An LP-settlement hold placed first only delays the steps, because exit requests are still accepted during it; a hold kept until the mark has returned and the SHORT has closed leaves the high-water mark unlowered (S4a, S4b).
- A matured Senior exit request with a non-zero payout stops the settlements: each reverts with `HousePool__NoLpEpochProgress` (S2: 10% of the LP's shares; S2b: 1.99 USDG). While the deficit is open a partial Senior request is rejected with `TrancheVault__WithdrawalTooSmall`; only a full-balance one is accepted (S2c).

Preconditions:

- A terminal LP deficit: unrealized trader profit plus claims above pool assets. The market is a six-currency dollar index capped at `CAP_PRICE = 2e8`; LONG profits when the price falls. In the PoC a 140,000 LONG and a 140,000 SHORT face 158,002 of pool assets; the price goes 1.00 → 0.50, the LONG is paid, then 1.90, leaving the SHORT in profit. The SHORT posted 140,000 of margin (it survives the first leg), the LONG 10,000. For a symmetric book the two moves, each relative to entry, must sum to more than pool assets divided by one side's notional (1.129 here; about 1.0025 at maximum utilisation, from the 25 bps settlement buffer): a round trip above roughly 100% of entry, extreme for an FX basket. We found no other route to a deficit; this is reasoning from the solvency gate, not a test. The runbook "Explicit terminal-deficit cascade" (`SECURITY.md:1499-1511`) covers this state (step 4: "Recapitalize or clear positions").
- The owner recapitalizes `R <= 2 × deficit` and then clears degraded mode; every test here uses that sequence, with no other owner action while degraded, and we make no claim about the bound outside it. At the smallest `R` that satisfies `clearDegradedMode` this needs `L + E <= 2U` (`L` max liability, `E` pool assets less claims, `U` the open side's unrealized profit). In the PoC book the sweep's lowest second price, 1.82, already gives a loss; no lower price was run. S5 measures seven points at that minimum plus 100 USDG (`R` = 51,914.84, settlements two hours apart; the last row uses `R` = 66,889.87):

  | second price | `D` | extra settlements | Senior loss |
  |---|---|---|---|
  | 1.82 | 25,300 | 2 | 25,300 |
  | 1.86 | 19,700 | 2 | 19,700 |
  | 1.90 | 14,100 | 3 | 28,200 |
  | 1.94 | 8,500 | 6 | 42,500 |
  | 1.98 | 2,900 | 17 | 46,400 |
  | 1.995 | 800 | 48 (test cap) | 38,400 |
  | 1.80, 150,000 per side | 30,100 | 2 | 30,100 |

- The mark stays near the second price while the settlements run.
- No Senior holder has a matured exit request with a non-zero payout queued when the settlements run.
- For form B, the pool stays degraded for 33 epochs (two steps, 7,814.84 of the bucket left pending) or 49 epochs (bucket drained) before the owner acts.

Closest exclusions.

- Privileged access. The trigger includes the owner calls `recapitalizePool` and `clearDegradedMode`, a documented recovery used as intended, and the program lists a legitimate, documented action by a trusted role that triggers a bug as in scope.
- Known issues. In this state every extra settlement gets its queue progress from a zero-payout Junior refund epoch, in S1 as well, and form B builds its backlog the way Cecuro MED-24 describes (sub-accounts filing full-balance exits in consecutive epochs). That refund epochs count as progress is known: the program lists "MED-24 and MED-29 (LP refund epochs and redemption rounding)" among the open Cecuro findings, and MED-24's stated effect is delay of other redeemers. The loss here does not come from the settlement being allowed to run; it comes from the reconcile inside it, which changes the tranches although nothing economic has changed. That root cause sits in the SC-L1 fix, and it became reachable through the fix for Cecuro LOW-22 ("Governance cannot use the documented non-dilutive recapitalization path"), which added `recapitalizePool` in PR #121, after the commit Cecuro audited. For fixed findings the program says a report on the same root cause "is valid only if it shows that the fix at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 is incomplete or introduces a new bug"; this report shows the SC-L1 fix is incomplete. The fixes for Cecuro MED-12 and MED-1 (both "Addressed") changed the same reconcile path for different defects: revenue parked in `unassignedAssets` after a same-batch recapitalization (MED-12), and a rate-change gate checking mark age against the pool's staleness limit only (MED-1). `SECURITY.md` lists under its pool invariants "Senior impairment must be restored before junior extracts surplus", which this behaviour contradicts.
- Preconditions. We make no claim about the price path's likelihood.

## Recommendation
Cap the settleable part of a pending bucket by the equity that is not already assigned to principal or unassigned assets, on every reconcile after the one that first applies the bucket. Capping the whole bucket against `distributable - (seniorPrincipal + juniorPrincipal + unassignedAssets)` is not correct: it reads the stored principals and so would also change that first reconcile; the residual left by the previous reconcile has to be tracked apart from new inflows. We tested a patch of that shape only on the earlier fixture (seeds of 1,000 USDG), where it removed the effect in a stateful fuzz but left the unbacked residual reserved: in one sequence the Senior tranche's cash-backed principal could not be withdrawn. A complete fix also has to deal with that reserved residual; we have not built or tested one.

## References
SC Audit Studio pre-audit SC-L1 and its fix response; `SECURITY.md` "Owner-funded recapitalization" (line 155).

## Link to Proof of Concept

https://gist.github.com/mhkarazeybek/23e92d83d956c9bec61f81470b03a7a0

## Proof of Concept

## Proof of Concept
Two Foundry test files at the in-scope commit: `ResidualRecapSeniorTheft.t.sol` (forms A, B, C; 4 tests) and `ResidualRecapSupplement.t.sol` (S1–S5; 11 tests). All 15 pass.

```
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/poc
cp ResidualRecapSeniorTheft.t.sol ResidualRecapSupplement.t.sol packages/perps/test/poc/
forge test --root packages/perps --match-path 'test/poc/ResidualRecap*.t.sol' -vv
```

What the tests use:

- No in-scope contract is modified. There are no storage writes, no mocked calls and no impersonation of protocol contracts.
- After `setUp()`, apart from mock Pyth price setters, mock USDG mints, time and block cheatcodes and state snapshots (control and attack start from one snapshot), every state change is an external call from the actor's own address to a public entrypoint (orders go through `OrderRouter.commitOrder(OrderRequest)`). In form B the attacker deploys a bundler contract and moves share slivers to sub-accounts with ERC-20 transfers.
- The Router instance is the fixture's `LegacyOrderRouterHarness`, called only through the `OrderRouter` type (no harness-only function). Order requests carry permissive bounds.
- Form B sends the same price update several times in one transaction; these files do not show that a production Pyth deployment accepts the repeat. Form A does not rely on it.
- `setUp()` is the fixture's owner deployment plus the release values of `DEPLOYMENT.md`, asserted before any LP deposit: 1 USDG seeds per tranche, Senior exposure cap 40,000,000 USDG and share limit 8,000 bps, oracle adverse multiplier 2,500 bps, Router opening minimum 1,000 USDG. The last four use the owner's timelocked config paths (time cheatcodes pass the delays, then restore the fixture's start time). The ten risk parameters and `CAP_PRICE` are release values, set but not asserted; the fixture's other settings are not checked against the release table. Not release values: USDG and Pyth are the fixture's mocks, the basket is its two 50/50 feeds (release: six), and the Pyth mock reports zero confidence and zero fee, so the adverse multiplier is installed but has no effect in these runs. Separate runs (not in the gist): on six feeds, four inverted (a Pyth double), all 15 pass with assertions unchanged at 0, 1, 5 and 9 bps confidence per feed (10 refused; at 0, form A: Senior LP 52,814.75 → 22,814.73, attacker 46,831.11 → 109,622.50, within 0.07 of shipped); on the shipped fixture, at 1, 5 and 10 bps (11 refused). Confidence leaves the Senior LP's loss in forms A and B unchanged; wallet levels move by up to 243.
- After `setUp()`, owner calls in the main file are `recapitalizePool` and `clearDegradedMode`. The supplement also uses `assignUnassignedAssets`, a second `recapitalizePool`, `setPauser`, `pauseLpEpochSettlement` and `unpauseLpEpochSettlement`; an EOA stands in for the EmergencyPauseCoordinator as pauser.

Output of form A, excerpt (USDG has 6 decimals; gas and other lines are omitted):

```
[PASS] test_SeniorLosesPrincipalToJunior_WalletLevel()
  terminal deficit before recovery: 37814842804
  recapitalization R: 52814842804
  after owner recovery and the first ordinary settlement
     seniorPrincipal: 15000000000
     seniorHighWaterMark: 52814842804
     pendingRecapitalizationAssets: 37814842804
     pool totalAssets: 141000000000
  ATTACK: after an extra settlement sent by the attacker
     seniorPrincipal: 15000000000
     seniorHighWaterMark: 37814842804
     pendingRecapitalizationAssets: 22814842804
     pool totalAssets: 141000000000
  WALLET TABLE: three extra settlements (three hours apart) vs none
     Senior LP original deposit: 50000000000
     Senior LP wallet USDG, CONTROL: 52814751143
     Senior LP wallet USDG, ATTACK : 22814803209
     attacker wallet USDG, CONTROL: 46831073915
     attacker wallet USDG, ATTACK : 109622432297
     conservation: juniorGain - seniorLoss - unassignedDrop - lpValueDiff: 0
```

The second and third extra settlements print the same block with the high-water mark at 22,814,842,804 both times and the pending bucket at 7,814,842,804, then 0; pool assets and trader claims are unchanged in all three. The 33-epoch form B test prints block 45 before the owner's recovery and after the attacker's transaction, with three successful settlements.
