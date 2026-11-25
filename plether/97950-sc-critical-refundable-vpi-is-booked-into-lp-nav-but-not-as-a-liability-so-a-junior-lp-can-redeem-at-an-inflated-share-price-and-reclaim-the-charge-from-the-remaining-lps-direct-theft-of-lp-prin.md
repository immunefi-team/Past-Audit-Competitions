# \#97950 [SC-Critical] Refundable VPI is booked into LP NAV but not as a liability, so a Junior LP can redeem at an inflated share price and reclaim the charge from the remaining LPs (direct theft of LP principal)

**Submitted on Nov 4th 2026 at 02:59:05 UTC by @risinghorse for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97950
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief / Intro

Plether's Virtual Price Impact (VPI) skew charge is added to pool assets and to Junior LP principal the instant it is paid at open, but the paying position keeps the right to receive it back on a later skew-reducing close (bounded by its lifetime accrual). The NAV used to price LP redemptions and deposits (`HousePoolAccountingLib.buildReconcileSnapshot`) contains no offsetting liability for that outstanding, refundable VPI, and no receivable for a parked negative-VPI rebate. A party that holds Junior LP shares and also controls a trading account can therefore: (1) pay VPI to inflate NAV just before an epoch boundary, (2) redeem Junior shares at that inflated NAV, and (3) close the position to pull the VPI back out of the pool — which now comes from the LPs who stayed. Because closes are not subject to any skew cap, a delta-neutral hedged pair can repeat this each epoch and drain a victim LP's principal. Senior is unaffected; the loss falls on passive Junior LPs and is split between the attacker and the protocol treasury's fees.

## Vulnerability Details

**Where VPI enters NAV (and is never removed as a liability).**
- On open, the positive VPI charge is recorded as pool cash revenue through `recordClaimantInflow(..., Revenue, CashArrived)` ([`CfdEngineSettlementSidecar.sol:454-464`](../../../src/CfdEngineSettlementSidecar.sol)), which increases `accountedAssets` and, at the next reconcile, Junior principal.
- LP entry and exit are both priced from `buildReconcileSnapshot` ([`libraries/HousePoolAccountingLib.sol:60-78`](../../../src/libraries/HousePoolAccountingLib.sol)): `distributable = physicalAssets − traderClaims (± terminal delta)`. There is **no term** for the sum of outstanding positive `vpiAccrued` (a pool liability, because it is refundable) and **no term** for the parked negative-VPI reserve (a pool receivable, because it will be clawed back).

**Why the charge is refundable.**
- A skew-reducing close computes a signed VPI delta and is clamped only so that lifetime VPI never goes negative ([`libraries/CloseAccountingLib.sol:107-118`](../../../src/libraries/CloseAccountingLib.sol)). A position that paid `V` at open can therefore receive up to `V` back at close.
- The resulting rebate is paid out of free pool cash ([`libraries/CfdEnginePlanLib.sol:968-978`](../../../src/libraries/CfdEnginePlanLib.sol) and [`CfdEngineSettlementSidecar.sol:621-624`](../../../src/CfdEngineSettlementSidecar.sol)).

**Why it is repeatable and large.**
- The skew cap is enforced on opens only ([`libraries/CfdEnginePlanLib.sol:473-480`](../../../src/libraries/CfdEnginePlanLib.sol)); `_planValidatedClose` has no skew check. A delta-neutral pair (one long, one short of equal size) can therefore close one leg fully — swinging the booked VPI by `≈ 0.5·k·N²/D` — then reopen it, with no cap binding.
- Outside the frozen regime there is no LP entry or exit fee: `_settlementFeeBps` returns 0 ([`HousePool.sol:1147-1155`](../../../src/HousePool.sol)), so the round trip costs only the execution fee and gas.
- The epoch cutoff and the 1-hour deposit cooldown pace the cycle at roughly three hours; nothing prevents repetition.

