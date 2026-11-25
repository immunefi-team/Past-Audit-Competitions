# \#99574 [SC-High] A leftover recapitalization intent is settled against later trading revenue and paid to Senior above its high-water mark

**Submitted on Nov 14th 2026 at 09:54:39 UTC by @Tradi3 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99574
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

When trader claims exceed pool cash, the owner's documented recovery is `CfdEngine.recapitalizePool`. The whole injection is queued as recapitalization intent, including the part that only fills the trader-claim deficit. That part leaves the pool as the trader's payout, but the intent stays queued.

At the next LP settlement the leftover intent is settled against whatever cash has arrived since, which is ordinary trading revenue. The rule for a wiped pool then credits all of it to Senior and raises Senior's high-water mark to the full injection. The Senior LP is paid its high-water mark plus the deficit.

Measured at release parameters on Forge 1.5.1, with real Router order flow and two arms from one snapshot taken at the deficit. Both arms use the same owner money and receive the same later trading revenue. All figures are USDG.

- Senior high-water mark before the recovery: 39,999,991.00. Trader-claim deficit: 40,243,235.92.
- Runbook recovery, `recapitalizePool(deficit + Senior high-water mark)`: the Senior LP is paid **80,243,224.80**, Junior holds **0**, unassigned assets are 0, and the pool is left with 2.01.
- Control, where only the Senior part is queued as recapitalization: the Senior LP is paid 40,000,727.29, Junior holds 242,507.51, and 39,999,991.00 stays in the pool as unassigned assets.

The Senior LP takes 40,242,497.51 more than in the control.

**Target:** `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`: `packages/perps/src/HousePool.sol`, `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol`, `packages/perps/src/CfdEngine.sol`

## Vulnerability Details

### 1. The whole injection is queued as intent

`CfdEngine.recapitalizePool` (`CfdEngine.sol:632-653`) records the full amount as `Recapitalization`, and `HousePool.recordClaimantInflow` adds it to `pendingRecapitalizationAssets` (`HousePool.sol:624-651`, `:1794-1802`). With a trader-claim deficit `D`, the owner must inject at least `D` before degraded mode can be cleared, so the intent always contains money that is about to leave as a trader payout.

### 2. Paying the claim does not reduce the intent

`settleTraderClaim` pays the claim from pool cash. `pendingRecapitalizationAssets` is unchanged. After the payout the intent exceeds the cash behind it by `D`.

### 3. The leftover intent absorbs later cash

At reconcile, the settleable part of the pending buckets is the smaller of the pending amount and the pool's distributable value, and it is held out of LP equity before the waterfall runs (`HousePool.sol:1555-1575`, `HousePoolPendingPreviewLib.capClaimantBuckets`). With a leftover intent of `D + S` and cash of `S + G`, where `S` is the Senior part of the injection and `G` is revenue that arrived afterwards, all of the cash is treated as recapitalization. The waterfall sees no equity and both tranche principals are zero.

### 4. The wiped-pool rule then gives all of it to Senior

`HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent` (`HousePoolPendingPreviewLib.sol:111-137`):

```solidity
if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
    state.waterfall.seniorPrincipal += remaining;
    state.waterfall.seniorHighWaterMark =
        recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
    remaining = 0;
}
```

This rule is written for a wiped pool that is recapitalized with cash that stays in the pool, and the repository tests it that way (`test/perps/accounting/waterfall/TrancheRecoveryBehavior.t.sol:24`, `:577`). Here it is applied to trading revenue, because the queued intent no longer matches the cash that backed it. Senior principal becomes `S + G` and the high-water mark becomes `D + S`.

### What the documents say

The whitepaper's tranche rules (`WHITEPAPER.md:634-639`) say that a loss does not reduce the Senior high-water mark, that later revenue first restores Senior to it, and: "Revenue remaining after senior restoration accrues to junior." The accounting specification says the queued recapitalization goes to Senior restoration or to unassigned assets (`ACCOUNTING_SPEC.md:641-642`). Neither lets trading revenue be paid to Senior above its high-water mark.

