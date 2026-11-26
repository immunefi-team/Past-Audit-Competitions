# \#98045 [SC-Critical] Repeated Junior entry around a VPI rebate round trip makes incumbent Junior LPs redeem below deposit

**Submitted on Nov 4th 2026 at 15:37:15 UTC by @itko69 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98045
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A skew-reducing open is paid its VPI rebate `r` out of HousePool cash, and the account must lock the same amount as a non-withdrawable VPI rebate reserve. The lifetime VPI clamp makes the full close return `r` to the pool, but terminal LP NAV leaves the reserve out, so Junior NAV is `r` lower until that close. The attacker requests a Junior deposit before the request cutoff, opens the skew-reducing leg 120 s before the epoch boundary `b`, settles the epoch through the permissionless Router at `b + 1 s` and commits the close at `b + 2 s`. The deposit activates on the lowered NAV, the attacker holds a share `α` of the Junior tranche when `r` returns, and it redeems after the 1-hour cooldown. The incumbent Junior LPs fund the entrant's `α·r`. The trading account's VPI nets to zero, and it pays its two execution fees.

In the PoC (launch parameters, basket held at 1.20), an incumbent Junior LP deposits 10,000,000 USDG and the attacker runs eight cycles in one trading day with a 40M USDG Junior deposit; at the end every position is closed and every incumbent redeems in full. The incumbent's `claimRedeem` pays 9,955,325.067168 USDG in a Junior-only pool and 9,863,536.928756 USDG with 20M Senior, against 10,001,717.044833 and 10,000,402.386472 USDG without the attacker, and each of the attacker's two legs on its own leaves it at or above its deposit. The attacker nets +21,965.393356 and +63,671.134698 USDG. At 10 bps, the widest basket confidence the oracle accepts, the incumbent still ends 33,655.807162 and 103,384.037261 USDG below its deposit. After one cycle it is still above its deposit, because its NAV still includes the VPI charge paid by T, the independent trader whose long creates the skew; from the second cycle on it is below it.

**Severity.** Critical, "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield": over one trading day the incumbent's `claimRedeem` falls below its deposit only when both attacker legs run (U1, first `forge test` command under Proof of Concept; see Classification). If the amount taken is classified as yield, the applicable impact is "Theft of unclaimed yield" (High): `test_R7_1` and `test_R7_4` to `test_R7_12` prove that impact on their own, against the controls `test_R7_2` and `test_R7_3` (R7, second `forge test` command).

## Vulnerability Details

### Root cause

Three properties combine. Each one is intended on its own (see "Relation to the documented design").

**(1) The rebate leaves pool cash and is locked as a reserve.**
- When an open's trade cost (VPI plus fee) is negative, the planner sets `poolRebatePayoutAssets = -tradeCost` (`src/libraries/CfdEnginePlanLib.sol:333`), and the sidecar pays it out of the pool (`src/CfdEngineSettlementSidecar.sol:442-444`). Because `-tradeCost` is the rebate net of the fee, the open's execution fee is then paid from pool cash to the protocol treasury (`CfdEngineSettlementSidecar.sol:498, 905-926`), so pool assets fall by the full rebate `r` (asserted in test 1).
- The account's reserve target is its gross negative lifetime VPI (`CfdEnginePlanLib.sol:348-349`). It is locked at `CfdEngineSettlementSidecar.sol:466-478`.
- The skew cap does not stop the open, because the open takes skew to zero (`CfdEnginePlanLib.sol:473-480`).

**(2) The clamp guarantees the reserve returns to the pool, at a time the position owner chooses.**

