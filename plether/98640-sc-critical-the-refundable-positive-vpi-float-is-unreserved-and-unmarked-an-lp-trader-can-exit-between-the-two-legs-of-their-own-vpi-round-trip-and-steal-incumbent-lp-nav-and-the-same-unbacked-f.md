# \#98640 [SC-Critical] The refundable positive-VPI float is unreserved and unmarked: an LP-trader can exit between the two legs of their own VPI round trip and steal incumbent LP NAV (and the same unbacked float drains the solvency base to latch degraded mode)

**Submitted on Nov 9th 2026 at 07:57:50 UTC by @Matgothmog for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98640
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

# Description

## Brief

A trader's positive lifetime-VPI balance (the charge they paid at open) is **refundable**: the
lifetime clamp lets them recover up to that accrual through future skew-reducing closes. The
protocol treats this refundable float as free equity everywhere — it is collected as pool revenue,
counted in LP NAV, withdrawable by the LP firewall, and counted as solvency in the `E ≥ L + B`
admission gate — while its mirror (negative lifetime-VPI) is backed by a dedicated 1:1
non-withdrawable reserve. That asymmetry enables two concrete exploits:

1. **LP straddle (High — direct theft of user funds) :** a trader-LP exits the tranche *between the
   two realized legs* of their own VPI round trip. The charge (paid at open) inflates NAV; their
   redemption funds at the inflated symmetric NAV; then the clamped rebate (paid at close) drains
   the pool after they have left — leaving the unmarked rebate liability to the incumbent LPs.
   Twin-run measured: attacker +5,664,481,761 atoms extracted, incumbents −5,657,368,617; the
   straddle flips an otherwise losing round trip (−3,923,089,758) into +1,734,986,024. Repeatable
   per skew window.
2. **Solvency-base breach (Medium — griefing):** an LP redeems to the firewall floor
   (`E = L + B` exactly — certified protected), then the refund drains the **L base itself**
   (the rebate path caps only at post-claim cash, unlike the fee top-up right beside it which
   respects the liability reserves). Measured: `E = 1,198,381.67 < L = 1,199,880.00` — degraded
   mode latches at zero attacker cost, blocking new risk and LP entry until an owner acts.

One fix — give the refundable float the same dedicated reserve its negative mirror already has —
closes both.

## Vulnerability Details

All references are against the in-scope code at commit `a82d4fef`.

