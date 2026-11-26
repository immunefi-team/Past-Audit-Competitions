# \#98284 [SC-High] One atom of claim-net depth turns a triggered take-profit into a charge of 21.6 times its own notional and takes the owner's entire settlement balance

**Submitted on Nov 6th 2026 at 17:56:12 UTC by @skydev for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98284
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

**Severity:** Critical

**Impact:** Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

**Code:** `packages/perps/src` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (`libraries/CloseAccountingLib.sol`, `libraries/CfdEnginePlanLib.sol`, `CfdMath.sol`)

Why this impact:

- **A triggered take-profit takes the owner's whole settlement balance instead of paying it a gain.** In the attached PoC the owner holds a 12-lot (1,200-token) SHORT with a take-profit armed while the pool is healthy. When the take-profit finally crosses, the generated close is assessed an action charge of 3,296,780,471.208208 USDG, collects 51,173.869354 USDG, waives the uncollectible remainder, and leaves `clearinghouse.balanceAssets(victim)` at exactly `0`. The closed notional is 2,373.709668 USDG (1,200 tokens at the execution price 1.97809139), so the protocol collected **21.558605 times the notional of the position it was closing** and assessed 1,388,872 times it.
- **The charged amount is the owner's at-rest balance, not its exposure.** The loss is bounded by what the account holds, not by the size of the protected position. An account with a small protected position and a large balance loses the balance.
- **The owner does not choose the moment.** Triggering an armed protection is permissionless (`PositionProtectionBook`), so a third party decides when the owner's close is created, and it can wait for the pool state in which the charge consumes the account. The owner is explicitly asserted non-liquidatable at that price, so this is not a liquidation outcome.
- **The collected value becomes LP equity and an unprivileged attacker captures it pro rata.** In a matched control that differs in one stored field, the attacker's own tranche position ends 49,513.655380 USDG above the control branch on the same price path (its value is 20,000.254622 before either branch runs, and the control branch leaves it at zero). A share ladder shows the attacker still gains 20,919.670158 / 10,459.835078 / 4,183.934030 USDG while holding only 49.99 % / 24.99 % / 9.99 % of the tranches, and cash is actually withdrawn through the production asynchronous redemption flow.
- **Conservation holds to the atom**, so nothing here is a display artefact: the victim's debit equals the pool's gain plus the treasury's execution fee, and aggregate trader claims are byte-identical in both branches.
- **The fix is attached and its effect is measured.** The patch in the Recommendation caps a positive close charge at the envelope the governed skew ratio already implies. Installed in the same scenario, it leaves the protected close executing normally, the owner keeping 51,163.425140 USDG instead of zero, and the attacker's LP premium at exactly zero: the charge falls from 51,173.869461 to 10.444321 USDG.
- **Scope.** Finite close ceilings prevent the debit; this report does not claim a bypass of them. The exposed population is owners whose stored ceilings are `type(uint256).max`.

## Brief/Intro

`CloseAccountingLib.buildCloseState` contains a deliberate safeguard for closes that happen when the pool has no economic depth. Its purpose is stated in the code:

```solidity
// CloseAccountingLib.sol:105-110
// With no economic depth, keep protective exits live without creating a fresh charge or rebate.
// The lifetime clamp below still recovers any previously received negative VPI.
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
```

The guard is exact-zero only, and the curve behind it has no floor on its denominator and no cap relative to the closed notional. One atom of claim-net depth therefore inverts the safeguard's stated purpose: instead of a protective exit with no fresh charge, the owner gets a charge on a `1 / D` curve that exceeds its entire custody, and a terminal full close collects everything the account has before waiving the remainder.

Because protections are triggered permissionlessly, the party that decides whether the owner's close is created on the zero side or the one-atom side of that boundary is not the owner. The result is a transfer of the owner's at-rest settlement balance to HousePool, where any incumbent LP, including the attacker, owns it pro rata.

This is the opposite tail of the SCAN-03/04 remediation. PR #118, "fix(perps): use net risk depth and fingerprint calendar changes" (`6d6622f2871dbdfc4904c20da2e1f7662f5763b6`, 2026-10-15), replaced gross pool assets with claim-net depth and, in its own words, "Subtract outstanding trader claims consistently in carry, skew limits, VPI, previews, and maximum-open quotes. Checkpoint carry before claim mutations and define zero-depth admission and close behavior." Before that change there was no claim-net `0 -> epsilon` boundary to land on, and the exact-zero close rule did not exist. The report is therefore filed under the program's rule that a report on the same root cause as a fixed or acknowledged finding is valid where it shows the fix "introduces a new bug".

## Vulnerability Details

### 1. What the charge does at small depth

`CfdMath.calculateVPI` (`CfdMath.sol:150-161`, via `tryCalculateVPI` `:164-180`) prices the skew transition as the difference of two skew costs, each proportional to `1 / depth`. There is no lower bound on `depthAssets` other than the exact-zero branch above it, and no bound on the result relative to the notional being closed.

| quantity | value in the main PoC |
|---|---|
| claim-net depth used for the curve | 94 atoms (0.000094 USDG) |
| closed notional | 2,373.709668 USDG |
| assessed action charge | 3,296,780,471.208208 USDG |
| collected from the owner | 51,173.869354 USDG |
| of which: the position's withheld price gain | 1,173.709656 USDG |
| of which: the owner's settlement cash | 50,000.159698 USDG |
| waived remainder | 3,296,729,297.338854 USDG |
| owner's settlement balance afterwards | 0 |

The owner armed a take-profit, so the first line of that split is the gain the protection existed to realise, and the second is the rest of the account.

A terminal full close consumes spendable action reserve, free settlement, safely released pledge and committed margin, and waives only what it cannot reach (`CfdEnginePlanLib.sol:1093-1133`). The waiver keeps the close alive; it does not keep the owner's money.

A separate pure test (`test_D0VsD1FreshVpiDiscontinuity`, 2,807 gas, no state) isolates the step: the same skew transition yields a fresh VPI of zero at `D = 0` and a charge of over a billion USDG at `D = 1` atom.

### 2. Claim-net depth is not the pool's liquidity

`D` is `max(HousePool.totalAssets() - totalTraderClaimBalanceAssets, 0)` (`libraries/CfdEnginePlanLib.sol:1339-1341` via `PositionRiskAccountingLib.computeNetPoolDepthAssets`). It is a bookkeeping residual, not a cash balance. In the main PoC, at the moment the owner's close is priced:

- live pool backing immediately before the close is 9,279.999946 USDG (aggregate claims of 9,280.000000 less the printed 54-atom deficit), and raw token custody is at least that;
- the planner then projects 148 atoms of collectible carry into its own snapshot, giving a `poolAssets` input of 9,280.000094 USDG;
- the denominator handed to the curve = 0.000094 USDG.

The owner is charged as if the pool were empty while the pool holds about 9.28 thousand USDG of real assets. The matched control prints the same identity on live state: accounted assets, real token custody and pool assets are all 9,280.080012 USDG at child time, against a depth of 0.080012 USDG.

### 3. A third party decides when the owner's close happens

A protection trigger is permissionless, and the trigger queues the close through the Router's own path (`OrderRouter.sol:440-470`, `_recordCommittedOrder` at `:459`) rather than through an ordinary trader commit. Consequences used here:

- the owner cannot choose the execution moment, and cannot cancel a queued close (documented);
- the attempt carries the ceilings stored on the protection, copied unchanged;
- the attempt is appended without the carry checkpoint that an ordinary close commit performs, so the owner's pending carry is still unrealized when the close is planned.

### 4. Three measured ways the close lands just above zero

The defect is the discontinuity and the missing cap; the path to a small positive `D` is incidental. Three are measured, from the narrowest to the widest:

1. **The owner's own carry (94 atoms).** Aggregate claims exceed pool assets by 54 atoms, so claim-net depth is exactly zero when the protection is triggered and the guard would apply. Inside `planClose` the owner's pending carry is realized first (`CfdEnginePlanLib.sol:815`) and added to pool assets (`:638-639`), and only afterwards is the depth sampled (`:1339-1341`). 148 atoms of the owner's own carry turn `D = 0` into `D = 94` within one plan, and the guard no longer applies.
2. **A queued partial close (80,012 atoms, 0.080012 USDG).** The matched control reaches this depth with an ordinary tuner order ahead of the owner's close in the FIFO. The ladder rungs run the same construction at 107,905 atoms (0.107905 USDG), the share transfer's own carry checkpoint accounting for the difference.
3. **An ordinary, non-knife-edge depth.** `test_attackRealisticFortyCentDepthStillConsumesEntireVictimSettlement` passes in the attached run and asserts that a depth of the order of 0.4 USDG still consumes the owner's entire settlement; `test_probeRealisticFortyCentDepthWithLongPartialClose` demonstrates that a sub-dollar depth of 0.250006 USDG is reachable through ordinary public tuning (printed). Neither figure is offered as a drain threshold, and the exact depth of the 0.4 USDG variant is asserted inside that test rather than printed in the log. The point is only that the transfer is not a single-atom coincidence.

Note on path 1: the program's known-issue text for zero economic depth describes net depth as measured "after projected carry collection", so the post-carry evaluation is documented and this report does not claim a bypass of the guard's definition. The point is narrower and survives that text: for a protection child the guard's promise is unreachable whenever the owner has collectible (realized) carry larger than the deficit, which is precisely the class of closes the guard was written to protect.

### 5. PR #118 routed the same claim-net depth through a bounded carry ratio

PR #118 introduced `computeNetPoolDepthAssets` and fed its output to both denominator-sensitive consumers: carry utilisation and close VPI. The carry ratio it reused was already saturated; close VPI was not.

```solidity
// libraries/PositionRiskAccountingLib.sol:61-75, computeBorrowUtilizationBps
if (borrowBaseAssets == 0) {
    return 0;
}
if (poolAssets == 0) {
    return UTILIZATION_BPS;                 // zero depth is full utilisation, not a skip
}
utilizationBps = (borrowBaseAssets * UTILIZATION_BPS) / poolAssets;
if (utilizationBps > UTILIZATION_BPS) {
    utilizationBps = UTILIZATION_BPS;       // and the ratio itself is capped
}
```

For an empty borrow side the ratio is zero, and for a positive one it saturates, so in the carry path a vanishing denominator is the continuous limit of a bounded function rather than a discontinuity. To be precise about history: that cap and both zero branches predate PR #118, whose only change to this function was to re-point its parameter documentation at "Economic pool depth after reserving outstanding trader claims". The remediation therefore routed its new claim-net depth into an already-bounded, zero-continuous consumer, and separately routed the same depth into the close VPI curve, where it added the exact-zero exception around an output with no cap.

