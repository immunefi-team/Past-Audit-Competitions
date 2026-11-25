# \#99679 [SC-Low] Same-batch recapitalization collapses the Senior high-water mark and misroutes impaired-Senior revenue to Junior LPs

**Submitted on Nov 14th 2026 at 19:32:06 UTC by @crypt0n1337 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99679
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Summary
After a full HousePool wipeout, if a **recapitalization and queued trading revenue settle in the same reconcile batch** — the canonical degraded-mode recovery (`recapitalizePool` → `clearDegradedMode` → first permissionless `settleLpEpoch`) — the pool (1) **overwrites the Senior high-water mark downward** to the recapitalization amount and (2) **routes the queued trading revenue to Junior** even though Senior is still impaired below its true pre-wipe high-water mark.

The identical cash, settled revenue-first (a separate batch), correctly restores Senior and preserves the high-water mark. **Batch composition alone decides tranche ownership and the Senior recovery target.** This violates three documented invariants:
- `SECURITY.md:394` — "High-water-mark protection | **Senior impairment must be restored before junior extracts surplus**".
- `ACCOUNTING_SPEC.md:337` — "**Losses do not reduce the Senior high-water mark.**"; `:338` — "Route surplus first to restore Senior principal to its high-water mark, then to Junior."
- `WHITEPAPER.md:772` — "**Revenue restores impaired senior principal to its high-water mark before junior participates.**"

## Root cause
In `applyClaimantRecapitalizationIntent` (`HousePoolPendingPreviewLib.sol:111-137`), the post-wipe bootstrap branch runs when `seniorSupply > 0 && seniorPrincipal == 0 && juniorPrincipal == 0`:
```solidity
state.waterfall.seniorPrincipal += remaining;                          // :119
state.waterfall.seniorHighWaterMark =                                   // :120-121  <-- OVERWRITE
    recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
```
It **unconditionally overwrites** `seniorHighWaterMark` with `max(recapTarget, amount)`, discarding the preserved pre-wipe value `H0`. This directly contradicts the loss path, which deliberately keeps the mark: `HousePoolWaterfallAccountingLib.sol:161-184` ("The senior high-water mark is deliberately unchanged").

Then `applyPendingClaimantBucketsPreview` (`:46-56`) flips the revenue-continuation flag true because principal is now non-zero (`:50-52`), and `applyRevenueIntent(state, amount, true)` (`:158-188`) computes `gap = seniorHighWaterMark - seniorPrincipal`. Because the bootstrap set `seniorPrincipal == seniorHighWaterMark == R`, `gap == 0`, so the whole revenue is credited to Junior (`:180-183`).

