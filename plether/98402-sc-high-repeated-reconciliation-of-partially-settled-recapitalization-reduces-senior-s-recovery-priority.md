# \#98402 [SC-High] Repeated reconciliation of partially settled recapitalization reduces Senior’s recovery priority

**Submitted on Nov 7th 2026 at 17:35:17 UTC by @n0fr33w1f14u for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98402
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`HousePool` can settle the same available NAV against a pending recapitalization more than once. When a recapitalization is only partially settleable, a later reconciliation can treat the remaining pending amount as newly available, temporarily wipe Senior principal, and reset the Senior high water mark to the smaller remaining recapitalization amount. Future revenue that should restore Senior’s recovery target can then accrue to Junior.

## Vulnerability Details

During reconciliation, `HousePool` caps pending claimant buckets against the full current distributable NAV:

```solidity
// HousePool.sol
function _settleablePendingClaimantBuckets(
    HousePoolAccountingLib.ReconcileSnapshot memory reconcileSnapshot,
    HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets
) internal pure returns (
    HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets
) {
    return HousePoolPendingPreviewLib.capClaimantBuckets(
        claimantBuckets, reconcileSnapshot.distributable
    );
}
```

The reconciliation plan subtracts the settleable amount from distributable NAV before comparing the remainder with existing tranche principal:

```solidity
// HousePoolReconcilePlanLib.sol
if (pendingBucketAssets > 0) {
    snapshot.distributable =
        snapshot.distributable > pendingBucketAssets
            ? snapshot.distributable - pendingBucketAssets
            : 0;
}
```

This prevents the pending amount from being counted twice during a single reconciliation. However, after a partial settlement, the next reconciliation again caps the remaining pending amount against the entire current NAV. It does not distinguish NAV already credited to Senior during the previous pass 

If that subtraction reduces distributable NAV to zero, the waterfall absorbs an apparent loss and can wipe the Senior principal credited in the preceding pass. The pending recapitalization is then applied. Because both tranche principals are zero, the recapitalization handler treats the state as a bootstrap and overwrites the Senior high water mark with the remaining recapitalization intent:

```solidity
// HousePoolPendingPreviewLib.sol
if (state.seniorSupply > 0) {
    if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
        state.waterfall.seniorPrincipal += remaining;
        state.waterfall.seniorHighWaterMark =
            recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
        remaining = 0;
    } else {
        // Restore Senior principal up to its existing high-water mark.
    }
}
```

For example, with $50,000 distributable NAV, $100,000 pending recapitalization, and surviving Senior shares:

| State | Distributable NAV | Senior principal | Senior high-water mark | Pending recap |
|---|---:|---:|---:|---:|
| After first reconciliation | $50,000 | $50,000 | $100,000 | $50,000 |
| After second reconciliation | $50,000 | $50,000 | $50,000 | $0 |

No cash movement or economic loss is required between the two reconciliations. The second pass wipes and re credits the same $50,000 of Senior principal, but resets the recovery target from $100,000 to $50,000.

A later increase in distributable NAV is routed according to the reduced target:

```solidity
// HousePoolWaterfallAccountingLib.sol
if (remaining > 0 && nextState.seniorPrincipal < nextState.seniorHighWaterMark) {
    uint256 deficit = nextState.seniorHighWaterMark - nextState.seniorPrincipal;
    uint256 restore = remaining < deficit ? remaining : deficit;
    nextState.seniorPrincipal += restore;
    remaining -= restore;
}

if (nextState.seniorPrincipal > nextState.seniorHighWaterMark) {
    nextState.seniorHighWaterMark = nextState.seniorPrincipal;
}

nextState.juniorPrincipal += remaining;
```

For example, if $10,000 of revenue arrives after the second reconciliation, Senior is already at its new $50,000 high water mark, so the $10,000 accrues to Junior. With the original $100,000 target preserved, the revenue would first restore Senior.

The reconciliations can persist through ordinary LP settlement. A Junior holder can request redemption of their full redeemable balance even when its estimated value is below the minimum:

```solidity
// TrancheVault.sol
if (estimateRedeemAssets(shares) < POOL.minTrancheDepositAssets() && shares < maxShares) {
    revert TrancheVault__WithdrawalTooSmall();
}
```

When Junior principal is zero, settlement refunds a zero value redemption request and counts it as processed work:

```solidity
// HousePool.sol
if (fullHeadAssets == 0) {
    // ... validate and refund the epoch remainder ...

    phase.processedEpochs += 1;
    continue;
}
```