### Sequence

No attacker is needed. The trigger is a market sequence followed by the owner using the documented recovery (`SECURITY.md:155-160`, `:1499-1511`), which the program scopes in as a legitimate trusted action that triggers a bug.

1. Junior and Senior LPs deposit. The Senior LP requests a full redemption.
2. Traders hold a two-sided book. One side closes in profit and is paid. The other side then closes in profit and its payout becomes a trader claim larger than pool cash. Deficit `D` = 40,243,235.92.
3. The owner calls `recapitalizePool(D + S)` with `S` equal to the Senior high-water mark, then `clearDegradedMode()`.
4. The trader calls `settleTraderClaim`. Claims are zero and the pool holds exactly `S`.
5. An unrelated trader loses 40,243,235.81 to the pool through ordinary opens and closes.
6. Anyone calls `settleLpEpoch`. The Senior LP claims 80,243,224.80.

## Impact Details

| | Runbook: `recapitalizePool(D + S)` | Control: only `S` queued |
|---|---:|---:|
| Owner money | 80,243,226.92 | 80,243,226.92 |
| Later trading revenue | 40,243,235.81 | 40,243,235.81 |
| Projected Senior principal before settlement | 80,243,226.81 | 40,000,728.29 |
| Projected Junior principal before settlement | 0 | 242,507.51 |
| Senior LP paid | **80,243,224.80** | 40,000,727.29 |
| Junior LP value | **0** | 242,507.51 |
| Unassigned assets left in the pool | 0 | 39,999,991.00 |
| Pool cash left | 2.01 | 40,242,499.52 |

In the control the deficit part of the owner's money is admitted with `accountExcess()` and only the Senior part goes through `recapitalizePool`. Everything else is identical. Senior is paid its high-water mark plus the coupon accrued meanwhile, Junior receives the revenue above that, and the recapitalization that turned out not to be needed stays in the pool as unassigned assets.

In the runbook arm the Senior LP is paid 40,242,497.51 more. That money is the 242,507.51 the control gives Junior and the 39,999,991.00 the control leaves unassigned. The Senior LP does nothing special: it redeems normally.

Severity: Critical, direct theft of user funds. A Senior LP withdraws about twice its entitlement, and Junior LPs receive nothing from revenue the rules assign to them.

Conditions, stated plainly:

- A trader-claim deficit must exist. In the proof it is produced by an extreme price path (1.00 to 0.10, then 1.90) on a two-sided book. This is the same incident our report #98025 uses.
- The owner recapitalizes with more than the deficit in one call, which is what restoring Senior requires.
- Trading revenue reaches the pool before the next LP settlement. The Senior LP is overpaid by that revenue, up to the deficit.

## Relation to #98025

#98025 and its group report that this leftover intent reserves all pool cash, so `settleLpEpoch` reverts and matured redemptions wait. Those reports describe a freeze in which no value moves. This report is about the other outcome of the same leftover: once more cash arrives, settlement succeeds and the leftover is paid to Senior. A change that only stops the revert, or only lets reconcile run, leaves this transfer in place.

## Recommendation

- Queue only the part of an injection that exceeds the current trader-claim deficit, or reduce `pendingRecapitalizationAssets` when the cash behind it is used to pay claims.
- Apply the wiped-pool rule only to cash that arrived as recapitalization, so that later revenue goes through the ordinary waterfall.

## References

