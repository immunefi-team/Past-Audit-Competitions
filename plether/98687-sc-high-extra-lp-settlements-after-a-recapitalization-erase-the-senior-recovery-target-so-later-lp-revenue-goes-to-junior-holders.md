# \#98687 [SC-High] Extra LP settlements after a recapitalization erase the Senior recovery target, so later LP revenue goes to Junior holders

**Submitted on Nov 9th 2026 at 15:38:13 UTC by @eureka_hunter for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98687
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description


## Brief/Intro

When the pool ends up with trader claims it cannot pay, the owner's recovery action is `CfdEngine.recapitalizePool`. Part of that money only pays the claims. That part stays in `pendingRecapitalizationAssets`, and every later LP settlement treats it as if it were new money again. Each such settlement books a loss that never happened, sets Senior principal to zero for a moment, and reruns the "empty pool" branch, which overwrites `seniorHighWaterMark` with the smaller amount still pending. No USDG moves, but the recovery target the protocol recorded for Senior LPs goes down. Anyone can trigger the settlements through the public `settleLpEpoch`. A Junior LP who does so before new revenue arrives captures LP revenue that the waterfall reserves for Senior. In my PoC, with release parameters, 54,199 USDG of revenue moves from the Senior LP to Junior holders, and the attacker spends no USDG.

The same root cause also moves revenue that is already in the pool. In a second scenario the revenue arrives before the recapitalization is applied. One settlement books it as Senior principal, and the Senior vault reports that value to its holders. A second settlement, with no inflow and no price change, then moves 30,971 USDG of that booked Senior principal to Junior. In this scenario the attacker does not have to race any revenue.

## Vulnerability Details

### How recovery is supposed to work

The pool keeps the LP value of each tranche in `seniorPrincipal` and `juniorPrincipal`. Losses hit Junior first, then Senior. `seniorHighWaterMark` (HWM) is the Senior value that later revenue must restore before Junior gets anything. The docs say this in several places:

- `seniorHighWaterMark` "records the Senior entitlement that future revenue must restore after a loss", and surplus goes "first to restore Senior principal to its high-water mark, then to Junior" ([ACCOUNTING_SPEC.md:332-338](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L332-L338)).
- Invariant: "Senior impairment must be restored before junior extracts surplus" ([SECURITY.md:394](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L394)).
- "recapitalization never accrues to junior principal" ([HousePoolPendingPreviewLib.sol:107](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L107)).

The code applies the waterfall in `distributeRevenue`: it fills the gap up to the HWM first and gives the rest to Junior ([HousePoolWaterfallAccountingLib.sol:140-158](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L140-L158)). So a lower HWM directly means more revenue for Junior.

The SECURITY.md runbook "Explicit terminal-deficit cascade" says what to do when trader claims exceed pool cash: "Recapitalize or clear positions", then clear degraded mode ([SECURITY.md:1499-1511](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1499-L1511)). `recapitalizePool` pulls `R` USDG from the owner and records it as a `Recapitalization` inflow, which the pool adds to `pendingRecapitalizationAssets` ([CfdEngine.sol:632-650](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L632-L650)). The next reconcile turns the pending amount into tranche value.

Symbols used below (all in USDG):

- `D0`: the deficit, i.e. trader claims minus pool cash, just before the recapitalization.
- `R`: the recapitalization.
- `D = R − D0`: the LP value actually left after the claims are covered.

### What goes wrong

Every reconcile (`_reconcile`, [HousePool.sol:1555-1599](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1555-L1599)) runs these steps.

**Step 1. The pending amount is capped by the whole distributable value.**

```solidity
// https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1829-L1834
function _settleablePendingClaimantBuckets(...) internal pure returns (...) {
    return HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, reconcileSnapshot.distributable);
}
```

`distributable` is the full LP equity: pool cash minus trader claims, plus the mark-to-market of open positions. It also contains value that an earlier reconcile already credited to `seniorPrincipal`. The cap does not subtract that value.

**Step 2. That amount is removed from distributable before the existing principal is valued.**