```solidity
// src/libraries/CloseAccountingLib.sol:112-118
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
// where an attacker opens at high depth, reduces pool depth, then closes to extract a
// net-negative lifetime VPI rebate. ...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

- A full close, or a liquidation (`ACCOUNTING_SPEC.md:1181-1184`), therefore charges at least `r`, whatever the depth at that moment.
- Until then, the reserve can only be consumed by that clawback, or released against equivalent withholding or a lower surviving target (`SECURITY.md:376`).
- The close moves the reserve into the pool (`CfdEngineSettlementSidecar.sol:690-693`). `MarginClearinghouse.sol:883` describes this as "Returns reserved negative-VPI backing to the LP pool as cash recovery."
- The close records the amount as claimant `Revenue` / `CashArrived` (`CfdEngineSettlementSidecar.sol:604-613`).

**(3) LP NAV leaves the reserve out, and deposits are priced on that NAV.**
- Distributable LP equity is physical assets, plus the signed terminal price delta, minus trader claims (`src/libraries/HousePoolAccountingLib.sol:60-78`).
- The terminal snapshot has no VPI or reserve term (`src/TerminalNavBookV2.sol:208-237`).
- `HousePool._settleLpEpoch` reconciles first (`src/HousePool.sol:835`). The reconcile compares distributable equity with the tranches' claimed equity and books the net difference: a shortfall against Junior principal first, a surplus as revenue (`src/libraries/HousePoolReconcilePlanLib.sol:84-104`; `src/libraries/HousePoolWaterfallAccountingLib.sol:98-110, 140-159, 167-184`).
- It then prices Junior deposits on the post-reconcile `juniorPrincipal` (`HousePool.sol:914-920`; `src/TrancheVault.sol:1398-1410`).

### Attack cycle

Test 1 setup: Junior 1M, Senior 2M, entrant X = 4M. An independent trader T holds a long of 1,140,000 USDG notional, which is 0.38 of pool depth. All amounts are in USDG.

| Time | Step | Measured |
|---|---|---|
| before `b − 300 s` | The attacker's LP wallet requests a Junior deposit of X for epoch `b` (`TrancheVault.sol:545-556`). Junior requests are uncapped (`:1313-1326`) and can be cancelled until `b` (`:851-860`). | — |
| `b − 120 s` | Account B commits a SHORT of T's size. The attacker executes it as keeper in the next block. | Pool assets fall by r = 2,164.435835, which is 0.5·k·N²/D at D = 3,002,167.998556. B's reserve = r and B's lifetime VPI = −r. Pending-reconcile Junior NAV goes from 1,002,131.082085 to 999,966.641176. |
| `b + 1 s` | Anyone calls `OrderRouter.settleLpEpoch` (`src/router/OrderLiquidationBatchLogic.sol:549-581` → `HousePool.sol:797-821`). | X activates 21.60 bps below the Junior share price just before B's open. That price includes T's charge. α = 0.800005. |
| `b + 2 s` | B commits its full close. The attacker executes it in the next block. | The unclamped close VPI at the post-activation depth D − r + X would be 928.285241 (logged as "depth D+X", with D measured after B's open). The assessed charge is 2,620.435835 = r + the 456 fee. Reserve consumed: r. |
| activation + 1 h | Once the cooldown has passed, the attacker requests redemption (`TrancheVault.sol:617-619`; the cooldown starts at activation, `:1108-1110`). The request targets `b + 2 h`, and the attacker settles and claims. | Redeemed: 4,001,702.363794 (LP P&L +1,702.363794). Treasury: +912 (B's two fees). |

The next cycle's boundary is `b + 3 h`; its deposit request is placed right after the `b + 2 h` exit settles, once the attacker's shares have left Junior.

### Exit direction (same rule, opposite sign)

An LP that also trades can reverse the legs around its own redemption:
1. It opens a skew-increasing position 60 s before its redemption is priced. Its VPI charge `c` is now pool cash (recorded as revenue at `CfdEngineSettlementSidecar.sol:454-463`), so it is inside the NAV that prices the redemption.
2. It closes right after the settlement. The clamp allows this skew-reducing close a rebate of up to `c`, paid from pool cash that now belongs to the remaining LPs.

RT6 tests A3, A4 and A6 measure this. A7 runs the entry direction at Senior/Junior = 3 (see Impact Details).

### Relation to the documented design

The following is documented, and this report does not dispute it:
- **Anticipated VPI and the VPI reserve are outside terminal `K` and LP NAV.** "Carry, VPI, execution/frozen spreads, protocol fees, liquidation reserve, order margin, and other action reserves are deliberately excluded" (`ACCOUNTING_SPEC.md:141-142`). "The VPI reserve therefore does not inflate marked LP receivables or symmetric LP NAV" (`ACCOUNTING_SPEC.md:1064-1066`). See also `SECURITY.md:863`, `SECURITY.md:871-872`, `WHITEPAPER.md:510-512` and `WHITEPAPER.md:887-888`.
- **Entry and exit are priced from one terminal-NAV snapshot.** "an entrant ... receives credit only for collectible marked receivables" (`ACCOUNTING_SPEC.md:172-173`; invariant 7 at `ACCOUNTING_SPEC.md:1288`; `WHITEPAPER.md:544-547`).
- **Trading can change NAV between the request cutoff and settlement** (`ACCOUNTING_SPEC.md:228-229`; `SECURITY.md:1044-1048`; `WHITEPAPER.md:742-744`).
- **For carry, collection-time ownership is intentional.** "This intentionally allows a trader and incoming LP to time collection after LP activation and share carry accrued before that activation" (`ACCOUNTING_SPEC.md:737-740`; `WHITEPAPER.md:974-977`).

The PoC does not show any of these rules being broken. It shows what they produce when the party that times the deposit also creates the VPI rebate reserve that returns to the pool after activation. That result differs from the documented carry case in four ways (the first three measured, the fourth from the documentation):
1. **No revenue is created beyond B's own payments.** Test 3 runs B's round trip without the entrant:
   - incumbent Junior value changes by +0.035145 USDG, and the treasury receives B's two fees, 912 USDG;
   - after the settlement at `b`, stored `juniorPrincipal` (the Junior tranche's stored NAV, which also takes in revenue) is 2,164.435835 lower than without B (999,966.032348 against 1,002,130.468183). The reconcile at `b` nets the dip against T's VPI charge (2,165.998556), which had not been distributed yet and covers the whole dip, so in this cycle the dip reduces undistributed revenue. The remaining 34.966977 between stored `juniorPrincipal` and the 1,000,000.999325 recorded after bootstrap is the Senior coupon accrued since bootstrap (36.529698, the same gap the world without B shows against bootstrap plus T's charge) minus the 1.562721 by which T's charge exceeds `r`. B's close restores the 2,164.435835.
   - What reaches the pool after activation is the pool's own rebate, coming back from B's reserve.
   - Over one trading day (U1_1, NEG world; see Impact Details), eight such round trips without the entrant leave the incumbent's `claimRedeem` 42.607766 USDG above the run without them.
2. **The loss is not limited to revenue.**
   - U1: over one trading day, the incumbent's loss against the run without the attacker is 3.37 to 5.19 times an upper bound on all trading revenue the pool receives in that run, and the incumbent redeems less than it deposited (see Impact Details).
   - Test 4: three natural-skew cycles cost incumbents 5,199.366316 USDG against the control. That is more than twice T's 2,165.998556 USDG charge.
   - Test 5 has no third-party trader. Account A opens the skew, paying 2,165.998556 USDG of VPI into the pool, and closes it after the third cycle, when the pool pays out 1,696.684425 USDG. Each B round trip returns exactly the rebate it received. With every position closed, incumbents end 4,642.421468 USDG (46.43 bps) below a run in which the attacker never acts.
   - By contrast, the carry text accepts only that timing can "transfer part of old carry revenue to a new LP" (`ACCOUNTING_SPEC.md:739-740`).
3. **Value collected before activation is left out of the activation price.** For carry, "Carry collected before deposit activation is already included in the activation price" (`ACCOUNTING_SPEC.md:734-735`; `WHITEPAPER.md:969-970`).
   - In the control of test 1, the entrant pays for T's charge: it activates at the pre-open price, a 0.00 bps discount.
   - In the attack, B's open, committed 120 s before the boundary and executed in the next block, removes r from pool assets, and the entrant activates 21.60 bps lower. The clamp returns exactly r seconds after activation.
   - The carry rule's other half, sharing carry collected after activation, concerns pending carry, which the spec defines as trader debt, not an LP NAV receivable (`ACCOUNTING_SPEC.md:731-733`); it has not yet been paid to the pool. `r` was pool cash until B's open executed, 119 s before the boundary.
4. **Unlike the carry case, this result was not found in the documentation.**
   - For carry, the corresponding result is stated as intentional (`ACCOUNTING_SPEC.md:737-740`). For the VPI reserve, the documentation states the exclusion ("does not inflate marked LP receivables or symmetric LP NAV", `ACCOUNTING_SPEC.md:1065-1066`; also `SECURITY.md:871-872`) and the withdrawal rule ("LPs cannot withdraw it as marked equity before the matching position obligation settles", `WHITEPAPER.md:887-888`; also `ACCOUNTING_SPEC.md:821-822`).
   - No passage was found in the perps documentation or the NatSpec on an entrant that activates while a VPI rebate reserve is outstanding and then shares it when the clawback returns it to the pool. Neither "LP accounting limitations" (`SECURITY.md:1380-1399`) nor "VPI limitations" (`SECURITY.md:1434-1444`) lists it.

The clamp comment quoted above names "LP sandwich attacks" around a VPI round trip as the threat the clamp addresses (`CloseAccountingLib.sol:112-114`). "VPI limitations" states that "the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips" (`SECURITY.md:1443`). Both hold for B's account in the PoC; the value moves through the deposit's share price instead.

The anti-flash cooldown starts at activation (`WHITEPAPER.md:715-716`), and the attacker respects it. Because the reserve returns seconds after activation, the cooldown only sets how long the attacker holds the shares. Of the request-to-activation delay, the whitepaper says: "The delay separates request funding from activation; it is not a substitute for exact valuation." (`WHITEPAPER.md:734-735`).

### Known issues

- **AI scan "Temporary pool liquidity can reduce a skew-increasing close charge"** (info, `audits/scan-plether-fi-plether-core-d4541b49.md:1693`). The entrant's deposit does raise depth before B's skew-increasing close, and the clamp then holds B's charge at `r` + fee in every cycle (test 1: 2,620.435835 USDG assessed, against an unclamped VPI of 928.285241 USDG at depth D − r + X). Without the entrant, B's close is priced at the depth reduced by `r` and repays slightly more than `r`, about r²/D per cycle (U1_1 NEG: the incumbent ends 42.607766 USDG above BASELINE over eight round trips, B's carry included). The deposit therefore lowers B's close charge only by that depth term, while the entrant takes α·r of the returned reserve.
- **AI scan "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate"** (info, `:1742`). This concerns the exit direction, where the attacker's own redemption lowers depth before its skew-reducing close.
  - In A3, the pool pays back 50,911.852575 USDG at the close, against a 63,680.398407 USDG charge at the open.
  - A6 runs the same close before the boundary, at unchanged depth, and the pool pays back 50,861.740002 USDG.
  - The attacker's gain comes from the redemption price. Its LP leg receives 50,944.596856 USDG more than in the plain round trip, while its trading account ends 25,536.545832 USDG below its funding.
- **Cecuro HIGH-1, MED-7 and MED-18** (clock alignment, component selection, mark rewrite). The basket is 1.20 on every update, so no price choice is involved.
- **SC Audit Studio SC-H1, "Deposit Before Yield is Accrued Allows to Dilute Yield".** This concerns the Senior coupon and was fixed by the Senior rewrite (plether-core#23). In test 2 the entrant pays its share of the coupon (−29.211846 USDG), so that fix holds here. The value source in this report is the VPI rebate reserve. SC-H1's stated impact is "Yield can be diluted leading to loss of funds for previous depositors"; in U1 the incumbent's redemption falls below its deposit.
- **SECURITY.md "Known Limitations"** (`SECURITY.md:1328-1457`). None of its seven subsections, including "LP accounting limitations" and "VPI limitations", describes an entrant capturing a reserve that returns to the pool after it activates.

## Impact Details

### One trading day: the incumbent redeems less than it deposited (U1; USDG; release build)

`packages/perps/test/poc/u1/U1_JitJuniorPrincipalLoss.t.sol` reuses the R7 harness unchanged: launch parameters, 1 USDG seeds, production entrypoints and a basket of 1.20 on every update. It runs one trading day of eight cycles, one every 3 h, in four worlds on one absolute timestamp schedule:
- **ATTACK:** B's round trip and the entrant's Junior deposit and redemption in every cycle;
- **CONTROL:** the same deposits and redemptions, without B;
- **NEG:** B's round trips, without any deposit;
- **BASELINE:** no attacker activity.

Trader T holds a long of 0.38 of pool depth all day and closes it after the last cycle, so no position is open in any world. Every incumbent LP then redeems in full through `requestRedeem`, `OrderRouter.settleLpEpoch` and `claimRedeem`. The test asserts that no pending or claimable redeem request remains. The first table gives the USDG the incumbent Junior LP receives from `claimRedeem` minus the 10,000,000 USDG it deposited. Attacker net is defined as for the R7 table below.

| Test | Pool | Confidence | ATTACK | BASELINE | CONTROL | NEG | Attacker net |
|---|---|---|---|---|---|---|---|
| U1_1 | J 10M, X 40M | 0 | −44,674.932832 | +1,717.044833 | +1,615.080433 | +1,759.652599 | +21,965.393356 |
| U1_2 | J 10M, X 40M | 10 bps | −33,655.807162 | +3,611.937429 | +3,510.072523 | +18,821.489942 | +12,846.537849 |
| U1_3 | J 10M, S 20M, X 40M | 0 | −136,463.071244 | +402.386472 | +2,522.242295 | +530.220406 | +63,671.134698 |
| U1_4 | J 10M, S 20M, X 40M | 10 bps | −103,384.037261 | +6,087.182932 | +8,205.536810 | +51,716.822251 | +36,292.926453 |

| Test | Incumbent loss vs BASELINE | Revenue bound, BASELINE day | Loss / bound | Senior `claimRedeem`, ATTACK − BASELINE |
|---|---|---|---|---|
| U1_1 | 46,391.977665 | 8,936.739801 | 5.19 | n/a (seed only) |
| U1_2 | 37,267.744591 | 10,828.023190 | 3.44 | n/a (seed only) |
| U1_3 | 136,865.457716 | 26,811.243763 | 5.10 | +0.527123 |
| U1_4 | 109,471.220193 | 32,485.212145 | 3.37 | +0.527123 |

What the runs show:
- **Below the deposit only when both legs run.** With no open position and no price move, the ATTACK incumbent redeems less than it deposited (44.67 bps less in U1_1, 136.46 bps in U1_3), while BASELINE, CONTROL and NEG each return at least the deposit. Both are asserted in every U1 test.
- **More than all revenue of the same day without the attacker.** In BASELINE the only trading inflows to the pool are T's VPI charge `cT` at its open and what T pays at its close. T's close also pays T a VPI rebate, which the lifetime clamp caps at `cT`, so what T pays at its close is at most T's close flow plus `cT`, and the day's trading revenue is at most 2·`cT` + T's close flow (U1_1: 2 × 7,219.694559 − 5,502.649317 = 8,936.739801). The incumbent's loss against BASELINE is 3.37 to 5.19 times that bound (the test asserts more than 3 times) and larger than its whole realized BASELINE yield (also asserted).
- **The returned rebates are recorded as revenue, but they are not new value.** In ATTACK the pool records the eight returned reserves as `Revenue`. Each returns exactly the rebate the pool paid B at its open (asserted every cycle), so together they equal the logged sum of B's rebates, Σr = 57,832.891994 USDG in U1_1, which is more than the entrant takes (Σα·r = 46,285.066506). NEG runs the same round trips without the entrant, and the incumbent ends 42.607766 USDG above BASELINE: B's close is priced at the depth reduced by `r`, so B repays slightly more than `r`; B's carry adds to that. At 10 bps NEG ends higher still (U1_2: +15,209.552513 against BASELINE), because B's adverse fills also pay the pool.
- **Above the deposit after cycle 1, below it from cycle 2 on, lower after every cycle.** After cycle 1 the ATTACK incumbent is still above its deposit in every test (U1_1: 10,001,448.129496). From cycle 2 on, about 5 h after the first boundary, it is below it (U1_1: 9,995,672.567372 after cycle 2, 9,960,934.816881 after cycle 8), and every later cycle lowers it further. All three facts are asserted. In U1_1's BASELINE the same valuation stays at 10,007,219.69 USDG, the deposit plus T's VPI charge, until T closes; T's close then refunds most of that charge (pool flow at T's close: −5,502.649317). The margin above the deposit after cycle 1 (U1_1: 1,448.129496) is smaller than that refund. In U1_1 the cycle-1 gap between BASELINE and ATTACK (5,771.564305) is already larger than BASELINE's realized yield for the whole day (1,717.044833); this comparison is arithmetic on the logged values, and the test asserts the cumulative form.
- **Where the value goes.** The incumbent's loss against CONTROL equals Σα·r minus the adverse-fill drag to within 2 % (asserted; U1_1: 46,290.013265 against 46,285.066506; U1_2: 37,165.879685 against 46,284.272489 − 9,118.061613). The attacker keeps that take net of B's 16 execution fees, which are the whole treasury difference (asserted; U1_1: 24,319.488).
- **CONTROL against BASELINE.** −101.964400 in U1_1 and −101.864906 in U1_2: T's carry is utilization-priced (`WHITEPAPER.md:922-939`), so the entrant's capital lowers T's carry while it is in the pool, and the difference equals the difference in T's close flow to within 0.0002 USDG. +2,119.855823 in U1_3 and +2,118.353878 in U1_4, where the entrant pays part of the Senior coupon; this is why the loss against CONTROL is larger than the loss against BASELINE in those two tests.
- **Senior** redeems more than its deposit in ATTACK and in BASELINE, and the attack changes its `claimRedeem` by +0.527123 USDG (U1_3, U1_4).

The per-cycle values are vault valuations (`convertToAssets`) at a fresh mark, read at the same second in every world; the pending-reconcile view applies the mark-dependent part of the reconcile only while the mark is fresh (`HousePool.sol:1648`). Where no settlement ran at that second, the test takes a state snapshot, applies one permissionless mark refresh, reads and reverts, so the read does not become part of that world's history. The realized figures are the `claimRedeem` amounts.

### Single-cycle and three-cycle results (R7 and RT6; USDG; release build)

Definitions:
- **Attacker net:** the change in value of all attacker addresses over the run (wallets, clearinghouse balances, and Junior shares at NAV). It is after B's fees, carry, and the entrant's share of the Senior coupon.
- **Control:** the same schedule with the entrant but without B's round trip.
- **bps (this table):** the loss divided by the incumbent Junior LP's value at the end of the comparison run (the control; for test 5, the run without the attacker), truncated to two decimals as the PoC prints it.

| Test | Setup | Attacker net | Incumbent Junior LPs |
|---|---|---|---|
| 1 | J 1M / S 2M / X 4M; natural skew 0.38·D; 1 cycle; confidence 0 | +790.328977 | −1,731.539093 vs control (17.27 bps) |
| 7 / 8 / 9 | test 1 at confidence 1 / 5 / 10 bps | +756.196181 / +619.659826 / +448.977744 | −1,697.406164 / −1,560.869283 / −1,390.186543 vs control |
| 6 | J 10M / S 20M / X 40M; otherwise as test 1 | +7,903.303339 | −17,315.419987 vs control (17.27 bps) |
| 10 / 11 / 12 | test 6 at confidence 1 / 5 / 10 bps | +7,561.975417 / +6,196.612040 / +4,489.791427 | −16,974.090440 / −15,608.720568 / −13,901.891837 vs control |
| 4 | test 1 repeated for three consecutive cycles | +2,375.733126 | −5,199.366316 vs control (51.88 bps) |
| 5 | J 1M / S 2M / X 4M; skew opened and finally closed by attacker account A; three cycles | +1,450.418995 | −4,642.421468 vs no attacker activity (46.43 bps) |
| 2 | control: entrant without B's round trip | −29.211846 | +29.211040 vs no attacker activity |
| 3 | negative control: B's round trip without the entrant | B pays its two fees (912) | +0.035145 vs no B round trip |

Where the value goes:
- **Test 1:** the incumbents' loss against the control (1,731.539093) equals the attacker's gain against the control (819.540823) plus B's two fees to the treasury (912), to within 0.001730. Senior differs by 0.000000.
- **Test 6:** the same split holds: 17,315.419987 against 8,195.421717 + 9,120.
- **Test 5:** the incumbents' loss equals the attacker's net plus the treasury's 3,192, to within 0.002473.
- **With confidence (tests 7–12):** every fill is shifted against the trader by 25 % of the basket confidence; the PoC asserts this from the fill events. This lowers both the attacker's gain and the incumbents' loss. The sign does not change up to the 10 bps limit, which is the widest confidence the oracle accepts (`PletherOracle.sol:869-873`).

Exit direction and entry at S/J = 3 (RT6 tests, release build). Setup: Senior 30M, honest Junior 10M, attacker Junior 40M, basket 1.20, confidence 0.

| Test | What the attacker does | Attacker net (plain LP round trip) | Honest Junior LP value, plain → attack |
|---|---|---|---|
| A3 | SHORT of 31,920,000 notional (just under 0.4 of depth), opened 60 s before its exit settlement and closed after it | +24,969.688449 (−438.362575) | 9,999,616.439766 → 9,961,440.316452 (−38,176.123314) |
| A4 | A3 three times in a row | +75,098.378071 (−1,315.133846) | 9,998,849.297878 → 9,884,131.721317 (−114,717.576561) |
| A6 | timing control: the same trade, opened and closed before the settlement | −15,770.038270 (−438.362575, the same plain round trip as A3) | 9,999,616.439766 → 10,002,180.115205 (+2,563.675439) |
| A7 | entry: a SHORT against an honest LONG of 15,960,000 notional, opened before its activation and closed after it | +12,245.611184 (−438.083600) | 10,031,456.279911 → 10,006,004.587670 (−25,451.692241) |

### Classification

Selected impact: **Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield (Critical).** The attack takes part of the USDG the incumbent Junior LP deposited, not only yield owed to it:
1. **Counterfactual.** With every position closed, no price move and a full redemption at the end, the incumbent's `claimRedeem` falls below its 10,000,000 USDG deposit only when both attacker legs run (U1): 9,955,325.067168 USDG in a Junior-only pool and 9,863,536.928756 USDG with 20M Senior. BASELINE, CONTROL and NEG all return at least the deposit. Without the attacker, the incumbent's realized yield for the day is +1,717.044833 USDG (U1_1); the attack costs it 46,391.977665 USDG against that day.
2. **More than all revenue of the same day without the attacker.** The loss against BASELINE is 3.37 to 5.19 times an upper bound on all trading revenue the pool receives in the same day without the attacker (BASELINE). Tests 4 and 5 show the same in R7 (see "Relation to the documented design", item 2).
3. **The `Revenue` record is the pool's own cash coming back.** The returned reserve is recorded as claimant `Revenue` (`CfdEngineSettlementSidecar.sol:604-613`), and in ATTACK the returned reserves alone exceed the entrant's take (U1_1: Σr = 57,832.891994 against 46,285.066506). Each record closes a round trip that nets to zero for the pool: the same `r` left pool cash 122 s earlier (`HousePool.payOut`, `HousePool.sol:604-613`), and each close returns exactly `r` (asserted). Without the entrant, the same eight round trips leave the incumbent 42.607766 USDG above BASELINE (NEG). The entrant's share of each return is therefore value the incumbents held before B's open.
4. **Repetition.** Each cycle takes α·r of the incumbents' NAV (less the adverse-fill drag at nonzero confidence), whatever that NAV consists of; the test asserts `r` ≤ 0.08 % of pool depth at every B open. After cycle 1 the incumbent is still above its deposit in all four U1 tests, because T's VPI charge is still in its NAV. From cycle 2 on it is below its deposit, and each of cycles 3 to 8 starts below the deposit and lowers its value again, so each of those cycles takes deposited USDG (both asserted; values under "What the runs show"). Every step is permissionless, and on a trading day a new cycle can start every 3 hours (entry activation is deferred while the oracle is frozen, `SECURITY.md:1387-1388`).

For comparison, Immunefi reports #37315 and #37311 (Jito Restaking) are published as High, "Theft of unclaimed yield": a depositor that joins just before a reward is distributed takes part of that reward (99 % of it in #37311). Both describe the existing depositors' loss as part of the reward; neither describes a loss of deposited principal. Here the incumbent's redemption falls below its deposit.

Every executed attack run is net profitable; the lowest are +448.977744 USDG for one cycle (test 9) and +12,846.537849 USDG for one day (U1_2), both at the 10 bps confidence limit. The griefing rule therefore does not apply.

**Fallback.** If the amount taken is classified as yield, the applicable impact is **Theft of unclaimed yield (High)**, and the R7 tests prove it without U1: `test_R7_1` and `test_R7_4` to `test_R7_12` measure the take, and `test_R7_2` (entrant without B) and `test_R7_3` (B without entrant) are the controls. In every cycle B's open lowers the incumbents' Junior NAV by `r` before the entrant activates, so the entrant does not pay for `r`: in test 1 it activates 21.60 bps below the pre-open price, while without B it activates at the pre-open price (asserted in test 2). When the same `r` returns seconds after activation, the protocol records it as LP revenue (`CfdEngineSettlementSidecar.sol:604-613`), and the next reconcile shares it pro rata with the entrant. The incumbents lose α·r per cycle against the control at zero confidence (less the adverse-fill drag in tests 7–12), and the attacker keeps that amount net of B's two fees (both asserted within 1.325342 USDG in test 1).

### Conditions and limits

- **Price.** The basket is held at 1.20, on a two-feed harness basket (launch uses six feeds). From B's close until the exit settlement two hours later, the attacker's Junior shares carry their part of the pool's exposure to the open skew position. In test 5, account A's long (1,140,000 USDG notional) stays open for the whole campaign. This price exposure is not measured. In U1, CONTROL runs the same deposit schedule, so the entrant holds the same exposure with and without B's legs.
- **Skew.** All R7 and U1 runs use a skew of 0.38 of depth (the launch cap is 0.40), held for the whole cycle:
  - by an independent trader in tests 1–4 and 6–12;
  - by attacker account A in test 5, where A's net over the campaign is −925.514131 USDG;
  - by an independent trader for the whole day in U1, who closes it after the last cycle.
  - The exit direction uses only the attacker's own position.
- **Capital.** Every R7 and U1 run with an entrant uses X = 4 × incumbent Junior (α ≈ 0.8). B is funded with 5 % of the skew notional (R7: 57,000 / 570,000 USDG; U1: 189,996 / 570,000 USDG), and A with 114,000 USDG in test 5. In U1 the attacker holds about 40.6M (Junior-only pool) or 41.0M USDG in total, including a 1 % idle buffer on X. The program page states: "A high capital requirement is generally not a valid reason to downgrade a report. Attacks requiring USD 100 million or more to execute are considered case by case."
- **Senior/Junior ratio.** The ratio of Senior to incumbent Junior is 2 in R7, 0 (U1_1, U1_2: Senior holds only its 1 USDG seed) or 2 (U1_3, U1_4) in U1, and 3 in A3, A4, A6 and A7. None of the runs reported here has a ratio above 3. The Junior withdrawal covenant (`HousePoolSeniorCapacityLib.sol:75-89`; `HousePool.sol:1205-1217`) keeps the Junior principal left after a withdrawal at no less than a quarter of the committed Senior exposure at the launch 8,000 bps Senior share (exposure × 2,000 / 8,000), so it does not limit the attacker's exit as long as Senior exposure is at most four times the Junior principal that remains, that is, a Senior share of at most 80 %, the launch cap.
- **Break-even skew (derived, not executed).** From the relation asserted in tests 7–12, with the coupon ignored: 0.16/α of depth at confidence 0 (0.20 at α = 0.8), and 0.275 at α = 0.8 with 10 bps confidence.
- **Scope of each run.** Each run has at most one entrant and no other LP requests, trades or liquidations in the window. The exception is the negative control (test 3): it has no Junior entrant, and a 10,000 USDG honest Senior deposit for epoch `b`, made in both of its worlds, makes the reconcile at `b` run. R7 runs at most three consecutive cycles (one cycle takes 3 h); U1 runs eight, one trading day. Longer campaigns were not executed.
- **Costs not counted.** Gas and Pyth update fees are not included (the harness Pyth fee is 0). One cycle takes about a dozen attacker transactions.
- **Exit-direction rebate.** The close-time rebate in the exit direction is paid only from free pool cash (`WHITEPAPER.md:885-886`). In RT6, enough free cash was available.
- **RT6 configuration.** RT6 uses the same launch risk parameters and Senior cap, with zero confidence. It keeps two harness defaults that differ from launch: adverseConfidenceMultiplierBps 2,000 and minOpenNotionalAssets 100 USDG. With zero confidence and multi-million notionals, neither value affects its results.
- **Separate addresses.** Account A is separate from B because each clearinghouse account holds one directional position (`SECURITY.md:1448`). No per-address limit is used: Junior requests are uncapped (`TrancheVault.sol:1325`), and execution and settlement calls are permissionless.

## Recommendation

Options for the project to evaluate (none of them is tested here):
1. **Do not pay a skew-reducing open's negative VPI out of pool cash.** Keep it as an account-level offset that is netted at close, or against a later charge. LP NAV is then unchanged by the open, and the reserve rule at `WHITEPAPER.md:887-888` is unaffected. The exit-direction counterpart is to hold open-time positive VPI in a pending bucket until the position closes, and to pay close rebates from that bucket first.
2. **Defer deposit activation, or apply an entry haircut,** while unconsumed VPI rebate reserves are outstanding.
3. **Price LP entry and exit on a NAV adjusted for the VPI that live positions can still reverse.** Count reserve-backed negative accruals as a receivable, and hold positive accruals back until close. This reverses the documented choice at `ACCOUNTING_SPEC.md:1064-1066` and `WHITEPAPER.md:887-888`.

## References

- `packages/perps/src/libraries/CfdEnginePlanLib.sol:333, 348-349, 473-480`
- `packages/perps/src/CfdEngineSettlementSidecar.sol:442-444, 454-463, 466-478, 498, 604-613, 690-693, 905-926`
- `packages/perps/src/MarginClearinghouse.sol:883-907`
- `packages/perps/src/libraries/CloseAccountingLib.sol:103-118`
- `packages/perps/src/libraries/HousePoolAccountingLib.sol:60-78`; `packages/perps/src/TerminalNavBookV2.sol:208-237`
- `packages/perps/src/libraries/HousePoolReconcilePlanLib.sol:84-104`; `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:98-110, 140-159, 167-184`
- `packages/perps/src/HousePool.sol:604-613, 797-821, 835, 914-920, 1205-1217, 1648`; `packages/perps/src/libraries/HousePoolSeniorCapacityLib.sol:64-90`
- `packages/perps/src/TrancheVault.sol:545-556, 617-619, 851-860, 1108-1110, 1313-1326, 1398-1410`
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol:549-581`; `packages/perps/src/PletherOracle.sol:869-873, 905-923`
- `packages/perps/ACCOUNTING_SPEC.md:21-22, 141-142, 172-173, 228-229, 731-740, 821-822, 1064-1066, 1181-1184, 1288`
- `packages/perps/SECURITY.md:376, 863, 871-872, 1044-1048, 1328-1457, 1380-1399, 1434-1444, 1448`
- `packages/perps/WHITEPAPER.md:510-512, 544-547, 715-716, 734-735, 742-744, 885-888, 922-939, 969-970, 974-977`
- `audits/scan-plether-fi-plether-core-d4541b49.md:1693, 1742`; SC Audit Studio pre-audit SC-H1
- Immunefi report #37315 (Jito Restaking; "Theft of unclaimed yield", High): https://reports.immunefi.com/jito-restaking/37315
- Immunefi report #37311 (Jito Restaking; "Theft of unclaimed yield", High): https://reports.immunefi.com/jito-restaking/37311


