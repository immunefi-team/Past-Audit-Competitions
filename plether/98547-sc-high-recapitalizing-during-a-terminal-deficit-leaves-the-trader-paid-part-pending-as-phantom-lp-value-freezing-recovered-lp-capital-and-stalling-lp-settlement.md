# \#98547 [SC-High]  Recapitalizing during a terminal deficit leaves the trader-paid part pending as phantom LP value, freezing recovered LP capital and stalling LP settlement

**Submitted on Nov 8th 2026 at 16:24:27 UTC by @Afriauditor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98547
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description


## Summary

When trader claims exceed pool cash, the protocol's runbook is to recapitalize (`SECURITY.md:1499-1509`). `recordClaimantInflow` records the whole recap in `pendingRecapitalizationAssets`, including the part that only fills the trader-claim shortfall and is then paid to traders. Reconcile can settle pending value only up to `distributable`, and the deficit part never appears there, so it is never settled and never written off. Every withdrawal snapshot reserves it from LP free cash: LPs cannot redeem the capital the recap actually restored, and `settleLpEpoch` reverts with `HousePool__NoLpEpochProgress`, rolling back its own reconcile. The residual shrinks by only one `distributable` per persisted reconcile, and the only reconcile the owner can force is a senior-rate change behind a 72-hour timelock.

In the PoC, a 10,000 USDG recap into an 8,000 USDG deficit pays the trader in full, but the remaining 2,000 USDG of LP capital stays locked through four owner timelocks: the Senior LP exits 20 days after the recap. The same 2,000 USDG without a deficit is paid out in the first settlement. Each of those reconciles also rewrites Senior's high-water mark lower.

## Description

**1. The deficit part of a recap is booked as pending LP value.** `recordClaimantInflow` adds the full amount to `pendingRecapitalizationAssets` (`HousePool.sol:644-645`) without looking at the terminal deficit. When trader claims exceed pool cash, the part of the recap that fills the shortfall is recorded as if it were new LP equity, but it belongs to the traders, and `settleTraderClaim` pays it out (`CfdEngine.sol:559-590`). In the PoC the pool holds 11,968 USDG against a 19,968 USDG claim; the owner recaps 10,000 USDG, the trader is paid in full, 2,000 USDG of LP cash remains, and `pendingRecapitalizationAssets` still reads 10,000.

**2. Reconcile can never settle that part, and never writes it off.** Each reconcile caps the settleable bucket at `distributable`, terminal assets minus trader claims (`HousePool.sol:1565-1566`, `:1829-1834`; `HousePoolPendingPreviewLib.sol:72-82`; `HousePoolAccountingLib.sol:60-78`). The deficit part is netted against the claims, so it never appears in `distributable`. `_applyPendingClaimantBucketsLive` decrements only what it settled (`HousePool.sol:1772`); the rest stays pending. Later reconciles "settle" `min(pending, distributable)` against value the LPs already own: the plan reserves it out of distributable, booking a loss on existing principal (`HousePoolReconcilePlanLib.sol:68-71`), and the bucket re-creates it as recapitalization. Value is unchanged, but the residual falls by only one `distributable` per reconcile.

**3. The residual is reserved from LP cash, so settlement cannot run.** `_buildWithdrawalSnapshot` reserves the whole pending balance from free assets (`HousePool.sol:1733`). In `_settleLpEpoch`, Senior's budget is `min(freeAssets, seniorPrincipal)` (`:848`), Junior's capacity comes from the same free assets, and deposit activation is deferred while any pending claimant value exists (`:908`, `:1108`). While the pending balance left after the call's own reconcile still covers the LP cash, every phase does nothing and `_emitLpEpochSettled` reverts with `HousePool__NoLpEpochProgress` (`:1183-1184`), undoing the reconcile that ran inside the call (`:835`). So settlement cannot even work the residual down.

**4. Only the owner can work it down, one 72-hour timelock at a time.** Outside settlement, `finalizePoolConfig` reconciles only when the senior rate changes (`HousePool.sol:336-342`), after the 72-hour timelock. `assignUnassignedAssets` needs unassigned value, and there is none. `initializeSeedPosition` runs once per tranche. `reconcile()` is `onlyVault` (`:1471-1473`), and neither vault calls it. Exits therefore resume only after deficit ÷ fresh equity senior-rate changes (four in the PoC), once later trading revenue lifts LP cash above the remaining residual, or if the owner pays the deficit a second time. A second 8,000 USDG recap leaves the same 8,000 residual but enough cash to settle it one epoch later, and that extra 8,000 ends up with the LPs.