**1. The float is refundable but treated as free equity.** The close-time VPI accounting
(`CloseAccountingLib.sol:103-118`) clamps the close delta at `−proportionalAccrual`, so a trader who
paid a positive `vpiAccrued` at open can recover up to that accrual on later closes. The collected
charge, however, is booked immediately as `recordClaimantInflow(Revenue)` — LP equity
(`CfdEngineSettlementSidecar.executeOpen:455-471`). The spec's NAV model **explicitly excludes**
VPI from the marked delta ("Carry, VPI, execution/frozen spreads, protocol fees, liquidation
reserve, order margin, and other action reserves are deliberately excluded: they are separate
realized-settlement quantities, not marked price PnL", `ACCOUNTING_SPEC.md:142`) — so the
refundable portion is neither marked as a liability nor reserved. The **negative** side is backed
1:1 by a dedicated non-withdrawable action reserve ("Every live negative lifetime-VPI balance has
equal dedicated action-reserve backing", SECURITY.md invariants table). The positive side has no
analog.

**2. The straddle (PoC `WaveRLpValueStraddle.t.sol`, 3/3 green).** Sequence through public
entrypoints (V3 `commitOrder`/`executeOrder`, `TrancheVault.requestRedeem`/`claimRedeem`, LP epoch
settlement, deploy-config parameters):
   1. Attacker is a Junior LP + trader. They open a position paying VPI charge `c1` — realized cash
      to pool revenue; symmetric NAV rises.
   2. They immediately request an LP redemption (async; funds at the next epoch settlement at the
      then-current symmetric NAV — which contains their realized charge but NOT the offsetting
      refundable liability, since VPI is excluded from the marked delta).
   3. After the redemption funds, they close the position taking the clamped rebate (up to `c1`),
      paid from pool cash.
   Net: the rebate is paid by the pool whose NAV they exited at the pre-rebate price — the
   incumbents absorb it. Twin-run (identical round trip with/without the straddle):
   attacker funded redemption 12,001,684,390,299 → 12,007,348,872,060 (**+5,664,481,761** =
   closed form φ·(c1 − fee), match 0.1%); incumbent funded redemption 8,001,122,923,823 →
   7,995,465,555,206 (**−5,657,368,617**). The straddle converts an all-in loss of
   −3,923,089,758 into a gain of +1,734,986,024. The trader-side clamp promise ("the lifetime
   clamp intentionally zeroes otherwise extractable rebate-only round trips", `SECURITY.md:1443`)
   holds exactly — and is defeated purely at the LP layer by ordering.

**3. The solvency breach (PoC `WaveSVpiFloatReserveBreach.t.sol`, 2/2 green).** The LP withdrawal
firewall reserves `C + L + B` (`HousePool.getSettlementFreeAssets:1242-1256`), so LPs can redeem
down to `E = L + B` exactly — a state the docs certify as solvent and protected. The close-rebate
path, however, pays out of cash capped only at post-claim cash `max(P − C, 0)`
(`CfdEnginePlanLib:970-978`) — it does **not** reserve `L + B`, unlike the MED-6-fixed fee top-up
immediately below it (`_freePoolCashAfterLiabilityReserves:1022-1033`). Measured chain (deploy
params, zero price move): float 6,498,300,642 → redemption to the floor (`E = L + B` =  
1,202,879,700,000) → wall close refund 4,498,025,635 → `E = 1,198,381,674,365 < L =
1,199,880,000,000` (**breach 1,498,325,635**) → `degradedMode` latches. The control run (no drain)
stays clean. Degraded mode blocks all further risk expansion and LP deposit activation until the
owner clears it — a repeatable, zero-cost, permissionless griefing lever.

## Recommended Mitigation

**Primary (closes both):** mirror the negative-VPI machinery for the positive side — back every
live positive lifetime-VPI accrual with a dedicated non-withdrawable reserve (excluded from LP NAV
and from `getSettlementFreeAssets`), releasing it only as refunds are paid. Then (a) the straddle's
redeemed NAV no longer contains the refundable float, and (b) the refund cannot drain the `L`
base. Alternative partials: mark the refundable float as a NAV liability at exit pricing (fixes the
straddle only), or cap close-rebate payouts by the same `_freePoolCashAfterLiabilityReserves`
reserve the fee top-up uses (fixes the breach only). Also fix the receipt split (`FrozenCloseSpreadSettled`)
to report rebate-offset collections as recovered rather than waived.

## Link to Proof of Concept

https://gist.github.com/Matgothmog/7466396653b00dd38ccac35900b07fa1

## Proof of Concept

# Proof of Concept

Two Foundry twin-run test files against the frozen commit `a82d4fef` (Forge 1.5.1, solc 0.8.35
via-IR), both in `poc/` (drop into `packages/perps/test/perps/regression/` and run
`forge test --root packages/perps --match-contract "WaveRLpValueStraddle|WaveSVpiFloat" -vv`).
Full run output: `poc/RUN_OUTPUT.txt` — **5/5 PASS** (3 straddle + 2 breach).

Both use the Wave-M twin-run standard: `vm.snapshot`/`vm.revertTo`, identical victim scenarios with
and without the attacker's move, all state through PUBLIC entrypoints (V3 `commitOrder`/`executeOrder`
with real Pyth payloads, `TrancheVault.requestRedeem`/`claimRedeem`, `HousePool` LP epoch settlement),
deploy-config parameters (k = 0.01e18, maxSkew 40%, fee 4 bps), no storage writes.

## A. LP straddle — `WaveRLpValueStraddle.t.sol` (3/3 PASS)

| quantity (6-dec USDG atoms) | baseline world | straddle world | delta |
|---|---:|---:|---:|
| attacker funded redemption | 12,001,684,390,299 | 12,007,348,872,060 | **+5,664,481,761** |
| closed form `φ·(c1 − fee)` | — | — | 5,669,265,073 (match 0.1%) |
| incumbent funded redemption | 8,001,122,923,823 | 7,995,465,555,206 | **−5,657,368,617** |
| attacker all-in (incl. trader leg cost 5,613,906,126) | −3,923,089,758 | +1,734,986,024 | straddle flips a loss into a profit |

- Test 2 proves the trader-side clamp promise holds exactly (the round trip nets zero at the
  trader layer) — the extraction is purely the LP-layer ordering.
- Test 3 (repeatability): a second extraction measured at $7M leg scale: 5,664,481,761.

## B. Solvency-base breach — `WaveSVpiFloatReserveBreach.t.sol` (2/2 PASS)

`test_FloatIsUnreserved_FirewallFundsToFloor_RebateBreachesReserveBase`:

```
float (wall lifetime vpiAccrued):   6,498,300,642
E after redemption (floor):     1,202,879,700,000   (= L + B exactly — firewall-certified)
L:                              1,199,880,000,000
B:                                  2,999,700,000
E after wall close:             1,198,381,674,365   (< L — breach)
breach (L − E):                     1,498,325,635   → degradedMode latches
```

`test_Control_NoDrain_CloseStaysAboveLiability` — identical scenario without the firewall drain
stays above `L`, isolating the unreserved refund as the cause.

Deterministic, no forks, no privileged actors; the straddle needs only ordinary LP+trader roles and
the async exit window; the breach needs a single trader close after an ordinary redemption.