```solidity
// https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L71
if (pendingBucketAssets > 0) {
    snapshot.distributable =
        snapshot.distributable > pendingBucketAssets ? snapshot.distributable - pendingBucketAssets : 0;
}
```

The rest is compared with `seniorPrincipal + juniorPrincipal`. If it is smaller, the difference is booked as a loss through `absorbLoss` ([HousePoolReconcilePlanLib.sol:103](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L103)). The same USDG is now counted twice: once as Senior principal and once as backing for the pending recapitalization. The "loss" sets Senior principal to zero.

**Step 3. With both principals at zero, the bootstrap branch overwrites the HWM.**

```solidity
// https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L117-L122
if (state.seniorSupply > 0) {
    if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
        state.waterfall.seniorPrincipal += remaining;
        state.waterfall.seniorHighWaterMark =
            recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
```

The target passed in is the **current** pending amount ([HousePool.sol:1598](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1598), [HousePoolPendingPreviewLib.sol:47-49](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L47-L49)). On the first reconcile that amount is the full `R`. On later ones it is only the leftover. The branch assigns the HWM and never keeps a larger stored value. Senior principal comes back to the same `D`, so its value does not change, but its target does. Only the settled part is then removed from the pending bucket ([HousePool.sol:1787-1792](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1787-L1792)), and the next reconcile repeats the steps with the remainder.

### The sequence, with the numbers from my PoC

The numbers are measured on-chain in the Foundry test. The trading setup is described in the Proof of Concept section.

| Step | Who acts | Senior principal | Senior HWM | Pending recap |
|---|---|---:|---:|---:|
| Trader claims exceed cash by `D0` = 772,817.967009 | traders | (not yet reconciled) | | 0 |
| Owner recapitalizes `R` = 1,159,226.950513 (1.5 × `D0`), clears degraded mode | owner, runbook | | | 1,159,226.950513 |
| 1st settlement: correct | keeper | 386,408.983504 (= `D`) | **1,159,226.950513** (= `R`) | 772,817.967009 |
| 2nd settlement: nothing changed | attacker | 386,408.983504 | **772,817.967009** | 386,408.983505 |
| 3rd settlement: nothing changed | attacker | 386,408.983504 | **386,408.983505** | 0.000001 |

After two settlements that move no USDG, the Senior target equals current Senior principal, plus one atom of rounding. The protocol now treats Senior as fully restored, even though the owner's recapitalization was meant to restore it to `R`.

The cut happens when `D0 ≥ D`, i.e. `R ≤ 2·D0`. An owner who pays the claims plus up to the same amount again for LPs is inside this range, and the runbook gives no other guidance. My boundary test confirms that `R = 2·D0` still erases the target and `R = 2·D0 + 1 atom` does not.

### Why the existing checks do not stop it

- `capClaimantBuckets` checks that the pending bucket is not larger than all LP equity. It does not check how much of that equity is still unowned.
- `_decreasePendingClaimantBuckets` removes only the part that was settled. The part the deficit absorbed is never marked as used.
- The bootstrap branch checks only "principals are zero". The fake loss from step 2 makes that true.
- `HousePool__NoLpEpochProgress` only requires that some matured LP request is processed. An attacker's own Junior redemption request satisfies it. At zero Junior NAV the request is refunded in shares, so it costs nothing. This limits the attack to one settlement per hourly epoch, which is enough.

### Who can trigger it

`OrderRouter.settleLpEpoch` ([OrderRouter.sol:479](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L479)) is permissionless, and `HousePool.settleLpEpoch` ([HousePool.sol:797](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L797)) is permissionless when no positions are open. The owner only follows the documented runbook. Routine keeper settlements also lower the target: in my reference branch the target falls from `R` to `D0` on the settlement that processes the revenue. A Junior holder who settles right after the recapitalization makes sure the whole target is gone before any revenue arrives.

### Second scenario: revenue already booked as Senior principal

