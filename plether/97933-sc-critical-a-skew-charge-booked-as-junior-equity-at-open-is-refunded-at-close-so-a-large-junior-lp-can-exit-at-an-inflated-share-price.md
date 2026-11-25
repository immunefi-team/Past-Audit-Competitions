# \#97933 [SC-Critical] A skew charge booked as Junior equity at open is refunded at close, so a large Junior LP can exit at an inflated share price

**Submitted on Nov 3rd 2026 at 23:50:02 UTC by @humbleservant for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97933
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

When a trader's open moves skew away from balance, the skew charge (VPI) it pays is booked straight into Junior principal at the next reconcile, and Junior redemptions funded at that settlement are priced on it. The same position takes that charge back as a rebate when it closes and moves skew back, and terminal NAV records no liability for that pending reversal. A party that holds a large share of the Junior tranche and trades the position can open just before an epoch boundary, have its own Junior redemption funded at the inflated price, and close right after: the rebate is then paid out of a pool that only the remaining Junior LPs own. At the release configuration with live oracle confidence, a holder of 80% of Junior nets about 921 USDG per event while the other Junior LPs lose about 1,398 USDG, and four events in ten hours move 3,694 USDG to the holder and cost the remaining Junior LPs 5,601 USDG, 1.4% of their 400,000 USDG. I found this while comparing how the protocol protects LP pricing against negative VPI with how it treats positive VPI.

## Vulnerability Details

### 1. The open's charge becomes distributable Junior equity at once

The open settlement sends the trade cost, net of the protocol fee, to the pool as revenue with the cash already arrived (`CfdEngineSettlementSidecar.sol` L446-464):

```solidity
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

`HousePool.recordClaimantInflow` adds it to `accountedAssets` (L640-642). The next reconcile treats it as waterfall revenue and distributes it into Junior principal (`HousePoolReconcilePlanLib.sol` L92-98), and the Junior redemption phase of that same settlement prices burned shares on that principal (`HousePool.sol` L840-841: `juniorPhase.pricingPrincipal = juniorPrincipal`).

### 2. The same charge is refundable to the same position

On close, `CloseAccountingLib` charges or rebates VPI on the skew the close moves (L103-111) and clamps only so that the account's lifetime VPI never goes below zero (L116-118):

```solidity
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

A close that moves skew back to where the open found it therefore rebates up to the full charge that position paid. The rebate is paid from free pool cash (`CfdEnginePlanLib.sol` L972-977; `CfdEngineSettlementSidecar.sol` L442-444, `payOut`).

### 3. Terminal NAV carries no liability for that reversal

`ACCOUNTING_SPEC.md` L1064-1066: "anticipated carry, VPI, fees, spreads, and other action economics stay outside terminal `K`". So between the open and the close, Junior principal includes the charge in full while nothing records that the same position can take it back. My first test shows signed terminal LP equity rising by exactly the charge, 2,398.401065 USDG, at the open, with the terminal price delta unchanged.

### 4. The sequence

1. The large Junior holder queues a redemption for the next epoch (requests close 5 minutes before the boundary).
2. About two minutes before the boundary its trading account opens a position that moves skew away from balance, paying charge C into the pool.
3. A keeper settles the epoch. Reconcile assigns C to Junior principal, and the holder's redemption is funded at a share price that includes C.
4. Right after settlement the position closes, moving skew back. The clamp lets the close rebate C, which the pool pays from cash that now belongs to the Junior LPs who did not exit.

The holder keeps its pro-rata share of C as part of its exit. The trading account gets C back, less two execution fees. The remaining Junior LPs fund the rebate. My second test isolates the timing. With the large LP's exit funded after the close instead, both cohorts end with the same per-share result (within 1 atom). Funded between the open and the close, the same round trip moves +1,534.27 USDG to the large LP and -1,530.45 USDG from the remaining Junior LPs.

### 5. Why the design documents do not cover this

- The protocol already prevents exactly this for the negative side. `WHITEPAPER.md` L887-888: "The reserve is excluded from the terminal price curve, so LPs cannot withdraw it as marked equity before the matching position obligation settles." A positive charge that the same position can take back is the mirror case, and it has no matching protection.
- The collection-time ownership policy in `ACCOUNTING_SPEC.md` L731-740 is written for carry: debt the trader owes, collected once. A skew charge is different. It is refundable to the payer, and the payer decides when.
- The clamp's own comment says it exists to prevent "LP sandwich attacks" (`CloseAccountingLib.sol` L112-115). `WHITEPAPER.md` L1793 says synchronized epoch pricing "removes the entry/exit NAV mismatch", and `SECURITY.md` L859-860 promises that LPs do not inherit old trader obligations at a different NAV. Here an exit is priced on value that an open position's obligation is about to take back.
- This is not the public AI-scan item SCAN-31. That item enlarges a rebate beyond the normal VPI curve by shrinking pool depth around a close. Here the rebate stays on the normal curve, and the value moves through the redemption price, so fixing SCAN-31's depth denominator leaves this intact. In my timing comparison, 1,532.35 USDG of the transfer goes through the redemption price and only 1.91 USDG through the trading leg's rebate.

### 6. Conditions, stated plainly

- The holder needs a large share of the Junior tranche: with 1 bp oracle confidence and small price moves between the open, the settlement and the close, the holder's net turns positive at about 42% of Junior (my fourth test: -292.17 USDG at 30%, -49.46 at 40%, +71.89 at 45%, +193.24 at 50%, +435.95 at 60% and +921.37 at 80%).
- The Junior exit covenant caps a Junior redemption at `J - (X + R) / 4` (`HousePool.sol` L1205-1216, `_juniorSettlementCapacity` with the 80% Senior share cap), so Senior must be at most about 70% of the pool for a large Junior exit to fund.
- The open needs skew headroom on the side it adds to, and the market must be live (no frozen window).
- Each event needs a queued exit, the round trip and a re-deposit, so one wallet repeats it about once every three epochs.

### 7. Suggested fix

Keep positive accrued VPI of open positions out of distributable tranche equity until the position closes, the same way the negative-VPI reserve is kept out of the terminal price curve. Equivalently, count the open positions' refundable accrued VPI as a liability in terminal NAV and in the redemption pricing input.

## Impact Details

Impact: "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield". The Junior LPs who do not exit end each event with less principal than the same timeline without the round trip, and the exiting holder ends with more, measured as settled USDG from `claimRedeem`.