**The sequence the PoC performs (one cycle):**
1. The attacker holds Junior LP shares (and, in the hedged form, a delta-neutral pair opened earlier).
2. It requests a Junior redemption targeting the next hourly epoch boundary.
3. Seconds before the boundary, it makes NAV high: a skew-reducing close of one hedged leg books a large positive VPI into pool assets (or, in the single-sided form, a skew-creating open pays VPI in).
4. A permissionless `settleLpEpoch` prices the redemption at the inflated NAV, and the attacker's shares are funded at that price.
5. The attacker reverses the trade (reopen / close), pulling the VPI back out of the pool via the lifetime-clamped rebate. That cash now comes from the LPs still in the tranche.
6. It re-deposits the redeemed assets (in the hedged form, at a NAV depressed by the parked rebate) and repeats.

A causal control confirms the mechanism: if the position is closed **before** settlement prices the redemption, or if the attacker holds **no** LP shares, the same trades leave the attacker with a net loss and the victim unharmed (falsifier tests below). A test-only NAV fix that prices the refundable VPI on both sides turns the attacker's result negative and leaves the honest control unchanged, confirming the root cause (see References).

## Impact Details

All figures are wallet-level USDG, measured as attack-minus-control from a single snapshot, with every party fully exited, on the strict release fixture (production `OrderRouter`, six release feeds, release parameters asserted on-chain). Conservation holds to the atom (attacker gain + treasury fees = victim loss; Senior unchanged).

- **Headline (hedged, 62% of Junior):** pool of 30M Senior + 20M Junior, 4 cycles over ~13 hours. Attacker **+197,692**; victim **−361,960** on a 7,600,000 deposit (cash exit 7,236,478, below principal); treasury +164,267; Senior +0.25.
- **Sustained:** same book, 32 cycles Monday–Friday (no FAD/frozen window): attacker **+1,710,289**; victim **−2,682,845 (−35.3% of principal)**; treasury +972,538; Senior +12.
- **Single-sided basic form** (1M Senior / 1M Junior, attacker 3M): attacker **+1,346/cycle**; remaining Junior **−2,928/cycle**.
- **Depth not required:** with a third-party deposit keeping depth unchanged at the close, the attacker still nets **+1,380** and a late depositor overpays — so this is not the known depth-manipulation info item.
- **Robustness:** profitable on 40/40 random FX-noise paths; still profitable at the 10 bps confidence ceiling; works with the basket above and below 1.00 and on either side.
- **Break-even:** roughly 17–25% of Junior, depending on form; below that the attacker's one-off costs dominate.

Impact row: **Direct theft of user funds at rest** — the victim's realized cash falls below its deposit, so the loss reaches Junior principal, not merely yield. Senior tranche value and protocol solvency are never impaired, so this is a Junior-LP-to-attacker transfer, not insolvency.

**Honest scope limits (stated for the reviewer):**
- A victim that redeems in the **same epoch** as the attacker is funded pro-rata at the same inflated NAV and is made whole for that cycle; the loss falls on **passive** LPs that do not co-exit within the ~65-second boundary window.
- The headline figures assume all LPs exit together; if a third-party Senior stays, the Junior covenant caps how much of the attacker's own stake it can realize as cash (the documented Senior-exposure covenant).
- No trusted role is involved; all steps use public entrypoints, and keepers are permissionless.

## References

