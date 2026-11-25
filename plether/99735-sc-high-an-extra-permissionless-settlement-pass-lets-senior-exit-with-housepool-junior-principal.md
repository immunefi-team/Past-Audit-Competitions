# \#99735 [SC-High] An extra permissionless settlement pass lets Senior exit with HousePool Junior principal

**Submitted on Nov 15th 2026 at 02:07:08 UTC by @ChaseTheLight for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99735
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro

`HousePoolWaterfallAccountingLib` applies the two halves of the tranche waterfall in orders that are not inverses of each other. `distributeRevenue` restores impaired Senior principal up to its high-water mark before Junior receives anything (`:147-151`, then `:158`). `absorbLoss` charges a shortfall to Junior first and reaches Senior only once Junior is zero (`:172-183`), and it never writes the high-water mark, which the function's own comment calls deliberate (`:162-163`). Each half on its own is the first-loss ordering the project documents. Their composition is not. Whenever `seniorPrincipal < seniorHighWaterMark` and `juniorPrincipal > 0`, reconciling once more over a closed round trip of the pool's distributable value is not the identity: the upward leg credits Senior and the downward leg debits Junior, so each extra observation moves `min(senior deficit, junior principal, swing)` across the tranche boundary. It moves back only if the pool later earns enough to fill the whole deficit, and value Senior has redeemed in the meantime does not come back. `seniorPrincipal` falls at exactly two sites in the whole package and neither of them pays Junior.

How many times the value is observed is the caller's choice. `OrderRouter.settleLpEpoch(bytes[])` is `external payable nonReentrant` with no caller check, and its own comment reads "Permissionless and available while the router admin is paused" (`OrderRouter.sol:475-484`). It validates a caller-supplied `PoolReconcile` basket, installs that exact mark on the Engine, and invokes the pool's Router-only settlement callback in the same frame, so the pool's authorization branch is satisfied by construction (`OrderLiquidationBatchLogic.sol:549-581` against `HousePool.sol:810-815`). An address with no role, holding Senior tranche shares it bought with an ordinary deposit request while the pool was healthy, takes one settlement pass at a displaced mark and a second after the mark returns to where it started. On the reproduction's arming the Junior tranche ends empty: `juniorPrincipal` ends at zero, `seniorPrincipal` ends higher by the same amount net of whatever the holder's own queued redemption realized along the way, on an arming whose tranche is around 30,000 USDG, and the pool's assets are reported at the end of both branches. The control, one pass over the same basket history from the same state, leaves the split essentially where the arming put it, moved only by the senior coupon and the redemption that pass funded. Both are valid executions of the same economic history and they disagree, and the caller of settlement decides which one the books record. This report files at **High** on "theft of unclaimed yield": a transfer of up to the whole Junior tranche into Senior principal, forced by an unprivileged caller and made final when Senior redeems at the raised price, with the Critical case set out under Impact Details.

The state this needs is `seniorPrincipal < seniorHighWaterMark` with `juniorPrincipal > 0`. Of the five sites in `packages/perps/src` that increase `juniorPrincipal`, four cannot leave the pool in that state. The fifth is the junior branch of `assignUnassignedAssets`, and it is a bare `juniorPrincipal += amount` with no gate of any kind (`HousePool.sol:699`), while the senior branch of the same call raises the high-water mark by the same amount as the principal (`:695-696`) and so cannot open a deficit. That call is one of the owner's documented immediate actions and the code compels it: `unassignedAssets > 0` makes `trancheDepositsAllowed` return false (`HousePoolTrancheGateLib.sol:28`), so it is the only operation that clears the unassigned-assets block on tranche deposits, and the function's own comment calls the assignment the remedy that "Prevents later LPs from implicitly capturing previously ownerless value" (`:657`). Two legitimate, documented owner operations put the pool in the state; the subject of this report is what an unprivileged caller then takes out of it, and the owner is on the defending side throughout. The proof of concept runs locally and offline, in process in the repository's own Foundry suite inside a network namespace whose only interface is loopback, with no fork URL and nothing dialled, signed or sent anywhere (Proof of Concept).

## Vulnerability Details

### Root cause

A reconcile pass compares the pool's distributable value against total claimed principal and routes the difference through one of two rules. The comparison is in `HousePoolWaterfallAccountingLib.planReconcile` (`:98-110`), and the quantity it compares against is the pair the pass is about to rewrite:

```solidity
    function planReconcile(
        uint256 seniorPrincipal,
        uint256 juniorPrincipal,
        uint256 distributableAssets
    ) internal pure returns (ReconcilePlan memory plan) {
        uint256 claimedEquity = seniorPrincipal + juniorPrincipal;
        if (distributableAssets > claimedEquity) {
            plan.isRevenue = true;
            plan.deltaAssets = distributableAssets - claimedEquity;
        } else if (distributableAssets < claimedEquity) {
            plan.deltaAssets = claimedEquity - distributableAssets;
        }
    }
```

The two routing rules are the defect. Revenue fills the senior deficit first:

```solidity
    function distributeRevenue(
        WaterfallState memory state,
        uint256 revenueAssets
    ) internal pure returns (WaterfallState memory nextState) {
        nextState = state;
        uint256 remaining = revenueAssets;

        if (remaining > 0 && nextState.seniorPrincipal < nextState.seniorHighWaterMark) {
            uint256 deficit = nextState.seniorHighWaterMark - nextState.seniorPrincipal;
            uint256 restore = remaining < deficit ? remaining : deficit;
            nextState.seniorPrincipal += restore;
            remaining -= restore;
        }

        if (nextState.seniorPrincipal > nextState.seniorHighWaterMark) {
            nextState.seniorHighWaterMark = nextState.seniorPrincipal;
        }

        nextState.juniorPrincipal += remaining;
    }
```

A loss empties Junior first, and leaves the mark where it stands:

```solidity
    function absorbLoss(
        WaterfallState memory state,
        uint256 lossAssets
    ) internal pure returns (WaterfallState memory nextState) {
        nextState = state;
        if (lossAssets <= nextState.juniorPrincipal) {
            nextState.juniorPrincipal -= lossAssets;
            return nextState;
        }

        uint256 seniorLoss = lossAssets - nextState.juniorPrincipal;
        nextState.juniorPrincipal = 0;
        if (nextState.seniorPrincipal > seniorLoss) {
            nextState.seniorPrincipal -= seniorLoss;
        } else {
            nextState.seniorPrincipal = 0;
        }
    }
```

Each rule is correct for a senior tranche read on its own, and each one matches a numbered step in the project's own specification. What no rule, comment or specification line states is that the pair composes to the identity, and it does not. The upward leg routes the delta to Senior because a deficit stands; the downward leg routes the same delta out of Junior because Junior is non-empty. The anchor both legs measure against is `seniorPrincipal + juniorPrincipal`, which the previous leg just rewrote, so the split a pass produces becomes an input to the next pass's comparison. The result is a function of the sequence of observations rather than of the value observed.

Start from `seniorPrincipal` 50,000, `seniorHighWaterMark` 100,000, `juniorPrincipal` 30,000, with distributable value at 80,000 USDG, and take the pool's distributable value through 80,000 to 110,000 and back to 80,000.

| Branch | Pass | Comparison | Routed by | End state |
| --- | --- | --- | --- | --- |
| Control | one pass at 80,000 | `80,000` against `50,000 + 30,000` | neither rule, the delta is zero | `S = 50,000 / J = 30,000` |
| Attack | first pass at 110,000 | `110,000` against `50,000 + 30,000`, revenue 30,000 | `distributeRevenue:147-151`, deficit is 50,000, so the whole 30,000 restores Senior | `S = 80,000 / J = 30,000` |
| Attack | second pass at 80,000 | `80,000` against `80,000 + 30,000`, loss 30,000 | `absorbLoss:172-173`, the loss is at or below Junior, so Junior pays all of it | `S = 80,000 / J = 0` |

The pool holds the same 80,000 USDG at the end of both branches and `seniorPrincipal + juniorPrincipal` is the same in both. What differs is who owns it. Senior is 30,000 USDG richer and Junior is 30,000 USDG poorer, exactly, and 30,000 USDG is the entire Junior tranche. The closed form over one round trip is `min(senior deficit, junior principal, swing)`, so no price peak is required: a dip and recovery of the same size produces the same transfer in the other order, which means the caller needs no price influence and no directional position of its own.

The transfer is written to pool storage. `HousePool._reconcile` (`:1549-1599`) builds the plan at `:1568-1575` and writes it at `:1584` with no condition on the write. The lines after it touch only `unassignedAssets` (`:1585`), and then, behind `markFresh`, `terminalDeficitAssets`, `lastReconcileTime` and the ownerless-junior-revenue adjustment (`:1587-1596`), which is zero while junior share supply is non-zero. `:1598` applies pending claimant buckets, of which there are none on this path. Nothing between the plan and storage can undo the split.

### When the loss becomes final

The state transition a recovery would need is a decrease of `seniorPrincipal` paired with an increase of `juniorPrincipal`. `seniorPrincipal` decreases at exactly two sites in `packages/perps/src`:

- `scaleSeniorOnWithdraw` (`:119-131`), which funds a senior redemption and reduces the high-water mark with the principal, `floor(HWM * fundedAssets / seniorPrincipal)`, so it pays the redeeming Senior holder rather than Junior and cannot widen the deficit; and
- `absorbLoss`'s senior branch (`:180-182`), which is reached only once `juniorPrincipal` is already zero.

Neither credits Junior. `absorbLoss` never writes `seniorHighWaterMark`, so later revenue fills the remaining senior deficit before Junior sees anything. Because the attack shrank that deficit by exactly the amount moved, a recovery that fills the whole deficit brings the two branches back together: with no Senior exit the reproduction measures the same Junior value in both, 56,098.64 USDG at the release risk parameters. Until that recovery arrives the moved principal is exposed, because Senior redemptions are priced on the raised `seniorPrincipal` (`HousePool.sol:845-848`) and pay it out of the pool. With the caller exiting 80% of its Senior position between the transfer and the same recovery, at the release risk parameters, its cash is 75,150.81 USDG against 51,153.73 in the control branch, and Junior's value after the recovery is 60,948.49 against 84,945.57. The caller's gain and Junior's shortfall are asserted equal, 23,997.08 USDG, and that amount does not return. The caller's total of 95,150.98 USDG is still below its 99,999 USDG deposit, because Senior was impaired before the attack: the gain is measured against not taking the extra pass. Recapitalization is explicitly excluded as a cure by its own documentation: `HousePoolPendingPreviewLib.sol:103-107` states that it never accrues to junior principal. No owner action, vault callback or keeper action moves principal from Senior back to Junior; only revenue that fills the whole remaining senior deficit closes the gap, and it closes it fully only where no Senior holder has redeemed in between.

### Trigger and preconditions

**The entry point, and that it is permissionless.** `OrderRouter.settleLpEpoch(bytes[])` (`:479-484`) delegates to `OrderLiquidationBatchLogic.settleLpEpoch` (`:549-581`), which quotes the Pyth fee, obtains a price snapshot in `PriceMode.PoolReconcile` from the caller's own update data (`:561-562`), installs that snapshot's mark on the Engine (`:563`), and then calls `HousePool.settleLpEpoch(markPrice, publishTime)` with that exact mark and publish time (`:566-578`). The pool's authorization branch requires `msg.sender == ENGINE.orderRouter()`, `ENGINE.lastMarkPrice() == expectedMarkPrice` and `lastMarkTime == expectedPublishTime` (`HousePool.sol:810-815`), and the delegate frame supplies all three. So an unprivileged caller gets a mark-fresh reconcile pass while positions are open, which is what makes a real swing in distributable value available. The project documents this route itself, at `SECURITY.md:1135-1136` and `ACCOUNTING_SPEC.md:296-298`.

**The cost of a pass.** `_emitLpEpochSettled` reverts `HousePool__NoLpEpochProgress` when no senior, junior or deposit epoch advanced (`HousePool.sol:1183-1185`), and that revert rolls back the whole pass, the write at `:1584` included, so each harmful pass has to advance a real queue item. That raises the caller's outlay from nothing to dust and does not bound it, and neither harmful pass funds a senior redemption, which matters for the duplicate analysis below. Which queued item each pass advances, and the refund branch that lets the second pass clear the guard against an emptied tranche, are in the Proof of Concept.

**The object attacked, and how the caller comes to hold a position in it.** The object is the Junior tranche of the deployed `HousePool`, in the state the owner's own remedy leaves it: a tranche whose NAV has just been credited out of previously ownerless pool value while Senior still carries a deficit. The position the attack needs is Senior tranche shares, and the caller acquires them the ordinary way, with a permissionless `TrancheVault.requestDeposit` (`:562` or `:569`) while the pool is healthy, activated by any settlement pass. That is the caller's own act: no role, no counterparty, no transfer of anybody else's shares, and the position can be held long before the state it exploits exists (Proof of Concept).

**Reaching the state.** `juniorPrincipal` increases at exactly five sites in `packages/perps/src`, and four of them cannot leave the pool with `seniorPrincipal < seniorHighWaterMark` and `juniorPrincipal > 0`:

| Site | Why it cannot arm the state |
| --- | --- |
| `HousePool.sol:1090`, deposit activation | Deposit phases are gated at `:908` on `_entriesMaySettle` (`:1100-1118`), which passes the then-current `seniorPrincipal` and `seniorHighWaterMark` (`:1115-1116`) to `HousePoolTrancheGateLib.trancheDepositsAllowed`, which returns false at `:36-38` while Senior is impaired. The same predicate also runs at request admission, through `TrancheVault._requireRequestDepositPreflight` (`TrancheVault.sol:1574-1601`, called from `_requestDeposit` at `:582`), where `maxRequestDeposit(controller)` reads zero and the request reverts `TrancheVault__DepositsUnavailable`, or `TrancheVault__TerminallyWiped` first when the tranche has shares and no assets (`:1585-1586`). The gate runs at admission and again at settlement, so a request queued while healthy still cannot land while impaired |
| `HousePoolWaterfallAccountingLib.sol:158`, revenue | Junior is reached only after the senior deficit is filled (`:147-151`), so any revenue pass that raises `juniorPrincipal` exits with `seniorPrincipal == seniorHighWaterMark` |
| `HousePoolPendingPreviewLib.sol:181`, the pending revenue bucket | Same senior-gap-first ordering at `:169-178`, skipped only when senior share supply is zero, which the permanent senior seed share floor makes unreachable once seeded: the seed call configures it at `HousePool.sol:760`, `TrancheVault.configureSeedPosition` (`TrancheVault.sol:1557-1572`) writes `seedShareFloor`, and the transfer guard at `TrancheVault.sol:322` refuses to let the seed receiver fall below it |
| `HousePool.sol:756`, the tranche seed | `onlyOwner`, once per tranche, refused on a second call at `:730`, and performed at bootstrap before any impairment exists |
| **`HousePool.sol:699`, the junior branch of `assignUnassignedAssets`** | **No gate of any kind.** The senior branch raises `seniorHighWaterMark` by the same amount as the principal (`:695-696`), so it cannot open a deficit, and it is capacity-checked besides (`:677-679`). The junior branch is a bare `juniorPrincipal += amount`. The specification states the asymmetry itself: assigning ownerless `unassignedAssets` to senior "does create new senior shares and goes through the same admission checks" (`ACCOUNTING_SPEC.md:452-453`), and it is silent on junior |

A loss cannot produce the state either: `absorbLoss` zeroes Junior before touching Senior (`:172-178`), so at the moment Senior falls below its mark, Junior is already empty. A senior redemption cannot produce it, because `scaleSeniorOnWithdraw` removes mark and principal together (`:127-130`). An unsolicited donation cannot produce it, because `accountExcess` (`HousePool.sol:568-579`) raises `accountedAssets` only and the resulting surplus reconciles as revenue, which fills the senior deficit first.

So the state is reached through the junior branch of the assignment, and the two owner steps that lead there are both on the programme's list of documented immediate owner actions, "recapitalize the pool" and "assign ownerless pool assets". A recapitalization above the standing senior deficit restores the deficit and leaves the remainder quarantined in `unassignedAssets` (`HousePoolPendingPreviewLib.sol:111-137`, the gap fill at `:124-131` and the remainder at `:134-136`), which is the allocation the specification itself names: "Normal reconciliation allocates the queued amount to senior restoration or unassigned assets" (`ACCOUNTING_SPEC.md:642`). Ordinary trader profit then wipes Junior and impairs Senior again, at which point the quarantine blocks every tranche deposit and the assignment is the only operation that clears that block, which is why the second step is not discretionary (Brief). The senior branch of the assignment is not available in that state: with Junior principal at zero the release share limit leaves no senior capacity and the call reverts `HousePool__ExceedsSeniorDepositCapacity` (`HousePool.sol:677-679`, `:1515`), which the reproduction asserts, so crediting Junior is the only assignment the code accepts. The assignment leaves the senior deficit standing, so deposits remain refused on the independent impairment clause at `HousePoolTrancheGateLib.sol:36-38` afterwards; what the assignment changes is who owns the quarantined value. The full sequence, with the production validator that accepts each step, is in the Proof of Concept section.

**The mechanism's own firing predicate reads no configuration value, and the reproduction runs at the release risk parameters.** The mark must be fresh for the pass, which the caller's own Pyth update supplies. `maxSeniorShareBps` is `8_000` at release (`DEPLOYMENT.md:155`), which is what sets the junior realization throttle discussed under the loss mechanism, and the code refuses to activate trading at the constructor's `10_000` placeholder: `HousePool.sol:536` reverts `HousePool__SeniorCapacityNotConfigured` while `maxSeniorShareBps >= 10_000`. `seniorRateBps` is `800` at release (`HousePool.sol:251`, `ACCOUNTING_SPEC.md:352`), and the reproduction also drives the sequence with it at zero. One end-to-end contract holds all five release risk parameters, read back from the engine: `vpiFactor` `0.01e18`, `maxSkewRatio` `0.4e18`, `maintMarginBps` `10`, `initMarginBps` `20` and `baseCarryBps` `500` (`DEPLOYMENT.md:131-137`). It opens a 59,200-unit short at a basket level of 0.85, a notional of 50,320 USDG against a 50,400 USDG skew limit on 126,001 USDG of depth, and the armed 30,000 USDG Junior tranche ends empty as it does elsewhere. The other two end-to-end contracts open a larger position with `maxSkewRatio` at `1e18`, carry off and the harness's price impact and margin values; the project's own `test/perps/accounting/waterfall/TrancheRecoveryBehavior.t.sol:98-99` and `test/perps/shared/SourceBoundBountyFixture.sol:25-27` carry the same price impact and skew values. No liquidation path runs in any branch: the short is in profit for the whole sequence.

### Recommended fix

The invariant to restore is that two sequences of observations ending at the same distributable value must produce the same split. The structural fix is to stop feeding the reconcile's output back into its own comparison. Keep a senior and a junior **principal basis**, written only by capital events (deposits, redemptions, assignments, seeds, coupon, recapitalization), and derive the split on each mark-fresh pass as a pure function of the distributable value and those bases rather than by routing a delta against the pair the previous pass wrote:

```text
claimed = seniorBasis + juniorBasis
if distributable >= claimed:
    surplus        = distributable - claimed
    seniorRestore  = min(surplus, seniorHighWaterMark - seniorBasis)
    seniorPrincipal = seniorBasis + seniorRestore
    juniorPrincipal = juniorBasis + (surplus - seniorRestore)
else:
    shortfall       = claimed - distributable
    juniorPrincipal = juniorBasis - min(shortfall, juniorBasis)
    seniorPrincipal = seniorBasis - (shortfall - min(shortfall, juniorBasis))
```

Both declared rules survive unchanged: a shortfall is still charged to Junior first and then Senior, surplus still restores Senior to its high-water mark before reaching Junior, and losses still do not reduce the mark. What goes away is the dependence on how many times the comparison ran, because `seniorBasis` and `juniorBasis` do not move when the split does. On every case in this report the fix makes both branches agree, which is the point: the control's answer is the one the specification's own step 2 prescribes, a comparison of terminal equity against total principal.

If a basis pair is too large a change for a patch release, a narrower change closes the peak path only. It is one additional field on `WaterfallState`, persisted in one new `HousePool` storage slot, recording senior principal restored out of revenue, incremented by `restore` at `distributeRevenue:150` and consumed by `absorbLoss` before it reaches Junior:

```solidity
        uint256 unwind =
            lossAssets < nextState.seniorRevenueRestored ? lossAssets : nextState.seniorRevenueRestored;
        if (unwind > 0) {
            nextState.seniorPrincipal -= unwind;
            nextState.seniorRevenueRestored -= unwind;
            lossAssets -= unwind;
            if (lossAssets == 0) {
                return nextState;
            }
        }
```

A loss then unwinds the restoration it is reversing before it charges the tranche that never received it, and a funded senior redemption reduces the field alongside the principal it pays out, rounded in the protocol's favour. With no restoration outstanding the field is zero and the junior-first waterfall is untouched, so an ordinary first loss behaves exactly as it does today. It does not close the trough path: a dip charges Junior while no restoration is outstanding, and the recovery then restores Senior. The principal basis is the complete fix.

Either way, the regression test that detects this class is a property test over observation sequences: for any list of distributable values ending at `D`, assert that the resulting `(seniorPrincipal, juniorPrincipal)` equals the result of a single pass at `D` from the same starting state. The repository's existing waterfall property file is the natural home for it; it currently asserts nothing about observation count.

## Impact Details

### The in-scope impact this maps to

The governing rung, quoted byte for byte from the programme's Impacts in Scope list:

```text
High: temporary freezing of funds ≥24h; theft of unclaimed yield; permanent freezing of unclaimed yield.
```

This report files at **High**, on "theft of unclaimed yield". The Junior principal the caller takes reached the Junior tranche as previously ownerless pool value, quarantined in `unassignedAssets` and then vested in the Junior share holders by the owner's assignment. On the conservative reading, that is unclaimed yield. Senior holders receive it pro rata, so the caller's share is its share of the Senior tranche.

The rung above is quoted here with equal care, because the report does not want the reviewer to have to guess which case it is making:

```text
Critical: direct theft of any user funds (at-rest or in-motion) other than unclaimed yield; permanent
freezing of funds; protocol insolvency.
```

The Critical case is real and worth stating. At the moment the attack fires, the Junior share holders hold minted `TrancheVault` shares and `juniorPrincipal` is, in the project's own words, "the current NAV attributed to the two tranches, not immutable records of historical deposits" (`ACCOUNTING_SPEC.md:331-332`). The assignment exists precisely to vest that NAV in them, which the function's comment states as its purpose (`HousePool.sol:657`). On that reading what the caller removes is user funds at rest and the entry is Critical.

The report lands at High on one condition and one only: that the carve-out in the Critical entry, "other than unclaimed yield", covers value whose origin is ownerless pool assets even after the owner's assignment has vested it in named share holders. If a reviewer reads that carve-out as not reaching vested tranche NAV, then on the same evidence the entry is "direct theft of any user funds (at-rest or in-motion)" and the rung is Critical. Nothing else about the report changes between the two readings, and the programme's own separation of impact from feasibility, quoted below, means the arming chain does not bear on the choice.

### Loss mechanism and its bound