Measured on the real contracts at the release configuration (Senior 1,000,000 USDG, Junior 2,000,000 USDG, a 1,200,000 USDG LONG at the skew cap, charge 2,398.40 USDG, 1 bp oracle confidence, both mirrored price paths):

| Junior share of the large LP | Large LP net per event (mean of both paths) | Remaining Junior LPs per event |
|---|---|---|
| 40% | -49.46 USDG | -430.37 USDG |
| 50% | +193.24 USDG | -672.76 USDG |
| 80% | +921.37 USDG | -1,398.06 USDG |

Repeated on a flat book at 80% (exits at 10:00, 13:00, 16:00 and 19:00; each cycle queues the exit, runs the round trip and re-deposits at a fair price):

| Cycle | Large LP net so far (mean) | Remaining Junior LPs so far (mean) |
|---|---|---|
| 1 | +921.37 USDG | -1,398.06 USDG |
| 2 | +1,844.20 USDG | -2,797.59 USDG |
| 3 | +2,768.51 USDG | -4,198.59 USDG |
| 4 | +3,694.28 USDG | -5,601.05 USDG (-1.40% of 400,000) |

- Senior is unchanged in every run. Every paired run conserves value exactly.
- Each event is bounded by the holder's pro-rata share of its own refundable charge, and repeats once the holder re-deposits.
- The code is not deployed, so I understand funds at risk are scaled to the program's minimum. The conditions in section 6 are the honest limits on who can run this: a holder of a large Junior share, with Senior at most about 70% of the pool.

## References

- Frozen competition commit (the code is not deployed): https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8
- Open cost routed to the pool as revenue: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L442-L464
- Revenue inflow accounting: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L624-L650
- Revenue distributed to Junior at reconcile: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L92-L98
- Junior redemption pricing input: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L840-L841
- Close VPI and the lifetime clamp: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118
- Rebate paid from free pool cash: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L972-L977
- Junior exit covenant: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1205-L1216
- `packages/perps/ACCOUNTING_SPEC.md` L731-740 (collection-time ownership for carry) and L1064-1066 (anticipated VPI outside terminal K)
- `packages/perps/WHITEPAPER.md` L811 (VPI beneficiary), L887-888 (negative-VPI reserve kept out of withdrawable equity), L1793 (epoch pricing and the entry/exit NAV mismatch)
- `packages/perps/SECURITY.md` L859-860 and L863
- Public items checked: SCAN-31 (depth-manipulated rebate; different channel), SECMD-VPI-6 (lifetime clamp for rebate-only round trips; not used here), MED-24 and MED-29 (refund epochs and rounding; not used here).

## Proof of Concept

## Proof of Concept

Save the file below as `packages/perps/test/poc/RefundableSkewChargeLpExit.t.sol` in a checkout of commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. It extends the repo's `BasePerpTest`, so the oracle, Router, execution sidecars, engine, clearinghouse, HousePool and both TrancheVaults are the real contracts, with MockUSDG and MockPyth as in the repo's tests. It installs the release configuration from `DEPLOYMENT.md` (constructor values through the fixture's risk-parameter hook, the rest through the owner's documented 72-hour setters) and reads every value back. Every trade is a trader-signed `commitOrder` executed by an unprivileged keeper with Pyth payloads, epochs settle through the Router, every LP result is settled USDG from `claimRedeem` measured against a snapshot of the same timeline without the trades, and every paired run asserts exact conservation. No attack step pranks a privileged role or writes storage.

The five tests:

1. `test_openBooksRefundableChargeAsDistributableEquity`: LP equity rises by exactly the charge at the open with no terminal liability, and the close rebates it (net of the close fee) to the trading account.
2. `test_control_exitAfterTheClose_noTransfer`: with the exit after the close the cohorts end equal per share; with the exit between the open and the close the large LP gains and the remaining LPs lose.
3. `test_exitBetweenOpenAndClose_realisticConfidenceAndPricePath`: 1 bp oracle confidence (adverse fill asserted on both legs) and a small price path in both directions.
4. `test_breakEvenShare`: the same event at 30% to 80% of Junior.
5. `test_repeatsEveryCycle_multiCycleFlatBook`: four consecutive cycles.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {HousePoolEngineViewTypes} from "@plether/perps/interfaces/HousePoolEngineViewTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {Vm} from "forge-std/Vm.sol";