- `packages/perps/src/CfdEngine.sol:632-653`
- `packages/perps/src/HousePool.sol:624-651`, `:1555-1575`, `:1794-1802`
- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:72-88`, `:111-137`
- `packages/perps/test/perps/accounting/waterfall/TrancheRecoveryBehavior.t.sol:24`, `:577`
- `packages/perps/WHITEPAPER.md:634-639`, `packages/perps/ACCOUNTING_SPEC.md:641-642`
- `packages/perps/SECURITY.md:155-160`, `:1499-1511`

## Proof of Concept

## Environment

- Commit: `a82d4fefb02768e5b3bddcc97dee371eee24fea8`
- Forge: `1.5.1-v1.5.1` (commit b0a9dd9c), Solidity 0.8.35
- No file under `packages/perps/src` is changed. Mocks: only the settlement token and the Pyth source. No storage is written.
- Release risk parameters, execution fee, Senior cap (40M) and Senior share (80%). Orders go through the Router's `commitOrder` and `executeOrder`.

## Steps

1. Save the file below as `packages/perps/test/perps/poc/RecapSeniorOverpay.t.sol`.
2. From `packages/perps` run:

```bash
forge test --match-path test/perps/poc/RecapSeniorOverpay.t.sol -vv
```

## Output (run 2026-11-14)

```text
[PASS] test_RunbookRecapPaysSeniorTheDeficitOutOfJuniorRevenue() (gas: 240972730)
Logs:
  Senior deposit before the incident    39999990000000
  Senior high-water mark at the deficit 39999991000675
  trader-claim deficit                  40243235915351
  owner money in both arms              80243226916026
  later trading revenue, both arms      40243235806177
  A runbook: projected Senior principal 80243226806852
  A runbook: projected Junior principal 0
  A runbook: Senior LP paid             80243224799415
  A runbook: Junior LP value            0
  A runbook: pool cash left             2007437
  A runbook: unassigned assets          0
  B control: projected Senior principal 40000728291224
  B control: projected Junior principal 242507514953
  B control: Senior LP paid             40000727290530
  B control: Junior LP value            242507505260
  B control: pool cash left             40242499516322
  B control: unassigned assets          39999991000675
  Senior paid more in A by              40242497508885
  Junior LP lost in A                   242507505260
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

## RecapSeniorOverpay.t.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolAsyncTestBase} from "../shared/HousePoolTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {console2} from "forge-std/console2.sol";

