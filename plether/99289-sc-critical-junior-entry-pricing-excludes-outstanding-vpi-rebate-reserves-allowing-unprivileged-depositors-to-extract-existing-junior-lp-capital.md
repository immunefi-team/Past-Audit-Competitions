# \#99289 [SC-Critical] Junior entry pricing excludes outstanding VPI rebate reserves, allowing unprivileged depositors to extract existing Junior LP capital

**Submitted on Nov 12th 2026 at 22:04:25 UTC by @CarrotStudio for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99289
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Plether intentionally leaves the VPI rebate reserve out of the NAV that prices both LP deposits and redemptions (`SECURITY.md:871-872`), so that LPs cannot withdraw it before the position behind it settles (`WHITEPAPER.md:887-888`); we do not dispute that purpose. This report shows what the exclusion does to share issuance: by timing when its Junior deposit is activated, an unprivileged depositor using public calls only moves part of the existing Junior LPs' capital to itself, without the pool gaining new value. The reserve is pool cash: when an order reduces skew, the pool pays the trader a rebate that stays locked until the position ends and then returns to the pool. A deposit activated meanwhile is priced at V - R, and when the reserve returns the new shares own part of it. With every other flow removed, the entrant gains exactly D*R/(V - R + D) (deposit D, reserve R, Junior value V) and the existing Junior LPs lose the same amount, with no additional total LP equity created and the trader's lifetime VPI at zero; settling the same epoch after the reserve returns gives the entrant nothing.

In the PoC (release parameters, 38% skew, 6,000,000 USDG deposit against a 2,000,000 USDG Junior tranche), attack and control differ only in when the epoch is settled, and one round is enough to see it: the attacker's accounts end +874.96 USDG against -1,321.49 in the control. After 100 rounds the attacker's accounts end +97,620.57 USDG (control -132,173.47); attack minus control is +229,794.04 for the attacker's accounts and -229,781.81 for the existing Junior, with Senior and treasury unchanged, and the existing LPs redeem 177,073.49 USDG for one tenth of their shares against 200,051.67 in the control. Repeated until the withdrawal floor stops full exits (506 rounds), the existing Junior falls from 1,997,052.79 to 511,827.97 USDG and the attacker's accounts end +829,685.92 USDG net of trading fees, fully withdrawn. The items the specification does price into LP entry, a trader's collectible loss and unrealized profit, give the entrant 0.00 in the same sequence, and a depositor that places no order also profits from other traders' reserves.

## Vulnerability Details

Section letters (A-I) refer to the sections of `outputs.txt` in the gist.

### 1. The documented rule and its stated reason

"neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV" (`SECURITY.md:871-872`; also `ACCOUNTING_SPEC.md:1064-1066`), and one NAV prices deposits and redemptions (`ACCOUNTING_SPEC.md:160-161`). The reason given: "The reserve is excluded from the terminal price curve, so LPs cannot withdraw it as marked equity before the matching position obligation settles" (`WHITEPAPER.md:887-888`).

### 2. From the rule to the transfer

1. **The pool pays the rebate out.** On a skew-reducing open the pool pays the rebate to the clearinghouse (`CfdEngineSettlementSidecar.sol:442-443`), which locks it as the trader's "dedicated nonwithdrawable reserve equal to `max(-vpiAccrued, 0)`" (`SECURITY.md:866`; `CfdEngineSettlementSidecar.sol:466-470`, `MarginClearinghouse.sol:843`). Pool assets fall by exactly the reserve (1,002,199,600,798 to 1,002,000,039,753 atoms for a 199,561,045-atom reserve in the project's parity sequence, section C), and the existing LPs' value by exactly R while it is outstanding (4,004,889.18 to 4,002,002.08, section D).
2. **The trader cannot take it.** Withdrawing one atom more than B's free balance reverts, and the reserve stays locked at 2,887.24 (section D).
3. **Entry is priced without it.** LP entry and exit share one terminal snapshot whose price delta has no reserve term (code below).
4. **Anyone can activate pending deposits.** `OrderRouter.settleLpEpoch` (`OrderRouter.sol:479`) takes any fresh price ("Keepers are permissionless executors", `SECURITY.md:508`); in the PoC an unrelated address calls it.
5. **The reserve comes back.** Close and liquidation consume it (`CfdEngineSettlementSidecar.sol:692`, `:803`) through a function documented as "Returns reserved negative-VPI backing to the LP pool as cash recovery" (`MarginClearinghouse.sol:883`). The existing LPs' value is back at 4,004,891.26 (section D), and on six end paths, liquidation included, LP equity rose by the full reserve (section C); we did not test every market state.

