# \#98066 [SC-High] Pending recapitalization is not reduced when its cash pays trader claims, so every later reconcile re-applies it and cuts Senior's recovery target

**Submitted on Nov 4th 2026 at 18:56:41 UTC by @Mclovin for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98066
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary
After a terminal deficit the owner recapitalizes the pool (the documented runbook, SECURITY.md:1499-1511). If trader claims still exceed pool cash at that moment, the first reconcile credits only the backed part of the recap and keeps the rest in `pendingRecapitalizationAssets`. The vendor's own test covers this state and expects the residual to "preserve senior's recovery target" and to "settle once backing becomes available".

The bucket is only ever reduced when a reconcile applies it (HousePool.sol:1790). Nothing reduces it when the claim it was waiting on is **paid**. `CfdEngine.settleTraderClaim` pays the claim out of pool cash (CfdEngine.sol:238 → HousePool.payOut, HousePool.sol:604-613). After that payout the residual has no cash behind it, yet every later fresh reconcile does three things:

1. It caps the residual against the full distributable value (`capClaimantBuckets`, HousePoolPendingPreviewLib.sol:72-82; HousePool.sol:1564-1566, 1829-1834). At that point that value is Senior's own equity.
2. It subtracts that amount before the waterfall (HousePoolReconcilePlanLib.sol:68-71). Senior absorbs a loss that never happened and drops to zero.
3. It re-applies the amount as a recap (`applyClaimantRecapitalizationIntent`, HousePoolPendingPreviewLib.sol:111-137). With both principals at zero, the bootstrap branch sets the high-water mark to `max(remaining residual, amount)`, overwriting Senior's recovery target.

Senior's principal ends up unchanged, but its recovery target falls by the size of Senior's equity with no economic event. The recovery target is what future revenue must restore before Junior earns anything. The same calls in a different order also route later revenue differently.

## What the docs promise
- ACCOUNTING_SPEC.md:337: "Losses do not reduce the Senior high-water mark." Here there is no loss at all.
- ACCOUNTING_SPEC.md:338: "Route surplus first to restore Senior principal to its high-water mark, then to Junior."
- ACCOUNTING_SPEC.md:339-340: pending buckets are reserved so that "Reserving them in step 2 prevents counting the same inflow twice." The same recap inflow is counted again at each reconcile.
- ACCOUNTING_SPEC.md:649-650: prefer seeded ownership paths; "only value with no safe claimant path belongs in `unassignedAssets`."
- Vendor test `HousePoolWaterfall.t.sol::test_PendingRecapitalization_CapsAgainstTraderClaimLiabilitiesAndCarriesResidual`: "Residual recap should preserve senior's recovery target". This holds after the first reconcile and is lost at the second. That test only models the claim disappearing while its cash stays in the pool (`_setTotalTraderClaim(0)`), never the claim being paid.

## Numbers (PoC below, all on a82d4fef)
Same starting state as the vendor test: wiped pool, a 100,000 recap through `engine.recapitalizePool`, and 50,040 of trader claims. The claim is then paid exactly as `settleTraderClaim` pays it. The pool holds 49,960 cash, with no claims and no positions, and all of it is Senior's.

| Step | Senior principal | Senior high-water mark | Pending recap |
|---|---|---|---|
| Claim paid | 49,960 | 100,000 | 50,040 |
| Reconcile (nothing changed) | 49,960 | **50,040** | 80 |
| Reconcile (nothing changed) | 49,960 | 50,040 | 0 |

**No special actor is needed.** One Senior holder redeems 0.1% of their shares and the redemption is settled through the permissionless `settleLpEpoch`. That moves the high-water mark from 100,000 to **49,994.51**. In the control, where the claim is removed but its cash is kept, the same redemption gives 99,909.09, which is only the withdrawal's own scaling.

**Same economics, different outcome.** 60,000 of revenue arrives after the payout:

