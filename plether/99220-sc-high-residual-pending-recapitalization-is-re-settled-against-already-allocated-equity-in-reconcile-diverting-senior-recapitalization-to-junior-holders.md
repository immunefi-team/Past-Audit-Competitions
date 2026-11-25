# \#99220 [SC-High] Residual Pending Recapitalization Is Re-Settled Against Already Allocated Equity in _reconcile, Diverting Senior Recapitalization to Junior Holders

**Submitted on Nov 12th 2026 at 12:57:23 UTC by @Hacken for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99220
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

When governance recapitalizes the pool through CfdEngine.recapitalizePool, the amount is queued in pendingRecapitalizationAssets. The next committed reconcile applies the part backed by distributable equity, called D below. The rest stays pending as a residual, to be applied once more equity becomes available. Its intent is to restore senior principal before any value reaches junior. The team’s remediation note for Cecuro MED-12 also states that partial-settlement residuals stay reserved.

Every reconcile, however, caps the entire stored bucket against the entire current distributable equity. Nothing records which part of that equity already backs principal credited by an earlier pass:

```solidity
return HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, reconcileSnapshot.distributable);
```

HousePoolReconcilePlanLib.planReconcile removes only the settleable amount from distributable equity. It then reconciles the waterfall against what remains, which books the same amount as a loss to existing claimants. _applyPendingClaimantBucketsLive credits the amount back through the recapitalization intent and decrements the stored residual.

On the first reconcile after the recapitalization this is correct. On any later reconcile, the same D, which senior already owns, is used again to settle the residual.

While the residual is at least D, each extra pass:

1. wipes senior principal to zero on paper;
2. re-credits D through the empty-state bootstrap of applyClaimantRecapitalizationIntent;
3. overwrites the senior high-water mark with the current residual (HousePoolPendingPreviewLib.sol:121).

The residual drops by D on every pass, and the high-water mark follows it down. When equity later recovers, senior is restored only up to the decayed mark, and the rest flows to junior.

If the recapitalization exceeds twice the deficit, the residual is smaller than D. The re-settlement loss is then partial, the bootstrap is never taken, and the high-water mark survives, so nothing is misallocated.

Each committed reconcile happens inside settleLpEpoch, which reverts unless some epoch makes progress. That progress is easy to get. Hourly keeper settlement of any matured request provides it. A wiped junior holder can also provide it for free: a full-balance junior redemption is allowed below the minimum size, is priced at zero while junior principal is zero, and is refunded and counted as progress.

Example: senior and junior seeds of 1,000, both wiped; a 50,040 trader claim outstanding; a recapitalization of 100,000.

1. Reconcile 1 credits D = 49,960 to senior and sets the high-water mark to 100,000. The residual of 50,040 stays pending.
2. Reconcile 2 changes nothing economically. It still books a 49,960 loss against senior, re-credits 49,960, and resets the high-water mark to 50,040. The residual drops to 80.
3. Reconcile 3 erases the remaining 80.
4. When the 50,040 held back by the trader claim becomes distributable, senior is restored only to 50,040, and junior receives 49,960.

Without steps 2 and 3, senior ends at 100,000 and junior at 0. That is the outcome the repository’s own test test_PendingRecapitalization_CapsAgainstTraderClaimLiabilitiesAndCarriesResidual expects.

A smaller recapitalization loses more. With 65,000 (D = 14,960), the mark decays over three passes: 50,040, then 35,080, then 20,120. Senior ends at 20,120, and junior receives 44,880.

In general, for a recapitalization between the deficit and twice the deficit, senior ends between D and 2D instead of at the full recapitalized amount. Junior receives the rest.

For a recapitalization between the deficit and twice the deficit, senior ends between D and 2D instead of at the recapitalized amount. The rest goes to junior holders whose principal had been wiped, and they can withdraw it as cash. With a 65,000 recapitalization, junior receives 44,880.

**Recommendation:**

**Settle** a residual bucket only from equity that is not yet allocated to claimants, and preserve the existing high-water mark in the bootstrap:

```solidity
// cap later passes by unallocated equity only
uint256 allocated = seniorPrincipal + juniorPrincipal + unassignedAssets;
uint256 settleableBase = distributable > allocated ? distributable - allocated : 0;

// keep prior senior protection
state.waterfall.seniorHighWaterMark =
    Math.max(state.waterfall.seniorHighWaterMark, Math.max(recapitalizationTargetAssets, remaining));
```

## Proof of Concept

Steps to Reproduce:

1. Bring the pool to zero LP equity while traders hold a claim: senior and junior principal are wiped by a reconcile, and an unpaid trader claim of 50,040 remains.

2. As the owner, recapitalize the pool by more than the deficit but less than twice the deficit: CfdEngine.recapitalizePool(100_000e6).

3. Trigger the first reconcile, for example through settleLpEpoch. Senior principal becomes 49,960, the high-water mark becomes 100,000, and 50,040 stays in pendingRecapitalizationAssets.

4. Without any change in cash, claims or mark, trigger a second reconcile. Hourly keeper settlement does this, as does a wiped junior holder’s zero-priced redemption refund. The high-water mark drops to 50,040 and the residual drops to 80.

5. Trigger a third reconcile. The residual is now 0.

6. Let the equity held back by the trader claim become distributable: the claim is netted, the mark reverses, or new LP revenue arrives. Then reconcile.

7.Check the result: senior principal is 50,040 and junior principal is 49,960. If steps 4 and 5 are skipped, senior ends at 100,000 and junior at 0.

POC Code: 
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolTestBase} from "./shared/HousePoolTestBase.sol";

/// @notice PoC: residual pending recapitalization is re-settled against already allocated equity on every reconcile,
///         resetting the senior high-water mark and diverting the recapitalization to wiped junior holders.
/// @dev Loss and recovery are simulated as in the repository test
///      `test_PendingRecapitalization_CapsAgainstTraderClaimLiabilitiesAndCarriesResidual`.
contract AuditHousePoolResidualRecapTest is HousePoolTestBase {

    function test_ResidualRecapitalizationDivertedToJunior() public {
        // Senior and junior are wiped while a 50,040 trader claim is outstanding.
        settlementToken.burn(address(pool), pool.totalAssets());
        vm.prank(address(juniorVault));
        pool.reconcile();
        _setTotalTraderClaim(50_040e6);

        // Governance recapitalizes 100,000 through the engine.
        settlementToken.mint(address(this), 100_000e6);
        settlementToken.approve(address(engine), 100_000e6);
        engine.recapitalizePool(100_000e6);

        // Reconcile 1: restored equity D = 49,960 goes to senior, the residual 50,040 stays pending.
        vm.prank(address(seniorVault));
        pool.reconcile();
        assertEq(pool.seniorPrincipal(), 49_960e6);
        assertEq(pool.seniorHighWaterMark(), 100_000e6);
        assertEq(pool.pendingRecapitalizationAssets(), 50_040e6);

        // Reconcile 2: no economic change, yet the residual is settled again and the HWM is overwritten.
        vm.prank(address(seniorVault));
        pool.reconcile();
        assertEq(pool.seniorPrincipal(), 49_960e6);
        assertEq(pool.seniorHighWaterMark(), 50_040e6);
        assertEq(pool.pendingRecapitalizationAssets(), 80e6);

        // Reconcile 3 erases the rest of the residual.
        vm.prank(address(seniorVault));
        pool.reconcile();
        assertEq(pool.pendingRecapitalizationAssets(), 0);

        // Recovery: the equity held back by the trader claim becomes distributable.
        _setTotalTraderClaim(0);
        vm.prank(address(seniorVault));
        pool.reconcile();

        // Without reconciles 2 and 3 senior ends at 100,000 and junior at 0.
        assertEq(pool.seniorPrincipal(), 50_040e6);
        assertEq(pool.juniorPrincipal(), 49_960e6);
    }

}
```