**5. Each persisted reconcile also lowers Senior's high-water mark.** Once the reserve consumes principal to zero, the re-created bucket takes the zero-principal branch of `applyClaimantRecapitalizationIntent` and rewrites the mark to the shrinking pending target (`HousePoolPendingPreviewLib.sol:118-122`): 10,000 → 8,000 → 6,000 → 4,000, until it equals principal. That branch's own NatSpec names the full recapitalization intent as the target mark (`:110`). The first drop, from Senior's pre-loss 51,000 to 10,000, is the separately reported mark reset; the walk-down below the recap amount is caused by this residual.

**6. This contradicts the documented design.** The spec describes a recap as "intended to restore waterfall claimants", allocated to "senior restoration or unassigned assets" (`ACCOUNTING_SPEC.md:634`, `:642`). Pending buckets are reserved so the same inflow is not counted twice (`:336-340`), not to hold value that has already been paid to traders. `SECURITY.md:1515` requires LP capital to remain "immediately accessible USDG". Prior reports touch the pending buckets (Cecuro's MED-12 on revenue routing after a post-wipe recap, and its stale-mark senior-rate finalization finding), but neither covers recap value consumed by trader claims.

**Fix.** Apply a recapitalization to the existing terminal deficit first and queue only the excess. In `recordClaimantInflow`, measure the deficit from a reconcile snapshot taken before `accountedAssets += amount`, record `pendingRecapitalizationAssets += amount - min(amount, deficit)`, and emit the consumed amount. The deficit part restores trader solvency, not LP principal; any later recovery of mark-to-market losses still reaches Senior first as ordinary revenue. Independently, keeping the stored mark on the zero-principal branch (`max(oldMark, target, amount)`) stops the walk-down.

## Impact

**Severity: temporary freezing of LP funds; LP settlement unable to operate.**

- **Recovered LP capital is frozen.** In the PoC, the Senior LP's 1,960.78 USDG share of the recovered 2,000 USDG cannot be redeemed for 20 days after the recap. The owner's fastest timelock path, four senior-rate changes, took 13 days in the PoC (four 72-hour timelocks plus a weekend). The number of cycles is deficit ÷ fresh equity, so a recap that adds 10% over the deficit needs ten (30 days). An owner who doesn't know to cycle the senior rate leaves it frozen until trading revenue lifts LP cash above the residual.
- **No LP queue moves.** Every `settleLpEpoch` call reverts for the whole period, so no redemption or deposit epoch is processed.
- **Unfreezing at once costs the deficit a second time.** A second recap equal to the deficit restores exits one epoch later, and that capital becomes LP principal rather than covering anything.
- **Senior's restoration claim is ratcheted away.** Each forced reconcile lowers the high-water mark until it equals principal, so later revenue no longer restores what the recap was meant to restore.




## Proof of Concept

## Proof of Concept

## Proof of concept

File: `packages/perps/test/poc/M1_RecapDuringDeficit.t.sol`, on the repo's `BasePerpTest` fixture with the team's degraded-mode risk parameters. The loss is modelled the way the team's own `DegradedSettlementBehavior.t.sol:136-137` does it (drain the pool, then a winning close becomes a trader claim). The recap goes through the public `CfdEngine.recapitalizePool`, the trader is paid through `CfdEngine.settleTraderClaim`, settlement uses the permissionless `HousePool.settleLpEpoch`, and the owner's reconciles use `proposePoolConfig`/`finalizePoolConfig` with real 72-hour waits. No direct `reconcile()` calls.

```bash
cd packages/perps
forge test --match-path test/poc/M1_RecapDuringDeficit.t.sol -vv
```

Three tests:

- `test_RecapDuringDeficitFreezesRecoveredPrincipal`: 10,000 USDG recap into an 8,000 USDG deficit. The trader is paid, 2,000 USDG of LP cash remains, and the preview shows Senior owning it with a withdrawable amount of 0. Settlement reverts immediately and a week later. Four owner rate-change reconciles each settle 2,000 of the phantom and lower the mark; only then does settlement fund the Senior LP.
- `test_Control_SameFreshEquityWithoutDeficitIsWithdrawableAtOnce`: the same 2,000 USDG of fresh equity after a loss with no outstanding trader claims. The first settlement pays the Senior LP 1,960.78 USDG.
- `test_OwnerUnfreezesQuicklyOnlyByPayingTheDeficitAgain`: a second 8,000 USDG recap leaves the same 8,000 residual; exits resume one settlement later and the Senior LP receives 9,803.92 USDG.

```
[PASS] test_Control_SameFreshEquityWithoutDeficitIsWithdrawableAtOnce() (gas: 727022)
Logs:
  Control: Senior funded in first settlement: 1960783612

[PASS] test_OwnerUnfreezesQuicklyOnlyByPayingTheDeficitAgain() (gas: 2099981)
Logs:
  Trader claim paid from the recap: 19968000000
  Pool cash after traders are paid: 2000000000
  pendingRecapitalizationAssets:   10000000000
  Senior principal (preview):      2000000000
  Senior max withdraw (preview):   0
  Owner capital in (recap + second recap): 18000000000
  Senior funded, first settlement:         2000000000
  Senior funded, second settlement:        7803918058
  Senior LP receives:                      9803918058

[PASS] test_RecapDuringDeficitFreezesRecoveredPrincipal() (gas: 3711303)
Logs:
  Trader claim paid from the recap: 19968000000
  Pool cash after traders are paid: 2000000000
  pendingRecapitalizationAssets:   10000000000
  Senior principal (preview):      2000000000
  Senior max withdraw (preview):   0
  Senior mark before the loss:     51000018264
  owner rate-change reconcile 1
     pendingRecapitalizationAssets 8000000000
     seniorPrincipal               2000000000
     seniorHighWaterMark           10000000000
     getMaxSeniorWithdraw          0
  owner rate-change reconcile 2
     pendingRecapitalizationAssets 6000000000
     seniorPrincipal               2000000000
     seniorHighWaterMark           8000000000
     getMaxSeniorWithdraw          0
  owner rate-change reconcile 3
     pendingRecapitalizationAssets 4000000000
     seniorPrincipal               2000000000
     seniorHighWaterMark           6000000000
     getMaxSeniorWithdraw          0
  owner rate-change reconcile 4
     pendingRecapitalizationAssets 2000000000
     seniorPrincipal               2000000000
     seniorHighWaterMark           4000000000
     getMaxSeniorWithdraw          0
  After settlement: pending / mark 0 39216388
  Senior LP finally receives:      1960783612
  Owner timelock cycles needed:    4
  Days spent in owner cycles:      13
  Days from recap to first exit:   20
```

The final mark of 39.22 USDG equals Senior's remaining principal after the redemption (the seed holder's share), so no restoration claim is left.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {console2} from "forge-std/console2.sol";