/// @notice A skew charge C paid by a skew-increasing open is booked at once as distributable Junior equity, although
///         the same position's skew-reducing close pays it back as a rebate. A Junior redemption funded at an epoch
///         settlement between that open and that close is priced at a NAV that still contains C, and the close then
///         pays C, less the close fee, back to the trading account out of the Junior LPs who stayed. An LP that holds
///         a share f of Junior and also trades the position takes f x C out through its exit and gets the charge back
///         through the rebate: it nets f x C less the two execution fees, carry and the open's keeper bounty, while
///         the Junior LPs who stay lose f x C less the close fee and carry.
/// @dev Release parameters from DEPLOYMENT.md ("Initial release parameters") are installed through the `_riskParams()`
///      constructor hook and the owner's 72-hour setters, then read back and asserted. After setUp every step is an
///      unprivileged call on a production entry point: the trader's V3 `commitOrder`, a keeper's `executeOrder`,
///      `updateMarkPrice` and `settleLpEpoch` on the Router, and each LP's own vault requests and claims. LP outcomes
///      are settled USDG (balance change across `claimRedeem`), every case is paired with a snapshot control that runs
///      the same LP timeline without the two trades, and every pair conserves USDG exactly.
///      Actors: `lpWhale` (large Junior LP, also owns the `trader` account), `lpOther` (remaining Junior LP),
///      `bystander` (Senior LP), `keeper` (unprivileged executor).
contract RefundableSkewChargeLpExitTest is BasePerpTest {

    uint256 internal constant PRICE = 1e8;
    /// @dev Monday 2024-03-11 06:00 UTC, one week after the fixture's setup time (also a Monday, 06:00 UTC).
    uint256 internal constant MONDAY = 1_710_136_800;
    uint256 internal constant JUNIOR_TOTAL = 2_000_000e6;
    uint256 internal constant SENIOR_TOTAL = 1_000_000e6;
    uint256 internal constant WHALE_80 = 1_600_000e6;
    /// @dev 1,200,000 USDG notional at 1.00, just under the 0.4 skew cap on 3,002,000 USDG of depth (both tranches plus
    ///      the fixture's two 1,000 USDG seeds).
    uint256 internal constant TRADE_SIZE = 1_200_000e18;
    uint256 internal constant MARGIN = 20_000e6;
    uint256 internal constant TRADER_DEPOSIT = 50_000e6;
    /// @dev 4 bps execution fee on 1,200,000 USDG notional.
    uint256 internal constant FEE = 480e6;
    /// @dev Pyth confidence of 1 bp on a 1.00 price (expo -8) and the adverse execution shift it causes:
    ///      confidence x 2,500 / 10,000 = 0.25 bp.
    uint64 internal constant CONF_1BP = 10_000;
    uint256 internal constant SHIFT_1BP = 2500;

    bytes32 internal constant CARRY_TOPIC = keccak256("CarryRealized(address,uint256,uint256,uint256,uint256)");
    bytes32 internal constant REBATE_TOPIC = keccak256("ActionRebateSettled(address,uint256,uint256,uint256)");
    bytes32 internal constant CLOSED_TOPIC = keccak256("PositionClosed(address,uint8,uint256,uint256,int256)");

    address internal keeper;
    address internal lpWhale;
    address internal lpOther;
    address internal bystander;
    address internal trader;

    /// @dev Neutral basket price at the open, the epoch settlement, the close and every later mark, plus the Pyth
    ///      confidence applied to order executions.
    struct Path {
        uint256 pOpen;
        uint256 pSettle;
        uint256 pClose;
        uint256 pLater;
        uint64 conf;
    }

    /// @dev One run of up to four exit cycles. Index k is cycle k.
    struct Run {
        uint256[4] whaleIn; // USDG the large LP holds in Junior when the cycle starts
        uint256[4] whaleOut; // settled USDG from the large LP's exit in the cycle
        uint256[4] traderOut; // settled USDG the trading account withdraws after the close
        uint256[4] otherOut; // settled USDG the remaining Junior LP receives exiting one epoch after the cycle
        uint256[4] seniorOut; // settled USDG the Senior LP receives at that same epoch
        int256[4] tail; // treasury, keeper and trading-account clearinghouse balances plus pool and vault USDG
        uint256 whaleShares; // Junior shares the large LP redeems in the first cycle
        uint256 otherShares; // Junior shares the remaining LP redeems after the first cycle
        int256 charge; // lifetime skew charge of the first open
        uint256 entryPrice; // execution price of the first open
        uint256 closePrice; // execution price of the first close
        uint256 endTime; // settlement time of the last exit read
    }

    struct CloseLog {
        uint256 assessed;
        uint256 paid;
        uint256 waived;
        uint256 carry;
        uint256 price;
        int256 pnl;
    }

    function setUp() public override {
        super.setUp();
        keeper = makeAddr("keeper");
        lpWhale = makeAddr("lpWhale");
        lpOther = makeAddr("lpOther");
        bystander = makeAddr("bystander");
        trader = makeAddr("trader");
        _applyReleaseConfig();
        vm.warp(MONDAY);
        assertFalse(engine.isFadWindow(), "Monday is outside FAD");
        assertFalse(engine.isOracleFrozen(), "Monday is outside the frozen window");
        _assertReleaseConfig();
    }

    // ---------------- Release configuration ----------------

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

    /// @dev The tranches are funded by named LPs through the asynchronous deposit path, not by the fixture.
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _applyReleaseConfig() internal {
        IHousePool.PoolConfig memory poolConfig = _currentPoolConfig();
        poolConfig.maxSeniorExposureAssets = 40_000_000e6;
        poolConfig.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(poolConfig);
        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        routerConfig.minOpenNotionalAssets = 1000e6;
        routerConfig.adverseConfidenceMultiplierBps = 2500;
        routerAdmin.proposeRouterConfig(routerConfig);
        uint256 activation = pool.poolConfigActivationTime();
        if (routerAdmin.routerConfigActivationTime() > activation) {
            activation = routerAdmin.routerConfigActivationTime();
        }
        vm.warp(activation);
        pool.finalizePoolConfig();
        routerAdmin.finalizeRouterConfig();
    }

    function _assertReleaseConfig() internal view {
        CfdTypes.RiskParams memory rp = _engineRiskConfig().riskParams;
        assertEq(rp.vpiFactor, 0.01e18, "vpiFactor");
        assertEq(rp.maxSkewRatio, 0.4e18, "maxSkewRatio");
        assertEq(rp.maintMarginBps, 10, "maintMarginBps");
        assertEq(rp.initMarginBps, 20, "initMarginBps");
        assertEq(rp.fadMarginBps, 300, "fadMarginBps");
        assertEq(rp.baseCarryBps, 500, "baseCarryBps");
        assertEq(rp.minBountyAssets, 1e6, "minBountyAssets");
        assertEq(rp.bountyBps, 10, "bountyBps");
        assertEq(rp.keeperShareBps, 5000, "keeperShareBps");
        assertEq(rp.protocolShareBps, 0, "protocolShareBps");
        assertEq(engine.executionFeeBps(), 4, "executionFeeBps");
        assertEq(engine.settlementBufferBps(), 25, "settlementBufferBps");
        assertEq(engine.frozenCloseSpreadBps(), 50, "frozenCloseSpreadBps");
        assertEq(engine.CAP_PRICE(), 2e8, "CAP_PRICE");
        assertEq(router.minOpenNotionalAssets(), 1000e6, "minOpenNotionalAssets");
        assertEq(router.pletherOracle().adverseConfidenceMultiplierBps(), 2500, "adverseConfidenceMultiplierBps");
        assertEq(router.pletherOracle().basketMaxConfidenceRatioBps(), 10, "basketMaxConfidenceRatioBps");
        assertEq(router.maxPendingOrders(), 5, "maxPendingOrders");
        assertEq(router.positionProtectionTriggerBountyAssets(), 200_000, "positionProtectionTriggerBountyAssets");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "maxSeniorExposureAssets");
        assertEq(pool.maxSeniorShareBps(), 8000, "maxSeniorShareBps");
        assertEq(pool.seniorRateBps(), 800, "seniorRateBps (HousePool constructor default)");
        assertEq(pool.markStalenessLimit(), 60, "markStalenessLimit (HousePool constructor default)");
        assertEq(pool.juniorFrozenLpFeeBps(), 75, "juniorFrozenLpFeeBps");
        assertEq(pool.seniorFrozenLpFeeBps(), 25, "seniorFrozenLpFeeBps");
    }

    // ---------------- Production Router path ----------------

    /// @dev via_ir folds repeated TIMESTAMP reads across vm.warp, so the clock is read through the cheatcode.
    function _warpTo(
        uint256 timestamp
    ) internal {
        if (vm.getBlockTimestamp() < timestamp) {
            vm.warp(timestamp);
        }
    }

    /// @dev The fixture's Pyth update (one block and one second later, publish time after the pending commit) with a
    ///      per-feed confidence on the order-execution price. Marks and LP settlements use the neutral encoded price.
    function _pyth(
        uint256 price,
        uint64 conf
    ) internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = _mockHistoricalPublishTime();
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(),
            SafeCast.toInt64(SafeCast.toInt256(price)),
            conf,
            int32(-8),
            publishTime,
            publishTime == 0 ? 0 : publishTime - 1
        );
        updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
    }

    function _mark(
        uint256 price
    ) internal {
        bytes[] memory updateData = _pyth(price, 0);
        vm.prank(keeper);
        router.updateMarkPrice(updateData);
    }

    function _settle(
        uint256 price
    ) internal {
        bytes[] memory updateData = _pyth(price, 0);
        vm.prank(keeper);
        router.settleLpEpoch(updateData);
    }

    function _fundTradingAccount() internal {
        settlementToken.mint(trader, TRADER_DEPOSIT);
        vm.startPrank(trader);
        settlementToken.approve(address(clearinghouse), TRADER_DEPOSIT);
        clearinghouse.deposit(trader, TRADER_DEPOSIT);
        vm.stopPrank();
    }

    /// @dev A V3 LONG request with the trader's loosest limits; the protocol's own checks are untouched.
    function _request(
        uint256 margin,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = router.maxExecutionWindowSeconds();
        request.clientOrderId = bytes32(uint256(router.nextCommitId()));
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = TRADE_SIZE;
        request.marginDelta = margin;
        request.targetPrice = isClose ? type(uint256).max : 1;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + window),
            executionWindowSeconds: uint32(window),
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

    /// @dev The trader commits; an unprivileged keeper executes the queue head with a fresh Pyth payload.
    function _tradeLong(
        uint256 margin,
        bool isClose,
        uint256 price,
        uint64 conf
    ) internal {
        OrderV3Types.OrderRequest memory request = _request(margin, isClose);
        vm.prank(trader);
        uint64 orderId = OrderRouter(address(router)).commitOrder(request);
        bytes[] memory updateData = _pyth(price, conf);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, updateData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    function _withdrawAll() internal returns (uint256 out) {
        uint256 free = clearinghouse.getAccountAssetBuckets(trader).freeSettlementAssets;
        uint256 before = settlementToken.balanceOf(trader);
        vm.prank(trader);
        clearinghouse.withdraw(trader, free);
        out = settlementToken.balanceOf(trader) - before;
        assertEq(clearinghouse.balanceAssets(trader), 0, "nothing stays in the trading account");
    }

    // ---------------- LP lifecycle ----------------

    function _deposit(
        TrancheVault vault,
        address lp,
        uint256 amount,
        bool mint
    ) internal returns (uint256 requestId) {
        if (mint) {
            settlementToken.mint(lp, amount);
        }
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
    }

    function _claimDeposit(
        TrancheVault vault,
        address lp,
        uint256 requestId
    ) internal {
        uint256 assets = vault.claimableDepositRequest(requestId, lp);
        assertGt(assets, 0, "deposit activated");
        vm.prank(lp);
        vault.claimDeposit(requestId, assets, lp, lp);
    }

    function _requestExit(
        TrancheVault vault,
        address lp
    ) internal returns (uint256 requestId, uint256 shares) {
        shares = vault.balanceOf(lp);
        vm.prank(lp);
        requestId = vault.requestRedeem(shares, lp, lp);
    }

    function _claimExit(
        TrancheVault vault,
        address lp,
        uint256 requestId
    ) internal returns (uint256 assets) {
        uint256 shares = vault.claimableRedeemRequest(requestId, lp);
        assertGt(shares, 0, "redemption funded");
        uint256 before = settlementToken.balanceOf(lp);
        vm.prank(lp);
        vault.claimRedeem(requestId, shares, lp, lp);
        assets = settlementToken.balanceOf(lp) - before;
    }

    /// @dev Junior requests made at 06:00 activate at 07:00; the Senior request (capacity bounded by active Junior) is
    ///      made at 07:00 and activates at 08:00. A keeper settles each epoch through the Router.
    function _seed(
        uint256 whaleJunior
    ) internal {
        uint256 whaleId = _deposit(juniorVault, lpWhale, whaleJunior, true);
        uint256 otherId = _deposit(juniorVault, lpOther, JUNIOR_TOTAL - whaleJunior, true);
        _warpTo(pool.lpEpochStart(otherId));
        _settle(PRICE);
        _claimDeposit(juniorVault, lpWhale, whaleId);
        _claimDeposit(juniorVault, lpOther, otherId);
        uint256 seniorId = _deposit(seniorVault, bystander, SENIOR_TOTAL, true);
        _warpTo(pool.lpEpochStart(seniorId));
        _settle(PRICE);
        _claimDeposit(seniorVault, bystander, seniorId);
    }

    // ---------------- Reads ----------------

    /// @dev Signed terminal LP equity exactly as LP settlement prices it: physical assets - trader claims + terminal
    ///      price delta, and the terminal price delta on its own.
    function _equity() internal view returns (int256 equity, int256 priceDelta) {
        HousePoolEngineViewTypes.HousePoolInputSnapshot memory s =
            engineProtocolLens.getHousePoolInputSnapshot(pool.markStalenessLimit());
        priceDelta = s.terminalLpPriceDeltaAssets;
        equity = int256(s.physicalAssets) - int256(s.traderClaimBalanceAssets) + priceDelta;
    }

    function _tail() internal view returns (int256) {
        return int256(
            clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT) + clearinghouse.balanceAssets(keeper)
                + clearinghouse.balanceAssets(trader) + pool.rawAssets()
                + settlementToken.balanceOf(address(juniorVault)) + settlementToken.balanceOf(address(seniorVault))
        );
    }

    /// @dev Backstop for the conservation identity: the holders it reads are every holder of USDG.
    function _assertAllUsdgCounted() internal view {
        uint256 held = settlementToken.balanceOf(lpWhale) + settlementToken.balanceOf(lpOther)
            + settlementToken.balanceOf(bystander) + settlementToken.balanceOf(trader)
            + settlementToken.balanceOf(address(clearinghouse)) + pool.rawAssets()
            + settlementToken.balanceOf(address(juniorVault)) + settlementToken.balanceOf(address(seniorVault));
        assertEq(held, settlementToken.totalSupply(), "every USDG holder is counted");
    }

    /// @dev Decodes the trading account's close: carry realized at commit and execution, the net rebate settlement and
    ///      the close execution price and price PnL. Logs must be recorded from before the close commit.
    function _closeLog() internal returns (CloseLog memory l) {
        Vm.Log[] memory logs = vm.getRecordedLogs();
        bytes32 account = bytes32(uint256(uint160(trader)));
        for (uint256 i; i < logs.length; ++i) {
            if (logs[i].topics.length < 2 || logs[i].topics[1] != account) {
                continue;
            }
            bytes32 topic = logs[i].topics[0];
            if (topic == CARRY_TOPIC) {
                (uint256 carry,,,) = abi.decode(logs[i].data, (uint256, uint256, uint256, uint256));
                l.carry += carry;
            } else if (topic == REBATE_TOPIC) {
                (l.assessed, l.paid, l.waived) = abi.decode(logs[i].data, (uint256, uint256, uint256));
            } else if (topic == CLOSED_TOPIC) {
                (,, l.price, l.pnl) = abi.decode(logs[i].data, (uint8, uint256, uint256, int256));
            }
        }
    }

    // ---------------- Scenario ----------------

    /// @dev Up to four exit cycles on a flat book. Cycle k: the large LP queues a full Junior exit ten minutes before
    ///      the epoch boundary at 10:00 + 3k h; with `trade`, its trading account opens LONG at the skew cap two
    ///      minutes before the boundary, a keeper settles the epoch (funding the exit), and the LONG closes right
    ///      after. With `exitAfterClose` the exit is instead queued for the following epoch together with the other
    ///      LPs, and the boundary only gets a keeper mark. After each cycle a snapshot reads the settled exits of every
    ///      LP still in the pool one epoch later (no trade around it); a multi-cycle run then re-deposits the large
    ///      LP's proceeds for that epoch (fair price, no position open) and continues. Cycles are three hours apart:
    ///      the exit, the re-deposit activated one epoch later, then the 1-hour deposit cooldown before the next exit.
    function _run(
        uint256 whaleJunior,
        uint256 cycles,
        bool trade,
        bool exitAfterClose,
        Path memory p
    ) internal returns (Run memory r) {
        _seed(whaleJunior);
        uint256 stake = whaleJunior;
        for (uint256 k; k < cycles; ++k) {
            uint256 epoch = MONDAY + 4 hours + k * 3 hours;
            r.whaleIn[k] = stake;
            uint256 exitId;
            if (!exitAfterClose) {
                _warpTo(epoch - 10 minutes);
                uint256 shares;
                (exitId, shares) = _requestExit(juniorVault, lpWhale);
                assertEq(pool.lpEpochStart(exitId), epoch, "the exit matures at the next boundary");
                if (k == 0) {
                    r.whaleShares = shares;
                }
            }
            if (trade) {
                _warpTo(epoch - 2 minutes);
                _fundTradingAccount();
                _tradeLong(MARGIN, false, p.pOpen, p.conf);
                if (k == 0) {
                    (,, r.entryPrice,,,, r.charge) = engine.positions(trader);
                }
            }
            _warpTo(epoch);
            if (exitAfterClose) {
                _mark(p.pSettle);
            } else {
                _settle(p.pSettle);
            }
            if (trade) {
                vm.recordLogs();
                _tradeLong(0, true, p.pClose, p.conf);
                uint256 closePrice = _closeLog().price;
                if (k == 0) {
                    r.closePrice = closePrice;
                }
                r.traderOut[k] = _withdrawAll();
            }
            if (!exitAfterClose) {
                r.whaleOut[k] = _claimExit(juniorVault, lpWhale, exitId);
            }
            _probe(r, k, epoch, exitAfterClose, p.pLater);
            if (k + 1 < cycles) {
                stake = r.whaleOut[k];
                _warpTo(epoch + 5 minutes);
                uint256 depositId = _deposit(juniorVault, lpWhale, stake, false);
                _warpTo(epoch + 1 hours);
                _settle(p.pLater);
                _claimDeposit(juniorVault, lpWhale, depositId);
            }
        }
    }

    /// @dev Every LP still in the pool exits at the epoch after cycle k (Senior is funded first, then Junior). Taken on
    ///      a snapshot so that a multi-cycle run can continue; after the last cycle it is the end of the run.
    function _probe(
        Run memory r,
        uint256 k,
        uint256 epoch,
        bool whaleToo,
        uint256 price
    ) internal {
        uint256 snap = vm.snapshotState();
        _warpTo(epoch + 5 minutes);
        uint256 whaleId;
        if (whaleToo) {
            (whaleId, r.whaleShares) = _requestExit(juniorVault, lpWhale);
        }
        (uint256 otherId, uint256 otherShares) = _requestExit(juniorVault, lpOther);
        (uint256 seniorId,) = _requestExit(seniorVault, bystander);
        assertEq(pool.lpEpochStart(otherId), epoch + 1 hours, "the remaining LPs exit at the following epoch");
        _warpTo(epoch + 1 hours);
        _settle(price);
        r.endTime = vm.getBlockTimestamp();
        r.seniorOut[k] = _claimExit(seniorVault, bystander, seniorId);
        r.otherOut[k] = _claimExit(juniorVault, lpOther, otherId);
        if (whaleToo) {
            r.whaleOut[k] = _claimExit(juniorVault, lpWhale, whaleId);
        }
        if (k == 0) {
            r.otherShares = otherShares;
        }
        r.tail[k] = _tail();
        _assertAllUsdgCounted();
        vm.revertToState(snap);
    }

    /// @dev Control (same LP timeline, no trades) and trading run from the same state. Exact conservation after every
    ///      cycle: the large LP's wallet, the trading account's withdrawals less its deposits, the remaining Junior LP,
    ///      the Senior LP, and the tail (treasury, keeper, pool and vault balances) sum to zero against the control.
    function _pair(
        uint256 whaleJunior,
        uint256 cycles,
        bool exitAfterClose,
        Path memory p
    ) internal returns (Run memory c, Run memory a) {
        uint256 snap = vm.snapshotState();
        c = _run(whaleJunior, cycles, false, exitAfterClose, p);
        vm.revertToState(snap);
        a = _run(whaleJunior, cycles, true, exitAfterClose, p);
        vm.revertToState(snap);
        int256 traderFlow;
        for (uint256 k; k < cycles; ++k) {
            traderFlow += _d(a.traderOut[k], TRADER_DEPOSIT);
            int256 residual = _d(a.whaleOut[k], c.whaleOut[k]) + traderFlow + _rem(c, a, k)
                + _d(a.seniorOut[k], c.seniorOut[k]) + a.tail[k] - c.tail[k];
            assertEq(residual, 0, "exact conservation across control and trades");
            assertEq(a.seniorOut[k], c.seniorOut[k], "Senior LP unchanged");
        }
    }

    /// @dev Both mirrored price paths at 1 bp Pyth confidence.
    function _mirrored(
        uint256 whaleJunior,
        uint256 cycles
    ) internal returns (Run memory cUp, Run memory aUp, Run memory cDown, Run memory aDown) {
        (cUp, aUp) = _pair(whaleJunior, cycles, false, _path(true));
        (cDown, aDown) = _pair(whaleJunior, cycles, false, _path(false));
    }

    /// @dev +2 bp by the settlement, +3 bp at the close, +1 bp for every later mark; the mirror path moves down.
    function _path(
        bool up
    ) internal pure returns (Path memory p) {
        p.pOpen = PRICE;
        p.pSettle = up ? PRICE + 20_000 : PRICE - 20_000;
        p.pClose = up ? PRICE + 30_000 : PRICE - 30_000;
        p.pLater = up ? PRICE + 10_000 : PRICE - 10_000;
        p.conf = CONF_1BP;
    }

    function _d(
        uint256 x,
        uint256 y
    ) internal pure returns (int256) {
        return int256(x) - int256(y);
    }

    /// @dev Large LP net for cycle k: its LP return against the control's plus the trading account's result.
    function _net(
        Run memory c,
        Run memory a,
        uint256 k
    ) internal pure returns (int256) {
        return _d(a.whaleOut[k], a.whaleIn[k]) - _d(c.whaleOut[k], c.whaleIn[k]) + _d(a.traderOut[k], TRADER_DEPOSIT);
    }

    /// @dev Remaining Junior LP's settled exit after cycle k against the control (cumulative over the cycles).
    function _rem(
        Run memory c,
        Run memory a,
        uint256 k
    ) internal pure returns (int256) {
        return _d(a.otherOut[k], c.otherOut[k]);
    }

    // ---------------- Tests ----------------

    /// @notice The open books the skew charge as LP equity with no offsetting liability, the Junior exit funded at the
    ///         boundary is paid out of that equity, and the close reverses the charge: the pool pays it back to the
    ///         same trading account as a rebate, net of the close fee that the protocol nets inside the rebate.
    function test_openBooksRefundableChargeAsDistributableEquity() public {
        _seed(WHALE_80);
        _warpTo(MONDAY + 3 hours + 50 minutes);
        (uint256 exitId,) = _requestExit(juniorVault, lpWhale);
        _warpTo(MONDAY + 3 hours + 58 minutes);
        _fundTradingAccount();
        _mark(PRICE);
        (int256 e0, int256 delta0) = _equity();
        uint256 treasury0 = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);

        _tradeLong(MARGIN, false, PRICE, 0);
        (,,,,,, int256 charge) = engine.positions(trader);
        (int256 e1, int256 delta1) = _equity();
        // vpiFactor x skew^2 / (2 x depth) = 0.01 x 1,200,000^2 / (2 x 3,002,000) = 2,398.401065 USDG, rounded down.
        assertEq(charge, 2_398_401_065, "skew charge paid at the open");
        assertEq(e1 - e0, charge, "open: signed terminal LP equity rises by exactly the charge");
        assertEq(delta1, delta0, "open: terminal price delta unchanged, no liability for the refundable charge");
        assertEq(clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT) - treasury0, FEE, "open: fee to treasury");

        _warpTo(MONDAY + 4 hours);
        _settle(PRICE);
        (int256 e2,) = _equity();

        uint256 balance0 = clearinghouse.balanceAssets(trader);
        vm.recordLogs();
        _tradeLong(0, true, PRICE, 0);
        CloseLog memory cl = _closeLog();
        (int256 e3,) = _equity();
        assertEq(cl.assessed, uint256(charge) - FEE, "close: the charge comes back less the close fee");
        assertEq(cl.paid, cl.assessed, "close: rebate paid in full");
        assertEq(cl.waived, 0, "close: nothing waived");
        assertEq(cl.pnl, 0, "close: no price PnL at a constant price");
        assertEq(
            int256(clearinghouse.balanceAssets(trader)) - int256(balance0),
            int256(cl.paid) - int256(cl.carry),
            "close: the trading account receives the rebate, less carry"
        );
        assertEq(e3 - e2, int256(cl.carry) - int256(cl.paid), "close: the rebate leaves LP equity");

        uint256 exitPaid = _claimExit(juniorVault, lpWhale, exitId);
        assertEq(e1 - e2, int256(exitPaid), "the exit was paid out of equity that still held the charge");

        emit log_named_int("skew charge booked as LP equity at the open", charge);
        emit log_named_uint("rebate paid back to the trading account at the close", cl.paid);
        emit log_named_uint("close fee netted inside the rebate", FEE);
        emit log_named_uint("carry paid by the trading account", cl.carry);
        emit log_named_uint("large LP exit funded between the open and the close", exitPaid);
    }

    /// @notice Negative control: the same round trip with the large LP's exit funded at the epoch after the close.
    ///         Both Junior cohorts then end at the same per-share result against the no-trade control (each keeps its
    ///         pro-rata share of the close fee the pool retains) and the round trip only costs the large LP its fees.
    ///         Moving the exit between the open and the close is what moves value between the cohorts.
    function test_control_exitAfterTheClose_noTransfer() public {
        Path memory flat = Path({pOpen: PRICE, pSettle: PRICE, pClose: PRICE, pLater: PRICE, conf: 0});
        (Run memory cIn, Run memory aIn) = _pair(WHALE_80, 1, false, flat);
        (Run memory cAfter, Run memory aAfter) = _pair(WHALE_80, 1, true, flat);

        int256 whaleAfter = _d(aAfter.whaleOut[0], cAfter.whaleOut[0]);
        int256 otherAfter = _rem(cAfter, aAfter, 0);
        int256 netAfter = _net(cAfter, aAfter, 0);
        assertEq(aAfter.whaleShares, cAfter.whaleShares, "same large LP shares in both runs");
        assertEq(aAfter.otherShares, cAfter.otherShares, "same remaining LP shares in both runs");
        assertApproxEqAbs(
            otherAfter,
            whaleAfter * int256(aAfter.otherShares) / int256(aAfter.whaleShares),
            1,
            "exit after the close: same per-share result for both cohorts, nothing moves between them"
        );
        assertGt(otherAfter, 0, "exit after the close: the remaining LP keeps its share of the retained close fee");
        assertLt(netAfter, 0, "exit after the close: the round trip only costs the large LP its fees");

        int256 netIn = _net(cIn, aIn, 0);
        int256 otherIn = _rem(cIn, aIn, 0);
        assertGt(netIn, 0, "exit between open and close: the large LP nets a gain");
        assertLt(otherIn, 0, "exit between open and close: the remaining LP ends below the control");

        emit log_named_int("exit after the close: large LP exit delta", whaleAfter);
        emit log_named_int("exit after the close: remaining Junior LP delta", otherAfter);
        emit log_named_int("exit after the close: large LP net", netAfter);
        emit log_named_int("exit after the close: trading result", _d(aAfter.traderOut[0], TRADER_DEPOSIT));
        emit log_named_int("exit between: large LP exit delta", _d(aIn.whaleOut[0], cIn.whaleOut[0]));
        emit log_named_int("exit between: remaining Junior LP delta", otherIn);
        emit log_named_int("exit between: large LP net", netIn);
        emit log_named_int("exit between: trading result", _d(aIn.traderOut[0], TRADER_DEPOSIT));
        emit log_named_int("moved by the exit timing alone, large LP net", netIn - netAfter);
        emit log_named_int("moved by the exit timing alone, remaining Junior LP", otherIn - otherAfter);
    }

    /// @notice Senior 1.0M, Junior 2.0M with the large LP at 80%, no other position. 1 bp live Pyth confidence on both
    ///         executions and a small price path between the open, the settlement and the close, in both directions.
    function test_exitBetweenOpenAndClose_realisticConfidenceAndPricePath() public {
        (Run memory cUp, Run memory aUp, Run memory cDown, Run memory aDown) = _mirrored(WHALE_80, 1);
        // A LONG profits when the basket falls, so the adverse shift opens it lower and closes it higher.
        assertEq(aUp.entryPrice, PRICE - SHIFT_1BP, "open shifted by the confidence (up path)");
        assertEq(aUp.closePrice, PRICE + 30_000 + SHIFT_1BP, "close shifted by the confidence (up path)");
        assertEq(aDown.entryPrice, PRICE - SHIFT_1BP, "open shifted by the confidence (down path)");
        assertEq(aDown.closePrice, PRICE - 30_000 + SHIFT_1BP, "close shifted by the confidence (down path)");
        assertGt(aUp.charge, 0, "the open paid a skew charge");

        int256 up = _net(cUp, aUp, 0);
        int256 down = _net(cDown, aDown, 0);
        int256 otherUp = _rem(cUp, aUp, 0);
        int256 otherDown = _rem(cDown, aDown, 0);
        assertGt(up, 0, "large LP net gain (up path)");
        assertGt(down, 0, "large LP net gain (down path)");
        assertLt(otherUp, 0, "remaining Junior LP below the control (up path)");
        assertLt(otherDown, 0, "remaining Junior LP below the control (down path)");

        emit log_named_int("skew charge at the open", aUp.charge);
        emit log_named_int("large LP exit delta, up path", _d(aUp.whaleOut[0], cUp.whaleOut[0]));
        emit log_named_int("trading result, up path", _d(aUp.traderOut[0], TRADER_DEPOSIT));
        emit log_named_int("large LP net, up path", up);
        emit log_named_int("large LP exit delta, down path", _d(aDown.whaleOut[0], cDown.whaleOut[0]));
        emit log_named_int("trading result, down path", _d(aDown.traderOut[0], TRADER_DEPOSIT));
        emit log_named_int("large LP net, down path", down);
        emit log_named_int("large LP net, mean of both paths", (up + down) / 2);
        emit log_named_int("remaining Junior LP delta, up path", otherUp);
        emit log_named_int("remaining Junior LP delta, down path", otherDown);
        emit log_named_int("remaining Junior LP delta, mean of both paths", (otherUp + otherDown) / 2);
    }

    /// @notice The same event under the same conditions at six shares of Junior (mean of the two price paths).
    function test_breakEvenShare() public {
        uint16[6] memory shareBps = [3000, 4000, 4500, 5000, 6000, 8000];
        int256 previous = type(int256).min;
        for (uint256 i; i < shareBps.length; ++i) {
            (Run memory cUp, Run memory aUp, Run memory cDown, Run memory aDown) =
                _mirrored(JUNIOR_TOTAL * shareBps[i] / 10_000, 1);
            int256 mean = (_net(cUp, aUp, 0) + _net(cDown, aDown, 0)) / 2;
            string memory share = vm.toString(uint256(shareBps[i]));
            emit log_named_int(string.concat("Junior share bps ", share, ": large LP net, mean"), mean);
            emit log_named_int(
                string.concat("Junior share bps ", share, ": remaining Junior LP delta, mean"),
                (_rem(cUp, aUp, 0) + _rem(cDown, aDown, 0)) / 2
            );
            if (shareBps[i] == 3000) {
                assertLt(mean, 0, "net loss at 30% of Junior");
            }
            if (shareBps[i] >= 5000) {
                assertGt(mean, 0, "net gain at 50% of Junior and above");
            }
            assertGt(mean, previous, "net rises with the share of Junior");
            previous = mean;
        }
    }

    /// @notice Four consecutive cycles at 80% of Junior on a flat book, under the conditions of the previous test, each
    ///         paired end to end with a control that runs the same LP timeline (exit, re-deposit) without trades.
    ///         Cycles one to three read the remaining LP's settled exit as if it left right after the cycle; the
    ///         fourth is the end of the run.
    function test_repeatsEveryCycle_multiCycleFlatBook() public {
        (Run memory cUp, Run memory aUp, Run memory cDown, Run memory aDown) = _mirrored(WHALE_80, 4);
        int256[2] memory total;
        int256[2] memory previous;
        for (uint256 k; k < 4; ++k) {
            int256 up = _net(cUp, aUp, k);
            int256 down = _net(cDown, aDown, k);
            int256 otherUp = _rem(cUp, aUp, k);
            int256 otherDown = _rem(cDown, aDown, k);
            assertGt(up, 0, "every cycle nets the large LP a gain (up path)");
            assertGt(down, 0, "every cycle nets the large LP a gain (down path)");
            assertLt(otherUp, previous[0], "the remaining Junior LP loss grows every cycle (up path)");
            assertLt(otherDown, previous[1], "the remaining Junior LP loss grows every cycle (down path)");
            previous[0] = otherUp;
            previous[1] = otherDown;
            total[0] += up;
            total[1] += down;
            string memory n = vm.toString(k + 1);
            emit log_named_uint(string.concat("cycle ", n, ": hours after the first exit epoch"), 3 * k);
            emit log_named_int(string.concat("cycle ", n, ": large LP net, up path"), up);
            emit log_named_int(string.concat("cycle ", n, ": large LP net, down path"), down);
            emit log_named_int(string.concat("cycle ", n, ": remaining Junior LP cumulative delta, up path"), otherUp);
            emit log_named_int(
                string.concat("cycle ", n, ": remaining Junior LP cumulative delta, down path"), otherDown
            );
            emit log_named_int(
                string.concat("cycle ", n, ": large LP net so far, mean of both paths"), (total[0] + total[1]) / 2
            );
        }
        emit log_named_uint(
            "four cycles: hours from the first exit epoch to the last remaining-LP exit",
            (aUp.endTime - (MONDAY + 4 hours)) / 1 hours
        );
        emit log_named_int("four cycles: large LP net, up path", total[0]);
        emit log_named_int("four cycles: large LP net, down path", total[1]);
        emit log_named_int("four cycles: large LP net, mean of both paths", (total[0] + total[1]) / 2);
        emit log_named_int(
            "four cycles: remaining Junior LP delta, mean of both paths", (previous[0] + previous[1]) / 2
        );
    }

}
```

Run from `packages/perps` (the repo compiles with via_ir, so the first run takes a few minutes):

```bash
FOUNDRY_PROFILE=quick forge test --match-path test/poc/RefundableSkewChargeLpExit.t.sol -vv
```

Output:

```
Ran 5 tests for test/poc/RefundableSkewChargeLpExit.t.sol:RefundableSkewChargeLpExitTest
[PASS] test_breakEvenShare() (gas: 150313894)
Logs:
  Junior share bps 3000: large LP net, mean: -292168859
  Junior share bps 3000: remaining Junior LP delta, mean: -187885144
  Junior share bps 4000: large LP net, mean: -49461995
  Junior share bps 4000: remaining Junior LP delta, mean: -430367918
  Junior share bps 4500: large LP net, mean: 71891431
  Junior share bps 4500: remaining Junior LP delta, mean: -551578770
  Junior share bps 5000: large LP net, mean: 193244852
  Junior share bps 5000: remaining Junior LP delta, mean: -672761129
  Junior share bps 6000: large LP net, mean: 435951678
  Junior share bps 6000: remaining Junior LP delta, mean: -914997696
  Junior share bps 8000: large LP net, mean: 921365222
  Junior share bps 8000: remaining Junior LP delta, mean: -1398063458

