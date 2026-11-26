# \#99276 [SC-Critical] VPI rebate cross-cohort funding lets an exiting LP capture principal from remaining LPs

**Submitted on Nov 12th 2026 at 20:14:18 UTC by @Kentavr for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99276
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A VPI charge paid by a trader is credited pro-rata to all LP shares, but the later VPI rebate is paid from the pool's free cash. An LP can exit after the charge and before the rebate, keeping its share of the charge while the remaining LP cohort funds the rebate. At the documented $2M launch depth (1M Junior + 1M Senior), the direct split PoC shows the remaining Junior LP loses $753.888612/cycle (0.1839%) versus the A-stays control and its claim is $622.164145 (0.1517%) below its $410,000 deposit after one cycle. At 8 cycles/day this is $6,031.11 (1.4710%) loss and $4,977.31 (1.2140%) principal loss per day, while the exiting LP/trader captures ~$301.19/cycle net.

## Vulnerability Details

The VPI path is active at launch (`vpiFactor = 0.01e18`, `maxSkewRatio = 0.4e18`, execution fee 4 bps; `DEPLOYMENT.md:131-142`). When a trader opens, the VPI charge is computed in `CfdEngineSettlementSidecar`/`CfdEnginePlanLib` and recorded into pool accounting through `HousePool.recordClaimantInflow(Revenue, CashArrived)`, which increases `accountedAssets` shared by all LP shares. When the position closes, the VPI rebate is paid from the pool's free cash in `HousePool.payOut`/`settleLpEpoch`.

The per-position lifetime clamp in `CloseAccountingLib` caps the rebate at that position's lifetime charge, but it does not bind the rebate to the LP cohort that received the charge. An LP can therefore:
- hold shares when the VPI charge is credited;
- exit through `TrancheVault.requestRedeem`/`redeem`/`claim` before the rebate is paid;
- let the trader close; the rebate is paid from the remaining pool cash; the remaining LP's claim falls below its deposit.

The VPI charge is bounded at 8 bps of depth at the 0.4 skew cap and telescopes over cumulative skew (`CfdMath`), so splitting accounts/orders or using same-block/multi-epoch does not bypass the cap. The 1-hour LP epoch grid and 3-hour cycle bound repeatability, but repeated cycles each create a fresh charge/rebate and repeat the transfer; there is no global clamp.

Release-parameter PoC: `poc/P1-CP-04_escalation.t.sol`, 6/6 PASS with:
`forge test --match-path test/perps/spec/poc_step6/P1-CP-04_escalation.t.sol -vv`
It uses real `HousePool`, `TrancheVault`, `CfdEngine`, `CfdEngineSettlementSidecar` and `MarginClearinghouse`; only USDG and Pyth are test mocks; no in-scope source is modified.

At the documented $2M junior-only fixture: victim loss $755.18/cycle, $6,041/day; victim principal $623.39/cycle below deposit. The direct released 1M Junior + 1M Senior split gives: victim loss $753.888612/cycle (0.1839%), $6,031.11/day (1.4710%); principal loss $622.164145/cycle (0.1517%), $4,977.31/day (1.2140%); attacker net $301.186610/cycle. The $10M and $50M stress rows are in the run record. The $79,965.44 program "Vault TVL" is the Immunefi reward vault, not protocol liquidity; severity is assessed against the documented $2M launch depth.

Known-issue distinction:
- The AI scan finding "temporary depth vs VPI" describes VPI charge/rebate mechanics and depth effects, not the cross-cohort funding after an LP exit.
- The `SECURITY.md` VPI limitations section describes VPI mechanics, the lifetime clamp and zero-depth behavior; it does not cover the rebate being funded by a different LP cohort.
- Cecuro MED-22 is carry accounting, not VPI.
- `WHITEPAPER.md`/`SECURITY.md` state that a rebate comes from free pool cash, but do not state the harmful cross-cohort consequence and do not guarantee same-cohort funding. The finding is therefore neither KNOWN nor DOCUMENTED under the program's strict criterion.

## Impact Details

Victim: the remaining junior LP. The victim's claim falls below its deposited principal; the loss is not merely unclaimed yield. Attacker: the exiting LP (and the trader, if the same actor) receives the pro-rata charge share and exits before the rebate; the attacker's net is ~$301.19/cycle at the documented $2M split. The transfer repeats every ~3 hours (up to 8 cycles/day). The principal drain is ~0.61%/day in the junior-only $2M fixture and ~1.21%/day in the 1M/1M split; the loss-vs-control is ~1.47%/day in the split. Pause/hold does not compensate the cohort; `recapitalizePool` does not make the victim whole; the paid rebate is not clawable.

