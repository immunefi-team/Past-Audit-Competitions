# \#97957 [SC-High] Close-time VPI charge is bounded only at exactly zero pool depth, so at attacker-tuned tiny non-zero depth a skew-increasing close is assessed many times the position's notional and seizes the trader's entire account into LP revenue

**Submitted on Nov 4th 2026 at 03:38:12 UTC by @risinghorse for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97957
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief / Intro

Plether's close-time skew charge (VPI) is `C(S) = 0.5·k·S²/D`, where `D` is pool depth. The close planner skips a fresh charge **only when `D == 0` exactly**; at any tiny non-zero `D` the full curve runs, and because `D` is the denominator the charge grows without bound — it can exceed the closing position's notional many times over. The charge is then collected from everything the account owns on a full close (free settlement, released pledge, committed margin), capped only by the trader's own `maxActionChargeAssets` order limit, and the collected cash becomes LP revenue. Pool depth `D = poolAssets − traderClaims` is set by the immediately-preceding order in the FIFO queue, so a party that holds LP shares and controls trading accounts can drain `D` to ~1 USDG and then cause a third party's skew-increasing close (e.g. by triggering the victim's own take-profit) to be assessed a charge that consumes the victim's whole account. Because the attacker holds the first-loss Junior claim, the seized value returns to it on redemption.

## Vulnerability Details

**The guard is only at exactly zero depth.** `CloseAccountingLib.buildCloseState` ([`libraries/CloseAccountingLib.sol:105-111`](../../../src/libraries/CloseAccountingLib.sol)):

```solidity
// With no economic depth, keep protective exits live without creating a fresh charge or rebate.
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
```

At `D == 0` the charge is skipped; at `D == 1` the curve runs at full strength. `CfdMath.getSkewCost` / `calculateVPI` ([`CfdMath.sol:102-161`](../../../src/CfdMath.sol)) compute `0.5·k·S²/D`, which diverges as `D → 0⁺`. This is a discontinuity: a close at `D = 0` is charge-free, while the same close at `D = 1` USDG is assessed a charge far larger than the position.

**The charge is assessed on attacker-controlled depth and collected from the whole account.** The close planner takes `poolDepthAssets = computeNetPoolDepthAssets(poolAssets, traderClaims) = P − C` ([`libraries/CfdEnginePlanLib.sol:1339-1340`](../../../src/libraries/CfdEnginePlanLib.sol)). `D` is therefore whatever the immediately-preceding FIFO order left it at: paying out claims or a prior close can drive `P − C` down to any small positive value (or over-shoot to exactly 0, which is charge-free). On a full close the assessed charge is collected from withheld gain, the action reserve, free settlement, released pledge and committed margin ([`libraries/CfdEnginePlanLib.sol:1093-1134`](../../../src/libraries/CfdEnginePlanLib.sol)); any remainder is waived. The only ceiling is the trader's own `maxActionChargeAssets` order-bound, enforced by the policy evaluator. The collected cash is recorded as LP revenue ([`CfdEngineSettlementSidecar.sol:604-613`](../../../src/CfdEngineSettlementSidecar.sol)).

**Why a victim is exposed.** A trader whose close order sets `maxActionChargeAssets` to the integer-max sentinel (documented as "no practical ceiling," and the value the protocol's own integration adapter uses) authorizes an unbounded charge. A take-profit / stop-loss child inherits the parent's limits and is triggered permissionlessly, so the attacker can have the victim's close executed at a depth the attacker has just tuned, behind its own depth-tuning order in the FIFO.

**The sequence the PoC performs (deterministic variant, `test_R5B`):**
1. The attacker holds Junior LP shares and a hedged pair. The market moves up (mark 0.977 → 1.526); the attacker closes most of its short, which latches degraded mode.
2. An innocent trader holds a long with ~150,000 free settlement and an armed take-profit with an effectively unlimited charge limit.
3. The market falls; the attacker partially closes its long to tune `D` to ≈ 1 USDG and keeps its residual short heavy so the victim's close is skew-increasing.
4. The attacker commits a depth-tuning partial close, then (once the mark passes the trigger) any keeper triggers the victim's take-profit, which lands behind the tuning order in FIFO.
5. The victim's close is assessed ≈ 227,806 on a ≈ 19,000-notional position; ≈ 209,882 is recovered from its account (the rest waived), leaving the victim with 0. The attacker closes out, the owner clears the latch, and the attacker redeems its Junior shares to collect the seized value.

A causal control (`test_A_ZeroDepth_NoCharge_Discontinuity`) confirms the root cause: the identical close at `D = 0` has no charge and the victim keeps its value (151,010 → 208,275).

## Impact Details

All figures from `TinyDepthVpi.t.sol` and `TinyDepthRefute.t.sol` on the strict release fixture, USDG, wallet-level against a same-state control.

**Deterministic theft (`test_R5B`, trigger below the drain level, any keeper):** economic depth tuned to ≈ 1.05 USDG; the victim's triggered close is assessed **227,806.15**, recovers **209,881.69**, waives 17,924.46. Victim wallet **0 vs 209,105.17** control (**−209,105.17**); attacker **+209,097.90**. This variant does not depend on winning a keeper race.

**Direct-trigger form (`test_A_TinyDepth_VictimTakeProfitTriggeredByAttacker`):** depth ≈ 1 USDG; assessed 283,872.80, recovered 208,323.78, waived 75,549.02; victim wallet 0 vs 207,402.88 (**−207,402.88**), attacker **+207,394.97**. (This exact-level arm can be dodged by a third-party keeper triggering earlier at harmless depth — `test_R5A` — which is why `R5B` is the one to rely on.)

**Discontinuity (`test_A_ZeroDepth_NoCharge_Discontinuity`):** the same close at `D = 0` is charge-free; the guard at `CloseAccountingLib.sol:107` is exactly the boundary.

**Scope and the honest objections:**
- **User consent (the main objection).** The loss requires the victim's own `maxActionChargeAssets` to be effectively unlimited. With any finite limit the policy evaluator rejects the close (`test_A_TinyDepth_BoundedVictimCannotExit` ends terminal `ConstraintViolation`, no value lost). A reviewer can read the integer-max limit as accepted exposure, like an `amountOutMin = 0` swap. Counter: a charge of ~15× notional is hard to call a "price," and the docs recommend the max sentinel to mean "no practical ceiling," so it is not obviously consent to an attacker-set depth.
- **The "freeze" fallback fails — do not claim it.** A bounded victim is not trapped: from the same state it exits with floor-sized partial closes — the first 27-lot chunk is charged 2,940.19 and drives depth to 0, after which further chunks pay only the ~2.6 USDG execution fee (`test_R1`). So a bounded victim leaves at ~1–5% cost; there is no 24-hour freeze.
- **Profit needs the first-loss claim.** With an innocent third-party Senior LP present (`test_R4`), the attacker's LP wallet gains nothing (net ≈ −115), the victim still loses ≈ 73,565, and that value accrues to the **Senior LP**. So the attack is "direct theft" only when the attacker holds the first-loss Junior claim; otherwise it is griefing (damage to the victim, no attacker profit) at a high damage-to-cost ratio.
- **Feasibility.** The setup needs a basket round trip of roughly 115% of entry to reach the drained/degraded state. Under the program's rules this affects payout, not severity, but a reviewer may argue such a print is oracle-failure territory.
- **Correction to the raw depth figure.** The VPI denominator is not literally ~1 USDG: the planner adds the victim's projected pending carry into depth first, so the effective `D` is ~18 USDG; the charge is computed on that. This does not change the outcome (the charge still vastly exceeds notional) but the report should state it.

**Severity, honestly:** literal **Critical** (direct theft of a trader's at-rest free settlement and in-motion gain) only for a victim with an effectively unlimited charge limit and an attacker holding the first-loss claim. Realistic outcome **Medium**, with meaningful probability of rejection on consent/design grounds. No protocol insolvency and no permanent freeze.

## References

- `packages/perps/src/libraries/CloseAccountingLib.sol:105-111` — VPI curve skipped only at `D == 0`; runs at full strength for any non-zero `D`.
- `packages/perps/src/CfdMath.sol:102-161` — `getSkewCost`/`calculateVPI` compute `0.5·k·S²/D`, which diverges as `D → 0⁺`.
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:1339-1340` (depth = `P − C`), `:1093-1134` (charge collected from all account funds on a full close, remainder waived).
- `packages/perps/src/CfdEngineSettlementSidecar.sol:604-613` — the collected charge is booked as LP revenue (`Revenue / CashArrived`).
- Documented context: SECURITY.md "VPI limitations" (depth-dependence; the only exception stated is `D == 0`); ACCOUNTING_SPEC.md (full-close collection from eligible funds). The charge *exceeding notional at tiny non-zero depth* is not documented.
- Known-issue check / **disclosure risk (stated openly):** not in the Cecuro AI audit, SC pre-audit or AI scan list. The AI scan's `scan.md:1693/1742` info items concern a trader enlarging *its own* rebate by moving depth — opposite direction, different victim. However, repo test helpers reference "near-zero-depth new VPI charges" (e.g. `test/perps/invariant/handlers/PerpMixedAccountingHandler.sol`, `test/perps/spec/keeper/OrderBountyLifecycle.t.sol`), so a reviewer could argue the root cause was visible in the repository; the rebuttal is that the lifetime clamp those tests exercise bounds only *rebates*, not the charge side, and the seizure of a third party's account is not disclosed anywhere.


## Proof of Concept

## Proof of Concept

Runs in the repository's Foundry suite at the in-scope commit, no in-scope contract modified; only Pyth and USDG are mocked. All steps use public entrypoints and EOA signers; the only owner call is the documented `clearDegradedMode`. No `vm.store`, `vm.etch`, `mockCall`, or prank as router/engine/pool/vault.

**Files:** `packages/perps/test/audit_poc/leftover/TinyDepthVpi.t.sol` (main arms) and `leftover/TinyDepthRefute.t.sol` (the controls and the deterministic variant), with their shared bases. 4/4 and 6/6 pass.

**Run:**
```bash
cd packages/perps && "$HOME/.foundry/bin/forge" test --match-path "test/audit_poc/leftover/TinyDepthVpi.t.sol" --skip "test/perps/**/*.t.sol" --skip "script/**" -vv
```
```bash
cd packages/perps && "$HOME/.foundry/bin/forge" test --match-path "test/audit_poc/leftover/TinyDepthRefute.t.sol" --skip "test/perps/**/*.t.sol" --skip "script/**" -vv
```

**Key tests and what they show:**
- `test_R5B_TriggerBelowDrainLevel_AnyKeeper` — the deterministic theft: victim 0 vs 209,105 control; attacker +209,098; no keeper race.
- `test_A_TinyDepth_VictimTakeProfitTriggeredByAttacker` — the direct-trigger form: charge 283,873 assessed on a ~19k position, victim account fully seized.
- `test_A_ZeroDepth_NoCharge_Discontinuity` — the same close at `D = 0` is charge-free (pins the root cause to the `D != 0` guard).
- `test_R1_NoProtection_PartialCloseExit` — a bounded victim exits via floor-sized partials at ~1–5% cost (shows the freeze claim is false).
- `test_R4_ThirdPartySenior` — with an innocent Senior LP the attacker profits nothing and the Senior LP receives the seized value (profit requires holding the first-loss claim).
- `test_R5A_KeeperWinsRace` — the exact-level trigger can be dodged by an earlier keeper (why `R5B` is the reliable form).

### Suggested remediation (directional)

Bound the close-time VPI charge the way the curve already assumes — e.g. cap `C(S)` at a fraction of the closed notional, or guard the charge against a minimum effective depth rather than only `D == 0`, so that a skew-increasing close at tiny depth cannot be assessed more than the position is worth. The existing `D == 0` carve-out for protective exits should extend to the near-zero region it was clearly meant to cover.