Revenue is queued in `pendingTradingRevenueAssets`, instead of being reconciled as ordinary revenue, only when both stored principals are zero at the moment it is recorded ([HousePool.sol:644-648](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L644-L648)). Both stored principals are zero when a settlement has recorded the deficit loss and no recapitalization has been applied yet.

The owner can reach that state while following the runbook. Once all positions are closed, the maximum liability `L` is 0. `clearDegradedMode` checks only `cash − claims ≥ L`, where the left side is floored at 0 ([CfdEngine.sol:658-667](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L658-L667)). The check therefore passes before any recapitalization, which matches the runbook's "Recapitalize or clear positions", then clear degraded mode. The runbook also says to clear only once raw solvency `E ≥ L` is "genuinely restored". With no positions and unpaid claims, `E` is 0 and `L` is 0, and the code accepts that as restored. The order of events:

1. The owner clears degraded mode. All positions are already closed.
2. A keeper settlement records the deficit loss. Both principals are stored as 0, and the tranche shares survive.
3. The owner recapitalizes `R`, and the short collects its claim.
4. A trader loses money to the pool. Both principals are 0, so the revenue `Rev` is queued in `pendingTradingRevenueAssets`.
5. **Settlement 1 is correct.** `capClaimantBuckets` serves the recapitalization first ([HousePoolPendingPreviewLib.sol:76-81](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L76-L81)). The whole distributable value `R − D0 + Rev`, revenue included, is bootstrapped as Senior principal with HWM `R`. The revenue entry stays in the pending bucket, although its USDG now backs Senior principal.
6. **Settlement 2 is the bug.** Nothing changed, but the pending recapitalization remainder plus the pending revenue again exceed what is left after Senior principal. A phantom loss sets Senior to 0, and the bootstrap branch sets Senior principal and the HWM to the recapitalization remainder. The settled part of the revenue then finds no Senior gap, so it is credited to Junior principal ([HousePoolPendingPreviewLib.sol:168-182](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L168-L182)).

In my PoC (`R = 1.9 × D0`):
- Settlement 1 books 749,663.019182 USDG as Senior principal.
- Settlement 2 lowers Senior principal to 718,691.118135 and raises Junior principal from 0 to 30,971.901047.
- `rawAssets` and the Senior LP's share balance do not change.

Before settlement 1, `seniorVault.totalAssets()` reports 749,663.019182 to Senior holders. It reads the pool's pending preview ([TrancheVault.sol:510-513](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L510-L513)).

Settlement 2 cannot run on a normal redemption request. While the buckets are pending they reserve all free cash, so a request with non-zero value is neither funded nor refunded, and the call reverts `HousePool__NoLpEpochProgress`. A 1-share Junior redemption avoids this: its value rounds to zero, so it is refunded, and the refund counts as progress. Junior shares are transferable after the deposit cooldown, so the attacker moves 1 share to a second address of their own. An ordinary deposit request would also count as progress.

This scenario needs `R` and `Rev` to fall in a window. Write `g = 2·D0 − R`. The main window is `g/2 < Rev ≤ g`, and Junior then receives about `2·Rev − g`, never more than `Rev`. I derived this window from an off-chain model of `_reconcile`. The model reproduces the PoC within 0.5% and shows a few narrower windows for smaller `R`. I have only run the PoC at `R = 1.9 × D0`.

### Suggested fix

1. Cap the pending recapitalization by the part of distributable value that nobody owns yet, i.e. `distributable − (seniorPrincipal + juniorPrincipal + unassignedAssets)`, not by all of `distributable`. Alternatively, when a recapitalization is only partly backed, drop the unbacked part from the bucket once it has been applied, and keep the intent only as the HWM.
2. In the bootstrap branch, never lower an existing `seniorHighWaterMark`: use `max(stored HWM, target, amount)`.

Either change stops the erosion in this scenario. By my reading of the code, either change also stops the second scenario: with fix 1 nothing is settleable once Senior owns all distributable value, and with fix 2 the HWM stays at `R`, so the revenue fills the Senior gap. I have not run the PoC against a patched build.

## Impact Details

**Selected impact: Theft of unclaimed yield (High).**