```solidity
// CfdEngineProtocolLens.sol:70
snapshot.terminalLpPriceDeltaAssets = terminalSnapshot.terminalLpPriceDeltaAssets;

// HousePoolAccountingLib.sol:63-74 (buildReconcileSnapshot, used for entry and exit)
uint256 terminalAssets = engineSnapshot.physicalAssets;            // the rebate has already left
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;  // no term for the outstanding reserve
// ... the delta is added to assets, or to liabilities when negative
snapshot.distributable = terminalAssets - terminalLiabilities;
```

**One PoC round** (steps in the Proof of Concept). Existing LPs hold Junior 2,000,000 and Senior 2,000,000 and a third-party trader T holds a 38% LONG skew; the attacker's trader account B offsets it from e1-1 s to e1+3 s around an epoch boundary e1 (VPI -2,887.24, then +2,887.24), while its depositor LP2 (6,000,000 USDG) is activated when an unrelated address settles e1, and redeems after the 1-hour cooldown. LP2 ends +2,167.21 and B -1,292.64 (fees and price), net +874.96 with the keeper's bounties; with e1 settled after B's close the same orders end -1,321.49.

### 3. Causality: the settlement order, not the trading

- **Every other flow removed** (`VpiReserveOwnership.t.sol`: no carry, coupon or oracle confidence, flat price). In each of 10 rounds LP2's gain equals D*R/(V - R + D) exactly (asserted), and 0 within 2 atoms in the control (asserted); the existing Junior's change equals minus the gain, total LP value does not change and B's net VPI is zero. In the final redemption the existing LPs receive 3,978,860.06 against 4,000,630.87 in the control; the remaining differences are the traders' own depth effects, such as 2.08 USDG of VPI per round for B in the control (section D).
- **Release parameters, same orders at the same times:** attack minus control in the first Impact table (section A).
- **Exit before the reserve returns.** An entrant that redeems while the reserve is still outstanding gets its deposit back: -0.00 on 6,000,000 (asserted within 2 atoms, section D).
- **Counterfactual correction.** With `fix.patch`, which adds the outstanding reserves to the price delta at `CfdEngineProtocolLens.sol:70`, the closed-form gain is 0 and every attack we reran loses (sections A, B, D, E, F).

### 4. Does the documented design cover this consequence?

**Withdrawal and ownership are separate questions in the same documents.** "Share pricing asks an ownership question, not a cash-withdrawal question" (`WHITEPAPER.md:484`), although "LP entry and exit pricing, however, deliberately share one exact terminal-NAV view" (`ACCOUNTING_SPEC.md:21-22`). For marked items the documents answer the two questions separately: "marked receivables affect ownership pricing but remain unavailable to fund redemptions until physically collected" (`SECURITY.md:862`), and "an entrant receives the same discount for inherited marked liabilities that an exiting LP bears, and receives credit only for collectible marked receivables" (`ACCOUNTING_SPEC.md:172-173`). `VpiReserveSiblings.t.sol` runs the same sequence, without VPI, on a trader's collectible loss (4,001.51) and unrealized profit (4,001.60), each turning into cash while LP2 is in the pool: the entrant gains 0.00 on both. With the marked price delta removed from the snapshot (`mutant-lens70.patch`, one line at `CfdEngineProtocolLens.sol:70`), they give +3,003.49 and -3,003.55, equal to D*X/(V_entry + D) within 2 atoms (asserted), with V_entry the valuation implied by LP2's shares (1,993,726.93). The reserve follows the same formula with V_entry = V - R, and its run is unchanged under the mutant (section G).

**Carry is the closest documented precedent, and the specification accepts a transfer there.** "LP ownership follows collection time. Indexed pending carry and retained `unsettledCarryAssets` are trader debt, not LP NAV receivables" (`ACCOUNTING_SPEC.md:731-732`); "This is a collection-time ownership policy, not a claim that timing cannot transfer part of old carry revenue to a new LP" (`ACCOUNTING_SPEC.md:739-740`). The contracts and documents distinguish the reserve from carry three ways:

- *Origin.* Pending carry is trader debt the pool has not received; the reserve is cash the pool paid out (step 1).
- *Return.* It comes back as "cash recovery" (`MarginClearinghouse.sol:883`), and in the isolated test no additional LP equity is created across a round (section D). The close does book it as claimant revenue (`CfdEngineSettlementSidecar.sol:604-612`), which is how the waterfall shares it with the entrant.
- *Stated policy.* For carry the specification states an ownership policy and accepts the transfer to new LPs. The cited justification for excluding the reserve addresses withdrawal availability (`WHITEPAPER.md:887-888`), but does not explicitly address the ownership effect of issuing new Junior shares while the reserve is outstanding, and we found no passage that accepts a transfer of the reserve to new LPs.