## Link to Proof of Concept

https://gist.github.com/itko-security/2d04f93757cc926e340eb3ddd5784850

## Proof of Concept

## Proof of Concept

The PoC runs in the repository's Foundry suite at commit a82d4fe without modifying any existing file; `git diff` of `packages/perps/src` is empty. Only the repository's `MockPyth` and `MockSettlementToken` are mocked. No `vm.store`, `vm.etch`, `vm.mockCall` or `deal` is used.

Files. All five are new. They are in the secret Gist attached to this report, and the U1 test, which demonstrates the claimed Critical impact, is also reproduced at the end of this section. They must be placed at exactly these paths, because the tests import `../LaunchParamsPocBase.sol` and the U1 test imports `../r7/R7_JitJuniorCapturesVpiClawback.t.sol`.
1. `packages/perps/test/poc/LaunchParamsPocBase.sol` (35 lines): sets the risk parameters of the repository's `BasePerpTest` to the DEPLOYMENT.md "Initial release parameters".
2. `packages/perps/test/poc/r7/R7_JitJuniorCapturesVpiClawback.t.sol` (1,292 lines, 12 tests): the entry direction.
3. `packages/perps/test/poc/u1/U1_JitJuniorPrincipalLoss.t.sol` (425 lines, 4 tests): one trading day with every position closed and every incumbent fully redeemed. Its contract inherits the R7 contract unchanged.
4. `packages/perps/test/poc/rt6/RT6_LpNavTiming.t.sol` and `packages/perps/test/poc/rt6/RT6_LpNavTimingExtra.t.sol`: the exit direction (A3, A4, A6) and the entry direction at Senior/Junior = 3 (A7). The second file inherits from the first. The two files also contain tests this report does not use (A1, A2, D1 and B1; D1 is a donation check that mints USDG directly into the pool, both vaults and the clearinghouse). The third `forge test` command's `--match-test` filter excludes them.