Because a processed epoch counts as progress, the settlement does not revert for lack of progress. Separate matured requests can therefore persist both reconciliations.

## Impact Details

Contract fails to deliver promised returns, but doesn't lose value — Low.

The demonstrated impact is a reduction in Senior’s priority over future recovery revenue. In the example, Senior’s current principal remains $50,000 when the target is reduced, but up to $50,000 of subsequent revenue that would otherwise restore Senior to the original $100,000 target can accrue to Junior instead.

The scenario requires a partially settleable recapitalization. For a flat pool, this can occur when outstanding trader claims exceeded pool cash before the recapitalization. The owner can then use the documented recapitalization and degraded mode recovery actions.

## Proof of Concept

The following is a step by step accounting reproduction. All amounts are in USDG, ignoring fees and rounding for clarity.

1. Start with an active pool whose Senior and Junior share supplies survive a loss. The pool is flat, with $25,000 in cash and $75,000 in outstanding trader claims. Its terminal deficit is $50,000.

2. The owner calls `CfdEngine.recapitalizePool(100_000)`. Pool cash becomes $125,000, trader claims remain $75,000, distributable NAV becomes $50,000, and pending recapitalization is $100,000. Once the conditions for recovery are met, the owner clears degraded mode.

3. A matured zero value Junior redemption request is processed through `HousePool.settleLpEpoch`. Reconciliation caps the settleable recapitalization at $50,000. It credits Senior principal with $50,000, sets the Senior high water mark to the full $100,000 recapitalization intent, and leaves $50,000 pending. The Junior request is made refundable, so the settlement counts as progress.

4. With no intervening change in cash, claims, or NAV, process another matured zero value Junior redemption request through `settleLpEpoch`.

5. The second reconciliation caps the remaining $50,000 pending recapitalization against the same $50,000 NAV. It subtracts that amount before the waterfall, wiping the $50,000 Senior principal. The recapitalization handler then bootstraps Senior again with $50,000 and resets the high water mark to the remaining $50,000 intent. Pending recapitalization becomes zero.

6. Subsequently, $10,000 of trading revenue increases distributable NAV to $60,000. Reconciliation allocates the $10,000 to Junior because Senior principal already equals the reduced $50,000 high water mark. With the original $100,000 target preserved, that revenue would instead restore Senior principal to $60,000.

## Proof of Concept