## Reachability (release config)
1. A tail event drives terminal equity ≤ 0. `absorbLoss` saturates both principals to 0 and **preserves** `seniorHighWaterMark = H0` (e.g. H0 = the pool's pre-wipe Senior size). Seed shares keep both supplies non-zero. A wipeout latches degraded mode (`CfdEngine.sol:1350-1368`).
2. While degraded, trader losses / fees / carry / liquidation proceeds queue as `pendingTradingRevenueAssets` (`HousePool.sol:646-648`). Keeper reconciles are blocked (`settleLpEpoch` reverts while degraded, `HousePool.sol:832,1864-1870`), so the revenue cannot be routed to Senior first.
3. The owner performs the **documented** recovery: `recapitalizePool(R)` with `R < H0` (partial recovery of a large pool — the normal case), then `clearDegradedMode()`.
4. Any LP/keeper calls `settleLpEpoch`. The first reconcile applies recapitalization and all accumulated revenue in **one** `_applyPendingClaimantBucketsLive` pass → HWM collapses to R, revenue goes to Junior.

The trigger is "a legitimate, documented action by a trusted role (`recapitalizePool`) that triggers a bug harming users", which the program scope places **in scope**. No attacker is required; a Junior LP is the passive beneficiary and can redeem the misrouted revenue at the next epoch.

## Harm (release-scale example)
Pre-wipe Senior principal = HWM = 10,000,000 USDG (within the 40M / 80% caps); Junior 2,500,000. A tail event wipes both. 300,000 USDG of trader losses/fees/carry queue as revenue while degraded. The owner recapitalizes 1,000,000.
- **Buggy (same batch)**: Senior P = HWM = 1,000,000; Junior P = 300,000. Senior should hold 1,300,000 and retain a 10,000,000 recovery target.
- Senior LPs lose the 300,000 of revenue immediately, **and** 9,000,000 of restoration priority: every future surplus up to that gap now accrues to Junior instead of restoring Senior.
- Junior LPs receive 300,000 (and all future misrouted surplus) of yield the waterfall owes to impaired Senior.

## Severity
- **Primary — High, "Theft of unclaimed yield"**: queued trading revenue (unclaimed yield) owed by the waterfall to impaired Senior principal is diverted to Junior owners; the HWM collapse makes the diversion recurring for all future surplus up to `H0 − R`.
- **Fallback — Low, "Contract fails to deliver promised returns"**: if triage holds that value staying inside the pool (wrong tranche, no external exfiltration) is not "theft", the Senior tranche is still denied the restoration the spec promises.
- Griefing (Medium) is a weaker fit (there is a beneficiary, Junior, not pure damage).

## Proof of Concept

## Proof of concept
Two Foundry suites, release solc 0.8.35 / via-IR. **All pass.**

**End-to-end, through the live `HousePool.reconcile()` path** — `pocw2/test/HwmRecapEndToEnd.t.sol` (subclasses the repo's own `HousePoolAccountingParityBase` from `test/perps/differential/HousePoolSnapshotParity.t.sol`; copy into `packages/perps/test/perps/poc/` with import `harn/differential/HousePoolSnapshotParity.t.sol`):

| Test | Pre-wipe HWM (H0) | After same-batch recap 40k + revenue 15k | Verdict |
|---|---|---|---|
| `test_e2e_SameBatchCollapse` | 101,000,018,264 | HWM → **40,000,000,000** (collapsed); Junior → **15,000,000,000**; Senior = 40,000,000,000 (impaired) | **BUG** |
| `test_e2e_control_RevenueFirst` | 101,000,018,264 | revenue-first: Senior 15,000e6 then 55,000e6; Junior **0**; HWM stays **101,000,018,264** | correct |
| `test_e2e_maskedRegression_HwmRisesNotFalls` | 1,000,009,132 (repo default seed) | HWM → 40,000e6 (a **RISE**); Junior 15,000e6 | shows why the repo suite misses it |

`H0` is captured live from `pool.seniorHighWaterMark()`, not hardcoded. The waterfall math in `HousePoolPendingPreviewLib` is configuration-independent; the only precondition is **pre-wipe Senior HWM > recapitalization amount**, which is the ordinary partial-recovery case.

**The project's own regression masks the bug.** `test/perps/differential/HousePoolSnapshotParity.t.sol::test_MixedPendingInflowsPreserveRecapitalizationPriorityAfterWipe` (line 135) runs the exact same `_wipePrincipal()` → recap 40k → revenue 15k → live `pool.reconcile()` sequence and asserts `ExpectedAccounting(40_000e6, 15_000e6, 40_000e6)` with the comment *"same-batch revenue then belongs to surviving junior owners"* (line 140). But it seeds Senior at only `1000e6` (`_initialSeniorSeedDeposit`, below the 40,000e6 recap), so the final HWM of 40,000e6 is a benign **rise** — the downward collapse is never exercised. `test_e2e_maskedRegression_HwmRisesNotFalls` reproduces this exactly.

(A configuration-independent library-level suite `pocw2/test/HwmRecapMisroute.t.sol` drives `HousePoolPendingPreviewLib.applyPendingClaimantBucketsPreview` directly over a post-`absorbLoss` state and passes 3/3 with the same before/after numbers.)

### Run
```
cd pocw2 && forge test --offline --match-path test/HwmRecapEndToEnd.t.sol -vv
```
Result: `Suite result: ok. 3 passed; 0 failed; 0 skipped`.

### Test source (end-to-end, live HousePool.reconcile())
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolAccountingParityBase} from "harn/differential/HousePoolSnapshotParity.t.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

/// @title HwmRecapEndToEnd
/// @notice Level-(B) end-to-end PoC through the live HousePool.reconcile() path. Shows that the same canonical recovery
///         sequence the repo's own regression exercises (test/perps/differential/HousePoolSnapshotParity.t.sol::
///         test_MixedPendingInflowsPreserveRecapitalizationPriorityAfterWipe) collapses the Senior high-water mark and
///         misroutes revenue to Junior once the PRE-WIPE Senior HWM exceeds the recapitalization amount. The repo test
///         seeds Senior at only 1000e6 (below the 40_000e6 recap), so it only ever shows the benign HWM *rise* and the
///         regression stays masked. Violates SECURITY.md:394, ACCOUNTING_SPEC.md:337-338, WHITEPAPER.md:772.
abstract contract HwmRecapEndToEndBase is HousePoolAccountingParityBase {
    uint256 internal constant RECAP = 40_000e6;
    uint256 internal constant REVENUE = 15_000e6;

    /// @dev Burn every asset and reconcile as the junior vault: a total loss that wipes both principals to zero while
    ///      the loss waterfall (HousePoolWaterfallAccountingLib.absorbLoss) preserves the Senior HWM. Mirrors the repo's
    ///      HousePoolSnapshotParityTest._wipePrincipal exactly.
    function _wipePrincipal() internal returns (uint256 hwmBeforeWipe) {
        hwmBeforeWipe = pool.seniorHighWaterMark();
        settlementToken.burn(address(pool), pool.totalAssets());
        vm.prank(address(juniorVault));
        pool.reconcile();
        assertEq(pool.seniorPrincipal(), 0, "wipe: senior principal 0");
        assertEq(pool.juniorPrincipal(), 0, "wipe: junior principal 0");
        assertGt(seniorVault.totalSupply(), 0, "senior ownership survives the wipe");
        assertGt(juniorVault.totalSupply(), 0, "junior ownership survives the wipe");
        assertEq(pool.seniorHighWaterMark(), hwmBeforeWipe, "wipe: HWM preserved (losses do not reduce HWM)");
    }

    function _reconcileAsJunior() internal {
        vm.prank(address(juniorVault));
        pool.reconcile();
    }

    function _logTranche(string memory tag) internal {
        emit log_string(tag);
        emit log_named_uint("  seniorPrincipal", pool.seniorPrincipal());
        emit log_named_uint("  juniorPrincipal", pool.juniorPrincipal());
        emit log_named_uint("  seniorHighWaterMark", pool.seniorHighWaterMark());
    }
}

/// @notice Pre-wipe Senior HWM ABOVE the recap -> the live reconcile exposes the collapse.
contract HwmRecapEndToEndTest is HwmRecapEndToEndBase {
    // Senior deposit well above the 40_000e6 recap. The harness relaxes caps in _bypassAllTimelocks
    // (maxSeniorShareBps = 9999, maxSeniorExposureAssets ~= type(uint256).max), so 100_000e6 seats.
    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 100_000e6;
    }

    function test_e2e_SameBatchCollapse() public {
        uint256 h0 = pool.seniorHighWaterMark();
        emit log_named_uint("pre-wipe seniorPrincipal", pool.seniorPrincipal());
        emit log_named_uint("pre-wipe seniorHighWaterMark (H0)", h0);
        assertGt(h0, RECAP, "precondition: pre-wipe HWM exceeds the recap amount");

        uint256 hwmBeforeWipe = _wipePrincipal();
        _logTranche("after wipe (0/0, HWM preserved)");
        assertEq(hwmBeforeWipe, h0, "HWM unchanged by wipe");

        // Canonical recovery: recapitalization + queued trading revenue settle in ONE reconcile batch.
        _recordInflow(IHousePool.ClaimantInflowKind.Recapitalization, RECAP);
        _recordInflow(IHousePool.ClaimantInflowKind.Revenue, REVENUE);
        _reconcileAsJunior();
        _logTranche("after same-batch recap+revenue reconcile");

        // COLLAPSE observed through the live pool.reconcile():
        assertEq(pool.seniorHighWaterMark(), RECAP, "LIVE BUG: HWM collapsed to recap amount");
        assertLt(pool.seniorHighWaterMark(), h0, "LIVE BUG: HWM fell below pre-wipe entitlement");
        assertEq(pool.juniorPrincipal(), REVENUE, "LIVE BUG: revenue routed to Junior");
        assertEq(pool.seniorPrincipal(), RECAP, "LIVE BUG: senior holds only recap");
        assertLt(pool.seniorPrincipal(), h0, "LIVE BUG: Senior impaired below true HWM while Junior is paid");
    }

    function test_e2e_control_RevenueFirst() public {
        uint256 h0 = pool.seniorHighWaterMark();
        assertGt(h0, RECAP, "precondition: pre-wipe HWM exceeds the recap amount");
        _wipePrincipal();

        // Batch 1: revenue only, in its own reconcile.
        _recordInflow(IHousePool.ClaimantInflowKind.Revenue, REVENUE);
        _reconcileAsJunior();
        _logTranche("control: after revenue-only reconcile");
        assertEq(pool.seniorPrincipal(), REVENUE, "CTRL: revenue restores Senior");
        assertEq(pool.juniorPrincipal(), 0, "CTRL: Junior gets nothing while Senior impaired");
        assertEq(pool.seniorHighWaterMark(), h0, "CTRL: HWM preserved at H0");

        // Batch 2: recapitalization, in a later reconcile.
        _recordInflow(IHousePool.ClaimantInflowKind.Recapitalization, RECAP);
        _reconcileAsJunior();
        _logTranche("control: after recap reconcile");
        assertEq(pool.seniorPrincipal(), REVENUE + RECAP, "CTRL: senior principal rises (15k+40k)");
        assertEq(pool.juniorPrincipal(), 0, "CTRL: Junior stays 0");
        assertEq(pool.seniorHighWaterMark(), h0, "CTRL: HWM still preserved at H0");
    }
}