Check out the commit, then copy the five files from the Gist into place:

```
git clone https://github.com/immunefi-team/audit-comp-plether.git && cd audit-comp-plether && git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
git clone https://gist.github.com/2d04f93757cc926e340eb3ddd5784850.git /tmp/r7-poc
cd packages/perps && mkdir -p test/poc/r7 test/poc/u1 test/poc/rt6
cp /tmp/r7-poc/LaunchParamsPocBase.sol test/poc/
cp /tmp/r7-poc/R7_JitJuniorCapturesVpiClawback.t.sol test/poc/r7/
cp /tmp/r7-poc/U1_JitJuniorPrincipalLoss.t.sol test/poc/u1/
cp /tmp/r7-poc/RT6_LpNavTiming.t.sol /tmp/r7-poc/RT6_LpNavTimingExtra.t.sol test/poc/rt6/
```

Run from `packages/perps` in a POSIX shell (bash or zsh), using the repository's default profile (optimizer on, 200 runs, via_ir) and forge 1.5.1-stable:

```
FOUNDRY_TEST=test/poc/u1 forge test --match-path test/poc/u1/U1_JitJuniorPrincipalLoss.t.sol --match-test U1 -vv
FOUNDRY_TEST=test/poc/r7 forge test --match-path test/poc/r7/R7_JitJuniorCapturesVpiClawback.t.sol -vv
FOUNDRY_TEST=test/poc/rt6 forge test --match-path test/poc/rt6/RT6_LpNavTimingExtra.t.sol --match-test "A3|A4|A6|A7" -vv
```