The remediation's own new test file is the clearest statement of what it considered at that boundary. `test/perps/accounting/NetEconomicDepth.t.sol`, added by PR #118, covers claims at or above assets from three sides: `testFuzz_ClaimsAtOrAboveAssetsCapCarryAndLeaveEmptySideIdle` fixes carry as capped, `testFuzz_ClaimsAtOrAboveAssetsRejectOpenAndReturnZeroQuote` fixes opens as rejected, and `testFuzz_ZeroNetDepthCloseDoesNotPayFreshVpiRebate` together with `test_ZeroNetDepthRouterClosePreservesLifetimeRebateClawback` fix the close at exactly zero depth, in the rebate direction. No case in that file, or anywhere else in the suite, covers a close one atom above zero paying an unbounded charge. The boundary this report is about is the one quadrant the remediation left untested, and it is the only one of the four in which the output is unbounded.

### 6. Root cause

Three properties combine. Removing either of the first two makes the charge safe; removing the third would leave the formula unsafe and merely make the loss self-inflicted rather than imposed:

1. the zero-depth safeguard for protective exits is evaluated as an exact-zero test, so it offers no margin;
2. the close VPI curve has no minimum denominator and the collectible action charge has no cap relative to the closed notional;
3. the close is created by a third party at a time the owner does not choose, with ceilings the owner set long before, so the owner cannot react to the state in which its close will be priced.

## Impact Details

All figures below are printed by the attached tests or are differences of printed values. The runs are a clean build (`--force`) at the repository's default profile, with no compiler-setting or optimizer override: solc 0.8.35, optimizer on, 200 runs, via-IR, 198 files compiled. The only environment variable used is `FOUNDRY_SPARSE_MODE`, which selects sparse compilation and changes no setting that affects produced bytecode.

### Main attack and its paired control

`C08CarryProtectionProbe`, 2/2 PASS.

| | attack (ceilings at `type(uint256).max`) | control (`maxActionChargeAssets = 100 USDG`) |
|---|---|---|
| deficit before the trigger | 54 atoms | 54 atoms |
| owner's carry realized in the plan | 148 atoms | rolled back |
| depth used for the curve | 94 atoms | n/a, rejected before settlement |
| assessed action charge | 3,296,780,471.208208 USDG | rejected |
| collected from the owner | 51,173.869354 USDG | 0 |
| owner's position | closed | unchanged |
| owner's settlement balance | 0 | unchanged |
| protection | consumed | relatched |

The control is the same scenario with one stored field changed. It proves the charge is the mechanism and that finite ceilings are a complete defence. A second control (`test_controlFiniteGrossDebitBoundRejectsBeforeSettlement`) rejects on the gross-debit ceiling instead.

### Matched control: what the event moves, with everything else held fixed

`C08MatchedControlProbe`, 1/1 PASS. One EVM snapshot, identical setup, time, block, price path, FIFO ids, token mint and share balances; the only difference between branches is the stored `maxActionChargeAssets`.

| | attack | control | delta |
|---|---|---|---|
| HousePool assets (= accounted = real custody) | 59,279.290375 | 9,280.080012 | +49,999.210363 |
| aggregate trader claims | 9,280.000000 | 9,280.000000 | 0 |
| treasury | — | — | +0.949483 |
| keeper accounts' value afterwards | 1,002.040000 | 1,002.040000 | 0 |
| attacker's LP value | 49,513.655380 | 0 | +49,513.655380 |
| independent LP's value | 4.949408 | 0 | +4.949408 |

- **Conservation:** the owner's cash debit of 50,000.159846 USDG equals the pool's gain of 49,999.210363 plus the treasury's 0.949483 execution fee, exactly. This is the cash leg only; the total recovered in the main attack additionally withholds the position's price gain, which never reaches the owner's balance and so does not appear as a debit. The treasury figure is itself a check on the closed notional: 0.949483 is the 4 bps execution fee on 2,373.709668 USDG.
- **Claims are byte-identical in both branches.** The owner's money did not settle a claim, repay a liability, or discharge any debt of the owner's own. It became LP equity and nothing else.
- Keeper compensation is identical in both branches: the keeper accounts hold 1,001.040000 USDG before either branch and 1,002.040000 USDG after each of them, so the same 1.000000 USDG of bounty is paid on both sides and the difference between the branches is the charge and only the charge.

### The attacker does not need to own the pool

`C08ShareLadderProbe`, 4/4 PASS. Existing tranche shares are redistributed before the common snapshot; pool assets, supplies, oracle path, branch inputs and total LP ownership are unchanged, so the attacker's share is the only variable. Every rung shares one depth (0.107905 USDG), one owner debit (42,325.049969 USDG) and identical claims (9,280.000000 in both branches).

| attacker's share | attacker LP, start | attacker LP, attack | attacker LP, control | gain vs control | as % of the owner's loss |
|---|---|---|---|---|---|
| 49.99 % | 9,999.999995 | 20,919.670158 | 0 | +20,919.670158 | 49.426 % |
| 24.99 % | 4,999.999997 | 10,459.835078 | 0 | +10,459.835078 | 24.713 % |
| 9.99 % | 1,999.999998 | 4,183.934030 | 0 | +4,183.934030 | 9.885 % |

The attacker's gain tracks its share of the tranches almost exactly, which is the expected signature of pro-rata LP revenue. Conservation again holds: pool gain 42,324.100486 plus treasury 0.949483 equals the owner's debit of 42,325.049969.

The attacker is profitable at every rung and pays only gas plus the cost of positioning, so the griefing standard ("applies to High or Critical findings where the attacker gains no profit") does not apply.

### Cash leaves the protocol

`C08CaptureProbe`, 8/8 PASS, measures the exit path rather than only the entitlement: after the charge, incumbent tranche owners hold 47,895.262918 USDG of value and 42,782.019880 USDG is actually received through the production asynchronous redemption flow. In the same file, `test_attackUnlimitedBoundsConsumeEntireVictimSettlement` records an assessed 559,689,841.487069 USDG against 49,961.759846 USDG recovered and a pool delta of 50,000.019844 USDG with claims unchanged at 9,280.000000.

### Preconditions and limits, stated plainly

- **The pool must already be claims-heavy.** A small `D` means aggregate claims have consumed almost all pool assets, which outside degraded mode the solvency checks prevent. In the PoC that precondition is constructed through public opens and closes together with mocked but internally valid price updates along the path 1.00 -> 1.95 -> 0.20. The PoC does not establish that a production market would traverse such a path, and it does not show that an attacker could force it. In production this is an opportunistic attack on a state the attacker waits for rather than creates.
- **The attacker cannot author prices.** Pyth updates are mocked because the program allows mocking out-of-scope components, not because the attacker controls them. The trigger happens at the first genuine crossing of a take-profit armed long before, and no step depends on choosing between ticks.
- **Position-size coupling.** The skew cap ties notional to depth at open time, so an owner of this size must have opened while depth was healthy, which is what the PoC does.
- **Capture needs an existing LP position.** New deposits are blocked once the pool is degraded, so the attacker must already hold tranche shares; the ladder shows 9.99 % is enough. The owner's loss, however, is complete and permissionless before any of that.
- **Cash-out timing.** The owner's loss and the LP entitlement happen with no trusted action. Realising the entitlement as cash follows the ordinary degraded-mode recovery, in which the owner of the protocol clears degraded mode after the inflow restores solvency. That is an operational dependency of the attacker's monetisation, not of the owner's loss.
- **The size of the loss varies with the variant.** The primary carry variant empties the account (balance exactly `0`); the ladder construction debits 42,325.049969 USDG, which is a material but partial loss for that owner. The report claims the complete loss only for the variant that asserts it.
- **Not claimed:** no bypass of finite ceilings, no control of oracle data, no arbitrary victim, no instant payment to the triggering account, and no High fallback (the High impacts cover unclaimed yield and freezes, not settlement principal).

<details>
<summary>Relation to the listed known issues, the out-of-scope list, and the documentation</summary>

Each rule below is quoted from the program's reviewed scope text, with its conditions, followed by the delta.

| item | what it says | delta |
|---|---|---|
| SCAN-I17, pinned to `CloseAccountingLib.sol:107-111` | "Close VPI is computed from a single execution-time net pool depth, so a **higher** depth at execution **lowers** the positive VPI charge on a skew-increasing close." Condition: "Report concerns changing HousePool depth around a close **to reduce its positive VPI action charge**." | Opposite direction and a different actor. This report is about an unbounded positive charge imposed on a third party's protected close, not about an actor reducing its own charge. The narrow fixes also differ: a minimum denominator or a cap on the collectible charge relative to notional stops this and does nothing for depth manipulation at ordinary depths. |
| SCAN-I18, pinned to `:103-118` | lower depth magnifies the **negative** VPI of a skew-reducing close; "the rebate is clamped to the position's proportional lifetime accrued VPI". Condition: "reducing HousePool depth around a close **to enlarge its VPI rebate**." | Wrong sign. Here the signed VPI is positive, the lifetime clamp never binds, and no rebate is paid. |
| Zero economic depth, pinned to `:105-118` | "When net depth is zero after projected carry collection, every open or increase is rejected. **Closes at zero depth skip the VPI curve** but still apply the lifetime clamp." Condition: "Report concerns opens being rejected, or **closes paying no fresh VPI** while the lifetime clamp still applies." | This report is the complement: a close paying an unbounded fresh VPI just above zero. The quoted text is also the reason the carry ordering is presented as a mechanism and not as a bypass: depth is documented as post-carry, and the finding is that the guard built on it has no margin. |
| Full-close waiver, pinned to `CfdEnginePlanLib.sol:1130-1133` | a charge the account cannot fund is waived instead of blocking the close. Condition: "Impact is limited to the **uncollected** charge or written-off loss that this design deliberately forgoes." | The impact here is the collected 51,173.869354 USDG, not the waived remainder. The waiver is what keeps the close alive after the account has been emptied. |
| Lifetime VPI clamp | "On close, VPI is clamped so the closed portion's lifetime VPI cannot go negative." | Accepted and unused: the charge is positive, so the clamp does not engage. The clamp's comment does describe the adjacent attack shape ("an attacker opens at high depth, reduces pool depth, then closes"), which shows depth collapse around a close was considered, in the rebate direction only. |
| This researcher's report #97695, and the earlier #97670 it was matched to | Root cause as the triage summary worded it: "a skew-increasing open's positive VPI charge is booked as Junior NAV, and the later clamped close rebate refunds it from pool cash, so a trader-LP can exit between the charge and the refund". | Different defect, different loser, and the fixes do not overlap. There the charge is correctly sized and the defect is in LP exit pricing; the party that loses is the set of passive LPs, and the attacker is the same account that paid the charge. Here the charge itself is mis-sized by a factor of about 1,388,872 against the closed notional; the party that loses is an unrelated account whose close a third party triggered; and LPs, including passive ones, gain. The one-fix test is decisive: treating outstanding positive VPI as a refundable liability in LP pricing, which is the minimal fix for #97695 and #97670, leaves this finding untouched, because a close priced at one atom of depth still assesses billions and still empties the account. |
| Out of scope: "Lack of liquidity impacts" | program out-of-scope list, Smart Contract Specific. | Disclosed, and the response below is this report's argument rather than a settled reading. On our reading the exclusion addresses impacts that consist of illiquidity: an exit that cannot be funded, a liquidation without a counterparty, slippage. Here the close succeeds, the pool holds about 9,280 USDG of real custody at the moment of pricing, the denominator used is 0.000094 USDG, and the loss is an active collection produced by an internal formula whose output (1,388,872 times the closed notional) has no liquidity interpretation. The project also wrote a safeguard for the illiquid case, quoted at the top of this report, and a defect in that safeguard is a code defect. Note also that the listed known issue addresses behaviour at zero depth, not just above it. |
| Out of scope: "Impacts requiring basic economic and governance attacks" | program out-of-scope list. | No market power is required. The attacker holds ordinary positions and tranche shares, opens and closes through public entrypoints, and does not move the oracle. |
| MED-30, SCAN-L09, "unpaid protection-attempt cleanup" (all pinned to `PositionProtectionBook.sol`) | frozen-market triggers; the trigger bounty is paid at trigger; cleanup and retries are unpaid, and liveness depends on keepers. | None concerns the amount charged to a triggered close. This report does not concern keeper incentives, retry liveness, frozen triggers, or a protection left latched. |
| HIGH-1 clock alignment | commit time may trail a published Pyth price. | Not used. The trigger is at the first genuine crossing of a threshold armed long before, and no step depends on choosing between ticks. |
| `SECURITY.md` VPI limitations | document that VPI depth is canonical assets net of trader claims and that closes at zero depth create no fresh VPI. | Both accepted and reproduced exactly. Neither passage states that one atom above zero produces a charge larger than the account, nor that a third party may choose which side of that boundary another account's close falls on. |
| `CloseFinancialLimits` NatSpec, `perps-aa-client` README | call the four values trader-authorised ceilings and `type(uint256).max` an explicit unlimited allowance. | Acknowledged, and the reason finite ceilings are presented here as a complete defence. A maximum-loss ceiling is a limit on what a close may cost under the protocol's pricing; it is not consent for a charge of 21.558605 times the closed notional assessed because a third party chose the moment. The loss is also unrelated to the authorised exposure: it is bounded by the account's balance, not by the protected position. |
| Reachability context (out of scope, cited for exposure only) | `plether-app` writes `type(uint256).max` into all four protection close limits for every take-profit and stop-loss ticket and exposes no amend action; on Arbitrum Sepolia the deployed `PositionProtectionBook` at `0x81e3b03cf29865b6c0f85925839567e2bc61073d` held 13 protection records, at least two still armed, every observed record with all four limits at `type(uint256).max` (read-only `eth_call`, blocks 325430302 and 326323894). | The frontend is out of scope and is not the root cause; this is cited only to show that the exposed configuration is the one that exists in practice. Take-profit and stop-loss are listed as active at launch on the program page. |