/// @dev When trader claims exceed pool cash, the owner's documented recovery is CfdEngine.recapitalizePool. The whole
///      injection is queued as recapitalization intent, including the part that only pays the trader-claim deficit.
///      That part leaves the pool as the trader's payout, but the intent stays. At the next reconcile the full intent
///      is held out of LP equity, both tranches compute to zero, and HousePoolPendingPreviewLib's empty-bootstrap
///      branch gives every settled unit to Senior principal and sets seniorHighWaterMark to the full intent. Cash that
///      arrived in between (ordinary trading revenue) is therefore paid to Senior as recapitalization.
///      Senior LPs are paid their high-water mark plus the deficit. Junior LPs are left with nothing.
///
///      Arms from one snapshot taken at the deficit, before the owner acts. The owner spends the same amount in both:
///        A (runbook):  recapitalizePool(deficit + Senior high-water mark)
///        B (control):  the deficit part is admitted through accountExcess(), and only the Senior high-water mark is
///                      sent through recapitalizePool
///      Both arms then run the same calls: clear degraded mode, pay the trader claim, the same losing trades by an
///      unrelated trader, one permissionless LP settlement, and the Senior LP's claim.
///
///      Release RiskParams, execution fee, Senior cap (40M) and Senior share (80%). Real Router order flow. Only Pyth
///      and USDG are mocks. No storage is written and no in-scope file is changed.
contract RecapSeniorOverpayTest is HousePoolAsyncTestBase {
    address private constant SENIOR_LP = address(0x51E0);
    address private constant JUNIOR_LP = address(0x1B0);
    address private constant LONG_TRADER = address(0x10A6);
    address private constant SHORT_TRADER = address(0x5407);
    address private constant LOSS_TRADER = address(0x1055);
    address private constant KEEPER = address(0xEE9);

    uint256 private constant SCALE = 325;
    uint256 private constant SENIOR_DEPOSIT = 40_000_000e6 - 10e6;
    uint256 private constant JUNIOR_DEPOSIT = 25_000_000e6;
    uint256 private constant LOSS_SIZE = 39_000e18 * SCALE;
    uint256 private constant LOSS_MARGIN = 4_600e6 * SCALE;

    uint256 private deficit;
    uint256 private seniorTarget;
    uint256 private redeemEpochId;

    enum Recovery {
        Runbook,
        DonationThenRecap
    }

    struct Out {
        uint256 seniorPaid;
        uint256 juniorValue;
        uint256 traderLoss;
        uint256 projectedSenior;
        uint256 projectedJunior;
        uint256 poolLeft;
        uint256 unassigned;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release Senior cap");
        assertEq(pool.maxSeniorShareBps(), 8000, "release Senior share");
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Real Router order flow
    // ---------------------------------------------------------------------------------------------------------------

    function _commitAndExecute(
        address trader,
        CfdTypes.Side side,
        uint256 sizeDelta,
        uint256 marginDelta,
        uint256 price,
        bool isClose,
        uint256 t
    ) internal returns (uint256) {
        vm.prank(trader);
        uint64 orderId = router.commitOrder(side, sizeDelta, marginDelta, price, isClose);
        uint256 nextT = t + 1;
        vm.warp(nextT);
        vm.roll(vm.getBlockNumber() + 1);
        baseMockPyth.setAllUniquePrices(_basePythFeedIds(), int64(uint64(price)), 0, -8, nextT, nextT - 1);
        bytes[] memory updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
        uint256 fee = pletherOracle.getOrderExecutionFee(updateData);
        vm.deal(KEEPER, fee);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder{value: fee}(orderId, updateData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
        return nextT;
    }

    function _refreshLiveMark(uint256 price, uint256 t) internal returns (uint256) {
        uint256 nextT = t + 1;
        vm.warp(nextT);
        vm.roll(vm.getBlockNumber() + 1);
        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(price)), -8, nextT);
        bytes[] memory updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
        uint256 fee = pletherOracle.getUpdateFee(updateData);
        vm.deal(KEEPER, fee);
        vm.prank(KEEPER);
        router.updateMarkPrice{value: fee}(updateData);
        return nextT;
    }

    // ---------------------------------------------------------------------------------------------------------------
    // 1. The incident: trader claims exceed pool cash (same construction as the confirmed recap-residual reports)
    // ---------------------------------------------------------------------------------------------------------------

    function _reachDeficit() internal {
        _fundJunior(JUNIOR_LP, JUNIOR_DEPOSIT);
        _fundSenior(SENIOR_LP, SENIOR_DEPOSIT);
        uint256 t = vm.getBlockTimestamp() + seniorVault.DEPOSIT_COOLDOWN() + 1;
        vm.warp(t);
        redeemEpochId = _requestAsyncRedeem(seniorVault, SENIOR_LP, seniorVault.balanceOf(SENIOR_LP));

        _fundTrader(LONG_TRADER, 50_000e6 * SCALE);
        _fundTrader(SHORT_TRADER, 200_000e6 * SCALE);
        t = _refreshLiveMark(1e8, t);
        t = _commitAndExecute(LONG_TRADER, CfdTypes.Side.LONG, 75_000e18 * SCALE, 20_000e6 * SCALE, 1e8, false, t);
        t = _commitAndExecute(SHORT_TRADER, CfdTypes.Side.SHORT, 150_000e18 * SCALE, 155_000e6 * SCALE, 1e8, false, t);
        t = _commitAndExecute(LONG_TRADER, CfdTypes.Side.LONG, 105_000e18 * SCALE, 25_000e6 * SCALE, 1e8, false, t);
        t = _commitAndExecute(SHORT_TRADER, CfdTypes.Side.SHORT, 30_000e18 * SCALE, 35_000e6 * SCALE, 1e8, false, t);
        t = _commitAndExecute(LONG_TRADER, CfdTypes.Side.LONG, 180_000e18 * SCALE, 0, 10_000_000, true, t);
        assertTrue(engine.degradedMode(), "first close latches degraded mode");
        t = _commitAndExecute(SHORT_TRADER, CfdTypes.Side.SHORT, 180_000e18 * SCALE, 0, 190_000_000, true, t);

        uint256 claims = engine.totalTraderClaimBalanceAssets();
        uint256 cash = pool.totalAssets();
        assertGt(claims, cash, "trader claim exceeds pool cash");
        deficit = claims - cash;
        seniorTarget = pool.seniorHighWaterMark();
        if (t < pool.lpEpochStart(redeemEpochId)) {
            vm.warp(pool.lpEpochStart(redeemEpochId));
        }
    }

    // ---------------------------------------------------------------------------------------------------------------
    // 2. The owner's recovery, the trader's claim, later trading revenue, one settlement
    // ---------------------------------------------------------------------------------------------------------------

    function _recover(Recovery mode) internal {
        uint256 total = deficit + seniorTarget;
        settlementToken.mint(address(this), total);
        if (mode == Recovery.Runbook) {
            settlementToken.approve(address(engine), total);
            engine.recapitalizePool(total);
        } else {
            settlementToken.transfer(address(pool), deficit);
            pool.accountExcess();
            settlementToken.approve(address(engine), seniorTarget);
            engine.recapitalizePool(seniorTarget);
        }
        engine.clearDegradedMode();
        vm.prank(SHORT_TRADER);
        engine.settleTraderClaim(SHORT_TRADER);
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "trader claim paid in full");
        assertEq(pool.totalAssets(), seniorTarget, "pool holds exactly the Senior target");
    }

    /// @dev An unrelated trader loses about `deficit` USDG to the pool: LONG opened at 1.00 and closed at 1.10,
    ///      repeated, then one sized trade to land within a few USDG below the target.
    function _tradingLosses() internal returns (uint256 lost) {
        _fundTrader(LOSS_TRADER, 70_000_000e6);
        uint256 targetAssets = seniorTarget + deficit;
        uint256 bigGain = LOSS_SIZE * 1e7 / 1e20;
        uint256 t = vm.getBlockTimestamp();
        for (uint256 i; i < 80; i++) {
            if (pool.totalAssets() + bigGain + 3_000e6 >= targetAssets) {
                break;
            }
            t = _commitAndExecute(LOSS_TRADER, CfdTypes.Side.LONG, LOSS_SIZE, LOSS_MARGIN, 1e8, false, t);
            t = _commitAndExecute(LOSS_TRADER, CfdTypes.Side.LONG, LOSS_SIZE, 0, 11e7, true, t);
        }
        for (uint256 k; k < 12; k++) {
            uint256 remaining = targetAssets > pool.totalAssets() ? targetAssets - pool.totalAssets() : 0;
            if (remaining <= 20e6) {
                break;
            }
            uint256 notional = remaining + 10e6;
            if (notional < 1_100e6) {
                notional = 1_100e6;
            }
            uint256 trimSize = ((notional * 1e12) / 100e18) * 100e18;
            uint256 dp = (remaining * 9_996 / 10_000) * 1e20 / trimSize;
            if (dp == 0) {
                dp = 1;
            }
            if (dp > 1e8) {
                dp = 1e8;
            }
            t = _commitAndExecute(
                LOSS_TRADER, CfdTypes.Side.LONG, trimSize, remaining * 12 / 10 + 200e6, 1e8, false, t
            );
            t = _commitAndExecute(LOSS_TRADER, CfdTypes.Side.LONG, trimSize, 0, 1e8 + dp, true, t);
        }
        lost = pool.totalAssets() - seniorTarget;
    }

    function _arm(Recovery mode) internal returns (Out memory o) {
        _recover(mode);
        o.traderLoss = _tradingLosses();
        (o.projectedSenior, o.projectedJunior) = pool.getPendingDepositTrancheState();

        pool.settleLpEpoch(0, 0);
        (, o.seniorPaid) = _claimAsyncRedeem(seniorVault, redeemEpochId, SENIOR_LP);
        o.juniorValue = juniorVault.convertToAssets(juniorVault.balanceOf(JUNIOR_LP));
        o.poolLeft = pool.totalAssets();
        o.unassigned = pool.unassignedAssets();
    }

    function test_RunbookRecapPaysSeniorTheDeficitOutOfJuniorRevenue() public {
        _reachDeficit();
        uint256 snapshot = vm.snapshotState();
        Out memory a = _arm(Recovery.Runbook);
        assertTrue(vm.revertToState(snapshot));
        Out memory b = _arm(Recovery.DonationThenRecap);

        console2.log("Senior deposit before the incident   ", SENIOR_DEPOSIT);
        console2.log("Senior high-water mark at the deficit", seniorTarget);
        console2.log("trader-claim deficit                 ", deficit);
        console2.log("owner money in both arms             ", deficit + seniorTarget);
        console2.log("later trading revenue, both arms     ", a.traderLoss);
        console2.log("A runbook: projected Senior principal", a.projectedSenior);
        console2.log("A runbook: projected Junior principal", a.projectedJunior);
        console2.log("A runbook: Senior LP paid            ", a.seniorPaid);
        console2.log("A runbook: Junior LP value           ", a.juniorValue);
        console2.log("A runbook: pool cash left            ", a.poolLeft);
        console2.log("A runbook: unassigned assets         ", a.unassigned);
        console2.log("B control: projected Senior principal", b.projectedSenior);
        console2.log("B control: projected Junior principal", b.projectedJunior);
        console2.log("B control: Senior LP paid            ", b.seniorPaid);
        console2.log("B control: Junior LP value           ", b.juniorValue);
        console2.log("B control: pool cash left            ", b.poolLeft);
        console2.log("B control: unassigned assets         ", b.unassigned);
        console2.log("Senior paid more in A by             ", a.seniorPaid - b.seniorPaid);
        console2.log("Junior LP lost in A                  ", b.juniorValue - a.juniorValue);

        assertEq(a.traderLoss, b.traderLoss, "identical trading revenue in both arms");
        assertGe(a.traderLoss, deficit - 60e6, "revenue within 60 USDG of the deficit");

        // Control: Senior is paid its high-water mark (plus the coupon accrued meanwhile), Junior keeps what the
        // waterfall leaves it, and the rest of the owner's money stays in the pool as unassigned assets.
        assertGe(b.seniorPaid, seniorTarget, "control: Senior is paid at least its high-water mark");
        assertLt(b.seniorPaid, seniorTarget + 1_000e6, "control: Senior is paid no more than high-water mark plus coupon");
        assertGt(b.juniorValue, 200_000e6, "control: Junior keeps its remaining principal");
        assertGt(b.unassigned, seniorTarget - 1_000e6, "control: the unused recapitalization stays unassigned");

        // Runbook: Senior is paid its high-water mark plus the deficit, Junior is left with nothing, the pool is empty.
        assertGe(a.seniorPaid, seniorTarget + deficit - 100e6, "runbook: Senior is paid high-water mark plus deficit");
        assertEq(a.juniorValue, 0, "runbook: Junior is left with nothing");
        assertLt(a.poolLeft, 10e6, "runbook: the pool is paid out");
        assertEq(a.unassigned, 0, "runbook: nothing stays unassigned");
        assertGe(a.seniorPaid - b.seniorPaid, deficit - 2_000e6, "Senior is overpaid by about the deficit");
    }
}
```
