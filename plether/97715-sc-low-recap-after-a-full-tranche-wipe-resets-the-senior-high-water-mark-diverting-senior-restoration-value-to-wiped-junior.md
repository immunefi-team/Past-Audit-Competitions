# \#97715 [SC-Low] Recap after a full tranche wipe resets the Senior high-water mark, diverting Senior restoration value to wiped Junior

**Submitted on Nov 2nd 2026 at 19:37:40 UTC by @Furk_jr for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97715
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief / Intro

When both tranches have been fully wiped by trader losses and the owner recapitalizes the pool following the documented terminal-deficit runbook, `applyClaimantRecapitalizationIntent` takes the "empty bootstrap" branch and **overwrites** the Senior high-water mark (HWM) with `max(recapTarget, settledAmount)` instead of preserving the pre-loss HWM that `absorbLoss` deliberately kept. From then on, Senior is only "restored" up to the recap amount R; every unit of recovery/revenue above R flows to the Junior shares that were just wiped. The amount diverted from Senior to Junior is `H0 - R` (pre-loss Senior entitlement minus recap), which at release scale (Senior cap 40M USDG) can be millions of USDG. The unsettled part of the recap is also pushed into `unassignedAssets`, which blocks LP deposits until the owner assigns it.

## Vulnerability Details

**Location** (commit a82d4fefb02768e5b3bddcc97dee371eee24fea8)
- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:111-139`, `applyClaimantRecapitalizationIntent`, lines 118-121:
  ```solidity
  if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
      state.waterfall.seniorPrincipal += remaining;
      state.waterfall.seniorHighWaterMark =
          recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining; // overwrites the pre-loss HWM
      remaining = 0;
  }
  ```
- Reached from `HousePool._reconcile` -> `_applyPendingClaimantBucketsLive` (`HousePool.sol:1598`, `1759-1776`), which runs **after** `absorbLoss` (`HousePoolReconcilePlanLib.sol:103`) has zeroed both principals in the same reconcile.

**Root cause**
- `absorbLoss` intentionally keeps `seniorHighWaterMark` when Senior is wiped: SECURITY.md ("High-water-mark protection: Senior impairment must be restored before junior extracts surplus"), and the gap-restore branch of the same helper (`gap = HWM - principal`) relies on that preserved HWM.
- The bootstrap branch, however, is also entered when both principals are 0 **because of losses**, not because the pool is new. It then replaces the preserved HWM `H0` by `max(R, settled)`. The natspec calls this branch an "empty bootstrap", but with live Senior shares holding a pre-loss claim it is a loss of entitlement, not a bootstrap.
- Afterwards, recovery/revenue restores Senior only up to R (the new HWM); everything above goes to Junior, whose shares were already wiped.

**Steps (every step is a legitimate, documented action; no trusted role misbehaves)**
1. Traders hold a hedged book near the admission limit. An extreme two-sided move pays the long profit first, then marks short profit above the remaining LP equity -> terminal deficit, degraded mode.
2. The owner follows the SECURITY.md "explicit terminal-deficit cascade" runbook: `recapitalizePool(R)` then `clearDegradedMode()` (a documented operational procedure; per the competition rules, "a legitimate, documented action by a trusted role that triggers a bug harming users or the protocol" is in scope).
3. Anyone calls `settleLpEpoch`. In one reconcile both principals go to 0 (`absorbLoss`) and the HWM is reset from `H0` to `R`.
4. The market recovers / revenue accrues: Senior is restored only to R, Junior is credited the remainder, and the residual recap sits in `unassignedAssets` (LP deposits blocked until the owner assigns it).

## Impact Details

Harness numbers (pool ~3.02M USDG, Senior 2.0M, recap R = 1.4M):
- Senior HWM: 2,001,036 -> **1,400,000** USDG.
- After recovery: Senior holds 1.40M; the previously wiped Junior holds **1.39M**.
- 167,420 USDG of the recap stuck in `unassignedAssets`.
- **601,036 USDG owed to Senior by the waterfall goes to Junior** (= H0 - R). In general the diverted amount is `H0 - R` of future recovery/revenue; with the release Senior cap of 40M USDG this scales to millions.

Severity: Medium proposed - misallocation of LP funds between tranches that violates the documented waterfall invariant (Senior restored before Junior extracts surplus), plus an LP-deposit freeze until owner action. Likelihood is low (requires a total wipe of both tranches: a tail event for an FX basket, or the aftermath of another incident), hence not High.

## Recommended fix

Never lower the HWM on recapitalization; treat a post-wipe recap like the gap-restore path:
```solidity
uint256 target = recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
if (target > state.waterfall.seniorHighWaterMark) state.waterfall.seniorHighWaterMark = target;
```
or route a recap with both principals at 0 through the gap-restore branch (gap = preserved HWM - 0). Preview and live paths share this helper, so they stay in parity.

## Relation to known issues
- Cecuro MED-12 concerns the same helper but a different root cause (revenue routed to `unassignedAssets`); its fix (f852f96) did not change the HWM assignment.
- LOW-65 / MED-26 have different root causes. No SECURITY.md limitation covers an HWM reset on recapitalization.


## Proof of Concept

## Proof of Concept

File: `packages/perps/test/poc/RecapHwmReset.t.sol` (project's `BasePerpTest` harness: real HousePool, TrancheVaults, Clearinghouse and Engine; orders applied by pranking the Router into `engine.processOrderTyped`; marks set via `updateMarkPrice`; prices 1.00 -> 0.50 -> 1.60 to wipe both tranches).

Build note: a full `forge build` of packages/perps (via_ir) can exceed 8 GB RAM. Use:
```
cd packages/perps
forge build --skip test --skip script
FOUNDRY_TEST=test/poc forge test --match-path test/poc/RecapHwmReset.t.sol -vv
```
Output:
```
[PASS] test_RecapAfterWipeErasesSeniorHighWaterMark() (gas: 9816335)
  senior principal before 2001036566375
  senior HWM before       2001036566375
  junior principal before 1020963433625
  --- after recap reconcile ---
  senior principal 1232580002785
  senior HWM       1400000000000
  junior principal 0
  pending recap    167419997215
  --- after recovery reconcile ---
  senior principal 1400000000000
  senior HWM       1400000000000
  junior principal 1391417142747
  unassigned       167419997215
  value diverted from Senior to Junior 601036566375
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {console2} from "forge-std/console2.sol";