| | Senior | Junior | Unassigned |
|---|---|---|---|
| Spec (ACCOUNTING_SPEC.md:338) | 100,000 | 9,960 | 0 |
| X: no reconcile in between | 100,000 | 0 | **9,960** |
| Y: one no-op reconcile in between | **50,040** | 59,840 | 80 |

- **Path X:** Junior holders (who exist) see their revenue become ownerless `unassignedAssets`. That blocks new tranche deposits (ACCOUNTING_SPEC.md:324) and reaches LPs only through the owner's `assignUnassignedAssets`.
- **Path Y:** this is the default, since any later LP settlement triggers it. Senior is short 49,960 of its promised recovery, and Junior receives it.
- The LP-side total is identical in every case, so no value leaves the pool.

## Impact
Senior LPs lose their promised recovery priority without any loss event, or Junior LPs' revenue is parked in ownerless assets. Which one happens depends only on how many reconciles run between the claim payout and later revenue. Any LP can cause a reconcile through the permissionless settlement. This is the program's Low row: the contract fails to deliver promised returns, but doesn't lose value.

## Preconditions
- A terminal deficit with trader payout claims that pool cash cannot service, followed by the owner's recapitalization. This is the documented recovery procedure: SECURITY.md:1499-1511, step 4 "Recapitalize or clear positions"; ACCOUNTING_SPEC.md:639-643.
- A trader then settles its claim, which is the normal next step, and any LP settlement follows.
- The PoC uses the vendor's own harness and starting state, and changes one thing: the claim is paid instead of disappearing with its cash still in the pool. I also reproduced the state end-to-end through public trading calls. Reaching a terminal deficit that way needs a severe market move, which is the situation the recapitalization path exists for. I'm happy to share it.

## Why this is not MED-12
MED-12 was the revenue-continuation flag being computed before recapitalization, so revenue in the same batch went to unassigned. Here there is no pending revenue bucket and the flag is true. The root cause is different: the recap bucket is never reduced when its backing leaves the pool, so later reconciles reserve and re-apply it. It reproduces on a82d4fef with the MED-12 fix in place.

## Suggested fix
When a trader claim is paid while a pending recapitalization residual exists, reduce `pendingRecapitalizationAssets` by the amount the payout consumed (up to the residual), and leave the high-water mark unchanged. Later revenue then restores Senior through the normal waterfall (ACCOUNTING_SPEC.md:338). The vendor's existing case, where the claim is removed and the cash kept, still settles the residual as before. As a regression test, take the vendor's residual test with `payOut` plus `_setTotalTraderClaim(0)`, reconcile twice, and check that the high-water mark stays at 100,000.


## Proof of Concept

## Proof of concept
Save as `packages/perps/test/poc/h-pr-1/ResidualRecapPaidClaim.t.sol`. Any folder two levels below `test/` works, which the `HousePoolTestBase` import needs. Run from `packages/perps`:

```
FOUNDRY_FFI=false forge test --match-contract ResidualRecapPaidClaimTest -vv
```

Result on a82d4fef: `Suite result: ok. 5 passed; 0 failed; 0 skipped`. The two `Control_*` tests are the vendor's branch (claim removed, cash retained) and behave correctly. The three `ClaimPaid_*` tests show the bug.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolTestBase} from "../../perps/shared/HousePoolTestBase.sol";
import {console2} from "forge-std/console2.sol";