**Asset:** `HousePool.sol`, which holds LP USDG and keeps the Senior/Junior split, and the Senior `TrancheVault` instance, through which Senior LPs redeem that value.

**What is taken.** LP revenue in this protocol is the money traders lose to the pool. The waterfall gives that revenue to Senior until Senior is back at its HWM. Once the HWM has been erased, the same revenue is credited to Junior principal, and Junior holders can redeem it. The yield had not been paid out to Senior yet when it was redirected, which is why I selected this impact.

**PoC numbers** (same deficit, same recapitalization, same later revenue, same exits):

| | A: revenue settled first | B: attacker settles twice first | Change |
|---|---:|---:|---:|
| Revenue from a later losing trader | 54,199.578245 | 54,199.578245 | |
| Senior LP receives on exit | 440,608.488087 | 386,408.918905 | **−54,199.569182** |
| Attacker (half of Junior) receives | 0 | 27,099.782367 | **+27,099.782367** |
| Other Junior LP receives | 0 | 27,099.779726 | +27,099.779726 |
| Left in pool (seed positions) | 0.073662 | 0.080751 | |
| Total | 440,608.561749 | 440,608.561749 | 0 |

Branch B gives the Senior LP 12.3% less USDG for the same shares. All of it goes to Junior holders; no value is created. The attacker's share is proportional to their Junior balance.

**Second scenario, PoC numbers.** Both branches have the same deficit, the same `R = 1.9 × D0` = 1,468,354.137317, the same losing trade and the same exits. The only difference is whether the trade lands before or after the first keeper settlement after the recapitalization.

| | A: trade after settlement 1 | B: trade before settlement 1 | Change |
|---|---:|---:|---:|
| Revenue from the losing trader | (not queued) | 54,149.064857 | |
| Senior principal booked by settlement 1 | | 749,663.019182 | |
| Senior principal after settlement 2 (no inflow) | | 718,691.118135 | −30,971.901047 |
| Senior LP receives on exit | 749,662.893852 | 718,710.688267 | **−30,952.205585** |
| Attacker (half of Junior) receives | 0 | 15,476.101522 | **+15,476.101522** |
| Other Junior LP receives | 0 | 15,476.079196 | +15,476.079196 |
| Left in pool (seed positions) | 0.125330 | 0.150197 | |
| Total | 749,663.019182 | 749,663.019182 | 0 |

In branch B the Senior LP receives 4.1% less USDG for the same shares. The change at exit is about 20 USDG smaller than the change at settlement 2. The difference is the 8% Senior coupon, which moves a little value from Junior back to Senior in the later epochs. In branch A, Senior ends with exactly the principal that settlement 1 booked in branch B. The test asserts this equality.

**Bound per incident.** After the erosion the HWM sits at about `D`, while the recapitalization intent was `R`. Up to `R − D = D0` of later revenue (772,817 USDG in the PoC) goes to Junior instead of Senior, as revenue comes in. The remaining Senior shortfall is never restored. No setter can raise the HWM again.

**Attacker cost.** Two calls to `settleLpEpoch` at about 225,500 gas each, plus a full-balance Junior redemption request per epoch, which is refunded in shares. My test asserts that the attacker's USDG balance and Junior share balance are unchanged afterwards. The only capital involved is the Junior position the attacker already holds.

**When Junior can take it out.** Junior withdrawals are limited by the 80% Senior share covenant. In the PoC the Senior LP exits first, and then both Junior LPs redeem the diverted value in full.

**Preconditions and feasibility.**

- The pool must reach the documented terminal-deficit state: trader claims above pool cash, with both tranches wiped. The admission check `E ≥ L + B` keeps the pool solvent against moves in one direction, so the PoC reaches this state with a two-sided book and a large round trip of the basket price. This is a tail event, and it is the event the runbook and `recapitalizePool` exist for.
- The owner then recapitalizes with `R ≤ 2·D0`.
- The second scenario also needs this order: the owner clears degraded mode before recapitalizing. The code allows it once positions are closed, and the runbook lists "clear positions" as an alternative to recapitalizing. It also needs `R` and the revenue to fall in the window described above. No attacker timing is needed: settlement 2 gives the same result whoever calls it.

