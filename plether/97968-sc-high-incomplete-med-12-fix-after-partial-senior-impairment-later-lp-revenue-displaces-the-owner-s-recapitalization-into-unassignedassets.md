# \#97968 [SC-High] Incomplete MED-12 fix: after partial Senior impairment, later LP revenue displaces the owner's recapitalization into unassignedAssets

**Submitted on Nov 4th 2026 at 05:37:42 UTC by @tinnohofficial for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97968
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Permanent freezing of unclaimed yield

## Description

## Brief/Intro

`CfdEngine.recapitalizePool()` is the documented, non-dilutive owner recovery action for an impaired Senior tranche (it was added to fix LOW-22). It transfers USDG into `HousePool` immediately but only queues the allocation in `pendingRecapitalizationAssets` until the next reconciliation. When Senior is **partially** impaired (`0 < seniorPrincipal S < seniorHighWaterMark H`, `juniorPrincipal J = 0`, Junior shares outstanding), LP revenue that reaches the pool before the next LP settlement is not queued: `recordClaimantInflow()` queues revenue only when `S + J == 0`, so it is ordinary surplus. The next reconciliation spends that later revenue on the Senior gap first (`planReconcile()` → `distributeRevenue()`) and only then applies the pending recapitalization. The recapitalization now finds less gap, and the remainder is booked as ownerless `unassignedAssets`.

For a recapitalized gap `G` and later revenue `T`:

```text
a settlement runs between the two inflows : S = H, J = T,              unassignedAssets = 0
the revenue reaches the pool first        : S = H, J = T - min(G, T),  unassignedAssets = min(G, T)
```

Junior receives `min(G, T)` less, and the same amount of the owner's recapitalization becomes ownerless. While that bucket is nonzero, new deposit requests in both tranches and deposit activation are blocked. The bucket absorbs losses only after both tranches, and only the owner can release it, by minting new shares to one receiver. Which outcome occurs depends only on whether a permissionless keeper calls `settleLpEpoch()` before or after the revenue-producing execution.

The strongest reasons this is not intended:

1. **In the pool's own canonical view, a claimant revenue inflow is what creates the ownerless assets.** Right after `recapitalizePool(G)`, `getPendingTrancheState()` (the projection `TrancheVault.totalAssets()` uses for share pricing) reports Senior at its high-water mark and Junior at 0, and, with a fresh mark, `canSettleDepositEntries()` returns `true` again. When LP revenue `T` then arrives, the same views leave Senior unchanged, credit Junior with only `T - min(G, T)`, and `canSettleDepositEntries()` returns `false`, because the projection now contains `min(G, T)` of ownerless assets. ACCOUNTING_SPEC sends to `unassignedAssets` only "revenue with no eligible share owner" and "value with no safe claimant path". Here Junior shares exist and the pool itself reports Senior whole.
2. **LP ownership becomes a function of settlement timing.** The protocol treats settlement timing chosen by a permissionless caller as an attack surface: a settlement that advances no queue item reverts ("This prevents per-block no-op checkpoint grief"), and the residual SECURITY.md discloses as "execution-frequency dependent" is "sub-micro-USDG rounding". Here the order of two progress-making permissionless calls moves `min(G, T)`.
3. **The documented recovery action becomes a no-op for Junior.** With revenue first, Junior ends exactly where it would be had the owner never recapitalized (`J = T - min(G, T)` in both cases, proven by a counterfactual test). When `T > G`, no claimant gains anything from the recapitalization: all of it is ownerless, and the only way to deploy it is the share-minting bootstrap assignment that LOW-22 listed among the "economically different" alternatives.
4. **The project's own MED-12 regression codifies the opposite outcome.** It asserts *"Recapitalization restores senior first; same-batch revenue then belongs to surviving junior owners"* and *"mixed recovery must not block projected deposit settlement"*. With `0 < S < H`, both properties fail.

The waterfall order in ACCOUNTING_SPEC (ordinary surplus in step 4 before pending buckets in step 5) and the whitepaper sentence "Revenue restores impaired senior principal to its high-water mark before junior participates" produce this result mechanically. The section **Specification conflict** below quotes them and explains why they do not decide this case.