</details>

## Recommendation

The fix below is attached as a patch, and its effect is measured rather than asserted. With it installed, this attack collects 10.444321 USDG instead of 51,173.869461, the attacker's LP premium falls to exactly zero, and the protected close still executes normally.

**Cap a positive fresh close VPI at the envelope the skew cap already implies.** For admitted skew ratios no greater than `maxSkewRatio`, the quadratic curve's marginal charge per unit of closed notional is bounded by `vpiFactor * maxSkewRatio`. Capping a positive close charge at `closedNotional * vpiFactor * maxSkewRatio` therefore restores a bound the configuration already expresses, instead of introducing a new constant to argue about. At the release parameters (`vpiFactor = 0.01e18`, `maxSkewRatio = 0.4e18`) it is 40 bps of closed notional. Two files, and the cap applies only when the signed delta is positive, so rebates, the lifetime clamp and the zero-depth branch are untouched:

```diff
diff --git a/packages/perps/src/libraries/CfdEnginePlanLib.sol b/packages/perps/src/libraries/CfdEnginePlanLib.sol
index b576415e..5f7a6360 100644
--- a/packages/perps/src/libraries/CfdEnginePlanLib.sol
+++ b/packages/perps/src/libraries/CfdEnginePlanLib.sol
@@ -1340,6 +1340,7 @@ library CfdEnginePlanLib {
                     snap.poolAssets, snap.totalTraderClaimBalanceAssets
                 ),
                 vpiFactor: snap.riskParams.vpiFactor,
+                maxSkewRatio: snap.riskParams.maxSkewRatio,
                 frozenCloseSpreadBps: snap.frozenCloseSpreadBps,
                 oracleFrozen: snap.oracleFrozen,
                 executionFeeBps: snap.executionFeeBps
diff --git a/packages/perps/src/libraries/CloseAccountingLib.sol b/packages/perps/src/libraries/CloseAccountingLib.sol
index 2e05f596..455a5a41 100644
--- a/packages/perps/src/libraries/CloseAccountingLib.sol
+++ b/packages/perps/src/libraries/CloseAccountingLib.sol
@@ -8,8 +8,8 @@ import {CfdTypes} from "@plether/perps/CfdTypes.sol";
 /// @title CloseAccountingLib
 /// @notice Calculates the proportional position state and signed economic settlement for a close.
 /// @dev settlement token fields use 6 decimals, prices use 8 decimals, sizes use 18 decimals, basis points use a 10,000
-///      denominator, and `vpiFactor` uses 18-decimal WAD precision. Multiplication followed by integer division
-///      rounds down for unsigned values and toward zero for signed values.
+///      denominator, and `vpiFactor`/`maxSkewRatio` use 18-decimal WAD precision. Multiplication followed by integer
+///      division rounds down for unsigned values and toward zero for signed values.
 library CloseAccountingLib {
 
     /// @notice Inputs needed to value a full or partial close.
@@ -22,6 +22,7 @@ library CloseAccountingLib {
     /// @param postSkewAssets Absolute market skew immediately after the close.
     /// @param poolDepthAssets Pool depth used by the VPI curve.
     /// @param vpiFactor VPI impact factor, scaled by 1e18.
+    /// @param maxSkewRatio Maximum admitted skew divided by pool depth, scaled by 1e18.
     /// @param frozenCloseSpreadBps Additional close spread in basis points when the oracle is frozen.
     /// @param oracleFrozen Whether to charge the frozen-market spread.
     /// @param executionFeeBps Execution fee rate applied to closed notional, in basis points.
@@ -35,6 +36,7 @@ library CloseAccountingLib {
         uint256 postSkewAssets;
         uint256 poolDepthAssets;
         uint256 vpiFactor;
+        uint256 maxSkewRatio;
         uint256 frozenCloseSpreadBps;
         bool oracleFrozen;
         uint256 executionFeeBps;
@@ -102,6 +104,7 @@ library CloseAccountingLib {
 
         state.proportionalAccrualAssets =
             (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);
+        uint256 notionalAssets = (inputs.sizeDelta * inputs.oraclePrice) / CfdMath.ASSETS_TO_TOKEN_SCALE;
         // With no economic depth, keep protective exits live without creating a fresh charge or rebate.
         // The lifetime clamp below still recovers any previously received negative VPI.
         if (inputs.poolDepthAssets != 0) {
@@ -109,6 +112,16 @@ library CloseAccountingLib {
                 inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
             );
         }
+        // For admitted skew ratios no greater than m, the quadratic VPI curve's marginal charge is bounded by
+        // k * m * closed notional. Preserve that governed envelope when claim-net depth becomes vanishingly small.
+        if (state.vpiDeltaAssets > 0) {
+            uint256 maxPositiveVpiAssets = Math.mulDiv(
+                Math.mulDiv(notionalAssets, inputs.vpiFactor, CfdMath.WAD), inputs.maxSkewRatio, CfdMath.WAD
+            );
+            if (uint256(state.vpiDeltaAssets) > maxPositiveVpiAssets) {
+                state.vpiDeltaAssets = int256(maxPositiveVpiAssets);
+            }
+        }
         // Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
         // where an attacker opens at high depth, reduces pool depth, then closes to extract a
         // net-negative lifetime VPI rebate. This rule is identical in live and oracle-frozen markets; frozen-market
@@ -117,7 +130,6 @@ library CloseAccountingLib {
             state.vpiDeltaAssets = -state.proportionalAccrualAssets;
         }
 
-        uint256 notionalAssets = (inputs.sizeDelta * inputs.oraclePrice) / CfdMath.ASSETS_TO_TOKEN_SCALE; state.executionFeeAssets = (notionalAssets * inputs.executionFeeBps) / 10_000; if (inputs.oracleFrozen) { state.frozenSpreadAssets = (notionalAssets * inputs.frozenCloseSpreadBps) / 10_000;
```

### The patch installed and measured

`C08FixControlProbe` (1/1 PASS) runs three branches. A zero-ceiling branch where the close is rejected is the no-close control. The vulnerable branch replays the public bootstrap, the trigger and both FIFO commitments, and takes a second snapshot immediately before the batch that executes the tuner and then the protected child. The patched branch reverts to that exact pending state, etches only the separately compiled stateless planner runtime at `engine.planner()`, and executes the same queued order ids at the same timestamp, block and execution mark. Both the policy evaluator and the Engine's planning reader resolve that address, so one replacement reaches assessment and application alike, and a terminal `Executed` result also proves the two agree.

| | vulnerable | patched | no-close control |
|---|---|---|---|
| assessed charge | 58,654.773938 | 10.444321 | rejected by the zero ceiling |
| recovered charge | 51,173.869461 | 10.444321 | 0 |
| waived | 7,480.904477 | 0 | 0 |
| owner's clearinghouse cash | 0 | 50,000.159805 | 50,000.159846 |
| owner's total value | 0 | 51,163.425140 | position still open |
| attacker's LP value | 49,513.655380 | 0 | 0 |
| independent LP's value | 4.949408 | 0 | 0 |

The patched charge is the cap plus the ordinary fee: 9.494838 of capped VPI on a 2,373.709668 notional, plus 0.949483 of execution fee, giving 10.444321 with nothing waived. Claim-net depth at child time is identical in both branches at 0.080012 USDG, so the fix changes the price and not the state it is priced from.

**The flow still works with the patch.** The test asserts the patched protection is `Executed`, its child's lifecycle status, terminal status and terminal reason are all `Executed`, and the owner's position is closed. It does not revert and does not relatch. The owner keeps 51,163.425140 USDG where the unpatched run leaves zero, losing 41 settlement atoms of cash to the capped charge. This matters for reading the defect: a bound that leaves every mechanism running and removes only the extraction is a missing bound, not a design decision.

Alternatives, if the project prefers a different shape:

- **A positive floor on the denominator**, replacing the exact-zero test, so the guarantee stated at `CloseAccountingLib.sol:105` degrades continuously instead of inverting one atom away.
- **A manipulation-resistant reference depth** for close pricing, for example the depth at the start of the epoch or a time-weighted value, so that an order immediately ahead of a third-party-triggered close cannot set the denominator.
- Not sufficient on its own: snapshotting pre-carry depth for the guard. It addresses the narrowest measured path and leaves the 0.080012 and sub-dollar depths intact.
- Independently of the choice, make finite ceilings the canonical integration default and surface them, and add regressions at `D = 0`, one atom above it, the chosen bound's boundary, and a permissionlessly triggered protection under both finite and unlimited ceilings.

