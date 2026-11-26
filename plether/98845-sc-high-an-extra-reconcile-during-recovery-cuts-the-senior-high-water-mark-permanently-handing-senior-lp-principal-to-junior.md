# \#98845 [SC-High] An extra reconcile during recovery cuts the Senior high-water mark, permanently handing Senior LP principal to Junior

**Submitted on Nov 10th 2026 at 14:27:02 UTC by @RichJamo for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98845
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

After a HousePool becomes insolvent and the owner recapitalises it while trader claims are still outstanding, one ordinary reconcile landing before those claims settle permanently cuts the Senior tranche's recovery target — its high-water mark — down to the size of the unpaid claims and empties the reserve the recapitalisation set aside for Senior. Later revenue then restores Senior only to that reduced target, and the difference — up to nearly the whole recapitalisation — is permanently redirected to the Junior cohort. In production this means Senior LPs permanently lose principal to Junior LPs after any recapitalised insolvency whose claims clear across more than one reconcile, which is the ordinary case; the protocol's own accounting spec (`ACCOUNTING_SPEC.md:337`) and its own test (`HousePoolWaterfall.t.sol:201`) say this must never happen.

## Vulnerability Details

**Affected code** (`packages/perps/src`, commit `a82d4fef`): `HousePool.sol` (the reconcile, the waterfall, the high-water mark), `HousePoolPendingPreviewLib.sol` (the mark re-bootstrap), `HousePoolWaterfallAccountingLib.sol` (loss absorption and revenue distribution).

**Preconditions.** No attacker role is required — this is a correctness bug in the recovery accounting, triggered by ordinary recovery operations. The pool must be insolvent (both tranches terminally wiped), with an outstanding trader claim and a partly-pending recapitalisation (claims exceed roughly half the recapitalisation). Insolvency is an anticipated state the protocol provides recovery for — degraded mode was added "for post-close insolvency" (commit `d32c452e`) and `SECURITY.md` documents the owner recapitalising an insolvent pool. It does **not** require any other bug; the PoC reaches it through ordinary trading.

**Root cause.** The Senior high-water mark should be a persisted recovery target that losses and reconciles never lower (`ACCOUNTING_SPEC.md:337`, "losses do not reduce the Senior high-water mark"). Instead, on a reconcile that finds both tranches wiped, the mark is **re-bootstrapped** from the pending-recapitalisation amount being applied that reconcile. `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent` (`:116-121`):

```solidity
if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
    state.waterfall.seniorPrincipal += remaining;
    state.waterfall.seniorHighWaterMark =
        recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
    remaining = 0;
}
```

Both `remaining` and `recapitalizationTargetAssets` here come from the pending-recapitalisation bucket — and that bucket is drawn down on **every** reconcile (`HousePool.sol:1787`):

```solidity
function _decreasePendingClaimantBuckets(
    HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets
) internal {
    pendingRecapitalizationAssets -= claimantBuckets.recapitalizationAssets;
    pendingTradingRevenueAssets -= claimantBuckets.revenueAssets;
}
```

On the recap-applying reconcile the full recapitalisation is still intended, so the mark is set to 100,000. But an extra reconcile first re-charges the just-restored Senior principal as a loss (both tranches wiped again), then re-bootstraps the mark from the now-decremented bucket: `recapitalizationTargetAssets` is the residual, so `seniorHighWaterMark = max(residual, residual) = 51,447`, and the target is never restored past that. Nothing persists the original recovery target across reconciles. This contradicts the stated invariant that Senior impairment is restored before Junior extracts surplus (`SECURITY.md:394`), the accounting rule at `ACCOUNTING_SPEC.md:337`, and the team's own assertions (`HousePoolWaterfall.t.sol:201` and `:217`).

The defect is in `_reconcile` itself, so it fires on **any** production path that reaches a reconcile while a claim is outstanding: the owner's `finalizePoolConfig` (a senior-rate change, `HousePool.sol:341`), `assignUnassignedAssets` (`:671`), or the permissionless hourly keeper settle `settleLpEpoch` (`:835`, `LP_EPOCH_DURATION == 1 hour`). The direct `reconcile()` is `onlyVault` (`:1471`) and no production caller reaches it.