Conservation, stated explicitly. **In:** nothing. Claimed equity is the same at the end of both branches once each branch's own payouts are added back, which is the asserted form: `seniorPrincipal + juniorPrincipal + the junior redemption that branch's passes funded` is equal across the two branches, from one snapshot over the same basket history. Each branch can pay a queued redemption out through its own passes, which is why those payouts are in the equation rather than outside it. Pool assets are reported alongside it rather than asserted equal. **Out:** nothing on the round trip beyond those funded redemptions, which the pool pays into the tranche vault as escrow the holder can then claim (`HousePool.sol:1007-1008`, `TrancheVault.sol:1211`). **Who is short, and by how much:** the Junior share holders, by `min(outstanding senior deficit, Junior principal, cumulative swing)`, which on the reproduction's arming is the armed Junior tranche of around 30,000 USDG net of the dust redemption the holder's own queue work realized, because the arming asserts the senior deficit exceeds the tranche. The tranche itself is asserted to end empty, so what is left for the holder to redeem prices to nothing. Senior redemption carries no corresponding ratio cap and is priced against `seniorPrincipal` directly (`HousePool.sol:845-848`), so the caller's own matured senior redeem request pays out strictly more in the attack branch than in the control branch from the same shares, asserted as settlement token received in both. That is where the transferred principal leaves the protocol.

The entitlement that is violated is the project's own. `seniorHighWaterMark` "records the Senior entitlement that future revenue must restore after a loss" (`ACCOUNTING_SPEC.md:332-333`). There was no net revenue over the round trip. The caller's first pass restores Senior out of a revenue leg and the second pass charges the reversal of that same leg to Junior, so the restoration is funded by Junior principal rather than by revenue, and the mark stays where the restoration left it.

The harm is realized at the redemption window, not only in an internal ledger. `juniorPhase.pricingPrincipal = juniorPrincipal` (`HousePool.sol:840`) and that value is consumed by the redemption-share pricing at `:957-959` and `:992-999`, so a Junior holder's matured redeem request is priced against the reduced principal. In the attack branch `juniorPrincipal` is zero, so the holder's shares price to zero and the matured request is refunded with nothing (`:957-987`). In the control branch, over the identical price history from the same snapshot, the holder's remaining position funds. Each branch exits whatever Junior position it still holds, and the reproduction prints both share counts rather than asserting them equal; what the comparison turns on is that one of the two prices to nothing.

**The victims cannot step aside, and the reason is a release parameter.** Junior realization per pass is capped by `_juniorSettlementCapacity` (`HousePool.sol:1205-1217`), which takes `min(freeAssets, min(juniorPrincipal, juniorWithdrawalRatioCap))`, and `HousePoolSeniorCapacityLib.juniorWithdrawalRatioCap` (`:64-90`) leaves `juniorPrincipal - ceil(exposure * (BPS - maxSeniorShareBps) / maxSeniorShareBps)`, where `exposure` is `max(seniorPrincipal, seniorHighWaterMark)` (`:17-22`). With `exposure` at the 100,000 the release seed and the caller's deposit supply, `juniorPrincipal` at 30,000 and the release `maxSeniorShareBps` of `8_000`, the covenant reserves 25,000 USDG and leaves a withdrawal capacity of 5,000; the coupon raises the mark the cap reads above that nominal figure while Senior is unimpaired (`HousePoolWaterfallAccountingLib.sol:87-90`), so the arming's own capacity is at or below that 5,000. So a Junior holder who sees the impairment and queues a redemption can realize at most 5,000 USDG on the first funded pass and nothing after it, because the capacity is then zero and stays zero until the senior entitlement itself falls. The caller needs two passes. This report therefore does not claim 30,000 USDG of same-block withdrawable cash: it claims that the holder's claim on the value is impaired until a full recovery, that a Senior exit makes that loss final, and that the covenant is what stops the holder from reaching it first. The reproduction reads the holder's receipts in both branches rather than their ledger entry, and reads back the share price after the attack. At the release risk parameters the holder's whole remaining position is quoted at 29,997.24 USDG in the control branch and zero in the attack branch; the cash it realizes on one pass in the control branch is 4,996.20 USDG because of that covenant, so the position value rather than the one-pass receipt is the size of the claim at stake.

**The bound.** Across round trips the transfer accumulates, because the mark does not move and each recovery leg finds the deficit still open. The cumulative ceiling is `min(the senior deficit outstanding when the passes begin, the Junior principal available before them, the cumulative effective swing)`, and it is the whole Junior tranche exactly when the first of those three is at least the second, which is the case the arming asserts and the reproduction drives. Where the deficit is the binding term the tranche is not emptied: the reproduction drives that case too, a 60,000 Junior tranche against a 50,000 senior deficit, where 50,000 is asserted to cross and the 10,000 remainder is asserted to stand. Intervening capital events move the terms, and a senior redemption that funds lowers the mark alongside the principal.

### Why not lower, and the out-of-scope entries to weigh

The rung below is Medium, quoted byte for byte here because the only entry on it that could reach this shape is its griefing clause:

```text
Medium: theft of gas; SC unable to operate due to lack of token funds; block stuffing; griefing (no
profit motive but damage to users/protocol); unbounded gas consumption; temporary freezing of funds ≥1h.
```

That clause does not apply, and the reason is asserted against the contracts rather than argued from source. The caller holds Senior tranche shares obtained by an ordinary permissionless deposit. Senior redemption is priced off `seniorPrincipal` across the whole senior share supply, so the caller's senior redeemable NAV rises in proportion to its senior shareholding, and the reproduction drives the consequence directly, in the receipts comparison recorded under the loss mechanism above. There is a profit motive and the caller collects it, so the clause's own precondition fails. The surrounding baseline on the same page reads "Griefing baseline: $1 to cause ≤$10 is griefing; $1 to cause ≥$100 is not", and the caller's whole outlay is dust queue work plus one Pyth update fee per pass, against the entire Junior tranche.

The programme's own words put the attacking half of this in scope on its own, independently of anything the owner did, quoted byte for byte:

```text
**KEEPERS ARE NOT TRUSTED: anyone can execute orders, liquidations, price updates and protections, so
harm caused by any caller acting as a keeper is IN SCOPE.** Keepers have no special privileges; all
actions remain subject to on-chain checks. Consider any contract-permitted behavior, including
deviations from the reference implementation.
```

The caller of `OrderRouter.settleLpEpoch(bytes[])` is doing exactly that: a price update and a keeper settlement, with no special privilege, subject to every on-chain check the pool applies. The clause names the act and declares harm from any such caller in scope. Seven out-of-scope entries and downgrade criteria are then worth weighing explicitly, each as a second and independent answer.

- **"attacks requiring privileged-address access (except where a code vuln grants privileges the address did not originally hold)"** does not reach this report, because the programme's own Trusted Roles section carves in the case this is: "IN SCOPE: a legitimate, documented action by a trusted role that triggers a bug harming users or the protocol. ... A report is not OOS only because a privileged address is affected." Both owner actions are on the same page's list of documented immediate owner powers, "recapitalize the pool" and "assign ownerless pool assets". The attacker holds no role at any point, and the owner appears only on the defending side.
- **"attacks requiring a trusted role to act maliciously, be compromised, or act outside documented operational procedures"** does not reach it either, and the third limb is the one worth answering on its own terms. Each owner step is performed in the circumstance its own documentation names. `CfdEngine.recapitalizePool` "is an immediate, owner-only, non-reentrant recovery action" (`SECURITY.md:157`) whose purpose is "recapitalization intended to restore waterfall claimants" (`ACCOUNTING_SPEC.md:634`), and a recapitalization above the standing senior deficit does restore the claimant and quarantines only the overshoot, which is the allocation the specification prescribes (`ACCOUNTING_SPEC.md:642`). The assignment is then the only operation that clears the deposit block the quarantine creates (`HousePoolTrancheGateLib.sol:28`), and the function's own comment states its purpose (`HousePool.sol:657`). The reproduction's fixture drives the degenerate case, a recapitalization against a zero deficit, because it makes the quarantined amount exact and the arithmetic auditable; an overshoot on an impaired pool reaches the same armed state through the same two branches of `applyClaimantRecapitalizationIntent` (`HousePoolPendingPreviewLib.sol:124-131` then `:134-136`).
- **"Setting a parameter to a value the code allows is a trusted action, not a bug"** does not reach it: neither routing function reads a parameter, and the transfer is asserted at the release risk parameters and with the senior coupon rate at zero.
- **"issues reachable only via inactive feature gates/flags/configs/networks"** does not apply, and the reason is what the mechanism depends on rather than what the fixture sets. Permissionless keepers, trading on the six-currency basket and both tranches are on the programme's own list of features active at launch. The firing condition is `seniorPrincipal < seniorHighWaterMark` with `juniorPrincipal > 0` and a swing in distributable value, and neither routing function reads a risk parameter, so nothing the composition itself depends on is behind a gate, a flag or a config value. The reproduction also runs at the release `maxSkewRatio` of `0.4e18` (`CfdEnginePlanLib.sol:473-480`).
- **"incorrect third-party oracle data"** is not relied on. The round trip is movement in the basket: a dip and a recovery of the same size works as well as a peak, so no manipulated price, no stale price and no price influence of any kind is required. The swing has to be large against the book: the reproduction uses 0.508 per unit on the release-parameter book and 0.301 on the larger one, supplied through the mock oracle. The caller supplies a valid Pyth basket and pays its fee.
- **The containment downgrade** requires "demonstrated monitoring coverage and response timing", and the repository's dedicated monitoring document answers the first limb against itself. `EMERGENCY_RESPONSE_GUIDE.md` publishes a per-signal action map with a "Minimum action policy" column (`:171-182`), and the state this finding produces is not on it. Claimed equity is conserved and token balances reconcile, so neither "NAV aggregate faults" nor "Pool/vault custody or escrow deficits" (`:175-176`) is raised. What the pool does show is senior impairment, and the guide's own list of conditions a guardian must **not** act on names it: "`NoFreeCash`, matured Senior precedence, capacity exhaustion, Senior impairment, or an ordinary deposit deferral" (`:144-147`). On the second limb, the programme states that "No detection/response deadline is assumed unless explicitly documented", and the guide documents none: its timing words are conditions on confirmation rather than published deadlines, and the one line in the repository that mentions a response time, `DEPLOYMENT.md:502`, instructs an operator to set a standby response time rather than publishing one. A guardian's LP epoch settlement hold also does not undo a transfer that has already been written.
- **"Pre-impact monitoring"** allows a one-level downgrade only for "automated prevention with 100% certainty", with "human-action prevention needs 100% objective certainty of both detection and prevention". The trigger is a permissionless settlement pass that passes every on-chain check, so there is no automated prevention on the path at all, and the guide's own instruction is to leave the enabling state alone.

Finally, the arming chain is a feasibility question rather than a severity one, by the programme's own terms: "Impact is calculated first, separately from feasibility. Feasibility does not change impact/severity, only the amount paid." The chain reduces the award, not the rung.

### Not a duplicate, and scope

**The nearest prior art is the AI scan's only Medium, and this report is distinguished from it by what the reproduction drives, not by argument.** That finding, "Checkpoint frequency compounds a configured simple senior coupon", has the same actor (a Senior LP) and the same victim (Junior LPs), and the two share the trick of keeping a small matured LP epoch item so that a pass advances queue work. They do not share a lever, because they run in mutually exclusive regimes. The scan's own claim restricts itself to the regimes where a caller can settle without the Router: "A senior LP can force hourly successful checkpoints during permissionless settlement regimes (no open positions or frozen-oracle mode) by maintaining a small matured LP epoch item", and its evidence is the direct `HousePool.settleLpEpoch` branch. This finding cannot run there at all: with no open positions there is no distributable swing to observe twice, so it runs only through the Router, where `HousePool.sol:810-815` is a hard gate on `msg.sender`. The project states the separation itself: "direct cached-mark settlement is restricted to zero-open-position state or `oracleFrozen`; a separately refreshed live cached mark cannot bypass the atomic Router path" (`ACCOUNTING_SPEC.md:318-319`). Its mechanism is also a different pair of functions: `paySeniorCoupon` (`:65-91`) capitalizes paid coupon into senior principal, so splitting an elapsed period across more checkpoints yields `exp(rate)-1` instead of `rate`. That mechanism is inert at a zero coupon rate, because `calculateSeniorCoupon` (`:45-54`) returns `(seniorPrincipal * seniorRateBps * elapsed) / (BPS * SECONDS_PER_YEAR)`, which is identically zero for every elapsed value when the rate is zero. The reproduction drives the whole sequence with the rate finalized to zero in protocol storage and asserts the transfer, plus the holder's own funded redemption, against the armed Junior tranche exactly. The magnitude laws differ as well: that one is about 33 basis points a year at the default and grows with elapsed time, while this one is `min(senior deficit, junior principal, swing)` per round trip and is time independent. Their preconditions differ too: that one needs only elapsed time, this one needs `seniorPrincipal < seniorHighWaterMark` with `juniorPrincipal > 0`.

**This report sits in the incomplete-fix lane the programme defines, and the lane is earned rather than asserted.** The programme's terms read: "For findings marked fixed or Acknowledged in the reports above, a report on the SAME ROOT CAUSE is valid ONLY if it shows the fix at a82d4fef is incomplete or introduces a NEW bug." Two things in the repository show what the project did about disclosed settlement-frequency dependence, and neither reaches this channel.

First, `SECURITY.md:908-911` is the project's own statement of what survives its guard, quoted byte for byte:

```text
A settlement pass that advances no queue item reverts, rolling back both waterfall reconciliation and engine carry
checkpoints. This prevents per-block no-op checkpoint grief. Senior coupon arithmetic still floors independently at
each settlement that advances real epoch work, so sub-micro-USDG rounding remains execution-frequency dependent; the
hourly aggregation of requests bounds that residual rather than making coupon accrual mathematically remainder-exact.
```

The declared residual is sub-micro-USDG, and it is located in senior coupon arithmetic. The effect this report claims is around 30,000 USDG through a different pair of functions, and it persists with the coupon at zero.

Second, the documentation change the project made in response. The AI scan quotes `IHousePool.sol:56` as reading "Annualized simple senior target coupon rate in basis points." At the scoped commit the same parameter reads:

```text
    /// @param seniorRateBps Nominal annual senior target coupon rate in basis points, applied linearly to current
    ///        senior principal over each checkpoint interval. Paid coupon is capitalized into senior principal and
    ///        compounds at subsequent checkpoints; effective yield depends on checkpoint frequency and available
    ///        junior NAV.