## References

Repository links are at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

- **The guard and its stated purpose:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L105-L110
- **The unbounded curve:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdMath.sol#L150-L180
- **Carry realised before depth is sampled:**
  - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L815 - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L614-L643 - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1332-L1346
- **Terminal collection and waiver:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1093-L1133
- **Protection child recorded through the Router:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L440-L470
- **Release parameters:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md?plain=1#L123-L156
- **The same divisor, capped, in the carry path:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/PositionRiskAccountingLib.sol#L55-L75
- **Documentation:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1434-L1444
- **PR #118, the SCAN-03/04 remediation that introduced claim-net depth and the exact-zero exception:**
  - https://github.com/Plether-Fi/plether-core/commit/6d6622f2871dbdfc4904c20da2e1f7662f5763b6 - https://github.com/Plether-Fi/plether-core/pull/118
- **AI scan report (SCAN-I17, SCAN-I18, SCAN-03/04):** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md

## Link to Proof of Concept

https://gist.github.com/Skydev0h/0c9780e04f3bd004eec0fbb127b5e915

## Proof of Concept

## Proof of Concept

**Environment.** The tests run in the repository's Foundry suite at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, which is one of the two allowed environments. No file in `packages/perps/src` is modified (`diff -rq` against a pristine clone of the frozen commit is empty).

- **Files.** Five new files under `packages/perps/test/poc/` (all in the Gist cloned below; the three decisive ones are inlined at the end):

  | file | lines | sha256 | role | |---|---|---|---| | `C08CarryProtection.t.sol` | 140 | `8f107708e12d7c2faec27ac3c9550a56921e0952a9deb2fb78eb1825653df658` | main attack and its finite-ceiling control | | `C08MatchedControl.t.sol` | 427 | `b027a7e2e54a88597fc664a61c56ebb5f827e15447f421f40bbf88c3652d95a1` | one-snapshot matched control, LP and conservation accounting | | `C08FixControl.t.sol` | 263 | `e8ebcf759162465b70bb9533baa99f8c045ba7457e8b1a7b69bb563da68e7554` | installs the recommended patch and measures its effect | | `C08FixedPlannerRuntime.hex` | — | `a760cbc61ab6a4eb1c81118b07d87ea05b5a446ae39542578d6ebea8896dbeed` | the patched planner runtime, 24,002 deployed bytes, codehash `0xff118a15226081ea3af3c5a9e4495598fe57b9bb6aa87420eb045254e8330218` | | `c08-vpi-cap.diff` | — | `8a46418a9431cf24bc97dbdc0f7513da1d4fcdac478140882d67ef80a88bd172` | the two-file patch the hex was built from | | `C08ShareLadder.t.sol` | 191 | `b7f046684dc0f684a6be8a2670397d81e0b460052844b148e697a2dd011f9e01` | the same control at 49.99 / 24.99 / 9.99 % attacker ownership | | `C08Capture.t.sol` | — | `fd8e5abef9c272f82439a6ca68f35a3f5e4aeda67521bdbb3da96908ac35c121` | `D = 0` vs `D = 1` unit step, the 0.4 USDG-depth variant, async redemption, gross-debit control | | `C08ZeroDepth.t.sol` | — | `6954044f644f5e3ddb056aefa6de9c84816d640ada7ae9c03d77e0a1dca8622a` | shared fixture, plus earlier probes of this finding that `--match-contract` excludes (see below) |

- **Base and fixture.** The probes inherit `FuzzF2StressConfig` from `C08ZeroDepth.t.sol`, this researcher's existing Foundry harness for a plan-versus-apply property campaign, which itself inherits the suite's `test/perps/BasePerpTest.sol` unchanged. That file also carries earlier, superseded probes of this finding and the campaign's own fuzz tests; the `--match-contract` filter in the commands below excludes all of them, and they compile cleanly. Two of its test-only capabilities are worth naming because they are not used here: a fuzz-only handler action that burns raw pool cash as fault injection, which **no probe in this report calls**, and diagnostic replays that impersonate the Engine or Router inside an always-reverted snapshot, which no probe in this report reaches. The claims-heavy state is produced only by `router.updateMarkPrice` with authored MockPyth ticks plus ordinary opens and closes through the public V3 path (`bootstrapClaimsHistory`); no helper on that path burns, donates or writes storage directly.
- **Mocks.** Only out-of-scope components: MockPyth price updates and the suite's mintable six-decimal token as USDG. `PletherOracle`, the Engine, the Router, `HousePool` and both vaults are the unmodified in-scope contracts.
- **No storage cheats; no impersonation in the drain path.** No file contains `vm.store` or `vm.mockCall`. `vm.etch` appears in exactly one place and is the purpose of that test: `C08FixControl.t.sol` installs the separately compiled patched planner runtime at `engine.planner()` in its third branch, to measure the recommended fix. Its attack and control branches run the unmodified deployed code, and the test asserts the frozen planner's codehash is unchanged in both of them before any etching. Nothing on the attack path impersonates a protocol contract: every step that moves the owner's money is an ordinary external call from an unprivileged address. Two impersonations exist elsewhere and are stated here rather than left to be found: `C08Capture.t.sol:50` and `:149` prank the Junior vault to call `pool.reconcile()` while measuring the cash-out *after* the impact, which is the vault-authorised caller for that function, and the inherited fixture's diagnostic replays prank the Engine or Router inside always-reverted snapshots and are not reached by these probes. Other cheatcodes: `warp`/`roll` for time, `prank`/`startPrank` for externally owned accounts, `recordLogs`/`getRecordedLogs` to read terminal receipts, `expectRevert` for the controls, and state snapshots so the attack and control branches start from one identical state.
- **Configuration.** The release values from `packages/perps/DEPLOYMENT.md` are set in the fixture: vpiFactor 0.01e18, maxSkewRatio 0.4e18, MM 0.1 %, IM 0.2 %, FAD margin 3 %, base carry 500 bps, keeper share 50 %, protocol share 0, 100-token lots, with a 10,000 USDG Junior plus 10,000 USDG Senior pool. Severity figures in the description use these values.
- **Liquidatability.** The owner is asserted non-liquidatable at the execution price through the in-scope `CfdEngineLens`. The source of the pool-assets argument differs by variant: the main carry attack passes the live `pool.totalAssets()` read immediately before the trigger; the matched control also checks live state before the close is queued; and each ladder rung additionally checks the lens with `controlChildPoolAssets` decoded from that branch's authenticated `OrderFinalized` receipt, so no convenient constant is supplied anywhere.

**Run.**

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
git clone https://gist.github.com/Skydev0h/0c9780e04f3bd004eec0fbb127b5e915 c08-poc
mkdir -p packages/perps/test/poc
cp c08-poc/C08*.t.sol c08-poc/C08FixedPlannerRuntime.hex packages/perps/test/poc/
export C08_PATCHED_PLANNER_RUNTIME="$(cat packages/perps/test/poc/C08FixedPlannerRuntime.hex)"
FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --force \
  --match-path 'test/poc/C08*.t.sol' \
  --match-contract 'C08CarryProtectionProbe|C08MatchedControlProbe|C08ShareLadderProbe|C08FixControlProbe' -vv
FOUNDRY_SPARSE_MODE=true forge test --root packages/perps \
  --match-path 'test/poc/C08*.t.sol' --match-contract 'C08CaptureProbe' -vv
```

The patched runtime is supplied through an environment variable, so no repository file and no Foundry configuration is modified. It can be rebuilt instead of trusted: apply `c08-vpi-cap.diff` to a clean checkout of the commit, build, and take the planner's deployed bytecode. Doing that reproduces the attached hex byte for byte (24,002 bytes, codehash `0xff118a15226081ea3af3c5a9e4495598fe57b9bb6aa87420eb045254e8330218`), and the test asserts that codehash after etching.

Both commands pass with the repository's own compiler settings and no compiler-setting or optimizer override: solc 0.8.35, optimizer on, 200 runs, via-IR, taken from `packages/perps/foundry.toml`. The first command is a forced clean build; it compiles 198 files in about 4 minutes and the 9 tests then run in about 0.2 s. The only environment variable used is `FOUNDRY_SPARSE_MODE`, which selects sparse compilation and changes nothing that affects produced bytecode. Tested with Forge 1.7.1.

**Expected output** (first command; USDG amounts have 6 decimals):

```text
Ran 2 tests for test/poc/C08CarryProtection.t.sol:C08CarryProtectionProbe
[PASS] test_controlFiniteActionLimitRollsBackCarryAndRelatches()
[PASS] test_exactZeroProtectionSelfCreatesVpiDepthFromPendingCarry()
  C08 carry deficit/assessed 54 3296780471208208
  C08 carry realized/post-carry depth 148 94
  C08 carry recovered/waived 51173869354 3296729297338854

Ran 1 test for test/poc/C08MatchedControl.t.sol:C08MatchedControlProbe
[PASS] test_attackVsFiniteBoundMatchedLpControl()
  C08 matched identical setup mint 794189891
  C08 matched common attacker/passive LP 20000254622 2000034
  C08 matched child-time pool/depth 9280080012 80012
  C08 matched attack/control pool 59279290375 9280080012
  C08 matched attack/control real cash 59279290375 9280080012
  C08 matched attack/control claims 9280000000 9280000000
  C08 matched attack LP attacker/passive 49513655380 4949408
  C08 matched control LP attacker/passive 0 0
  C08 matched attack-control LP premium 49513655380 4949408
  C08 matched attack-control treasury premium 949483
  C08 matched attack keeper/treasury 1002040000 17336337
  C08 matched control keeper/treasury 1002040000 16386854

Ran 4 tests for test/poc/C08ShareLadder.t.sol:C08ShareLadderProbe
[PASS] test_attackVsFiniteBoundMatchedLpControl()
[PASS] test_ladderFiftyPercentAttackerOwnership()
[PASS] test_ladderTenPercentAttackerOwnership()
[PASS] test_ladderTwentyFivePercentAttackerOwnership()
  C08 ladder target/actual attacker bps 5000 4999
  C08 ladder common attacker/passive LP 9999999995 10002000003
  C08 ladder attack attacker/passive LP 20919670158 20923852647
  C08 ladder control attacker/passive LP 0 0
  C08 ladder target/actual attacker bps 2500 2499
  C08 ladder common attacker/passive LP 4999999997 15002000001
  C08 ladder attack attacker/passive LP 10459835078 31383687727
  C08 ladder target/actual attacker bps 1000 999
  C08 ladder common attacker/passive LP 1999999998 18002000000
  C08 ladder attack attacker/passive LP 4183934030 37659588774
  C08 ladder child-time pool/depth 9280107905 107905
  C08 ladder attack/control pool 51604208391 9280107905
  C08 ladder attack/control claims 9280000000 9280000000
  C08 ladder treasury premium 949483

