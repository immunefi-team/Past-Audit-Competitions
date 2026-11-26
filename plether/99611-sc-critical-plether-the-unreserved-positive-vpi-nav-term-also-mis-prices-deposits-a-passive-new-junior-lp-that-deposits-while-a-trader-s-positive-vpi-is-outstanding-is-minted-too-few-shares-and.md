# \#99611 [SC-Critical] Plether: the unreserved positive-VPI NAV term also mis-prices DEPOSITS — a passive new Junior LP that deposits while a trader's positive VPI is outstanding is minted too few shares and loses to the LPs already in the pool

**Submitted on Nov 14th 2026 at 13:41:20 UTC by @SombraX for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99611
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

# The unreserved positive-VPI NAV term also mis-prices deposits

**Severity:** filed conservatively at **Low** — the exact dropdown #99588 used before it was raised to Critical — with the real class argued below. This is the deposit-side of the defect accepted **Critical** as report #99588, so do not under-read it. The victim loses part of the principal it just deposited (bounded per event by its pro-rata share of `V` ≈ 8 bps of depth); the honest value-loss category is therefore **Critical "Direct theft of user funds"** on the upside, with **Medium "Griefing"** as the floor if triage prefers the no-net-principal-leaves-protocol framing. "Theft of unclaimed yield" does **not** fit — the loss is deposited principal, not yield. We leave the final level to triage, flagged against #99588.

**Impact:** Contract fails to deliver promised returns, but doesn't lose value (see severity note — the real impact is the same class as #99588, Critical)

**Asset:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src (`HousePool.sol`, `TrancheVault.sol`, `HousePoolAccountingLib.sol`, `CfdEngineSettlementSidecar.sol`)

**Pin:** `a82d4fefb02768e5b3bddcc97dee371eee24fea8`

## What is and is not claimed

Claimed: the same positive VPI charge `V` that a trader pays on a skew-**increasing** open — booked immediately into pool accounted assets as LP revenue with no reserve for its own refund — also inflates the Junior principal used to price **deposits** that settle in that LP epoch. A passive new Junior depositor whose deposit epoch is settled while any trader has outstanding positive VPI is minted **too few shares** (it buys in at an NAV that includes a refundable liability). When the trader closes and `V` is refunded out of pool cash, Junior NAV falls and the depositor's shares are worth less than it deposited. The lost value accrues to the LPs that were already in the pool. The depositor holds no prior stake, runs no trade, and performs no privileged action — it only deposits.