```

By its own words the declaration is scoped to coupon **yield**. It says nothing about the revenue and loss waterfall moving junior **principal**, and the two functions this report turns on were not changed. The post-audit remediation pass over this same library changed exactly one function in it, `scaleSeniorOnWithdraw`, whose signature and mark-reduction arithmetic were both rewritten; `absorbLoss` and `distributeRevenue` were left as they are. So the library was audited, the senior-withdraw mark scaling was fixed, the coupon's frequency dependence was documented as intended, and the composition of the two halves was left untouched and undeclared.

**The remaining neighbours, taken one at a time.**

- **The Cecuro report's LOW-54, "Senior redeemers can leave dust high-water mark by splitting withdrawals"** (Low, Acknowledged), is the closest row in the corpus by consequence, and it shares a sentence with this report: "The extra high-water mark is restored before junior receives revenue". Its root cause is floor rounding inside `scaleSeniorOnWithdraw`, which its locations list first; its lever is a split senior redemption; and its authors bound and classify it themselves: "The error is less than one USDC base unit per funded senior settlement pass, which makes this a dust accounting defect rather than a material value leak." The accepted residual is recorded with the same bound and a clearing condition, "full principal funding clears the HWM". Its own stated precondition is that at least one senior redemption is funded through `HousePool._settleLpEpoch`. The reproduction's passes fund none: `scaleSeniorOnWithdraw` is called from exactly one place in the in-scope tree, `HousePool.sol:857-859`, behind `if (seniorPhase.fundedAssets > 0)`, and the reproduction asserts zero senior redemption assets on both harmful passes. The function that row is about is not on this path, and the magnitudes are ten orders of magnitude apart: the transfer here is exact rather than dust.
- **The same residual appears in the project's declared Known Limitations**, in the section the programme names as known, and it is declared at the same size: "An impaired Senior withdrawal may retain less than one extra USDG atom of high-water-mark protection per funded pass because of conservative rounding. This can accumulate across passes and does not describe a universal bound on all LP allocation dust." The declared quantity is the rounding residual of a funded senior withdrawal, per funded pass. The reproduction's harmful passes fund no senior withdrawal, so `scaleSeniorOnWithdraw` is never reached and the accumulation the sentence contemplates has no leg to accumulate on.
- **LOW-63, "Partial LP settlement can round dust away from redeemers"** (Low, Acknowledged, no code change) is the corpus's other path-dependence row, and it is path dependence in a different place. Its own words: "splitting funding can produce rounding differences or a residual share refund compared with a single pass ... This is a path-dependent rounding issue rather than a material redemption loss", with "Any caller settles LP epochs" among its preconditions and its locations at `HousePool.sol:973-1005` and `HousePoolRedemptionMathLib.sol:63-107`. Its arithmetic is floor rounding against the ERC-4626 virtual offsets across a **partially funded** redemption epoch, and it is self-bounded to "only dust at partial-fill boundaries". Its cited range sits inside `_fundRedemptionPhase` (`HousePool.sol:944`), which the junior settlement branch reaches at `:882`, so what separates the two rows is magnitude and which function carries the loss rather than non-contact: that row's quantity is a partial-fill remainder produced by redemption-share pricing, while this one's is the tranche, written by `distributeRevenue` and `absorbLoss`.
- **MED-24, "LP can delay other redeemers with zero-payout refund epochs"** (Medium, Acknowledged, no code change, and one of the open Cecuro findings the programme page lists as known) is the row whose branch the second pass runs through. A head redeem epoch whose quote rounds to zero takes `refundRedeemEpochRemainder` and still increments `processedEpochs` (`HousePool.sol:957-987`, the increment at `:985`), and that increment is what lets the second pass clear `HousePool__NoLpEpochProgress` (`:1183-1185`) against an emptied tranche. The root cause is a different one. MED-24's is a progress guard that counts refund-only work, and its harm is settlement work spent before a later redeemer is funded; nothing in it moves principal between tranches. This report's is the order of the two waterfall functions, and the refund epoch is only the queue work that pays for the second observation. The first pass funds a real Junior redemption out of Junior principal, which the capture prints from the pool's own settlement event. Whether another progress source could stand in for the refund on the second pass is not established here: a funded Senior redemption also counts as progress, but it runs `scaleSeniorOnWithdraw` (`:857-863`), which neither harmful pass reaches, so it is a different path from the one tested.
- **The Cecuro entry on a senior rate change against a stale mark** is a freshness-predicate mismatch on an owner rate change, and its own remediation is to use one freshness predicate for both the rate-change gate and the coupon checkpoint. Its stated consequence overlaps this report's on one point, junior LPs bearing a charge they should not: it also ratchets an inflated senior high-water mark that "persists into later revenue allocation", which this report's mechanism does not do, because the reproduction asserts the mark unchanged after the harmful pass and after every later pass. That row reaches its own consequence through a coupon checkpoint taken on stale senior principal, which is a different root cause from the non-inverse ordering of `distributeRevenue` and `absorbLoss`, and it needs the engine's effective freshness limit to be tighter than the pool's with the mark sitting in the gap between them. The reproduction changes no coupon rate at all on the release-rate branch, and refreshes the mark immediately before the one rate change the zero-coupon branch makes.
- **The SC Audit Studio pre-audit** names these two functions in exactly two places, both about `HousePool`'s private duplicates rather than the library: a refactor inventory, and a recommendation to delete `_distributeRevenue`, `_absorbLoss` and the `Reconciled` event as dead code.
- **The collaborative audit and the Cantina review**, both in the listed repository and therefore treated here as disclosure capable even though the known-issue list does not cite them, contain no mention of `TrancheVault` and predate the perps package.

No prior-art row in any of the five documents states that `absorbLoss` and `distributeRevenue` are not inverses, or that the Senior and Junior split depends on how many times the reconcile observed a closed round trip. The rows that reach a similar consequence get there by other means: LOW-54 and LOW-63 through floor rounding, and the senior-rate row through a stale-mark coupon checkpoint. None of the three is about the function pair this report turns on. LOW-54's authors state its bound directly, at less than one base unit per funded senior settlement pass, and its function is not called on this path at all. LOW-63's description bounds its leak to dust at partial-fill boundaries at normal six-decimal scale, and its function is on this path but prices redemption shares rather than routing the revenue and loss delta.

**Nor is it a declared design decision.** The project's specification writes reconciliation as a state function rather than as a function of observation history, quoted byte for byte:

```text
2. Compare terminal equity, excluding reserved pending claimant inflows and `unassignedAssets`, with total principal.
3. Charge a shortfall to Junior first, then Senior. Losses do not reduce the Senior high-water mark.
4. Route surplus first to restore Senior principal to its high-water mark, then to Junior.
```

Step 2 is a comparison of terminal equity against total principal, and the control branch is what it prescribes. The nearest adjacent declaration, `ACCOUNTING_SPEC.md:448-452`, says that the senior **limits** are "admission controls, not alternate waterfall arithmetic" and that the four named operations "continue under their ordinary priority rules". It is about the limits, and it is silent on observation count. The sticky high-water mark is declared, deliberate and not what this report attacks: what is undeclared is the composition.

**Scope.** Every file this report turns on is in scope. The asset list covers "All contracts, libraries, router contracts and interfaces in `packages/perps/src`, EXCEPT the six view-only contracts listed out of scope below", `HousePool.sol` is named separately, and none of `HousePoolWaterfallAccountingLib.sol`, `HousePoolReconcilePlanLib.sol`, `HousePoolTrancheGateLib.sol`, `HousePoolSeniorCapacityLib.sol`, `HousePool.sol`, `OrderRouter.sol` or `router/OrderLiquidationBatchLogic.sol` appears in the six-file out-of-scope list. No in-scope contract is modified by the reproduction. The only substitutions are the repository's own Pyth and settlement-token mocks, both of which are out-of-scope components that the programme's PoC rules permit replacing.

**Time bar.** The listed repository's tip is the scoped commit, `a82d4fef` of 2026-10-29. Its public source carries exactly one later commit, `6909f261` of 2026-10-29, "fix(test): speed up local perps fast tests with persistent builds", which touches `.gitignore`, a test map and four build scripts and no file in `packages/perps/src`. The most recent commit touching `HousePoolWaterfallAccountingLib.sol` is `9acf2533`, which is an ancestor of the pin and therefore already present in the scoped tree, so there is no fix of this mechanism to bar it. Neither repository carries a public issue, pull request or commit disclosing this mechanism: the listed repository's issue list is empty and its pull requests are disabled, and the public source's full issue and pull-request history carries one issue, a deployed-configuration note on risk parameters that states in its own body that it is not a code-vulnerability report, and nothing on the waterfall, on `absorbLoss`, on `distributeRevenue` or on observation count.

## References

Repository: `https://github.com/immunefi-team/audit-comp-plether`, at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. Every path below is relative to `packages/perps/`.