Ran 2 tests for test/poc/C08FixControl.t.sol:C08FixControlProbe
[PASS] test_attackVsFiniteBoundMatchedLpControl()
[PASS] test_negative_vpiCapMakesProtectionSafeAndAttackUnprofitable()
  C08 fix child gross pool / net depth 9280080012 80012
  C08 fix attack assessed/recovered/waived 58654773938 51173869461 7480904477
  C08 fix patched assessed/recovered/waived 10444321 10444321 0
  C08 fix victim cash before/attack/patched 50000159846 0 50000159805
  C08 fix victim total attack/patched 0 51163425140
  C08 fix attacker LP attack/patched/control 49513655380 0 0
  C08 fix passive LP attack/patched/control 4949408 0 0

Ran 4 test suites: 9 tests passed, 0 failed, 0 skipped (9 total tests)
```

Second command: `8 tests passed, 0 failed`, including

```text
  C08 assessed/recovered/waived 559689841487069 49961759846 559639879727223
  C08 claim/recovered/poolDelta 9280000000 49961759846 50000019844
  C08 incumbentValue/cashOut 47895262918 42782019880
  C08 long-tuner threshold price/depth 197820472 250006
```

**Tests.**

1. `test_exactZeroProtectionSelfCreatesVpiDepthFromPendingCarry` (main attack). The owner arms a take-profit at 1.96 while the pool is healthy at 1.00, with all four close ceilings at `type(uint256).max`. Public leveraged trading over an exogenous price path whose maximum is 1.95 leaves aggregate claims of 9,280.000000 USDG above pool assets, so the take-profit was never previously crossed and claim-net depth is exactly zero. An attacker-controlled account then executes its own full close, leaving the deficit at 54 atoms. Sixty seconds later an arbitrary caller triggers the protection at the first genuine crossing, 1.97809139, where the owner is asserted non-liquidatable. The generated close realizes 148 atoms of the owner's own carry inside `planClose`, so the curve is priced at `D = 94` atoms; it assesses 3,296,780,471.208208 USDG, collects 51,173.869354 USDG, waives the remainder, closes the position and leaves the owner's settlement balance at exactly `0`.
2. `test_controlFiniteActionLimitRollsBackCarryAndRelatches` (paired control). The same scenario with `maxActionChargeAssets = 100 USDG`. The policy evaluator rejects the attempt before Engine settlement; the owner's position, balance and the pool's assets and carry state are unchanged, and the protection returns to `Latched`.
3. `test_attackVsFiniteBoundMatchedLpControl` (matched control). Both branches run from one snapshot with identical setup, time, block, price path, FIFO ids, token mint, share balances and non-owner trader value; the only difference is the stored ceiling. It asserts branch identity, then measures the pool, real token custody, accounted assets, claims, treasury, keeper bounties and both LP positions on each side, with the child-time pool and depth decoded from the authenticated `OrderFinalized` receipt in each branch.
4. `test_ladder{Fifty,TwentyFive,Ten}PercentAttackerOwnership`. Existing tranche shares are redistributed before the common snapshot, so pool assets, supplies, oracle path, branch inputs and total LP ownership are fixed and the attacker's share is the only variable. Each rung repeats the matched control and reports both LP positions.
5. `test_negative_vpiCapMakesProtectionSafeAndAttackUnprofitable` (`C08FixControlProbe`, 1/1 PASS). Three branches from one public history: a zero-ceiling no-close control, the vulnerable unlimited branch, and a patched branch that reverts to the exact pending state captured immediately before the tuner-and-child batch, etches the patched planner at `engine.planner()`, and executes the same order ids at the same timestamp, block and mark. It asserts the patched child is terminally `Executed`, the position closes, the owner keeps 51,163.425140 USDG, and both the attacker's and the independent LP's premiums are exactly zero. Claim-net depth at child time is 0.080012 USDG in both branches.
6. `C08CaptureProbe`, 8 tests: the pure `D = 0` versus `D = 1` step; the public path reaching a 36-atom depth; `test_attackRealisticFortyCentDepthStillConsumesEntireVictimSettlement`, which asserts that a depth of the order of 0.4 USDG still consumes the owner's entire settlement; `test_probeRealisticFortyCentDepthWithLongPartialClose`, which shows a sub-dollar depth of 0.250006 USDG is reachable by ordinary public tuning (that figure is a reachability measurement, not a drain threshold); the attack under unlimited ceilings; two independent finite-ceiling controls, one on the action charge and one on the gross debit; and the production asynchronous redemption that turns the incumbent LP entitlement into 42,782.019880 USDG of cash.

**Assertions in the main attack.**

- Public claims exceed pool assets before the trigger, the remaining deficit is under 100 atoms, and triggering leaves claim-net depth at zero.
- The owner is not liquidatable at the execution price, checked through the in-scope lens.
- The assessed action charge exceeds the owner's entire custody; the recovered amount is above 49,000 USDG; `clearinghouse.balanceAssets(owner)` is exactly `0` afterwards.
- The realized carry exceeds the deficit, and the depth the curve is priced at is exactly 94 atoms.
- In the control, the position size and the owner's balance are unchanged, the pool's assets are restored to their pre-attempt value (the rejected item rolls the carry back), and the protection relatches.

**Matched control and ladder invariants.** In every branch and on every rung: aggregate trader claims are identical between attack and control; `HousePool.totalAssets()`, `accountedAssets()` and the real token balance agree; and the owner's debit equals the pool's gain plus the treasury's execution fee exactly, with no residual (50,000.159846 = 49,999.210363 + 0.949483 in the matched control; 42,325.049969 = 42,324.100486 + 0.949483 on every ladder rung).

### Attached source files

<details>
<summary><b><ins><code>C08CarryProtection.t.sol</code></ins></b></summary>

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {FuzzF2StressConfig} from "./C08ZeroDepth.t.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {Vm} from "forge-std/Vm.sol";
import {console2} from "forge-std/console2.sol";

contract C08CarryProtectionProbe is FuzzF2StressConfig {
    uint256 internal constant TRIGGER_PRICE = 197_809_139;
    bytes32 internal constant ACTION_CHARGE_SETTLED_TOPIC =
        keccak256("ActionChargeSettled(address,uint256,uint256,uint256)");
    bytes32 internal constant CARRY_REALIZED_TOPIC =
        keccak256("CarryRealized(address,uint256,uint256,uint256,uint256)");
    IPositionProtectionActions internal protectionActions;

    function setUp() public override {
        super.setUp();
        protectionActions = IPositionProtectionActions(address(router.positionProtectionBook()));
    }

    function _settledCharge(Vm.Log[] memory logs, address account)
        internal pure returns (uint256 assessed, uint256 recovered, uint256 waived)
    {
        for (uint256 i; i < logs.length; ++i) {
            if (logs[i].topics.length == 2 && logs[i].topics[0] == ACTION_CHARGE_SETTLED_TOPIC
                && address(uint160(uint256(logs[i].topics[1]))) == account) {
                return abi.decode(logs[i].data, (uint256, uint256, uint256));
            }
        }
        revert("C08 carry: charge event missing");
    }

    function _realizedCarry(Vm.Log[] memory logs, address account) internal pure returns (uint256 realizedCarry) {
        for (uint256 i; i < logs.length; ++i) {
            if (logs[i].topics.length == 2 && logs[i].topics[0] == CARRY_REALIZED_TOPIC
                && address(uint160(uint256(logs[i].topics[1]))) == account) {
                (realizedCarry,,,) = abi.decode(logs[i].data, (uint256, uint256, uint256, uint256));
                return realizedCarry;
            }
        }
        revert("C08 carry: carry event missing");
    }

    function _triggerAt(uint64 protectionId, uint256 price) internal returns (uint64 childId) {
        _mockPythUpdateData(price);
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, block.timestamp, previous)
            );
        }
        bytes[] memory updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
        vm.prank(address(0xC08));
        childId = protectionActions.triggerPositionProtection(protectionId, updateData);
    }

    function _queuePublicCarryProtection(uint256 maxActionChargeAssets)
        internal returns (address victim, uint64 protectionId, uint64 childId, uint256 claims, uint256 poolBeforeTrigger)
    {
        victim = handler.traderAt(5);
        handler.debugSetPrice(100_000_000);
        PositionProtectionTypes.PositionProtectionParams memory params;
        params.takeProfitTriggerPrice = 196_000_000;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: maxActionChargeAssets,
            maxExplicitFeesAssets: type(uint256).max
        });
        vm.prank(victim);
        protectionId = protectionActions.createPositionProtection(params);

        claims = handler.bootstrapClaimsHistory();
        assertGt(claims, pool.totalAssets(), "C08 carry: public claims history");
        address tuner = handler.traderAt(1);
        vm.prank(tuner);
        engine.addMargin(tuner, 2_000e6);
        uint64 tunerId = handler.debugCommitFullCloseWithBounds(tuner, type(uint256).max, type(uint256).max);
        handler.debugExecutePendingAtPrice(tunerId, 70_833_762);

        poolBeforeTrigger = pool.totalAssets();
        claims = engine.totalTraderClaimBalanceAssets();
        assertLe(poolBeforeTrigger, claims, "C08 carry: canonical D must be exactly zero");
        assertLt(claims - poolBeforeTrigger, 100, "C08 carry: tiny deficit");

        vm.warp(block.timestamp + 60);
        assertFalse(engineLens.isLiquidatableAt(victim, TRIGGER_PRICE, pool.totalAssets()),
            "C08 carry: victim not liquidatable");
        childId = _triggerAt(protectionId, TRIGGER_PRICE);
        assertLe(pool.totalAssets(), claims, "C08 carry: trigger leaves D zero");
    }

    function test_exactZeroProtectionSelfCreatesVpiDepthFromPendingCarry() public {
        (address victim,, uint64 childId, uint256 claims, uint256 poolBeforeTrigger) =
            _queuePublicCarryProtection(type(uint256).max);

        uint256 balanceBefore = clearinghouse.balanceAssets(victim);
        vm.recordLogs();
        handler.debugExecutePendingAtPrice(childId, TRIGGER_PRICE);
        Vm.Log[] memory logs = vm.getRecordedLogs();
        (uint256 assessed, uint256 recovered, uint256 waived) = _settledCharge(logs, victim);
        uint256 realizedCarry = _realizedCarry(logs, victim);
        uint256 carryCreatedDepth = poolBeforeTrigger + realizedCarry - claims;
        assertGt(assessed, balanceBefore, "C08 carry: epsilon VPI exceeds custody");
        assertGt(recovered, 49_000e6, "C08 carry: victim value recovered");
        assertEq(clearinghouse.balanceAssets(victim), 0, "C08 carry: victim drained");
        assertGt(realizedCarry, claims - poolBeforeTrigger, "C08 carry: victim carry creates positive depth");
        assertEq(carryCreatedDepth, 94, "C08 carry: exact VPI denominator");
        console2.log("C08 carry deficit/assessed", claims - poolBeforeTrigger, assessed);
        console2.log("C08 carry realized/post-carry depth", realizedCarry, carryCreatedDepth);
        console2.log("C08 carry recovered/waived", recovered, waived);
    }

    function test_controlFiniteActionLimitRollsBackCarryAndRelatches() public {
        (address victim, uint64 protectionId, uint64 childId,,) = _queuePublicCarryProtection(100e6);
        uint256 balanceBefore = clearinghouse.balanceAssets(victim);
        uint256 poolBefore = pool.totalAssets();
        (uint256 sizeBefore,,,,,,) = engine.positions(victim);

        handler.debugExecutePendingAtPrice(childId, TRIGGER_PRICE);

        (uint256 sizeAfter,,,,,,) = engine.positions(victim);
        assertEq(sizeAfter, sizeBefore, "C08 carry: bounded close keeps victim position");
        assertEq(clearinghouse.balanceAssets(victim), balanceBefore, "C08 carry: bounded close prevents debit");
        assertEq(pool.totalAssets(), poolBefore, "C08 carry: rejected item rolls back carry");
        assertEq(
            uint256(IPositionProtectionViews(address(router.positionProtectionBook()))
                .getPositionProtection(protectionId).status),
            uint256(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "C08 carry: constraint failure relatches protection"
        );
    }
}
```