The first command reports `Suite result: ok. 4 passed; 0 failed; 0 skipped`. It needs `--match-test U1`, because the U1 contract inherits the twelve R7 tests; the filter selects only `test_U1_1` to `test_U1_4`. The second command reports `Suite result: ok. 12 passed; 0 failed; 0 skipped`, and the third `Suite result: ok. 4 passed; 0 failed; 0 skipped`. The `FOUNDRY_TEST` prefix limits the build to `src`, that PoC directory and their imports. Every output shown below comes from runs scoped this way; a fresh build of the U1 or R7 directory took about two minutes in this profile.

Setup:
- **Harness.** The tests extend `BasePerpTest`. Its router is `LegacyOrderRouterHarness`, which inherits the production `OrderRouter` and adds only a legacy scalar `commitOrder` overload and an `IS_TEST` marker constant. The PoC does not call that overload.
- **Risk parameters.** `LaunchParamsPocBase` sets vpiFactor 0.01e18, maxSkewRatio 0.4e18, maintenance / initial / FAD margin of 10 / 20 / 300 bps, baseCarry 500 bps, bounty 10 bps, keeper share 5,000 and protocol share 0.
- **Other launch values.** The R7 test sets three more through the owner's 72-hour propose/finalize path:
  - Senior cap 40,000,000 USDG / 8,000 bps;
  - adverseConfidenceMultiplierBps 2,500;
  - minOpenNotionalAssets 1,000 USDG.
- **Checks.** At the start of every test, the R7 test asserts the launch values it depends on (`_assertLaunchParams`). The U1 contract inherits the R7 setup and runs the same check.
- **Seeds and confidence.** Seeds are 1 USDG per tranche. Tests 1–6 publish zero confidence; tests 7–12 publish 1, 5 or 10 bps on every update. U1_1 and U1_3 publish zero confidence, U1_2 and U1_4 10 bps.
- **Entrypoints.** Every trader, keeper and LP action is an unprivileged call:
  - `OrderRouter.commitOrder` (V3 request), `executeOrder`, `updateMarkPrice` and `settleLpEpoch`;
  - `TrancheVault.requestDeposit`, `claimDeposit`, `requestRedeem` and `claimRedeem`;
  - `MarginClearinghouse.deposit` (and `MarginClearinghouse.withdraw` in RT6).
- **Schedule.** All worlds compared within a test start from one state snapshot (`vm.snapshotState` / `vm.revertToState`) and follow one absolute timestamp schedule; in R7 and U1, `_warpTo` reverts on overrun.

One attack cycle, as run by `_cycle` in the R7 file (the Description's "Attack cycle" table gives the measured values; U1's ATTACK world runs it eight times, one boundary `b` every 3 h):
1. Before `b − 300 s`, the attacker's LP wallet requests a Junior deposit of X for epoch `b` (`requestDeposit`).
2. At `b − 120 s`, account B commits a SHORT of T's size, and the attacker executes it as keeper in the next block. Pool assets fall by the rebate `r`, and B's VPI rebate reserve becomes `r`.
3. At `b + 1 s`, the attacker calls `settleLpEpoch`. The deposit activates on the Junior NAV lowered by `r`, and the LP wallet claims its shares (`claimDeposit`).
4. At `b + 2 s`, B commits its full close, which the attacker executes in the next block. The clamp charges at least `r` plus the fee, and the reserve `r` returns to the pool.
5. After the 1-hour cooldown, the LP wallet requests redemption of all its shares for epoch `b + 2 h` (`requestRedeem`); at `b + 2 h + 1 s` the attacker calls `settleLpEpoch` and the LP wallet calls `claimRedeem`.

| Test | Shows |
|---|---|
| `test_U1_1_JuniorOnly_OneDay_VictimRedeemsBelowDeposit`, `test_U1_2_JuniorOnly_OneDay_Confidence10Bps` | One trading day (8 cycles) in a Junior-only pool, J 10M / X 40M, at confidence 0 and 10 bps, in four worlds (ATTACK, CONTROL, NEG, BASELINE); T then closes and every incumbent redeems in full. Asserts: every redeem request fully funded and claimed; ATTACK `claimRedeem` below the deposit, and BASELINE, CONTROL and NEG at or above it; in ATTACK, value above the deposit after cycle 1, below it after cycle 2 and falling in every later cycle, while the other worlds never fall below it; loss vs BASELINE above 3 × the BASELINE-day revenue bound and above the incumbent's whole BASELINE yield; loss vs CONTROL = Σ(α·r − drag) within 2 %; r ≤ 0.08 % of depth at every B open; treasury difference = B's fees; attacker net > 0. |
| `test_U1_3_SeniorJunior_OneDay_LossExceedsAllRevenue`, `test_U1_4_SeniorJunior_OneDay_Confidence10Bps` | The same at J 10M / S 20M / X 40M. Also asserts that Senior's `claimRedeem` differs by at most 1 USDG between ATTACK and BASELINE and exceeds its deposit in both. |
| `test_R7_1_Attack_NaturalSkew_JitEntrantCapturesClawback` | Full cycle against the control. Asserts: r = 0.5·k·N²/D; reserve = r; NAV dip ≈ r; activation below the pre-open price; clamp branch (charge ≥ r + fee, reserve consumed = r); treasury = 2 fees; incumbent loss ≈ α·r and net vs control ≈ α·r − 2·fee, both within 1.325342 USDG; Senior unchanged. |
| `test_R7_2_Control_NoBLeg_NoExtraction` | Entrant without B: priced at the pre-open NAV, and pays only its coupon share. |
| `test_R7_3_NegativeControl_BRoundTripWithoutJit_NavRestored` | B's round trip without an entrant: the dip is booked at `b` and fully restored. |
| `test_R7_4_Repetition_ThreeCycles` | Three consecutive cycles, with the clamp branch and full reserve return asserted in each. |
| `test_R7_5_SelfMadeSkew_AttackerOwnsBothLegs_ThreeCycles` | Skew from attacker account A. Asserts: A's close rebate ≤ its open charge; attacker net > 0; incumbents below the run with no attacker activity. |
| `test_R7_6_LaunchScale_Junior10M_Senior20M_OneCycle` | Test 1 at J 10M / S 20M / X 40M. |
| `test_R7_7_Confidence1Bp_AttackVsControl_PocScale`, `test_R7_8_Confidence5Bps_AttackVsControl_PocScale`, `test_R7_9_Confidence10BpsMaxAccepted_AttackVsControl_PocScale` | Test 1 at 1 / 5 / 10 bps confidence. Fill shifts are asserted exactly from the Engine events. |
| `test_R7_10_Confidence1Bp_LaunchScale`, `test_R7_11_Confidence5Bps_LaunchScale`, `test_R7_12_Confidence10BpsMaxAccepted_LaunchScale` | Test 6 at 1 / 5 / 10 bps confidence. |
| `test_A3_ExitSandwich_LaunchScale_Senior30M_Junior10M`, `test_A4_ExitSandwich_RepeatsEveryCycle`, `test_A6_TimingControl_RoundTripBeforeSettlementGivesNoLpGain`, `test_A7_EntrySandwich_LaunchScale_HonestMaxSkew` (`RT6_LpNavExtraTest`) | Exit direction, three exit cycles, the timing control, and the entry direction at S/J = 3. |

Observed output of the U1 command (release build, optimizer on; excerpt):

```
Ran 4 tests for test/poc/u1/U1_JitJuniorPrincipalLoss.t.sol:U1_JitJuniorPrincipalLoss
[PASS] test_U1_1_JuniorOnly_OneDay_VictimRedeemsBelowDeposit() (gas: 116682966)
  === U1 (1) Junior-only pool, J 10M / X 40M, 8 cycles, confidence 0 ===
  per-cycle values: vault valuation at a fresh mark, same second in every world; realized = claimRedeem
  gross trading revenue bound, BASELINE day (2*cT + T close flow): 8936.739801 USDG
  incumbent Junior value before cycle 1 (fresh mark; deposit + cT share): 10007219.693828 USDG
  --- ATTACK (B legs + JIT) ---
    incumbent Junior LP redeemed (claimRedeem): 9955325.067168 USDG
    attacker NET (all attacker addresses): 21965.393356 USDG
    sum of B rebates r paid by the pool: 57832.891994 USDG
    sum alpha*r: 46285.066506 USDG
    incumbent Junior value after cycle 1 (fresh mark): 10001448.129496 USDG
    incumbent Junior value after cycle 2 (fresh mark): 9995672.567372 USDG
    incumbent Junior value after cycle 8 (fresh mark): 9960934.816881 USDG
  --- BASELINE (no attacker) ---
    incumbent Junior LP redeemed (claimRedeem): 10001717.044833 USDG
    pool flow at T's final close (rebate to T, net of carry and fee): -5502.649317 USDG
    incumbent Junior value after cycle 1 (fresh mark): 10007219.693801 USDG
    incumbent Junior value after cycle 8 (fresh mark): 10007219.693609 USDG
  --- summary ---
  ATTACK: incumbent Junior redeemed minus deposit: -44674.932832 USDG
  BASELINE: incumbent Junior redeemed minus deposit: 1717.044833 USDG
  CONTROL: incumbent Junior redeemed minus deposit: 1615.080433 USDG
  NEG: incumbent Junior redeemed minus deposit: 1759.652599 USDG
  incumbent loss vs BASELINE: 46391.977665 USDG
  incumbent loss vs CONTROL: 46290.013265 USDG
  loss vs BASELINE / revenue bound 5.1911x
  ATTACK principal shortfall in bps of deposit 44.67 bps
  ATTACK: first cycle after which incumbent value < deposit 2
[PASS] test_U1_2_JuniorOnly_OneDay_Confidence10Bps() (gas: 117390904)
  ATTACK: incumbent Junior redeemed minus deposit: -33655.807162 USDG
  BASELINE: incumbent Junior redeemed minus deposit: 3611.937429 USDG
  loss vs BASELINE / revenue bound 3.4417x
[PASS] test_U1_3_SeniorJunior_OneDay_LossExceedsAllRevenue() (gas: 119679562)
  ATTACK: incumbent Junior redeemed minus deposit: -136463.071244 USDG
  BASELINE: incumbent Junior redeemed minus deposit: 402.386472 USDG
  Senior: attack minus baseline: 0.527123 USDG
  loss vs BASELINE / revenue bound 5.1047x
[PASS] test_U1_4_SeniorJunior_OneDay_Confidence10Bps() (gas: 120420593)
  ATTACK: incumbent Junior redeemed minus deposit: -103384.037261 USDG
  BASELINE: incumbent Junior redeemed minus deposit: 6087.182932 USDG
  Senior: attack minus baseline: 0.527123 USDG
  loss vs BASELINE / revenue bound 3.3698x
Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 358.94ms (1.28s CPU time)
```