Under the program rules, feasibility affects the reward, not the severity. A documented action by a trusted role that triggers a bug is in scope, and keepers are not trusted.

**Scale.** Plether Perps is not on mainnet yet. The loss scales with the deficit and with the revenue that follows it, so it grows with pool size. Senior capacity at release is 40M USDG.

**Strongest counterargument.** "Senior keeps its current USDG; only future revenue is affected, so this is a missed promised return." The HWM is not a promised return. It is Senior's recorded entitlement to recover lost principal, stored on-chain, and the protocol's own invariant says Junior may not extract surplus before it is restored. The bug erases that recorded entitlement without any value moving. The redirected revenue is then paid out to Junior holders, and the person who triggers the erasure is one of them. The second scenario removes the "future revenue" premise. There, the revenue is already in the pool and already booked as Senior principal, which the Senior vault reports to its holders. It is then moved to Junior without any loss.

**Why I still select High rather than Critical.** In both scenarios Junior is credited only from trading revenue, never from the recapitalization or from Senior's deposits. Junior's gain is therefore bounded by that revenue (the second test asserts `Junior gain ≤ revenue`). Whether revenue that was already booked to Senior still counts as "unclaimed yield" is for triage to judge. I have kept to the narrower impact.

## References

- `_reconcile`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1555-L1599
- `_settleablePendingClaimantBuckets`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1829-L1834
- `capClaimantBuckets`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L72-L82
- `planReconcile` (pending reserved, then `absorbLoss`): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L104
- Bootstrap branch: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L111-L122
- `_decreasePendingClaimantBuckets`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1787-L1792
- `distributeRevenue`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L140-L158
- `recapitalizePool`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L632-L650
- `OrderRouter.settleLpEpoch`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L479
- `recordClaimantInflow` (revenue queued only at zero principal): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L624-L651
- `clearDegradedMode`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L658-L667
- `applyRevenueIntent`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L158-L188
- `TrancheVault.totalAssets` (pending preview): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L510-L513
- SECURITY.md invariant table and terminal-deficit runbook: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L394 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1499-L1511
- ACCOUNTING_SPEC.md waterfall rules: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L330-L340

## Link to Proof of Concept

https://gist.github.com/eureka-finder/6d484ddc271cbffc260147a5371f1207

## Proof of Concept

# Proof of Concept

I recommend to take the POC from Gist Link:  https://gist.github.com/eureka-finder/6d484ddc271cbffc260147a5371f1207

Gist link contains POC test, log files. 

**Environment.** The repository Foundry suite at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The program accepts this suite, and Plether Perps has no mainnet deployment to fork. All risk, fee, router and Senior-capacity values are set to the release values in `packages/perps/DEPLOYMENT.md`. `test_productionConfig` prints and asserts them: `vpiFactor` 0.01e18, `maxSkewRatio` 0.4e18, maintenance 10 bps, initial 20 bps, execution fee 4 bps, settlement buffer 25 bps, Senior cap 40M / 80%, minimum open notional 1,000 USDG, 1 USDG seeds. Only Pyth (`MockPyth`) and USDG (`MockSettlementToken`) are mocked. No contract in scope is changed.

**Actors.**

- LPs, traders and the attacker sign only their own vault and order transactions.
- Orders go through `OrderRouter.commitOrder` and are executed by an unprivileged `keeper` through `OrderRouter.executeOrder` with a Pyth update.
- All LP settlements go through the public `OrderRouter.settleLpEpoch`.
- The test contract is the owner. It uses only `recapitalizePool` and `clearDegradedMode`, as in the runbook.
- No `prank` impersonates a protocol contract, and no storage is written directly.

**Scenario 1** (tests 2–5 below).