Claimed severity: Medium. The impact line is direct theft of LP principal, but the release-parameter magnitude is bounded by the lifetime clamp, the pool-level free-assets cap and the 1M/1M split; there is no single-cycle protocol insolvency. Severity is assessed against the documented $2M launch depth. The PoC uses the release-parameter file; the legacy `vpiFactor=1e18` fixture is not used for severity.

## References

- `packages/perps/src/HousePool.sol` — `recordClaimantInflow`, `payOut`, `settleLpEpoch`, free-assets cap.
- `packages/perps/src/TrancheVault.sol` — `requestRedeem`, `redeem`, `claimRedeem`, `claimRedeemRefund`.
- `packages/perps/src/CfdEngineSettlementSidecar.sol` — VPI charge/rebate; `CfdEnginePlanLib.sol` — VPI planning.
- `packages/perps/DEPLOYMENT.md:131-142` — `vpiFactor = 0.01e18`, `maxSkewRatio = 0.4e18`, execution fee.
- `packages/perps/SECURITY.md` — VPI limitations, known limitations.
- `audits/scan-plether-fi-plether-core-d4541b49.md` — AI scan finding "temporary depth vs VPI".
- `audits/plether-ai-audit-report-2026-10-26.pdf` — Cecuro MED-22 (carry rounding).
- `WHITEPAPER.md`, `SECURITY.md` — free-cash rebate language.
- PoC: `poc/P1-CP-04_escalation.t.sol`; run record: `poc/STEP6_release_validation_run_record.txt`.
- Secret gist: https://gist.github.com/kentavr111/5981da4f9089ea2515c366b3139757aa

## Link to Proof of Concept

https://gist.github.com/kentavr111/5981da4f9089ea2515c366b3139757aa

## Proof of Concept

## Proof of Concept

Pinned repo: `immunefi-team/audit-comp-plether` @ `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (`packages/perps`).
Release params: `vpiFactor = 0.01e18`, `maxSkewRatio = 0.4e18`, `executionFeeBps = 4`, documented launch depth $2M (1M Junior + 1M Senior) — `DEPLOYMENT.md:131-142`.

Reproduce:
1. Copy the secret-gist PoC file `P1-CP-04_escalation.t.sol` to `test/perps/spec/poc_step6/P1-CP-04_escalation.t.sol`.
2. Run: `forge test --match-path test/perps/spec/poc_step6/P1-CP-04_escalation.t.sol -vv`
3. Expected: 6/6 PASS (0 failed, 0 skipped); no in-scope source modified; mocks only USDG and Pyth.

Tests and measured results:
- `test_P1CP04_Escalation_ReleaseParams_Documented2M_OneCycle` — documented $2M junior-only depth; victim loss $755.18/cycle, $6,041/day; principal below deposit $623.39/cycle.
- `test_P1CP04_Escalation_ReleaseParams_DirectSplit_1MJunior_1MSenior_OneCycle` — released 1M/1M split; B ($410,000 deposit) loses $753.888612/cycle (0.1839%) vs the A-stays control and is $622.164145 (0.1517%) below its deposit after one cycle; 8 cycles/day = $6,031.11 (1.4710%) loss, $4,977.31 (1.2140%) principal/day; attacker net $301.186610/cycle.
- `test_P1CP04_Escalation_ReleaseParams_Mid10M_OneCycle` — $10M stress; same direction, clamp and free-assets cap bind.
- `test_P1CP04_Escalation_ReleaseParams_Stress50M_TwoCycles` — $50M stress, two cycles; fresh charge/rebate each cycle; no global clamp.
- `test_P1CP04_Escalation_ReleaseParams_ActualTVL_OneCycle` — $79,965.44 reward-vault proxy; not protocol liquidity; $29.51/cycle (~$236/day).
- `test_P1CP04_Escalation_ReleaseParams_ActualTVL_10pctSkew_NoPrincipalLoss` — signed low-skew negative control; B remains +$0.880940 above deposit, showing the 40% max-skew condition.

Each test builds the real `HousePool`/`TrancheVault`/`CfdEngine`/`CfdEngineSettlementSidecar`/`MarginClearinghouse` stack, seeds the LP cohorts, opens a LONG at the 0.4 max-skew VPI charge, exits LP A before the close/rebate, closes the position and compares the remaining LP B's claim against the A-stays control.

The per-position lifetime clamp caps the rebate at the position's lifetime charge but does not bind the rebate to the LP cohort that received the charge. The 8 bps VPI cap/telescoping over cumulative skew, the per-position clamp, the pool-level free-assets cap and the 1-hour LP epoch grid bound the per-cycle magnitude; multi-account, same-block and multi-epoch attempts do not bypass.

The secret gist contains the exact test and the raw forge output/materiality table:
https://gist.github.com/kentavr111/5981da4f9089ea2515c366b3139757aa