Observed output of the R7 command (release build, optimizer on; excerpt):

```
[PASS] test_R7_12_Confidence10BpsMaxAccepted_LaunchScale() (gas: 13616175)
  attacker NET (attack, absolute): 4489.791427 USDG
  incumbent Junior loss vs control: 13901.891837 USDG
[PASS] test_R7_1_Attack_NaturalSkew_JitEntrantCapturesClawback() (gas: 13644208)
  pool totalAssets drop at B open (r): 2164.435835 USDG
  B vpiRebateReserve after open: 2164.435835 USDG
  Junior NAV before B open: 1002131.082085 USDG
  Junior NAV after B open: 999966.641176 USDG
  activation discount vs pre-open NAV 21.60 bps
  alpha (entrant share of post-entry Junior) ppm 800005
  unclamped close VPI at depth D+X: 928.285241 USDG
  close action charge assessed (clamped VPI + fee + carry): 2620.435835 USDG
  B reserve consumed at close: 2164.435835 USDG
  attacker NET (absolute, after B fees): 790.328977 USDG
  attacker NET vs control: 819.540823 USDG
  incumbent Junior loss vs control: 1731.539093 USDG
  incumbent loss in bps of incumbent Junior 17.27 bps
[PASS] test_R7_3_NegativeControl_BRoundTripWithoutJit_NavRestored() (gas: 11010707)
  stored juniorPrincipal after b settlement, B world: 999966.032348 USDG
  stored juniorPrincipal after b settlement, no-B world: 1002130.468183 USDG
  incumbent JLP difference (B - none): 0.035145 USDG
  treasury difference (B - none) = B's two fees: 912.000000 USDG
[PASS] test_R7_4_Repetition_ThreeCycles() (gas: 28347528)
  cumulative attacker NET (attack, absolute): 2375.733126 USDG
  cumulative incumbent Junior loss vs control: 5199.366316 USDG
  cumulative incumbent loss in bps 51.88 bps
[PASS] test_R7_5_SelfMadeSkew_AttackerOwnsBothLegs_ThreeCycles() (gas: 24505296)
  A open: VPI charge c paid to the pool: 2165.998556 USDG
  A close: rebate paid by the pool: 1696.684425 USDG
  attacker NET over the campaign (absolute, all accounts): 1450.418995 USDG
  incumbent Junior loss vs no-attacker baseline: 4642.421468 USDG
  treasury gain (attack): 3192.000000 USDG
  incumbent loss in bps 46.43 bps
[PASS] test_R7_6_LaunchScale_Junior10M_Senior20M_OneCycle() (gas: 13484821)
  attacker NET (absolute): 7903.303339 USDG
  incumbent Junior loss vs control: 17315.419987 USDG
Suite result: ok. 12 passed; 0 failed; 0 skipped; finished in 59.76ms (364.99ms CPU time)
```

Observed output of the RT6 command (release build, optimizer on; excerpt, `test_A3` only; values in USDG atoms, 6 decimals):

```
[PASS] test_A3_ExitSandwich_LaunchScale_Senior30M_Junior10M() (gas: 9018793)
  A3 VPI charged into pool at attacker open 63680398407
  A3 net rebate paid from pool at attacker close 50911852575
  A3 attacker net (plain)
  -438362575
  A3 attacker net (attack)
  24969688449
  A3 honest junior value (plain) 9999616439766
  A3 honest junior value (attack) 9961440316452
```

### `packages/perps/test/poc/u1/U1_JitJuniorPrincipalLoss.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {R7_JitJuniorCapturesVpiClawback} from "../r7/R7_JitJuniorCapturesVpiClawback.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {console2} from "forge-std/console2.sol";

