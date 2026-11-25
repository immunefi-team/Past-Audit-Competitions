# \#97689 [SC-Insight] FrozenCloseSpreadSettled reports the whole frozen spread as waived even when LPs retained part of it through the VPI rebate netting

**Submitted on Nov 2nd 2026 at 17:39:41 UTC by @crypt0n1337 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97689
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

In the frozen oracle regime a voluntary close pays a fixed spread on the reduced notional, and that spread belongs to LPs. The close planner nets the action components: `actionNet = vpiDelta + fee + spread + carry - realizedCarry` (`libraries/CfdEnginePlanLib.sol:1041-1061`). When `actionNet < 0` the entire net amount becomes `actionRebateAssets`, and `actionChargeAssessedAssets` stays zero.

`_recoveredFrozenSpreadAssets` (`CfdEngineSettlementSidecar.sol:724-748`) then infers that no charge was assessed, so no spread was collected: `effectiveSpread = 0`, `recovered = 0`, and it emits `FrozenCloseSpreadSettled(account, spread, 0, spread)`, reporting the entire spread as waived. In reality the spread reduced the trader's rebate, so LPs did retain it. The same computation is duplicated for the preview surface in `CfdEngineLens._frozenSpreadPaidAssets`.

This contradicts the documented semantics. `ACCOUNTING_SPEC.md:153` describes the waived amount as "actually retained or collected", `SECURITY.md:1306-1309` calls it "terminally uncollectible", and `SECURITY.md:1372` promises that complete terminal economics are event backed. No funds move incorrectly: the trader and the LPs receive exactly what they would receive with a correct report. But the event, and every indexer or dashboard that reconstructs close economics from logs, systematically understates LP revenue by the VPI rebated portion of the spread (in the rebate branch, by the entire spread).

Reachability at the release configuration (`script/DeployPerpsArbitrumSepolia.s.sol:436`: `vpiFactor 0.01e18`, `maxSkew 0.4`, spread 50 bps, fee 4 bps). The rebate branch is unreachable because the maximum VPI accrual is 20 bps which is below 54 bps, so the refund clamps to the accrual. The same defect is reachable today in the charge branch: a 400 000 notional close in the frozen regime reports `recovered 1 202.314764` and `waived 797.685236`, where `waived` equals exactly the VPI delta that LPs actually retained, so it should not be reported as waived at all.

Why this is not a listed known issue: neither `audits/scan-plether-fi-plether-core-d4541b49.md` nor the Cecuro AI audit report mentions `FrozenCloseSpreadSettled`, `frozenSpreadPaidAssets` or `waived`. The closest known issues are MED-30 (protection during frozen markets), MED-22 (carry rounding) and MED-7 (basket component selection), all of which have different roots. The combination "the spread replaces the adverse confidence shift, but its collection is accounted for only through assessed" appears neither in SECURITY.md nor in ACCOUNTING_SPEC.md.

Suggested remediation: report `waived` from the actual cash movement (`spread - min(spread, rebateOffset)`), or return split components (`spreadAssessedAssets`, `spreadRebatedAssets`) from `_planCloseActionSettlement` and pass them into `_recoveredFrozenSpreadAssets`, and add a test asserting `waived + recovered == actual LP cash delta` for a frozen close with a nonzero VPI offset.

## Proof of Concept

Environment: forge 1.8.1, solc 0.8.35+commit.47b9dedd, optimizer 200, via_ir enabled, HEAD a82d4fef. Local deployment only, no network access and no transactions to any public chain.
Test file: packages/perps/test/poc/D2_FrozenSpreadEventLies.t.sol, 3 of 3 PASS.
Run: forge test --root packages/perps --match-path "test/poc/*" -vv
Result at the in-scope commit: 5 suites, 11 tests passed, 0 failed.

Test A, rebate branch (vpiFactor = 0.1e18), 300 000 notional closed at 1.0 on a frozen Saturday:
1. vpiDelta = -4 470.978796, fee = 120, spread = 1 500, so actionNet = -2 850.978796.
2. The planner puts the whole net into the rebate, so the trader receives 2 850.978796.
3. The event reports assessed 1 500, recovered 0, waived 1 500.
4. Cash movement: pool -2 850.978796, trader +2 850.978796.
5. Control comparison: the same close on a live market rebates 4 350.978796. The difference is exactly 1 500.000000, which is the frozen spread. LPs retained it, yet the event reports it as waived.

Test B, negative control (vpiFactor = 0), close at 0.98:
1. assessed 1 587.6 = fee 117.6 + spread 1 470, recovered 1 470, waived 0.
2. Trader delta 4 412.4 = gain - fee - spread; pool delta -4 530.
3. Versus the live market the difference is exactly 1 470, so in this branch the event matches the money.

Test C, release configuration reachability:
1. With vpiFactor 0.01e18 and maxSkew 0.4 the maximum VPI accrual is 20 bps, and the refund clamps to the accrual, so the rebate branch is unreachable on the default configuration.
2. The same defect is reachable today in the charge branch. A 400 000 notional frozen close reports assessed spread 2 000, recovered 1 202.314764 and waived 797.685236.
3. waived equals |vpiDelta| exactly, and that is the part LPs actually retained through the netting, so it should not be reported as waived at all.
4. Running the same close with the spread set to 0 bps in the same block moves the pool by 1 840.000000 less, so the pool did keep spread value while the event reports 797.685236 of the assessed spread as waived.