/// @notice A recapitalization recorded while the pool is in terminal deficit keeps the deficit-covering part in
///         `pendingRecapitalizationAssets` after that cash has been paid to traders. The phantom balance is reserved
///         from LP free cash, so recovered LP principal cannot be redeemed and `settleLpEpoch` cannot make progress.
contract M1_RecapDuringDeficitTest is BasePerpTest {

    address internal constant SENIOR_LP = address(0x5E1);
    address internal constant TRADER = address(0x7AD3);

    uint256 internal constant DEFICIT = 8000e6; // trader claims the pool cannot cover
    uint256 internal constant FRESH = 2000e6; // recap in excess of the deficit: the only new LP equity
    uint256 internal constant RECAP = DEFICIT + FRESH;

    /// @dev Same parameters as the team's DegradedSettlementBehavior.t.sol.
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0,
            maxSkewRatio: 1e18,
            maintMarginBps: 100,
            initMarginBps: 150,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 5e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 100_000e6;
    }

    function setUp() public override {
        super.setUp();
        _fundSenior(SENIOR_LP, 50_000e6);
    }

    // ---------------------------------------------------------------------------------------------
    // Main case: recapitalization during a trader-claim deficit
    // ---------------------------------------------------------------------------------------------

    function test_RecapDuringDeficitFreezesRecoveredPrincipal() public {
        uint256 markBefore = pool.seniorHighWaterMark();
        uint256 recapTime = _lossThenRecapDuringDeficit();

        // A Senior LP tries to exit the recovered pool; settlement cannot make progress, now or a week later.
        _seniorRequestsFullExit();
        vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
        pool.settleLpEpoch(0, 0);
        vm.warp(vm.getBlockTimestamp() + 7 days);
        _skipFrozenWindow();
        vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
        pool.settleLpEpoch(0, 0);

        // The only reconcile left to the owner is a senior-rate change, one per 72-hour timelock.
        // Each one settles only FRESH of the phantom and rewrites the mark lower.
        console2.log("Senior mark before the loss:    ", markBefore);
        uint256 ownerCyclesStart = vm.getBlockTimestamp();
        uint256 cycles;
        IHousePool.LpEpochSettlementResult memory result;
        bool settled;
        while (!settled) {
            _ownerRateChangeReconcile();
            ++cycles;
            console2.log("owner rate-change reconcile", cycles);
            console2.log("   pendingRecapitalizationAssets", pool.pendingRecapitalizationAssets());
            console2.log("   seniorPrincipal              ", pool.seniorPrincipal());
            console2.log("   seniorHighWaterMark          ", pool.seniorHighWaterMark());
            console2.log("   getMaxSeniorWithdraw         ", pool.getMaxSeniorWithdraw());
            assertEq(pool.seniorPrincipal(), FRESH, "the recovered principal is FRESH throughout");
            assertEq(pool.getMaxSeniorWithdraw(), 0, "the phantom reserves all of it");
            try pool.settleLpEpoch(0, 0) returns (IHousePool.LpEpochSettlementResult memory r) {
                result = r;
                settled = true;
            } catch (bytes memory err) {
                assertEq(bytes4(err), IHousePool.HousePool__NoLpEpochProgress.selector, "stall, not another error");
            }
            assertLt(cycles, 10);
        }
        assertEq(cycles, DEFICIT / FRESH, "one owner timelock per FRESH of phantom, the last cleared in settlement");
        assertEq(pool.pendingRecapitalizationAssets(), 0, "phantom finally consumed");
        assertGt(result.seniorFundedAssets, 0, "Senior redemption funded at last");
        assertEq(
            pool.seniorHighWaterMark(), pool.seniorPrincipal(), "the mark has walked down to principal: no claim left"
        );

        // The Senior LP is paid from cash that was in the pool all along.
        uint256 claimable = seniorVault.maxWithdraw(SENIOR_LP);
        vm.prank(SENIOR_LP);
        seniorVault.withdraw(claimable, SENIOR_LP, SENIOR_LP);
        console2.log("After settlement: pending / mark", pool.pendingRecapitalizationAssets(), pool.seniorHighWaterMark());
        console2.log("Senior LP finally receives:     ", settlementToken.balanceOf(SENIOR_LP));
        console2.log("Owner timelock cycles needed:   ", cycles);
        console2.log("Days spent in owner cycles:     ", (vm.getBlockTimestamp() - ownerCyclesStart) / 1 days);
        console2.log("Days from recap to first exit:  ", (vm.getBlockTimestamp() - recapTime) / 1 days);
        assertGt(settlementToken.balanceOf(SENIOR_LP), 0);
    }

    /// @notice The owner's other way out: recapitalize the deficit a second time. A second recap leaves the same
    ///         residual, so the LP cash is released only after it and the extra capital goes to LPs.
    function test_OwnerUnfreezesQuicklyOnlyByPayingTheDeficitAgain() public {
        _lossThenRecapDuringDeficit();
        _seniorRequestsFullExit();

        _recapitalize(DEFICIT); // the deficit, paid a second time
        IHousePool.LpEpochSettlementResult memory first = pool.settleLpEpoch(0, 0);
        assertEq(pool.pendingRecapitalizationAssets(), DEFICIT, "a second recap leaves the same residual");
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        IHousePool.LpEpochSettlementResult memory second = pool.settleLpEpoch(0, 0);
        assertEq(pool.pendingRecapitalizationAssets(), 0, "residual consumed one settlement later");

        uint256 claimable = seniorVault.maxWithdraw(SENIOR_LP);
        vm.prank(SENIOR_LP);
        seniorVault.withdraw(claimable, SENIOR_LP, SENIOR_LP);
        console2.log("Owner capital in (recap + second recap):", RECAP + DEFICIT);
        console2.log("Senior funded, first settlement:        ", first.seniorFundedAssets);
        console2.log("Senior funded, second settlement:       ", second.seniorFundedAssets);
        console2.log("Senior LP receives:                     ", settlementToken.balanceOf(SENIOR_LP));
    }

    /// @dev Loss with an uncovered trader claim, then the documented owner recapitalization and trader payout.
    function _lossThenRecapDuringDeficit() internal returns (uint256 recapTime) {
        // Loss: the pool is drained (the team's modelling of earlier losses, DegradedSettlementBehavior.t.sol)
        // and a winning close is owed more than the pool holds, so the payout becomes a trader claim.
        _fundTrader(TRADER, 11_000e6);
        _open(TRADER, CfdTypes.Side.LONG, 100_000e18, 9000e6, 1e8);
        uint256 poolAssets = pool.totalAssets();
        vm.prank(address(pool));
        settlementToken.transfer(address(0xDEAD), poolAssets - 12_000e6);
        _close(TRADER, CfdTypes.Side.LONG, 100_000e18, 80_000_000);
        uint256 claim = engine.traderClaimBalanceAssets(TRADER);
        assertGt(claim, DEFICIT, "the close created a trader claim");
        // Fix the deficit at exactly DEFICIT for readable numbers.
        uint256 excessCash = pool.totalAssets() - (claim - DEFICIT);
        vm.prank(address(pool));
        settlementToken.transfer(address(0xDEAD), excessCash);
        assertEq(claim - pool.totalAssets(), DEFICIT, "trader claims exceed pool cash by DEFICIT");

        // Documented recovery: the owner recapitalizes the deficit plus fresh equity, and the trader is paid.
        _recapitalize(RECAP);
        vm.prank(TRADER);
        engine.settleTraderClaim(TRADER);
        recapTime = vm.getBlockTimestamp();

        assertEq(engine.traderClaimBalanceAssets(TRADER), 0, "trader paid in full");
        assertEq(pool.totalAssets(), FRESH, "only FRESH of LP-owned cash remains in the pool");
        assertEq(pool.pendingRecapitalizationAssets(), RECAP, "the whole recap is still pending, deficit part included");
        (uint256 seniorPreview,, uint256 seniorMaxPreview,) = pool.getPendingTrancheState();
        assertEq(seniorPreview, FRESH, "after reconcile Senior would own the FRESH cash");
        assertEq(seniorMaxPreview, 0, "but none of it is withdrawable");
        console2.log("Trader claim paid from the recap:", claim);
        console2.log("Pool cash after traders are paid:", pool.totalAssets());
        console2.log("pendingRecapitalizationAssets:  ", pool.pendingRecapitalizationAssets());
        console2.log("Senior principal (preview):     ", seniorPreview);
        console2.log("Senior max withdraw (preview):  ", seniorMaxPreview);
    }

    function _seniorRequestsFullExit() internal {
        vm.warp(vm.getBlockTimestamp() + 2 hours);
        uint256 shares = seniorVault.balanceOf(SENIOR_LP);
        vm.prank(SENIOR_LP);
        seniorVault.requestRedeem(shares, SENIOR_LP, SENIOR_LP);
        vm.warp(vm.getBlockTimestamp() + 2 hours);
    }

    // ---------------------------------------------------------------------------------------------
    // Control: the same FRESH equity after a loss with no outstanding trader claims
    // ---------------------------------------------------------------------------------------------

    function test_Control_SameFreshEquityWithoutDeficitIsWithdrawableAtOnce() public {
        uint256 poolAssets = pool.totalAssets();
        vm.prank(address(pool));
        settlementToken.transfer(address(0xDEAD), poolAssets); // wiped to exactly zero, no claims outstanding

        _recapitalize(FRESH);
        (uint256 seniorPreview,, uint256 seniorMaxPreview,) = pool.getPendingTrancheState();
        assertEq(seniorPreview, FRESH, "Senior owns the FRESH cash");
        assertEq(seniorMaxPreview, FRESH, "and all of it is withdrawable");

        _seniorRequestsFullExit();
        IHousePool.LpEpochSettlementResult memory result = pool.settleLpEpoch(0, 0);
        assertGt(result.seniorFundedAssets, 0, "first settlement funds the Senior LP");
        assertEq(pool.pendingRecapitalizationAssets(), 0, "recap fully settled in one reconcile");
        console2.log("Control: Senior funded in first settlement:", result.seniorFundedAssets);
    }

    // ---------------------------------------------------------------------------------------------
    // Helpers
    // ---------------------------------------------------------------------------------------------

    function _recapitalize(
        uint256 amount
    ) internal {
        settlementToken.mint(address(this), amount);
        settlementToken.approve(address(engine), amount);
        engine.recapitalizePool(amount);
    }

    /// @dev The owner's only reconciling call that does not need LP-epoch progress: a senior-rate change.
    function _ownerRateChangeReconcile() internal {
        IHousePool.PoolConfig memory config = _currentPoolConfig();
        config.seniorRateBps = config.seniorRateBps == 800 ? 801 : 800;
        pool.proposePoolConfig(config);
        vm.warp(pool.poolConfigActivationTime());
        _skipFrozenWindow();
        vm.prank(address(router));
        engine.updateMarkPrice(1e8, uint64(vm.getBlockTimestamp()));
        pool.finalizePoolConfig();
    }

    /// @dev Owner and keepers act on weekdays; skip the weekend oracle-frozen window.
    function _skipFrozenWindow() internal {
        for (uint256 i; i < 72 && engine.isOracleFrozen(); ++i) {
            vm.warp(vm.getBlockTimestamp() + 1 hours);
        }
    }

}
```