The proof of concept changes protocol state only through public entrypoints: vault requests and claims; the inherited, unmodified V3 `OrderRouter.commitOrder(OrderRequest)` and `executeOrder`; permissionless `OrderRouter.settleLpEpoch` and `updateMarkPrice`; and the owner's documented `recapitalizePool` and `assignUnassignedAssets`. It runs on the repository's `BasePerpTest` suite, which uses `MockPyth`, `MockSettlementToken` and its `LegacyOrderRouterHarness`; only the harness's inherited V3 and keeper paths are called. The test asserts the DEPLOYMENT.md release parameters, including the $1,000 minimum open notional, the 2,500-bps adverse-confidence multiplier and the 40,000,000 USDG / 80% Senior limits, all finalized through the owner timelocks. The one remaining difference is the suite's two-feed equal-weight basket; the effect depends only on the aggregate mark. Ten tests, including a 256-run fuzz, pass against unmodified code at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`:

| Scenario (~1M USDG pool) | Revenue reaches the pool before the next settlement | Settlement runs first (control) |
|---|---|---|
| 1.0% shock: gap `G` = 1,797.998603, later revenue `T` = 420.79 | Junior **0**; unassigned **420.790409** (= `T`); all deposits closed | Junior 421.211619; unassigned 0; Junior deposits open |
| 0.2% shock: `G` = 199.600202, `T` = 563.64 | Junior **364.044744**; unassigned **199.600202** (= `G`, the whole recapitalization); all deposits closed | Junior 564.210060; unassigned 0; Junior deposits open |

`T` differs by about 0.1% between the columns because the control funds a 1,000 USDG Senior exit before the trade, which slightly changes VPI depth. Each column is asserted against its own measured `T`.

## Why this is an incomplete fix of MED-12 (and of LOW-22)

The Cecuro report lists MED-12, *"Mixed recovery inflows can strand LP revenue in ownerless pool assets"*:

> Already-funded LP revenue can lose its claimant path during wipeout recovery and get parked in unassignedAssets even when seeded LP ownership still exists. [...] That 15,000 USDC is then excluded from normal LP ownership, blocks new tranche deposits, and can reach LPs only through the owner-only assignUnassignedAssets() path. This contradicts the protocol's own claimant-continuity rule that value should use seeded ownership paths when available and only truly ownerless value should enter unassignedAssets.

Its stated impact: *"LP-owned revenue is misclassified as ownerless cash. The funds stop following normal senior/junior ownership, new deposits remain blocked because unassignedAssets > 0, and recovering the value requires an owner-only reassignment that can arbitrarily choose the receiver."*

The team's resolution:

> MED-12 — Addressed. The shared pending-accounting helper recomputes revenue continuation after applying recapitalization. When recapitalization restores principal and LP shares survive, revenue in the same batch follows the restored claimant ownership instead of being quarantined as unassigned. The reported 40,000 USDC recapitalization plus 15,000 USDC revenue case now produces senior principal of 40,000, junior principal of 15,000, and zero unassigned assets. Preview and live reconciliation use the same helper. Recapitalization remains senior-first; partial-settlement residuals stay reserved, and surplus without eligible owners remains unassigned.

The fix lives entirely in `HousePoolPendingPreviewLib.applyPendingClaimantBucketsPreview()`, which recomputes `allowRevenueContinuation` after the recapitalization. That helper only sees revenue recorded in `pendingTradingRevenueAssets`, and `HousePool.recordClaimantInflow()` records revenue there only when `seniorPrincipal + juniorPrincipal == 0`:

```solidity
if (kind == IHousePool.ClaimantInflowKind.Recapitalization) {
    _recordPendingClaimantInflow(kind, amount);
} else if (seniorPrincipal + juniorPrincipal == 0) {   // revenue is pending only after a total wipe
    _recordPendingClaimantInflow(kind, amount);
}
```

After a partial impairment the recapitalization is pending but the later revenue is not, so the revenue never reaches the fixed helper. It is consumed by the ordinary plan before the helper runs, and the MED-12 impact reappears: value that the surviving Junior shares would otherwise own lands in `unassignedAssets`, deposits are blocked, and recovery requires the owner-only reassignment. The diagnostic test `test_DiagnosticPendingHelperGivesTheSameTwoAmountsToJunior` applies the fixed helper to the same two amounts (as if the revenue had been queued, as it is after a total wipe): it yields `Junior = 563.644946, unassigned = 0`, while the live reconciliation of the same state yields `Junior = 364.044744, unassigned = 199.600202`.

LOW-22 (*"Governance cannot use the documented non-dilutive recapitalization path"*) was fixed by adding `recapitalizePool()`. Its stated impact was that *"the only in-scope alternatives are economically different actions such as dilutive seeding or bootstrap assignment"*. In the `T > G` case above, the recapitalization leaves every claimant's principal at the no-recapitalization value (Senior whole, Junior `T - G`) and turns all of the owner's cash into `unassignedAssets`. The only way to deliver that cash to LPs is `assignUnassignedAssets()`, the bootstrap assignment LOW-22 described as economically different.

## Vulnerability Details

### Preconditions

- **Senior is partially impaired: `0 < S < H` and `J = 0`, with Junior shares outstanding.** Outside owner bootstrap assignments, `S < H` implies `J = 0`: losses are charged to Junior before Senior, revenue restores the Senior gap before it reaches Junior, and deposit activation is blocked while Senior is impaired. Junior shares always survive because the seed shares are permanent. This is the state `recapitalizePool()` exists to repair, and the project's own `test_RecapitalizePool_RestoresSeniorWithoutMintingShares` uses it (`S = 60,000`, `H = 100,000`, `J = 0`, both seed supplies alive).
- **The owner recapitalizes the observed gap** through `CfdEngine.recapitalizePool(G)`, a documented, immediate, trusted recovery action. No owner error is involved.
- **Some LP revenue or favourable mark move reaches the pool before the next progress-making LP settlement.** This is readily reachable:
  - `recapitalizePool()` only queues the amount, and the protocol exposes no recap-and-reconcile entrypoint;
  - `HousePool.reconcile()` is vault-only, and no vault calls it;
  - `settleLpEpoch()` reverts with `HousePool__NoLpEpochProgress` unless a matured LP request exists, and LP epochs are hourly with a five-minute cutoff;
  - the owner's other reconciling paths are timelocked (`finalizePoolConfig` with a rate change), one-time (`initializeSeedPosition`), or revert when nothing is unassigned (`assignUnassignedAssets`);
  - `recapitalizePool()` is documented to remain available "with stale marks, frozen markets, and LP pause or settlement holds", including conditions in which no LP settlement can run.

  Qualifying inflows include VPI on any skew-increasing open, the LP share of liquidation charges, collected carry, frozen-close spreads, and any mark move in the LPs' favour. In addition, any keeper can deliberately execute a third party's revenue-producing order before `settleLpEpoch()` in the same block, at the cost of gas only.

How the first condition arises: Senior can only be impaired after Junior is exhausted. Senior admission at the release 80% share cap requires Junior to be at least 20% of protected capital, and the Junior covenant stops funding Junior redemptions that would take Junior below a quarter of protected Senior exposure. With the 40% opening skew cap, a mark-to-market loss alone would need a roughly 50% basket move to wipe a full 20% Junior. Senior impairment therefore follows a period in which cumulative trader gains and/or the Junior-funded coupon have thinned Junior (an 8% coupon on Senior costs 32% of initial Junior capital per year at the cap). The PoC compresses a 1,188-day (about 3.25-year) low-revenue period, then applies a 0.2% or 1.0% move to a SHORT of 20% of depth. Any path that exhausts Junior leads to the same state.

### Root cause

1. `HousePool.recordClaimantInflow()` always queues `Recapitalization` but queues `Revenue` only when `seniorPrincipal + juniorPrincipal == 0` (HousePool.sol L644-L648). With `S > 0`, later revenue is just an increase in distributable equity.
2. `HousePool._reconcile()` reserves the settleable pending recapitalization and calls `HousePoolReconcilePlanLib.planReconcile()` **before** applying the pending buckets (HousePool.sol L1564-L1598).
3. `planReconcile()` subtracts the pending recapitalization from distributable value, finds the later revenue as ordinary surplus and distributes it Senior-gap-first through `distributeRevenue()` (HousePoolReconcilePlanLib.sol L68-L99). The revenue fills `min(G, T)` of the gap, measured on stored principal that does not yet include the recapitalization.
4. Only then does `_applyPendingClaimantBucketsLive()` call `applyClaimantRecapitalizationIntent()`. It sees a remaining gap of `G - min(G, T)`, restores that much and books the rest of the recapitalization, exactly `min(G, T)`, as `unassignedAssets`, because "recapitalization never accrues to junior principal" (HousePoolPendingPreviewLib.sol L103-L137).

Total value is unchanged; only its owner changes. The preview path (`_previewPendingAccountingState`, used by `getPendingTrancheState()` and therefore by `TrancheVault.totalAssets()`) runs the same sequence, so the vaults report the displacement as soon as the later revenue arrives.

### Specification conflict

The texts that support the observed result:

- ACCOUNTING_SPEC L335-L340: *"2. Compare terminal equity, excluding reserved pending claimant inflows and `unassignedAssets`, with total principal. [...] 4. Route surplus first to restore Senior principal to its high-water mark, then to Junior. 5. Route pending recapitalization and trading-revenue buckets according to their explicit ownership intent, described below. Reserving them in step 2 prevents counting the same inflow twice."*
- WHITEPAPER §4.5: *"Revenue restores impaired senior principal to its high-water mark before junior participates."*
- LOW-22 response: *"Allocation is deferred to normal reconciliation under existing senior-recovery priority; residual value without a claimant path remains unassigned."*
- `test_RecapitalizePool_RestoresSeniorWithoutMintingShares` books a 50,000 recapitalization of a 40,000 gap as 10,000 of `unassignedAssets` while Junior shares exist. A recapitalization that exceeds the gap when it is funded is intended to stay unassigned.

Why they do not decide this case:

1. **The step order is a processing rule, not an ownership rule.** Its stated purpose is double-count prevention ("Reserving them in step 2 prevents counting the same inflow twice"). Ownership is assigned elsewhere: "source semantics decide the owner of the inflow" (L647), a recapitalization is "intended to restore waterfall claimants" (L634), and the pending recapitalization bucket holds "assets intended to restore senior principal, then remain unassigned", i.e. only what is left after the gap is restored is unassigned. No text says that revenue earned after a funded recapitalization takes over the recapitalization's role. The `_reconcile()` comment "Nonpending equity can restore seeded LP principal before the queued revenue is routed" shows that the stage-order interaction was handled for queued revenue, but not for the queued recapitalization.
2. **The whitepaper rule presupposes an impaired Senior.** When `T` arrives, the pool's canonical projection already reports Senior at its high-water mark. Applied to the state the protocol publishes and prices shares at, the rule sends `T` to Junior. The code applies it instead to stored principal that has not yet absorbed the earlier recapitalization.
3. **The recapitalization here is neither excess nor "residual value without a claimant path".** It equals the gap that existed when it was funded and becomes excess only because a later inflow is counted first. The project test covers a recapitalization that is oversized at funding. Here the outcome of an exactly sized one depends on whether a keeper settled in between.
4. **The reading that makes the code correct also makes ownership depend on call order, which the protocol forbids elsewhere.** A no-progress settlement reverts so that "its reconcile and carry checkpoints cannot be retained by a permissionless no-op caller" (ACCOUNTING_SPEC L303-L304). SECURITY.md L908-L911 adds "This prevents per-block no-op checkpoint grief" and names "sub-micro-USDG rounding" as the residual that "remains execution-frequency dependent". The WHITEPAPER (L704-L706) says it "prevents permissionless no-op calls from changing time-based accounting". Under the step-order reading, two progress-making permissionless calls in either order allocate up to `G` differently.
5. **The same two inflow types get the opposite priority after a total wipe.** With `S = J = 0` the revenue is queued, and the MED-12-fixed helper applies its documented rule "Recapitalization is always processed before revenue" (`HousePoolPendingPreviewLib` NatSpec; `capClaimantBuckets()` likewise caps "with recapitalization taking priority over revenue"). The regression's result is `J = T`, `unassignedAssets = 0`. These statements are scoped to pending buckets, but they record the designers' choice for these two inflow types. Nothing in the documentation explains why a partial impairment should reverse it.
6. **The result breaks the explicit ownership rules.** These are: "only value with no safe claimant path belongs in `unassignedAssets`", "`unassignedAssets` should be exceptional telemetry, not a normal operating bucket", "claimant continuity should prefer seeded ownership paths when available" (L647-L652), and "seeded ownership continuity is preferred over governance re-assignment" (L327).

The specification is therefore inconsistent for this sequence. Only the chronological allocation satisfies the ownership rules, the canonical projection, the MED-12 regression and the timing neutrality the protocol enforces elsewhere.

### Step-by-step reproduction (public entrypoints only)

1. A pool funded at the release Senior share cap (200,000 USDG Junior, 800,000 USDG Senior, 1 USDG seeds) goes through a low-revenue period. The Junior-funded 8% Senior coupon exhausts Junior principal. Junior shares survive, and Senior stays whole.
2. A trader holds a SHORT of 20% of pool depth. A 0.2% or 1.0% adverse basket move creates a loss larger than the remaining Junior principal, so a keeper's `settleLpEpoch()` persists `0 < S < H`, `J = 0`.
3. The owner calls `recapitalizePool(G)` with `G = H - S`. `getPendingTrancheState()` now reports Senior at `H`, and with a fresh mark `canSettleDepositEntries()` returns `true`.
4. An LP has an ordinary matured request (the PoC uses a 1,000 USDG Senior exit), so anyone can settle.
5. **Revenue first:** another trader's skew-increasing SHORT is executed through the Router (VPI revenue `T`). `canSettleDepositEntries()` flips to `false`. A keeper then calls `settleLpEpoch()`. Result: `unassignedAssets = min(G, T)` and `juniorPrincipal = T - min(G, T)`.
6. **Control:** the keeper calls `settleLpEpoch()` first, then the same trade executes, then the next epoch is settled. Result: `unassignedAssets = 0` and `juniorPrincipal = T`.

### Recommendation

Apply the recapitalization to the Senior gap that existed when it was funded, before ordinary surplus is distributed, in both live and preview reconciliation:

```solidity
// HousePool._reconcile() and _previewPendingAccountingState():
// 1. coupon checkpoint (unchanged)
// 2. apply the settleable recapitalization to the waterfall (restore Senior up to HWM; genuine excess -> unassigned)
// 3. planReconcile() with only pending *revenue* reserved, so ordinary surplus flows Senior-gap-first against the
//    already-restored Senior, i.e. to Junior
// 4. apply pending revenue with continuation recomputed (existing MED-12 logic)
```

Losses stay order-independent: a loss applied after the recapitalization gives the same final state as today. Only gains change owner, back to the chronological result, and genuinely oversized recapitalizations still go to `unassignedAssets`. A narrower mitigation for recorded inflows is to queue `Revenue` whenever a recapitalization is pending (`seniorPrincipal + juniorPrincipal == 0 || pendingRecapitalizationAssets != 0`), so that the MED-12 helper sees both. That alone does not cover mark-to-market gains. Add a regression for `0 < S < H`, `J = 0`, recapitalization `G`, then revenue `T` before settlement (`T < G`, `T = G`, `T > G`), expecting `unassignedAssets == 0`, `juniorPrincipal == T` and `canSettleDepositEntries() == true`.

## Impact Details

All figures come from the attached PoC on a ~1M USDG pool. The core displacement is exactly `min(G, T)` (asserted over 256 fuzz runs), so it scales with the Senior gap and with the revenue or favourable mark move that accrues before reconciliation. The concrete figures are scenario-specific because of six-decimal rounding, the fixed seeds, the fixed 1,000 USDG exit and VPI depth.

1. **Junior LPs lose revenue that the control ordering gives them.**
   - `T < G`: Junior principal stays 0 although Junior earned 420.790409. `unassignedAssets` rises by the same amount; in the control Junior receives 421.211619.
   - `T > G`: Junior principal is 364.044744 versus 564.210060 in the control, and the entire 199.600202 recapitalization is ownerless.
   - With no later trade, a favourable 0.25-percentage-point basket retracement worth 500 USDG to LPs gives the same result: Junior 0, unassigned 500.
2. **The recovery action does nothing for Junior.** With revenue first, Junior's result equals the no-recapitalization counterfactual. For `T > G`: 364.044744 with the recapitalization versus 364.157451 without it (the 0.11 difference is the VPI effect of the extra pool depth). Senior is whole in both, and 199.600202 of owner cash sits ownerless.
3. **Deposits close.** `unassignedAssets > 0` makes `HousePoolTrancheGateLib.trancheDepositsAllowed()` false. `maxRequestDeposit()` returns 0 for both vaults and `canSettleDepositEntries()` stays false until the owner intervenes, although the recapitalization had reopened the projected gate. In the control, Junior deposits reopen as soon as the revenue is reconciled. In the `T < G` regime Junior is also reported as terminally wiped (`totalSupply > 0`, `totalAssets == 0`) although it earned the revenue.
4. **Senior loses first-loss protection.** `planReconcile()` caps `unassignedAssets` only by total distributable value, so later losses hit Junior, then Senior, before ownerless value. After a further 0.12% adverse move, Senior is impaired again by 122.031116 USDG while 199.600202 USDG sits untouched in `unassignedAssets`. In the control, Junior absorbs the same loss and keeps 87.114152.
5. **The built-in cure does not automatically restore incumbent holders.** `assignUnassignedAssets()` is owner-only and mints new shares for one receiver at the current price. In the PoC the incumbent Junior LP's value is unchanged by the assignment (364.042925 before, 364.040397 after, one second of coupon), and the receiver obtains 199.600201. When Junior principal is zero (`T < G`), Senior assignment reverts for lack of capacity, and the Junior assignment mints at the wiped price: the receiver obtains essentially the whole Junior tranche, and the incumbent keeps 2 parts per billion of all future Junior distributions. Restoring incumbents pro rata requires governance to pick them as receivers or to distribute off-protocol. Without assignment, later ordinary gains do not clear the bucket, so new deposit requests and activation stay blocked while it persists.

**Severity: High (Permanent freezing of unclaimed yield).** The displaced amount is yield that belongs to the surviving Junior holders under the chronological allocation and under the pool's own canonical view when it arrives. After reconciliation it sits in `unassignedAssets`, from which no LP can redeem or claim. No permissionless or automatic mechanism returns it to them: later revenue does not clear the bucket, and the only release is an owner-only mint of new shares to a single receiver, which does not restore incumbents pro rata. Triggering it costs nothing. It happens whenever revenue reaches the pool before the next settlement, and any keeper can force it at gas cost by executing a third party's revenue-producing order before `settleLpEpoch()`. The damage is hundreds of USDG per event on a 1M pool and grows with the gap and the revenue. The same events also remove Senior's first-loss cushion and close both tranches to deposits. Value is not destroyed, and governance can compensate incumbents deliberately. The claim is that the protocol's own accounting removes this yield from its owners' claim path.

### Why this is not a known issue

- MED-12's fix and regression cover only revenue in the pending bucket (total wipe). LOW-22's comment describes deferral "under existing senior-recovery priority" and leaves "residual value without a claimant path" unassigned. The value displaced here has a claimant path and is not residual (see **Specification conflict**).
- SECURITY.md "LP accounting limitations" and "LP claim and allocation limitations" do not mention it, and no other public report covers it. MED-11, MED-24 and MED-29 are not involved. The PoC uses an ordinary funded Senior exit only as settlement queue work. The displacement is fixed by the reconciliation that `_settleLpEpoch()` runs before it processes any queue item, so the type of matured request is irrelevant.

## References

- [`HousePool.recordClaimantInflow()`: revenue queued only when both principals are zero](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L615-L651)
- [`HousePool._reconcile()`: ordinary plan before pending buckets](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1555-L1599)
- [`HousePool._applyPendingClaimantBucketsLive()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1759-L1776)
- [`HousePool.canSettleDepositEntries()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L505-L519) and [`_projectedEntriesMaySettle()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1120-L1137)
- [`HousePoolReconcilePlanLib.planReconcile()`: pending reserved, surplus Senior-gap-first, unassigned capped only by distributable](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L47-L106)
- [`HousePoolPendingPreviewLib`: "Recapitalization is always processed before revenue" and the MED-12 continuation fix](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L6-L57)
- [`HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent()`: remainder to unassigned](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L103-L137)
- [`HousePoolTrancheGateLib.trancheDepositsAllowed()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolTrancheGateLib.sol#L19-L41)
- [`HousePool.assignUnassignedAssets()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L653-L704)
- [`CfdEngine.recapitalizePool()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L627-L653)
- [`TrancheVault.totalAssets()` uses the reconcile-first projection](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L510-L513)
- [ACCOUNTING_SPEC: no-op settlement rule, required consequences and waterfall steps](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L300-L357)
- [ACCOUNTING_SPEC: ownership routing for pool inflows](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L628-L652)
- [SECURITY.md: owner-funded recapitalization](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L155-L163)
- [SECURITY.md: no-op checkpoint grief and execution-frequency dependence](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L908-L911)
- [WHITEPAPER: no-op settlement cannot change time-based accounting](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L700-L708)
- [WHITEPAPER §4.5: recapitalization and ownership routing](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L759-L775)
- [MED-12 regression: "Recapitalization restores senior first; same-batch revenue then belongs to surviving junior owners"](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/differential/HousePoolSnapshotParity.t.sol#L135-L153)
- [Project test using the partially impaired state for recapitalization](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/admin/EngineRecapitalization.t.sol#L33-L64)
- [Cecuro AI audit report (MED-12, LOW-22)](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)


## Proof of Concept

## Proof of Concept

The test below reproduces the issue against unmodified code at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. It runs on the repository's `BasePerpTest` suite with the DEPLOYMENT.md "Initial release parameters": the release Engine `RiskParams` and 1 USDG seeds are passed to the suite deployment, and the release values that differ from the suite defaults are finalized through the owner timelocks. `setUp()` then asserts the 0.04% execution fee, 0.25% settlement buffer, 0.50% frozen-close spread, 10-bps basket confidence ratio, 2,500-bps adverse-confidence multiplier, $1,000 minimum open notional, 8% Senior rate, and 40,000,000 USDG / 80% Senior limits.

Test doubles and harness, disclosed in full:

- Pyth (out of scope) is the suite's `MockPyth`, fed deterministic two-feed equal-weight baskets with zero confidence. The release uses six feeds; the effect depends only on the aggregate mark.
- USDG is the suite's `MockSettlementToken`.
- `BasePerpTest` deploys `LegacyOrderRouterHarness`. This test calls only its inherited, unmodified production V3 and keeper paths.

Every in-scope protocol state change goes through a public entrypoint:

- LP liquidity: `TrancheVault.requestDeposit` / `claimDeposit` / `requestRedeem` / `withdraw`;
- trading: the inherited, unmodified V3 `OrderRouter.commitOrder(OrderRequest)` and permissionless `OrderRouter.executeOrder`;
- LP settlement and marks: permissionless `OrderRouter.settleLpEpoch` and `OrderRouter.updateMarkPrice`, called by an ordinary keeper address;
- trusted, documented owner actions: `CfdEngine.recapitalizePool` (exactly the observed Senior gap), `HousePool.assignUnassignedAssets`, and, in `setUp()`, the timelocked Router and pool configuration.

No protocol contract is impersonated. One test (`test_DiagnosticPendingHelperGivesTheSameTwoAmountsToJunior`) also calls the pure library helper directly as a diagnostic comparison; that call changes no state.

Scenario: a ~1M USDG pool at the 80% Senior cap (200,000 Junior, 800,000 Senior) goes through a low-revenue period (1,188 days compressed into one interval) in which the Junior-funded 8% Senior coupon exhausts Junior principal. A trader holding a SHORT of 20% of depth then gains from a 0.2% or 1.0% basket move, which partially impairs Senior (`0 < S < H`, `J = 0`, Junior shares alive). The owner recapitalizes the gap. Then, while an ordinary 1,000 USDG Senior exit sits matured in the queue, the test compares two orderings of the same later events:

- **revenue first:** another trader's skew-increasing open pays VPI to the pool, then a keeper settles;
- **settle first (control):** the keeper settles, then the same open executes, then the next epoch is settled.

`T` differs by about 0.1% between the orderings because the control funds the 1,000 USDG exit before the trade, which slightly changes VPI depth. Every assertion uses the `T` measured in its own branch. The primary comparison is at the first reconciliation that includes `T`. The "final ..." lines are read after the second settlement of each branch, i.e. at the same time. In the revenue-first branch Junior already holds principal during the epoch between its two settlements and pays about 9.10 USDG of Senior coupon on it; in the control, Junior's stored principal is still zero during that epoch. The final-value gap (354.940460 vs 564.207241) therefore includes that coupon-timing difference on top of the displacement (364.044744 vs 564.210060 at the first reconciliation that includes `T`).

| Test | What it proves |
|---|---|
| `test_LaterRevenueDisplacesPendingRecapitalizationIntoUnassignedInsteadOfJunior` | 1.0% shock (`G` = 1,797.998603 > `T`): the later revenue fills the Senior gap, the same amount of the recapitalization becomes ownerless, and Junior receives **none** of the 420.790409 earned after the recapitalization. All deposits stay closed. Control: Junior 421.211619, unassigned 0, Junior deposits open. Also asserts that the pool's reconcile-first view reports Senior fully restored before the revenue arrives. |
| `test_EntireRecapitalizationBecomesOwnerlessWhenLaterRevenueExceedsTheGap` | 0.2% shock (`G` = 199.600202 < `T`): the **entire** recapitalization becomes ownerless, and Junior keeps 364.044744 instead of 564.210060. |
| `test_LpRevenueClosesTheDepositGateThatTheRecapitalizationReopened` | Canonical view: after the recapitalization, Senior is reported at its high-water mark and `canSettleDepositEntries()` is `true`. The arrival of `T` leaves reported Senior unchanged, credits Junior with only `T - G`, and turns the gate `false`. |
| `test_WithRevenueFirstJuniorEndsAsIfTheOwnerHadNeverRecapitalized` | Counterfactual without any recapitalization. With revenue first, Junior gets `T - G` (T > G) or 0 (T < G) in both cases, and the recapitalization only adds ownerless assets. |
| `test_DiagnosticPendingHelperGivesTheSameTwoAmountsToJunior` | Diagnostic: the MED-12-fixed helper applied to the same two amounts (as if the revenue were queued, as after a total wipe) gives `Junior = T, unassigned = 0`. The live reconcile gives `unassigned = min(G, T)`. |
| `test_LaterMarkToMarketGainAlsoPreemptsTheRecapitalization` | No later revenue-producing trade: a 0.25-percentage-point basket retracement (500 USDG gain to LPs) gives Junior 0 and unassigned 500 if it is settled before the recapitalization is persisted. |
| `test_DisplacedValueStopsProtectingSeniorFromTheNextLoss` | After a further 0.12% adverse move, Senior is impaired again by 122.031116 USDG while 199.600202 USDG sits in `unassignedAssets`. Control: Junior absorbs the loss and keeps 87.114152. |
| `test_OwnerAssignmentReopensDepositsButDoesNotRestoreIncumbentJuniorValue` | Deposits reopen only after the owner-only assignment, which gives the displaced 199.60 USDG to the owner-chosen receiver. The incumbent Junior LP stays at 364.04 USDG. |
| `test_OwnerCureAtZeroJuniorPrincipalHandsTheJuniorTrancheToTheReceiver` | When `T < G`, Senior assignment reverts (no capacity), and the Junior assignment mints at the wiped price: the incumbent keeps 2 ppb of the Junior tranche. |
| `testFuzz_DisplacementEqualsMinOfGapAndLaterRevenue` | 256 runs over shock size (0.15%-1.2%), later notional (10,000-190,000 USDG) and ordering: `unassignedAssets == min(G, T)` and `juniorPrincipal == T - min(G, T)` when revenue precedes the settlement; `0` and `T` otherwise. |

### How to run

Save the file below as `packages/perps/test/poc/PartialImpairmentRecapitalizationOrdering.t.sol` and run, from the repository root:

```bash
forge test --root packages/perps --match-path test/poc/PartialImpairmentRecapitalizationOrdering.t.sol -vv
```

### Test file

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {HousePoolPendingPreviewLib} from "@plether/perps/libraries/HousePoolPendingPreviewLib.sol";

/// @notice After a partial Senior impairment the owner's documented recapitalization is only queued. LP revenue that
///         reaches the pool before the next LP settlement is applied to the Senior gap first, and the matching part
///         of the pending recapitalization is then classified as ownerless `unassignedAssets`. Surviving Junior
///         shareholders receive min(gap, revenue) less than when a settlement runs between the two inflows.
///         Protocol state changes use public entrypoints only: vault requests and claims, the inherited, unmodified
///         V3 `OrderRouter.commitOrder(OrderRequest)` / `executeOrder`, permissionless `settleLpEpoch` /
///         `updateMarkPrice`, and the owner's documented `recapitalizePool` and `assignUnassignedAssets`. Pyth and
///         USDG are the suite's standard test doubles (MockPyth, MockSettlementToken); the suite deploys its
///         LegacyOrderRouterHarness, and only its inherited V3 and keeper paths are called.
contract PartialImpairmentRecapitalizationOrderingTest is BasePerpTest {

    address internal constant JUNIOR_LP = address(0x1A0001);
    address internal constant SENIOR_LP = address(0x5E0001);
    address internal constant EXITING_SENIOR_LP = address(0x5E0002);
    address internal constant SHOCK_TRADER = address(0xA11CE);
    address internal constant LATER_TRADER = address(0xB0B);
    address internal constant KEEPER = address(0xBEEF);
    address internal constant PROSPECTIVE_LP = address(0x1A0002);
    address internal constant OWNER_CHOSEN_RECEIVER = address(0x0A55);

    uint256 internal constant JUNIOR_ASSETS = 200_000e6;
    uint256 internal constant SENIOR_ASSETS = 790_000e6;
    uint256 internal constant EXITING_SENIOR_ASSETS = 10_000e6;
    uint256 internal constant EXIT_SLICE_ASSETS = 1000e6;

    /// @dev ~3.25 years in which LP revenue does not cover the Junior-funded 8% Senior coupon.
    uint256 internal constant LOW_REVENUE_PERIOD = 1188 days;
    uint256 internal constant SHOCK_SIZE = 200_000e18; // 20% of pool depth
    uint256 internal constant SHOCK_MARGIN = 10_000e6;
    uint256 internal constant LARGE_GAP_MARK = 1.01e8; // 1.0% basket move against the pool
    uint256 internal constant SMALL_GAP_MARK = 1.002e8; // 0.2% basket move against the pool
    uint256 internal constant LATER_MARGIN = 10_000e6;

    uint256 internal clientNonce;
    uint256 internal pendingExitRequest;

    struct Outcome {
        uint256 gap;
        uint256 laterRevenue;
        uint256 highWaterMarkBeforeRevenue;
        uint256 projectedSeniorBeforeRevenue;
        uint256 projectedJuniorBeforeRevenue;
        uint256 projectedJuniorAfterRevenue;
        uint256 juniorAtFirstSettlementWithRevenue;
        uint256 unassignedAtFirstSettlementWithRevenue;
        uint256 senior;
        uint256 junior;
        uint256 highWaterMark;
        uint256 unassigned;
        uint256 pendingRecapitalization;
        uint256 pendingRevenue;
        uint256 juniorLpValue;
        bool depositsOpen;
    }

    /// @dev The same later revenue settled once with and once without the owner's recapitalization.
    struct Counterfactual {
        uint256 gap;
        uint256 revenueWithout;
        uint256 juniorWithout;
        uint256 unassignedWithout;
        uint256 seniorGapWithout;
        uint256 revenueWith;
        uint256 juniorWith;
        uint256 unassignedWith;
        uint256 seniorGapWith;
    }

    // ------------------------------------------------------------------ release configuration

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        // packages/perps/DEPLOYMENT.md "Initial release parameters".
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6; // release seed: 1 USDG per vault
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0; // funded below through public vault requests and Router settlement
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        // Release Router values that differ from the suite defaults ($1,000 minimum open notional, 2,500-bps
        // adverse-confidence multiplier) are proposed together with the release Senior limits (40M USDG, 80% share);
        // both are finalized after the same 72-hour owner timelock.
        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        routerConfig.minOpenNotionalAssets = 1000e6;
        routerConfig.adverseConfidenceMultiplierBps = 2500;
        routerAdmin.proposeRouterConfig(routerConfig);
        _setSeniorCapacity(40_000_000e6, 8000);
        routerAdmin.finalizeRouterConfig();
        _assertReleaseConfiguration();

        _fundTranche(juniorVault, JUNIOR_LP, JUNIOR_ASSETS);
        _fundTranche(seniorVault, SENIOR_LP, SENIOR_ASSETS);
        _fundTranche(seniorVault, EXITING_SENIOR_LP, EXITING_SENIOR_ASSETS);
        // A few hours of Senior coupon accrue while the deposit epochs activate.
        assertApproxEqAbs(pool.seniorPrincipal(), SENIOR_ASSETS + EXITING_SENIOR_ASSETS + 1e6, 100e6, "Senior funded");
        assertApproxEqAbs(pool.juniorPrincipal(), JUNIOR_ASSETS + 1e6, 100e6, "Junior first-loss capital");
    }

    /// @dev DEPLOYMENT.md "Initial release parameters". The suite's two-feed equal-weight mock basket stands in for
    ///      the six release feeds; the accounting below depends only on the aggregate mark.
    function _assertReleaseConfiguration() internal view {
        assertEq(router.minOpenNotionalAssets(), 1000e6, "release minimum open notional");
        assertEq(router.pletherOracle().adverseConfidenceMultiplierBps(), 2500, "release adverse-confidence multiplier");
        assertEq(router.pletherOracle().basketMaxConfidenceRatioBps(), 10, "release basket confidence ratio");
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(engine.settlementBufferBps(), 25, "release settlement buffer");
        assertEq(engine.frozenCloseSpreadBps(), 50, "release frozen-close spread");
        assertEq(pool.seniorRateBps(), 800, "release Senior coupon rate");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release Senior exposure limit");
        assertEq(pool.maxSeniorShareBps(), 8000, "release Senior share limit");
    }

    // ------------------------------------------------------------------ tests

    /// @dev Later revenue smaller than the recapitalized gap: it fills the Senior gap, the same amount of the pending
    ///      recapitalization becomes ownerless, and the surviving Junior owners receive none of the later revenue.
    function test_LaterRevenueDisplacesPendingRecapitalizationIntoUnassignedInsteadOfJunior() public {
        (Outcome memory revenueFirst, Outcome memory settlementFirst) = _compareOrderings(LARGE_GAP_MARK, 1500);
        _logOutcomes(revenueFirst, settlementFirst);
        assertLt(revenueFirst.laterRevenue, revenueFirst.gap, "regime: later revenue below the gap");
        assertEq(revenueFirst.juniorAtFirstSettlementWithRevenue, 0, "Junior receives none of the later revenue");
        assertEq(
            revenueFirst.unassignedAtFirstSettlementWithRevenue,
            revenueFirst.laterRevenue,
            "an amount equal to all later revenue becomes ownerless"
        );
    }

    /// @dev Later revenue larger than the recapitalized gap: the owner's entire recapitalization becomes ownerless.
    function test_EntireRecapitalizationBecomesOwnerlessWhenLaterRevenueExceedsTheGap() public {
        (Outcome memory revenueFirst, Outcome memory settlementFirst) = _compareOrderings(SMALL_GAP_MARK, 1900);
        _logOutcomes(revenueFirst, settlementFirst);
        assertGt(revenueFirst.laterRevenue, revenueFirst.gap, "regime: later revenue above the gap");
        assertEq(
            revenueFirst.unassignedAtFirstSettlementWithRevenue, revenueFirst.gap, "the whole recapitalization is parked"
        );
    }

    /// @dev Diagnostic comparison, not a state transition: the MED-12-fixed pending helper applied to the same two
    ///      amounts, i.e. the result the pool would reach if the later revenue were queued as a pending revenue
    ///      bucket, as it is after a total wipe.
    function test_DiagnosticPendingHelperGivesTheSameTwoAmountsToJunior() public {
        _reachPartiallyImpairedSenior(SMALL_GAP_MARK);
        uint256 gap = pool.seniorHighWaterMark() - pool.seniorPrincipal();
        _ownerRecapitalizes(gap);
        _queueExitAndMature();

        // The later revenue arrives before any settlement; no keeper choice is involved.
        uint256 revenue = _laterRevenueTrade(1900);
        assertEq(pool.pendingTradingRevenueAssets(), 0, "revenue is not queued as a pending bucket because S > 0");

        // The MED-12-fixed helper applied to the same two amounts: recapitalization first, then revenue.
        HousePoolPendingPreviewLib.PendingAccountingState memory helper;
        helper.waterfall.seniorPrincipal = pool.seniorPrincipal();
        helper.waterfall.juniorPrincipal = pool.juniorPrincipal();
        helper.waterfall.seniorHighWaterMark = pool.seniorHighWaterMark();
        helper.unassignedAssets = pool.unassignedAssets();
        helper.seniorSupply = seniorVault.totalSupply();
        helper.juniorSupply = juniorVault.totalSupply();
        HousePoolPendingPreviewLib.ClaimantPendingBuckets memory both =
            HousePoolPendingPreviewLib.ClaimantPendingBuckets(gap, revenue);
        HousePoolPendingPreviewLib.applyPendingClaimantBucketsPreview(helper, both, both, true);

        // The live reconciliation of the same inflows through a permissionless Router settlement.
        _settleThroughRouter(engine.lastMarkPrice());
        _claimExit();

        emit log_named_uint("recapitalized Senior gap", gap);
        emit log_named_uint("later revenue", revenue);
        emit log_named_uint("pending helper: Junior principal", helper.waterfall.juniorPrincipal);
        emit log_named_uint("pending helper: unassigned assets", helper.unassignedAssets);
        emit log_named_uint("live reconcile: Junior principal", pool.juniorPrincipal());
        emit log_named_uint("live reconcile: unassigned assets", pool.unassignedAssets());

        uint256 displaced = _min(gap, revenue);
        assertEq(helper.waterfall.seniorPrincipal, helper.waterfall.seniorHighWaterMark, "helper restores Senior");
        assertEq(helper.unassignedAssets, 0, "helper: nothing ownerless");
        assertEq(helper.waterfall.juniorPrincipal, revenue, "helper: Junior owns the later revenue");
        assertEq(pool.seniorPrincipal(), pool.seniorHighWaterMark(), "live: Senior restored as well");
        assertEq(pool.unassignedAssets(), displaced, "live: min(gap, revenue) becomes ownerless");
        assertEq(pool.juniorPrincipal(), revenue - displaced, "live: Junior loses exactly that amount");
    }

    /// @dev The pool's canonical projection (used for vault share pricing and the deposit-activation gate). After the
    ///      recapitalization it reports Senior whole and reopens deposit activation. The arrival of LP revenue then
    ///      leaves reported Senior unchanged, credits Junior with only T - G, and closes the gate again because the
    ///      projection now contains ownerless assets.
    function test_LpRevenueClosesTheDepositGateThatTheRecapitalizationReopened() public {
        _reachPartiallyImpairedSenior(SMALL_GAP_MARK);
        assertFalse(pool.canSettleDepositEntries(), "impaired Senior: deposit activation closed");
        uint256 gap = pool.seniorHighWaterMark() - pool.seniorPrincipal();
        _ownerRecapitalizes(gap);
        _queueExitAndMature();
        _refreshMark();

        bool gateAfterRecapitalization = pool.canSettleDepositEntries();
        uint256 seniorAfterRecapitalization = seniorVault.totalAssets();
        uint256 juniorAfterRecapitalization = juniorVault.totalAssets();
        uint256 revenue = _laterRevenueTrade(1900);
        bool gateAfterRevenue = pool.canSettleDepositEntries();
        uint256 seniorAfterRevenue = seniorVault.totalAssets();
        uint256 juniorAfterRevenue = juniorVault.totalAssets();

        emit log_named_uint("recapitalized Senior gap G", gap);
        emit log_named_uint("reported Senior after recapitalization", seniorAfterRecapitalization);
        emit log_named_uint("reported Junior after recapitalization", juniorAfterRecapitalization);
        emit log_named_uint("deposit activation open after recapitalization (1 = yes)", gateAfterRecapitalization ? 1 : 0);
        emit log_named_uint("later LP revenue T", revenue);
        emit log_named_uint("reported Senior after T", seniorAfterRevenue);
        emit log_named_uint("reported Junior after T", juniorAfterRevenue);
        emit log_named_uint("deposit activation open after T (1 = yes)", gateAfterRevenue ? 1 : 0);

        assertEq(seniorAfterRecapitalization, pool.seniorHighWaterMark(), "recapitalization: Senior reported whole");
        assertEq(juniorAfterRecapitalization, 0, "recapitalization: Junior reported at zero");
        assertTrue(gateAfterRecapitalization, "recapitalization reopens projected deposit activation");
        assertGt(revenue, gap, "regime: later revenue above the gap");
        assertEq(seniorAfterRevenue, seniorAfterRecapitalization, "T does not change reported Senior");
        assertEq(juniorAfterRevenue, revenue - gap, "Junior is credited with only T - G of the T it earned");
        assertFalse(gateAfterRevenue, "LP revenue closes projected deposit activation again");
    }

    /// @dev Counterfactual without any recapitalization. When the later revenue reaches the pool before the next LP
    ///      settlement, Junior ends with exactly what it would have had if the owner had never recapitalized.
    function test_WithRevenueFirstJuniorEndsAsIfTheOwnerHadNeverRecapitalized() public {
        uint256 snapshot = vm.snapshotState();
        Counterfactual memory small = _withAndWithoutRecapitalization(SMALL_GAP_MARK, 1900);
        assertTrue(vm.revertToState(snapshot));
        Counterfactual memory large = _withAndWithoutRecapitalization(LARGE_GAP_MARK, 1500);
        _logCounterfactual("T > G", small);
        _logCounterfactual("T < G", large);

        // T > G: the recapitalization changes nothing for any claimant; all of it becomes ownerless.
        assertGt(small.revenueWith, small.gap, "regime: T above the gap");
        assertEq(small.seniorGapWithout, 0, "no recapitalization: T alone restores Senior");
        assertEq(small.juniorWithout, small.revenueWithout - small.gap, "no recapitalization: Junior keeps T - G");
        assertEq(small.unassignedWithout, 0, "no recapitalization: nothing ownerless");
        assertEq(small.seniorGapWith, 0, "recapitalized: Senior whole");
        assertEq(small.juniorWith, small.revenueWith - small.gap, "recapitalized: Junior still keeps only T - G");
        assertEq(small.unassignedWith, small.gap, "recapitalized: the whole recapitalization is ownerless");

        // T < G: Junior keeps nothing either way; an amount equal to T of the recapitalization becomes ownerless.
        assertLt(large.revenueWith, large.gap, "regime: T below the gap");
        assertEq(large.juniorWithout, 0, "no recapitalization: Junior keeps nothing");
        assertGt(large.seniorGapWithout, 0, "no recapitalization: Senior stays partly impaired");
        assertEq(large.unassignedWithout, 0, "no recapitalization: nothing ownerless");
        assertEq(large.juniorWith, 0, "recapitalized: Junior still keeps nothing");
        assertEq(large.seniorGapWith, 0, "recapitalized: Senior whole");
        assertEq(large.unassignedWith, large.revenueWith, "recapitalized: an amount equal to T is ownerless");
    }

    function test_OwnerAssignmentReopensDepositsButDoesNotRestoreIncumbentJuniorValue() public {
        _reachPartiallyImpairedSenior(SMALL_GAP_MARK);
        uint256 gap = pool.seniorHighWaterMark() - pool.seniorPrincipal();
        _ownerRecapitalizes(gap);
        _queueExitAndMature();
        uint256 revenue = _laterRevenueTrade(1900);
        _settleThroughRouter(engine.lastMarkPrice());
        _claimExit();

        uint256 unassigned = pool.unassignedAssets();
        uint256 incumbentBefore = juniorVault.convertToAssets(juniorVault.balanceOf(JUNIOR_LP));
        assertEq(unassigned, _min(gap, revenue), "ownerless value exists");
        assertEq(juniorVault.maxRequestDeposit(PROSPECTIVE_LP), 0, "Junior deposit requests closed");
        assertEq(seniorVault.maxRequestDeposit(PROSPECTIVE_LP), 0, "Senior deposit requests closed");
        assertFalse(pool.canSettleDepositEntries(), "deposit activation closed");

        // The built-in cure is the owner-only assignment, which mints new shares to one receiver of the owner's choice.
        _refreshMark();
        pool.assignUnassignedAssets(false, OWNER_CHOSEN_RECEIVER);

        uint256 incumbentAfter = juniorVault.convertToAssets(juniorVault.balanceOf(JUNIOR_LP));
        uint256 receiverValue = juniorVault.convertToAssets(juniorVault.balanceOf(OWNER_CHOSEN_RECEIVER));
        emit log_named_uint("later revenue", revenue);
        emit log_named_uint("unassigned assets before owner assignment", unassigned);
        emit log_named_uint("incumbent Junior LP value before assignment", incumbentBefore);
        emit log_named_uint("incumbent Junior LP value after assignment", incumbentAfter);
        emit log_named_uint("owner-chosen receiver value after assignment", receiverValue);

        assertEq(pool.unassignedAssets(), 0, "assignment clears the bucket");
        assertGt(juniorVault.maxRequestDeposit(PROSPECTIVE_LP), 0, "deposits reopen only after the owner acts");
        // The assignment reconciles first; the only change for the incumbent is one second of Senior coupon.
        assertApproxEqAbs(incumbentAfter, incumbentBefore, 0.01e6, "the assignment does not restore the incumbent");
        assertApproxEqAbs(receiverValue, unassigned, 2, "displaced value goes to the owner-chosen receiver");
    }

    /// @dev No later revenue-producing trade: a later mark move in the pool's favour is ordinary surplus and displaces
    ///      the pending recapitalization in the same way. The settlement itself installs the newer Pyth mark.
    function test_LaterMarkToMarketGainAlsoPreemptsTheRecapitalization() public {
        _reachPartiallyImpairedSenior(LARGE_GAP_MARK);
        uint256 gap = pool.seniorHighWaterMark() - pool.seniorPrincipal();
        _ownerRecapitalizes(gap);
        _queueExitAndMature();
        uint256 recoveredMark = 1.0075e8; // the basket retraces 0.25 percentage points of the 1% move

        uint256 snapshot = vm.snapshotState();
        _settleThroughRouter(recoveredMark);
        _claimExit();
        uint256 unassignedA = pool.unassignedAssets();
        uint256 juniorA = pool.juniorPrincipal();
        assertTrue(vm.revertToState(snapshot));
        _settleThroughRouter(LARGE_GAP_MARK);
        _claimExit();
        _queueExitAndMature();
        _settleThroughRouter(recoveredMark);
        _claimExit();
        uint256 gainToJunior = pool.juniorPrincipal();

        emit log_named_uint("recapitalized Senior gap", gap);
        emit log_named_uint("mark-to-market gain, settled after the recapitalization", gainToJunior);
        emit log_named_uint("gain first: Junior principal", juniorA);
        emit log_named_uint("gain first: unassigned assets", unassignedA);

        assertEq(pool.unassignedAssets(), 0, "control: the gain belongs to Junior");
        assertGt(gainToJunior, 0, "control: Junior receives the gain");
        assertLt(gainToJunior, gap, "regime: gain below the gap");
        assertEq(juniorA, 0, "gain first: Junior receives nothing");
        assertEq(unassignedA, gainToJunior, "gain first: the same amount is parked as ownerless");
    }

    /// @dev Unassigned assets are capped only by distributable equity, so the waterfall charges a later loss to Junior,
    ///      then Senior, before ownerless value. The displaced amount stops protecting Senior.
    function test_DisplacedValueStopsProtectingSeniorFromTheNextLoss() public {
        _reachPartiallyImpairedSenior(SMALL_GAP_MARK);
        uint256 gap = pool.seniorHighWaterMark() - pool.seniorPrincipal();
        _ownerRecapitalizes(gap);
        _queueExitAndMature();
        uint256 lossMark = 1.0032e8; // a further ordinary 0.12% move against the pool

        uint256 snapshot = vm.snapshotState();
        (uint256 seniorGapA, uint256 juniorA, uint256 unassignedA) = _orderingThenLoss(true, lossMark);
        assertTrue(vm.revertToState(snapshot));
        (uint256 seniorGapB, uint256 juniorB, uint256 unassignedB) = _orderingThenLoss(false, lossMark);

        emit log_named_uint("after next loss, REVENUE FIRST: Senior impairment", seniorGapA);
        emit log_named_uint("after next loss, REVENUE FIRST: Junior principal", juniorA);
        emit log_named_uint("after next loss, REVENUE FIRST: unassigned assets", unassignedA);
        emit log_named_uint("after next loss, SETTLE FIRST:  Senior impairment", seniorGapB);
        emit log_named_uint("after next loss, SETTLE FIRST:  Junior principal", juniorB);
        emit log_named_uint("after next loss, SETTLE FIRST:  unassigned assets", unassignedB);

        assertEq(seniorGapB, 0, "control: Junior first-loss capital absorbs the next loss");
        assertGt(juniorB, 0, "control: Junior still has principal");
        assertGt(seniorGapA, 0, "revenue first: the same loss impairs Senior again");
        assertEq(juniorA, 0, "revenue first: Junior exhausted");
        assertEq(unassignedA, gap, "while the ownerless bucket is untouched");
        assertEq(unassignedB, 0, "control: nothing ownerless");
    }

    /// @dev Later revenue below the gap leaves Junior at zero with surviving shares. Deposits stay closed, Senior
    ///      assignment has no capacity, and the remaining built-in cure mints Junior shares at the wiped price.
    function test_OwnerCureAtZeroJuniorPrincipalHandsTheJuniorTrancheToTheReceiver() public {
        _reachPartiallyImpairedSenior(LARGE_GAP_MARK);
        uint256 gap = pool.seniorHighWaterMark() - pool.seniorPrincipal();
        _ownerRecapitalizes(gap);
        _queueExitAndMature();
        uint256 revenue = _laterRevenueTrade(1500);
        _settleThroughRouter(engine.lastMarkPrice());
        _claimExit();

        assertEq(pool.juniorPrincipal(), 0, "Junior earned the revenue but still has zero principal");
        assertEq(pool.unassignedAssets(), revenue, "an amount equal to all of it is ownerless");
        assertEq(juniorVault.totalAssets(), 0, "Junior vault reports zero assets with surviving shares");
        assertEq(juniorVault.maxRequestDeposit(PROSPECTIVE_LP), 0, "Junior deposits closed");
        assertEq(seniorVault.maxRequestDeposit(PROSPECTIVE_LP), 0, "Senior deposits closed");

        _refreshMark();
        vm.expectRevert(IHousePool.HousePool__ExceedsSeniorDepositCapacity.selector); // no Junior subordination
        pool.assignUnassignedAssets(true, OWNER_CHOSEN_RECEIVER);

        uint256 incumbentShares = juniorVault.balanceOf(JUNIOR_LP);
        pool.assignUnassignedAssets(false, OWNER_CHOSEN_RECEIVER);
        uint256 receiverShares = juniorVault.balanceOf(OWNER_CHOSEN_RECEIVER);
        uint256 supply = juniorVault.totalSupply();

        emit log_named_uint("unassigned assets (equal to the later revenue)", revenue);
        emit log_named_uint("incumbent Junior LP shares", incumbentShares);
        emit log_named_uint("owner-chosen receiver shares minted", receiverShares);
        emit log_named_uint("incumbent share of the Junior tranche (ppb)", incumbentShares * 1e9 / supply);
        emit log_named_uint("incumbent Junior LP value after cure", juniorVault.convertToAssets(incumbentShares));
        emit log_named_uint("receiver value after cure", juniorVault.convertToAssets(receiverShares));

        assertLt(incumbentShares * 1e9 / supply, 10, "incumbents keep < 10 ppb of every future Junior distribution");
        assertApproxEqAbs(juniorVault.convertToAssets(receiverShares), revenue, 2, "receiver gets the revenue");
    }

    function testFuzz_DisplacementEqualsMinOfGapAndLaterRevenue(
        uint256 shockMarkSeed,
        uint256 laterLotsSeed,
        bool revenueBeforeSettlement
    ) public {
        // 0.15% to 1.2% adverse basket move; 10,000 to 190,000 USDG of later skew-increasing notional.
        _reachPartiallyImpairedSenior(bound(shockMarkSeed, 1.0015e8, 1.012e8));
        uint256 gap = pool.seniorHighWaterMark() - pool.seniorPrincipal();
        _ownerRecapitalizes(gap);
        _queueExitAndMature();
        uint256 laterLots = bound(laterLotsSeed, 100, 1900);

        uint256 revenue;
        if (revenueBeforeSettlement) {
            revenue = _laterRevenueTrade(laterLots);
            _settleThroughRouter(engine.lastMarkPrice());
            _claimExit();
        } else {
            _settleThroughRouter(engine.lastMarkPrice());
            _claimExit();
            revenue = _laterRevenueTrade(laterLots);
            _queueExitAndMature();
            _settleThroughRouter(engine.lastMarkPrice());
            _claimExit();
        }

        uint256 displaced = _min(gap, revenue);
        assertEq(pool.pendingRecapitalizationAssets(), 0, "recapitalization consumed once");
        assertEq(pool.seniorPrincipal(), pool.seniorHighWaterMark(), "Senior restored");
        if (revenueBeforeSettlement) {
            assertEq(pool.unassignedAssets(), displaced, "displaced = min(gap, later revenue)");
            assertEq(pool.juniorPrincipal(), revenue - displaced, "Junior loses the displaced amount");
        } else {
            assertEq(pool.unassignedAssets(), 0, "control: nothing ownerless");
            assertEq(pool.juniorPrincipal(), revenue, "control: Junior owns all later revenue");
        }
    }

    // ------------------------------------------------------------------ scenario construction

    /// @dev Junior first-loss capital is exhausted by the Junior-funded 8% Senior coupon over a low-revenue period
    ///      (documented waterfall behaviour), then an ordinary adverse move on a 20%-of-depth position creates a
    ///      mark-to-market loss larger than the remaining Junior principal, which partially impairs Senior.
    function _reachPartiallyImpairedSenior(
        uint256 shockMark
    ) internal {
        vm.warp(vm.getBlockTimestamp() + LOW_REVENUE_PERIOD);
        _queueExitAndMature();
        _settleThroughRouter(1e8);
        _claimExit();
        assertEq(pool.juniorPrincipal(), 0, "Senior coupon exhausted Junior principal");
        assertEq(pool.seniorPrincipal(), pool.seniorHighWaterMark(), "coupon alone leaves Senior healthy");
        assertGt(juniorVault.totalSupply(), 0, "Junior owners survive");

        _trade(SHOCK_TRADER, SHOCK_SIZE, SHOCK_MARGIN, 1e8);
        _queueExitAndMature();
        _settleThroughRouter(shockMark);
        _claimExit();
        assertEq(pool.juniorPrincipal(), 0, "loss above Junior principal");
        assertGt(pool.seniorPrincipal(), 0, "Senior partially impaired, not wiped");
        assertLt(pool.seniorPrincipal(), pool.seniorHighWaterMark(), "Senior impaired");
        assertEq(pool.unassignedAssets(), 0, "no ownerless value before the recapitalization");
    }

    function _compareOrderings(
        uint256 shockMark,
        uint256 laterLots
    ) internal returns (Outcome memory revenueFirst, Outcome memory settlementFirst) {
        _reachPartiallyImpairedSenior(shockMark);
        uint256 gap = pool.seniorHighWaterMark() - pool.seniorPrincipal();
        _ownerRecapitalizes(gap);
        _queueExitAndMature();

        uint256 snapshot = vm.snapshotState();
        revenueFirst = _runOrdering(true, gap, laterLots);
        assertTrue(vm.revertToState(snapshot));
        settlementFirst = _runOrdering(false, gap, laterLots);

        uint256 displaced = _min(gap, revenueFirst.laterRevenue);
        // The pool's own reconcile-first view (used by the vaults' totalAssets) already reports Senior as fully
        // restored before the later revenue arrives.
        assertEq(
            revenueFirst.projectedSeniorBeforeRevenue, revenueFirst.highWaterMarkBeforeRevenue, "Senior reported restored"
        );
        assertEq(revenueFirst.projectedJuniorBeforeRevenue, 0, "Junior reported at zero");
        assertEq(
            revenueFirst.projectedJuniorAfterRevenue,
            revenueFirst.laterRevenue - displaced,
            "the newly earned revenue is not reported as Junior value"
        );
        assertEq(settlementFirst.projectedJuniorAfterRevenue, settlementFirst.laterRevenue, "control: it is");

        // Exact comparison at the first reconciliation that includes the later revenue (no coupon in between).
        assertEq(revenueFirst.unassignedAtFirstSettlementWithRevenue, displaced, "min(gap, revenue) parked");
        assertEq(revenueFirst.juniorAtFirstSettlementWithRevenue, revenueFirst.laterRevenue - displaced, "Junior");
        assertEq(settlementFirst.unassignedAtFirstSettlementWithRevenue, 0, "control: nothing parked");
        assertEq(settlementFirst.juniorAtFirstSettlementWithRevenue, settlementFirst.laterRevenue, "control: Junior");

        // State after the same second permissionless settlement in both orderings.
        assertEq(revenueFirst.senior, revenueFirst.highWaterMark, "Senior fully restored");
        assertEq(settlementFirst.senior, settlementFirst.highWaterMark, "Senior fully restored (control)");
        assertEq(revenueFirst.pendingRecapitalization + settlementFirst.pendingRecapitalization, 0, "consumed once");
        assertEq(revenueFirst.pendingRevenue + settlementFirst.pendingRevenue, 0, "no pending revenue");
        assertEq(revenueFirst.unassigned, displaced, "ownerless value persists");
        assertEq(settlementFirst.unassigned, 0, "control has no ownerless value");
        assertLt(revenueFirst.juniorLpValue, settlementFirst.juniorLpValue, "incumbent Junior LP is worse off");
        assertFalse(revenueFirst.depositsOpen, "ownerless value keeps Junior deposit requests closed");
        assertTrue(settlementFirst.depositsOpen, "control reopens Junior deposit requests");
    }

    function _runOrdering(
        bool revenueBeforeSettlement,
        uint256 gap,
        uint256 laterLots
    ) internal returns (Outcome memory out) {
        out.gap = gap;
        if (revenueBeforeSettlement) {
            out.highWaterMarkBeforeRevenue = pool.seniorHighWaterMark();
            out.projectedSeniorBeforeRevenue = seniorVault.totalAssets();
            out.projectedJuniorBeforeRevenue = juniorVault.totalAssets();
            out.laterRevenue = _laterRevenueTrade(laterLots);
            out.projectedJuniorAfterRevenue = juniorVault.totalAssets();
            _settleThroughRouter(engine.lastMarkPrice());
            _claimExit();
            out.juniorAtFirstSettlementWithRevenue = pool.juniorPrincipal();
            out.unassignedAtFirstSettlementWithRevenue = pool.unassignedAssets();
            _queueExitAndMature();
            _settleThroughRouter(engine.lastMarkPrice());
            _claimExit();
        } else {
            _settleThroughRouter(engine.lastMarkPrice());
            _claimExit();
            out.highWaterMarkBeforeRevenue = pool.seniorHighWaterMark();
            out.projectedSeniorBeforeRevenue = seniorVault.totalAssets();
            out.projectedJuniorBeforeRevenue = juniorVault.totalAssets();
            out.laterRevenue = _laterRevenueTrade(laterLots);
            out.projectedJuniorAfterRevenue = juniorVault.totalAssets();
            _queueExitAndMature();
            _settleThroughRouter(engine.lastMarkPrice());
            _claimExit();
            out.juniorAtFirstSettlementWithRevenue = pool.juniorPrincipal();
            out.unassignedAtFirstSettlementWithRevenue = pool.unassignedAssets();
        }
        out.senior = pool.seniorPrincipal();
        out.junior = pool.juniorPrincipal();
        out.highWaterMark = pool.seniorHighWaterMark();
        out.unassigned = pool.unassignedAssets();
        out.pendingRecapitalization = pool.pendingRecapitalizationAssets();
        out.pendingRevenue = pool.pendingTradingRevenueAssets();
        out.juniorLpValue = juniorVault.convertToAssets(juniorVault.balanceOf(JUNIOR_LP));
        out.depositsOpen = juniorVault.maxRequestDeposit(PROSPECTIVE_LP) > 0;
    }

    function _withAndWithoutRecapitalization(
        uint256 shockMark,
        uint256 laterLots
    ) internal returns (Counterfactual memory c) {
        _reachPartiallyImpairedSenior(shockMark);
        c.gap = pool.seniorHighWaterMark() - pool.seniorPrincipal();
        uint256 snapshot = vm.snapshotState();

        // The owner never recapitalizes; the same later revenue is settled by the next LP settlement.
        _queueExitAndMature();
        c.revenueWithout = _laterRevenueTrade(laterLots);
        _settleThroughRouter(engine.lastMarkPrice());
        _claimExit();
        c.juniorWithout = pool.juniorPrincipal();
        c.unassignedWithout = pool.unassignedAssets();
        c.seniorGapWithout = pool.seniorHighWaterMark() - pool.seniorPrincipal();
        assertTrue(vm.revertToState(snapshot));

        // The owner recapitalizes the gap; the later revenue reaches the pool before the next LP settlement.
        _ownerRecapitalizes(c.gap);
        _queueExitAndMature();
        c.revenueWith = _laterRevenueTrade(laterLots);
        _settleThroughRouter(engine.lastMarkPrice());
        _claimExit();
        c.juniorWith = pool.juniorPrincipal();
        c.unassignedWith = pool.unassignedAssets();
        c.seniorGapWith = pool.seniorHighWaterMark() - pool.seniorPrincipal();
    }

    function _logCounterfactual(
        string memory regime,
        Counterfactual memory c
    ) internal {
        emit log_named_uint(string.concat(regime, ": recapitalized Senior gap G"), c.gap);
        emit log_named_uint(string.concat(regime, ", no recapitalization: later revenue T"), c.revenueWithout);
        emit log_named_uint(string.concat(regime, ", no recapitalization: Junior principal"), c.juniorWithout);
        emit log_named_uint(string.concat(regime, ", no recapitalization: Senior impairment"), c.seniorGapWithout);
        emit log_named_uint(string.concat(regime, ", no recapitalization: unassigned assets"), c.unassignedWithout);
        emit log_named_uint(string.concat(regime, ", recapitalized, revenue first: later revenue T"), c.revenueWith);
        emit log_named_uint(string.concat(regime, ", recapitalized, revenue first: Junior principal"), c.juniorWith);
        emit log_named_uint(string.concat(regime, ", recapitalized, revenue first: Senior impairment"), c.seniorGapWith);
        emit log_named_uint(string.concat(regime, ", recapitalized, revenue first: unassigned assets"), c.unassignedWith);
    }

    function _orderingThenLoss(
        bool revenueBeforeSettlement,
        uint256 lossMark
    ) internal returns (uint256 seniorGap, uint256 junior, uint256 unassigned) {
        _runOrdering(revenueBeforeSettlement, pool.pendingRecapitalizationAssets(), 1900);
        _queueExitAndMature();
        _settleThroughRouter(lossMark);
        _claimExit();
        seniorGap = pool.seniorHighWaterMark() - pool.seniorPrincipal();
        junior = pool.juniorPrincipal();
        unassigned = pool.unassignedAssets();
    }

    function _logOutcomes(
        Outcome memory revenueFirst,
        Outcome memory settlementFirst
    ) internal {
        emit log_named_uint("Senior gap G recapitalized by the owner", revenueFirst.gap);
        emit log_named_uint("reported Senior right after recapitalization", revenueFirst.projectedSeniorBeforeRevenue);
        emit log_named_uint("REVENUE FIRST: later revenue T", revenueFirst.laterRevenue);
        emit log_named_uint("REVENUE FIRST: reported Junior after T arrives", revenueFirst.projectedJuniorAfterRevenue);
        emit log_named_uint("REVENUE FIRST: Junior principal after settlement", revenueFirst.juniorAtFirstSettlementWithRevenue);
        emit log_named_uint("REVENUE FIRST: unassigned after settlement", revenueFirst.unassignedAtFirstSettlementWithRevenue);
        emit log_named_uint("SETTLE FIRST:  later revenue T", settlementFirst.laterRevenue);
        emit log_named_uint("SETTLE FIRST:  reported Junior after T arrives", settlementFirst.projectedJuniorAfterRevenue);
        emit log_named_uint("SETTLE FIRST:  Junior principal after settlement", settlementFirst.juniorAtFirstSettlementWithRevenue);
        emit log_named_uint("SETTLE FIRST:  unassigned after settlement", settlementFirst.unassignedAtFirstSettlementWithRevenue);
        emit log_named_uint("final incumbent Junior LP value, revenue first", revenueFirst.juniorLpValue);
        emit log_named_uint("final incumbent Junior LP value, settle first", settlementFirst.juniorLpValue);
        emit log_named_uint("final unassigned assets, revenue first", revenueFirst.unassigned);
        emit log_named_uint("final unassigned assets, settle first", settlementFirst.unassigned);
        emit log_named_uint("Junior deposit requests open, revenue first (1 = yes)", revenueFirst.depositsOpen ? 1 : 0);
        emit log_named_uint("Junior deposit requests open, settle first (1 = yes)", settlementFirst.depositsOpen ? 1 : 0);
    }

    function _laterRevenueTrade(
        uint256 lots
    ) internal returns (uint256 revenue) {
        revenue = _trade(LATER_TRADER, lots * 1e20, LATER_MARGIN, engine.lastMarkPrice());
        assertGt(revenue, 0, "skew-increasing open pays positive VPI to the pool");
    }

    function _ownerRecapitalizes(
        uint256 amount
    ) internal {
        settlementToken.mint(address(this), amount);
        settlementToken.approve(address(engine), amount);
        engine.recapitalizePool(amount); // this test contract is the protocol owner
        assertEq(pool.pendingRecapitalizationAssets(), amount, "recapitalization is queued for reconciliation");
    }

    // ------------------------------------------------------------------ public-path helpers

    function _fundTranche(
        TrancheVault vault,
        address lp,
        uint256 assets
    ) internal {
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), assets);
        uint256 requestId = vault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
        vm.warp(vault.depositEpochStart(requestId));
        _settleThroughRouter(engine.lastMarkPrice() == 0 ? 1e8 : engine.lastMarkPrice());
        uint256 claimable = vault.claimableDepositRequest(requestId, lp);
        assertEq(claimable, assets, "deposit activates");
        vm.prank(lp);
        vault.claimDeposit(requestId, claimable, lp, lp);
    }

    /// @dev An ordinary Senior LP exit: a matured redemption request is the queue work that lets any keeper settle.
    function _queueExitAndMature() internal {
        uint256 shares = seniorVault.convertToShares(EXIT_SLICE_ASSETS);
        vm.prank(EXITING_SENIOR_LP);
        pendingExitRequest = seniorVault.requestRedeem(shares, EXITING_SENIOR_LP, EXITING_SENIOR_LP);
        vm.warp(pool.lpEpochStart(pendingExitRequest));
    }

    function _claimExit() internal {
        if (pendingExitRequest == 0) {
            return;
        }
        uint256 claimable = seniorVault.claimableRedeemAssets(pendingExitRequest, EXITING_SENIOR_LP);
        if (claimable > 0) {
            vm.prank(EXITING_SENIOR_LP);
            seniorVault.withdraw(claimable, EXITING_SENIOR_LP, EXITING_SENIOR_LP);
        }
        pendingExitRequest = 0;
    }

    function _settleThroughRouter(
        uint256 mark
    ) internal {
        bytes[] memory data = _mockPythUpdateData(mark);
        vm.prank(KEEPER);
        router.settleLpEpoch(data);
    }

    function _refreshMark() internal {
        bytes[] memory data = _mockPythUpdateData(engine.lastMarkPrice());
        vm.prank(KEEPER);
        router.updateMarkPrice(data);
    }

    /// @dev Opens a SHORT through the inherited, unmodified V3 commit and permissionless execution.
    /// @return poolGain Increase in canonical pool assets caused by the executed order (its VPI charge).
    function _trade(
        address trader,
        uint256 size,
        uint256 margin,
        uint256 mark
    ) internal returns (uint256 poolGain) {
        _fundTrader(trader, margin + 1000e6); // committed margin plus free settlement for the execution bounty
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("recapitalization-ordering", trader, ++clientNonce)),
            side: CfdTypes.Side.SHORT,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: type(uint256).max, // no slippage bound on the open
            isClose: false,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
                executionWindowSeconds: uint32(window),
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: type(uint256).max,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
        vm.prank(trader);
        uint64 orderId = router.commitOrder(request);

        uint256 poolBefore = pool.totalAssets();
        bytes[] memory data = _mockPythUpdateData(mark);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "public Router executes");
        poolGain = pool.totalAssets() - poolBefore;
    }

    function _min(
        uint256 a,
        uint256 b
    ) internal pure returns (uint256) {
        return a < b ? a : b;
    }

}
```

### Representative output: run 1 (timings are machine-dependent)

```text
Ran 10 tests for test/poc/PartialImpairmentRecapitalizationOrdering.t.sol:PartialImpairmentRecapitalizationOrderingTest
[PASS] testFuzz_DisplacementEqualsMinOfGapAndLaterRevenue(uint256,uint256,bool) (runs: 256, μ: 7386802, ~: 7382664)
[PASS] test_DiagnosticPendingHelperGivesTheSameTwoAmountsToJunior() (gas: 7069718)
Logs:
  recapitalized Senior gap: 199600202
  later revenue: 563644946
  pending helper: Junior principal: 563644946
  pending helper: unassigned assets: 0
  live reconcile: Junior principal: 364044744
  live reconcile: unassigned assets: 199600202

[PASS] test_DisplacedValueStopsProtectingSeniorFromTheNextLoss() (gas: 13363593)
Logs:
  after next loss, REVENUE FIRST: Senior impairment: 122031116
  after next loss, REVENUE FIRST: Junior principal: 0
  after next loss, REVENUE FIRST: unassigned assets: 199600202
  after next loss, SETTLE FIRST:  Senior impairment: 0
  after next loss, SETTLE FIRST:  Junior principal: 87114152
  after next loss, SETTLE FIRST:  unassigned assets: 0

[PASS] test_EntireRecapitalizationBecomesOwnerlessWhenLaterRevenueExceedsTheGap() (gas: 11866740)
Logs:
  Senior gap G recapitalized by the owner: 199600202
  reported Senior right after recapitalization: 998002000002
  REVENUE FIRST: later revenue T: 563644946
  REVENUE FIRST: reported Junior after T arrives: 364044744
  REVENUE FIRST: Junior principal after settlement: 364044744
  REVENUE FIRST: unassigned after settlement: 199600202
  SETTLE FIRST:  later revenue T: 564210060
  SETTLE FIRST:  reported Junior after T arrives: 564210060
  SETTLE FIRST:  Junior principal after settlement: 564210060
  SETTLE FIRST:  unassigned after settlement: 0
  final incumbent Junior LP value, revenue first: 354940460
  final incumbent Junior LP value, settle first: 564207241
  final unassigned assets, revenue first: 199600202
  final unassigned assets, settle first: 0
  Junior deposit requests open, revenue first (1 = yes): 0
  Junior deposit requests open, settle first (1 = yes): 1

[PASS] test_LaterMarkToMarketGainAlsoPreemptsTheRecapitalization() (gas: 6070766)
Logs:
  recapitalized Senior gap: 1797998603
  mark-to-market gain, settled after the recapitalization: 500000000
  gain first: Junior principal: 0
  gain first: unassigned assets: 500000000

[PASS] test_LaterRevenueDisplacesPendingRecapitalizationIntoUnassignedInsteadOfJunior() (gas: 11377275)
Logs:
  Senior gap G recapitalized by the owner: 1797998603
  reported Senior right after recapitalization: 998002000002
  REVENUE FIRST: later revenue T: 420790409
  REVENUE FIRST: reported Junior after T arrives: 0
  REVENUE FIRST: Junior principal after settlement: 0
  REVENUE FIRST: unassigned after settlement: 420790409
  SETTLE FIRST:  later revenue T: 421211619
  SETTLE FIRST:  reported Junior after T arrives: 421211619
  SETTLE FIRST:  Junior principal after settlement: 421211619
  SETTLE FIRST:  unassigned after settlement: 0
  final incumbent Junior LP value, revenue first: 0
  final incumbent Junior LP value, settle first: 421209515
  final unassigned assets, revenue first: 420790409
  final unassigned assets, settle first: 0
  Junior deposit requests open, revenue first (1 = yes): 0
  Junior deposit requests open, settle first (1 = yes): 1

[PASS] test_LpRevenueClosesTheDepositGateThatTheRecapitalizationReopened() (gas: 7322707)
Logs:
  recapitalized Senior gap G: 199600202
  reported Senior after recapitalization: 998002000002
  reported Junior after recapitalization: 0
  deposit activation open after recapitalization (1 = yes): 1
  later LP revenue T: 563644946
  reported Senior after T: 998002000002
  reported Junior after T: 364044744
  deposit activation open after T (1 = yes): 0

[PASS] test_OwnerAssignmentReopensDepositsButDoesNotRestoreIncumbentJuniorValue() (gas: 8129133)
Logs:
  later revenue: 563644946
  unassigned assets before owner assignment: 199600202
  incumbent Junior LP value before assignment: 364042925
  incumbent Junior LP value after assignment: 364040397
  owner-chosen receiver value after assignment: 199600201

[PASS] test_OwnerCureAtZeroJuniorPrincipalHandsTheJuniorTrancheToTheReceiver() (gas: 7692136)
Logs:
  unassigned assets (equal to the later revenue): 420790409
  incumbent Junior LP shares: 200133288636943
  owner-chosen receiver shares minted: 84214589170884087888687
  incumbent share of the Junior tranche (ppb): 2
  incumbent Junior LP value after cure: 0
  receiver value after cure: 420790409

[PASS] test_WithRevenueFirstJuniorEndsAsIfTheOwnerHadNeverRecapitalized() (gas: 19445192)
Logs:
  T > G: recapitalized Senior gap G: 199600202
  T > G, no recapitalization: later revenue T: 563757653
  T > G, no recapitalization: Junior principal: 364157451
  T > G, no recapitalization: Senior impairment: 0
  T > G, no recapitalization: unassigned assets: 0
  T > G, recapitalized, revenue first: later revenue T: 563644946
  T > G, recapitalized, revenue first: Junior principal: 364044744
  T > G, recapitalized, revenue first: Senior impairment: 0
  T > G, recapitalized, revenue first: unassigned assets: 199600202
  T < G: recapitalized Senior gap G: 1797998603
  T < G, no recapitalization: later revenue T: 421548351
  T < G, no recapitalization: Junior principal: 0
  T < G, no recapitalization: Senior impairment: 1375068557
  T < G, no recapitalization: unassigned assets: 0
  T < G, recapitalized, revenue first: later revenue T: 420790409
  T < G, recapitalized, revenue first: Junior principal: 0
  T < G, recapitalized, revenue first: Senior impairment: 0
  T < G, recapitalized, revenue first: unassigned assets: 420790409

Suite result: ok. 10 passed; 0 failed; 0 skipped; finished in 433.95ms (518.40ms CPU time)

Ran 1 test suite in 434.88ms (433.95ms CPU time): 10 tests passed, 0 failed, 0 skipped (10 total tests)
```

### Representative output: run 2 (identical to run 1 apart from timings)

```text
Ran 10 tests for test/poc/PartialImpairmentRecapitalizationOrdering.t.sol:PartialImpairmentRecapitalizationOrderingTest
[PASS] testFuzz_DisplacementEqualsMinOfGapAndLaterRevenue(uint256,uint256,bool) (runs: 256, μ: 7386802, ~: 7382664)
[PASS] test_DiagnosticPendingHelperGivesTheSameTwoAmountsToJunior() (gas: 7069718)
Logs:
  recapitalized Senior gap: 199600202
  later revenue: 563644946
  pending helper: Junior principal: 563644946
  pending helper: unassigned assets: 0
  live reconcile: Junior principal: 364044744
  live reconcile: unassigned assets: 199600202

[PASS] test_DisplacedValueStopsProtectingSeniorFromTheNextLoss() (gas: 13363593)
Logs:
  after next loss, REVENUE FIRST: Senior impairment: 122031116
  after next loss, REVENUE FIRST: Junior principal: 0
  after next loss, REVENUE FIRST: unassigned assets: 199600202
  after next loss, SETTLE FIRST:  Senior impairment: 0
  after next loss, SETTLE FIRST:  Junior principal: 87114152
  after next loss, SETTLE FIRST:  unassigned assets: 0

[PASS] test_EntireRecapitalizationBecomesOwnerlessWhenLaterRevenueExceedsTheGap() (gas: 11866740)
Logs:
  Senior gap G recapitalized by the owner: 199600202
  reported Senior right after recapitalization: 998002000002
  REVENUE FIRST: later revenue T: 563644946
  REVENUE FIRST: reported Junior after T arrives: 364044744
  REVENUE FIRST: Junior principal after settlement: 364044744
  REVENUE FIRST: unassigned after settlement: 199600202
  SETTLE FIRST:  later revenue T: 564210060
  SETTLE FIRST:  reported Junior after T arrives: 564210060
  SETTLE FIRST:  Junior principal after settlement: 564210060
  SETTLE FIRST:  unassigned after settlement: 0
  final incumbent Junior LP value, revenue first: 354940460
  final incumbent Junior LP value, settle first: 564207241
  final unassigned assets, revenue first: 199600202
  final unassigned assets, settle first: 0
  Junior deposit requests open, revenue first (1 = yes): 0
  Junior deposit requests open, settle first (1 = yes): 1

[PASS] test_LaterMarkToMarketGainAlsoPreemptsTheRecapitalization() (gas: 6070766)
Logs:
  recapitalized Senior gap: 1797998603
  mark-to-market gain, settled after the recapitalization: 500000000
  gain first: Junior principal: 0
  gain first: unassigned assets: 500000000

[PASS] test_LaterRevenueDisplacesPendingRecapitalizationIntoUnassignedInsteadOfJunior() (gas: 11377275)
Logs:
  Senior gap G recapitalized by the owner: 1797998603
  reported Senior right after recapitalization: 998002000002
  REVENUE FIRST: later revenue T: 420790409
  REVENUE FIRST: reported Junior after T arrives: 0
  REVENUE FIRST: Junior principal after settlement: 0
  REVENUE FIRST: unassigned after settlement: 420790409
  SETTLE FIRST:  later revenue T: 421211619
  SETTLE FIRST:  reported Junior after T arrives: 421211619
  SETTLE FIRST:  Junior principal after settlement: 421211619
  SETTLE FIRST:  unassigned after settlement: 0
  final incumbent Junior LP value, revenue first: 0
  final incumbent Junior LP value, settle first: 421209515
  final unassigned assets, revenue first: 420790409
  final unassigned assets, settle first: 0
  Junior deposit requests open, revenue first (1 = yes): 0
  Junior deposit requests open, settle first (1 = yes): 1

[PASS] test_LpRevenueClosesTheDepositGateThatTheRecapitalizationReopened() (gas: 7322707)
Logs:
  recapitalized Senior gap G: 199600202
  reported Senior after recapitalization: 998002000002
  reported Junior after recapitalization: 0
  deposit activation open after recapitalization (1 = yes): 1
  later LP revenue T: 563644946
  reported Senior after T: 998002000002
  reported Junior after T: 364044744
  deposit activation open after T (1 = yes): 0

[PASS] test_OwnerAssignmentReopensDepositsButDoesNotRestoreIncumbentJuniorValue() (gas: 8129133)
Logs:
  later revenue: 563644946
  unassigned assets before owner assignment: 199600202
  incumbent Junior LP value before assignment: 364042925
  incumbent Junior LP value after assignment: 364040397
  owner-chosen receiver value after assignment: 199600201

[PASS] test_OwnerCureAtZeroJuniorPrincipalHandsTheJuniorTrancheToTheReceiver() (gas: 7692136)
Logs:
  unassigned assets (equal to the later revenue): 420790409
  incumbent Junior LP shares: 200133288636943
  owner-chosen receiver shares minted: 84214589170884087888687
  incumbent share of the Junior tranche (ppb): 2
  incumbent Junior LP value after cure: 0
  receiver value after cure: 420790409

[PASS] test_WithRevenueFirstJuniorEndsAsIfTheOwnerHadNeverRecapitalized() (gas: 19445192)
Logs:
  T > G: recapitalized Senior gap G: 199600202
  T > G, no recapitalization: later revenue T: 563757653
  T > G, no recapitalization: Junior principal: 364157451
  T > G, no recapitalization: Senior impairment: 0
  T > G, no recapitalization: unassigned assets: 0
  T > G, recapitalized, revenue first: later revenue T: 563644946
  T > G, recapitalized, revenue first: Junior principal: 364044744
  T > G, recapitalized, revenue first: Senior impairment: 0
  T > G, recapitalized, revenue first: unassigned assets: 199600202
  T < G: recapitalized Senior gap G: 1797998603
  T < G, no recapitalization: later revenue T: 421548351
  T < G, no recapitalization: Junior principal: 0
  T < G, no recapitalization: Senior impairment: 1375068557
  T < G, no recapitalization: unassigned assets: 0
  T < G, recapitalized, revenue first: later revenue T: 420790409
  T < G, recapitalized, revenue first: Junior principal: 0
  T < G, recapitalized, revenue first: Senior impairment: 0
  T < G, recapitalized, revenue first: unassigned assets: 420790409

Suite result: ok. 10 passed; 0 failed; 0 skipped; finished in 435.72ms (515.75ms CPU time)

Ran 1 test suite in 436.61ms (435.72ms CPU time): 10 tests passed, 0 failed, 0 skipped (10 total tests)
```