- `packages/perps/src/libraries/HousePoolAccountingLib.sol:60-78` — NAV snapshot used for LP entry/exit; omits the refundable-VPI term.
- `packages/perps/src/CfdEngineSettlementSidecar.sol:454-464`, `:621-624` — open charge booked as revenue; close rebate paid from the pool.
- `packages/perps/src/libraries/CloseAccountingLib.sol:107-118` — lifetime clamp that keeps the charge refundable.
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:473-480` (skew cap on opens only), `:968-978` (rebate from free cash).
- `packages/perps/src/HousePool.sol:1147-1155` — zero LP fee outside the frozen regime.
- Design context (for the "documented intent" discussion): WHITEPAPER.md §5.1 and the "VPI limitations" subsection of SECURITY.md describe VPI as pool revenue and the lifetime clamp as preventing *trader-side* rebate round trips; neither states that an LP can exit at a NAV holding a charge that is then rebated from the remaining LPs. The accepted collection-time-ownership policy (ACCOUNTING_SPEC.md) applies to carry, which is never refunded.
- Known-issue check: not present in the Cecuro AI audit, SC pre-audit or AI scan. Closest is the AI scan's info item "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate" (`scan.md:1742`), which concerns a trader enlarging its *own* rebate by moving depth; the unchanged-depth test here shows depth manipulation is **not** required, and the gain runs through LP exit pricing, so the root cause differs.

## Proof of Concept

## Proof of Concept

The PoC runs in the repository's Foundry suite at the in-scope commit, without modifying any in-scope contract. Only Pyth and USDG are mocked (both out of scope). All attacker steps use public entrypoints and EOA signers; there is no `vm.store`, `vm.etch`, `mockCall`, or prank as router/engine/pool/vault.

**Files** (under `packages/perps/test/audit_poc/`):
- `vpi_nav/VpiNavPumpStrict.t.sol` with `vpi_nav/VpiStrictBase.sol`, `vpi_nav/StrictPyth.sol` — single-sided form, falsifiers, capital/confidence/noise sweeps, step trace.
- `vpi_hedged/StrictLpVpi.t.sol` with `vpi_hedged/StrictReleaseBase.sol` — the hedged multi-cycle headline and long run.

**Run:**
```bash
cd packages/perps && "$HOME/.foundry/bin/forge" test --match-path "test/audit_poc/vpi_nav/VpiNavPumpStrict.t.sol" --skip "test/perps/**/*.t.sol" --skip "script/**" -vv
```
```bash
cd packages/perps && "$HOME/.foundry/bin/forge" test --match-path "test/audit_poc/vpi_hedged/StrictLpVpi.t.sol" --skip "test/perps/**/*.t.sol" --skip "script/**" -vv
```

**Key tests and what they show:**
- `test_Strict_Hedged_62Pct_4Cycles_ConstantPrice` — the headline: attacker +197,692, victim −361,960, Senior flat.
- `test_Strict_Hedged_LongRun_Monday_To_Friday` — 32 cycles, victim −35.3%.
- `test_Strict_50_50` / `test_Strict_Repeat_24h_50_50` — single-sided form and its daily repetition.
- `test_Strict_UnchangedDepthAtClose` — the gain survives with depth held constant (distinguishes it from `scan.md:1742`).
- `test_Strict_Falsifier_CloseBeforeSettlement`, `test_Strict_Falsifier_NoLpLeg` — the causal controls: without pricing the redemption while the VPI is outstanding, or without an LP leg, the attacker only pays costs and no LP is harmed.
- `test_Strict_CapitalSweep`, `test_Strict_ConfidenceSweep`, `test_Strict_NoisePaths` — break-even stake, behaviour to the 10 bps confidence ceiling, and 40/40 profitable noise paths.

**Root-cause confirmation (not part of the PoC):** `vpi_nav/vpi-nav-root-cause-check.test-only.patch` adds a refundable-VPI liability/receivable term to the reconcile NAV. Applied to a throwaway copy of `src`, the headline attacker result flips from +197,692 to −533,026 and the victim from −361,960 to +368,758, while the honest control is unchanged. This is offered only as evidence of the root cause; it over-corrects and is not a production-ready fix.

### Suggested remediation (directional)

Price the outstanding refundable VPI symmetrically in the NAV used for LP entry and exit: treat the aggregate positive `vpiAccrued` (skew potential banked in the pool) as a liability and the parked negative-VPI reserve as a receivable, so that a skew-reducing close that will refund VPI does not inflate the share price a redeeming LP receives. Alternatively, defer recognition of VPI into LP NAV until it is no longer refundable.
