# \#97978 [SC-Low] Recapitalization after a total tranche wipe replaces the surviving Senior high-water mark with the recap size (knife-edge on juniorPrincipal == 0), silently handing the restoration entitlement to first-loss Junior LPs

**Submitted on Nov 4th 2026 at 07:14:16 UTC by @Matgothmog for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97978
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief

When a loss wipes both tranche principals to zero while Senior LP shares still exist, the Senior high-water mark (HWM) survives the loss — it is the wiped Senior LPs' restoration entitlement. But when governance then recovers the pool through `CfdEngine.recapitalizePool`, the reconciliation's bootstrap branch **overwrites the surviving HWM with `max(recapIntent, settledAmount)`**, destroying any entitlement above the recap size. Recovery value that the waterfall owes to Senior then flows to first-loss Junior instead.

The outcome flips on an exact-zero knife edge: an otherwise identical recap with `juniorPrincipal == 1` atom takes the other branch and **preserves** the HWM. Two branches of the same function disagree about whether the pre-wipe entitlement survives, decided by a single atom of Junior principal.

## Vulnerability Details

All references are against the in-scope code at commit `a82d4fef`.

**1. Losses preserve the Senior HWM.** `HousePoolWaterfallAccountingLib.absorbLoss` (`libraries/HousePoolWaterfallAccountingLib.sol:167-185`) saturates `seniorPrincipal`/`juniorPrincipal` to zero but deliberately leaves `seniorHighWaterMark` untouched. Per `ACCOUNTING_SPEC.md` (Senior/Junior waterfall rules 3-4): "Losses do not reduce the Senior high-water mark"; "Route surplus first to restore Senior principal to its high-water mark, then to Junior." The recapitalization contract is "intended to restore waterfall claimants" and, per the spec's senior-exposure section, "may restore an existing protected senior entitlement".

**2. The bootstrap branch replaces it.** On the next reconcile, the pending claimant routing reaches `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent` (`libraries/HousePoolPendingPreviewLib.sol:111-141`):

```solidity
if (state.seniorSupply > 0) {
    if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
        state.waterfall.seniorPrincipal += remaining;
        state.waterfall.seniorHighWaterMark =
            recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
        remaining = 0;
    } else {
        uint256 gap = ... seniorHighWaterMark - seniorPrincipal ...;   // HWM preserved here
        ...
    }
}
```

When both principals are zero at routing time, the HWM is **assigned** (not raised) to the recap size. With a pre-wipe HWM of 50,000 and a 30,000 recap, 20,000 of restoration entitlement is destroyed.

**3. The knife edge.** The `else` branch (taken when `juniorPrincipal` is nonzero, even 1 atom) restores Senior only up to the preserved HWM gap and never touches the HWM. Identical wipe, identical recap, opposite entitlement outcome depending on whether `juniorPrincipal` is exactly 0 or ≥ 1 atom. (Note the branch state is the state at routing time: a Junior principal that is charged as a shortfall earlier in the same reconcile — e.g. because the loss left it unbacked — also reaches the bootstrap branch.)

**4. Concrete harm (PoC numbers).** Wipe: `seniorPrincipal = 0`, `juniorPrincipal = 0`, `seniorHighWaterMark = 50,000` (surviving entitlement, Senior shares outstanding). Governance recaps 30,000 via `CfdEngine.recapitalizePool` → `HousePool.recordClaimantInflow(30,000, Recapitalization, CashArrived)` → reconcile. Result: `seniorPrincipal = 30,000`, **`seniorHighWaterMark = 30,000`** (was 50,000). A subsequent 20,000 recovery surplus — which per rule 4 must restore Senior toward the 50,000 HWM before Junior earns anything — is then paid entirely to **Junior**, the first-loss tranche. With `juniorPrincipal = 1` atom in the identical setup, the HWM stays 50,000 and the same 20,000 surplus restores Senior instead. Both scenarios are asserted by the attached PoC.

**5. Secondary effect.** The Senior impairment gate `projectedSeniorPrincipal < projectedSeniorHighWaterMark` (`HousePoolTrancheGateLib.trancheDepositsAllowed`, `libraries/HousePoolTrancheGateLib.sol:35-42`) clears once principal meets the reduced HWM (30,000 ≮ 30,000), reopening Senior deposit admission that the preserved entitlement would keep closed until Senior is made whole.

## Impact

- Wiped Senior LPs (shares outstanding, worth zero at the wipe) lose their documented priority restoration entitlement: recovery value up to `pre-wipe HWM − recap size` is redirected to first-loss Junior LPs and to new Senior depositors admitted early. In the PoC scenario that is 20,000 on a 50,000 entitlement (40%); in general it is the entire gap between the surviving HWM and the recap size.
- The behavior contradicts the accounting specification's waterfall rules ("Losses do not reduce the Senior high-water mark"; surplus restores Senior first) and the recapitalization contract ("restore an existing protected senior entitlement"), and it is internally inconsistent with the adjacent branch of the same function.
- The reduced HWM also defeats the impairment gate, allowing new Senior deposits to dilute the wiped-but-entitled Senior holders' claim on recovery.