/// @notice The repo's own fixture: Senior seeded at only 1000e6 (default), BELOW the recap. The IDENTICAL same-batch
///         sequence drives the final HWM to the same 40_000e6, but here that is a RISE from the tiny seed, so the
///         regression test never reveals the downward collapse. Same recap, opposite direction.
contract HwmRecapMaskedRegressionTest is HwmRecapEndToEndBase {
    // No _initialSeniorDeposit override: pre-wipe Senior principal is just the 1000e6 senior seed (repo fixture).

    function test_e2e_maskedRegression_HwmRisesNotFalls() public {
        uint256 h0 = pool.seniorHighWaterMark();
        emit log_named_uint("low-seed pre-wipe seniorHighWaterMark (H0)", h0);
        assertLt(h0, RECAP, "repo fixture: pre-wipe HWM is BELOW the recap");

        _wipePrincipal();
        _recordInflow(IHousePool.ClaimantInflowKind.Recapitalization, RECAP);
        _recordInflow(IHousePool.ClaimantInflowKind.Revenue, REVENUE);
        _reconcileAsJunior();
        _logTranche("low-seed: after same-batch recap+revenue reconcile");

        assertEq(pool.seniorHighWaterMark(), RECAP, "same final HWM 40_000e6 as the high-seed case");
        assertGt(pool.seniorHighWaterMark(), h0, "but here it is a RISE -> the collapse stays masked at low seed");
        assertEq(pool.juniorPrincipal(), REVENUE, "revenue routed to Junior (same as high-seed)");
        assertEq(pool.seniorPrincipal(), RECAP, "senior holds only recap (same as high-seed)");
    }
}

```