[PASS] test_control_exitAfterTheClose_noTransfer() (gas: 24206555)
Logs:
  exit after the close: large LP exit delta: 385410975
  exit after the close: remaining Junior LP delta: 96352744
  exit after the close: large LP net: -576793253
  exit after the close: trading result: -962204228
  exit between: large LP exit delta: 1917763450
  exit between: remaining Junior LP delta: -1434093383
  exit between: large LP net: 957473083
  exit between: trading result: -960290367
  moved by the exit timing alone, large LP net: 1534266336
  moved by the exit timing alone, remaining Junior LP: -1530446127

[PASS] test_exitBetweenOpenAndClose_realisticConfidenceAndPricePath() (gas: 24982838)
Logs:
  skew charge at the open: 2398281147
  large LP exit delta, up path: 2133559784
  trading result, up path: -1380434366
  large LP net, up path: 753125418
  large LP exit delta, down path: 1749751392
  trading result, down path: -660146365
  large LP net, down path: 1089605027
  large LP net, mean of both paths: 921365222
  remaining Junior LP delta, up path: -1230242558
  remaining Junior LP delta, down path: -1565884358
  remaining Junior LP delta, mean of both paths: -1398063458

[PASS] test_openBooksRefundableChargeAsDistributableEquity() (gas: 7377543)
Logs:
  skew charge booked as LP equity at the open: 2398401065
  rebate paid back to the trading account at the close: 1918401065
  close fee netted inside the rebate: 480000000
  carry paid by the trading account: 89627
  large LP exit funded between the open and the close: 1601903136928