/// @notice PoC: owner recapitalization (documented recovery step) after a full tranche wipe overwrites the
///         Senior high-water mark with the recapitalization amount, erasing the Senior restoration entitlement.
///         Later revenue then flows to wiped Junior shares before Senior is restored to its pre-loss HWM.
contract RecapHwmResetPoC is BasePerpTest {

    address alice = address(0xA11CE); // long trader
    address bob = address(0xB0B); // short trader
    address juniorLp = address(0x1111);
    address juniorLp2 = address(0x2222);

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 2_000_000e6;
    }

    function _lots(
        uint256 notionalAssets
    ) internal pure returns (uint256) {
        // notional = size * price / 1e20 at price 1e8  =>  size = notional * 1e12
        return notionalAssets * 1e12;
    }

    function _mark(
        uint256 price
    ) internal {
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.prank(address(router));
        engine.updateMarkPrice(price, uint64(vm.getBlockTimestamp()));
    }

    function test_RecapAfterWipeErasesSeniorHighWaterMark() public {
        // Two extra junior LPs whose redemption requests give each settlement pass real queue progress.
        _fundJunior(juniorLp, 10_000e6);
        _fundJunior(juniorLp2, 10_000e6);
        vm.warp(vm.getBlockTimestamp() + 1 hours);

        uint256 hwmBefore = pool.seniorHighWaterMark();
        console2.log("senior principal before", pool.seniorPrincipal());
        console2.log("senior HWM before      ", hwmBefore);
        console2.log("junior principal before", pool.juniorPrincipal());

        // Matured redemption request #1 (settled in pass A) and #2 (settled in pass B, a later epoch).
        uint256 s1 = juniorVault.balanceOf(juniorLp);
        vm.prank(juniorLp);
        juniorVault.requestRedeem(s1, juniorLp, juniorLp);

        // --- Trading: hedged book near the admission limit ---
        _fundTrader(alice, 1_000_000e6);
        _fundTrader(bob, 1_000_000e6);
        _mark(1e8);
        _open(alice, CfdTypes.Side.LONG, _lots(1_000_000e6), 300_000e6, 1e8);
        _open(bob, CfdTypes.Side.SHORT, _lots(2_000_000e6), 400_000e6, 1e8);
        _open(alice, CfdTypes.Side.LONG, _lots(1_900_000e6), 300_000e6, 1e8);
        _open(bob, CfdTypes.Side.SHORT, _lots(900_000e6), 200_000e6, 1e8);

        // --- Catastrophic two-sided move: longs realize, shorts then win -> terminal deficit + degraded ---
        _mark(0.5e8);
        _close(alice, CfdTypes.Side.LONG, _lots(2_900_000e6), 0.5e8);
        assertTrue(engine.degradedMode(), "degraded after long payout");
        _mark(1.6e8);

        // --- Documented runbook: owner recapitalizes and clears degraded mode ---
        uint256 recap = 1_400_000e6;
        settlementToken.mint(address(this), recap);
        settlementToken.approve(address(engine), recap);
        engine.recapitalizePool(recap);
        engine.clearDegradedMode();

        // --- Pass A: first LP settlement after recovery ---
        vm.warp((vm.getBlockTimestamp() / 1 hours + 1) * 1 hours + 10);
        // request #2 targets a later epoch (made after pass-A boundary)
        uint256 s2 = juniorVault.balanceOf(juniorLp2);
        vm.prank(juniorLp2);
        juniorVault.requestRedeem(s2, juniorLp2, juniorLp2);
        _mark(1.6e8);
        _settleLpEpochForTest();

        uint256 hwmAfterRecap = pool.seniorHighWaterMark();
        console2.log("--- after recap reconcile ---");
        console2.log("senior principal", pool.seniorPrincipal());
        console2.log("senior HWM      ", hwmAfterRecap);
        console2.log("junior principal", pool.juniorPrincipal());
        console2.log("pending recap   ", pool.pendingRecapitalizationAssets());

        // The pre-loss Senior entitlement was overwritten by the recapitalization target.
        assertLt(hwmAfterRecap, hwmBefore, "Senior HWM entitlement erased by recapitalization");
        assertEq(hwmAfterRecap, recap, "HWM reset to the recapitalization amount");

        // --- Market recovers: trader short no longer in profit. Revenue should restore Senior to its HWM first ---
        vm.warp((vm.getBlockTimestamp() / 1 hours + 1) * 1 hours + 10);
        _mark(1e8);
        // Short closes flat at the recovered price, releasing liquidity (no realized PnL).
        _close(bob, CfdTypes.Side.SHORT, _lots(2_900_000e6), 1e8);
        _mark(1e8);
        _settleLpEpochForTest();

        console2.log("--- after recovery reconcile ---");
        console2.log("senior principal", pool.seniorPrincipal());
        console2.log("senior HWM      ", pool.seniorHighWaterMark());
        console2.log("junior principal", pool.juniorPrincipal());
        console2.log("unassigned      ", pool.unassignedAssets());

        // Invariant violated: Junior extracted surplus while Senior is still below its pre-loss high-water mark.
        assertLt(pool.seniorPrincipal(), hwmBefore, "Senior not restored to pre-loss HWM");
        assertGt(pool.juniorPrincipal(), 0, "Junior received revenue before Senior restoration");
        console2.log("value diverted from Senior to Junior", hwmBefore - pool.seniorPrincipal());
    }

}
```