**The project's own tests of this property run with VPI off.** `test_DeltaNeutralZeroPnlCannotDiscountNewJuniorDeposits` asserts "Zero-PnL delta-neutral exposure must not let new LPs mint discounted junior shares" (`LpEntryExitNavParity.t.sol:113-142`) with vpiFactor 0. At the release vpiFactor its assertion still holds against the shares minted before any position opened, while a deposit with the book neutral mints 199 ppm more shares than right after the reserve returns, which is the reserve divided by pool assets (section C). `neutralMtmCannotCreateLpDepositWithdrawProfit` (`PerpValueConservationInvariant.t.sol:161-201`), also with vpiFactor 0, flags a violation only when the entrant ends above its deposit plus all pool inflow during its stay, as "only extraction beyond the attacker's contribution plus all contemporaneous pool inflow can have come from legacy claimant capital"; the returned reserve is such inflow and did come from legacy capital (step 1). At the release vpiFactor the scenario still passes. In our breakdown of the same calls, the entrant's gain beyond its share of the other inflow equals its share of the reserve within a cent, up to 103.50 of 448.90 (section I).

The lifetime clamp (`SECURITY.md:1443`; B's VPI nets to zero), the nonwithdrawable reserve (step 2) and the withdrawal floor (round 506) all hold as documented. The existing Junior still loses about three quarters of its capital to the attacker's deposits. We would welcome the team's view on whether this consequence for share ownership falls within the economic guarantee the protocol intends.

### 5. Known issues checked

- **AI scan, "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate" and "Temporary pool liquidity can reduce a skew-increasing close charge".** These concern changes in pool depth affecting a trader's close VPI. Here the transfer comes from Junior shares issued while the reserve is excluded from entry pricing; B's measured depth effect is 2.08 USDG per round (section D).
- **Pre-audit SC-H1, "Deposit Before Yield is Accrued Allows to Dilute Yield".** It concerns Senior yield held as `unpaidSeniorYield`, not the Junior ownership transfer that follows from the reserve's exclusion (`SECURITY.md:871-872`).
- **HIGH-1 (clock alignment) and MED-7 (older signed prices), listed on the program page.** The sequence depends on neither: in the timing replay every order executes at the first recorded tick after its commit, under the production oracle's parsing rules (Impact Details).
- **The `SECURITY.md` sections the program lists as known issues** (Known Limitations, Router and keeper limitations, live basket component selection, Arbitrum and Pyth clock alignment). None covers entry pricing while a reserve is outstanding, "VPI limitations" (`:1434-1444`) included. The exclusion itself is documented at `:871-872`, under "Exact symmetric LP NAV" (`:849`), which is not one of those sections; this report concerns the transfer of existing Junior LP value when new shares are issued during that exclusion.
- **Public sources.** On 5 November 2026 we searched the 134 pull requests (with reviews and comments), the issues and the commit history of `Plether-Fi/plether-core`, the competition repository (no pull requests, issues disabled) and the web, and on 12 November checked that nothing new had been published. We found none describing LP entry pricing while a VPI reserve is outstanding; issue #137 concerns the deployed risk parameters, which match those used here.

## Impact Details

The tests separate the mechanism from its economic impact: the isolated case demonstrates a transfer from existing Junior LPs to the entrant with no increase in total LP equity; under release parameters, the attacker realizes a net cash gain after trading fees while existing Junior LPs redeem less than in the control. These measured outcomes form the basis of the claimed direct-theft impact.

### Who gains and who loses

Main PoC (`VpiReserveEntryDiscount.t.sol`, release parameters, six-feed basket). The attacker's accounts are B, LP2 and the keeper, net of B's fees and price effects; "existing Junior" is the value of the existing LPs' Junior shares.

| rounds | attacker's accounts, attack / control | existing Junior, attack / control | attack - control, attacker / existing Junior | existing LPs redeem 1/10 of their shares, attack / control |
|---|---|---|---|---|
| 1 | +874.96 / -1,321.49 | -1,456.92 / +741.11 | +2,196.45 / -2,198.04 | 199,557.75 / 199,777.56 |
| 10 | +8,834.22 / -13,216.73 | -20,816.27 / +1,237.22 | +22,050.96 / -22,053.50 | 197,621.82 / 199,827.17 |
| 100 | +97,620.57 / -132,173.47 | -226,299.49 / +3,482.32 | +229,794.04 / -229,781.81 | 177,073.49 / 200,051.67 |

That tenth was worth 199,705.27 before the first round: the existing LPs lose deposited capital, and the attacker withdraws it as cash through LP2's redemptions. Senior and treasury end equal in both arms (asserted), T differs by at most 12.23, and all accounts reconcile within 1 atom (asserted to 0.01 USDG). Over 100 rounds B's VPI cancels exactly (-296,711.37 / +296,711.37), so the rounds create no VPI revenue.

**Until the withdrawal floor.** Repeated until the Junior withdrawal floor (25% of Senior, `HousePoolSeniorCapacityLib.sol:64`) stops LP2's full exit (round 506, after 2,318 hours of market time), the existing Junior falls from 1,997,052.79 to 511,827.97. After T closes and LP2's last shares are redeemed (2,319 hours):

| account | change (USDG) |
|---|---|
| existing Junior (value of their shares) | -1,482,140.48 |
| LP2, redeemed - deposited (cash) | +1,483,661.95 |
| B, fees, VPI and price | -654,077.43 |
| attacker's accounts (B + LP2 + keeper) | +829,685.92 |
| treasury (execution fees) | +616,184.38 |
| Senior | +42,909.45 |
| T | -6,639.27 |
| all accounts | 0 atoms |

### A depositor that places no order

`test_PassiveDepositor` (release parameters, flat price) separates the pricing defect from any trading by the attacker, which opens no position and places no order: LP2 sees a third-party trader B open a skew-reducing position on chain, requests a Junior deposit, and redeems after B's close; an unrelated address settles each epoch. B's positions open at random seconds, sized 20-100% of T and held 45 minutes to 6 hours (our distribution, not measured in production). The reserve was outstanding at all 12 activations: LP2 ends +19,113.04 (closed form 19,478.45) and the existing LPs -17,651.13, against +1,558.85 without LP2 (section D). With the same trade schedule, B scaled to T (section F), LP2 ends +2,538.98 at a 15% skew, +830.89 at 10% and -203.08 at 5%, where the Senior coupon exceeds its share; at 38%, +5,111.67, +1,217.97 and +127.27 with 500,000, 100,000 and 10,000 USDG.

The pending deposit can be cancelled until its epoch starts (`TrancheVault.cancelPendingDeposit`): with the depositor settling epochs itself, a position that closes before the boundary costs nothing (full refund), one that closes too close to the boundary to cancel costs 27.45, and one still open at the boundary yields +1,611.84 (section H). Across skews of 10-38% and B at 20-100% of T, a position still open at the boundary yields from +28.06 to +2,169.45, while a late close costs about 27.

The specification rules out one form of deposit-timing capture: "Revenue with no eligible share owner is routed to `unassignedAssets`; a new depositor cannot acquire it simply by being first to deposit" (`ACCOUNTING_SPEC.md:356-357`). Unlike unassigned revenue, the reserve originates from previously recognized pool assets: the existing LPs' value fell by exactly R when it was paid out (step 1). The isolated test shows that its subsequent recovery transfers value from existing Junior LPs to shares issued while the reserve was outstanding.

### Other variants

| variant | setting | attacker's accounts | source |
|---|---|---|---|
| attacker holds the skew itself | flat price, 100 rounds | +95,357.42 (control -134,424.38) | section A |
| the same, recorded ticks | 20 real boundaries | +20,887.64 (control -25,815.51) | section F |
| lower third-party skew | 17.5%, B at 35% of T | +343.76 | section A |
| | 15% | -69.71; -20.58 at the 0.25 bp confidence of the recorded ticks | section F |
| smaller deposit with its own orders | 750,000, B at 18% of T | +177.66 | section A |
| price moving, also while LP2 is in the pool | 100 rounds, three seeds | +92,032.59 / +115,688.53 / +117,661.90 | section A |
| busy market | 38%, 10 rounds | +6,682.19 (control -14,808.81) | section B |
| | 30%, B at 60% of T, 100 rounds | +50,861.58 (control -71,522.06) | section B |
| real boundaries (2-6 November 2026), recorded Hermes ticks | B held 6 s / 300 s | +18,089.73 / +16,585.97; attack - control +46,682.82 / +46,670.60 | section E |

Busy market: honest traders on both sides queued ahead of B, an LP rotating 100,000-300,000 through the Junior, an incumbent redeeming 2%, the same actions in both arms, with the same numbers of honest orders executed and failed (asserted). At 38%, attack minus control, every Junior holder loses: the existing Junior -16,324.43, the redeeming LP -3,769.51, the rotating LP -1,410.20 (section B).

### Limits

- Time: each round crosses an epoch boundary and a 1-hour cooldown, so nothing is atomic and LP2 carries price risk while in the pool.
- Only Pyth and USDG are mocks. The timing replay uses recorded Hermes ticks for EUR/USD and USD/JPY (71.2% of the basket weight), the other four feeds at their base price with a 2 bp confidence and no signature check; each run checks the production oracle's parsing rules on its 41 settlements and 42 executions. It replays the timing, not production.
- All results come from local Foundry runs at the competition commit; gas is not counted in any run.

## Recommendation

A direction to validate against the single entry/exit NAV (`ACCOUNTING_SPEC.md:21-22`), keeping the documented reason for the exclusion (`WHITEPAPER.md:887-888`), is to price Junior deposit activations with the outstanding VPI rebate reserves included and keep them out of redemption pricing and withdrawable cash, so entrants pay for the reserve they will share; we have not implemented it. `fix.patch`, the counterfactual of section 3, adds the reserves for entry and exit alike; in the PoC it leaves a residual attack - control of +302.41, +20.73 with zero oracle confidence, against +22,050.96 without it. Each reserve is counted once on the six recovery paths; it also pays an LP who exits during the reserve window 200,269.06 instead of 200,004.07 (section C), so we do not propose it as is.

## References

Code and documentation at the competition commit:
- [CfdEngineProtocolLens.sol#L70](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineProtocolLens.sol#L70)
- [HousePoolAccountingLib.sol#L57-L77](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L57-L77)
- [MarginClearinghouse.sol#L843-L884](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L843-L884)
- [CfdEngineSettlementSidecar.sol#L442-L470](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L442-L470)
- [CfdEngineSettlementSidecar.sol#L604-L612](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L604-L612)
- [CfdEngineSettlementSidecar.sol#L692](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L692)
- [OrderRouter.sol#L479](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L479)
- [HousePoolSeniorCapacityLib.sol#L64](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolSeniorCapacityLib.sol#L64)
- [LpEntryExitNavParity.t.sol#L113-L142](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/differential/LpEntryExitNavParity.t.sol#L113-L142)
- [PerpValueConservationInvariant.t.sol#L161-L201](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/invariant/properties/PerpValueConservationInvariant.t.sol#L161-L201)
- [SECURITY.md#L508](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L508)
- [SECURITY.md#L849-L872](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L849-L872)
- [SECURITY.md#L1434-L1444](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1434-L1444)
- [ACCOUNTING_SPEC.md#L21-L22](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L21-L22)
- [ACCOUNTING_SPEC.md#L160-L173](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L160-L173)
- [ACCOUNTING_SPEC.md#L356-L357](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L356-L357)
- [ACCOUNTING_SPEC.md#L731-L740](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L731-L740)
- [ACCOUNTING_SPEC.md#L1064-L1066](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L1064-L1066)
- [WHITEPAPER.md#L484-L487](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L484-L487)
- [WHITEPAPER.md#L887-L888](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L887-L888)
- [DEPLOYMENT.md#L123-L155](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L123-L155) (release parameters used by the PoC)


## Link to Proof of Concept

https://gist.github.com/Carrotstudiosoftwarehouse/9fa0f0c16a1cd84f3d12dafc7f9af0de

## Proof of Concept

Runnable Foundry tests in a secret gist (linked in the gist field of this submission and cloned in the Run block below): https://gist.github.com/Carrotstudiosoftwarehouse/9fa0f0c16a1cd84f3d12dafc7f9af0de

| file | purpose |
|---|---|
| `VpiReserveEntryDiscount.t.sol` | the PoC: attack and control from the same snapshot, and the run until the Junior floor |
| `VpiReserveEntryDiscountBusy.t.sol` | the same round in a busy market |
| `VpiReserveSupportingChecks.t.sol` | no attacker: the project's parity sequence with VPI on, six recovery paths, bank run |
| `VpiReserveOwnership.t.sol` | the closed form with every other LP-side flow removed, entrant risk, trader withdrawal, the depositor that places no order |
| `VpiReservePythTiming.t.sol`, `VpiReservePythReplayData.sol` | the round at real timestamps with recorded Hermes ticks, B held 6 to 300 seconds |
| `VpiReserveCancel.t.sol` | the depositor that places no order, with cancellation, settling epochs itself |
| `VpiReserveVendorScenario.t.sol` | the project's scenario `neutralMtmCannotCreateLpDepositWithdrawProfit` at its fixture's vpiFactor and at the release vpiFactor |
| `VpiReserveSiblings.t.sol`, `mutant-lens70.patch` | the same entry sequence on a marked receivable and a marked liability; the one-line control that removes the marked price delta from LP pricing |
| `fix.patch` | the counterfactual correction used to confirm the cause, and the matching update of the two test reference models |
| `outputs.txt` | the test output of every run cited above, copied without edits (amounts without thousands separators) |

Attached screenshots: `poc-10-rounds.png` (command 1 below), `closed-form.png` and `passive-depositor.png` (section D; one test each, output filtered with `sed`).

The PoC tests run in the repository's Foundry suite at the competition commit, with no change to any contract in scope; `fix.patch` and `mutant-lens70.patch` change source code only for the optional comparisons at the end of the Run block. Only Pyth and the settlement token are mocks. The setup reproduces the release configuration: risk parameters from `DEPLOYMENT.md`, Senior cap 80% / 40M, the six-feed basket of `script/DeployPerpsArbitrumSepolia.s.sol` installed through the owner's timelocked oracle rotation, adverse-confidence multiplier 2,500 bps, confidence cap 10 bps and a per-feed confidence of 1 bp.

**Steps (what `test_VpiReserveEntryDiscount` executes, all through public entry points):**
1. Existing LPs hold Junior 2,000,000 and Senior 2,000,000; T opens a 38% LONG (`OrderRouter.commitOrder` / `executeOrder`).
2. LP2 requests a Junior deposit of 6,000,000 for the next boundary e1 (`TrancheVault.requestDeposit`).
3. B commits an offsetting SHORT at e1-3 s, executed at e1-1 s: B receives the rebate, locked in its reserve (`MarginClearinghouse.sol:843`).
4. An unrelated address calls `OrderRouter.settleLpEpoch` after e1: LP2's deposit activates at a NAV without the reserve (`CfdEngineProtocolLens.sol:70`).
5. B closes at e1+3 s: the clawback returns the reserve to the pool (`CfdEngineSettlementSidecar.sol:692`).
6. After the 1-hour cooldown LP2 redeems everything at the next epoch.
7. Steps 2-6 repeat N times. T then closes, every account withdraws, all balances are reconciled, and the victims redeem one tenth of their Junior shares.

The control repeats the same orders at the same times but settles e1 after step 5. The round and the assertions, as in the gist:

```solidity
// VpiReserveEntryDiscount.t.sol:414-482, one round (attack = true: e1 settled while B's reserve is outstanding)
// _commit = router.commitOrder from the account; _execNext = router.executeOrder by KEEPER; _settle = router.settleLpEpoch by SETTLER
    function _round(uint256 lots, bool attack, uint256 i) internal returns (bool full, uint256 rid) {
        require(_size(B) == 0 && juniorVault.balanceOf(LP2) == 0, "round start");
        uint256 cash = settlementToken.balanceOf(LP2);
        vm.startPrank(LP2);
        settlementToken.approve(address(juniorVault), cash);
        uint256 did = juniorVault.requestDeposit(cash, LP2, LP2);
        vm.stopPrank();
        uint256 e1 = juniorVault.depositEpochStart(did);
        require(vm.getBlockTimestamp() + 3 < e1, "too close to the boundary");

        // B opens the offsetting SHORT and is executed BEFORE the epoch boundary
        vm.warp(e1 - 3);
        vm.roll(vm.getBlockNumber() + 1);
        uint256 bLots = (lots * bBps) / 10_000;
        _commit(B, CfdTypes.Side.SHORT, bLots * 100e18, _margin(bLots), false);
        vm.warp(e1 - 1);
        vm.roll(vm.getBlockNumber() + 1);
        _execNext(e1 - 2);
        acc_vpiOpenB += lastVpi;
        require(_size(B) == bLots * 100e18, "B not opened");
        uint256 reserve = clearinghouse.vpiRebateReserveAssets(B);
        require(reserve > 0, "no VPI rebate reserve");
        acc_reserve += reserve;

        // after the boundary: an unrelated address settles the epoch (attack) or nobody does yet (control)
        vm.warp(e1 + 1);
        vm.roll(vm.getBlockNumber() + 1);
        if (attack) _settle(e1);
        // B closes right away, or 30 minutes later when B is an ordinary third party (POC_HONEST_B=1)
        uint256 tClose = honestB ? e1 + 30 minutes : e1 + 1;
        if (tClose > e1 + 1) {
            vm.warp(tClose);
            vm.roll(vm.getBlockNumber() + 1);
        }
        _commit(B, CfdTypes.Side.SHORT, bLots * 100e18, 0, true);
        vm.warp(tClose + 2);
        vm.roll(vm.getBlockNumber() + 1);
        _execNext(tClose + 1);
        acc_vpiCloseB += lastVpi;
        require(_size(B) == 0, "B not closed");
        require(clearinghouse.vpiRebateReserveAssets(B) == 0, "reserve not cleared");
        if (!attack) {
            vm.warp(tClose + 3);
            vm.roll(vm.getBlockNumber() + 1);
            _settle(tClose + 3);
        }
        uint256 claimable = juniorVault.claimableDepositRequest(did, LP2);
        require(claimable == cash, "LP2 deposit not fully activated");
        vm.prank(LP2);
        juniorVault.claimDeposit(did, claimable, LP2, LP2);

        // cooldown, then LP2 redeems everything at the next epoch (the price may move meanwhile)
        _step(1 hours + 5);
        uint256 sh = juniorVault.balanceOf(LP2);
        vm.prank(LP2);
        rid = juniorVault.requestRedeem(sh, LP2, LP2);
        uint256 e2 = juniorVault.depositEpochStart(rid);
        if (pricePath) _pathStep(2 * i + 1);
        vm.warp(e2 + 1);
        vm.roll(vm.getBlockNumber() + 1);
        _settle(e2);
        full = juniorVault.pendingRedeemRequest(rid, LP2) == 0;
        uint256 out = juniorVault.claimableRedeemAssets(rid, LP2);
        if (out > 0) {
            vm.prank(LP2);
            juniorVault.withdraw(out, LP2, LP2);
        }
        acc_lp2 += int256(out) - int256(cash);
    }

// VpiReserveEntryDiscount.t.sol:238-249, the verdict after both arms
        assertLe(_abs(ctl.sum), 10_000, "control: accounts do not reconcile");
        assertLe(_abs(atk.sum), 10_000, "attack: accounts do not reconcile");
        assertLt(ctl.attacker, 0, "control: the same orders without the timing are not profitable");
        assertGt(atk.attacker, 0, "attack: the attacker ends with more USDG than it put in");
        assertLt(atk.victimJunior, ctl.victimJunior, "attack: victims' Junior value is lower than in the control");
        assertLt(atk.victimRedeemed, ctl.victimRedeemed, "attack: victims redeem less than in the control");
        assertApproxEqAbs(atk.treasury, ctl.treasury, 10_000, "fees are identical in both arms");
        // With POC_HONEST_B=1 the control settles 30 minutes later, so the Senior coupon checkpoint differs by cents.
        assertApproxEqAbs(
            atk.victimSenior, ctl.victimSenior, honestB ? 1e6 : 10_000, "Senior is identical in both arms"
        );
    }
```

**Run:**
```bash
git clone https://github.com/immunefi-team/audit-comp-plether && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
git clone https://gist.github.com/9fa0f0c16a1cd84f3d12dafc7f9af0de.git ../poc-gist
cp ../poc-gist/*.sol packages/perps/test/perps/
cd packages/perps

# 1. the PoC, 10 rounds (POC_ROUNDS=1 or POC_ROUNDS=100 for the other rows)
forge test --match-path test/perps/VpiReserveEntryDiscount.t.sol --match-contract '^VpiReserveEntryDiscountTest$' --match-test test_VpiReserveEntryDiscount -vv --gas-limit 9000000000000000000
# 2. busy market, 38% skew, 10 rounds
forge test --match-path test/perps/VpiReserveEntryDiscountBusy.t.sol --match-test test_BusyMarket -vv --gas-limit 9000000000000000000

# optional: until the Junior floor; supporting checks
forge test --match-path test/perps/VpiReserveEntryDiscount.t.sol --match-contract '^VpiReserveEntryDiscountTest$' --match-test test_UntilJuniorFloor -vv --gas-limit 9000000000000000000
forge test --match-path test/perps/VpiReserveSupportingChecks.t.sol --match-test 'test_(NeutralParity|RecoveryPath|ExitDuringReserveWindow|BankRunWhileReserveOutstanding)' -vv --gas-limit 9000000000000000000
# optional: closed form, entrant risk, depositor that places no order (section D); recorded timestamps and ticks, POC_HOLD = 6, 15, 30, 60, 120 or 300 (section E)
POC_NEUTRAL=1 forge test --match-path test/perps/VpiReserveOwnership.t.sol --match-contract '^VpiReserveOwnershipTest$' --match-test 'test_(Ownership|ReserveNotWithdrawableByTrader|EntrantExitsWhileReserveOutstanding|PassiveDepositor)' -vv --gas-limit 9000000000000000000
POC_HOLD=300 forge test --match-path test/perps/VpiReservePythTiming.t.sol --match-contract '^VpiReservePythTimingTest$' --match-test test_PythTiming -vv --gas-limit 9000000000000000000
# optional: the depositor with cancellation (section H); POC_SKEW_BPS, POC_CANCEL_B_BPS, POC_PRICE_PATH=1 with POC_SEED
forge test --match-path test/perps/VpiReserveCancel.t.sol --match-contract '^VpiReserveCancelTest$' --match-test test_PassiveDepositorWithCancel -vv --gas-limit 9000000000000000000
# optional: the project's scenario at vpiFactor 0 and 0.01 (section I)
forge test --match-path test/perps/VpiReserveVendorScenario.t.sol --match-contract '^VpiReserveVendorScenarioTest$' --match-test test_VendorScenarioAtReleaseVpi -vv --gas-limit 9000000000000000000
# optional: the marked receivable and liability, with the reserve's closed form in the same run (section G)
POC_NEUTRAL=1 forge test --match-path test/perps/VpiReserveSiblings.t.sol --match-contract '^VpiReserveSiblingsTest$' --match-test 'test_(Ownership|SiblingMarkedItems)' -vv --gas-limit 9000000000000000000
# the same with the one-line mutant (test_SiblingMarkedItems fails on "receivable: LP2 gain != 0"), then remove the mutant
(cd ../.. && git apply ../poc-gist/mutant-lens70.patch)
POC_NEUTRAL=1 forge test --match-path test/perps/VpiReserveSiblings.t.sol --match-contract '^VpiReserveSiblingsTest$' --match-test 'test_(Ownership|SiblingMarkedItems)' -vv --gas-limit 9000000000000000000
(cd ../.. && git apply -R ../poc-gist/mutant-lens70.patch)

# with the fix: apply it, then rerun 1, 2, the supporting checks, the closed-form command and the real-tick replay
(cd ../.. && git apply ../poc-gist/fix.patch)
# expected: the profit assertions of 1, 2 and the real-tick replay fail, the passive depositor fails on "existing LPs do
# not end lower", and so do the closed-form assertion and the two VPI-on parity checks; the floor, cancellation,
# siblings and project-scenario runs are for the original source only
# afterwards, back to the original source:
(cd ../.. && git apply -R ../poc-gist/fix.patch)
```
The other rows of the report use environment options (`POC_SKEW_BPS`, `POC_B_BPS`, `POC_CANCEL_B_BPS`, `POC_STAKE`, `POC_SELF_SKEW`, `POC_PRICE_PATH` with `POC_SEED`, `POC_NEUTRAL`, `POC_HOLD`, `POC_CONF_PPM` for the per-feed confidence, `POC_HONEST_B`, `POC_MAX_ROUNDS` for the floor run); each run in `outputs.txt` states its options and its expected result. Runs in which the attack is not profitable (for example the 5% skew of section F) end in [FAIL] on their impact assertion after printing their balances. The files that extend the PoC inherit its tests, so keep `--match-test`. The gas limit is needed above about 35 rounds; every command above passes it. Tested with forge 1.8.3 and solc 0.8.35.

**Expected output of 1 (excerpt):**
```
[PASS] test_VpiReserveEntryDiscount() (gas: 296408350)
  CONTROL (same orders, epoch settled after B's close)
    attacker net (B + LP2 + KEEPER, all withdrawn): -13216.73
    victims Junior / Senior / cash: 1237.22 549.35 0.00
    SUM of all accounts (atoms, must be ~0): 0
    victims redeem 1/10 of Junior shares: received / value of that tenth before: 199827.17 199705.27
  ATTACK (epoch settled while B's VPI rebate reserve is outstanding)
    attacker net (B + LP2 + KEEPER, all withdrawn): 8834.22
    victims Junior / Senior / cash: -20816.27 549.35 0.00
    SUM of all accounts (atoms, must be ~0): 0
    B VPI from protocol receipts, opens / closes (rebate < 0, clawback > 0): -28940.69 28940.69
    victims redeem 1/10 of Junior shares: received / value of that tenth before: 197621.82 199705.27
  ATTACK minus CONTROL (USDG)
    attacker       22050.96
    victim Junior  -22053.50
```