[PASS] test_repeatsEveryCycle_multiCycleFlatBook() (gas: 76021249)
Logs:
  cycle 1: hours after the first exit epoch: 0
  cycle 1: large LP net, up path: 753125418
  cycle 1: large LP net, down path: 1089605027
  cycle 1: remaining Junior LP cumulative delta, up path: -1230242558
  cycle 1: remaining Junior LP cumulative delta, down path: -1565884358
  cycle 1: large LP net so far, mean of both paths: 921365222
  cycle 2: hours after the first exit epoch: 3
  cycle 2: large LP net, up path: 754431223
  cycle 2: large LP net, down path: 1091239326
  cycle 2: remaining Junior LP cumulative delta, up path: -2461787670
  cycle 2: remaining Junior LP cumulative delta, down path: -3133398946
  cycle 2: large LP net so far, mean of both paths: 1844200497
  cycle 3: hours after the first exit epoch: 6
  cycle 3: large LP net, up path: 755736415
  cycle 3: large LP net, down path: 1092874888
  cycle 3: remaining Junior LP cumulative delta, up path: -3694634725
  cycle 3: remaining Junior LP cumulative delta, down path: -4702545023
  cycle 3: large LP net so far, mean of both paths: 2768506148
  cycle 4: hours after the first exit epoch: 9
  cycle 4: large LP net, up path: 757041021
  cycle 4: large LP net, down path: 1094511739
  cycle 4: remaining Junior LP cumulative delta, up path: -4928783137
  cycle 4: remaining Junior LP cumulative delta, down path: -6273323876
  cycle 4: large LP net so far, mean of both paths: 3694282528
  four cycles: hours from the first exit epoch to the last remaining-LP exit: 10
  four cycles: large LP net, up path: 3020334077
  four cycles: large LP net, down path: 4368230980
  four cycles: large LP net, mean of both paths: 3694282528
  four cycles: remaining Junior LP delta, mean of both paths: -5601053506

Suite result: ok. 5 passed; 0 failed; 0 skipped; finished in 176.58ms (344.59ms CPU time)

Ran 1 test suite in 196.41ms (176.58ms CPU time): 5 tests passed, 0 failed, 0 skipped (5 total tests)
```