/// @title U1: repeated R7 cycles take incumbent Junior principal, not only yield.
/// @notice Reuses the R7 harness unchanged (launch params, seeds 1e6, production entrypoints, basket 1.20) and runs a
///         full trading day (8 cycles) in four worlds on one absolute schedule:
///           ATTACK   = B's skew-reducing round trip + JIT Junior deposit/redemption every cycle,
///           CONTROL  = the same JIT deposits/redemptions without B,
///           NEG      = B's round trips without any JIT deposit,
///           BASELINE = no attacker activity.
///         After the last cycle, trader T closes (no open position anywhere) and every incumbent LP requests a full
///         redemption, which is settled through the permissionless Router and claimed. The test compares the USDG each
///         incumbent actually receives against the USDG it deposited.
///         The per-cycle incumbent values are vault valuations (convertToAssets) read at a fresh mark at the same
///         second in every world (`_juniorValueAtFreshMark`); the realized figures are the final claimRedeem amounts.
/// @dev Run only the U1 tests: `--match-test U1` (the contract inherits the twelve R7 tests).
contract U1_JitJuniorPrincipalLoss is R7_JitJuniorCapturesVpiClawback {

    uint256 internal constant U1_CYCLES = 8; // one day: entry every 3 h

    struct U1World {
        uint256 attackerStart;
        uint256 attackerEnd;
        int256 attackerNet;
        uint256 treasuryStart;
        uint256 treasuryEnd;
        uint256 jlpPaid; // USDG the incumbent Junior LP receives from claimRedeem at the end
        uint256 slpPaid; // USDG the incumbent Senior LP receives from claimRedeem at the end
        int256 tCloseFlow; // pool totalAssets change at T's close (VPI rebate paid back to T, net of collected carry)
        uint256 sumRebate;
        uint256 sumAlphaR;
        uint256 sumShiftDrag;
        uint256 sumBFees;
        uint256 exitBoundary;
        uint256[] jlpValueAfterCycle;
    }

    struct U1Result {
        uint256 cT; // T's open VPI charge (gross VPI revenue the pool ever received from T)
        uint256 revenueBound; // cT + carry bound over the whole campaign
        uint256 jlpValueBefore; // incumbent Junior value right before the first cycle (includes cT)
        uint256 slpValueBefore;
        U1World atk;
        U1World ctl;
        U1World neg;
        U1World base;
    }

    // ------------------------------------------------------------------
    // Setup / world runner
    // ------------------------------------------------------------------

    /// @dev R7 bootstrap, but the Senior incumbent is optional (seniorIncumbent == 0 -> Senior holds only its seed).
    function _bootstrapU1() internal {
        settlementToken.mint(JLP, juniorIncumbent);
        uint256 jid = _requestDeposit(juniorVault, JLP, juniorIncumbent);
        _warpTo(pool.lpEpochStart(jid) + 1);
        _settleViaRouter(HONEST_KEEPER);
        _claimDeposit(juniorVault, jid, JLP);
        uint256 lastDeposit = juniorVault.lastDepositTime(JLP) + juniorVault.DEPOSIT_COOLDOWN();
        if (seniorIncumbent > 0) {
            settlementToken.mint(SLP, seniorIncumbent);
            uint256 sid = _requestDeposit(seniorVault, SLP, seniorIncumbent);
            _warpTo(pool.lpEpochStart(sid) + 1);
            _settleViaRouter(HONEST_KEEPER);
            _claimDeposit(seniorVault, sid, SLP);
            lastDeposit = seniorVault.lastDepositTime(SLP) + seniorVault.DEPOSIT_COOLDOWN();
        }
        _warpTo(lastDeposit + 1);
        uint256 depth = pool.totalAssets();
        uint256 lots = ((depth * SKEW_PCT) / 100) * 1e20 / PRICE / 1e20;
        skewSize = lots * 1e20;
        _logUsdg("incumbent Junior deposit", int256(juniorIncumbent));
        _logUsdg("incumbent Senior deposit", int256(seniorIncumbent));
        _logUsdg("pool depth D after bootstrap", int256(depth));
        _logUsdg("skew notional N (0.38 * D)", int256(_skewNotional()));
    }

    function _runU1World(
        bool bLeg,
        bool jit,
        uint256 tClose
    ) internal returns (U1World memory w) {
        w.attackerStart = _attackerValue();
        w.treasuryStart = _treasury();
        w.jlpValueAfterCycle = new uint256[](U1_CYCLES);
        uint256 shiftCostPerLeg = (_adverseShift() * skewSize) / 1e20;
        for (uint256 i; i < U1_CYCLES; ++i) {
            CycleLog memory c = _cycle(bLeg, jit);
            if (bLeg) {
                assertEq(c.bReserveAfterOpen, c.rebate, "B locked exactly r");
                assertEq(c.bReserveConsumed, c.rebate, "full clawback each cycle");
                assertGe(c.closeChargeAssessed, c.rebate + c.closeFee, "clamp branch each cycle");
                // size of one cycle's rebate: r = 0.5 * k * N^2 / D with N = 0.38 * D, i.e. below 0.08% of the depth
                assertLe(c.rebate * 10_000, 8 * c.depthAtOpen, "r <= 0.08% of pool depth at B's open");
                w.sumRebate += c.rebate;
                w.sumBFees += c.treasuryGain;
            }
            if (bLeg && jit) {
                w.sumAlphaR += (c.alphaE18 * c.rebate) / 1e18;
                w.sumShiftDrag += ((2e18 - c.alphaE18) * shiftCostPerLeg) / 1e18;
            }
            bool refreshed;
            (w.jlpValueAfterCycle[i], refreshed) = _juniorValueAtFreshMark(JLP);
            // Worlds with a JIT exit settled through the Router at this very second (fresh mark, read directly);
            // the other worlds need the read-only refresh.
            assertEq(refreshed, !jit, "read-only mark refresh exactly where no settlement ran at this second");
        }

        // Campaign end, same absolute time in every world: T closes, so no position is open anywhere.
        _warpTo(tClose);
        uint256 poolBefore = pool.totalAssets();
        _trade(TRADER_T, CfdTypes.Side.LONG, true, skewSize, 0, HONEST_KEEPER);
        w.tCloseFlow = int256(pool.totalAssets()) - int256(poolBefore);
        (uint256 tSize,,,,,,) = engine.positions(TRADER_T);
        (uint256 bSize,,,,,,) = engine.positions(ATT_B);
        assertEq(tSize, 0, "T flat");
        assertEq(bSize, 0, "B flat");

        // Every incumbent LP exits in full through the production async flow.
        _warpTo(tClose + 60);
        _refreshMark(HONEST_KEEPER);
        uint256 jId = _requestRedeemAll(juniorVault, JLP);
        uint256 sId;
        if (seniorIncumbent > 0) {
            sId = _requestRedeemAll(seniorVault, SLP);
        }
        w.exitBoundary = pool.lpEpochStart(jId);
        _warpTo(w.exitBoundary + 1);
        _settleViaRouter(HONEST_KEEPER);
        w.jlpPaid = _claimRedeem(juniorVault, jId, JLP);
        assertEq(juniorVault.balanceOf(JLP), 0, "incumbent Junior LP holds no shares");
        assertEq(juniorVault.pendingRedeemRequest(jId, JLP), 0, "incumbent Junior LP redeem request fully funded");
        assertEq(juniorVault.claimableRedeemRequest(jId, JLP), 0, "incumbent Junior LP redeem request fully claimed");
        if (seniorIncumbent > 0) {
            w.slpPaid = _claimRedeem(seniorVault, sId, SLP);
            assertEq(seniorVault.balanceOf(SLP), 0, "incumbent Senior LP holds no shares");
            assertEq(seniorVault.pendingRedeemRequest(sId, SLP), 0, "incumbent Senior LP redeem request fully funded");
            assertEq(seniorVault.claimableRedeemRequest(sId, SLP), 0, "incumbent Senior LP redeem request fully claimed");
        }
        w.attackerEnd = _attackerValue();
        w.attackerNet = int256(w.attackerEnd) - int256(w.attackerStart);
        w.treasuryEnd = _treasury();
    }

    /// @dev Value of `lp`'s Junior shares at a fresh mark, at the current second, without changing the world.
    ///      The vault valuation (convertToAssets -> HousePool.getPendingTrancheState) applies the mark-dependent part of
    ///      the pending reconcile only while the cached mark is fresh (HousePool._previewPendingAccountingState passes
    ///      markFresh to planReconcile; 60 s limit). A world whose JIT exit was settled through the Router at this second
    ///      already has a fresh mark and is read directly. In a world without a JIT deposit nothing has refreshed the mark
    ///      since about b, so the read takes a state snapshot, applies one permissionless mark refresh at this second,
    ///      reads, and reverts to the snapshot. The refresh never becomes part of that world's history: every trade,
    ///      settlement and claimRedeem amount is the same as without this read.
    function _juniorValueAtFreshMark(
        address lp
    ) internal returns (uint256 value, bool refreshed) {
        if (pool.getPoolLiquidityView().markFresh) {
            return (_juniorValueOf(lp), false);
        }
        uint256 t = _now();
        uint256 snapshotId = vm.snapshotState();
        _refreshMark(HONEST_KEEPER);
        assertEq(_now(), t, "fresh-mark read at the same second");
        assertTrue(pool.getPoolLiquidityView().markFresh, "mark fresh for the read");
        value = _juniorValueOf(lp);
        assertTrue(vm.revertToStateAndDelete(snapshotId), "read-only mark refresh reverted");
        assertEq(_now(), t, "clock restored after the read");
        refreshed = true;
    }

    function _u1Campaign(
        string memory label
    ) internal returns (U1Result memory r) {
        _assertLaunchParams();
        _bootstrapU1();
        uint256 poolBefore = pool.totalAssets();
        _openNaturalSkew();
        r.cT = pool.totalAssets() - poolBefore;
        _fundAttacker(false);
        _refreshMark(HONEST_KEEPER);
        r.jlpValueBefore = _juniorValueOf(JLP);
        r.slpValueBefore = _seniorValueOf(SLP);

        uint256 firstB = _nextUsableBoundary();
        uint256 tClose = firstB + (U1_CYCLES - 1) * 3 * EPOCH + 2 * EPOCH + 300;
        uint256 snap = vm.snapshotState();

        r.atk = _runU1World(true, true, tClose);
        assertTrue(vm.revertToState(snap));
        r.ctl = _runU1World(false, true, tClose);
        assertTrue(vm.revertToState(snap));
        r.neg = _runU1World(true, false, tClose);
        assertTrue(vm.revertToState(snap));
        r.base = _runU1World(false, false, tClose);

        // Upper bound on ALL trading revenue the pool received in the no-attacker day: T's VPI charge cT plus what T
        // pays at its close (carry, plus the close fee, which is not credited to the treasury on this rebate close).
        // T's close flow = that payment - T's VPI rebate, and the lifetime clamp caps the rebate at cT, so that
        // payment <= tCloseFlow + cT and gross revenue <= 2*cT + tCloseFlow.
        int256 gross = 2 * int256(r.cT) + r.base.tCloseFlow;
        assertGt(gross, 0, "gross revenue bound positive");
        r.revenueBound = uint256(gross);

        _logU1(label, r);
        _assertU1Common(r);
    }

    // ------------------------------------------------------------------
    // Assertions shared by every scenario
    // ------------------------------------------------------------------

    function _assertU1Common(
        U1Result memory r
    ) internal view {
        // one schedule: every world exits at the same boundary
        assertEq(r.atk.exitBoundary, r.base.exitBoundary, "same exit boundary (atk)");
        assertEq(r.ctl.exitBoundary, r.base.exitBoundary, "same exit boundary (ctl)");
        assertEq(r.neg.exitBoundary, r.base.exitBoundary, "same exit boundary (neg)");

        // negative control: B's round trips alone never hurt the incumbent. B even pays slightly more than r back
        // (its close is priced at the depth reduced by r, ~r^2/D per cycle), plus carry: a small gain, no loss.
        // With nonzero confidence B's two side-adverse fills per cycle also pay the pool (2 * shift * size per cycle).
        uint256 shiftCostPerLeg = (_adverseShift() * skewSize) / 1e20;
        assertGe(r.neg.jlpPaid + 1e6, r.base.jlpPaid, "NEG: incumbent not harmed by B alone");
        assertApproxEqAbs(
            r.neg.jlpPaid,
            r.base.jlpPaid,
            U1_CYCLES * (_carryBound(180) + 2 * shiftCostPerLeg) + r.neg.sumRebate / 1000,
            "NEG: only B's own payments (depth effect, carry, adverse fills)"
        );
        // control: a fair JIT entrant extracts nothing. The only CONTROL-vs-BASELINE difference for the incumbent is
        // T's carry (utilization-priced: the entrant's capital lowers the side rate while it is in the pool, WHITEPAPER
        // 5.2) plus the entrant's Senior-coupon share, both visible in T's close flow / bounded by the coupon.
        assertLe(r.ctl.attackerNet, 0, "CONTROL: no extraction without B");
        int256 ctlResidual =
            (int256(r.ctl.jlpPaid) - int256(r.base.jlpPaid)) - (r.ctl.tCloseFlow - r.base.tCloseFlow);
        assertGe(ctlResidual, -int256(U1_CYCLES * _carryBound(180)), "CONTROL: no loss beyond T's carry difference");
        uint256 couponPerHold =
            (seniorIncumbent * pool.seniorRateBps() * (2 * EPOCH + 600)) / (10_000 * 365 days) + 1e6;
        assertLe(
            ctlResidual,
            int256(U1_CYCLES * (couponPerHold + _carryBound(180))),
            "CONTROL: gain bounded by entrant coupon share"
        );

        // attack: the incumbent's monotone decline, one step per cycle
        for (uint256 i = 1; i < U1_CYCLES; ++i) {
            assertLt(r.atk.jlpValueAfterCycle[i], r.atk.jlpValueAfterCycle[i - 1], "ATTACK: incumbent value falls every cycle");
        }
        // per cycle (fresh-mark valuations at the same second in every world): with the attack the incumbent is still
        // above its deposit after cycle 1 and below it from cycle 2 on; without the attack no world is ever below it
        assertGt(r.atk.jlpValueAfterCycle[0], juniorIncumbent, "ATTACK: still above the deposit after cycle 1");
        assertLt(r.atk.jlpValueAfterCycle[1], juniorIncumbent, "ATTACK: below the deposit after cycle 2");
        for (uint256 i; i < U1_CYCLES; ++i) {
            assertGe(r.base.jlpValueAfterCycle[i], juniorIncumbent, "BASELINE: never below the deposit");
            assertGe(r.ctl.jlpValueAfterCycle[i], juniorIncumbent, "CONTROL: never below the deposit");
            assertGe(r.neg.jlpValueAfterCycle[i], juniorIncumbent, "NEG: never below the deposit");
        }
        int256 lossVsCtl = int256(r.ctl.jlpPaid) - int256(r.atk.jlpPaid);
        int256 lossVsBase = int256(r.base.jlpPaid) - int256(r.atk.jlpPaid);
        assertApproxEqRel(
            lossVsCtl, int256(r.atk.sumAlphaR) - int256(r.atk.sumShiftDrag), 0.02e18, "loss vs control = sum(alpha*r - drag)"
        );
        // the cumulative loss exceeds EVERY unit of trading revenue the pool received in the no-attacker day
        assertGt(lossVsBase, int256(r.revenueBound), "loss > gross VPI revenue + carry of the BASELINE day");
        assertGt(lossVsBase, 3 * int256(r.revenueBound), "loss > 3x that revenue bound");
        // and the incumbent's entire realized yield in the no-attacker day (redeemed minus deposited)
        assertGt(lossVsBase, int256(r.base.jlpPaid) - int256(juniorIncumbent), "loss > incumbent's whole yield");
        // the attacker is net profitable after both execution fees on every B leg, coupon and adverse fills
        assertGt(r.atk.attackerNet, 0, "attacker net profitable");
        // treasury difference = B's execution fees
        assertEq(
            (r.atk.treasuryEnd - r.atk.treasuryStart) - (r.base.treasuryEnd - r.base.treasuryStart),
            r.atk.sumBFees,
            "treasury: only B's fees differ"
        );
    }

    // ------------------------------------------------------------------
    // Tests
    // ------------------------------------------------------------------

    /// @dev Junior-only pool (Senior holds only its 1 USDG seed, so there is no Senior coupon): J 10M, X 40M.
    ///      BASELINE, CONTROL and NEG return at least the deposit; ATTACK returns less than the deposit.
    function test_U1_1_JuniorOnly_OneDay_VictimRedeemsBelowDeposit() public {
        juniorIncumbent = 10_000_000e6;
        seniorIncumbent = 0;
        jitAssets = 40_000_000e6;
        U1Result memory r = _u1Campaign("U1 (1) Junior-only pool, J 10M / X 40M, 8 cycles, confidence 0");
        _assertPrincipalBelowDeposit(r);
    }

    /// @dev Same at the widest basket confidence the oracle accepts (10 bps = basketMaxConfidenceRatioBps).
    function test_U1_2_JuniorOnly_OneDay_Confidence10Bps() public {
        juniorIncumbent = 10_000_000e6;
        seniorIncumbent = 0;
        jitAssets = 40_000_000e6;
        confBps = 10;
        U1Result memory r = _u1Campaign("U1 (2) Junior-only pool, J 10M / X 40M, 8 cycles, confidence 10 bps");
        _assertPrincipalBelowDeposit(r);
    }

    /// @dev Launch-shaped tranche split J 10M / S 20M (R7 test 6 scale), X 40M. Junior pays the Senior coupon by design,
    ///      so the comparison is against the no-attacker BASELINE; Senior is unaffected. BASELINE, CONTROL and NEG
    ///      return at least the Junior deposit; ATTACK returns less than the Junior deposit.
    function test_U1_3_SeniorJunior_OneDay_LossExceedsAllRevenue() public {
        juniorIncumbent = 10_000_000e6;
        seniorIncumbent = 20_000_000e6;
        jitAssets = 40_000_000e6;
        U1Result memory r = _u1Campaign("U1 (3) J 10M / S 20M / X 40M, 8 cycles, confidence 0");
        _assertPrincipalBelowDeposit(r);
        _assertSeniorShape(r);
    }

    function test_U1_4_SeniorJunior_OneDay_Confidence10Bps() public {
        juniorIncumbent = 10_000_000e6;
        seniorIncumbent = 20_000_000e6;
        jitAssets = 40_000_000e6;
        confBps = 10;
        U1Result memory r = _u1Campaign("U1 (4) J 10M / S 20M / X 40M, 8 cycles, confidence 10 bps");
        _assertPrincipalBelowDeposit(r);
        _assertSeniorShape(r);
    }

    function _assertPrincipalBelowDeposit(
        U1Result memory r
    ) internal view {
        assertGe(r.base.jlpPaid, juniorIncumbent, "BASELINE: incumbent redeems at least its deposit");
        assertGe(r.ctl.jlpPaid, juniorIncumbent, "CONTROL: incumbent redeems at least its deposit");
        assertGe(r.neg.jlpPaid, juniorIncumbent, "NEG: incumbent redeems at least its deposit");
        assertLt(r.atk.jlpPaid, juniorIncumbent, "ATTACK: incumbent redeems LESS than it deposited");
    }

    function _assertSeniorShape(
        U1Result memory r
    ) internal view {
        assertApproxEqAbs(r.atk.slpPaid, r.base.slpPaid, 1e6, "Senior unaffected by the attack (within 1 USDG)");
        assertGt(r.base.slpPaid, seniorIncumbent, "BASELINE: Senior redeems its deposit plus coupon");
        assertGt(r.atk.slpPaid, seniorIncumbent, "ATTACK: Senior redeems its deposit plus coupon");
    }

    // ------------------------------------------------------------------
    // Logging
    // ------------------------------------------------------------------

    function _logU1World(
        string memory name,
        U1World memory w
    ) internal pure {
        console2.log(string.concat("--- ", name, " ---"));
        _logUsdg("  incumbent Junior LP redeemed (claimRedeem)", int256(w.jlpPaid));
        _logUsdg("  incumbent Senior LP redeemed (claimRedeem)", int256(w.slpPaid));
        _logUsdg("  attacker NET (all attacker addresses)", w.attackerNet);
        _logUsdg("  treasury gain", int256(w.treasuryEnd) - int256(w.treasuryStart));
        _logUsdg("  pool flow at T's final close (rebate to T, net of carry and fee)", w.tCloseFlow);
        _logUsdg("  sum of B rebates r paid by the pool", int256(w.sumRebate));
        _logUsdg("  sum alpha*r", int256(w.sumAlphaR));
        _logUsdg("  sum (2-alpha)*adverse-fill cost", int256(w.sumShiftDrag));
        _logUsdg("  B execution fees", int256(w.sumBFees));
        for (uint256 i; i < w.jlpValueAfterCycle.length; ++i) {
            _logUsdg(
                string.concat("  incumbent Junior value after cycle ", _u2s(i + 1), " (fresh mark)"),
                int256(w.jlpValueAfterCycle[i])
            );
        }
    }

    function _logU1(
        string memory label,
        U1Result memory r
    ) internal view {
        console2.log(string.concat("=== ", label, " ==="));
        console2.log("per-cycle values: vault valuation at a fresh mark, same second in every world; realized = claimRedeem");
        _logUsdg("T open: VPI charge cT received by the pool (all VPI revenue from T)", int256(r.cT));
        _logUsdg("gross trading revenue bound, BASELINE day (2*cT + T close flow)", int256(r.revenueBound));
        _logUsdg("incumbent Junior value before cycle 1 (fresh mark; deposit + cT share)", int256(r.jlpValueBefore));
        _logU1World("ATTACK (B legs + JIT)", r.atk);
        _logU1World("CONTROL (JIT only)", r.ctl);
        _logU1World("NEG (B legs only)", r.neg);
        _logU1World("BASELINE (no attacker)", r.base);
        console2.log("--- summary ---");
        _logUsdg("ATTACK: incumbent Junior redeemed minus deposit", int256(r.atk.jlpPaid) - int256(juniorIncumbent));
        _logUsdg("BASELINE: incumbent Junior redeemed minus deposit", int256(r.base.jlpPaid) - int256(juniorIncumbent));
        _logUsdg("CONTROL: incumbent Junior redeemed minus deposit", int256(r.ctl.jlpPaid) - int256(juniorIncumbent));
        _logUsdg("NEG: incumbent Junior redeemed minus deposit", int256(r.neg.jlpPaid) - int256(juniorIncumbent));
        _logUsdg("incumbent loss vs BASELINE", int256(r.base.jlpPaid) - int256(r.atk.jlpPaid));
        _logUsdg("incumbent loss vs CONTROL", int256(r.ctl.jlpPaid) - int256(r.atk.jlpPaid));
        _logUsdg("Senior: attack minus baseline", int256(r.atk.slpPaid) - int256(r.base.slpPaid));
        _logUsdg("attacker NET vs CONTROL", r.atk.attackerNet - r.ctl.attackerNet);
        if (r.base.jlpPaid > r.atk.jlpPaid) {
            console2.log(
                "loss vs BASELINE / revenue bound", _fmtRatio(r.base.jlpPaid - r.atk.jlpPaid, r.revenueBound)
            );
        }
        if (juniorIncumbent > r.atk.jlpPaid) {
            console2.log(
                "ATTACK principal shortfall in bps of deposit",
                _fmtBps(juniorIncumbent - r.atk.jlpPaid, juniorIncumbent)
            );
        }
        for (uint256 i; i < r.atk.jlpValueAfterCycle.length; ++i) {
            if (r.atk.jlpValueAfterCycle[i] < juniorIncumbent) {
                console2.log("ATTACK: first cycle after which incumbent value < deposit", i + 1);
                break;
            }
        }
    }

    /// @dev num / den with four decimals, rounded down (e.g. "3.3698x").
    function _fmtRatio(
        uint256 num,
        uint256 den
    ) internal pure returns (string memory) {
        uint256 e4 = (num * 1e4) / den;
        string memory frac = _u2s(e4 % 1e4);
        while (bytes(frac).length < 4) {
            frac = string.concat("0", frac);
        }
        return string.concat(_u2s(e4 / 1e4), ".", frac, "x");
    }
}
```