</details>

<details>
<summary><b><ins><code>C08MatchedControl.t.sol</code></ins></b></summary>

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {FuzzF2StressConfig} from "./C08ZeroDepth.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {Vm} from "forge-std/Vm.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Time/state-matched C-08 causality control.
/// @dev Both branches start from one EVM snapshot and execute the same public history, price path, tuner close,
///      trigger, child attempt, and elapsed time. The only input changed is the child's stored action-charge ceiling.
contract C08MatchedControlProbe is FuzzF2StressConfig {

    address internal constant PASSIVE_LP = address(0xC085151);
    address internal constant MATCHED_KEEPER = address(0xBEEF0001);
    uint256 internal constant TUNED_PRICE = 197_809_139;
    uint256 internal constant PRE_TRIGGER_PRICE = 195_900_000;
    uint256 internal constant VICTIM_TAKE_PROFIT = 196_000_000;
    uint256 internal constant TUNER_CLOSE_LOTS = 15;

    IPositionProtectionActions internal protectionActions;
    IPositionProtectionViews internal protectionViews;

    struct CommonMetrics {
        uint256 timestamp;
        uint256 blockNumber;
        uint256 supply;
        uint256 attackerLpValue;
        uint256 passiveLpValue;
        uint256 passiveJuniorShares;
        uint256 passiveSeniorShares;
        uint256 attackerJuniorShares;
        uint256 attackerSeniorShares;
        uint256 keeperValue;
        uint256 treasuryValue;
    }

    CommonMetrics internal matchedCommon;

    struct Metrics {
        uint256 startTimestamp;
        uint256 endTimestamp;
        uint256 startBlock;
        uint256 endBlock;
        uint256 supplyBefore;
        uint256 supplyAfter;
        uint256 attackerLpBefore;
        uint256 attackerLpAfter;
        uint256 attackerJuniorAfter;
        uint256 attackerSeniorAfter;
        uint256 passiveLpBefore;
        uint256 passiveLpAfter;
        uint256 poolAfter;
        uint256 claimsAfter;
        uint256 poolBefore;
        uint256 poolCashBefore;
        uint256 poolCashAfter;
        uint256 claimsBefore;
        uint256 poolAccountedBefore;
        uint256 poolAccountedAfter;
        uint256 keeperValueAfter;
        uint256 treasuryValueAfter;
        uint256 victimBalanceBefore;
        uint256 victimBalanceAfter;
        uint256 victimSizeAfter;
        uint256 nonVictimTraderValueAfter;
        uint256 lastMarkPrice;
        uint256 protectionStatus;
        uint64 tunerOrderId;
        uint64 victimOrderId;
    }

    /// @dev Preserve the first branch in a fixed buffer allocated before the snapshot. Keeping returned branch
    ///      structs and spilled scalar locals across revert plus the second via-IR execution caused memory aliasing.
    function _saveMetrics(
        Metrics memory m,
        uint256[30] memory saved
    ) internal pure {
        saved[0] = m.startTimestamp;
        saved[1] = m.endTimestamp;
        saved[2] = m.startBlock;
        saved[3] = m.endBlock;
        saved[4] = m.supplyBefore;
        saved[5] = m.supplyAfter;
        saved[6] = m.attackerLpBefore;
        saved[7] = m.attackerLpAfter;
        saved[8] = m.passiveLpBefore;
        saved[9] = m.passiveLpAfter;
        saved[10] = m.poolAfter;
        saved[11] = m.claimsAfter;
        saved[12] = m.victimBalanceBefore;
        saved[13] = m.victimBalanceAfter;
        saved[14] = m.victimSizeAfter;
        saved[15] = m.nonVictimTraderValueAfter;
        saved[16] = m.lastMarkPrice;
        saved[17] = m.protectionStatus;
        saved[18] = m.tunerOrderId;
        saved[19] = m.victimOrderId;
        saved[20] = m.attackerJuniorAfter;
        saved[21] = m.attackerSeniorAfter;
        saved[22] = m.poolBefore;
        saved[23] = m.poolCashBefore;
        saved[24] = m.poolCashAfter;
        saved[25] = m.claimsBefore;
        saved[26] = m.poolAccountedBefore;
        saved[27] = m.poolAccountedAfter;
        saved[28] = m.keeperValueAfter;
        saved[29] = m.treasuryValueAfter;
    }

    function setUp() public override {
        super.setUp();
        protectionActions = IPositionProtectionActions(address(router.positionProtectionBook()));
        protectionViews = IPositionProtectionViews(address(router.positionProtectionBook()));
    }

    /// @dev Keep the release-size permanent seed positions economically separate from the attacker LP accounts.
    function _juniorSeedReceiver() internal pure override returns (address) {
        return PASSIVE_LP;
    }

    function _seniorSeedReceiver() internal pure override returns (address) {
        return PASSIVE_LP;
    }

    function _lpValue(
        address account
    ) internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(account))
            + seniorVault.convertToAssets(seniorVault.balanceOf(account));
    }

    function _attackerLpValue() internal view returns (uint256) {
        return _lpValue(LP_JUNIOR_SEED) + _lpValue(LP_SENIOR_SEED);
    }

    function _accountValue(
        address account
    ) internal view returns (uint256) {
        return clearinghouse.balanceAssets(account) + engine.traderClaimBalanceAssets(account)
            + settlementToken.balanceOf(account);
    }

    function _nonVictimTraderValue() internal view returns (uint256 value) {
        for (uint256 i; i < 5; ++i) {
            address account = handler.traderAt(i);
            value += _accountValue(account);
        }
    }

    function _receiptPoolAssets(
        uint64 orderId,
        Vm.Log[] memory logs
    ) internal view returns (uint256 poolAssets) {
        IOrderLifecycleBook book = IOrderLifecycleBook(address(router.lifecycleBook()));
        for (uint256 i; i < logs.length; ++i) {
            if (
                logs[i].emitter != address(book) || logs[i].topics.length != 4
                    || logs[i].topics[0] != IOrderLifecycleBook.OrderFinalized.selector
                    || uint64(uint256(logs[i].topics[1])) != orderId
            ) {
                continue;
            }
            // OrderFinalized data has three leading words, then the fixed-shape OrderReceipt. poolDepthAssets is
            // receipt word 13, hence event-data word 16. Reading the one word avoids decoding the very large nested
            // receipt tuple, which triggers a Solidity 0.8.35 via-IR codegen failure in this inherited fuzz fixture.
            bytes memory data = logs[i].data;
            assertGe(data.length, 17 * 32, "C08 matched: complete child receipt");
            assembly ("memory-safe") {
                poolAssets := mload(add(data, 0x220))
            }
            return poolAssets;
        }
        revert("C08 matched: missing child receipt");
    }

    function _armProtection(
        uint256 maxActionChargeAssets
    ) internal returns (uint64 protectionId) {
        address victim = handler.traderAt(5);
        handler.debugSetPrice(100_000_000);
        PositionProtectionTypes.PositionProtectionParams memory params;
        params.takeProfitTriggerPrice = VICTIM_TAKE_PROFIT;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: maxActionChargeAssets,
            maxExplicitFeesAssets: type(uint256).max
        });
        vm.prank(victim);
        protectionId = protectionActions.createPositionProtection(params);
        assertEq(
            uint256(protectionViews.getPositionProtection(protectionId).status),
            uint256(PositionProtectionTypes.PositionProtectionStatus.Armed)
        );

        uint256 claims = handler.bootstrapClaimsHistory();
        assertGt(claims, pool.totalAssets(), "C08 matched: public path reaches D=0");
        assertTrue(engine.degradedMode(), "C08 matched: public path is degraded");
        assertEq(
            uint256(protectionViews.getPositionProtection(protectionId).status),
            uint256(PositionProtectionTypes.PositionProtectionStatus.Armed),
            "C08 matched: TP was not crossed early"
        );
    }

    function _queueTunerAndProtection(
        uint64 protectionId
    ) internal returns (uint64 tunerOrderId, uint64 victimOrderId) {
        address tuner = handler.traderAt(0);
        (uint256 size,,,, CfdTypes.Side side,,) = engine.positions(tuner);
        assertEq(uint256(side), uint256(CfdTypes.Side.LONG));
        assertGe(size / CfdTypes.SIZE_QUANTUM, TUNER_CLOSE_LOTS);

        handler.debugSetPrice(PRE_TRIGGER_PRICE);
        vm.prank(tuner);
        engine.addMargin(tuner, 10_000e6);
        address victim = handler.traderAt(5);
        assertFalse(engineLens.isLiquidatableAt(victim, TUNED_PRICE, pool.totalAssets()));

        (tunerOrderId, victimOrderId) = handler.debugCommitTunerThenTrigger(
            tuner, TUNER_CLOSE_LOTS, protectionId, address(protectionActions), TUNED_PRICE
        );
        assertEq(victimOrderId, tunerOrderId + 1, "C08 matched: FIFO adjacency");
    }

    /// @dev Optional derived-probe hook after the common public history and FIFO commitments, immediately before the
    ///      one batch that executes the tuner followed by the protected child. The baseline probe leaves it empty.
    function _beforeMatchedBatchExecution() internal virtual {}

    function _runBranch(
        uint256 maxActionChargeAssets
    ) internal returns (Metrics memory m) {
        m.startTimestamp = block.timestamp;
        m.startBlock = block.number;
        m.supplyBefore = settlementToken.totalSupply();
        m.attackerLpBefore = _attackerLpValue();
        m.passiveLpBefore = _lpValue(PASSIVE_LP);

        uint64 protectionId = _armProtection(maxActionChargeAssets);
        (m.tunerOrderId, m.victimOrderId) = _queueTunerAndProtection(protectionId);
        address victim = handler.traderAt(5);
        m.victimBalanceBefore = clearinghouse.balanceAssets(victim);
        m.poolBefore = pool.totalAssets();
        m.poolCashBefore = settlementToken.balanceOf(address(pool));
        m.poolAccountedBefore = pool.accountedAssets();
        m.claimsBefore = engine.totalTraderClaimBalanceAssets();

        _beforeMatchedBatchExecution();
        handler.debugExecuteBatchAtPrice(m.victimOrderId, TUNED_PRICE);

        m.endTimestamp = block.timestamp;
        m.endBlock = block.number;
        m.supplyAfter = settlementToken.totalSupply();
        m.attackerJuniorAfter = _lpValue(LP_JUNIOR_SEED);
        m.attackerSeniorAfter = _lpValue(LP_SENIOR_SEED);
        m.attackerLpAfter = m.attackerJuniorAfter + m.attackerSeniorAfter;
        m.passiveLpAfter = _lpValue(PASSIVE_LP);
        m.poolAfter = pool.totalAssets();
        m.poolCashAfter = settlementToken.balanceOf(address(pool));
        m.poolAccountedAfter = pool.accountedAssets();
        m.claimsAfter = engine.totalTraderClaimBalanceAssets();
        m.victimBalanceAfter = clearinghouse.balanceAssets(victim);
        (m.victimSizeAfter,,,,,,) = engine.positions(victim);
        m.nonVictimTraderValueAfter = _nonVictimTraderValue();
        m.keeperValueAfter = _accountValue(MATCHED_KEEPER);
        m.treasuryValueAfter = _accountValue(engine.protocolTreasury());
        m.lastMarkPrice = engine.lastMarkPrice();
        m.protectionStatus = uint256(protectionViews.getPositionProtection(protectionId).status);
    }

    function test_attackVsFiniteBoundMatchedLpControl() public {
        matchedCommon.timestamp = block.timestamp;
        matchedCommon.blockNumber = block.number;
        matchedCommon.supply = settlementToken.totalSupply();
        matchedCommon.attackerLpValue = _attackerLpValue();
        matchedCommon.passiveLpValue = _lpValue(PASSIVE_LP);
        matchedCommon.passiveJuniorShares = juniorVault.balanceOf(PASSIVE_LP);
        matchedCommon.passiveSeniorShares = seniorVault.balanceOf(PASSIVE_LP);
        matchedCommon.attackerJuniorShares = juniorVault.balanceOf(LP_JUNIOR_SEED);
        matchedCommon.attackerSeniorShares = seniorVault.balanceOf(LP_SENIOR_SEED);
        matchedCommon.keeperValue = _accountValue(MATCHED_KEEPER);
        matchedCommon.treasuryValue = _accountValue(engine.protocolTreasury());
        assertGt(matchedCommon.passiveJuniorShares, 0, "C08 matched: passive junior seed");
        assertGt(matchedCommon.passiveSeniorShares, 0, "C08 matched: passive senior seed");
        assertEq(matchedCommon.supply, 321_002e6, "C08 matched: fixed fixture capital");
        uint256[30] memory attackSaved;
        uint256 branchPoint = vm.snapshotState();

        vm.recordLogs();
        Metrics memory attack = _runBranch(type(uint256).max);
        uint256 attackChildPoolAssets = _receiptPoolAssets(attack.victimOrderId, vm.getRecordedLogs());
        _saveMetrics(attack, attackSaved);
        // Slots 20/21 are otherwise only diagnostic per-tranche values; preserve the authenticated child-time pool
        // sample in slot 20 before reverting the branch.
        attackSaved[20] = attackChildPoolAssets;
        assertTrue(vm.revertToState(branchPoint));
        // Foundry state snapshots restore contract storage, but the active block environment is a cheatcode input;
        // reset it explicitly so the second branch has the identical carry clock and publication eligibility window.
        vm.warp(matchedCommon.timestamp);
        vm.roll(matchedCommon.blockNumber);
        vm.recordLogs();
        Metrics memory control = _runBranch(100e6);
        uint256 controlChildPoolAssets = _receiptPoolAssets(control.victimOrderId, vm.getRecordedLogs());

        // Branch identity: same starting state, order ids, time, block, oracle path, capital injection, and tuner result.
        assertEq(attackSaved[0], matchedCommon.timestamp, "C08 matched: attack start time");
        assertEq(control.startTimestamp, matchedCommon.timestamp, "C08 matched: control start time");
        assertEq(attackSaved[2], matchedCommon.blockNumber, "C08 matched: attack start block");
        assertEq(control.startBlock, matchedCommon.blockNumber, "C08 matched: control start block");
        assertEq(attackSaved[1], control.endTimestamp, "C08 matched: same elapsed time");
        assertEq(attackSaved[3], control.endBlock, "C08 matched: same blocks");
        assertEq(attackSaved[16], control.lastMarkPrice, "C08 matched: same oracle path");
        assertEq(attackSaved[18], control.tunerOrderId, "C08 matched: same tuner id");
        assertEq(attackSaved[19], control.victimOrderId, "C08 matched: same child id");
        assertEq(attackSaved[22], control.poolBefore, "C08 matched: same pre-execution pool assets");
        assertEq(attackSaved[23], control.poolCashBefore, "C08 matched: same pre-execution real cash");
        assertEq(attackSaved[25], control.claimsBefore, "C08 matched: same pre-execution claims");
        assertEq(attackSaved[20], controlChildPoolAssets, "C08 matched: same child-time gross pool assets");
        uint256 childNetDepth = attackSaved[20] - attackSaved[25];
        if (
            matchedCommon.attackerJuniorShares == 10_006_664_431_847
                && matchedCommon.attackerSeniorShares == 9_993_254_559_917 && matchedCommon.passiveJuniorShares == 1e9
                && matchedCommon.passiveSeniorShares == 1e9
        ) {
            assertEq(childNetDepth, 80_012, "C08 matched: exact baseline child-time claim-net depth");
        } else {
            // Derived ownership-ladder probes move shares before the branch point. Vault share transfers can settle
            // maintenance/coupon accounting, so their matched child depth differs while remaining equal by branch.
            assertGt(childNetDepth, 0, "C08 matched: derived child-time depth remains positive");
        }
        assertEq(attackSaved[22], attackSaved[23], "C08 matched: attack pool assets are real cash");
        assertEq(control.poolBefore, control.poolCashBefore, "C08 matched: control pool assets are real cash");
        assertEq(attackSaved[22], attackSaved[26], "C08 matched: attack pre-pool is accounted cash");
        assertEq(control.poolBefore, control.poolAccountedBefore, "C08 matched: control pre-pool is accounted cash");
        assertEq(attackSaved[4], matchedCommon.supply);
        assertEq(control.supplyBefore, matchedCommon.supply);
        assertEq(
            attackSaved[5] - attackSaved[4],
            control.supplyAfter - control.supplyBefore,
            "C08 matched: identical setup mint/top-up"
        );
        assertEq(attackSaved[15], control.nonVictimTraderValueAfter, "C08 matched: tuner/claimant accounts identical");
        assertEq(attackSaved[28], control.keeperValueAfter, "C08 matched: keeper value identical");
        assertGt(attackSaved[29], control.treasuryValueAfter, "C08 matched: executed child pays explicit treasury fee");
        assertEq(
            juniorVault.balanceOf(PASSIVE_LP),
            matchedCommon.passiveJuniorShares,
            "C08 matched: passive holds same junior shares"
        );
        assertEq(
            seniorVault.balanceOf(PASSIVE_LP),
            matchedCommon.passiveSeniorShares,
            "C08 matched: passive holds same senior shares"
        );
        assertEq(
            juniorVault.balanceOf(LP_JUNIOR_SEED),
            matchedCommon.attackerJuniorShares,
            "C08 matched: attacker holds same shares"
        );
        assertEq(seniorVault.balanceOf(LP_SENIOR_SEED), matchedCommon.attackerSeniorShares);

        // Only the unlimited child executes. The finite child leaves the victim whole and relatches protection.
        assertEq(attackSaved[14], 0, "C08 matched: attack closes victim");
        assertEq(attackSaved[13], 0, "C08 matched: attack drains victim");
        assertGt(control.victimSizeAfter, 0, "C08 matched: control keeps victim position");
        assertEq(control.victimBalanceAfter, control.victimBalanceBefore, "C08 matched: control does not debit victim");
        assertEq(
            control.protectionStatus,
            uint256(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "C08 matched: finite control relatches"
        );

        // Both unrelated LP holders receive strictly more canonical marked value only in the attack branch.
        uint256 attackerAttackPremium = attackSaved[7] - control.attackerLpAfter;
        uint256 passiveAttackPremium = attackSaved[9] - control.passiveLpAfter;
        uint256 treasuryAttackPremium = attackSaved[29] - control.treasuryValueAfter;
        assertGt(attackerAttackPremium, 1000e6, "C08 matched: attacker LP captures victim transfer");
        assertGt(passiveAttackPremium, 1e6, "C08 matched: passive LP independently shares revenue");
        assertEq(treasuryAttackPremium, 949_483, "C08 matched: explicit fee is separately accounted");
        assertEq(
            attackSaved[10] - control.poolAfter + treasuryAttackPremium,
            attackSaved[12],
            "C08 matched: pool cash plus treasury fee equals drained victim cash"
        );
        assertEq(attackSaved[10], attackSaved[24], "C08 matched: attack post-pool assets are real cash");
        assertEq(control.poolAfter, control.poolCashAfter, "C08 matched: control post-pool assets are real cash");
        assertEq(attackSaved[10], attackSaved[27], "C08 matched: attack post-pool is accounted cash");
        assertEq(control.poolAfter, control.poolAccountedAfter, "C08 matched: control post-pool is accounted cash");
        assertEq(attackSaved[5], 321_796_189_891, "C08 matched: exact attack end supply");
        assertEq(control.supplyAfter, 321_796_189_891, "C08 matched: exact control end supply");

        console2.log(
            "C08 matched common attacker/passive LP", matchedCommon.attackerLpValue, matchedCommon.passiveLpValue
        );
        console2.log("C08 matched attack LP attacker/passive", attackSaved[7], attackSaved[9]);
        console2.log("C08 matched control LP attacker/passive", control.attackerLpAfter, control.passiveLpAfter);
        console2.log("C08 matched attack-control LP premium", attackerAttackPremium, passiveAttackPremium);
        console2.log("C08 matched attack-control treasury premium", treasuryAttackPremium);
        console2.log("C08 matched attack/control pool", attackSaved[10], control.poolAfter);
        console2.log("C08 matched pre-execution pool/cash", attackSaved[22], attackSaved[23]);
        console2.log("C08 matched attack/control real cash", attackSaved[24], control.poolCashAfter);
        console2.log("C08 matched attack/control accounted", attackSaved[27], control.poolAccountedAfter);
        console2.log("C08 matched attack/control claims", attackSaved[11], control.claimsAfter);
        console2.log("C08 matched child-time pool/depth", attackSaved[20], childNetDepth);
        console2.log("C08 matched common keeper/treasury", matchedCommon.keeperValue, matchedCommon.treasuryValue);
        console2.log("C08 matched attack keeper/treasury", attackSaved[28], attackSaved[29]);
        console2.log("C08 matched control keeper/treasury", control.keeperValueAfter, control.treasuryValueAfter);
        console2.log("C08 matched identical setup mint", attackSaved[5] - attackSaved[4]);
        console2.log(
            "C08 matched initial junior shares attacker/passive",
            matchedCommon.attackerJuniorShares,
            matchedCommon.passiveJuniorShares
        );
        console2.log(
            "C08 matched initial senior shares attacker/passive",
            matchedCommon.attackerSeniorShares,
            matchedCommon.passiveSeniorShares
        );
    }

}
```

</details>

<details>
<summary><b><ins><code>C08ShareLadder.t.sol</code></ins></b></summary>

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {C08MatchedControlProbe} from "./C08MatchedControl.t.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Runs the time-matched C-08 control at progressively smaller incumbent ownership shares.
/// @dev Existing shares move before the common branch point. Pool assets, total share supply, public history,
///      oracle path, and every execution input remain unchanged between attack and finite-limit control.
contract C08ShareLadderProbe is C08MatchedControlProbe {

    function _ageSeedSharesPastTransferCooldown() internal {
        uint256 cooldownEnd = juniorVault.lastDepositTime(LP_JUNIOR_SEED) + juniorVault.DEPOSIT_COOLDOWN();
        uint256 seniorCooldownEnd = seniorVault.lastDepositTime(LP_SENIOR_SEED) + seniorVault.DEPOSIT_COOLDOWN();
        if (seniorCooldownEnd > cooldownEnd) {
            cooldownEnd = seniorCooldownEnd;
        }
        if (block.timestamp <= cooldownEnd) {
            vm.warp(cooldownEnd + 1);
        }
    }

    function _moveSharesToPassive(
        uint256 targetBps
    ) internal {
        uint256 juniorBefore = juniorVault.balanceOf(LP_JUNIOR_SEED);
        uint256 seniorBefore = seniorVault.balanceOf(LP_SENIOR_SEED);
        uint256 juniorRetained = (juniorBefore * targetBps) / 10_000;
        uint256 seniorRetained = (seniorBefore * targetBps) / 10_000;

        vm.prank(LP_JUNIOR_SEED);
        assertTrue(juniorVault.transfer(PASSIVE_LP, juniorBefore - juniorRetained));
        vm.prank(LP_SENIOR_SEED);
        assertTrue(seniorVault.transfer(PASSIVE_LP, seniorBefore - seniorRetained));
    }

    function _captureCommon() internal {
        matchedCommon.timestamp = block.timestamp;
        matchedCommon.blockNumber = block.number;
        matchedCommon.supply = settlementToken.totalSupply();
        matchedCommon.attackerLpValue = _attackerLpValue();
        matchedCommon.passiveLpValue = _lpValue(PASSIVE_LP);
        matchedCommon.passiveJuniorShares = juniorVault.balanceOf(PASSIVE_LP);
        matchedCommon.passiveSeniorShares = seniorVault.balanceOf(PASSIVE_LP);
        matchedCommon.attackerJuniorShares = juniorVault.balanceOf(LP_JUNIOR_SEED);
        matchedCommon.attackerSeniorShares = seniorVault.balanceOf(LP_SENIOR_SEED);
        matchedCommon.treasuryValue = _accountValue(engine.protocolTreasury());
    }

    function _assertBranchIdentity(
        uint256[30] memory attack,
        Metrics memory control,
        uint256 controlChildPoolAssets
    ) internal view {
        assertEq(attack[0], matchedCommon.timestamp, "C08 ladder: attack start time");
        assertEq(control.startTimestamp, matchedCommon.timestamp, "C08 ladder: control start time");
        assertEq(attack[1], control.endTimestamp, "C08 ladder: elapsed time");
        assertEq(attack[3], control.endBlock, "C08 ladder: elapsed blocks");
        assertEq(attack[16], control.lastMarkPrice, "C08 ladder: oracle path");
        assertEq(attack[18], control.tunerOrderId, "C08 ladder: tuner order");
        assertEq(attack[19], control.victimOrderId, "C08 ladder: protection child");
        assertEq(attack[5] - attack[4], control.supplyAfter - control.supplyBefore, "C08 ladder: setup mint");
        assertEq(attack[15], control.nonVictimTraderValueAfter, "C08 ladder: other traders");
        assertEq(attack[11], control.claimsAfter, "C08 ladder: claims identical");
        assertEq(attack[11], 9280e6, "C08 ladder: exact claims");
        uint256 childDepth = attack[20] - attack[25];
        assertEq(childDepth, controlChildPoolAssets - control.claimsBefore, "C08 ladder: child depth identical");
        assertGt(childDepth, 0, "C08 ladder: positive depth branch");
        assertLt(childDepth, 200_000, "C08 ladder: bounded epsilon depth");
        assertFalse(
            engineLens.isLiquidatableAt(handler.traderAt(5), TUNED_PRICE, controlChildPoolAssets),
            "C08 ladder: victim non-liquidatable at authenticated child pool assets"
        );
    }

    function _assertVictimAndConservation(
        uint256[30] memory attack,
        Metrics memory control
    ) internal view {
        assertEq(attack[14], 0, "C08 ladder: victim position closed");
        assertLt(attack[13], attack[12], "C08 ladder: victim balance debited");
        uint256 victimDebit = attack[12] - attack[13];
        assertGt(victimDebit, 40_000e6, "C08 ladder: victim suffers material debit");
        assertGt(control.victimSizeAfter, 0, "C08 ladder: finite control keeps position");
        assertEq(control.victimBalanceAfter, control.victimBalanceBefore, "C08 ladder: finite control keeps cash");

        uint256 treasuryPremium = attack[29] - control.treasuryValueAfter;
        assertEq(treasuryPremium, 949_483, "C08 ladder: exact explicit 4 bps fee");
        assertEq(
            attack[10] - control.poolAfter + treasuryPremium, victimDebit, "C08 ladder: victim cash debit conserved"
        );
        assertEq(attack[10], attack[24], "C08 ladder: attack pool equals raw cash");
        assertEq(control.poolAfter, control.poolCashAfter, "C08 ladder: control pool equals raw cash");
    }

    function _assertLpOutcomes(
        uint256[30] memory attack,
        Metrics memory control
    ) internal view {
        assertGt(attack[7], matchedCommon.attackerLpValue, "C08 ladder: minority attacker net profit");
        assertGt(attack[9], matchedCommon.passiveLpValue, "C08 ladder: passive LP gain");
        assertEq(control.attackerLpAfter, 0, "C08 ladder: control attacker remains impaired");
        assertEq(control.passiveLpAfter, 0, "C08 ladder: control passive remains impaired");
        assertEq(juniorVault.balanceOf(LP_JUNIOR_SEED), matchedCommon.attackerJuniorShares);
        assertEq(seniorVault.balanceOf(LP_SENIOR_SEED), matchedCommon.attackerSeniorShares);
        assertEq(juniorVault.balanceOf(PASSIVE_LP), matchedCommon.passiveJuniorShares);
        assertEq(seniorVault.balanceOf(PASSIVE_LP), matchedCommon.passiveSeniorShares);
    }

    function _logRung(
        uint256 targetBps,
        uint256 actualBps,
        uint256[30] memory attack,
        Metrics memory control
    ) internal view {
        console2.log("C08 ladder target/actual attacker bps", targetBps, actualBps);
        console2.log(
            "C08 ladder common attacker/passive LP", matchedCommon.attackerLpValue, matchedCommon.passiveLpValue
        );
        console2.log("C08 ladder attack attacker/passive LP", attack[7], attack[9]);
        console2.log("C08 ladder control attacker/passive LP", control.attackerLpAfter, control.passiveLpAfter);
        console2.log(
            "C08 ladder net gain attacker/passive",
            attack[7] - matchedCommon.attackerLpValue,
            attack[9] - matchedCommon.passiveLpValue
        );
        console2.log("C08 ladder attack/control pool", attack[10], control.poolAfter);
        console2.log("C08 ladder attack/control claims", attack[11], control.claimsAfter);
        console2.log("C08 ladder child-time pool/depth", attack[20], attack[20] - attack[25]);
        console2.log("C08 ladder treasury premium", attack[29] - control.treasuryValueAfter);
    }

    function _executeMatchedRung(
        uint256 targetBps,
        uint256 actualBps
    ) internal {
        uint256[30] memory attackSaved;
        uint256 branchPoint = vm.snapshotState();
        vm.recordLogs();
        Metrics memory attack = _runBranch(type(uint256).max);
        uint256 attackChildPoolAssets = _receiptPoolAssets(attack.victimOrderId, vm.getRecordedLogs());
        _saveMetrics(attack, attackSaved);
        attackSaved[20] = attackChildPoolAssets;

        assertTrue(vm.revertToState(branchPoint), "C08 ladder: restore branch point");
        vm.warp(matchedCommon.timestamp);
        vm.roll(matchedCommon.blockNumber);
        vm.recordLogs();
        Metrics memory control = _runBranch(100e6);
        uint256 controlChildPoolAssets = _receiptPoolAssets(control.victimOrderId, vm.getRecordedLogs());

        _assertBranchIdentity(attackSaved, control, controlChildPoolAssets);
        _assertVictimAndConservation(attackSaved, control);
        _assertLpOutcomes(attackSaved, control);
        _logRung(targetBps, actualBps, attackSaved, control);
    }

    function _runOwnershipRung(
        uint256 targetBps
    ) internal {
        require(targetBps != 0 && targetBps < 10_000, "C08 ladder: target range");
        _ageSeedSharesPastTransferCooldown();
        uint256 poolBefore = pool.totalAssets();
        uint256 juniorSupplyBefore = juniorVault.totalSupply();
        uint256 seniorSupplyBefore = seniorVault.totalSupply();
        _moveSharesToPassive(targetBps);
        _captureCommon();

        uint256 totalLpValue = matchedCommon.attackerLpValue + matchedCommon.passiveLpValue;
        uint256 actualBps = (matchedCommon.attackerLpValue * 10_000) / totalLpValue;
        assertApproxEqAbs(actualBps, targetBps, 2, "C08 ladder: economic ownership target");
        assertEq(pool.totalAssets(), poolBefore, "C08 ladder: transfer changes no pool assets");
        assertEq(juniorVault.totalSupply(), juniorSupplyBefore, "C08 ladder: no junior dilution");
        assertEq(seniorVault.totalSupply(), seniorSupplyBefore, "C08 ladder: no senior dilution");

        _executeMatchedRung(targetBps, actualBps);
    }

    function test_ladderFiftyPercentAttackerOwnership() public {
        _runOwnershipRung(5000);
    }

    function test_ladderTwentyFivePercentAttackerOwnership() public {
        _runOwnershipRung(2500);
    }

    function test_ladderTenPercentAttackerOwnership() public {
        _runOwnershipRung(1000);
    }

}
```

</details>