## Dedup / relationship to MED-12

The Cecuro MED-12 report describes the same wiped-state recapitalization flow (its worked example even outputs `seniorHighWaterMark = 40,000` after a 40,000 recap) but its claimed issue and its fix were about **revenue continuation being misrouted to `unassignedAssets`**; the fix (commit f852f969) recomputes `allowRevenueContinuation` after recapitalization and left the HWM assignment of `applyClaimantRecapitalizationIntent` unchanged. This report's root cause — the HWM **replacement** and its disagreement with the gap branch — is neither the MED-12 claim nor part of its remediation. The nearby `HousePool.assignUnassignedAssets` bootstrap reset is a different, owner-gated function whose "fresh senior bootstrap replaces the stale HWM baseline" semantics are asserted by `HousePoolWaterfallTest.test_AssignUnassignedAssets_ResetsSeniorHwmWhenSeniorIsEmptyButJuniorStillExists` for the case where **no senior claimant history exists**; nothing documents or tests the reset for the recap path, and the project's own independent reference model (`test/perps/invariant/ghost/WaterfallReferenceMath.sol:37-38`) only ever ratchets the HWM **upward**.

## Recommended Mitigation

Make the bootstrap branch preserve a surviving entitlement: set
`seniorHighWaterMark = max(existingHwm, recapitalizationTargetAssets, amount)` instead of `max(recapitalizationTargetAssets, amount)`, so a recap can raise but never reduce the HWM. Alternatively, gate the replacement on there being no pre-wipe Senior history (e.g. an HWM recorded before the wipe is absent), and make both branches agree on the rule.

## Link to Proof of Concept

https://gist.github.com/Matgothmog/678d8283dc5945b64e0692326806c464

## Proof of Concept

# Proof of Concept

Foundry regression tests against the frozen commit `a82d4fef` (Forge 1.5.1, solc 0.8.35 via-IR).
File: `poc/RecapBootstrapHwmReset.t.sol` (drop into `packages/perps/test/perps/regression/` and run
`forge test --root packages/perps --match-contract RecapBootstrapHwmReset -vv`).
Full run output: `poc/RUN_OUTPUT.txt` — **2/2 PASS**.

The tests use the project's own `HousePoolTestBase` harness and production entrypoints
(`HousePool.recordClaimantInflow` exactly as `CfdEngine.recapitalizePool` drives it, then
`HousePool.reconcile`). Only the pre-recap wiped state is set up directly (storage writes standing in
for the catastrophic loss, with the remaining junior principal physically cash-backed so the same
reconcile does not charge it as a shortfall).

Both tests run the **identical** wipe and the **identical** 30,000 USDG recapitalization against a
surviving Senior HWM of 50,000 with Senior shares outstanding. The only difference is whether
`juniorPrincipal` is exactly 0 or 1 atom.

**Scenario A — `test_RecapBootstrap_ZeroPrincipalWipe_ResetsSeniorHwmAndFeedsJunior` (PASS):**
`seniorPrincipal = 0`, `juniorPrincipal = 0`, `seniorHighWaterMark = 50,000` → recap 30,000 →
reconcile. Asserts:

- `seniorPrincipal == 30,000` (recap bootstraps Senior),
- `seniorHighWaterMark == 30,000` — **the 50,000 entitlement is destroyed**,
- a subsequent 20,000 recovery surplus goes entirely to Junior
  (`seniorPrincipal` unchanged, `juniorPrincipal == 20,000`): first-loss Junior captures the
  upside that waterfall rule 4 assigns to Senior restoration.

**Scenario B — `test_RecapBootstrap_OneAtomJunior_PreservesSeniorHwm` (PASS):**
identical, but `juniorPrincipal = 1` atom (cash-backed) → same recap → reconcile. Asserts:

- `seniorPrincipal == 30,000`,
- `seniorHighWaterMark == 50,000` — **the entitlement survives** (gap branch),
- the same 20,000 recovery surplus restores Senior first
  (`seniorPrincipal == 50,000`).

Run output:

```
[PASS] test_RecapBootstrap_OneAtomJunior_PreservesSeniorHwm() (gas: 2302820)
[PASS] test_RecapBootstrap_ZeroPrincipalWipe_ResetsSeniorHwmAndFeedsJunior() (gas: 2304695)
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

The pair demonstrates (a) the entitlement loss in the bootstrap branch and (b) the exact-zero knife
edge between the two branches of `applyClaimantRecapitalizationIntent`
(`HousePoolPendingPreviewLib.sol:111-141`). Deterministic, no fork, no oracle dependency (no open
positions; NAV is mark-independent for this flow).