/// @notice The pending recapitalization bucket is never reduced when the cash behind it leaves through a trader-claim
///         payout. The leftover is then reserved again against Senior's real equity at every later reconcile and
///         re-applied, which cuts Senior's recovery target and makes the Senior/Junior split depend on how many
///         reconciles happen, not on economics.
///
///         Starting state = the vendor's own test
///         HousePoolWaterfall.t.sol::test_PendingRecapitalization_CapsAgainstTraderClaimLiabilitiesAndCarriesResidual
///         (wiped pool, 100,000 recap, 50,040 of trader claims). The vendor test then removes the claim liability while
///         keeping the cash (`_setTotalTraderClaim(0)`) and the residual settles to Senior. Here the claim is PAID, as
///         CfdEngine.settleTraderClaim does (CfdEngine.sol:238): the pool pays the cash out and the liability drops by
///         the same amount. Nothing else changes.
contract ResidualRecapPaidClaimTest is HousePoolTestBase {

    uint256 internal constant RECAP = 100_000e6;
    uint256 internal constant CLAIM = 50_040e6;

    function _wipeRecapAndFirstReconcile() internal {
        settlementToken.burn(address(pool), pool.totalAssets());
        vm.prank(address(juniorVault));
        pool.reconcile();

        // Documented owner action (ACCOUNTING_SPEC.md:639-643; SECURITY.md:1499-1511 runbook step 4).
        settlementToken.mint(address(this), RECAP);
        settlementToken.approve(address(engine), RECAP);
        engine.recapitalizePool(RECAP);
        _setTotalTraderClaim(CLAIM);

        vm.prank(address(seniorVault));
        pool.reconcile();
        assertEq(pool.seniorPrincipal(), RECAP - CLAIM, "first reconcile credits only the settleable recap");
        assertEq(pool.seniorHighWaterMark(), RECAP, "vendor: residual recap should preserve senior's recovery target");
        assertEq(pool.pendingRecapitalizationAssets(), CLAIM, "residual stays pending");
    }

    /// @dev What settleTraderClaim does to the pool: pay the claim out of pool cash; the liability drops equally.
    function _payClaim() internal {
        vm.prank(address(engine));
        pool.payOut(address(clearinghouse), CLAIM);
        _setTotalTraderClaim(0);
    }

    function _reconcile() internal {
        vm.prank(address(seniorVault));
        pool.reconcile();
    }

    function _log(
        string memory label
    ) internal view {
        console2.log(label);
        console2.log("   senior principal / senior HWM:", pool.seniorPrincipal(), pool.seniorHighWaterMark());
        console2.log("   junior principal / unassigned:", pool.juniorPrincipal(), pool.unassignedAssets());
        console2.log("   pending recap / pool cash:", pool.pendingRecapitalizationAssets(), pool.totalAssets());
    }

    /// Control = the vendor's branch: the claim liability disappears while its cash stays. Residual settles to Senior.
    function test_Control_ClaimRemovedCashRetained() public {
        _wipeRecapAndFirstReconcile();
        _setTotalTraderClaim(0);
        _reconcile();
        _log("CONTROL claim removed, cash retained, reconcile");
        assertEq(pool.seniorPrincipal(), RECAP);
        assertEq(pool.seniorHighWaterMark(), RECAP);
        assertEq(pool.pendingRecapitalizationAssets(), 0);
    }

    /// The claim is paid. Pool: 49,960 cash, no claims, no positions; all of it is Senior's. Two reconciles of that
    /// unchanged state cut Senior's high-water mark from 100,000 to 50,040 (ACCOUNTING_SPEC.md:337 "Losses do not
    /// reduce the Senior high-water mark"; :339-340 reservation "prevents counting the same inflow twice").
    function test_ClaimPaid_NoOpReconcileCutsSeniorRecoveryTarget() public {
        _wipeRecapAndFirstReconcile();
        _payClaim();
        _log("after claim paid (before any reconcile)");
        uint256 sBefore = pool.seniorPrincipal();
        _reconcile();
        _log("reconcile 2 (no economic change)");
        _reconcile();
        _log("reconcile 3 (no economic change)");
        assertEq(pool.seniorPrincipal(), sBefore, "Senior NAV unchanged");
        assertEq(pool.seniorHighWaterMark(), CLAIM, "Senior recovery target cut from 100,000 to 50,040 with no loss");
        assertEq(pool.pendingRecapitalizationAssets(), 0, "the phantom residual was consumed against Senior equity");
    }

    /// Same economics, two orders of the same calls. Revenue of 60,000 arrives after the claim was paid.
    /// Spec routing (ACCOUNTING_SPEC.md:338): restore Senior to its high-water mark (100,000), the rest (9,960) to
    /// Junior, which has share owners. Path X: no reconcile in between. Path Y: one no-op reconcile in between.
    function test_ClaimPaid_SameEconomicsDifferentSplit() public {
        _wipeRecapAndFirstReconcile();
        _payClaim();
        assertGt(juniorVault.totalSupply(), 0, "junior share owners exist");
        uint256 snap = vm.snapshotState();

        _mintAndAccountPoolExcess(60_000e6);
        _reconcile();
        _log("PATH X: revenue, one reconcile");
        (uint256 sx, uint256 jx, uint256 ux) = (pool.seniorPrincipal(), pool.juniorPrincipal(), pool.unassignedAssets());

        vm.revertToState(snap);
        _reconcile();
        _mintAndAccountPoolExcess(60_000e6);
        _reconcile();
        _log("PATH Y: one no-op reconcile, revenue, one reconcile");
        (uint256 sy, uint256 jy, uint256 uy) = (pool.seniorPrincipal(), pool.juniorPrincipal(), pool.unassignedAssets());

        console2.log("REFERENCE (spec): senior 100000000000, junior 9960000000, unassigned 0");
        assertEq(sx + jx + ux, sy + jy + uy, "same LP-side total");
        assertTrue(sx != sy || jx != jy || ux != uy, "same economics, different allocation");
        assertGt(ux, 0, "path X: Junior's revenue lands in ownerless unassigned assets");
        assertLt(sy, RECAP, "path Y: Senior is not restored to its recovery target");
    }

    /// The trigger needs no attacker: an ordinary Senior redemption (0.1% of one holder's shares), settled through
    /// the permissionless HousePool.settleLpEpoch, persists the reconcile that re-applies the residual.
    function _seniorRedemptionSettled(
        bool claimPaid
    ) internal returns (uint256 hwmBefore, uint256 hwmAfter) {
        _fundSenior(alice, 10_000e6);
        settlementToken.burn(address(pool), pool.totalAssets());
        vm.prank(address(juniorVault));
        pool.reconcile();
        settlementToken.mint(address(this), RECAP);
        settlementToken.approve(address(engine), RECAP);
        engine.recapitalizePool(RECAP);
        _setTotalTraderClaim(CLAIM);
        _reconcile();
        if (claimPaid) {
            _payClaim();
        } else {
            _setTotalTraderClaim(0);
        }
        hwmBefore = pool.seniorHighWaterMark();
        _finishAsyncCooldown(seniorVault, alice);
        uint256 shares = seniorVault.balanceOf(alice) / 1000;
        uint256 requestId = _requestAsyncRedeem(seniorVault, alice, shares);
        _settleAsyncRequest(requestId, true); // permissionless settleLpEpoch (no open positions)
        assertEq(seniorVault.claimableRedeemRequest(requestId, alice), shares, "redemption funded");
        hwmAfter = pool.seniorHighWaterMark();
        _log(claimPaid ? "claim PAID, then one ordinary Senior redemption settled" :
            "CONTROL claim removed (cash retained), then one ordinary Senior redemption settled");
    }

    function test_ClaimPaid_OrdinarySeniorRedemptionCutsRecoveryTarget() public {
        (uint256 before_, uint256 after_) = _seniorRedemptionSettled(true);
        console2.log("senior HWM before / after:", before_, after_);
        assertEq(before_, RECAP, "recovery target 100,000 before");
        assertLt(after_, 50_100e6, "recovery target cut to ~50,000 by a 0.1% redemption");
    }

    function test_Control_OrdinarySeniorRedemptionKeepsRecoveryTarget() public {
        (uint256 before_, uint256 after_) = _seniorRedemptionSettled(false);
        console2.log("senior HWM before / after:", before_, after_);
        assertGt(after_, 99_800e6, "recovery target kept (only the 0.1% withdrawal scaling)");
    }

}
```