`test/poc/RepeatedPartialRecapSettlement.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolTestBase} from "../../packages/perps/test/perps/shared/HousePoolTestBase.sol";
import {StdStorage, stdStorage} from "forge-std/StdStorage.sol";

/// @notice PoC for repeated partial recapitalization settlement lowering the Senior recovery target.
/// @dev The test storage-writes an outstanding trader-claim balance to model a flat insolvent pool after an
///      underfunded winning close. Recapitalization, degraded-mode recovery, LP redemption requests, and both
///      settlement passes use the deployed in-test contracts without mocks.
contract RepeatedPartialRecapSettlementPoC is HousePoolTestBase {

    using stdStorage for StdStorage;

    uint256 private constant RECAPITALIZATION = 100_000e6;
    uint256 private constant EXPECTED_NAV = 50_000e6;
    uint256 private constant NEW_REVENUE = 10_000e6;

    function test_Poc_RepeatedPartialRecapSettlementResetsSeniorRecoveryTarget() public {
        // Bob is a non-seed Junior LP, so his shares can be requested and refunded through the public vault path.
        _fundJunior(bob, 100_000e6);
        uint256 bobShares = juniorVault.balanceOf(bob);
        assertGt(bobShares, 0, "Junior holder must own redeemable shares");

        // Fixture only: write the aggregate claim balance a prior underfunded winning close could leave behind.
        // This storage write is not an attacker action; it creates the flat, insolvent precondition for the PoC.
        _finishAsyncCooldown(juniorVault, bob);
        uint256 cashBeforeRecapitalization = pool.totalAssets();
        uint256 claimsAfterRecapitalization = cashBeforeRecapitalization + RECAPITALIZATION - EXPECTED_NAV;
        _setTotalTraderClaim(claimsAfterRecapitalization);

        // Only the vault may call reconcile; this applies the terminal loss to both tranches while preserving the Senior HWM.
        vm.prank(address(juniorVault));
        pool.reconcile();
        // Confirm the prerequisite state: wiped principal and surviving Senior recovery entitlement/share ownership.
        assertEq(pool.seniorPrincipal(), 0, "Senior should be wiped before recapitalization");
        assertEq(pool.juniorPrincipal(), 0, "Junior should be wiped before recapitalization");
        assertGt(pool.seniorHighWaterMark(), 0, "Wipeout should preserve the prior Senior target");

        // Emulate degraded mode after the deficit, then use the documented owner-funded recap and recovery paths.
        // Setting the flag is test setup; the actual recapitalizePool and clearDegradedMode calls use the live Engine.
        stdstore.target(address(engine)).sig("degradedMode()").checked_write(true);
        settlementToken.mint(address(this), RECAPITALIZATION);
        settlementToken.approve(address(engine), RECAPITALIZATION);
        engine.recapitalizePool(RECAPITALIZATION);
        engine.clearDegradedMode();

        // The pre-existing claim deficit consumes half the recap, leaving $50,000 NAV and $50,000 pending.
        assertEq(
            pool.totalAssets() - engine.totalTraderClaimBalanceAssets(),
            EXPECTED_NAV,
            "Recapitalization should leave exactly $50,000 of terminal NAV"
        );
        assertEq(pool.pendingRecapitalizationAssets(), RECAPITALIZATION);

        // Full-balance redemption bypasses the minimum-value threshold. With Junior NAV at zero, settlement refunds
        // these shares without paying cash, while still counting the epoch as progress and persisting reconciliation.
        vm.prank(bob);
        uint256 firstRedeemEpoch = juniorVault.requestRedeem(bobShares, bob, bob);
        _settleAsyncRequest(firstRedeemEpoch, false);

        // First pass correctly credits current NAV and leaves the rest of the recap target pending.
        assertEq(pool.seniorPrincipal(), EXPECTED_NAV, "First pass should credit the available NAV to Senior");
        assertEq(pool.seniorHighWaterMark(), RECAPITALIZATION, "First pass should retain the full recap target");
        assertEq(
            pool.pendingRecapitalizationAssets(),
            RECAPITALIZATION - EXPECTED_NAV,
            "First pass should leave the recap residual pending"
        );

        uint256 cashAfterFirstPass = pool.totalAssets();
        // Claim the refunded shares and wait out the normal cooldown before requesting another full-balance redemption.
        vm.prank(bob);
        juniorVault.claimRedeemRefund(firstRedeemEpoch, bob, bob);
        _finishAsyncCooldown(juniorVault, bob);

        // A second permissionless LP settlement in a later epoch reprocesses the same NAV against the residual recap.
        vm.prank(bob);
        uint256 secondRedeemEpoch = juniorVault.requestRedeem(bobShares, bob, bob);
        assertGt(secondRedeemEpoch, firstRedeemEpoch, "Second refund must settle in a later epoch");
        _settleAsyncRequest(secondRedeemEpoch, false);

        // The bug must change accounting only: cash and current Senior principal remain fixed while HWM and pending recap fall.
        assertEq(pool.totalAssets(), cashAfterFirstPass, "Zero-value refunds must not change pool cash");
        assertEq(pool.seniorPrincipal(), EXPECTED_NAV, "Repeated pass must leave current Senior NAV unchanged");
        assertEq(
            pool.seniorHighWaterMark(),
            EXPECTED_NAV,
            "Repeated pass must expose the bug by lowering the recovery target to current NAV"
        );
        assertEq(pool.pendingRecapitalizationAssets(), 0, "Repeated pass incorrectly consumes the full recap bucket");

        // Test-only income fixture: MockSettlementToken.mint is freely callable and is not available on production USDG.
        // This models a later $10,000 pool asset increase; it does not prove the source of production revenue.
        settlementToken.mint(address(pool), NEW_REVENUE);
        // accountExcess is owner-only. It admits the fixture amount as pool assets, so this is not an attacker action.
        pool.accountExcess();
        // With no positions open, the vault can reconcile the increased terminal NAV against the broken HWM.
        vm.prank(address(juniorVault));
        pool.reconcile();

        // At the reduced target Senior receives nothing; a preserved $100,000 target would route this revenue to Senior first.
        assertEq(pool.seniorPrincipal(), EXPECTED_NAV, "Senior should miss restoration toward the original target");
        assertEq(pool.juniorPrincipal(), NEW_REVENUE, "Revenue should incorrectly accrue to Junior");
    }
}

```