1. LPs deposit 6M Senior and 4M Junior (the attacker holds 2M of the Junior).
2. One long and one short trader each open 9M at 1.00 in three steps, which keeps skew under 40%.
3. The basket falls to 0.40 and the long takes 5.4M profit, paid from LP cash. Degraded mode latches.
4. The basket rises to 1.60 and the short takes 5.38M profit. The pool has only 4.61M, so the rest becomes a trader claim: `D0` = 772,817.967009.
5. The owner recapitalizes 1.5 × `D0` and clears degraded mode. The short collects its claim. A keeper settlement applies the recapitalization correctly.
6. Branch A: a later trader loses money to the pool, and keepers settle. Branch B: the attacker runs two settlements first, then the same trade and settlements follow. Every LP then exits.

**Scenario 2** (test 6). It reuses steps 1–4, then:

1. The owner clears degraded mode first, and a keeper settlement records the loss. Both principals are stored as 0.
2. The owner recapitalizes 1.9 × `D0`, and the short collects its claim.
3. The attacker moves 1 Junior share to a second address of their own. Every later settlement processes only that address's 1-share redemption, which is refunded.
4. Branch A: keeper settlement, then the losing trade. Branch B: the losing trade, then keeper settlement 1, then attacker settlement 2. Both branches then settle until nothing is pending, and every LP exits.

**How to run.** The test file is in this Gist:  https://gist.github.com/eureka-finder/6d484ddc271cbffc260147a5371f1207 . 

Save it as `packages/perps/test/perps/p3h01/RecapResidualHwm.t.sol` and run:

```bash
forge test --root packages/perps --match-path 'test/perps/p3h01/RecapResidualHwm.t.sol' -vv
```

The file must sit one folder below `test/perps/` because it imports `../BasePerpTest.sol`. The first via-IR build is slow.

**Tests.**

| # | Test | What it asserts |
|---|---|---|
| 1 | `test_productionConfig` | The release values from `DEPLOYMENT.md` are in force |
| 2 | `test_recovery_firstSettlementIsCorrect` | After the first settlement: Senior = `D`, HWM = `R`, `D0` still pending |
| 3 | `test_attackerSettlements_eraseSeniorTargetWithoutMovingFunds` | Two attacker settlements lower the HWM from `R` to `D0`, then to `D` + 1 atom. Pool USDG, Senior shares, attacker shares and attacker USDG do not change |
| 4 | `test_headline_revenueMovesFromSeniorToJunior` | Same deficit, recapitalization and revenue in two branches. Pinned exit amounts; Senior loss = Junior gain; total conserved |
| 5 | `test_boundary_twiceDeficitErases_oneAtomMoreDoesNot` | Boundary: `R = 2·D0` erases the target, `R = 2·D0 + 1 atom` keeps it |
| 6 | `test_variant_pendingRevenue_bookedSeniorPrincipalMovesToJunior` | Scenario 2. Revenue is queued while both principals are 0. Settlement 1 books it as Senior principal, equal to branch A's final Senior principal. Settlement 2 moves 30,971.901047 from Senior to Junior principal with `rawAssets` unchanged. Pinned exit amounts; total conserved; Junior gain ≤ revenue |

**Result.** `6 passed; 0 failed` (run 2026-11-09, forge 1.5.0 with solc 0.8.35). Key values, in USDG:

| | Scenario 1 | Scenario 2 |
|---|---:|---:|
| Deficit `D0` | 772,817.967009 | 772,817.967009 |
| Recapitalization `R` | 1,159,226.950513 | 1,468,354.137317 |
| Revenue from the losing trader | 54,199.578245 | 54,149.064857 |
| Senior LP exit, branch A | 440,608.488087 | 749,662.893852 |
| Senior LP exit, branch B | 386,408.918905 | 718,710.688267 |
| Attacker exit, branch B (branch A: 0) | 27,099.782367 | 15,476.101522 |
| Other Junior LP exit, branch B (branch A: 0) | 27,099.779726 | 15,476.079196 |
| Senior loss | 54,199.569182 | 30,952.205585 |

In both scenarios the total LP value is the same in the two branches, within the dust left on the seed positions. The full `-vv` log is in the Gist next to the test file.