1. `src/libraries/HousePoolWaterfallAccountingLib.sol` - `#L140-L159` (`distributeRevenue`, senior restoration at `#L147-L151`, junior remainder at `#L158`, and the ratchet branch at `#L154-L156`, which the function's own comment declares at `#L135-L136` and which cannot fire at the scoped commit because every senior-principal increase in the package is either capped at the standing deficit, out of revenue at `#L147-L151`, out of the coupon at `#L80-L85`, out of a recapitalization at `HousePoolPendingPreviewLib.sol:127-131` and out of the pending revenue bucket at `:173-177`, or leaves senior principal at or below the mark, either by raising the mark by the same amount (`HousePool.sol:695-696`, `:752-753`, `:1087-1088`, `HousePoolWaterfallAccountingLib.sol:87-89`) or by setting principal and mark together from a zeroed tranche (`HousePool.sol:692-693`, `HousePoolPendingPreviewLib.sol:119-121`). `scaleSeniorOnWithdraw` (`#L127-L130`) cannot invert the relation either: its callers must supply `fundedAssets <= seniorPrincipal` (`#L113`) and the mark reduction is the floor-rounded same fraction, which leaves the mark at or above the reduced principal. `HousePool.sol:1881` and `:1883` write principal and mark back together and introduce no new relation), `#L167-L184` (`absorbLoss`, junior-first at `#L172-L173`, senior branch at `#L177-L183`) with the design comment at `#L162-L163`, `#L98-L110` (`planReconcile`, the anchor at `#L103`), `#L119-L131` (`scaleSeniorOnWithdraw`, mark reduction at `#L127-L129`), `#L45-L54` (`calculateSeniorCoupon`), `#L65-L91` (`paySeniorCoupon`), `#L89` (the only other site that raises the mark in this library)
2. `src/libraries/HousePoolReconcilePlanLib.sol` - `#L47-L106` (the composer), `#L84-L90` (the anchor call), `#L95-L99` (the revenue branch), `#L102-L105` (the loss branch)
3. `src/HousePool.sol` - `#L1549-L1599` (`_reconcile`), with the unconditional storage write at `#L1584` and the post-plan block at `#L1585-L1598`; `#L797-L821` (the pool's settlement entry point), with the authorization branch at `#L810-L815` and the reconcile's place inside settlement at `#L835`; `#L1179-L1185` (`_emitLpEpochSettled` and `HousePool__NoLpEpochProgress`); `#L835-L863` (the phase block: junior pricing principal at `#L840`, senior redemption priced against senior principal with no ratio cap at `#L845-L848`, the only call site of `scaleSeniorOnWithdraw` gated on funded assets at `#L857-L859`); `#L957-L999` (junior principal into redemption pricing at `#L957-L959` and `#L992-L999`, and the zero-price refund branch that still advances epoch work at `#L985`); `#L1205-L1217` (`_juniorSettlementCapacity`); `#L653-L704` (`assignUnassignedAssets` with its own comment at `#L653-L657`, the senior capacity check at `#L677-L679`, the matched senior principal and mark increase at `#L695-L696`, the ungated junior credit at `#L699`); `#L714-L762` (`initializeSeedPosition`, the one-per-tranche guard at `#L730`, the junior credit at `#L756`, the seed-floor configuration call at `#L760`); `#L908` with `#L1100-L1118` (`_entriesMaySettle` passing the then-current senior state to the gate at `#L1115-L1116`) and `#L1090` (deposit activation's junior credit); `#L458-L478` (`canAcceptOrdinaryDeposits` at `#L462` and `canAcceptTrancheDeposits` at `#L474`, with the first one's own dev comment at `#L459-L460` stating that it checks neither unassigned assets nor senior impairment, and the second one's at `#L469-L470` naming both); `#L568-L579` (`accountExcess`); `#L532-L542` (`activateTrading` refusing an unconfigured senior share limit at `#L536`); `#L251` (the constructor's `seniorRateBps`); `#L1471` (`reconcile()`, `onlyVault`); `#L1510-L1530` (`_requireSeniorDepositCapacity` and the capacity it reads)
4. `src/libraries/HousePoolTrancheGateLib.sol` - `#L19-L41` (`trancheDepositsAllowed`), with the unassigned-assets clause at `#L28` and the impairment gate at `#L36-L38`
5. `src/libraries/HousePoolSeniorCapacityLib.sol` - `#L17-L22` (`protectedExposure`), `#L64-L90` (`juniorWithdrawalRatioCap`)
6. `src/OrderRouter.sol` - `#L474-L484` (the permissionless settlement entry point and its own comment), delegating through `src/router/OrderLiquidationBatchLogic.sol` - `#L549-L581`, with the `PoolReconcile` snapshot at `#L561-L562`, the engine mark install at `#L563` and the pool callback at `#L566-L578`
7. `src/libraries/HousePoolPendingPreviewLib.sol` - `#L111-L137` (the recapitalization remainder that quarantines into unassigned assets), `#L158-L188` (`applyRevenueIntent`, the senior-gap block at `#L169-L178`, the junior credit at `#L181`), `#L103-L107` (recapitalization never accrues to junior principal)
8. `src/interfaces/IHousePool.sol` - `#L55-L59` (the senior rate parameter's declared scope)
9. `src/TrancheVault.sol` - `#L1574-L1601` (`_requireRequestDepositPreflight`, the admission-time refusal, called from `_requestDeposit` at `#L582`), `#L1557-L1572` (`configureSeedPosition`, which writes `seedShareFloor`), `#L322` (the transfer guard that keeps the seed receiver above the floor)
10. `ACCOUNTING_SPEC.md` - `#L331-L340` (the waterfall rules, with the terminal-equity comparison at `#L336`), `#L352` (the default senior rate), `#L296-L298` (the router settlement route), `#L318-L319` (direct cached-mark settlement restricted to zero-open-position or frozen state), `#L448-L455` (the limits are admission controls, with the senior-assignment admission check at `#L452-L453`), `#L634` and `#L642` (recapitalization's declared purpose and its allocation to senior restoration or unassigned assets)
11. `SECURITY.md` - `#L157` (recapitalization declared as a recovery action), `#L908-L911` (the no-progress guard and the declared frequency-dependent residual), `#L1135-L1136` (settlement with open positions is available only through the router entry point), `#L1417-L1419` (the declared high-water-mark rounding residual, inside the Known Limitations section)
12. `EMERGENCY_RESPONSE_GUIDE.md` - `#L144-L147` (the states a guardian must not act on, senior impairment among them), `#L171-L182` (the monitor signal and minimum-action map)
13. `DEPLOYMENT.md` - `#L131`, `#L133`, `#L134`, `#L135` and `#L137` (the release price-impact, skew, maintenance-margin, initial-margin and carry parameters, the five the release-parameter contract installs), `#L154-L156` (release senior limits and tranche seeds), `#L502` (the only response-time line in the repository, which instructs rather than publishes)
14. `test/perps/BasePerpTest.sol` - `#L232-L245` (the harness risk parameters the fixture inherits three of), `test/perps/accounting/waterfall/TrancheRecoveryBehavior.t.sol` - `#L98-L99`, and `test/perps/shared/SourceBoundBountyFixture.sol` - `#L25-L27` (the project's own fixtures carrying the same parameter values)
15. `src/libraries/CfdEnginePlanLib.sol` - `#L473-L480` (the skew limit as a fraction of pool depth, and the `SKEW_TOO_HIGH` refusal on the production open path), `src/CfdTypes.sol` - `#L92`, `#L95` and `#L106-L107` (the declared meaning of the skew ratio and the maintenance-margin ratio), `src/libraries/LiquidationAccountingLib.sol` - `#L63` (the maintenance margin the ratio requires)

File fingerprints at the scoped commit:

| File | Bytes | Lines | sha256 |
| --- | --- | --- | --- |
| `src/libraries/HousePoolWaterfallAccountingLib.sol` | 9,406 | 186 | `83cc6cf276620f439d8c7500e43ca50370214c41304b7e26c9ee682274c477ab` |
| `src/libraries/HousePoolReconcilePlanLib.sol` | 6,931 | 125 | `1cfebbedd22401966024276920276156bbaae03b21dba41af20d0e6217bb86af` |
| `src/libraries/HousePoolTrancheGateLib.sol` | 1,921 | 43 | `92f0e835024dd19e12e00b312954921204bd09e46ab2c25028f084b0213561bd` |
| `src/libraries/HousePoolSeniorCapacityLib.sol` | 5,383 | 133 | `92c7946d7fd364487da95dff4277a2f432c7b11cfd9f4063515540dd311c379f` |
| `src/HousePool.sol` | 91,455 | 1,886 | `57cde380a49f5fa0607fa101e790e64f539b32a52da7baaae520daac0ee40159` |
| `src/OrderRouter.sol` | 29,515 | 557 | `8f8318db0889f5d098d2f5c8c6afc41c473d1ca516817fe2363ba3899bb48d7d` |

## Link to Proof of Concept

https://gist.github.com/ChaseTheLight01/84d7371d0832bfe04a2d3747e0cdba53

## Proof of Concept

## Proof of Concept

The two test files, both runners, the fix patch, `RUN.md` and both captures are in this Gist: https://gist.github.com/ChaseTheLight01/84d7371d0832bfe04a2d3747e0cdba53

The test runs locally. It is a Solidity test file placed under `packages/perps/test/perps/` and executed in process by `forge test` against a local checkout of `packages/perps` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, extending the repository's own `BasePerpTest` harness, which deploys a real `HousePool`, both real `TrancheVault` instances, a real `CfdEngine` and a real `OrderRouter`. The shipped runner enters a network namespace itself (`unshare -rn`) and checks from inside it before any test runs: that the only interface is `lo`, that there is no usable IPv4 or IPv6 route, and that outbound connect probes to two public addresses do not succeed. If any check fails the runner stops without running anything, and those checks are the first lines of `CAPTURE.log`. No node and no validator is started, no chain RPC is contacted, `forge` is given no `--fork-url` and runs with `FOUNDRY_OFFLINE=true`, nothing of Plether's is reached, no key material is used, and nothing is signed or sent anywhere. The only stand-ins are the repository's own out-of-scope mocks, `MockPyth` for the oracle payload and `MockSettlementToken` for USDG; both are permitted substitutions under the programme's PoC rules, and both are instantiated in process by the harness. Nothing in this report rests on a request to any Plether endpoint, a chain RPC, a block explorer or a node.

No in-scope contract is modified. The runner fingerprints the seven source files the finding turns on against their sha256 at the scoped commit and stops before compiling if one of them differs; the table at the end of the Bug Description lists six of the seven with the same hashes. The remaining compiled dependencies are reached through the repository's own harness and are not individually fingerprinted, so run it against a clean checkout at the pinned commit.

**Status of the run.** The reproduction was run with the shipped runner against the pinned commit: 22 tests passed and none failed. The runner's output is in the capture block below.

Orders enter through the repository's own `LegacyOrderRouterHarness` adapter, which calls the production order path.

### The two files, and what each one covers

| File | Enters at | Covers |
| --- | --- | --- |
| `WaterfallObservationCountMovesJuniorPrincipal.t.sol` | the production entry points, as an account with no role | the whole sequence on a real `HousePool`: the arming through the owner's two documented actions, both harmful passes through `OrderRouter.settleLpEpoch(bytes[])`, the control branch from the same snapshot, the junior holder's receipts, the caller's receipts, the same sequence again with the senior coupon rate at zero, and again at all five release risk parameters |
| `WaterfallCompositionIsObservationDependent.t.sol` | `HousePoolWaterfallAccountingLib` directly | the library pair in isolation, on both the peak and the trough path, with the healthy case (`seniorHighWaterMark == seniorPrincipal`) asserted path independent, each of the three binding terms driven to the point where it binds, and a 41-point sweep showing that no later observation up to the senior entitlement returns any of the transferred principal, with Junior paid again only out of revenue beyond the whole remaining senior deficit |

The second file exists because the waterfall state is passed to both library functions as a struct argument, so the composition can be driven with no protocol harness at all, which isolates the defect from everything else in the pool. The first file is the one the severity rests on: no figure in it comes from a library return value. The split is read back out of `HousePool` storage after the pass returns, the funded redemption amounts are decoded from the pool's own settlement event, each side's receipts are read as settlement-token balances before and after, and the junior share price and the share count each branch exits are read through the junior vault's own view functions.

### What the test drives, and at which entry point

Every step is driven through a production entry point. In the end-to-end file there is no `vm.store`, no `stdstore` and no `checked_write`, so every figure the severity rests on is produced by the contracts' own validators. The library-level file is a differential over the two pure functions and supplies its starting state as the struct argument the pool passes them, which is what isolates the composition from the rest of the pool; the table above records that it enters at the library directly. The owner steps are performed by the owner account and are both on the programme's list of documented immediate owner actions; every other step is performed by an account holding no role, which the test asserts for the settling account: it is not the pool owner, not the router, not the router admin, not the engine admin module and neither tranche vault.

Figures below are the assertions the test makes. Where a quantity depends on protocol internals the assertion is relational and the value is printed, so the capture reports it rather than the report predicting it.

| # | Step | Entry point | The production code that validates it, and the canary asserted after it |
| --- | --- | --- | --- |
| 1 | Both tranches are seeded and trading is activated | `HousePool.initializeSeedPosition` then `activateTrading` (owner) | `HousePoolSeedLifecycleLib.tradingActivationReady` at `HousePool.sol:533`, and the senior-limit check at `:536`, which refuses activation while `maxSeniorShareBps >= 10_000`. Canary: `maxSeniorShareBps() == 8_000`, `maxSeniorExposureAssets() == 40_000_000e6` and both tranche seeds at `1e6`, the release values |
| 2 | The victim Junior LP deposits the ordinary way while the pool is healthy, and a settlement pass activates the request | `TrancheVault.requestDeposit` `:562`/`:569`, then `OrderRouter.settleLpEpoch` | the vault's deposit preflight, then `HousePool.sol:908` to `:1100-1118` to `HousePoolTrancheGateLib:19-41`. Junior capital has to exist before senior capacity does: at the release `maxSeniorShareBps` of `8_000`, `HousePoolSeniorCapacityLib._ratioExposureLimit` caps senior exposure at four times junior principal. Canary: the whole deposit request activates at its requested basis |
| 3 | The caller becomes an ordinary Senior LP the same way | the same two calls | the same gate, then the senior capacity check. Canary: `seniorVault.maxRequestDeposit(caller)` exceeds the deposit before it is made, the caller's senior share balance is non-zero after it, and `seniorPrincipal() == seniorHighWaterMark()` |
| 4 | Traders open positions on the basket | `OrderRouter.commitOrder` then `OrderRouter.executeOrder` | the engine's open path and the settlement sidecar. Canary: the order's status is `Executed`, the position's size is the committed size, and `terminalNavSnapshot().hasOpenPositions` is true |
| 5 | The owner recapitalizes the pool, and the amount above the standing senior deficit quarantines | `CfdEngine.recapitalizePool` `:632` (owner), then a settlement pass | `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent:111-137`: the gap fill at `:124-131`, the remainder to unassigned assets at `:134-136`. The fixture drives the degenerate case of a zero deficit, so the whole amount quarantines and is exact. Canary: `unassignedAssets()` equals the recapitalized amount, and `seniorPrincipal() == seniorHighWaterMark()`, so the quarantine restored nothing |
| 6 | Ordinary trader profit draws the pool down, wiping Junior and impairing Senior | the position opened at step 4 is left open and the basket moves against it; the loss is recognized by the next settlement pass | `absorbLoss:172-183` inside the reconcile. Canary: `juniorPrincipal() == 0`, `seniorPrincipal() < seniorHighWaterMark()`, and `unassignedAssets()` still the recapitalized amount |
| 7 | A tranche deposit is refused in that state, which is why the assignment is not optional | `TrancheVault.requestDeposit` on each vault | `HousePoolTrancheGateLib:28` and `:36-38`, reached through `TrancheVault._requireRequestDepositPreflight:1574-1601`. Asserted, not argued: `maxRequestDeposit(caller) == 0` on both vaults, and the request reverts on both: `TrancheVault__TerminallyWiped` on the Junior vault, which has shares and no assets at this point (`TrancheVault.sol:1585-1586`), and `TrancheVault__DepositsUnavailable` on the Senior vault |
| 8 | The owner performs the documented remedy and assigns the ownerless balance to Junior | `HousePool.assignUnassignedAssets(false, receiver)` `:660` (owner) | The senior branch is tried first and asserted to revert `HousePool__ExceedsSeniorDepositCapacity`. `:664-688`, including the reconcile at `:671` and the non-zero-balance check at `:673-676`, then the ungated junior credit at `:699`. Canary, the armed state: `juniorPrincipal()` equals the whole ownerless balance, `unassignedAssets() == 0`, `seniorPrincipal() < seniorHighWaterMark()`, `seniorHighWaterMark() - seniorPrincipal()` asserted greater than `juniorPrincipal()` and greater again than `juniorPrincipal()` plus the swing headroom the harmful passes use, so the whole swing sits inside the standing deficit, and junior share supply is non-zero. Senior principal and the mark are printed rather than predicted; the relation the rest of the report uses is the asserted one |
| 9 | The state is snapshotted, and both branches run from it | `vm.snapshotState()` | the two branches share every byte of starting state, which the restore asserts |
| 10 | A matured Junior redeem request stands in the queue, which is the real work each pass has to advance | `TrancheVault.requestRedeem` `:606` | the vault's redeem preflight and its epoch window. Canary: the account queuing it holds redeemable shares. The assignment at step 8 restarts the receiver's one-hour redeem cooldown (`TrancheVault.sol:1554`), so the test advances one hour after arming |
| 11 | **Attack, first pass.** The caller takes a settlement pass at a displaced mark | `OrderRouter.settleLpEpoch(bytes[])` `:479`, called by the roleless account | `OrderLiquidationBatchLogic:549-581` and the pool's authorization branch at `HousePool.sol:810-815`, which the delegate frame satisfies. Canary: `juniorPrincipal()` is no higher than the armed value less the junior redemption this pass funded. Two debits and no credit reach Junior on this pass: that funded redemption, and the senior coupon the reconcile accrues at the release rate (`HousePoolWaterfallAccountingLib.sol:77`). The revenue leg credits Junior only with revenue above the standing senior deficit (`:158`), and this pass's swing is sized inside that deficit, which the arming canary at step 8 asserts; the companion contract with the coupon rate at zero in protocol storage is where the arithmetic closes exactly. The funded figure is decoded from the pool's own settlement event and printed; `seniorPrincipal()` is strictly higher than the armed value; and the same event reports zero senior redemption assets |
| 12 | **Attack, second pass.** After the mark returns to where it started, the caller takes a second pass | the same entry point | the same validators, then `absorbLoss:167-184`. The swing carries a deliberate headroom above the armed junior tranche, so the loss exceeds junior principal: the junior-first branch empties the tranche at `:172-178` and the remainder above it reaches Senior at `:180-182`. Canary: `juniorPrincipal() == 0`; `seniorPrincipal()` strictly higher than the control branch's; `seniorHighWaterMark()` identical to the control branch's and to the armed mark; claimed equity identical across branches once each branch's own funded junior redemptions are added back; the senior gain exactly equal to the junior loss with no residual; and the settlement event again reporting zero senior redemption assets |
| 13 | Two further passes, over a revenue leg sized at half the remaining senior deficit and then its reversal, then a recovery pass over a revenue leg 26,000 USDG above the whole remaining deficit | the same entry point | after each of the two passes, `juniorPrincipal()` asserted still `0`, `seniorHighWaterMark()` asserted unchanged, and `seniorPrincipal()` asserted not below where the transfer left it. After the recovery pass, `seniorPrincipal()` and `seniorHighWaterMark()` are both asserted equal to the armed mark, Junior principal plus the redemption that pass funded is asserted within 1 USDG of the 26,000 USDG above the deficit, and it is printed beside the principal that moved, which it falls below |
| 14 | The victim's matured Junior redeem request of its whole remaining position is settled | `TrancheVault.requestRedeem` then `OrderRouter.settleLpEpoch` | `HousePool.sol:840` into `:957-987`. Asserted: the victim's settlement-token balance before and after in each branch, asserted above zero in the control branch and exactly zero in the attack branch, with the net assets quoted for the whole position read back as zero in the attack branch |
| 15 | The caller's matured Senior redeem request on a fixed slice of its position is settled | the same two calls | `HousePool.sol:845-863`. Asserted: the caller's settlement-token balance before and after in each branch, asserted strictly higher in the attack branch, with the slice asserted to fund in full in both so the comparison is on the share price rather than on a funding ceiling |
| 16 | **Control.** One pass over the identical basket history, with steps 10, 14 and 15, is taken from the same snapshot first; the attack branch restores that snapshot and takes **two** | the same entry points | same validators. Canary: the control pass's settlement event also reports zero senior redemption assets, and the control branch's split is the one every cross-branch assertion above compares against |

Steps 11 and 12 each need the pass to advance real queue work, because `_emitLpEpochSettled` reverts `HousePool__NoLpEpochProgress` otherwise (`HousePool.sol:1183-1185`) and the revert rolls the pass back. In the reproduction that work is the Junior holder's own matured redeem request: the victim's attempt to leave is what pays for the pass that empties it. The pool prices the junior redemption phase after the reconcile has already run (`HousePool.sol:835`, then `:840`), funds it up to the per-pass junior realization capacity, which the release ratio covenant caps at `min(juniorPrincipal, juniorPrincipal - ceil(exposure * (BPS - maxSeniorShareBps) / maxSeniorShareBps))`, and then subtracts the funded amount from `juniorPrincipal` at `:886`. So the first pass does take a bite out of Junior principal on its own account, independently of the waterfall, and the test's canary is written against the funded figure the pool's own settlement event reports rather than against a predicted one. On the second pass `juniorPrincipal` is zero when the redemption phase runs, so the head prices to zero and `HousePool.sol:957-987` takes the refund branch, which still increments `processedEpochs` at `:985`, and the shares are reclaimable. The transfer the report claims is therefore asserted as the armed junior tranche net of whatever the holder realized through its own redemption, which is the conservative direction: a redemption the holder is paid for is not value the caller took. Neither pass funds a senior redemption, which the canaries assert, so `scaleSeniorOnWithdraw` is never reached on the path the reproduction drives. The queued item does not have to be the victim's: both calls that produce one, an ordinary Junior deposit request while the pool is healthy and a small Junior redeem request afterwards, are permissionless, so a caller holding its own Junior position can supply the work itself. The reproduction uses the victim's; a caller supplying its own queue work is not driven. The only Junior queue work either file drives is the holder's own; the file's other queued items are the two ordinary tranche deposit requests the arming needs and the caller's own Senior redeem request, and none of them is Junior work the harmful passes consume.

### The object attacked, and how the caller comes to hold a position in it

The object attacked is the Junior tranche of the deployed `HousePool`, in the state the owner's own remedy leaves it: a tranche whose NAV has just been credited out of previously ownerless pool value while Senior still carries a deficit. The position the attack needs is Senior tranche shares, and **the caller creates its own position**: an ordinary permissionless deposit request through `TrancheVault.requestDeposit` while the pool is healthy, activated by any settlement pass (step 3). No role, no counterparty, no transfer of anybody else's shares, and nothing inherited from another party. The caller can hold the position long before the state it exploits exists, and the test drives it through the project's own deposit path rather than seeding a balance.

### The two branches, both driven

Both branches are taken from one `vm.snapshotState()` and both replay the identical sequence of basket levels, down against the open short and back to where it started, with the displaced level sized from the armed junior principal the arming reads back out of pool storage so that the whole tranche crosses. The only difference is how many settlement passes observe it: two in the attack branch, one in the control. The test reads `pool.seniorPrincipal()`, `pool.juniorPrincipal()` and `pool.seniorHighWaterMark()` at the end of each branch and asserts:

- the two splits differ, which is the finding;
- `sAttack + jAttack + fAttack == sControl + jControl + fControl`, where `f` is the junior redemption each branch's own passes funded out to the holder, so value is conserved and nothing leaves the pool unaccounted for;
- `sAttack - sControl == (jControl + fControl) - (jAttack + fAttack)`, so the senior gain is exactly the junior loss, with no residual term;
- `jAttack == 0`, so the Junior tranche ends empty;
- `seniorHighWaterMark()` is identical in both branches, so the entitlement that funds the next restoration is untouched.

### The controls, in the same files

Seven, so that the effect is attributable to the composition rather than to the fixture.

1. **The control branch itself**, above: one pass over the same basket history, from the same snapshot, through the same entry point.
2. **The healthy case, at library level.** The same round trip with `seniorHighWaterMark == seniorPrincipal` is asserted path independent, which pins the defect to the deficit-restoration asymmetry rather than to reconciliation in general. Its own canary asserts that the swing really moved the distributable value, so the control is not passing vacuously.
3. **The zero-coupon run.** The whole sequence is driven twice. Each run asserts `pool.seniorRateBps()` against protocol storage, `assertGt(..., 0)` in the first and `assertEq(..., 0)` in the second after the rate is finalized to zero through the owner's timelocked configuration path, and each asserts its own transfer, plus the junior redemption its own passes funded, against its own armed junior principal: within the coupon the release rate can accrue over the window in the first, and exactly in the second. Both armings assert that the junior principal is the whole assigned balance, so the two land on the same figure. That is what separates this from the coupon channel (Bug Description, not a duplicate).
4. **A freshly built state per branch**, at library level. Each branch of each differential constructs its own `WaterfallState` field by field rather than copying one, so no branch can write through another's reference. This matters because the two library functions open with a memory-to-memory struct assignment, which in Solidity is a reference rather than a copy, so a pass does write through the state it was handed; the file is structured around that rather than asserting against it.

5. **The release risk parameters.** A third end-to-end contract repeats the sequence with all five committed risk parameters, read back from the engine, and a position inside the release skew limit.

6. **Full recovery, no exit.** From one snapshot, one pass and two passes, then the same recovery. The branches end together, which is the limit of the transfer.

7. **Full recovery after a Senior exit.** The same, with the caller exiting 80% of its position first. Junior ends short by exactly what the caller gained.

### Attack steps, by hand

A reviewer who wants to reproduce this without the harness can follow the same sequence against a local deployment at the release parameters in `packages/perps/DEPLOYMENT.md`.

1. Hold Senior tranche shares. Request an ordinary Senior deposit while the pool is healthy and let a settlement pass activate it. Request a dust Junior deposit at the same time, which is what supplies the queue work later. Neither needs a role or a counterparty.
2. Wait for the pool to be in the state the owner's remedy produces: a senior deficit, `juniorPrincipal` credited by `assignUnassignedAssets(false, receiver)`, and `unassignedAssets` back to zero. The pool's own view functions report all three.
3. Queue a dust Junior redeem request from that position, or let any other matured Junior item in the queue serve, and wait for maturity. One LP epoch is one hour.
4. When the basket moves the pool's distributable value up, call `OrderRouter.settleLpEpoch(pythUpdateData)` with a valid `PoolReconcile` basket and the exact Pyth fee. The reconcile routes the surplus through `distributeRevenue:147-151` and closes the senior deficit by the size of the swing.
5. When the basket returns to where it started, queue another dust Junior redeem and take a second pass through the same entry point. `absorbLoss` charges the reversal to Junior first, up to `min(junior principal, swing)`, at `:172-178`, and reaches Senior at `:180-182` only for any part above the tranche.
6. Read `juniorPrincipal()`. It is lower, `seniorPrincipal()` is higher by exactly the same amount, and `seniorHighWaterMark()` has not moved, so a later revenue leg fills the remaining deficit first.
7. Repeat over any later oscillation until `juniorPrincipal()` is zero. No pass is required to be profitable for the trader side and no price influence is needed: a dip and a recovery works in the other order.
8. Redeem the Senior shares from step 1. The proceeds are priced against the raised `seniorPrincipal` (`HousePool.sol:845-848`), with no ratio cap on the senior side.

### The counterfactual: the fix patch

The patch is the narrow change in the Bug Description's recommended fix: one additional field on `WaterfallState`, with one new `HousePool` storage slot, recording senior principal restored out of revenue, incremented at `distributeRevenue:150` and consumed by `absorbLoss` before it reaches Junior. It ships as `reverse_revenue_restoration_before_charging_junior.patch` and applies with `patch -p1`.

Applied and run, it closes the peak path, which is the path the end to end file drives: with the coupon at zero the attack branch ends at the control branch's senior principal, and at the release rate Junior no longer ends at zero but keeps the armed tranche less the senior coupon and less the redemption the passes funded, both ordinary debits the patch does not touch. It does not close the trough path. A dip charges Junior while no restoration is outstanding and the recovery then restores Senior, so the library-level trough test still ends at a different split from the peak path. Only the principal basis in the Bug Description is independent of the path. The assertions that record the peak-path defect stop holding, which is the counterfactual. `run_with_fix_local_only.sh` applies the patch to a private copy of the checkout and runs the same tests in the same kind of namespace; `CAPTURE_WITH_FIX.log` records 8 tests passing and 14 failing. The capture below is the run of the unmodified source. `RUN.md` lists the affected assertions by test and by assertion, and the two in `test_impairedSplitDependsOnHowManyTimesTheRoundTripWasObserved` are `assertGt(attack.seniorPrincipal, control.seniorPrincipal, ...)` and `assertEq(attack.juniorPrincipal, 0, ...)`. The healthy library-level control is unaffected in outcome: with no senior restoration outstanding the new field stays zero, so the junior-first branch runs exactly as it does today. The structural alternative in the Bug Description, a principal basis per tranche, is not patched here because it changes how every capital event writes principal, which is a release decision rather than a demonstration.

### What is asserted, and what is argued

Sorted by how much of the severity each row carries. **Asserted** means the reproduction drives the
claim against the contracts and asserts it, and the capture block below records the result. **Argued** means the row is
settled by reading the source rather than by anything the reproduction executes.

| Claim | How it is settled | What settles it |
| --- | --- | --- |
| An extra observation of a closed round trip moves Junior principal into Senior principal, in pool storage | Asserted | both branches from one snapshot, `pool.seniorPrincipal()` and `pool.juniorPrincipal()` read after each, asserted unequal across branches, with claimed equity asserted identical once each branch's own funded junior redemptions are added back, and `jAttack == 0` |
| The caller needs no role | Asserted | both harmful passes executed as an account asserted not to be the pool owner, the router, the router admin, the engine admin module or either tranche vault, through `OrderRouter.settleLpEpoch(bytes[])`. `HousePool.reconcile()` is `onlyVault` (`:1471`) with no production caller anywhere in `packages/perps/src`, and the test does not use it |
| A full recovery with no Senior exit brings the branches back together, and a Senior exit before it makes the shortfall final | Asserted | `test_aFullRecoveryWithNoSeniorExitBringsTheBranchesBackTogether` and `test_aSeniorExitBeforeTheRecoveryLeavesJuniorShort`, in all three end to end contracts, each with a control branch from one snapshot and a recovery at the opening basket level. Without an exit Junior value is asserted equal within 100 USDG, and measured equal to the atom. With an exit of 80% of the caller's position, the caller's gain is asserted equal to Junior's shortfall. The library sweep and the source enumeration show that nothing short of filling the deficit reaches Junior: `seniorPrincipal` decreases at exactly two sites, `HousePoolWaterfallAccountingLib.sol:130` and `:180-182`, and neither credits Junior |
| Every arming step is accepted by its own production validator, in that order | Asserted | steps 1 to 8 above, each driven through its own entry point with the canary asserted after it, and no storage write anywhere in the test |
| The caller gains against not taking the extra pass | Asserted | the 80% exit test: at the release risk parameters the caller's cash is 75,150.81 USDG against 51,153.73, and its total after the recovery 95,150.98 against 71,153.90. That total is below its 99,999 USDG deposit because Senior was impaired before the attack, so the gain is relative. Senior gains are pro rata and the fixture's caller holds nearly all of the Senior tranche |
| A Junior holder's redemption proceeds fall | Asserted | the holder's whole remaining Junior position exited in each branch, with the victim's settlement-token balance read before and after each, asserted above zero in the control branch and zero in the attack branch, and the net assets quoted for the whole position read back as zero in the attack branch. The share count each branch exited is printed rather than asserted equal across branches, because each branch exits whatever position it still holds; what the comparison turns on is that the attack branch's prices to nothing. In the control branch that exit realizes only part of the position on the pass, 4,996.20 of 29,997.24 USDG at the release risk parameters, because of the covenant. The per-pass realization cap from `juniorWithdrawalRatioCap` is derived from the release `maxSeniorShareBps` and the armed senior exposure under the loss mechanism in the Bug Description rather than measured here, so no claim of same-block withdrawable cash is made |
| The senior gain is exactly the junior loss, and no senior redemption is funded | Asserted | `sAttack - sControl == (jControl + fControl) - (jAttack + fAttack)` with no residual term, and the pool's own settlement event reporting zero senior redemption assets on both harmful passes and on the control pass, which is the gate on the only call site of `scaleSeniorOnWithdraw` (`HousePool.sol:857`) |
| The effect is not the declared coupon channel | Asserted | the whole sequence driven at the release coupon rate and at zero, with `pool.seniorRateBps()` asserted against protocol storage in each, and each run's transfer plus its own funded junior redemption asserted against its armed junior principal: exactly at the zero rate, and within the coupon the release rate can accrue over the window |
| The effect is attributable to the senior deficit and not to reconciliation in general | Asserted | the healthy library-level case asserted path independent, with its own canary proving the swing moved the distributable value |
| The five release risk parameters are live and the position is admitted at the release skew ratio | Asserted | `test_theReleaseRiskParametersAreLiveAndTheOpenIsInsideTheSkewLimit` reads all five back from the engine, and the inherited tests run the whole sequence on that contract |
| The release senior limits are the committed ones | Asserted | `maxSeniorShareBps()` and `maxSeniorExposureAssets()` asserted against the values in `packages/perps/DEPLOYMENT.md` |
| The transfer law is `min(senior deficit, junior principal, swing)` | Asserted at each of the three binding terms | each term is driven to the point where it binds and the other two do not, in `test_theAmountThatCrossesIsBoundedByTheDeficitTheTrancheAndTheSwing`, including the deficit-binds case, a 60,000 tranche against a 50,000 deficit, where the senior side is asserted to stop at the high-water mark and the 10,000 remainder is asserted to stand; the general form is arithmetic over `distributeRevenue:147-151` and `absorbLoss:167-184` |
| Junior principal cannot be credited while Senior is impaired by any route other than the assignment | Asserted at the revenue and pending revenue sites and, before the assignment, at the deposit site; argued for the deposit site after the assignment and for the seed | `juniorPrincipal` increases at exactly five sites in `packages/perps/src`, each gate opened and listed in the Bug Description. The revenue site is covered by the library sweep and the pending revenue bucket by `test_thePendingRevenueBucketReachesJuniorOnlyAfterTheSeniorDeficit`, where Junior receives only the unit above the deficit. The seed is `onlyOwner` and once per tranche (`HousePool.sol:730`), performed at bootstrap. What step 7 asserts is the adjacent fact that a tranche deposit is refused outright in the pre-assignment state, which on the Senior vault is the unassigned-assets clause at `HousePoolTrancheGateLib:28` rather than the impairment clause at `:36-38`, and on the Junior vault is the terminal-wipe check at `TrancheVault.sol:1585-1586` |
| The delegate frame satisfies the pool's authorization branch by construction | Argued from source, and asserted by consequence | `OrderLiquidationBatchLogic.sol:549-581` against `HousePool.sol:810-815`. The passes at steps 11 and 12 succeeding from a roleless account is what the reproduction asserts |
| No public fix of this mechanism exists between the scoped commit and now | Argued, from the repository's history | the listed repository's tip is the scoped commit; its public source carries one later commit and that commit touches no file in `packages/perps/src`; and the newest commit touching the defect library is an ancestor of the pin (Bug Description, time bar) |

### The captures

The runner's output, verbatim, isolation self-check lines first.

```text
== isolation self-check, from inside the network namespace ==
started at 2026-11-08T23:37:18Z (UTC)
interfaces present: lo
ipv4 routes: 0    ipv6 routes: 0
outbound tcp connect probe to 1.1.1.1:443 refused, as required
outbound tcp connect probe to 8.8.8.8:53 refused, as required
loopback connect probe to 127.0.0.1:9 returned without leaving the namespace
== isolation self-check passed ==

== toolchain and sources ==
forge Version: 1.6.0-v1.7.0 Commit SHA: f83bad912a9dba7bf0371def1e70bb1896048356 Build Timestamp: 2026-04-28T14:07:10.568929921Z (1777385230) 
test files placed under packages/perps/test/perps/waterfall_observation/
83cc6cf276620f439d8c7500e43ca50370214c41304b7e26c9ee682274c477ab  src/libraries/HousePoolWaterfallAccountingLib.sol
1cfebbedd22401966024276920276156bbaae03b21dba41af20d0e6217bb86af  src/libraries/HousePoolReconcilePlanLib.sol
92f0e835024dd19e12e00b312954921204bd09e46ab2c25028f084b0213561bd  src/libraries/HousePoolTrancheGateLib.sol
92c7946d7fd364487da95dff4277a2f432c7b11cfd9f4063515540dd311c379f  src/libraries/HousePoolSeniorCapacityLib.sol
57cde380a49f5fa0607fa101e790e64f539b32a52da7baaae520daac0ee40159  src/HousePool.sol
8f8318db0889f5d098d2f5c8c6afc41c473d1ca516817fe2363ba3899bb48d7d  src/OrderRouter.sol
5573fef6edfd1c09f86ba93a55fa88f0607929564f2406c6e25fdf320d939d5c  src/router/OrderLiquidationBatchLogic.sol
all seven source fingerprints match the pinned commit

== compiling (compiler diagnostics go to standard output) ==
compile exit status: 0

== test run ==
command: forge test --root packages/perps --threads 1 --match-path 'test/perps/waterfall_observation/*.t.sol' -vv
lines reaching CAPTURE.log are filtered with: grep -vE '^(Warning|Note) \(|^warning|^[[:space:]]*[0-9]+ \||^[[:space:]]*\||^[[:space:]]*-->'
the unfiltered stream is on standard output

No files changed, compilation skipped

Ran 6 tests for test/perps/waterfall_observation/WaterfallCompositionIsObservationDependent.t.sol:WaterfallCompositionIsObservationDependentTest
[PASS] test_aDipAndARecoveryMovesTheSameAmountInTheOtherOrder() (gas: 12333)
Logs:
  down then back up
    senior principal       80000000000
    junior principal       0
    senior high-water mark 100000000000
  USDG moved on the trough path 30000000000

[PASS] test_anUnimpairedSeniorTrancheIsObservationIndependent() (gas: 12562)
Logs:
  healthy senior, one pass
    senior principal       50000000000
    junior principal       30000000000
    senior high-water mark 50000000000
  healthy senior, two passes
    senior principal       50000000000
    junior principal       30000000000
    senior high-water mark 50000000000

[PASS] test_impairedSplitDependsOnHowManyTimesTheRoundTripWasObserved() (gas: 16266)
Logs:
  one pass over the round trip
    senior principal       50000000000
    junior principal       30000000000
    senior high-water mark 100000000000
  two passes over the round trip
    senior principal       80000000000
    junior principal       0
    senior high-water mark 100000000000
  senior principal, control then attack 50000000000 80000000000
  junior principal, control then attack 30000000000 0
  USDG moved out of junior principal    30000000000

[PASS] test_noLaterObservationReturnsTheTransferredPrincipal() (gas: 91203)
Logs:
  senior deficit junior must wait out before it is paid again 20000000000

[PASS] test_theAmountThatCrossesIsBoundedByTheDeficitTheTrancheAndTheSwing() (gas: 12930)
Logs:
  swing binds, USDG moved       10000000000
  the tranche binds, USDG moved 30000000000
  the deficit binds, USDG moved 50000000000

[PASS] test_thePendingRevenueBucketReachesJuniorOnlyAfterTheSeniorDeficit() (gas: 3030)
Suite result: ok. 6 passed; 0 failed; 0 skipped; finished in 4.39ms (3.98ms CPU time)

Ran 6 tests for test/perps/waterfall_observation/WaterfallObservationCountMovesJuniorPrincipal.t.sol:WaterfallObservationCountMovesJuniorPrincipalAtReleaseRiskParametersTest
[PASS] test_aFullRecoveryWithNoSeniorExitBringsTheBranchesBackTogether() (gas: 12248720)
Logs:
  recapitalized while healthy
    senior principal       100000914794
    junior principal       26099554698
    senior high-water mark 100000914794
    unassigned assets      30000000000
    pool assets            156100469492
  drawn down
    senior principal       63940469492
    junior principal       0
    senior high-water mark 100001828044
    unassigned assets      30000000000
    pool assets            156100469492
  ARMED
    senior principal       63940469492
    junior principal       30000000000
    senior high-water mark 100001828044
    unassigned assets      0
    pool assets            156100469492
  no exit: junior value, control then attack 56098641448 56098641448
  no exit: senior principal, control then attack 100001828044 100001828044

[PASS] test_aSeniorExitBeforeTheRecoveryLeavesJuniorShort() (gas: 13532225)
Logs:
  recapitalized while healthy
    senior principal       100000914794
    junior principal       26099554698
    senior high-water mark 100000914794
    unassigned assets      30000000000
    pool assets            156100469492
  drawn down
    senior principal       63940469492
    junior principal       0
    senior high-water mark 100001828044
    unassigned assets      30000000000
    pool assets            156100469492
  ARMED
    senior principal       63940469492
    junior principal       30000000000
    senior high-water mark 100001828044
    unassigned assets      0
    pool assets            156100469492
  exit: caller cash, control then attack      51153731849 75150814927
  exit: caller remaining value, control then attack 20000165295 20000165295
  exit: caller total, control then attack     71153897144 95150980222
  exit: caller senior deposit                 99999000000
  exit: junior value, control then attack     84945570777 60948487699

[PASS] test_anExtraObservationMovesJuniorPrincipalIntoSenior() (gas: 12796023)
Logs:
  recapitalized while healthy
    senior principal       100000914794
    junior principal       26099554698
    senior high-water mark 100000914794
    unassigned assets      30000000000
    pool assets            156100469492
  drawn down
    senior principal       63940469492
    junior principal       0
    senior high-water mark 100001828044
    unassigned assets      30000000000
    pool assets            156100469492
  ARMED
    senior principal       63940469492
    junior principal       30000000000
    senior high-water mark 100001828044
    unassigned assets      0
    pool assets            156100469492
  CONTROL, one pass over the round trip
    senior principal       63942221285
    junior principal       29997238227
    senior high-water mark 100001828044
    unassigned assets      0
    pool assets            156099459512
  attack, after the first pass
    senior principal       94042220933
    junior principal       29997238227
    senior high-water mark 100001828044
    unassigned assets      0
    pool assets            156099459512
  ATTACK, two passes over the round trip
    senior principal       93939459512
    junior principal       0
    senior high-water mark 100001828044
    unassigned assets      0
    pool assets            156099459512
  senior principal, control then attack 63942221285 93939459512
  junior principal, control then attack 29997238227 0
  USDG moved out of junior principal    29997238227
  pool assets, control then attack      156099459512 156099459512
  junior redemption funded, control then attack 1009980 1009980
  after two further passes
    senior principal       93939459512
    junior principal       0
    senior high-water mark 100001828044
    unassigned assets      0
    pool assets            156099459512
  after a recovery leg above the remaining deficit
    senior principal       100001828044
    junior principal       25000457011
    senior high-water mark 100001828044
    unassigned assets      0
    pool assets            155099916703
  recovery leg: remaining senior deficit    6062368532
  recovery leg: revenue above that deficit  26000000000
  recovery leg: junior principal plus funded 25999999820
  recovery leg: principal that moved earlier 29997238227

[PASS] test_theHolderReceivesLessAndTheCallerReceivesMore() (gas: 13362703)
Logs:
  recapitalized while healthy
    senior principal       100000914794
    junior principal       26099554698
    senior high-water mark 100000914794
    unassigned assets      30000000000
    pool assets            156100469492
  drawn down
    senior principal       63940469492
    junior principal       0
    senior high-water mark 100001828044
    unassigned assets      30000000000
    pool assets            156100469492
  ARMED
    senior principal       63940469492
    junior principal       30000000000
    senior high-water mark 100001828044
    unassigned assets      0
    pool assets            156100469492
  junior holder USDG received, control then attack 4996197269 0
  caller USDG received, control then attack        1000036529 1469157207
  junior share price, control then attack          29997238227 0
  junior shares exited, control then attack        781178867179624608968088 781178867179766643126734

[PASS] test_theReleaseRiskParametersAreLiveAndTheOpenIsInsideTheSkewLimit() (gas: 7420985)
Logs:
  recapitalized while healthy
    senior principal       100000914794
    junior principal       26099554698
    senior high-water mark 100000914794
    unassigned assets      30000000000
    pool assets            156100469492
  drawn down
    senior principal       63940469492
    junior principal       0
    senior high-water mark 100001828044
    unassigned assets      30000000000
    pool assets            156100469492
  ARMED
    senior principal       63940469492
    junior principal       30000000000
    senior high-water mark 100001828044
    unassigned assets      0
    pool assets            156100469492

[PASS] test_theTransferIsTheWholeJuniorTrancheAtTheReleaseCouponRate() (gas: 10423426)
Logs:
  recapitalized while healthy
    senior principal       100000914794
    junior principal       26099554698
    senior high-water mark 100000914794
    unassigned assets      30000000000
    pool assets            156100469492
  drawn down
    senior principal       63940469492
    junior principal       0
    senior high-water mark 100001828044
    unassigned assets      30000000000
    pool assets            156100469492
  ARMED
    senior principal       63940469492
    junior principal       30000000000
    senior high-water mark 100001828044
    unassigned assets      0
    pool assets            156100469492
  senior coupon rate, basis points      800
  USDG moved out of junior principal    29997238227
  junior redemption funded to the holder 1009980
  junior tranche at the armed state     30000000000

Suite result: ok. 6 passed; 0 failed; 0 skipped; finished in 181.12ms (163.92ms CPU time)

Ran 5 tests for test/perps/waterfall_observation/WaterfallObservationCountMovesJuniorPrincipal.t.sol:WaterfallObservationCountMovesJuniorPrincipalAtZeroCouponTest
[PASS] test_aFullRecoveryWithNoSeniorExitBringsTheBranchesBackTogether() (gas: 12022196)
Logs:
  recapitalized while healthy
    senior principal       100000002191
    junior principal       39998987810
    senior high-water mark 100000002191
    unassigned assets      30000000000
    pool assets            169998990001
  drawn down
    senior principal       49998990001
    junior principal       0
    senior high-water mark 100000002191
    unassigned assets      30000000000
    pool assets            169998990001
  ARMED
    senior principal       49998990001
    junior principal       30000000000
    senior high-water mark 100000002191
    unassigned assets      0
    pool assets            169998990001
  no exit: junior value, control then attack 69998987810 69998987810
  no exit: senior principal, control then attack 100000002191 100000002191

[PASS] test_aSeniorExitBeforeTheRecoveryLeavesJuniorShort() (gas: 13303804)
Logs:
  recapitalized while healthy
    senior principal       100000002191
    junior principal       39998987810
    senior high-water mark 100000002191
    unassigned assets      30000000000
    pool assets            169998990001
  drawn down
    senior principal       49998990001
    junior principal       0
    senior high-water mark 100000002191
    unassigned assets      30000000000
    pool assets            169998990001
  ARMED
    senior principal       49998990001
    junior principal       30000000000
    senior high-water mark 100000002191
    unassigned assets      0
    pool assets            169998990001
  exit: caller cash, control then attack      39998791132 63997742614
  exit: caller remaining value, control then attack 19999800001 19999800000
  exit: caller total, control then attack     59998591133 83997542614
  exit: caller senior deposit                 99999000000
  exit: junior value, control then attack     109999396676 86000445195

[PASS] test_anExtraObservationMovesJuniorPrincipalIntoSenior() (gas: 12620074)
Logs:
  recapitalized while healthy
    senior principal       100000002191
    junior principal       39998987810
    senior high-water mark 100000002191
    unassigned assets      30000000000
    pool assets            169998990001
  drawn down
    senior principal       49998990001
    junior principal       0
    senior high-water mark 100000002191
    unassigned assets      30000000000
    pool assets            169998990001
  ARMED
    senior principal       49998990001
    junior principal       30000000000
    senior high-water mark 100000002191
    unassigned assets      0
    pool assets            169998990001
  CONTROL, one pass over the round trip
    senior principal       49998990001
    junior principal       29998990000
    senior high-water mark 100000002191
    unassigned assets      0
    pool assets            169997980001
  attack, after the first pass
    senior principal       80098990001
    junior principal       29998990000
    senior high-water mark 100000002191
    unassigned assets      0
    pool assets            169997980001
  ATTACK, two passes over the round trip
    senior principal       79997980001
    junior principal       0
    senior high-water mark 100000002191
    unassigned assets      0
    pool assets            169997980001
  senior principal, control then attack 49998990001 79997980001
  junior principal, control then attack 29998990000 0
  USDG moved out of junior principal    29998990000
  pool assets, control then attack      169997980001 169997980001
  junior redemption funded, control then attack 1010000 1010000
  after two further passes
    senior principal       79997980001
    junior principal       0
    senior high-water mark 100000002191
    unassigned assets      0
    pool assets            169997980001
  after a recovery leg above the remaining deficit
    senior principal       100000002191
    junior principal       25000000548
    senior high-water mark 100000002191
    unassigned assets      0
    pool assets            168997980739
  recovery leg: remaining senior deficit    20002022190
  recovery leg: revenue above that deficit  26000000000
  recovery leg: junior principal plus funded 25999999810
  recovery leg: principal that moved earlier 29998990000

[PASS] test_theHolderReceivesLessAndTheCallerReceivesMore() (gas: 13133896)
Logs:
  recapitalized while healthy
    senior principal       100000002191
    junior principal       39998987810
    senior high-water mark 100000002191
    unassigned assets      30000000000
    pool assets            169998990001
  drawn down
    senior principal       49998990001
    junior principal       0
    senior high-water mark 100000002191
    unassigned assets      30000000000
    pool assets            169998990001
  ARMED
    senior principal       49998990001
    junior principal       30000000000
    senior high-water mark 100000002191
    unassigned assets      0
    pool assets            169998990001
  junior holder USDG received, control then attack 4998989452 0
  caller USDG received, control then attack        999999999 1599991919
  junior share price, control then attack          29998990000 0
  junior shares exited, control then attack        1202564050562276138169277 1202564050562276138169277

[PASS] test_theTransferIsTheWholeJuniorTrancheAtTheReleaseCouponRate() (gas: 10198342)
Logs:
  recapitalized while healthy
    senior principal       100000002191
    junior principal       39998987810
    senior high-water mark 100000002191
    unassigned assets      30000000000
    pool assets            169998990001
  drawn down
    senior principal       49998990001
    junior principal       0
    senior high-water mark 100000002191
    unassigned assets      30000000000
    pool assets            169998990001
  ARMED
    senior principal       49998990001
    junior principal       30000000000
    senior high-water mark 100000002191
    unassigned assets      0
    pool assets            169998990001
  senior coupon rate, basis points      0
  USDG moved out of junior principal    29998990000
  junior redemption funded to the holder 1010000
  junior tranche at the armed state     30000000000

Suite result: ok. 5 passed; 0 failed; 0 skipped; finished in 156.83ms (146.00ms CPU time)

Ran 5 tests for test/perps/waterfall_observation/WaterfallObservationCountMovesJuniorPrincipal.t.sol:WaterfallObservationCountMovesJuniorPrincipalTest
[PASS] test_aFullRecoveryWithNoSeniorExitBringsTheBranchesBackTogether() (gas: 12033789)
Logs:
  recapitalized while healthy
    senior principal       100000914794
    junior principal       39998075230
    senior high-water mark 100000914794
    unassigned assets      30000000000
    pool assets            169998990024
  drawn down
    senior principal       49998990024
    junior principal       0
    senior high-water mark 100001828044
    unassigned assets      30000000000
    pool assets            169998990024
  ARMED
    senior principal       49998990024
    junior principal       30000000000
    senior high-water mark 100001828044
    unassigned assets      0
    pool assets            169998990024
  no exit: junior value, control then attack 69997161980 69997161980
  no exit: senior principal, control then attack 100001828044 100001828044

[PASS] test_aSeniorExitBeforeTheRecoveryLeavesJuniorShort() (gas: 13315894)
Logs:
  recapitalized while healthy
    senior principal       100000914794
    junior principal       39998075230
    senior high-water mark 100000914794
    unassigned assets      30000000000
    pool assets            169998990024
  drawn down
    senior principal       49998990024
    junior principal       0
    senior high-water mark 100001828044
    unassigned assets      30000000000
    pool assets            169998990024
  ARMED
    senior principal       49998990024
    junior principal       30000000000
    senior high-water mark 100001828044
    unassigned assets      0
    pool assets            169998990024
  exit: caller cash, control then attack      40000252559 63997743055
  exit: caller remaining value, control then attack 20000165295 20000165294
  exit: caller total, control then attack     60000417854 83997908349
  exit: caller senior deposit                 99999000000
  exit: junior value, control then attack     109997570599 86000080104

[PASS] test_anExtraObservationMovesJuniorPrincipalIntoSenior() (gas: 12706828)
Logs:
  recapitalized while healthy
    senior principal       100000914794
    junior principal       39998075230
    senior high-water mark 100000914794
    unassigned assets      30000000000
    pool assets            169998990024
  drawn down
    senior principal       49998990024
    junior principal       0
    senior high-water mark 100001828044
    unassigned assets      30000000000
    pool assets            169998990024
  ARMED
    senior principal       49998990024
    junior principal       30000000000
    senior high-water mark 100001828044
    unassigned assets      0
    pool assets            169998990024
  CONTROL, one pass over the round trip
    senior principal       50000359859
    junior principal       29997620181
    senior high-water mark 100001828044
    unassigned assets      0
    pool assets            169997980040
  attack, after the first pass
    senior principal       80100359859
    junior principal       29997620181
    senior high-water mark 100001828044
    unassigned assets      0
    pool assets            169997980040
  ATTACK, two passes over the round trip
    senior principal       79997980040
    junior principal       0
    senior high-water mark 100001828044
    unassigned assets      0
    pool assets            169997980040
  senior principal, control then attack 50000359859 79997980040
  junior principal, control then attack 29997620181 0
  USDG moved out of junior principal    29997620181
  pool assets, control then attack      169997980040 169997980040
  junior redemption funded, control then attack 1009984 1009984
  after two further passes
    senior principal       79997980040
    junior principal       0
    senior high-water mark 100001828044
    unassigned assets      0
    pool assets            169997980040
  after a recovery leg above the remaining deficit
    senior principal       100001828044
    junior principal       25000457011
    senior high-water mark 100001828044
    unassigned assets      0
    pool assets            168998437055
  recovery leg: remaining senior deficit    20003848004
  recovery leg: revenue above that deficit  26000000000
  recovery leg: junior principal plus funded 25999999996
  recovery leg: principal that moved earlier 29997620181

[PASS] test_theHolderReceivesLessAndTheCallerReceivesMore() (gas: 13146454)
Logs:
  recapitalized while healthy
    senior principal       100000914794
    junior principal       39998075230
    senior high-water mark 100000914794
    unassigned assets      30000000000
    pool assets            169998990024
  drawn down
    senior principal       49998990024
    junior principal       0
    senior high-water mark 100001828044
    unassigned assets      30000000000
    pool assets            169998990024
  ARMED
    senior principal       49998990024
    junior principal       30000000000
    senior high-water mark 100001828044
    unassigned assets      0
    pool assets            169998990024
  junior holder USDG received, control then attack 4996706546 0
  caller USDG received, control then attack        1000036529 1599977308
  junior share price, control then attack          29997620181 0
  junior shares exited, control then attack        1201783603817818526911833 1201783603817989823977888

[PASS] test_theTransferIsTheWholeJuniorTrancheAtTheReleaseCouponRate() (gas: 10209417)
Logs:
  recapitalized while healthy
    senior principal       100000914794
    junior principal       39998075230
    senior high-water mark 100000914794
    unassigned assets      30000000000
    pool assets            169998990024
  drawn down
    senior principal       49998990024
    junior principal       0
    senior high-water mark 100001828044
    unassigned assets      30000000000
    pool assets            169998990024
  ARMED
    senior principal       49998990024
    junior principal       30000000000
    senior high-water mark 100001828044
    unassigned assets      0
    pool assets            169998990024
  senior coupon rate, basis points      800
  USDG moved out of junior principal    29997620181
  junior redemption funded to the holder 1009984
  junior tranche at the armed state     30000000000

Suite result: ok. 5 passed; 0 failed; 0 skipped; finished in 161.63ms (153.69ms CPU time)

Ran 4 test suites in 515.51ms (503.97ms CPU time): 22 tests passed, 0 failed, 0 skipped (22 total tests)

forge test exit status: 0
finished at 2026-11-08T23:39:12Z (UTC)
```

### What this proof of concept does not establish

- **How much is at risk on the deployed pool.** The figures are the figures of the fixture's arming. What any particular `HousePool` carries in its Junior tranche, and how large an assignment its owner performs, is deployment state, and nothing here surveys it. The code establishes the law, `min(senior deficit, junior principal, swing)` per round trip, accumulating across round trips up to the senior deficit outstanding when the passes begin, which empties the tranche whenever that deficit is at least as large as it.
- **Every recovery size, end to end.** The end to end file drives one recovery leg above the remaining senior deficit, 26,000 USDG above it. That excess clears the 25,000 USDG the release senior share limit reserves against this entitlement, because at a much smaller excess the pass funded nothing, advanced no queue item and reverted `HousePool__NoLpEpochProgress`. Other sizes are carried by the library-level sweep, which observes 41 distributable values at even steps from zero up to the senior entitlement, eight of them above the post-transfer claimed equity and so revenue legs, with Junior asserted at zero in every one, and then one unit beyond the entitlement, where Junior is asserted to receive that one unit. That every intermediate value behaves the same way is the arithmetic of `distributeRevenue`, which fills the senior deficit at `:147-152` before it credits Junior at `:158`.
- **What the Junior holder received in total across every pass.** What the receipts comparison covers is the terminal exit of whatever Junior position each branch still holds at the end of it. Neither branch's earlier queue-work payouts to the holder are captured in that comparison, so they are neither added to the figure nor reported beside it; the differential test prints its own branches' funded totals and the two sets of figures are not combined.
- **How often any routine settlement cadence would reproduce it unattended.** The drift happens on any oscillation while Senior is impaired, so a pool whose epochs are settled on a routine schedule bleeds in the same direction. The cadence on the deployed pool is operational behaviour and is not covered here. It widens the exposure rather than narrowing it.
- **Profit above the caller's deposit.** The caller in this fixture was impaired before the attack and ends below its deposit. The measured gain is against not taking the extra pass.
- **Any other release book.** The release-parameter contract is one admissible book: a single short opened at a basket level of 0.85 and held to 1.90. Other books are not surveyed.