**How it plays out (the ordinary sequence).**

1. The pool goes insolvent: a winning trader is owed more than the pool holds; a reconcile recognises the loss and wipes both tranches, leaving the Senior high-water mark as the recovery target.
2. The owner recapitalises (`CfdEngine.recapitalizePool`). Because claims are unpaid, part is reserved as pending, and the mark is set to the full recapitalisation.
3. The trader has not settled yet — a claim is pulled by its owner (`settleTraderClaim` requires `msg.sender == account`); the protocol never pushes it, so a gap exists between step 2 and all claims clearing.
4. An ordinary reconcile lands in that gap — an owner recovery operation, or a routine hourly keeper settle processing LP redemptions. It cuts the mark to the unpaid-claim residual and empties the reserve.
5. The claim settles and revenue arrives. Senior is restored only to the cut target; the residual flows to Junior. Permanent. Avoiding the cut would require the owner to recapitalise and every trader to settle before the next reconcile of any kind — the unlikely, tight-timing case.

**Why this is direct theft of user funds, not accepted reallocation.** The Senior→Junior move transfers value between LP cohorts with nothing leaving the pool — the shape Immunefi once cited to close a VPI report. Two things make it theft here: (a) the protocol's own spec forbids it (`ACCOUNTING_SPEC.md:337`, `SECURITY.md:394`, and the team's own passing test `HousePoolWaterfall.t.sol:201`), unlike the VPI timing transfer the spec accepts for carry (`ACCOUNTING_SPEC.md:737`) — there is no document that accepts this; and (b) this program has already ruled a HousePool LP-cohort transfer Critical "Direct theft" (report #97591).

**Why this is not #97591.** #97591 (confirmed Critical) is refundable VPI counted in the Junior pricing NAV — `terminalLpPriceDeltaAssets` carrying open positions' `vpiAccrued`, fixed by subtracting an `openVpiAccruedAssets` term. It never touches the Senior high-water mark, the recovery reconcile, recapitalisation, or insolvency. This finding is the opposite in every axis: the **Senior** tranche (not Junior), the **recovery reconcile / high-water mark** (not VPI pricing), requiring **insolvency + recapitalisation** (not normal operation), and moving value **to** Junior (not away). The only shared surface is the contract.

**Recommended fix.** The defect is that the mark's bootstrap target is read from the live, decremented `pendingRecapitalizationAssets` bucket (`_reconcile` passes `_getPendingClaimantBuckets()` as the intent to `applyClaimantRecapitalizationIntent`). The fix is to **persist the recapitalisation's full target** when it is recorded and bootstrap the mark from that, not from the decremented bucket — a localized change, since the amount actually *settled* each reconcile is computed separately and stays as-is. Senior-first recovery, the reserved residual, the designed mark-reset on a full wipe, and revenue restoring Senior are all preserved; the team's own residual-recap test (`HousePoolWaterfall.t.sol:201`) then passes in the two-reconcile case, not only the one-reconcile case:

```diff
--- a/packages/perps/src/HousePool.sol
+++ b/packages/perps/src/HousePool.sol
@@ state
   uint256 public pendingRecapitalizationAssets;
+  /// @notice Full recovery target of the active recapitalisation, set when recorded and held across reconciles.
+  ///         The high-water-mark bootstrap must use this, never the live (decremented) pending bucket.
+  uint256 public seniorRecapitalizationTargetAssets;

@@ _recordPendingClaimantInflow(kind, amount)   // the Recapitalization branch (~:1798)
   if (kind == IHousePool.ClaimantInflowKind.Recapitalization) {
       pendingRecapitalizationAssets += amount;
+      seniorRecapitalizationTargetAssets += amount;   // remember the full target
   }

@@ _reconcile(...)   // ~:1564; `settleableClaimantBuckets` (the amount applied) is already computed from the real bucket
   HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets =
       _settleablePendingClaimantBuckets(reconcileSnapshot, claimantBuckets);
+  // Bootstrap the mark from the persisted full target, not the decremented bucket. Settlement is unaffected.
+  claimantBuckets.recapitalizationAssets = seniorRecapitalizationTargetAssets;

@@ _decreasePendingClaimantBuckets(...)   // ~:1787; clear the target once the recapitalisation is fully settled
   pendingRecapitalizationAssets -= claimantBuckets.recapitalizationAssets;
+  if (pendingRecapitalizationAssets == 0) {
+      seniorRecapitalizationTargetAssets = 0;
+  }
```

(Sketch — the reset condition may want refining for partial-settlement edge cases, but the three substantive hunks are the storage field, setting it on record, and sourcing the bootstrap target from it.)

## Impact Details

**Selected impact: Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield.** The stolen value is Senior LP principal, permanently redirected to the Junior cohort. The program rates severity by impact, not likelihood (`RULES.md`), and a high capital requirement is not a reason to downgrade.

PoC, USDG, control vs variant from the same recovery, differing only by whether one extra reconcile lands while the claim is outstanding:

| | Control (one reconcile applies the recap) | Variant (one extra reconcile first) |
|---|---:|---:|
| Senior recovery target before the extra reconcile | 100,000 | 100,000 |
| Senior high-water mark after the extra reconcile | n/a | **51,447** (cut by 48,553) |
| **Senior principal after claims clear and revenue arrives** | **100,000** | **51,447** |
| Junior principal at the end | ~197,106 | ~294,212 |

**Funds at risk: Senior LP principal.** In the PoC, Senior ends **48,553 of a 100,000 target short (~48.5%)**, permanently redirected to Junior. This is a **conservative mid-point, not a ceiling:** the loss equals the recapitalisation minus the settleable claim, so a *smaller* outstanding claim pushes Senior's loss toward ~100% of the recapitalisation. The pool's total is conserved (Senior's loss equals Junior's gain); the loss falls on the Senior cohort specifically, and is permanent — once the mark is cut and the reserve emptied, later revenue restores Senior only to the reduced target.

**Not reliant on any excluded vector:** no oracle manipulation, no governance or 51%-style action, no Sybil identities, and no large or privileged stake. The cut is produced by the protocol's own ordinary recovery operations — an owner config change (`finalizePoolConfig`), `assignUnassignedAssets`, or a permissionless hourly keeper settle (`settleLpEpoch`) — landing in the window between recapitalisation and claim settlement.

## References

- https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8 — the audit repository at the reviewed commit
- `ACCOUNTING_SPEC.md:337` — losses do not reduce the Senior high-water mark (the broken rule)
- `SECURITY.md:394` — Senior impairment restored before Junior extracts surplus
- `HousePoolWaterfall.t.sol:201`, `:217` — the team's own assertions that a residual recap preserves and settles the target
- `HousePool.sol` — `_decreasePendingClaimantBuckets` (`:1787`), reconcile entry points (`:341`, `:671`, `:835`), `reconcile()` onlyVault (`:1471`)
- `HousePoolPendingPreviewLib.sol:116-121` — `applyClaimantRecapitalizationIntent`, the mark re-bootstrap from the pending bucket

## Link to Proof of Concept

https://gist.github.com/RichJamo/03dd76b46e7f39c941186e23b128cb2b

## Proof of Concept

## Proof of Concept

The full PoC is one self-contained Foundry test, `RecapResidualEndToEnd.t.sol`, provided in the secret gist linked with this report: https://gist.github.com/RichJamo/03dd76b46e7f39c941186e23b128cb2b. It reaches an insolvent, recapitalised pool through real production calls only (no storage cheats): a real winning close records the trader claim, the owner's real `recapitalizePool` funds the recovery, real production reconciles apply and then cut the mark, and the trader's real `settleTraderClaim` plus fresh revenue finish it.

**Download `RecapResidualEndToEnd.t.sol` from that gist, then run** (Forge 1.5.1; the audit repository at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` with submodules; the default profile's via-IR is on — the numbers are identical with and without it):

```
cp RecapResidualEndToEnd.t.sol <repo>/packages/perps/test/perps/accounting/waterfall/
cd <repo>/packages/perps
forge test --match-path 'test/perps/accounting/waterfall/RecapResidualEndToEnd.t.sol' -vv
```

**Expected output** (abridged; amounts are USDG with 6 decimals, so `100000000000` = 100,000):

```
[PASS] test_Control_OneReconcileRestoresSeniorToTarget()
  [D] control: after the recap-applying reconcile
    senior HWM      : 100000000000
  [G] control: after revenue
    senior principal: 100000000000      <- Senior fully restored to its target

[PASS] test_Variant_SecondReconcileCutsTheMark()
  [D] variant: after the recap-applying reconcile
    senior HWM      : 100000000000
  [E] variant: after the SECOND reconcile (claim still outstanding)
    senior HWM      :  51447004537      <- cut by 48,553
  [G] variant: after revenue
    senior principal:  51447004537      <- Senior permanently short of 100,000
    junior principal: 294211981852      <- the stranded residual went to Junior

[PASS] test_Variant_KeeperSettleCutsTheMark()
  [E] after the permissionless hourly keeper settle (called from an arbitrary address)
    senior HWM      :  51396194263      <- same cut, via settleLpEpoch rather than the owner path

Suite result: ok. 3 passed; 0 failed; 0 skipped
```

**Core of the test** (copied from `RecapResidualEndToEnd.t.sol`). Two reconcile helpers — one an owner production path, one the permissionless keeper path — and the variant that proves the cut; the control differs only by not calling the second reconcile:

```solidity
// An owner production reconcile: finalizePoolConfig reconciles on a senior-rate change,
// and works on a terminally wiped pool where new deposits are blocked.
function _productionReconcile() internal {
    reconcilerNonce++;
    vm.warp(block.timestamp + 1 hours);
    _refreshMark();
    IHousePool.PoolConfig memory config = _currentPoolConfig();
    config.seniorRateBps = config.seniorRateBps == 500 ? 501 : 500;
    pool.proposePoolConfig(config);
    vm.warp(pool.poolConfigActivationTime());
    _refreshMark();
    pool.finalizePoolConfig();
}

// The permissionless hourly keeper path: with the position closed the pool has no open
// positions, so settleLpEpoch's router-auth branch is skipped and ANY address may call it.
function _keeperReconcile() internal {
    reconcilerNonce++;
    vm.warp(block.timestamp + 1 hours);
    _refreshMark();
    uint256 mark = engine.lastMarkPrice();
    vm.prank(address(0xC0FFEE)); // any address
    pool.settleLpEpoch(mark, block.timestamp);
}

function test_Variant_SecondReconcileCutsTheMark() public {
    _reachWipedWithClaim();   // winning close records the claim; a reconcile wipes both tranches
    _recapitalize();          // owner recapitalizePool(100_000)

    _productionReconcile();   // applies the recap
    assertEq(pool.seniorHighWaterMark(), RECAP_AMOUNT, "HWM is the full recovery target");
    uint256 residualReserved = pool.pendingRecapitalizationAssets();

    _productionReconcile();   // THE EXTRA RECONCILE, claim still outstanding
    // the recovery target is cut to the still-reserved residual:
    assertEq(pool.seniorHighWaterMark(), residualReserved, "the extra reconcile cut the HWM to the residual");
    assertLt(pool.seniorHighWaterMark(), RECAP_AMOUNT, "the recovery target is now below the recap");

    vm.prank(TRADER);
    engine.settleTraderClaim(TRADER);
    _bringRevenue(300_000e6);
    _productionReconcile();

    // the harm: Senior can no longer be restored past the cut target; the residual leaked to Junior:
    assertLt(pool.seniorPrincipal(), RECAP_AMOUNT, "Senior is permanently short of its recovery target");
    assertGt(pool.juniorPrincipal(), 0, "the stranded senior residual leaked to Junior");
}
```

Each reconcile is a real production entry point (`finalizePoolConfig`; and `settleLpEpoch` in the keeper variant), never the `onlyVault` `reconcile()`. Pool illiquidity is created by moving cash out of the pool, modelling earlier winners already paid — the team's own `EngineTraderClaims` technique.