Not claimed: theft of protocol principal or of trader collateral; a freeze. This is not a new root cause relative to S-08 (#99588) — it is the same unreserved NAV term reached through the deposit path. It is reported because it removes S-08's majority-Junior-holder precondition and broadens the victim set to any honest depositor.

## Summary

On a skew-increasing open the settlement path books the positive VPI charge `V` straight into the pool's accounted assets as revenue (`CfdEngineSettlementSidecar.sol:446-463` → `HousePool.recordClaimantInflow(... Revenue ...)`). The LP reconcile has no term for outstanding **refundable** positive VPI (`HousePoolAccountingLib.sol:61-79`). At LP-epoch settlement, `_settleLpEpoch` first reconciles that `V` into `juniorPrincipal` (`HousePool.sol:835`), then prices the same epoch's deposits against that inflated principal (`HousePool.sol:915`, `juniorDepositPhase.pricingAssets = juniorPrincipal`). So a deposit that matures in that epoch is minted shares at an NAV overstated by the outstanding `V`. The trader later closes; the refund (clamped to lifetime VPI) is drawn from pool cash and Junior NAV drops. The depositor, now holding too few shares at the corrected NAV, is short by its pro-rata share of `V`; that value stays with the LPs who were already in the pool.

The design already excludes the **negative**-VPI reserve from marked LP equity "so LPs cannot withdraw it as marked equity" (`WHITEPAPER.md:5.1`). The **positive** refundable direction is the mirror image and is not excluded — and it corrupts both sides of the share price: it over-pays a redeemer (S-08) and under-mints a depositor (this report).

## Root cause

- `CfdEngineSettlementSidecar.sol:446-463` (open settlement): the positive VPI charge is routed to `HousePool.recordClaimantInflow(... Revenue ...)`.
- `HousePool.sol:624-651` (`recordClaimantInflow`): adds the amount to `accountedAssets` with no reserve on the refundable portion.
- `HousePool.sol:835` (`_settleLpEpoch` → `_reconcile`): folds the open position's booked `V` into `juniorPrincipal` before entries are priced.
- `HousePool.sol:914-919` (`_settleLpEpoch` deposit phase): `juniorDepositPhase.pricingAssets = juniorPrincipal`; the deposit batch is quoted against that just-inflated principal via `vault.quoteDepositFromState(...)` (`HousePool.sol:1046-1049`) and minted at that price (`TrancheVault.finalizeDepositEpochFromPool`, `TrancheVault.sol:1445-1476`).
- `HousePoolAccountingLib.sol:61-79` (NAV / reconcile snapshot): physical − trader claims ± price delta; no term for outstanding refundable positive VPI. The same term also drives the deposit-pricing view path `getPendingDepositTrancheState` (`HousePool.sol:1330-1338`).
- `CfdEnginePlanLib.sol:970-978` / `CloseAccountingLib.sol:104-120` (close): the rebate is clamped to exactly the lifetime VPI and paid from pool cash above trader claims — so the `V` that inflated the deposit price is removed again at close.

## Path

Victim D is an innocent new Junior depositor. Actor A (optional) is a trader; in the self-dealing framing A is also a pre-existing Junior LP and captures D's loss minus A's own trade fees. No privileged role is used.

1. D submits a Junior deposit request (`requestDeposit`). D holds no prior stake.
2. A trader opens a max-skew position (notional ≈ 0.4 × depth). The price-impact charge `V` is booked into pool NAV as LP revenue.
3. Anyone calls the permissionless `OrderRouter.settleLpEpoch(pyth)`. The reconcile folds `V` into `juniorPrincipal`, and D's deposit is priced against that inflated principal — D is minted too few shares.
4. D claims its shares. The trader closes; `V` is refunded from pool cash and Junior NAV falls.
5. D now holds shares worth less than it deposited; the shortfall stays with the LPs already in the pool.

The position has to be open only at the moment of step 3 (the reconcile that prices the batch), so the open can be sent just before the epoch settles and the close just after. The test holds the price at 1.00 throughout; the trader's price exposure in that window is ordinary directional risk with no expected sign and is not part of the measurement.

## Measurements

Release risk parameters. Pool: Senior 20M, pre-existing Junior LP 30M (depth ≈ 50M). Victim deposit 5M. Trade: LONG, notional 20M = 0.4 × depth, self-funded. Attack run vs a no-trade control, same deposit. Amounts are 6-decimal assets; divide by 1e6 for USD. Shares are 18-decimal.

| Quantity | Attack | Control (no trade) |
|---|---|---|
| `V` booked into pool NAV on open | 39,998.40 | 0 |
| Victim deposit | 5,000,000.00 | 5,000,000.00 |
| Victim shares minted | 4,993,418.677e9 | 5,000,076.104e9 |
| Victim share value after close | 4,995,956.73 | 5,000,000.00 |
| **Victim loss vs deposit** | **4,043.27** | 0.00 |
| Pre-existing LP value, start → end | 30,000,000.00 → 30,015,522.47 | 30,000,000.00 → 29,999,817.35 |
| **Pre-existing LP gain** | **+15,522.48** | −182.65 (fees/carry) |

The passive depositor is minted ~6,657.4e9 fewer share-units than in the honest control (≈ 0.133% fewer) and ends ~4,043 USDG behind on a 5M deposit purely from the dilution — it did nothing but deposit while `V` was outstanding. The staying LP ends ~15.5k ahead of where the honest control leaves it. The value moved from the depositor to the incumbents.

Scaling: the per-deposit loss is bounded by the depositor's pro-rata share of the outstanding `V`, which at release parameters is ≈ 8 bps of depth per max-skew open. A larger deposit relative to the pool, or a larger/compounded outstanding VPI (several open positions, repeated epochs), raises it proportionally. We measured one open and one deposit.

## Impact

- Any Junior LP that deposits while a trader's positive VPI is outstanding is systematically under-minted and loses its pro-rata share of a charge the pool will refund. No majority holding, no prior stake, and no trade by the victim are required — this is the honest-depositor case that S-08 (#99588) left untested for the redemption side.
- In the self-dealing framing, a pre-existing LP that opens and closes its own max-skew position around an incoming deposit epoch captures the depositor's loss (measured +15.5k to the incumbent against a −4k innocent depositor on a single 5M deposit), repeatably and permissionlessly.
- No protocol principal and no trader collateral is taken; the loss is confined to Junior LPs' yield, moved from new depositors to incumbents.

## Why this is not intended and not a known issue

- Same asymmetry as S-08: the design excludes the **negative**-VPI reserve from marked LP equity (`WHITEPAPER.md:5.1`) so LPs cannot withdraw it; the **positive** refundable charge is not excluded and here it under-prices an entrant's shares. Entry and exit are supposed to price off "the same signed terminal-NAV reconcile" (`HousePool.sol:1325-1327` doc) — they do, and that shared basis is exactly what carries the unreserved `V` into the mint.
- `VpiDepth.t.sol` and `VpiRoundTripAccounting.t.sol` assert only that the **trader** does not profit from a VPI round-trip; neither asserts deposit or redemption share-pricing around one.
- The lifetime clamp (`CloseAccountingLib.sol:112-115`, `SECURITY.md:1443`) is described as preventing an "LP sandwich" on the trader's rebate leg; it does nothing about the share-pricing leg, where the value moves here.
- Not in the Cecuro report, the SC Audit Studio pre-audit, the 15 Oct AI scan, or the listed `SECURITY.md` known-limitation sections.

## Recommended fix

Mirror the reserve the design already applies on the **negative** VPI side. Today a trader with negative `vpiAccrued` must pre-lock `max(−vpiAccrued, 0)` as a reserve held **outside** pool NAV (`_negativeVpiReserveTarget` / `vpiRebateReserveAssets`, `CfdEnginePlanLib.sol:49-53`, `:344-349`, enforced at `:830`), precisely so LPs cannot price against a rebate that will be paid out. The positive direction has no such exclusion: subtract outstanding refundable positive `vpiAccrued` (aggregated across open positions) from `distributable` in `HousePoolAccountingLib.buildReconcileSnapshot` (`:60-78`), so that the inflated principal never reaches either the redemption pricing (S-08 / #99588) or the deposit pricing (`HousePool.sol:914-919`, this report). Equivalently, book the positive VPI into a reserve bucket released to LP revenue only after the position closes and the refund window has passed.

This is the same single fix as #99588, applied at the shared term. We note it explicitly because a fix scoped only to the redemption funding path would leave the deposit pricing at `HousePool.sol:914-919` — a *different* call site reading the *same* unreserved `juniorPrincipal` — still exposed. The fix must be at the NAV term, not at either pricing call site.




## Link to Proof of Concept

https://gist.github.com/dexloom/e593c86ce9dc8fac61fbe4a6bb656724

## Proof of Concept

## Proof of concept

Secret gist with README and test: https://gist.github.com/dexloom/e593c86ce9dc8fac61fbe4a6bb656724

Save as `packages/perps/test/perps/accounting/vault/R2DVpiDepositDilution.t.sol` in a fresh checkout at the pin, then from `packages/perps`:

```
forge test --match-contract R2DVpiDepositDilutionTest -vv
```

Output at the pin (6-decimal assets; divide by 1e6 for USD; shares 18-decimal):

```
[PASS] test_PassiveDepositor_DilutedByOutstandingVpi_vs_control()
  === run: ATTACK (deposit during open +VPI)
    V (pool delta on open):           39998400063
    victim deposited:                 5000000000000
    victim shares minted:             4993418677311568
    victim share value after close:   4995956726713
    beneficiary start value:          29999999990868
    beneficiary end value:            30015522469049
  === run: CONTROL (deposit, no trade)
    V (pool delta on open):           0
    victim shares minted:             5000076104309862
    victim share value after close:   4999999999999
    beneficiary end value:            29999817339422
  victim loss (attack):               4043273287
  victim loss (control):              1
  beneficiary gain (attack):          15522478181
  beneficiary gain (control):         -182651446
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

The test file is beside this note (`R2DVpiDepositDilution.t.sol`).
